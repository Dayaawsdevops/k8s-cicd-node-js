# <span style="color: #007acc;">⚙️ Kubernetes CI/CD with Github Actions and Helm 🚀</span>

This repository demonstrates a complete CI/CD pipeline for deploying an Express.js application to Amazon EKS using GitHub Actions and Helm charts.

## 🏗️ Architecture Overview
**🐳 Docker** → **🐙 GitHub Actions** → **📦 AWS ECR** → **⎈ Amazon EKS** → **⎈ Helm**

## 📋 **Table of Contents**
* [📋 **Table of Contents**](#table-of-contents)
* [📝 **Prerequisites**](#prerequisites)
* [☁️ **Step 1: Create Amazon EKS Cluster**](#step-1-create-amazon-eks-cluster)
* [🚀 **Step 2: Create an Express Application**](#step-2-create-an-express-application)
  * [🐙 Create Github Repository](#create-github-repository)
  * [📦 Install Express](#install-express)
  * [🐳 Dockerize Express Application](#dockerize-express-application)
  * [📦 Create AWS ECR Registry](#create-aws-ecr-registry)
* [⚙️ **Step 3: Create Deployment Scripts**](#step-3-create-deployment-scripts)
  * [⎈ Create Helm Chart](#create-helm-chart)
  * [🔄 Create Github Actions Workflow](#create-github-actions-workflow)
  * [🔐 Configure Github Action Environment Variable and Secrets](#configure-github-action-environment-variable-and-secrets)
* [✅ **Conclusion**](#conclusion)
* [🗑️ **Delete Resources**](#delete-resources)

---

## 📝 **Prerequisites**

Before starting this tutorial, ensure you have the following:

- ☁️ **AWS CLI** installed and configured with appropriate permissions
- ⎈ **kubectl** installed and configured
- 🐳 **Docker** installed locally
- 🟢 **Node.js** and npm installed
- ⎈ **Helm 3.x** installed
- 🐙 **GitHub** account with repository access
- 🧠 Basic knowledge of Kubernetes, Docker, and CI/CD concepts

**🔐 Required AWS Permissions:**
- ☁️ EKS cluster creation and management
- 📦 ECR repository creation and push/pull access
- 👤 IAM role creation for EKS nodes
- 🌐 VPC and security group management

---

## ☁️ **Step 1: Create Amazon EKS Cluster**

### 🏗️ 1.1 Create EKS Cluster using AWS CLI

```bash
# Create EKS cluster
aws eks create-cluster \
  --name my-eks-cluster \
  --version 1.27 \
  --role-arn arn:aws:iam::ACCOUNT-ID:role/eks-service-role \
  --resources-vpc-config subnetIds=subnet-xxxxx,subnet-yyyyy,securityGroupIds=sg-xxxxx

# Wait for cluster to be active
aws eks wait cluster-active --name my-eks-cluster
```

### 🖥️ 1.2 Create Node Group

```bash
# Create managed node group
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name worker-nodes \
  --subnets subnet-xxxxx subnet-yyyyy \
  --node-role arn:aws:iam::ACCOUNT-ID:role/NodeInstanceRole \
  --ami-type AL2_x86_64 \
  --capacity-type ON_DEMAND \
  --instance-types t3.medium \
  --scaling-config minSize=1,maxSize=3,desiredSize=2
```

### ⚙️ 1.3 Update kubeconfig

```bash
# Update kubeconfig to connect to EKS cluster
aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster

# Verify connection
kubectl get nodes
```

---

## 🚀 **Step 2: Create an Express Application**

### 🐙 Create Github Repository

1. Create a new repository on GitHub
2. Clone the repository locally:
```bash
git clone https://github.com/your-username/your-repo-name.git
cd your-repo-name
```

### 📦 Install Express

1. 🚀 Initialize npm project:
```bash
npm init -y
```

2. 📥 Install Express and dependencies:
```bash
npm install express
npm install --save-dev nodemon
```

3. 📝 Create `app.js`:
```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Express App!',
    version: '1.0.0',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

4. ⚙️ Update `package.json` scripts:
```json
{
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  }
}
```

### 🐳 Dockerize Express Application

📄 Create `Dockerfile`:
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

🚫 Create `.dockerignore`:
```
node_modules
npm-debug.log
Dockerfile
.dockerignore
.git
.gitignore
README.md
.env
coverage
nyc_output
```

🧪 Test Docker build locally:
```bash
docker build -t express-app .
docker run -p 3000:3000 express-app
```

### 📦 Create AWS ECR Registry

1. 🏗️ Create ECR repository:
```bash
aws ecr create-repository \
  --repository-name express-app \
  --region us-west-2
```

2. 🔐 Get login token and authenticate Docker:
```bash
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin ACCOUNT-ID.dkr.ecr.us-west-2.amazonaws.com
```

3. 🚀 Build and push initial image:
```bash
docker build -t express-app .
docker tag express-app:latest ACCOUNT-ID.dkr.ecr.us-west-2.amazonaws.com/express-app:latest
docker push ACCOUNT-ID.dkr.ecr.us-west-2.amazonaws.com/express-app:latest
```

---

## ⚙️ **Step 3: Create Deployment Scripts**

### ⎈ Create Helm Chart

1. 📁 Create Helm chart structure:
```bash
mkdir -p helm/express-app
cd helm/express-app
```

2. 📋 Create `Chart.yaml`:
```yaml
apiVersion: v2
name: express-app
description: Express.js Application Helm Chart
version: 1.0.0
appVersion: "1.0.0"
```

3. ⚙️ Create `values.yaml`:
```yaml
replicaCount: 2

image:
  repository: ACCOUNT-ID.dkr.ecr.us-west-2.amazonaws.com/express-app
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
  targetPort: 3000

ingress:
  enabled: false

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

nodeSelector: {}
tolerations: []
affinity: {}

healthCheck:
  enabled: true
  path: /health
```

4. 🚀 Create `templates/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "express-app.fullname" . }}
  labels:
    {{- include "express-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "express-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "express-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          {{- if .Values.healthCheck.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
            periodSeconds: 5
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

5. 🌐 Create `templates/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "express-app.fullname" . }}
  labels:
    {{- include "express-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
  selector:
    {{- include "express-app.selectorLabels" . | nindent 4 }}
```

6. 🔧 Create `templates/_helpers.tpl`:
```yaml
{{- define "express-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "express-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "express-app.labels" -}}
helm.sh/chart: {{ include "express-app.chart" . }}
{{ include "express-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "express-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "express-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "express-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}
```

### 🔄 Create Github Actions Workflow

📝 Create `.github/workflows/ci-cd.yml`:
```yaml
name: 🚀 CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: us-west-2
  EKS_CLUSTER_NAME: my-eks-cluster
  ECR_REPOSITORY: express-app

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: 📥 Checkout code
      uses: actions/checkout@v3

    - name: ☁️ Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: 🔐 Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: 🐳 Build, tag, and push image to Amazon ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

    - name: ⎈ Install and configure kubectl
      run: |
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        chmod +x kubectl
        sudo mv kubectl /usr/local/bin/
        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME

    - name: ⎈ Install Helm
      uses: azure/setup-helm@v3
      with:
        version: '3.12.0'

    - name: 🚀 Deploy to EKS
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        helm upgrade --install express-app ./helm/express-app \
          --set image.repository=$ECR_REGISTRY/$ECR_REPOSITORY \
          --set image.tag=$IMAGE_TAG \
          --wait

    - name: ✅ Verify deployment
      run: |
        kubectl get pods
        kubectl get services
```

### 🔐 Configure Github Action Environment Variable and Secrets

1. 🐙 Go to your GitHub repository settings
2. 🔧 Navigate to **Secrets and variables** → **Actions**
3. 🔐 Add the following secrets:

**🔑 Repository Secrets:**
```
AWS_ACCESS_KEY_ID: Your AWS Access Key ID
AWS_SECRET_ACCESS_KEY: Your AWS Secret Access Key
```

**⚙️ Environment Variables (Optional):**
```
AWS_REGION: us-west-2
EKS_CLUSTER_NAME: my-eks-cluster
ECR_REPOSITORY: express-app
```

4. 👤 Create IAM user with required permissions:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:DescribeCluster",
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## ✅ **Conclusion**

🎉 You have successfully set up a complete CI/CD pipeline that:

✅ **🐳 Builds** your Express.js application into a Docker container  
✅ **📦 Pushes** the container image to AWS ECR  
✅ **🚀 Deploys** the application to Amazon EKS using Helm  
✅ **🔄 Automates** the entire process with GitHub Actions  

**🔄 What happens on each commit:**
1. 📝 Code is pushed to the main branch
2. 🚀 GitHub Actions triggers the workflow
3. 🐳 Docker image is built and pushed to ECR
4. ⎈ Helm deploys the new version to EKS
5. 🌐 Application is accessible via LoadBalancer service

**🚀 Next Steps:**
- 📊 Set up monitoring with CloudWatch or Prometheus
- 🔄 Implement blue-green or canary deployments
- 🧪 Add automated testing in the CI pipeline
- 🔐 Configure SSL/TLS with cert-manager
- 📈 Set up horizontal pod autoscaling

---

## 🗑️ **Delete Resources**

⚠️ To clean up and avoid AWS charges, delete resources in this order:

### 1. ⎈ Delete Helm Release
```bash
helm uninstall express-app
```

### 2. 🖥️ Delete EKS Node Group
```bash
aws eks delete-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name worker-nodes

# Wait for deletion to complete
aws eks wait nodegroup-deleted \
  --cluster-name my-eks-cluster \
  --nodegroup-name worker-nodes
```

### 3. ☁️ Delete EKS Cluster
```bash
aws eks delete-cluster --name my-eks-cluster

# Wait for deletion to complete
aws eks wait cluster-deleted --name my-eks-cluster
```

### 4. 📦 Delete ECR Repository
```bash
aws ecr delete-repository \
  --repository-name express-app \
  --force
```

### 5. 👤 Delete IAM Roles (if created manually)
```bash
# Delete node group role
aws iam delete-role --role-name NodeInstanceRole

# Delete cluster service role
aws iam delete-role --role-name eks-service-role
```

### 6. ✅ Verify Cleanup
```bash
# Check EKS clusters
aws eks list-clusters

# Check ECR repositories  
aws ecr describe-repositories
```

> ⚠️ **Warning**: Make sure to delete all resources to avoid ongoing AWS charges. EKS clusters and EC2 instances incur hourly costs even when not in use.
