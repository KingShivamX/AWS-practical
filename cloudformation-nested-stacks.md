# CloudFormation Nested Stacks — VPC + IAM + EC2
## Complete Step-by-Step Guide (Full Marks)

Read this top to bottom and do it side by side on the AWS console.

**Region:** use **`us-east-1`** (US East, N. Virginia) for everything.

---

## 🗂️ PART 0 — WHAT WE ARE BUILDING

```
You (console / CLI)
        │
        │  deploy root-stack.yaml
        ▼
┌──────────────────────────────────────────────────────────┐
│  Root Stack  (root-stack.yaml)                           │
│                                                          │
│   ┌──────────────┐   ┌──────────────┐                   │
│   │  VPC Stack   │   │  IAM Stack   │                   │
│   │  (Child 1)   │   │  (Child 2)   │                   │
│   │              │   │              │                    │
│   │ VPC          │   │ IAM Role     │                   │
│   │ Subnet       │   │ Instance     │                   │
│   │ IGW          │   │ Profile      │                   │
│   │ Route Table  │   │              │                   │
│   │              │   │              │                    │
│   │ Outputs:     │   │ Outputs:     │                   │
│   │  VPCId       │   │  ProfileName │                   │
│   │  SubnetId    │   │              │                    │
│   └──────┬───────┘   └──────┬───────┘                   │
│          │                  │  (outputs passed as params)│
│          └────────┬─────────┘                            │
│                   ▼                                      │
│          ┌────────────────┐                              │
│          │   EC2 Stack    │                              │
│          │   (Child 3)    │                              │
│          │                │                              │
│          │ Security Group │                              │
│          │ EC2 Instance   │                              │
│          │ Apache (httpd) │                              │
│          │ index.html     │                              │
│          │                │                              │
│          │ Output:        │                              │
│          │  PublicIP      │                              │
│          └────────────────┘                              │
└──────────────────────────────────────────────────────────┘
        │
        ▼
   Browser: http://EC2-PUBLIC-IP
   Shows: "Hello from CloudFormation Nested Stack!"
```

**Flow:** Root stack creates VPC stack and IAM stack first (`DependsOn`). Each child exports Outputs. Root stack passes those outputs as Parameters to EC2 stack. EC2 starts with UserData that installs Apache and writes an HTML page. One click on CloudFormation creates all 3 child stacks automatically.

---

## 🗂️ PART 1 — RUBRIC CHECKLIST

| Rubric point | How it is met | Where |
|---|---|---|
| Nested stacks using `AWS::CloudFormation::Stack` | All 3 children declared in root | root-stack.yaml |
| VPC with subnet, IGW, route table | Complete networking in vpc-stack | vpc-stack.yaml |
| IAM role + instance profile | EC2SSMRole + profile in iam-stack | iam-stack.yaml |
| EC2 using VPC output + IAM output | EC2 takes SubnetId, VPCId, ProfileName as parameters from root | ec2-stack.yaml |
| Outputs exported between stacks | `!GetAtt ChildStack.Outputs.Key` in root | root-stack.yaml |
| `DependsOn` so VPC + IAM finish first | `DependsOn: [VPCStack, IAMStack]` on EC2Stack | root-stack.yaml |
| Parameters passed between stacks | 3 params: VPCId, SubnetId, InstanceProfileName | ec2-stack.yaml |
| Browser-visible webpage | Apache + index.html via UserData | ec2-stack.yaml |

---

## 🗂️ PART 2 — YOUR FILES (4 YAML FILES, COPY-PASTE READY)

> **Rule:** Every child stack template must be uploaded to S3 **before** you deploy the root stack. CloudFormation fetches them by S3 URL at deploy time.

---

### FILE 1: `vpc-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Child Stack 1 - VPC, Subnet, Internet Gateway, Route Table

Resources:

  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true        # lets instances resolve DNS names
      EnableDnsHostnames: true      # gives instances public DNS hostnames
      Tags:
        - Key: Name
          Value: NestedVPC

  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true     # auto-assign public IP to instances in this subnet
      AvailabilityZone: !Select [0, !GetAZs '']
      # !GetAZs '' returns all AZs in the current region as a list
      # !Select [0, ...] picks the first one (e.g. us-east-1a)
      Tags:
        - Key: Name
          Value: NestedPublicSubnet

  MyIGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: NestedIGW

  AttachIGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref MyIGW

  MyRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: NestedPublicRT

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachIGW              # route needs IGW attached first
    Properties:
      RouteTableId: !Ref MyRouteTable
      DestinationCidrBlock: 0.0.0.0/0  # all traffic not matching a more specific route
      GatewayId: !Ref MyIGW

  SubnetRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref MySubnet
      RouteTableId: !Ref MyRouteTable

