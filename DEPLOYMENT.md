# Deploy Plane (Custom Code) on AWS EC2 — Step by Step

This guide deploys your local fork/branch of Plane, not the official release.

## Prerequisites

- Your code is pushed to a Git remote (GitHub, GitLab, etc.)
- An AWS account with credits

---

## Step 1: Launch EC2 Instance

- **AMI:** Ubuntu Server 24.04 LTS (x86)
- **Instance type:** t3.medium (2 vCPU, 4 GiB RAM)
- **Storage:** 30 GiB gp3
- **Key pair:** Create or select one (download the `.pem` file)
- **Auto-assign public IP:** Enable
- **Security Group rules (inbound):**

| Type  | Port | Source               |
| ----- | ---- | -------------------- |
| SSH   | 22   | My IP                |
| HTTP  | 80   | Anywhere (0.0.0.0/0) |
| HTTPS | 443  | Anywhere (0.0.0.0/0) |

Launch the instance and note the **public IP address**.

---

## Step 2: SSH into the Instance

```bash
chmod 400 plane-ssh.pem
ssh -i plane-ssh.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

---

## Step 3: Install Docker and Git

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
exit
```

SSH back in so the group change takes effect:

```bash
ssh -i plane-ssh.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Verify:

```bash
docker --version
git --version
```

---

## Step 4: Clone Your Repository

```bash
git clone https://github.com/YOUR_USERNAME/plane.git
cd plane
git checkout YOUR_BRANCH
```

> Replace `YOUR_USERNAME` and `YOUR_BRANCH` with your actual GitHub username and branch name.

---

## Step 5: Configure Environment Files

### 5a: Root `.env` (for Docker Compose — database, redis, minio, rabbitmq)

```bash
cp .env.example .env
```

No changes needed — defaults work out of the box.

### 5b: API `.env` (for the backend services)

```bash
cp apps/api/.env.example apps/api/.env
```

Edit the API env file:

```bash
nano apps/api/.env
```

Change these values (replace `YOUR_EC2_PUBLIC_IP` with your actual IP):

```
WEB_URL=http://YOUR_EC2_PUBLIC_IP
CORS_ALLOWED_ORIGINS=http://YOUR_EC2_PUBLIC_IP
USE_MINIO=1
AWS_S3_ENDPOINT_URL=http://plane-minio:9000
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## Step 6: Build and Start All Services

```bash
docker compose up --build -d
```

This builds all containers from your source code and starts them. **First build takes 10-15 minutes** — it's compiling the frontend apps, installing Python dependencies, etc.

Monitor progress:

```bash
docker compose logs -f
```

(`Ctrl+C` to stop watching logs — services keep running)

---

## Step 7: Open in Browser

Go to:

```
http://YOUR_EC2_PUBLIC_IP
```

Create your admin account and start using Plane with your custom code.

---

## Useful Commands

| Action                     | Command                        |
| -------------------------- | ------------------------------ |
| View all logs              | `docker compose logs -f`       |
| View specific service logs | `docker compose logs -f api`   |
| Stop all services          | `docker compose down`          |
| Restart all services       | `docker compose restart`       |
| Rebuild after code changes | `docker compose up --build -d` |
| Check running containers   | `docker compose ps`            |

---

## Making Code Changes on the Server

If you push new commits to your branch:

```bash
git pull
docker compose up --build -d
```

This rebuilds only the changed services and restarts them.

---

## Stopping & Cleaning Up (to avoid charges)

When you're done, **terminate the EC2 instance** from the AWS Console:

1. Go to **EC2 → Instances**
2. Select your instance
3. **Instance state → Terminate instance**

> A stopped instance still charges for EBS storage. **Terminate** it to stop all charges.

---

## Cost

| Resource                  | 24-hour cost |
| ------------------------- | ------------ |
| t3.medium on-demand       | ~$1.00       |
| 30 GiB gp3 storage        | ~$0.08       |
| Data transfer (light use) | ~$0.00       |
| **Total**                 | **~$1.08**   |
