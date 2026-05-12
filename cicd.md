# Complete AWS CI/CD Pipeline — Full Step-by-Step Guide

I'll walk you through **everything** — every file, every console option, every button click. Read this top to bottom and do it side by side.

---

## 🗂️ PART 0 — WHAT WE ARE BUILDING (Big Picture)

**Region:** use **`us-east-1`** (US East, N. Virginia) for everything in this practical — EC2, S3, CodeCommit, CodeBuild, CodePipeline, CodeDeploy, and when you run `aws configure` / GitHub Secrets.

```
Your GitHub Repo
      │
      │  (you only `git push` here)
      ▼
 GitHub Actions  ──mirrors to──▶  CodeCommit  ──(webhook trigger)──▶  CodePipeline
                                           │
                              ┌────────────┼────────────┐
                              ▼            ▼            ▼
                           STAGE 1      STAGE 2      STAGE 3
                           Source       Build        Deploy
                        (CodeCommit) (CodeBuild)  (CodeDeploy)
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
├── .github/
│   └── workflows/
│       └── sync-to-codecommit.yml   ← GitHub pushes your commits into CodeCommit automatically
└── scripts/
    ├── install_dependencies.sh
    ├── start_server.sh
    ├── stop_server.sh
    └── validate_service.sh   ← required: appspec.yml references it; missing file = CodeDeploy ScriptMissing
```

> **Rule:** Every `location:` under `hooks:` in `appspec.yml` must exist in the repo at that path. CodeBuild zips the repo → CodeDeploy extracts it → if a script is missing, the matching hook fails (often **ValidateService** with `ScriptMissing`).

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

  # Do NOT run start_server.sh in AfterInstall AND ApplicationStart — that starts Node twice.
  # First start binds :3000; second hits EADDRINUSE → log error → ValidateService / health can fail (HEALTH_CONSTRAINTS).

  ApplicationStart:
    # Runs AFTER files are on the server — start the app once here
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
5. AfterInstall             ← optional (e.g. chmod); do not duplicate "start" here if ApplicationStart starts the app
6. ApplicationStart         ← start new app (one place only for listen/bind)
7. ValidateService          ← confirm it's working
```

---

### FILE 5: `scripts/stop_server.sh`
```bash
#!/bin/bash
# Stop the Node app before a new revision is installed (must exit 0 for CodeDeploy)

pkill -f "node app.js" 2>/dev/null || true
sleep 2
exit 0
```

---

### FILE 6: `scripts/install_dependencies.sh`
```bash
#!/bin/bash
# Runs before files are copied. Keep this fast — long yum update can hit the hook timeout.

# Do NOT run yum update -y here unless you raise appspec timeout; it often exceeds 5 minutes.

if ! command -v node &> /dev/null; then
    curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
    yum install -y nodejs
fi

mkdir -p /var/www/html
exit 0
```

---

### FILE 7: `scripts/start_server.sh`
```bash
#!/bin/bash
# Start once per deploy (only referenced from ApplicationStart in appspec.yml)

pkill -f "node app.js" 2>/dev/null || true
sleep 1
cd /var/www/html
npm install
nohup node app.js > /var/log/app.log 2>&1 &
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
> This URL matches **`us-east-1`**. If you ever use another region, replace `us-east-1` in the hostname with your region.

2. Click **Launch Instance**
3. Note down the **Public IP** of your EC2 instance

---

### STEP 3: Create S3 Bucket (for artifacts)

1. Go to **S3** → **Create Bucket**

**All options explained:**

| Option | What to set | Explanation |
|---|---|---|
| **Bucket name** | `my-app-artifacts-[yourname]` | Must be globally unique. Add your name to make it unique |
| **Region** | **`us-east-1`** | Same region everywhere |
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

### STEP 4: CodeCommit — you push only to GitHub

Flow: **`git push` → GitHub → GitHub Actions → CodeCommit** → pipeline (Steps 5–7 stay CodeCommit-based).

**4a — Create the CodeCommit repo**

1. Console region: **`us-east-1`**
2. **CodeCommit** → **Create repository**
3. Name: **`my-node-app`** (same name everywhere below)
4. Create

**4b — IAM user GitHub will use to push into CodeCommit**

1. **IAM** → **Users** → **Create user** → name e.g. `github-codecommit-sync`
2. Attach **`AWSCodeCommitPowerUser`** (simple for the exam) or a tighter custom policy limited to repo `my-node-app`
3. **Security credentials** → **Create access key** → **Application running outside AWS** → save **Access key ID** and **Secret access key**

**4c — GitHub secrets**

Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

| Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | From step 4b |
| `AWS_SECRET_ACCESS_KEY` | From step 4b |

**4d — Workflow file**

Create `.github/workflows/sync-to-codecommit.yml` in your GitHub repo:

```yaml
name: Sync GitHub to CodeCommit

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - run: pip install git-remote-codecommit

      - name: Push to CodeCommit
        run: |
          git remote add codecommit codecommit::us-east-1://my-node-app
          git push codecommit HEAD:refs/heads/main
```