Outputs:
  VPCId:
    Description: VPC ID for use by EC2 stack
    Value: !Ref MyVPC
    Export:
      Name: NestedVPCId             # cross-stack export name (must be unique in region)

  SubnetId:
    Description: Public Subnet ID for use by EC2 stack
    Value: !Ref MySubnet
    Export:
      Name: NestedSubnetId
```

---

### FILE 2: `iam-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Child Stack 2 - IAM Role and Instance Profile for EC2

Resources:

  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      RoleName: EC2SSMRole
      Description: Allows EC2 to be managed by SSM and access AWS services
      AssumeRolePolicyDocument:
        # Trust policy: who can assume this role
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com   # EC2 service can assume this role
            Action: sts:AssumeRole
      ManagedPolicyArns:
        # SSM lets you connect to EC2 without SSH key (Session Manager)
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      Tags:
        - Key: Name
          Value: EC2SSMRole

  EC2InstanceProfile:
    # Instance Profile = the container that attaches an IAM role to an EC2 instance
    # You cannot attach an IAM Role directly to EC2 — it must go through an Instance Profile
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: EC2SSMInstanceProfile
      Roles:
        - !Ref EC2Role

Outputs:
  InstanceProfileName:
    Description: Instance Profile Name for use by EC2 stack
    Value: !Ref EC2InstanceProfile    # !Ref on InstanceProfile returns its NAME
    Export:
      Name: NestedInstanceProfileName
```

---

### FILE 3: `ec2-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Child Stack 3 - EC2 Instance with Apache web server

Parameters:
  VPCId:
    Type: String
    Description: VPC ID passed from root stack (from VPC stack output)

  SubnetId:
    Type: String
    Description: Subnet ID passed from root stack (from VPC stack output)

  InstanceProfileName:
    Type: String
    Description: Instance Profile name passed from root stack (from IAM stack output)

