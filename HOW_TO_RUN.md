# Customer Segmentation — Complete How to Run Guide
**Author: Suresh D R | AI Product Developer & Technology Mentor | DV Analytics**

---

## What Is This Project?

Customer Segmentation assigns customers into 7 groups using K-Means++ clustering.
Simple web app + REST API deployment.
NO drift detection. NO Lambda. NO EventBridge. NO SageMaker pipeline.

```
Marketing manager → opens Streamlit at http://EXTERNAL-IP:8501
                  → uploads CSV → gets segments + AI messages → downloads Excel
                  → sends WhatsApp/Email per segment

CRM system        → calls POST /predict/s3 via API at port 8000
                  → results saved back to S3 automatically
```

---

## Prerequisites — Already Done

```
✅ NB1-NB7 ran in Google Colab → model saved to S3

S3 bucket: customer-segmentation-2026
  models/kmeans_centroids.csv     ← CORE FILE
  models/cluster_profiles.csv
  models/kmeans_plus_model.pkl
  models/cluster_features.csv
```

---

## Project Structure

```
customer-segmentation-deploy/
├── src/
│   ├── segment.py          ← Core centroid mapping
│   ├── llm_messages.py     ← OpenAI message generation
│   ├── api.py              ← FastAPI REST endpoints
│   └── app.py              ← Streamlit dashboard — 3 tabs
├── k8s/
│   └── deployment.yml      ← Kubernetes — 2 replicas
├── .github/workflows/
│   └── deploy.yml          ← CI/CD — push → ECR → EKS
├── Dockerfile
├── requirements.txt
├── .env                    ← Add your real keys here (gitignored)
└── HOW_TO_RUN.md
```

⚠️ **Important:** Never push `.env` or hardcoded keys to GitHub.
All keys must come from environment variables or `.env` file only.

---

# PHASE 1 — Local Setup and Test

## Step 1 — Open Project in VS Code

```bash
# Unzip customer-segmentation-deploy.zip
# Open VS Code → File → Open Folder → customer-segmentation-deploy
# Open Git Bash: Ctrl+Shift+P → Terminal: Select Default Profile → Git Bash
# Then Ctrl+` to open terminal
```

## Step 2 — Create Virtual Environment

```bash
python -m venv venv
source venv/Scripts/activate        # Windows Git Bash
pip install -r requirements.txt
```

## Step 3 — Set Up .env File

```bash
# Open .env file and fill in your real values:
# AWS_ACCESS_KEY_ID=YOUR_REAL_KEY
# AWS_SECRET_ACCESS_KEY=YOUR_REAL_SECRET
# AWS_REGION=ap-south-1
# S3_BUCKET=customer-segmentation-2026
# OPENAI_API_KEY=sk-proj-xxxxxxxxxxxx
```

## Step 4 — Verify S3 Has Model Files

```bash
aws configure
# Enter your AWS credentials when prompted

aws s3 ls s3://customer-segmentation-2026/models/
# Must see:
# kmeans_centroids.csv     ✅
# cluster_profiles.csv     ✅
# kmeans_plus_model.pkl    ✅
# cluster_features.csv     ✅
```

## Step 5 — Upload Sample Data to S3

```bash
aws s3 cp sample_10k_customers.csv \
    s3://customer-segmentation-2026/uploads/sample_10k_customers.csv

echo "Sample data uploaded ✅"
```

## Step 6 — Test Segmentation Locally

```bash
python src/segment.py
# Expected output:
# Loading model artifacts from S3...
# Loaded 7 segment centroids with 34 features
# Segment: At Risk | Confidence: 72.45%
```

## Step 7 — Run Streamlit Locally

```bash
streamlit run src/app.py
# Open http://localhost:8501
```

## Step 8 — Run FastAPI Locally

```bash
uvicorn src.api:app --reload --port 8000
# Open http://localhost:8000/docs
```

---

# PHASE 2 — ECR (Docker Image Storage)

## Step 9 — Create ECR Repository

```bash
aws ecr create-repository \
    --repository-name customer-segmentation \
    --region ap-south-1
```

## Step 10 — Build and Push Docker Image

⚠️ Switch to mobile hotspot for Docker push

```bash
# Build
docker build -t customer-segmentation:v1.0 .