Commit and push this file to **`main`**. After that, day-to-day work is only:

```bash
git push origin main
```

The workflow copies that commit into CodeCommit; **CloudWatch Events on CodeCommit** starts your pipeline (Step 7).

**Keep these matching**

| Setting | Value |
|---|---|
| AWS region | **`us-east-1`** everywhere (workflow, console, `aws configure`) |
| Branch | **`main`** (workflow, pipeline, CodeBuild) |
| Repo name | **`my-node-app`** in CodeCommit and in `codecommit::us-east-1://my-node-app` |

**If you need a manual push once** (e.g. debugging): install `git-remote-codecommit`, run `aws configure` with region **`us-east-1`**, then `git clone codecommit::us-east-1://my-node-app` and push — same repo URL pattern as in the workflow.

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
| **Source provider** | `AWS CodeCommit` | Where to pull code from |
| **Repository** | `my-node-app` | Your repo |
| **Reference type** | `Branch` | Build from a branch (not a specific commit) |
| **Branch** | `main` | Which branch to build |

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
| **Service role** | `New service role` | CodeBuild needs IAM permissions to access S3, CodeCommit, etc. |
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
| **Service role** | `New service role` | Auto-creates IAM role for pipeline to access CodeCommit, CodeBuild, CodeDeploy, S3 |
| **Artifact store** | `Default location` | Uses the S3 bucket in your region. Alternative: `Custom location` = specify your own bucket |
| **Encryption key** | `Default AWS managed key` | Encrypts pipeline artifacts |

---

**STAGE 1 — SOURCE:**

| Option | Value | Explanation |
|---|---|---|
| **Source provider** | `AWS CodeCommit` | Where pipeline reads code from |
| **Repository name** | `my-node-app` | |
| **Branch name** | `main` | Which branch to watch for changes |
| **Change detection options** | `Amazon CloudWatch Events (recommended)` | ← THIS IS THE WEBHOOK. CloudWatch watches CodeCommit and triggers pipeline automatically on every `git push`. Alternative: `AWS CodePipeline` = pipeline polls CodeCommit every minute (slower, old way) |
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
> It's an event trigger. When you push to CodeCommit, CloudWatch Events detects the change and automatically starts the pipeline. Without webhook, you'd have to manually click "Release Change" every time.

**Q: If I only push to GitHub, how does CodePipeline still run?**
> GitHub Actions copies your commit into **CodeCommit**. The pipeline is still triggered by CodeCommit (CloudWatch Events), same as if you had pushed to CodeCommit yourself.

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

Use **CLI on EC2** plus **AWS Console** together — interviewers often ask “where would you look in the console if X fails?”

### On the EC2 instance (SSH)

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

# Check CodeDeploy agent log (deployments, hook execution)
cat /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

Then open in browser: `http://YOUR_EC2_PUBLIC_IP:3000`

---

### AWS Console map — where to click for verification (`us-east-1`)

| AWS service | How to get there (console) | What you verify | Interview angle |
|---|---|---|---|
| **CodePipeline** | `Developer tools` → **CodePipeline** → Pipelines → `my-node-app-pipeline` | Latest **execution** is **Succeeded**; Source → Build → Deploy all green | “End-to-end orchestration; stages pass artifacts in S3.” |
| **CodePipeline execution** | Same pipeline → click the **pipeline name** → latest execution row | **Trigger** (what started it), **Commit ID**, duration, **Stage details** | “Each stage can fail independently — check which stage failed first.” |
| **CodeBuild** | **CodeBuild** → **Build projects** → `my-node-app-build` → **Build history** | Status **Succeeded**; click build → **Logs** tab | “Buildspec phases: install, pre_build, build, post_build; logs stream to CloudWatch.” |
| **CodeDeploy** | **CodeDeploy** → **Applications** → `my-node-app-deploy` → **Deployments** | **Status** Succeeded; **Lifecycle events** expanded | “Hooks run in order; failed hook shows which script broke.” |
| **CodeDeploy deployment detail** | Click one deployment ID | **Events** timeline (e.g. BeforeInstall, AfterInstall), instance ID | “Blue/Green vs In-place changes what you see here (e.g. target groups).” |
| **S3** | **S3** → your artifact bucket `my-app-artifacts-*` | New objects after build; versioning shows history | “Artifacts bucket vs pipeline default artifact store — know both exist.” |
| **CloudWatch Logs** | **CloudWatch** → **Log groups** | `/aws/codebuild/...`, `/aws/codedeploy/...` if configured | “First place for build failures and sometimes deploy diagnostics.” |
| **EC2** | **EC2** → **Instances** → select instance | **Status checks** 2/2; correct **IAM role**; **Security groups** (3000 open for lab) | “Instance must reach S3; agent must run; SG must allow traffic you test.” |
| **IAM** | **IAM** → **Roles** | CodePipeline service role, CodeBuild role, CodeDeploy role, EC2 instance profile | “Least privilege: each service has its own role; cross-service trust.” |

---

### CodePipeline source trigger options (often asked)