Resources:

  WebServerSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP and SSH access
      VpcId: !Ref VPCId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0       # allow HTTP from anywhere (needed for browser access)
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0       # allow SSH from anywhere (restrict to your IP in production)
      Tags:
        - Key: Name
          Value: WebServerSG

  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      # Dynamic SSM reference — always gets latest Amazon Linux 2023 AMI
      # No hardcoded AMI ID needed; works in any region
      ImageId: '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64}}'
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !Ref WebServerSG
      IamInstanceProfile: !Ref InstanceProfileName
      Tags:
        - Key: Name
          Value: NestedEC2WebServer
      UserData:
        # UserData runs once when the instance first boots (as root)
        # Fn::Base64 encodes the script because EC2 expects base64-encoded UserData
        Fn::Base64: !Sub |
          #!/bin/bash
          set -e

          # Install Apache web server
          yum update -y
          yum install -y httpd

          # Start Apache and enable on reboot
          systemctl start httpd
          systemctl enable httpd

          # Get instance metadata using IMDSv2 (secure, modern method)
          TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
            -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
          INSTANCE_ID=$(curl -s \
            -H "X-aws-ec2-metadata-token: $TOKEN" \
            http://169.254.169.254/latest/meta-data/instance-id)
          AZ=$(curl -s \
            -H "X-aws-ec2-metadata-token: $TOKEN" \
            http://169.254.169.254/latest/meta-data/placement/availability-zone)

          # Write the HTML page
          cat > /var/www/html/index.html <<EOF
          <!DOCTYPE html>
          <html>
          <head>
            <title>CloudFormation Nested Stack</title>
            <style>
              body { font-family: Arial, sans-serif; max-width: 600px; margin: 60px auto; }
              h1   { color: #FF9900; }
              .info { background: #f0f0f0; padding: 15px; border-radius: 6px; margin-top: 20px; }
            </style>
          </head>
          <body>
            <h1>Hello from CloudFormation Nested Stack!</h1>
            <div class="info">
              <p><strong>Stack:</strong> ${AWS::StackName}</p>
              <p><strong>Region:</strong> ${AWS::Region}</p>
              <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
              <p><strong>Availability Zone:</strong> $AZ</p>
            </div>
            <p>Deployed via: Root Stack → VPC Stack + IAM Stack + EC2 Stack</p>
          </body>
          </html>
          EOF

Outputs:
  EC2PublicIP:
    Description: Public IP address of the web server
    Value: !GetAtt MyEC2.PublicIp

  WebsiteURL:
    Description: URL to access the web server
    Value: !Sub 'http://${MyEC2.PublicIp}'
```

---

### FILE 4: `root-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Root Stack - orchestrates VPC, IAM, and EC2 nested stacks

Resources:

  VPCStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      # S3 URL of the child template — CloudFormation fetches it at deploy time
      TemplateURL: https://YOUR-BUCKET.s3.us-east-1.amazonaws.com/vpc-stack.yaml
      TimeoutInMinutes: 10   # if VPC stack takes longer than 10 min, mark as failed

  IAMStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://YOUR-BUCKET.s3.us-east-1.amazonaws.com/iam-stack.yaml
      TimeoutInMinutes: 5

  EC2Stack:
    Type: AWS::CloudFormation::Stack
    DependsOn:
      - VPCStack     # wait for VPC stack to complete before starting EC2 stack
      - IAMStack     # wait for IAM stack to complete (need profile name)
    Properties:
      TemplateURL: https://YOUR-BUCKET.s3.us-east-1.amazonaws.com/ec2-stack.yaml
      TimeoutInMinutes: 15
      Parameters:
        # !GetAtt StackLogicalId.Outputs.OutputKey — reads an output from a nested stack
        VPCId: !GetAtt VPCStack.Outputs.VPCId
        SubnetId: !GetAtt VPCStack.Outputs.SubnetId
        InstanceProfileName: !GetAtt IAMStack.Outputs.InstanceProfileName

Outputs:
  WebsiteURL:
    Description: URL of the deployed web server
    Value: !GetAtt EC2Stack.Outputs.WebsiteURL

  EC2PublicIP:
    Description: EC2 Public IP
    Value: !GetAtt EC2Stack.Outputs.EC2PublicIP
```

> Replace `YOUR-BUCKET` with the actual bucket name you create in Step 1 below.

---

## 🔧 PART 3 — AWS SETUP (Step by Step)

### STEP 1: Create an S3 Bucket for templates

The child template YAML files must be in S3 because CloudFormation fetches them by URL at deploy time. Your local machine cannot serve them.

1. Console → **S3** → **Create bucket**

| Option | Value | Explanation |
|---|---|---|
| **Bucket name** | `my-cfn-templates-[yourname]` e.g. `my-cfn-templates-shivam` | Must be globally unique. This is where you store the YAML files. |
| **Region** | **`us-east-1`** | CloudFormation and the bucket must be in the same region — cross-region TemplateURL works but is slower |
| **Block Public Access** | Keep ALL blocked | CloudFormation accesses templates using its IAM role, not public HTTP |
| **Bucket Versioning** | Enable (recommended) | If you update a template and redeploy, versioning keeps old versions |
| **Encryption** | SSE-S3 (default) | Fine for templates |

2. Click **Create bucket**

---

### STEP 2: Upload templates to S3

#### Option A — AWS Console (easiest for exam)

1. Click your bucket → **Upload** → **Add files**
2. Select all 4 files: `vpc-stack.yaml`, `iam-stack.yaml`, `ec2-stack.yaml`, `root-stack.yaml`
3. Click **Upload**
4. After upload, click each file → copy the **Object URL** (looks like `https://my-cfn-templates-shivam.s3.amazonaws.com/vpc-stack.yaml`)

#### Option B — AWS CLI

```bash
# Create the bucket
aws s3 mb s3://my-cfn-templates-shivam --region us-east-1

# Upload all 4 files
aws s3 cp vpc-stack.yaml  s3://my-cfn-templates-shivam/
aws s3 cp iam-stack.yaml  s3://my-cfn-templates-shivam/
aws s3 cp ec2-stack.yaml  s3://my-cfn-templates-shivam/
aws s3 cp root-stack.yaml s3://my-cfn-templates-shivam/
```

---

### STEP 3: Update `root-stack.yaml` with your bucket name

Before uploading `root-stack.yaml`, replace every `YOUR-BUCKET` with your actual bucket name:

```yaml
# Change this:
TemplateURL: https://YOUR-BUCKET.s3.us-east-1.amazonaws.com/vpc-stack.yaml

# To this:
TemplateURL: https://my-cfn-templates-shivam.s3.us-east-1.amazonaws.com/vpc-stack.yaml
```

Do the same for `iam-stack.yaml` and `ec2-stack.yaml` URLs. Then re-upload `root-stack.yaml`.

---

### STEP 4: Deploy via CloudFormation Console

1. Console → **CloudFormation** → **Create stack** → **With new resources (standard)**

**Every option explained:**

| Option | Value | Explanation |
|---|---|---|
| **Prepare template** | `Template is ready` | We have the template ready. Alternative: `Use a sample template` (AWS examples) or `Create template in Designer` (drag-drop visual editor) |
| **Template source** | `Amazon S3 URL` | Our templates are in S3. Alternative: `Upload a template file` — only works for the root; child stacks must always be S3 URLs |
| **Amazon S3 URL** | `https://my-cfn-templates-shivam.s3.us-east-1.amazonaws.com/root-stack.yaml` | URL of the ROOT template only. CloudFormation reads the root, then fetches children by the TemplateURL in the root |

2. Click **Next**

| Option | Value | Explanation |
|---|---|---|
| **Stack name** | `RootStack` | The logical name for this deployment. Child stacks are named automatically: `RootStack-VPCStack-XXXX` |

3. Click **Next** (no parameters — root stack has none)

**Configure stack options:**

| Option | Value | Explanation |
|---|---|---|
| **Tags** | Optional — add `Project: NestedDemo` | Tags apply to the root stack; child stacks inherit them |
| **Permissions** | Leave blank (use your IAM user) | You could specify a role for CloudFormation to use |
| **Stack failure options** | `Roll back all stack resources` | If anything fails, undo everything. Alternative: `Preserve successfully provisioned resources` — useful for debugging (leaves partial stacks) |
| **Advanced — Termination protection** | Disable for lab | Prevents accidental deletion. Enable for production |
| **Advanced — Stack policy** | None for lab | JSON policy that prevents specific resources from being updated |
| **Advanced — Notification options** | None for lab | SNS topic to notify on stack events |

4. Click **Next** → Review page

> ⚠️ At the bottom of the review page you will see:
> **"I acknowledge that AWS CloudFormation might create IAM resources with custom names."**
> **You must check this box.** Without it, the IAM stack fails because CloudFormation refuses to create IAM roles silently.

5. Click **Create stack**

---

### STEP 5: Watch the deployment

1. CloudFormation → Stacks → **RootStack** → **Events** tab

You will see events in this order:
```
RootStack                  CREATE_IN_PROGRESS
RootStack-VPCStack-XXXX    CREATE_IN_PROGRESS
RootStack-IAMStack-XXXX    CREATE_IN_PROGRESS
  (VPC and IAM create in parallel)
RootStack-VPCStack-XXXX    CREATE_COMPLETE
RootStack-IAMStack-XXXX    CREATE_COMPLETE
RootStack-EC2Stack-XXXX    CREATE_IN_PROGRESS
  (EC2 stack starts only after both finish — DependsOn)
RootStack-EC2Stack-XXXX    CREATE_COMPLETE
RootStack                  CREATE_COMPLETE
```

Total time: ~3–5 minutes.

2. Click the **Outputs** tab of **RootStack** — you will see:

| Key | Value |
|---|---|
| `WebsiteURL` | `http://3.92.X.X` |
| `EC2PublicIP` | `3.92.X.X` |

3. Open `http://YOUR_EC2_PUBLIC_IP` in your browser → see the HTML page.

> **Note:** Wait 60–90 seconds after the stack shows `CREATE_COMPLETE` for UserData (Apache install) to finish.

---

## 🧪 PART 4 — VERIFICATION CHECKLIST

### Console verification

| What to check | Where in console |
|---|---|
| Root stack status | CloudFormation → Stacks → RootStack → **Status: CREATE_COMPLETE** |
| All 3 child stacks | CloudFormation → Stacks → filter by `RootStack` — should see 4 stacks total |
| VPC created | VPC → Your VPCs → `NestedVPC` |
| Subnet created | VPC → Subnets → `NestedPublicSubnet` |
| IGW attached | VPC → Internet Gateways → `NestedIGW` → State: Attached |
| IAM role created | IAM → Roles → `EC2SSMRole` |
| Instance profile created | IAM → Roles → `EC2SSMRole` → Instance profiles tab |
| EC2 running | EC2 → Instances → `NestedEC2WebServer` → State: Running |
| Stack outputs | CloudFormation → RootStack → **Outputs** tab → WebsiteURL visible |

### Browser test

Open `http://YOUR_EC2_PUBLIC_IP` — you should see:

```
Hello from CloudFormation Nested Stack!

Stack:         RootStack-EC2Stack-XXXX
Region:        us-east-1
Instance ID:   i-0abc123def456789
Availability Zone: us-east-1a

Deployed via: Root Stack → VPC Stack + IAM Stack + EC2 Stack
```

### SSH test (optional)

```bash
# EC2 uses AmazonLinux2023 — default user is ec2-user
ssh -i your-keypair.pem ec2-user@YOUR_EC2_PUBLIC_IP

# Check Apache is running
sudo systemctl status httpd

# Check the HTML file
cat /var/www/html/index.html
```

> Note: The templates don't add a KeyPair to the EC2 instance — if you want SSH access add `KeyName: your-keypair` under EC2 Properties. SSM Session Manager works without it (the IAM role has `AmazonSSMManagedInstanceCore`). For the practical, browser test is sufficient.

---

## 🔵 PART 5 — CONCEPTS DEEP DIVE (for viva / edge case questions)

### What is a Nested Stack?

A nested stack is a CloudFormation stack created by another CloudFormation stack — using the resource type `AWS::CloudFormation::Stack`. The stack that creates others is called the **root** or **parent**. The stacks it creates are **children** or **nested stacks**.

```
Without nesting: one huge template with 50+ resources — hard to manage
With nesting:    root template + small focused child templates — reusable, maintainable
```

> **Viva:** "A nested stack is a stack resource inside another stack. The parent uses `AWS::CloudFormation::Stack` with a `TemplateURL` pointing to an S3 template. The parent passes Parameters to children and reads their Outputs using `!GetAtt ChildStack.Outputs.Key`."

---

### How outputs pass between stacks (two methods)

**Method 1 — Parent passes via Parameters (what we use)**
```yaml
# In root-stack.yaml:
EC2Stack:
  Properties:
    Parameters:
      VPCId: !GetAtt VPCStack.Outputs.VPCId   # read VPC stack output, pass to EC2 stack
```
Root stack is the intermediary. Child VPC exports an Output. Root reads it with `!GetAtt`. Root passes it to EC2 stack as a Parameter.

**Method 2 — Cross-Stack Reference with `Fn::ImportValue`**
```yaml
# In any separate stack (not nested):
SubnetId: !ImportValue NestedSubnetId   # reads the Export from vpc-stack
```
This is for **completely separate** stacks (not parent-child). VPC stack exports with `Export: Name: NestedSubnetId`. Any other stack in the same region can `!ImportValue NestedSubnetId`.

> **Viva:** "In nested stacks, the parent reads child outputs with `!GetAtt`. In completely separate stacks, you use `Fn::ImportValue` to consume an `Export`. They are different mechanisms."

**Important constraint on Exports:** You **cannot delete a stack** that has Outputs currently being imported by another stack via `Fn::ImportValue`. CloudFormation refuses the delete.

---

### DependsOn vs implicit dependency

**Implicit dependency (CloudFormation auto-detects):**
```yaml
MyRoute:
  DependsOn: AttachIGW     # explicit, needed here
  Properties:
    GatewayId: !Ref MyIGW  # this is an implicit dependency — CloudFormation sees !Ref MyIGW
```

When you use `!Ref` or `!GetAtt` on a resource, CloudFormation automatically knows it depends on that resource and creates it first. **You do not need `DependsOn` for these.**

**When you DO need explicit `DependsOn`:**
- When a dependency exists but is NOT expressed through `!Ref`/`!GetAtt`
- Example: `MyRoute` needs `AttachIGW` to complete (IGW attached to VPC) but it references `!Ref MyIGW` (not `AttachIGW`) — so we need explicit `DependsOn: AttachIGW`
- Example: EC2Stack needs VPCStack and IAMStack done first — the relationship goes through parameters, so CloudFormation would detect it, but being explicit with `DependsOn` makes intent clear

> **Viva edge case:** "If EC2Stack passes `!GetAtt VPCStack.Outputs.VPCId`, does it need `DependsOn`?" — Answer: No, CloudFormation infers the dependency from `!GetAtt`. But `DependsOn` is harmless and documents intent explicitly.

---

### `!Ref` vs `!GetAtt` vs `!Sub`

| Function | What it returns | Example |
|---|---|---|
| `!Ref ResourceName` | The **primary identifier** of a resource (depends on type) | `!Ref MyVPC` → VPC ID like `vpc-0abc123` |
| `!Ref ParameterName` | The **value** of a parameter | `!Ref VPCId` → whatever string was passed |
| `!GetAtt ResourceName.Attribute` | A **specific attribute** of a resource | `!GetAtt MyEC2.PublicIp` → `3.92.X.X` |
| `!GetAtt NestedStack.Outputs.Key` | An **output** from a nested stack | `!GetAtt VPCStack.Outputs.VPCId` |
| `!Sub 'string ${Variable}'` | String with **substitutions** | `!Sub 'http://${MyEC2.PublicIp}'` |

**What `!Ref` returns for common types:**

| Resource type | `!Ref` returns |
|---|---|
| `AWS::EC2::VPC` | VPC ID (`vpc-xxx`) |
| `AWS::EC2::Subnet` | Subnet ID (`subnet-xxx`) |
| `AWS::EC2::SecurityGroup` | Security Group ID (`sg-xxx`) |
| `AWS::IAM::Role` | Role name (string) |
| `AWS::IAM::InstanceProfile` | Instance profile name (string) |
| `AWS::CloudFormation::Stack` | Stack ID (ARN) |

---

### `Fn::Base64` and UserData

UserData is the script that runs when EC2 first boots (runs as root, once, at first launch only). EC2 expects it base64-encoded. CloudFormation's `Fn::Base64` handles the encoding:

```yaml
UserData:
  Fn::Base64: !Sub |   # !Sub lets you use ${AWS::StackName} etc inside the script
    #!/bin/bash
    yum install -y httpd
```

`!Sub |` with the pipe (`|`) starts a multi-line literal block. Variables like `${AWS::StackName}` are CloudFormation pseudo-parameters substituted at deploy time. Variables like `$INSTANCE_ID` (shell variables you set in the script) are left alone by CloudFormation and resolved at runtime on the EC2.

> **Viva:** "UserData only runs on first boot. If you need it to run again, you either terminate and relaunch, or use `cfn-init` with a `Metadata` section and `cfn-hup` for updates. For this practical, first boot is sufficient."

---

### IMDSv2 — why we use it for metadata

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
```

`169.254.169.254` is the **Instance Metadata Service (IMDS)** — a link-local address accessible only from within EC2.

- **IMDSv1**: Simple GET request — no token needed but vulnerable to SSRF attacks.
- **IMDSv2**: Session-based — first GET a token (PUT request), then use that token in subsequent requests. Secure by default on Amazon Linux 2023.

> **Viva:** "IMDSv2 requires a PUT to get a session token before querying metadata. This prevents server-side request forgery (SSRF) attacks where a malicious redirect could steal instance credentials."

---

### IAM Role vs Instance Profile

| Concept | What it is |
|---|---|
| **IAM Role** | A set of permissions. You define what actions it allows. |
| **Instance Profile** | A **wrapper** that attaches an IAM Role to an EC2 instance. |
| **Why the wrapper?** | EC2 cannot directly use an IAM Role — it needs the profile as the attachment point |

```
EC2 Instance → has → Instance Profile → contains → IAM Role → has → Policies
```

When CloudFormation creates the role for you (`AWS::IAM::Role`) and then the profile (`AWS::IAM::InstanceProfile`), both get a name. The `IamInstanceProfile` property on `AWS::EC2::Instance` takes the **profile name** (from `!Ref EC2InstanceProfile`), not the role name.

> **Viva:** "You attach an Instance Profile to EC2, not a Role directly. In CloudFormation, `AWS::IAM::InstanceProfile` wraps the role, and `IamInstanceProfile` on the EC2 resource references that profile's name."

---

### CloudFormation Stack states

| Status | Meaning |
|---|---|
| `CREATE_IN_PROGRESS` | Currently creating resources |
| `CREATE_COMPLETE` | All resources created successfully |
| `CREATE_FAILED` | At least one resource failed — stack may be rolled back |
| `ROLLBACK_IN_PROGRESS` | A failure triggered rollback — deleting created resources |
| `ROLLBACK_COMPLETE` | Rollback finished — stack exists but is empty |
| `DELETE_IN_PROGRESS` | Stack deletion in progress |
| `DELETE_COMPLETE` | Stack deleted (visible in console briefly) |
| `UPDATE_IN_PROGRESS` | Stack update running |
| `UPDATE_ROLLBACK_COMPLETE` | Update failed and rolled back to previous state |

> **Viva:** "`ROLLBACK_COMPLETE` means the stack exists but has no resources — it is in a failed state. You cannot update it. You must delete it and redeploy."

---

### Dynamic AMI reference — why not hardcode AMI IDs

Bad practice:
```yaml
ImageId: ami-0c02fb55956c7d316   # This was Amazon Linux 2 AMI in 2023; may be deregistered
```

Good practice:
```yaml
ImageId: '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64}}'
```

`{{resolve:ssm:...}}` is a **CloudFormation dynamic reference**. At deploy time, CloudFormation calls SSM Parameter Store to get the current value. AWS maintains `/aws/service/ami-amazon-linux-latest/...` and updates it when new AMIs are released. Your template always gets the latest AMI without any changes.

> **Viva:** "Hardcoded AMI IDs break when the AMI is deregistered. Dynamic SSM references (`{{resolve:ssm:...}}`) always resolve to the current latest value at stack create/update time."

---

### TemplateURL format

Two valid formats:

```
https://s3.amazonaws.com/BUCKET/KEY               ← path-style (older, works but deprecated)
https://BUCKET.s3.us-east-1.amazonaws.com/KEY     ← virtual-hosted + regional (recommended)
```

Use the **regional virtual-hosted** format. It is faster (routes to the correct region directly) and is the format AWS recommends going forward.

---

### What `ExpressionAttributeNames` is for (NOT this practical — but if asked)

This is a DynamoDB concept from the Lambda practical, not CloudFormation. If an interviewer mixes them up: in CloudFormation, reserved words are not an issue in YAML — they are in DynamoDB update expressions.

---

## 🧠 PART 6 — VIVA QUESTIONS + ANSWERS

**Q: What is a nested stack and why use it?**
> A nested stack is a `AWS::CloudFormation::Stack` resource inside a parent stack's template. It creates a completely independent child stack. Benefits: (1) Reusability — the same child template can be used by multiple parents. (2) Modularity — VPC, IAM, EC2 are separate concerns in separate files. (3) Limits — a single CloudFormation template has a 500-resource limit; splitting into nested stacks effectively removes this limit. (4) Team ownership — different teams manage different child stacks.

**Q: How does the EC2 stack get the VPC ID? Walk through it.**
> VPC child stack creates the VPC and has an `Outputs` section that exposes `VPCId: !Ref MyVPC`. The root stack reads this with `!GetAtt VPCStack.Outputs.VPCId`. The root stack then passes this value as a `Parameter` to the EC2 stack (`VPCId: !GetAtt VPCStack.Outputs.VPCId` under EC2Stack's `Parameters`). Inside `ec2-stack.yaml`, a `Parameters` block declares `VPCId: Type: String`, and the security group uses `VpcId: !Ref VPCId`. So the value travels: VPC resource → VPC stack Output → Root stack reads with GetAtt → Root passes as Parameter → EC2 stack reads with Ref.

**Q: What is the difference between `DependsOn` and implicit dependency?**
> Implicit dependency: when you use `!Ref` or `!GetAtt` on another resource, CloudFormation automatically creates that resource first. Explicit `DependsOn`: when a dependency exists but is not expressed through a reference. Example: `MyRoute` depends on `AttachIGW` being complete (so the IGW is attached before the route tries to use it), but `MyRoute` references `!Ref MyIGW` not `!Ref AttachIGW`, so CloudFormation would not know — hence explicit `DependsOn: AttachIGW`.

**Q: Why can't EC2 directly use an IAM Role? Why the Instance Profile?**
> IAM Roles can be assumed by many entity types (Lambda, ECS, cross-account). The Instance Profile is the **EC2-specific attachment mechanism** — it creates a link between the running instance and the role. When EC2 boots, it calls the metadata service to get temporary credentials, which are scoped to the role inside the profile. Without the profile, there is no way to attach a role to EC2.

**Q: What happens if you delete the VPC child stack while the EC2 child stack still exists?**
> The delete will fail. EC2 instances, security groups, and subnets are still in the VPC — CloudFormation (and AWS) cannot delete a VPC that has dependent resources. You must delete stacks in reverse creation order: EC2 stack first, then VPC and IAM stacks, then root.

**Q: What is `MapPublicIpOnLaunch: true` on the subnet?**
> It automatically assigns a public IPv4 address to any EC2 instance launched in this subnet. Without it, the instance gets only a private IP and cannot be reached from the internet even if the security group and route table are correct.

**Q: What is `EnableDnsHostnames: true` on the VPC?**
> Assigns public DNS hostnames to instances with public IPs (e.g. `ec2-3-92-X-X.compute-1.amazonaws.com`). Required for some AWS services (like RDS) and for SSM Session Manager to work correctly. `EnableDnsSupport: true` enables the Route 53 resolver in the VPC — DNS queries return the right IPs.

**Q: What is the `ConditionExpression` difference between CloudFormation and DynamoDB?**
> They are completely different things with the same name. In DynamoDB, `ConditionExpression` is a guard on write operations (e.g. only update if item exists). In CloudFormation, a `Condition` is a template-level feature that controls whether resources are created based on parameter values. No relation.

**Q: Can you update a nested stack independently without updating the root?**
> Yes. You can go to CloudFormation → find the child stack → click **Update** → provide the new template. However, this can cause **drift** — the root stack's internal record of the child stack may no longer match reality. Best practice: always update through the root stack so the dependency chain is re-evaluated.

**Q: What is stack drift?**
> Drift is when the actual state of a resource no longer matches what CloudFormation's template says it should be. Example: someone manually edits a security group rule in the console after CloudFormation created it. CloudFormation → select stack → **Actions** → **Detect drift** shows which resources have drifted and what changed.

**Q: What is a `Change Set` in CloudFormation?**
> A preview of what CloudFormation will do before it actually does it. You create a change set (propose an update), review which resources will be Added / Modified / Removed, and then choose to Execute or Discard. Prevents surprises in production — you can see "this update will replace the EC2 instance" before committing.

**Q: What is `{{resolve:ssm:...}}` and why is it better than hardcoding AMI IDs?**
> It is a CloudFormation dynamic reference that calls SSM Parameter Store at deploy time to get the current value. AWS maintains a public SSM path `/aws/service/ami-amazon-linux-latest/...` that always points to the latest AMI. Hardcoded IDs break when the AMI is deregistered; dynamic references always get the current version without any template changes.

**Q: Why must child template YAML files be in S3?**
> CloudFormation's control plane (which runs in AWS data centers, not your browser) needs to fetch and parse the child templates. It cannot reach your local machine. S3 is AWS-native object storage that CloudFormation can access with its own IAM permissions. This is why `TemplateURL` must be an S3 URL, not a local file path.

**Q: What is the 500-resource limit in CloudFormation?**
> A single CloudFormation template can define at most 500 resources. Nested stacks bypass this because each child stack has its own 500-resource limit. A root stack with 3 children can effectively have 4 × 500 = 2000 resources total.

---

## ⚡ PART 7 — COMMON ERRORS AND FIXES

| Error | Cause | Fix |
|---|---|---|
| `Template format error: unsupported structure` | YAML indentation wrong — tabs instead of spaces, or wrong nesting | Use 2-space indentation; no tabs; validate with online YAML linter |
| `Stack is in ROLLBACK_COMPLETE state` | A previous deploy failed and was rolled back | Delete the failed stack (CloudFormation → Delete), then redeploy |
| `IAM resource names with custom names — requires capability` | Creating IAM resources with explicit names (like `RoleName: EC2SSMRole`) without checking the IAM acknowledgement | Check the **"I acknowledge..."** box on the Review page |
| `Template URL must reference a valid S3 object` | `TemplateURL` in root stack still has `YOUR-BUCKET` or wrong bucket name | Update `root-stack.yaml` with real bucket name, re-upload to S3 |
| `Resource of type AWS::IAM::Role already exists` | Previous failed deploy left the IAM role behind (not cleaned up by rollback) | IAM → Roles → manually delete `EC2SSMRole`, then redeploy |
| `No export named NestedVPCId found` | Using `!ImportValue` but the VPC stack isn't deployed or was deployed in a different region | For nested stacks, use `!GetAtt` not `!ImportValue`; exports are only for separate stacks |
| `Instance unreachable — browser times out` | (a) UserData still running, (b) Security group missing port 80, (c) subnet not public | Wait 2 min; check SG has port 80; check `MapPublicIpOnLaunch: true` and route table |
| `ConditionalCheckFailedException` | Wrong practical — DynamoDB, not CloudFormation | |
| `TemplateURL must be in the same region` | Bucket is in `us-west-2` but stack is being created in `us-east-1` | Ensure bucket region matches CloudFormation region (`us-east-1`) |

---

## ✅ PART 8 — FINAL SUBMISSION CHECKLIST

```
□ S3 bucket created in us-east-1
□ All 4 YAML files uploaded to S3
□ YOUR-BUCKET replaced with actual bucket name in root-stack.yaml
□ CloudFormation → RootStack → Status: CREATE_COMPLETE
□ 4 stacks visible (root + 3 children)
□ VPC → NestedVPC exists
□ VPC → Subnet → NestedPublicSubnet with MapPublicIpOnLaunch
□ VPC → Internet Gateway attached to NestedVPC
□ IAM → Role → EC2SSMRole with AmazonSSMManagedInstanceCore
□ EC2 → NestedEC2WebServer → State: Running
□ CloudFormation → RootStack → Outputs → WebsiteURL shown
□ Browser: http://PUBLIC_IP → HTML page loads
□ HTML page shows Stack name, Region, Instance ID
□ Can explain: how VPCId travels from vpc-stack → root → ec2-stack
```

---

That's the complete CloudFormation Nested Stacks guide. One `root-stack.yaml` deployed on the console creates all three child stacks in the right order, passes values between them, and boots an EC2 with a visible web page.