# Login to ECR (replace ACCOUNT_ID with your AWS account ID)
aws ecr get-login-password --region ap-south-1 | \
    docker login --username AWS --password-stdin \
    ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com

# Tag
docker tag customer-segmentation:v1.0 \
    ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/customer-segmentation:latest

# Push
docker push \
    ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/customer-segmentation:latest

# Verify
aws ecr list-images \
    --repository-name customer-segmentation \
    --region ap-south-1
```

---

# PHASE 3 — EKS Deployment

## Step 11 — Create EKS Cluster

⚠️ Costs Rs 600-800/day. Delete after practice.

```bash
eksctl create cluster \
    --name customer-segmentation-cluster \
    --region ap-south-1 \
    --nodegroup-name workers \
    --node-type t3.medium \
    --nodes 2 \
    --nodes-min 1 \
    --nodes-max 3 \
    --managed

# Wait 15-20 minutes...
kubectl get nodes
# Should show 2 nodes Ready ✅
```

## Step 12 — Deploy to EKS

```bash
# Step 1 — Open k8s/deployment.yml
# Find ACCOUNT_ID → replace with your AWS account ID → save file

# Step 2 — Connect kubectl to EKS
aws eks update-kubeconfig \
    --region ap-south-1 \
    --name customer-segmentation-cluster

# Step 3 — Create AWS secrets in Kubernetes
kubectl create secret generic aws-secrets \
    --from-literal=aws-access-key-id=YOUR_AWS_ACCESS_KEY_ID \
    --from-literal=aws-secret-access-key=YOUR_AWS_SECRET_ACCESS_KEY

# Step 4 — Deploy
kubectl apply -f k8s/deployment.yml

# Step 5 — Check status
kubectl get pods
kubectl get service customer-segmentation-service
# Copy EXTERNAL-IP from output ↑
```

## Step 13 — Open Security Group Ports

```bash
# Get security group ID
aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=*customer-segmentation*" \
    --region ap-south-1 \
    --query "SecurityGroups[*].[GroupId,GroupName]"

# Use the sg-XXXXXXXXX that contains "eks-cluster-sg"
# Open port 8501 (Streamlit)
aws ec2 authorize-security-group-ingress \
    --group-id sg-XXXXXXXXX \
    --protocol tcp --port 8501 --cidr 0.0.0.0/0 --region ap-south-1

# Open port 8000 (FastAPI)
aws ec2 authorize-security-group-ingress \
    --group-id sg-XXXXXXXXX \
    --protocol tcp --port 8000 --cidr 0.0.0.0/0 --region ap-south-1
```

## Step 14 — Verify App is Live

```bash
kubectl get service customer-segmentation-service
# Copy EXTERNAL-IP

# Open Streamlit:
http://EXTERNAL-IP:8501

# Test API:
curl http://EXTERNAL-IP:8000/health
```

---

# PHASE 4 — GitHub CI/CD

## Step 15 — Push Code to GitHub

```bash
git init
git add .
git commit -m "Initial commit — Customer Segmentation deployment"
git branch -M main
git remote add origin https://github.com/drsuresh8453/customer-segmentation.git
git push --set-upstream origin main
```

⚠️ **Never push .env or any file with real AWS keys to GitHub.**
The .gitignore already excludes .env.

## Step 16 — Add GitHub Secrets

```
Go to: https://github.com/drsuresh8453/customer-segmentation/settings/secrets/actions
Click "New repository secret" — add all 6:

Name                      Value
──────────────────────────────────────────────────────────────────
AWS_ACCESS_KEY_ID         YOUR_AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY     YOUR_AWS_SECRET_ACCESS_KEY
AWS_REGION                ap-south-1
S3_BUCKET                 customer-segmentation-2026
ECR_REGISTRY              ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
EKS_CLUSTER_NAME          customer-segmentation-cluster
```

## Step 17 — Verify CI/CD

```bash
echo "# trigger" >> src/segment.py
git add . && git commit -m "Test CI/CD" && git push
```

```
Go to: https://github.com/drsuresh8453/customer-segmentation/actions
Should see:
✅ Build Docker image
✅ Push to ECR
✅ Deploy to EKS — rolling update, zero downtime
```

---

# How to Use the Live Dashboard

## Tab 1 — Single Customer Lookup

```
1. Fill in customer details
2. Click "Predict Segment & Generate Message"
3. See:
   - Segment card with urgency and recommended offer
   - Bar chart showing distance to all 7 centroids
   - Chat bubble with personalised AI message
