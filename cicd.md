# Complete AWS CI/CD Pipeline — Full Step-by-Step Guide

I'll walk you through **everything** — every file, every console option, every button click. Read this top to bottom and do it side by side.

---

## 🗂️ PART 0 — WHAT WE ARE BUILDING (Big Picture)

**Region:** Create every AWS resource in **`us-east-1`** (EC2, S3, CodeBuild, CodePipeline, CodeDeploy, CodeStar Connection).

```
Your GitHub Repo
      │
      │  (you only `git push` here)
      ▼
  CodeStar Connection  ──(change detection)──▶  CodePipeline
                              │
                              ┌────────────┼────────────┐
                              ▼            ▼            ▼
                           STAGE 1      STAGE 2      STAGE 3
                           Source       Build        Deploy
                         (GitHub)    (CodeBuild)  (CodeDeploy)
                                          │            │
                                          ▼            ▼
                                      S3 Bucket     EC2 Server
                                    (artifact zip) (your app runs here)
```

---

## 🗂️ PART 1 — YOUR PROJECT FOLDER STRUCTURE

Create this exact folder on your laptop:

```
my-node-app/
├── app.js                  ← your Node.js app
├── package.json            ← Node dependencies
├── buildspec.yml           ← tells CodeBuild what to do
├── appspec.yml             ← tells CodeDeploy how to deploy
└── scripts/
    ├── install_dependencies.sh
    ├── start_server.sh
    └── stop_server.sh
```

---

## 📄 ALL FILES — COPY-PASTE READY

### FILE 1: `app.js`
```javascript
const http = require('http');

const hostname = '0.0.0.0';
const port = 3000;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/html');
  res.end('<h1>Hello from Blue/Green Deployment! Version 1.0</h1>');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

---

### FILE 2: `package.json`
```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "description": "Demo app for AWS CI/CD pipeline",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"No tests yet\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "engines": {
    "node": ">=16.0.0"
  }
}
```

---

### FILE 3: `buildspec.yml` ← **MOST IMPORTANT FILE**
```yaml
version: 0.2

# 'version: 0.2' is the current buildspec version. Always use 0.2.

phases:
  install:
    # This phase runs first. Used to install tools, runtimes, dependencies.
    runtime-versions:
      nodejs: 18          # Tells CodeBuild to use Node.js version 18
    commands:
      - echo "=== INSTALL PHASE STARTED ==="
      - echo "Installing Node dependencies..."
      - npm install       # installs everything in package.json

  pre_build:
    # Runs BEFORE the actual build. Good for: login to ECR, run tests, set env vars
    commands:
      - echo "=== PRE-BUILD PHASE ==="
      - echo "Running tests..."
      - npm test          # runs your test script from package.json

  build:
    # THE MAIN BUILD PHASE. This is where you compile, bundle, zip your app.
    commands:
      - echo "=== BUILD PHASE ==="
      - echo "Build started on `date`"
      - echo "Build ID is $CODEBUILD_BUILD_NUMBER"
      - zip -r app-$CODEBUILD_BUILD_NUMBER.zip . -x "*.git*"
      # zip -r  → recursive zip (include all subfolders)
      # app-$CODEBUILD_BUILD_NUMBER.zip → unique name per build (e.g. app-5.zip)
      # . → zip everything in current folder
      # -x "*.git*" → EXCLUDE the .git folder (we don't need git history in zip)

  post_build:
    # Runs AFTER build. Good for: push Docker images, send notifications, cleanup
    commands:
      - echo "=== POST-BUILD PHASE ==="
      - echo "Build completed on `date`"

artifacts:
  # Artifacts = the OUTPUT of your build. What gets uploaded to S3.
  files:
    - '**/*'              # Upload ALL files and folders recursively
    # You can also be specific: just upload the zip:
    # - app-*.zip
  name: app-$CODEBUILD_BUILD_NUMBER
  # This sets the folder name inside your S3 bucket

cache:
  # Optional: Cache node_modules so next build is faster
  paths:
    - 'node_modules/**/*'
