# AWS ECR Beginner Guide  Push Existing or New Images

## Who this is for
- You have a Docker image locally (either built by you or pulled from Docker Hub) and you want to **push it to ECR**.
- You prefer **UI-first** steps, with CLI only for Docker operations.

---

## Prerequisites:

1. **AWS account** and an **IAM user/role** with ECR permissions  
   - For learning: attach managed policy `AmazonEC2ContainerRegistryFullAccess` (later switch to least privilege).
2. **AWS CLI v2** installed and configured (`aws configure`).  
3. **Docker** installed and running on your machine (or EC2 instance).  
4. (Optional) **Docker credential helper** (`docker-credential-ecr-login`) for auto-refreshing ECR tokens.
5. **Set your default region** to where you’ll create the ECR repo (e.g., `ap-south-1`).

> Verify credentials quickly:
> ```bash
> aws sts get-caller-identity --region ap-south-1
> ```
> If this fails, fix credentials before proceeding (see Troubleshooting).

---

## Quick variables (adjust if different)

- **AWS Account ID**: `920736617090`  
- **Region**: `ap-south-1`  
- **ECR repository** (choose one style):
  - Simple name: `test`
  - Or namespace style: `sohelqt8797/ec2-neon-pulse` (must also be created in ECR)

> ✅ **Important**: Docker only pushes images that are **tagged with the exact ECR repository URI**.  
> If your repo is `test`, your final tag must look like:  
> `920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:TAG`

---

## A. Create an ECR Repository (UI)

1. Sign in to the **AWS Console** → open **Elastic Container Registry**.  
2. Click **Create repository**.  
3. **Visibility**: Private (default).  
4. **Repository name**:  
   - Use lowercase letters, numbers, hyphens; you can also use slashes for namespaces (e.g., `sohelqt8797/ec2-neon-pulse`).  
   - Example: `test`
5. **Image tag immutability**: *Enable* (recommended for prod to prevent tag overwrite).  
6. **Scan on push**: *Enable* (recommended).  
7. **Encryption**: AES‑256 (default, managed by ECR) is fine.  
8. Click **Create repository**.

> You can always edit settings later (immutability, scan on push, lifecycle policy).

---

## B. Authenticate Docker to ECR (CLI — required)

ECR uses a short‑lived token for Docker login. Run the command shown in **“View push commands”** on your repo page, or use this:

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 920736617090.dkr.ecr.ap-south-1.amazonaws.com
```

Expected output: `Login Succeeded`.

If this fails, see **Troubleshooting** below.

---

## C. Push an image to your repository

You have two common paths — pick **Option 1** _or_ **Option 2**.

### Option 1 — You already have an image locally (e.g., from Docker Hub)

Example: you previously pulled `sohelqt8797/ec2-neon-pulse:v1` and want to push it to ECR **repo `test`**.

```bash
# (If needed) Pull the image from Docker Hub
docker pull sohelqt8797/ec2-neon-pulse:v1

# Tag the local image for your ECR repo (IMPORTANT: match your ECR repo name)
docker tag sohelqt8797/ec2-neon-pulse:v1   920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1

# Push to ECR
docker push 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

> ⚠️ If you prefer a namespaced ECR repo like `sohelqt8797/ec2-neon-pulse`, first **create that exact repo** in ECR.  
> Then tag accordingly:
> ```bash
> docker tag sohelqt8797/ec2-neon-pulse:v1 >   920736617090.dkr.ecr.ap-south-1.amazonaws.com/sohelqt8797/ec2-neon-pulse:v1
> docker push 920736617090.dkr.ecr.ap-south-1.amazonaws.com/sohelqt8797/ec2-neon-pulse:v1
> ```

### Option 2 — Build a new image locally and push

From your project directory containing a `Dockerfile`:

```bash
# Build a local image
docker build -t myapp:latest .

# Tag it for ECR (repo name must exist in ECR)
docker tag myapp:latest   920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:latest

# Push it
docker push 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:latest
```

---

## D. Verify your image

### In the AWS Console (UI)
1. ECR → **Repositories** → select your repository (e.g., `test`).  
2. Open the **Images** tab.  
3. You should see your **image tag** (e.g., `v1` or `latest`), **digest**, **pushed at**, and **scan status**.

### From the CLI
```bash
aws ecr describe-images --repository-name test --region ap-south-1
```

### (Optional) Pull from ECR to prove it’s there
```bash
docker pull 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

---

## E. (Optional) Enable lifecycle policy (UI)

Keep storage tidy by expiring older or untagged images.

1. ECR → your repository → **Lifecycle policy** tab → **Create lifecycle policy**.  
2. Example rule: expire untagged images older than **14 days**.  
3. Save. (Cleanup runs automatically.)

---

## F. (Optional) Enable/Review vulnerability scanning (UI)

1. ECR → your repository → **Settings**.  
2. Ensure **Scan on push** is **Enabled**.  
3. To scan an existing image manually:  
   - Go to **Images**, select the image → **Actions** → **Scan image**.  
4. Review findings in the image details page.

---

## G. Common pitfalls (read this if push didn’t work)

- **“An image does not exist locally with the tag: …/test:latest”**  
  You tried to push a tag that doesn’t exist locally. Fix by **tagging** the existing local image:
  ```bash
  docker tag LOCAL_IMAGE:TAG 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:latest
  docker push 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:latest
  ```

- **“repository does not exist” (ECR)**  
  Create the repo **first** in the console (or `aws ecr create-repository --repository-name test --region ap-south-1`).

- **“UnrecognizedClientException” / “InvalidClientTokenId”** during Docker login  
  Your AWS credentials are invalid/expired. Reconfigure:
  ```bash
  aws configure
  aws sts get-caller-identity --region ap-south-1
  ```
  If `sts` fails, create new access keys in **IAM → Users → Security credentials** and try again.

- **“no basic auth credentials”** while pushing  
  You aren’t logged in to ECR. Run the login command in section **B** and retry the push.

- **Tag immutability prevents overwriting**  
  If repo has immutability enabled and `latest` already exists, push with a **new tag** (e.g., `v2`) or disable immutability.

- **Wrong account ID / region**  
  Ensure you’re tagging/pushing to the **same account and region** where the repo exists. The URI must match exactly:
  `ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/REPO:TAG`.

---

## H. Quick copy‑paste session (works end‑to‑end)

```bash
# 0) Verify creds
aws sts get-caller-identity --region ap-south-1

# 1) (UI) Create ECR repo named "test"

# 2) Login to ECR
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 920736617090.dkr.ecr.ap-south-1.amazonaws.com

# 3) Use an existing local image (from Docker Hub) and retag for ECR
docker pull sohelqt8797/ec2-neon-pulse:v1
docker tag sohelqt8797/ec2-neon-pulse:v1   920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1

# 4) Push to ECR
docker push 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1

# 5) Verify (CLI)
aws ecr describe-images --repository-name test --region ap-south-1

# (Optional) Pull from ECR to confirm
docker pull 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

---

## I. Appendix — Minimal IAM policy for push/pull to one repo (example)

Replace region/account/repo with yours:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:920736617090:repository/test"
    }
  ]
}
```

---

## J. FAQ

- **Do repository names need lowercase?**  
  Use lowercase with numbers and hyphens. You can use slashes to create namespace‑style names.

- **Do I need to rebuild images to push to ECR?**  
  No. You can retag any existing local image and push, as shown above.

- **How do I avoid logging in every 12 hours?**  
  Install the ECR docker credential helper on your machine or CI runner.

---

Happy shipping 🚀