4. Download:
   - Message as .txt
   - Full result as .xlsx
```

## Tab 2 — Batch CSV Upload

```
1. Upload test_10_customers.csv or sample_10k_customers.csv
2. Set Max LLM messages: 10 (for quick test)
3. Click "Segment All Customers + Generate Messages"
4. Download Excel files:
   - 📥 Full Results Excel         → all customers
   - 🚨 At-Risk + Cannot Lose      → urgent customers only
   - ⭐ Champions Only              → VIP customers only
```

## Tab 3 — Segment Overview

```
- Cluster profiles from S3
- Strategy cards for all 7 segments
- Sample message per segment
```

---

# API Quick Reference

```bash
# Health check
curl http://EXTERNAL-IP:8000/health

# Single customer
curl -X POST http://EXTERNAL-IP:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"customer_id":"CUST001","name":"Priya","recency_days":45,
       "frequency":24,"monetary_value":22000,"city_tier":"Metro"}'

# S3 batch job
curl -X POST http://EXTERNAL-IP:8000/predict/s3 \
  -H "Content-Type: application/json" \
  -d '{"input_s3_key":"uploads/sample_10k_customers.csv",
       "output_s3_key":"results/segmented.csv",
       "generate_messages":true}'

# Check job status
curl http://EXTERNAL-IP:8000/jobs/{job_id}
```

---

# All Commands Quick Reference

```bash
# ── Local ───────────────────────────────────────────────────────
python -m venv venv && source venv/Scripts/activate
pip install -r requirements.txt
python src/segment.py
streamlit run src/app.py
uvicorn src.api:app --reload --port 8000

# ── S3 ──────────────────────────────────────────────────────────
aws s3 ls s3://customer-segmentation-2026/models/
aws s3 cp sample_10k_customers.csv \
    s3://customer-segmentation-2026/uploads/sample_10k_customers.csv

# ── ECR ─────────────────────────────────────────────────────────
aws ecr create-repository \
    --repository-name customer-segmentation --region ap-south-1
aws ecr get-login-password --region ap-south-1 | \
    docker login --username AWS --password-stdin \
    ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
docker build -t customer-segmentation:v1.0 .
docker tag customer-segmentation:v1.0 \
    ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/customer-segmentation:latest
docker push \
    ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/customer-segmentation:latest

# ── EKS ─────────────────────────────────────────────────────────
eksctl create cluster --name customer-segmentation-cluster \
    --region ap-south-1 --nodegroup-name workers \
    --node-type t3.medium --nodes 2 --managed
aws eks update-kubeconfig \
    --region ap-south-1 --name customer-segmentation-cluster
kubectl create secret generic aws-secrets \
    --from-literal=aws-access-key-id=YOUR_AWS_ACCESS_KEY_ID \
    --from-literal=aws-secret-access-key=YOUR_AWS_SECRET_ACCESS_KEY
kubectl apply -f k8s/deployment.yml
kubectl get pods
kubectl get service customer-segmentation-service

# ── GitHub ───────────────────────────────────────────────────────
git init && git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/drsuresh8453/customer-segmentation.git
git push --set-upstream origin main

# ── Cleanup ──────────────────────────────────────────────────────
eksctl delete cluster \
    --name customer-segmentation-cluster --region ap-south-1
aws ecr delete-repository \
    --repository-name customer-segmentation --region ap-south-1 --force
```

---

## Complete Flow After Setup

```
ONE TIME SETUP:
  ✅ NB1-NB7 run in Google Colab → model saved to S3
  ✅ ECR repository created
  ✅ Docker image built and pushed
  ✅ EKS cluster running
  ✅ Code pushed to GitHub → CI/CD set up
  ✅ App live at http://EXTERNAL-IP:8501

DAILY USAGE — MARKETING TEAM:
  Open http://EXTERNAL-IP:8501
  → Upload customer CSV
  → Download segmented Excel with AI messages
  → Send via WhatsApp Business API

QUARTERLY:
  Re-run NB7 in Colab → new centroids saved to S3
  App picks them up automatically — no redeployment needed
```

---

**Author: Suresh D R | AI Product Developer & Technology Mentor**
**DV Analytics — Industry ML Projects Series**