```

**Every `$CODEBUILD_*` variable explained:**

| Variable | What it means |
|---|---|
| `$CODEBUILD_BUILD_NUMBER` | Auto-incrementing build number (1, 2, 3...) |
| `$CODEBUILD_BUILD_ID` | Full build ID like `my-project:abc123` |
| `$CODEBUILD_SOURCE_VERSION` | Git commit hash that triggered this build |
| `$CODEBUILD_BUILD_SUCCEEDING` | `1` if build is passing so far, `0` if failing |
| `$AWS_DEFAULT_REGION` | The AWS region you're in |
| `$CODEBUILD_RESOLVED_SOURCE_VERSION` | Resolved Git commit hash |

---

### FILE 4: `appspec.yml` ← **SECOND MOST IMPORTANT**
```yaml
version: 0.0
# version 0.0 is the ONLY valid value for EC2/on-premises deployments. Don't change it.

os: linux
# 'linux' or 'windows' — the OS of your EC2 instance

files:
  # This section says: copy files FROM the artifact TO the server
  - source: /
    # source: / means "everything in the root of my artifact zip"
    destination: /var/www/html
    # destination: where on the EC2 server to put the files
    # Common destinations:
    # /var/www/html → for web apps (Apache/Nginx)
    # /home/ec2-user/app → for Node apps
    # /opt/myapp → custom location

hooks:
  # Hooks = scripts that run at specific points during deployment
  # ORDER OF HOOKS (top to bottom, this is the lifecycle):

  ApplicationStop:
    # Runs FIRST — stop the currently running old application
    - location: scripts/stop_server.sh
      timeout: 300      # max seconds this script can run (300 = 5 minutes)
      runas: root       # run the script as this user

  BeforeInstall:
    # Runs BEFORE files are copied to server
    # Good for: cleanup old files, create directories, install system packages
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root

  AfterInstall:
    # Runs AFTER files are copied to server
    # Good for: set permissions, configure the app, set env variables
    - location: scripts/start_server.sh
      timeout: 300
      runas: root

  ApplicationStart:
    # Runs to START your new application
    - location: scripts/start_server.sh
      timeout: 300
      runas: root

  ValidateService:
    # Runs LAST — verify that deployment was successful
    # If this script exits with non-zero, CodeDeploy marks deployment as FAILED
    - location: scripts/validate_service.sh
      timeout: 300
      runas: root
```

**Full CodeDeploy Lifecycle Hook Order (for viva):**
```
1. ApplicationStop          ← stop old app
2. DownloadBundle           ← CodeDeploy downloads artifact from S3 (automatic)
3. BeforeInstall            ← pre-copy setup
4. Install                  ← CodeDeploy copies files (automatic)
5. AfterInstall             ← post-copy setup
6. ApplicationStart         ← start new app
7. ValidateService          ← confirm it's working
```

---

### FILE 5: `scripts/stop_server.sh`
```bash
#!/bin/bash
# This script stops the currently running Node.js server

echo "=== STOPPING SERVER ==="

# Check if node is running and kill it
if pgrep -x "node" > /dev/null
then
    echo "Node process found. Killing it..."
    pkill -f "node app.js"
    echo "Server stopped."
else
    echo "No Node process running. Nothing to stop."
fi

# Exit 0 = success (important! CodeDeploy checks exit code)
exit 0
```

---

### FILE 6: `scripts/install_dependencies.sh`
```bash
#!/bin/bash
# Runs before files are copied. Install system-level stuff here.

echo "=== INSTALLING DEPENDENCIES ==="

# Update package manager
yum update -y          # Amazon Linux uses yum
# apt-get update -y   # Ubuntu uses apt-get

# Install Node.js if not present
if ! command -v node &> /dev/null
then
    echo "Node.js not found. Installing..."
    curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
    yum install -y nodejs
fi

# Create app directory if it doesn't exist
mkdir -p /var/www/html

echo "Dependencies installed successfully."
exit 0
```

---

### FILE 7: `scripts/start_server.sh`
```bash
#!/bin/bash
# This script starts your Node.js application

echo "=== STARTING SERVER ==="

cd /var/www/html

# Install npm packages on the server
npm install

# Start the server in background using nohup
# nohup = "no hang up" — keeps process running after SSH disconnects
# & = run in background
# > /var/log/app.log = redirect output to log file
# 2>&1 = redirect error output to same log file
nohup node app.js > /var/log/app.log 2>&1 &

