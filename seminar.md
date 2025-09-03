# AWS ECR Seminar Guide — Step by Step

## 🎯 Objective
By the end of this guide, you should be able to:
- Explain what Amazon ECR is and why it’s used.
- Create an ECR repository from the AWS Console UI.
- Authenticate Docker to ECR and push an image.
- Verify and scan images in the UI.
- Troubleshoot common issues like invalid credentials or missing repos.
- Share best practices for production use.

---

## 🗝️ Key Concepts (Terminology to use in seminar)

- **Registry**: The server that stores images. In AWS, that’s `ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com`.
- **Repository**: A collection of related images (e.g., `test`).
- **Image**: Packaged application, containing code, libraries, and filesystem layers.
- **Tag**: Human-friendly label like `v1` or `latest` (mutable).
- **Digest**: Immutable `sha256:...` hash that uniquely identifies an image.
- **Scan on push**: Security scanning for vulnerabilities when you push an image.
- **Image tag immutability**: Prevents overwriting existing tags.
- **Lifecycle policy**: Rules to automatically delete old or untagged images.
- **Authorization token**: Short-lived Docker login credential from AWS CLI.
- **Credential helper**: `docker-credential-ecr-login` for automatic login refresh.

---

## ✅ Prerequisites

1. **AWS account** + IAM user/role with permissions.  
   - For learning: attach `AmazonEC2ContainerRegistryFullAccess`.
2. **AWS CLI** v2 installed and configured (`aws configure`).  
3. **Docker** installed and running.  
4. (Optional) **docker-credential-ecr-login** for token auto-refresh.  
5. Set your **region** (e.g., `ap-south-1`).  

Test your credentials:
```bash
aws sts get-caller-identity --region ap-south-1
```

---

## 🛠️ Step-by-Step Demo Flow

### 1. Create an ECR Repository (UI)
1. AWS Console → **ECR** → **Repositories** → **Create repository**.  
2. **Name**: `test` (must be lowercase, use hyphens if needed).  
3. Options:
   - **Scan on push**: Enable.
   - **Image tag immutability**: Enable for prod.
   - **Encryption**: AES-256 (default).
4. Click **Create repository**.

### 2. Authenticate Docker to ECR (CLI)
Run the login command shown in **“View push commands”** in the Console, or:
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 920736617090.dkr.ecr.ap-south-1.amazonaws.com
```
Expected: `Login Succeeded`.

### 3. Retag your local image
If you already pulled `sohelqt8797/ec2-neon-pulse:v1` from Docker Hub:
```bash
docker tag sohelqt8797/ec2-neon-pulse:v1   920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

### 4. Push the image
```bash
docker push 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

### 5. Verify the image
- **In UI**: Open repo → **Images** tab → check tag, digest, and scan results.  
- **In CLI**:
```bash
aws ecr describe-images --repository-name test --region ap-south-1
```

### 6. (Optional) Pull back from ECR to confirm
```bash
docker pull 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

---

## 🔧 Troubleshooting (Q&A Style)

- **Error: “An image does not exist locally with the tag …”**  
  → You didn’t tag your local image with the ECR URI. Retag before pushing.

- **Error: “repository does not exist”**  
  → The ECR repo wasn’t created. Create it in the Console or CLI.

- **Error: “InvalidClientTokenId / UnrecognizedClientException”**  
  → AWS credentials expired/invalid. Run `aws configure` with fresh access keys.

- **Error: “no basic auth credentials”**  
  → Forgot to login. Run the `aws ecr get-login-password` login command.

- **Image tag immutability prevents overwrite**  
  → Use a new tag (e.g., `v2`) or disable immutability.

- **Wrong account/region**  
  → Ensure you are tagging with the correct URI:  
  `ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/repo:tag`.

---

## 📌 Best Practices to Share in Class

- Use **immutable tags** (e.g., `app:20250903-commitSHA`) instead of overwriting `latest`.
- Enable **scan-on-push** and act on high/critical vulnerabilities.
- Apply **lifecycle policies** to remove old/untagged images and save cost.
- Use **least privilege IAM policies** for pushing/pulling images.
- Avoid long-lived AWS keys → use **IAM roles** or **OIDC in CI/CD**.
- Use **credential helper** to avoid re-logins on dev machines.
- For secure supply chain: **sign images** (Cosign/Notation) and verify before deployment.

---

## 🧾 Cheat-Sheet Commands

Login:
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 920736617090.dkr.ecr.ap-south-1.amazonaws.com
```

Tag:
```bash
docker tag LOCAL_IMAGE:TAG 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

Push:
```bash
docker push 920736617090.dkr.ecr.ap-south-1.amazonaws.com/test:v1
```

Verify:
```bash
aws ecr describe-images --repository-name test --region ap-south-1
```

---

## 🎤 Closing Statement

> “Amazon ECR is a secure, fully-managed Docker registry that integrates directly with ECS, EKS, and CI/CD pipelines.  
> The workflow is simple: **create repo in UI → login with CLI → tag local image with repo URI → push → verify in UI.**  
> If anything breaks, check credentials, repo existence, and tagging. This workflow scales from classroom demos to production pipelines.”

---