| Setting (Source stage) | Console label / behavior | When it runs |
|---|---|---|
| **Amazon CloudWatch Events** | “Recommended” change detection | Near real-time when CodeCommit receives a push |
| **AWS CodePipeline** (polling) | Older polling mode | Checks periodically (minutes), slower |

Your guide uses **CloudWatch Events** on CodeCommit so a push (including one mirrored from GitHub Actions) starts the pipeline without manual **Release change**.

---

### CloudWatch Logs — names to recognize

| Log group pattern | What writes here | When you care |
|---|---|---|
| `/aws/codebuild/<project-name>` | CodeBuild | Build fails (npm, tests, zip, permissions) |
| `/aws/codedeploy/...` | Optional agent/plugin logging | Deploy debugging (not always enabled for EC2 the same way as Lambda) |
| **Custom** app log | Your `nohup` → `/var/log/app.log` on EC2 | Runtime errors after deploy |

---

### Symptom → first console stop (debugging map)

| Symptom | Open first | What to mention in an interview |
|---|---|---|
| Pipeline **Source** failed | CodeCommit repo exists; branch `main`; mirror workflow on GitHub succeeded | “Verify commit landed in CodeCommit before blaming pipeline.” |
| Pipeline **Build** failed | CodeBuild → failed build → **Logs** | “Read bottom of log for exit code; often tests or missing file in repo.” |
| Pipeline **Deploy** failed | CodeDeploy → deployment → **Lifecycle event** errors | “ValidateService non-zero, missing appspec, wrong paths, agent down.” |
| App not reachable in browser | EC2 **public IP**, **security group** port 3000, app binding `0.0.0.0` | “Network path vs deploy success — pipeline can succeed but app misconfigured.” |
| Old version still showing | Browser cache; confirm **new deployment** succeeded; `curl` on instance | “Distinguish deploy pipeline vs process restart/caching.” |
| **`EADDRINUSE` in `/var/log/app.log`** + **`HEALTH_CONSTRAINTS`** | Two causes: (1) **`start_server.sh` listed twice** in `appspec.yml` (AfterInstall + ApplicationStart) — second start fails; (2) old Node still on port 3000. Fix: start only in **ApplicationStart**; in `start_server.sh` run `pkill -f "node app.js"` before `nohup`. | “Only one process should bind the port; hooks must be idempotent.” |

---

## ⚡ PART 6 — TESTING THE PIPELINE

### Steps (make a real change)

1. Edit `app.js` — change the version text, for example:
```javascript
res.end('<h1>Hello from Blue/Green Deployment! Version 2.0</h1>');
```

2. Push to **GitHub** only (Step 4 workflow updates CodeCommit):
```bash
git add .
git commit -m "Update to version 2.0"
git push origin main
```

3. **GitHub** → **Actions** → workflow **Sync GitHub to CodeCommit** → wait until **green check**.

4. **CodePipeline** → your pipeline → confirm new execution; all stages **Succeeded**.

5. Refresh `http://YOUR_EC2_PUBLIC_IP:3000` — expect **Version 2.0**.

---

### End-to-end flow table (order matters)

| Step | Where | Success signal |
|---|---|---|
| 1 | Local / IDE | Commit saved |
| 2 | `git push origin main` | GitHub shows new commit |
| 3 | GitHub Actions | Workflow run **Succeeded** |
| 4 | CodeCommit (optional check) | Latest commit matches GitHub |
| 5 | CodePipeline | New execution started automatically |
| 6 | CodeBuild | Build **Succeeded**, artifact in S3 |
| 7 | CodeDeploy | Deployment **Succeeded** |
| 8 | Browser / `curl` | New version text |

---

### CodePipeline execution — statuses you see in the console

| Status | Meaning | Typical next step |
|---|---|---|
| **In progress** | A stage is running | Wait; open stage detail for live links to CodeBuild/CodeDeploy |
| **Succeeded** | All stages finished OK | Verify app behavior on EC2 |
| **Failed** | One stage stopped with error | Click the **failed stage** → link to CodeBuild logs or CodeDeploy events |
| **Superseded** | Newer execution replaced this one (if pipeline mode allows) | Compare execution IDs — interview: “avoid running outdated deploys twice.” |
| **Cancelled** | User or policy stopped it | Check who/what cancelled |

---

### Stage failure — likely causes (interview-style)

| Failed stage | Common causes | Console evidence |
|---|---|---|
| **Source** | Wrong branch name; CodeCommit empty; IAM pipeline role cannot read repo | Source error message; CodeCommit **Commits** tab |
| **Build** | `buildspec.yml` typo; `npm test` fails; missing dependency | CodeBuild log tail |
| **Deploy** | CodeDeploy agent stopped; `appspec.yml` path wrong; hook script exits non-zero | CodeDeploy event log; `/var/log/aws/codedeploy-agent/` on EC2 |

---

That's everything from zero to a working CI/CD pipeline. The files in Part 1 go in your project folder, and the steps in Part 2 onward you follow on the AWS console side by side. Let me know when you're stuck at any specific step!