echo "Server started. Check /var/log/app.log for output."
exit 0
```

---

### FILE 8: `scripts/validate_service.sh`
```bash
#!/bin/bash
# Validates that the app is actually running after deployment

echo "=== VALIDATING SERVICE ==="

# Wait 5 seconds for server to fully start
sleep 5

# Check if the server responds on port 3000
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000)

if [ "$response" = "200" ]; then
    echo "Validation PASSED. Server returned HTTP 200."
    exit 0
else
    echo "Validation FAILED. Server returned HTTP $response"
    exit 1   # Non-zero exit = CodeDeploy marks deployment as FAILED → triggers rollback
fi
```

---

## 🔧 PART 2 — AWS SETUP (Step by Step on Console)

### STEP 1: Create an IAM Role for EC2

> Why? Your EC2 needs permission to pull from S3 (to get the artifact from CodeDeploy)

1. Go to **IAM** → **Roles** → **Create Role**
2. **Trusted entity type**: `AWS Service`
3. **Use case**: `EC2`
4. Click **Next**
5. Search and attach these policies:
   - `AmazonEC2RoleforAWSCodeDeploy` ← lets EC2 talk to CodeDeploy
   - `AmazonS3ReadOnlyAccess` ← lets EC2 read artifacts from S3
   - `CloudWatchAgentServerPolicy` ← lets EC2 send logs to CloudWatch
6. **Role name**: `EC2-CodeDeploy-Role`
7. Click **Create Role**

---

### STEP 2: Launch an EC2 Instance

1. Go to **EC2** → **Launch Instance**

**All options explained:**

| Option | What to choose | Why |
|---|---|---|
| **Name** | `my-app-server` | just a label |
| **AMI** | Amazon Linux 2023 | CodeDeploy agent works best here |
| **Instance type** | `t2.micro` | free tier |
| **Key pair** | Create new → `my-keypair` | to SSH into the server |
| **Network** | Default VPC | fine for lab |
| **Auto-assign public IP** | Enable | so you can reach it from browser |
| **Security Group** | Create new | controls who can connect |

**Security Group Rules** (add these):
| Type | Port | Source | Why |
|---|---|---|---|
| SSH | 22 | My IP | So you can SSH in |
| HTTP | 80 | 0.0.0.0/0 | Public web access |
| Custom TCP | 3000 | 0.0.0.0/0 | Your Node app's port |

**Advanced Details → IAM Instance Profile**: Select `EC2-CodeDeploy-Role` you created in Step 1

**User Data** (paste this — it auto-installs CodeDeploy agent when EC2 starts):
```bash
#!/bin/bash
yum update -y
yum install -y ruby wget

# Install CodeDeploy Agent
cd /home/ec2-user
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x ./install
./install auto

# Start CodeDeploy Agent
service codedeploy-agent start

# Install Node.js
curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
yum install -y nodejs

echo "Setup complete"
```
> This guide uses **`us-east-1`** everywhere. The `wget` URL must match the region where your EC2 and CodeDeploy run.

2. Click **Launch Instance**
3. Note down the **Public IP** of your EC2 instance

---

### STEP 3: Create S3 Bucket (for artifacts)

1. Go to **S3** → **Create Bucket**

**All options explained:**

| Option | What to set | Explanation |
|---|---|---|
| **Bucket name** | `my-app-artifacts-[yourname]` | Must be globally unique. Add your name to make it unique |
| **Region** | `us-east-1` (same as everything else in this guide) | Keep all services in same region |
| **Object Ownership** | ACLs disabled | Recommended. Bucket owner owns all objects |
| **Block Public Access** | Keep ALL blocked | Artifacts are private — only CodeBuild/CodeDeploy access them via IAM |
| **Bucket Versioning** | **ENABLE THIS** | Keeps history of every artifact uploaded. If new build breaks, you can grab old version |
| **Default encryption** | SSE-S3 | Encrypts your artifacts at rest automatically |
| **Object Lock** | Disable | Not needed for CI/CD |

2. Click **Create Bucket**

**Now add Lifecycle Rule** (to auto-delete old artifacts):
1. Click your bucket → **Management** tab → **Create lifecycle rule**
2. **Rule name**: `delete-old-artifacts`
3. **Rule scope**: Apply to all objects
4. **Lifecycle rule actions**: Check `Expire current versions of objects`
5. **Days after object creation**: `30`
   > This means: any artifact older than 30 days gets auto-deleted. Saves money.
6. Also check `Delete expired object delete markers` — cleans up empty tombstones
7. Click **Create Rule**

---

### STEP 4: Connect GitHub with AWS CodeStar Connections

> **This guide uses GitHub as the only Git remote.** CodePipeline and CodeBuild pull code through a **CodeStar Connection**. There is **no CodeCommit** and no mirror workflow.

Do this **before** you create the CodeBuild project (**Step 5**) and the pipeline (**Step 7**), so you can pick the same connection everywhere.

1. **Create a GitHub repository** (if you have not already) and push your project files from Part 1 — at minimum `app.js`, `package.json`, `buildspec.yml`, `appspec.yml`, and the `scripts/` folder — on branch **`main`**.

2. In the AWS console, open **Developer Tools** → **Settings** → **Connections** (also listed under **CodePipeline** / **CodeStar** depending on UI version).

3. Click **Create connection**.

4. **Provider**: **GitHub**.

5. **Connection name**: e.g. `github-my-node-app` (any logical name).

6. Finish the wizard: you will be redirected to **GitHub** to **authorize AWS Connector for GitHub** (or similar). Approve access for the account/org that owns your repo.

7. Wait until the connection status is **Available** (green). If it stays **Pending**, complete authorization from the connection detail page.

**Keep these aligned across Steps 5 and 7:**

| Item | Value |
|---|---|
| **AWS Region** | **`us-east-1`** for this entire guide |
| **Git branch** | **`main`** (must match pipeline + CodeBuild) |
| **Repo** | Same GitHub repo you select in CodeBuild and in the pipeline source stage |

---

### STEP 5: Create CodeBuild Project

1. Go to **CodeBuild** → **Build projects** → **Create build project**

**Every option explained:**

**Project Configuration:**
| Option | Value | Explanation |
|---|---|---|
| **Project name** | `my-node-app-build` | Name for this build project |
| **Description** | `Builds and zips Node app` | Optional |
| **Build badge** | Enable | Shows a green/red badge you can put in your GitHub README |
| **Enable concurrent builds** | Leave default | Allows multiple builds to run at same time |

**Source:**
| Option | Value | Explanation |
|---|---|---|
| **Source provider** | **GitHub** (via CodeStar Connections) | Console may say **GitHub** with a note about connections |
| **Connection** | Select the connection you created in **Step 4** | Must be **Available** |
| **Repository** | Your GitHub repo (e.g. `YOUR_USERNAME/my-node-app`) | Same repo the pipeline will use |
| **Reference type** | `Branch` | Build from a branch (not a specific commit) |
| **Branch** | `main` | Must match the pipeline source branch |

**Environment:**
| Option | Value | Explanation |
|---|---|---|
| **Environment image** | `Managed image` | AWS provides the build machine. `Custom image` = bring your own Docker image |
| **Operating system** | `Amazon Linux 2` | The OS of the build machine |
| **Runtime** | `Standard` | General purpose runtime |
| **Image** | `aws/codebuild/amazonlinux2-x86_64-standard:5.0` | Latest standard image |
| **Image version** | `Always use the latest` | Auto-update the build image |
| **Environment type** | `Linux` | |
| **Privileged** | Disable (unless using Docker-in-Docker) | Enables Docker inside build. Only needed if your build creates Docker images |
| **Service role** | `New service role` | CodeBuild needs IAM permissions to access S3, CloudWatch Logs, and (via the connection) your GitHub source |
| **Role name** | `codebuild-my-node-app-service-role` | Auto-created |

**Buildspec:**
| Option | Value | Explanation |
|---|---|---|
| **Build specifications** | `Use a buildspec file` | Reads `buildspec.yml` from your repo root |
| **Buildspec name** | Leave blank (default: `buildspec.yml`) | If your file has a different name/path, specify it here |

> ⚠️ Alternative: `Insert build commands` — you can paste commands directly in the console instead of using a file. But using the file is better practice.

**Artifacts:**
| Option | Value | Explanation |
|---|---|---|
| **Type** | `Amazon S3` | Upload build output to S3 |
| **Bucket name** | `my-app-artifacts-[yourname]` | Your S3 bucket |
| **Name** | `my-app-build` | Folder name inside the bucket |
| **Artifacts packaging** | `Zip` | Zip the artifact before uploading |
| **Encryption key** | Default | Use default AWS managed key |

**Logs:**
| Option | Value | Explanation |
|---|---|---|
| **CloudWatch logs** | Enable | See build logs in real-time in CloudWatch |
| **Group name** | `/aws/codebuild/my-node-app-build` | Log group name |
| **Stream name** | Leave blank | Auto-generated per build |
| **S3 logs** | Optional | Also save logs to S3 (useful for long-term audit) |

2. Click **Create build project**

---

### STEP 6: Create CodeDeploy Application + Deployment Group

**Step 6a: Create Application**
1. Go to **CodeDeploy** → **Applications** → **Create Application**
2. **Application name**: `my-node-app-deploy`
3. **Compute platform**:
   - `EC2/On-premises` ← for deploying to EC2 (our case)
   - `Lambda` ← for deploying AWS Lambda functions
   - `ECS` ← for deploying Docker containers on ECS
4. Click **Create Application**

**Step 6b: Create IAM Role for CodeDeploy**
1. Go to **IAM** → **Roles** → **Create Role**
2. **Trusted entity**: `AWS Service` → `CodeDeploy`
3. **Use case**: `CodeDeploy`
4. It auto-attaches `AWSCodeDeployRole` policy
5. **Role name**: `CodeDeploy-Service-Role`
6. Create the role

**Step 6c: Create Deployment Group**
1. Back in CodeDeploy → your application → **Create deployment group**

**Every option explained:**

| Option | Value | Explanation |
|---|---|---|
| **Deployment group name** | `my-app-deployment-group` | Name for this group |
| **Service role** | `CodeDeploy-Service-Role` | The role we just created |
| **Deployment type** | `Blue/green` | ← THIS IS WHAT YOU NEED FOR THE PRACTICAL |

**Blue/Green specific options:**
| Option | Value | Explanation |
|---|---|---|
| **Environment configuration** | `Automatically copy Auto Scaling group` | CodeDeploy creates new Green instances automatically. Alternative: `Manually provision instances` = you create Green instances yourself |
| **Deployment settings - traffic rerouting** | `Reroute traffic immediately` | As soon as Green is healthy, switch traffic. Alternative: `Wait and manually reroute` = you click a button to switch |
| **Original revision termination** | `5 minutes` after traffic shift | How long to keep Blue instances running after switch (for easy rollback) |

**Load Balancer** (required for Blue/Green):
| Option | Value | Explanation |
|---|---|---|
| **Enable load balancing** | Yes | Blue/Green needs a load balancer to shift traffic between Blue and Green |
| **Load balancer** | Create an Application Load Balancer OR use existing | The LB is what actually routes traffic from Blue → Green |

> ⚠️ For a simple lab without a load balancer, choose `In-place` deployment instead. Blue/Green **requires** a load balancer on EC2.

**For In-Place deployment (simpler for lab):**
| Option | Value | Explanation |
|---|---|---|
| **Deployment type** | `In-place` | Stops old app on same server, installs new app. No duplicate servers. Downtime possible. |
| **Environment** | `Amazon EC2 instances` | |
| **Tag group** | Key: `Name`, Value: `my-app-server` | How CodeDeploy finds your EC2 — by its tag! |
| **Deployment configuration** | `CodeDeployDefault.AllAtOnce` | See below |

**Deployment Configurations explained:**

| Config | Behavior | Use case |
|---|---|---|
| `AllAtOnce` | Deploy to ALL instances at same time | Fast but risky — if it fails, all instances are down |
| `HalfAtATime` | Deploy to 50% of instances at a time | Balance of speed and safety |
| `OneAtATime` | Deploy to one instance, wait, then next | Safest, slowest |
| `Custom` | You define %, e.g. 25% at a time | Full control |

2. Click **Create deployment group**

---

### STEP 7: Create CodePipeline

1. Go to **CodePipeline** → **Pipelines** → **Create Pipeline**

**Pipeline settings:**
| Option | Value | Explanation |
|---|---|---|
| **Pipeline name** | `my-node-app-pipeline` | |
| **Execution mode** | `Superseded` | If a new commit comes in while pipeline is running, the old run gets cancelled and new one starts. Alternative: `Queued` = wait your turn. `Parallel` = run both at same time |
| **Service role** | `New service role` | Auto-creates IAM role for pipeline to access CodeStar Connections (GitHub), CodeBuild, CodeDeploy, S3 |
| **Artifact store** | `Default location` | Uses the S3 bucket in your region. Alternative: `Custom location` = specify your own bucket |
| **Encryption key** | `Default AWS managed key` | Encrypts pipeline artifacts |

---

**STAGE 1 — SOURCE:**

| Option | Value | Explanation |
|---|---|---|
| **Source provider** | **GitHub (Version 2)** or **GitHub** via **AWS CodeStar Connections** | Exact label depends on console version |
| **Connection** | Same **Available** connection as in Step 4 | |
| **Repository** | Same GitHub repo as CodeBuild (e.g. `owner/repo`) | |
| **Branch name** | `main` | Which branch to watch for changes |
| **Change detection options** | **Amazon CloudWatch Events (recommended)** | Starts the pipeline when GitHub notifies AWS via the connection (like a webhook). Alternative: polling — slower |
| **Output artifact format** | `CodePipeline default` | How the source code is packaged for the next stage |

---

**STAGE 2 — BUILD:**

| Option | Value | Explanation |
|---|---|---|
| **Build provider** | `AWS CodeBuild` | |
| **Region** | Same region | |
| **Project name** | `my-node-app-build` | The project we created in Step 5 |
| **Build type** | `Single build` | One build per pipeline run. Alternative: `Batch build` = run multiple builds in parallel |
| **Input artifacts** | `SourceArtifact` | The output from Stage 1 (your code) |
| **Output artifacts** | `BuildArtifact` | The zip file CodeBuild creates — this goes to Stage 3 |

---

**STAGE 3 — DEPLOY:**

| Option | Value | Explanation |
|---|---|---|
| **Deploy provider** | `AWS CodeDeploy` | |
| **Region** | Same region | |
| **Application name** | `my-node-app-deploy` | |
| **Deployment group** | `my-app-deployment-group` | |
| **Input artifacts** | `BuildArtifact` | The zip from Stage 2 |

---

2. Click **Create Pipeline**

> 🎉 The pipeline will immediately run its first execution using your current code!

---

## 🔵🟢 PART 3 — BLUE/GREEN DEPLOYMENT EXPLAINED DEEPLY

### What is it?

```
BEFORE DEPLOYMENT:
┌─────────────────────────────────────────┐
│  Load Balancer                          │
│       │                                 │
│       ▼                                 │
│  [BLUE servers] ← 100% traffic         │
│  (old version of your app)             │
└─────────────────────────────────────────┘

DURING DEPLOYMENT:
┌─────────────────────────────────────────┐
│  Load Balancer                          │
│       │                                 │
│  ┌────┴────┐                           │
│  ▼         ▼                           │
│ [BLUE]   [GREEN] ← new version         │
│  (old)   installing, running            │
│           health checks running...      │
└─────────────────────────────────────────┘

AFTER HEALTH CHECKS PASS:
┌─────────────────────────────────────────┐
│  Load Balancer                          │
│       │                                 │
│  ┌────┴────┐                           │
│  ▼         ▼                           │
│ [BLUE]   [GREEN] ← 100% traffic        │
│  (old,   (new, live)                   │
│  kept                                   │
│  for 5min)                             │
└─────────────────────────────────────────┘

IF SOMETHING FAILS ON GREEN:
┌─────────────────────────────────────────┐
│  Load Balancer                          │
│       │                                 │
│       ▼                                 │
│  [BLUE] ← traffic rerouted back!       │
│  (old version, still running)          │
│  [GREEN] ← terminated                  │
└─────────────────────────────────────────┘
```

### Why Blue/Green? (Key Viva Points)

| Feature | Blue/Green | In-Place |
|---|---|---|
| **Downtime** | Zero downtime | Brief downtime |
| **Rollback speed** | Instant (just reroute LB) | Slow (redeploy old version) |
| **Cost** | Higher (2x servers briefly) | Lower (same servers) |
| **Risk** | Low (Blue stays alive) | High (if deploy fails, app is down) |
| **Used for** | Production apps | Dev/test environments |

---

## 🧠 PART 4 — VIVA QUESTIONS + ANSWERS

**Q: What is the difference between `buildspec.yml` and `appspec.yml`?**
> `buildspec.yml` = instructions for **CodeBuild** (how to BUILD the app — install deps, run tests, zip it). `appspec.yml` = instructions for **CodeDeploy** (how to DEPLOY the app — which files to copy where, which scripts to run on the server).

**Q: What does `version: 0.2` mean in buildspec?**
> It's the buildspec syntax version. 0.2 is the current version. 0.1 was older and had limitations (couldn't set runtime versions in the file). Always use 0.2.

**Q: What is a CodeDeploy lifecycle hook?**
> A lifecycle hook is a point in the deployment process where you can run a script. The hooks run in this order: ApplicationStop → BeforeInstall → AfterInstall → ApplicationStart → ValidateService.

**Q: Why do we enable versioning on S3?**
> So every build artifact (zip file) is kept. If a new build breaks production, you can roll back by deploying an older version directly from S3. Without versioning, old artifacts get overwritten.

**Q: What is a webhook in CodePipeline?**
> With GitHub via **CodeStar Connections**, AWS receives repository change notifications so **CloudWatch Events** can start the pipeline right after you **`git push`**. If you turned off event-based detection and used polling instead, runs would be delayed until the next poll.

**Q: Where does the pipeline get source code if we are not using CodeCommit?**
> From **GitHub** through the **CodeStar Connection**. You authorize AWS once; the source stage clones the branch you configured (here **`main`** in **`us-east-1`**).

**Q: What is $CODEBUILD_BUILD_NUMBER?**
> An auto-incrementing number assigned to each build (1, 2, 3...). Used to give each artifact a unique name so they don't overwrite each other in S3.

**Q: What happens if `ValidateService` script exits with code 1?**
> CodeDeploy marks the deployment as FAILED. If Blue/Green deployment, it automatically reroutes traffic back to Blue (rollback). The Green instances are terminated.

**Q: Why is Blue/Green more expensive than In-Place?**
> Because you briefly run TWO sets of servers simultaneously — the Blue (old) and the Green (new). After the switch, Blue is terminated (but you keep it for a few minutes as fallback).

**Q: What is the artifact store in CodePipeline?**
> An S3 bucket where CodePipeline stores intermediate artifacts as they pass between stages. Stage 1 (Source) puts code there. Stage 2 (Build) reads from there and puts the built zip back. Stage 3 (Deploy) reads the zip.

**Q: Can CodePipeline have more than 3 stages?**
> Yes! You can add stages like: Approval (manual human approval before deploying to prod), Test (run automated tests), or a second Deploy stage (deploy to staging, then production).

---

## 🚀 PART 5 — QUICK VERIFICATION CHECKLIST

After everything is set up, verify:

```bash
# SSH into your EC2
ssh -i my-keypair.pem ec2-user@YOUR_EC2_PUBLIC_IP

# Check CodeDeploy agent is running
sudo service codedeploy-agent status
# Should say: "active (running)"

# Check your app is running
curl http://localhost:3000
# Should return: Hello from Blue/Green Deployment! Version 1.0

# Check app logs
cat /var/log/app.log

# Check CodeDeploy deployment logs
cat /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

Then open in browser: `http://YOUR_EC2_PUBLIC_IP:3000`

---

## ⚡ PART 6 — TESTING THE PIPELINE

1. Make a change to `app.js` — change the version text:
```javascript
res.end('<h1>Hello from Blue/Green Deployment! Version 2.0</h1>');
```

2. Push to **GitHub**:
```bash
git add .
git commit -m "Update to version 2.0"
git push origin main
```

3. Go to **CodePipeline** console — watch the pipeline automatically trigger and run through all 3 stages after GitHub notifies AWS (usually within about a minute).

4. After it completes, refresh `http://YOUR_EC2_PUBLIC_IP:3000` — it should now say **Version 2.0**

---

That's everything from zero to a working CI/CD pipeline. The files in Part 1 go in your project folder, and the steps in Part 2 onward you follow on the AWS console side by side. Let me know when you're stuck at any specific step!