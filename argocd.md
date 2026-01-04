# 🚀 Argo CD End-to-End GitOps Setup
## Complete Guide: KIND Cluster + Argo CD + GitOps Workflow

---

## 📋 Table of Contents

1. [Introduction & Architecture](#introduction)
2. [Prerequisites](#prerequisites)
3. [KIND Cluster Setup](#kind-cluster)
4. [Argo CD Installation](#argocd-installation)
5. [Accessing Argo CD](#accessing-argocd)
6. [GitOps Repository Structure](#gitops-repo)
7. [Creating Applications](#creating-applications)
8. [Complete CI/CD Workflow](#cicd-workflow)
9. [Security Best Practices](#security)
10. [Troubleshooting](#troubleshooting)
11. [Advanced Configuration](#advanced)

---

## 🎯 Introduction & Architecture {#introduction}

### What is GitOps?

**GitOps** is a modern approach to continuous deployment where:
- Git repository is the **single source of truth**
- All infrastructure and application configurations are stored as code
- Changes are made through Git commits and pull requests
- Automated tools sync the desired state from Git to the cluster

### What is Argo CD?

**Argo CD** is a declarative, GitOps continuous delivery tool for Kubernetes that:
- Monitors Git repositories for changes
- Automatically syncs applications to match Git state
- Provides a UI and CLI for managing deployments
- Supports rollback, health monitoring, and drift detection

---

### High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  Developer → Code Change → GitHub Repository    │
│                                                 │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│                                                 │
│  Jenkins CI Pipeline                            │
│  • Build application                            │
│  • Run tests                                    │
│  • Build Docker image                           │
│  • Push to registry                             │
│  • Update k8s-argocd repo (image tag)           │
│                                                 │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│                                                 │
│  GitOps Repository (k8s-argocd)                 │
│  • Kubernetes manifests                         │
│  • Deployment configurations                    │
│  • Service definitions                          │
│  • ConfigMaps & Secrets                         │
│                                                 │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│                                                 │
│  Argo CD (GitOps Controller)                    │
│  • Monitors Git repository                      │
│  • Detects changes automatically                │
│  • Syncs to Kubernetes cluster                  │
│  • Self-healing & auto-pruning                  │
│                                                 │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│                                                 │
│  Kubernetes Cluster (KIND)                      │
│  • Pods running applications                    │
│  • Services exposing apps                       │
│  • ConfigMaps & Secrets                         │
│  • Always synced with Git                       │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

### Key Benefits of This Setup

| Benefit | Description |
|---------|-------------|
| **Declarative** | Entire system state defined in Git |
| **Version Control** | Full history of all changes |
| **Rollback Easy** | Git revert = instant rollback |
| **Audit Trail** | Who changed what, when, and why |
| **Self-Healing** | Manual changes automatically reverted |
| **Multi-Environment** | Same process for dev/staging/prod |

---

## ✅ Prerequisites {#prerequisites}

### Required Tools

```bash
# Check Docker
docker --version
# Should output: Docker version 20.x or higher

# Check kubectl
kubectl version --client
# Should output: Client Version v1.28.x or higher
```

### System Requirements

- **OS**: Linux (Ubuntu/CentOS) or macOS
- **RAM**: Minimum 4GB (8GB recommended)
- **CPU**: 2 cores minimum
- **Disk**: 20GB free space

### Install Missing Tools

**Docker Installation (Ubuntu):**

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

**kubectl Installation:**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**KIND Installation:**

```bash
# For Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version
```

---

## 🐳 KIND Cluster Setup {#kind-cluster}

### What is KIND?

**KIND** (Kubernetes IN Docker) runs local Kubernetes clusters using Docker containers as "nodes". Perfect for:
- Local development and testing
- CI/CD pipelines
- Learning Kubernetes

---

### Step 1: Create Cluster Configuration

Create a file called `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: argocd-demo
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

**Configuration Explained:**

- `control-plane`: Master node that manages the cluster
- `worker` nodes: Run your application workloads
- `extraPortMappings`: Allow external access to services
- `node-labels`: Used for Ingress controller

---

### Step 2: Create the Cluster

```bash
kind create cluster --name argocd-demo --config kind-config.yaml
```

**Expected Output:**

```
Creating cluster "argocd-demo" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-argocd-demo"
```

---

### Step 3: Verify Cluster

```bash
# Check cluster info
kubectl cluster-info --context kind-argocd-demo

# List all nodes
kubectl get nodes

# Expected output:
# NAME                         STATUS   ROLES           AGE   VERSION
# argocd-demo-control-plane    Ready    control-plane   2m    v1.27.3
# argocd-demo-worker           Ready    <none>          2m    v1.27.3
# argocd-demo-worker2          Ready    <none>          2m    v1.27.3
```

---

### Step 4: Verify Core Components

```bash
# Check system pods
kubectl get pods -n kube-system

# All pods should be Running
```

**Troubleshooting:**

If cluster creation fails:

```bash
# Delete existing cluster
kind delete cluster --name argocd-demo

# Recreate
kind create cluster --name argocd-demo --config kind-config.yaml
```

---

## 🎨 Argo CD Installation {#argocd-installation}

### Step 1: Create Namespace

```bash
kubectl create namespace argocd
```

**Why a separate namespace?**
- Isolates Argo CD components
- Easier to manage and monitor
- Better security boundaries

---

### Step 2: Install Argo CD

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**What gets installed:**

| Component | Purpose |
|-----------|---------|
| `argocd-server` | Web UI and API server |
| `argocd-repo-server` | Manages Git repository connections |
| `argocd-application-controller` | Monitors apps and syncs state |
| `argocd-dex-server` | SSO and authentication |
| `argocd-redis` | Caching layer |

---

### Step 3: Wait for Pods to be Ready

```bash
# Watch pods until all are Running
kubectl get pods -n argocd -w

# Press Ctrl+C once all show Running

# OR check status
kubectl get pods -n argocd
```

**Expected Output:**

```
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-application-controller-0       1/1     Running   0          2m
argocd-dex-server-xxx                 1/1     Running   0          2m
argocd-redis-xxx                      1/1     Running   0          2m
argocd-repo-server-xxx                1/1     Running   0          2m
argocd-server-xxx                     1/1     Running   0          2m
```

⏳ **This may take 1-3 minutes**

---

### Step 4: Verify Installation

```bash
# Check all resources
kubectl get all -n argocd

# Check services
kubectl get svc -n argocd
```

---

## 🌐 Accessing Argo CD {#accessing-argocd}

### Method 1: Port Forward (Development)

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443 --address=0.0.0.0
```

**Access the UI:**

```
https://localhost:8080
# OR
https://<YOUR_VM_IP>:8080
```

⚠️ **Certificate Warning**: Browser will show security warning. Click "Advanced" → "Proceed" (this is normal for self-signed certificates)

---

### Method 2: NodePort (Persistent Access)

Edit the service type:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get the NodePort
kubectl get svc argocd-server -n argocd
```

**Access:**

```
https://<NODE_IP>:<NODE_PORT>
```

---

### Method 3: Ingress (Production)

Create `argocd-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
  tls:
    - hosts:
        - argocd.example.com
      secretName: argocd-tls
```

Apply:

```bash
kubectl apply -f argocd-ingress.yaml
```

---

## 🔐 Argo CD Login

### Step 1: Get Admin Username

Default username is always:

```
admin
```

---

### Step 2: Get Initial Admin Password

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Example Output:**

```
Kx8vZ2mP9nQ4rT6s
```

🔒 **IMPORTANT**: 
- Save this password securely
- Change it after first login
- Never commit to Git

---

### Step 3: Login via UI

1. Open browser: `https://localhost:8080`
2. Username: `admin`
3. Password: (from step 2)
4. Click "Sign In"

---

### Step 4: Change Admin Password (Recommended)

In UI:
1. Click "User Info" (top right)
2. Click "Update Password"
3. Enter new password
4. Save

Via CLI:

```bash
argocd account update-password \
  --current-password <OLD_PASSWORD> \
  --new-password <NEW_PASSWORD>
```

---

## 🛠️ Argo CD CLI Installation

### Install CLI

```bash
# Download latest version
curl -sSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Make executable
chmod +x argocd

# Move to PATH
sudo mv argocd /usr/local/bin/

# Verify
argocd version
```

**Expected Output:**

```
argocd: v2.9.3+c7eea24
  BuildDate: 2023-11-28T23:17:51Z
  GitCommit: c7eea2422028bfc2cc4d8bd3d0e91b9cf0f25dc9
  GoVersion: go1.21.4
```

---

### Login via CLI

```bash
argocd login localhost:8080 \
  --username admin \
  --password <YOUR_PASSWORD> \
  --insecure
```

**Flags Explained:**

- `--username`: Admin username
- `--password`: Your admin password
- `--insecure`: Skip TLS verification (for self-signed certs)

**Success Output:**

```
'admin:login' logged in successfully
Context 'localhost:8080' updated
```

---

## 📁 GitOps Repository Structure {#gitops-repo}

### Create GitOps Repository

```bash
# Create new repo
mkdir k8s-argocd
cd k8s-argocd
git init

# Create directory structure
mkdir -p k8s
```

---

### Complete Repository Structure

```
k8s-argocd/
├── README.md
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── ingress.yaml (optional)
└── .gitignore
```

---

### File: `k8s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: register
  labels:
    app: register
    managed-by: argocd
```

**Purpose**: Creates isolated namespace for your application

---

### File: `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: register-app
  namespace: register
  labels:
    app: register
spec:
  replicas: 3
  selector:
    matchLabels:
      app: register
  template:
    metadata:
      labels:
        app: register
    spec:
      containers:
        - name: register
          image: your-registry/register-app:v1.0.0  # Jenkins will update this
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"
          envFrom:
            - configMapRef:
                name: register-config
            - secretRef:
                name: register-secret
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
```

**Key Points:**

- `replicas: 3`: Three pods for high availability
- `image`: This tag will be updated by Jenkins
- `envFrom`: Loads environment variables from ConfigMap and Secret
- `resources`: Prevents resource starvation
- `probes`: Health checks for automatic recovery

---

### File: `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: register-service
  namespace: register
  labels:
    app: register
spec:
  type: ClusterIP
  selector:
    app: register
  ports:
    - name: http
      port: 80
      targetPort: 3000
      protocol: TCP
```

**Purpose**: Exposes your application internally within the cluster

---

### File: `k8s/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: register-config
  namespace: register
data:
  # Non-sensitive configuration
  APP_NAME: "Register Application"
  LOG_LEVEL: "info"
  API_TIMEOUT: "5000"
  MAX_CONNECTIONS: "100"
```

**Purpose**: Stores non-sensitive configuration

---

### File: `k8s/secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: register-secret
  namespace: register
type: Opaque
data:
  # Base64 encoded values
  # echo -n 'your-secret' | base64
  DATABASE_URL: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0BkYjoxMjM=
  JWT_SECRET: bXlzZWNyZXRqd3R0b2tlbg==
  API_KEY: YXBpa2V5MTIzNDU2Nzg5MA==
```

⚠️ **SECURITY WARNING**:
- **Never commit real secrets to Git**
- Use External Secrets Operator or Sealed Secrets
- This is for demonstration only

**To create properly:**

```bash
# Create secret from literal values
kubectl create secret generic register-secret \
  --from-literal=DATABASE_URL='postgresql://user:pass@db:5432/dbname' \
  --from-literal=JWT_SECRET='mySecretToken123' \
  --from-literal=API_KEY='apikey1234567890' \
  --dry-run=client -o yaml > k8s/secret.yaml
```

---

### File: `.gitignore`

```
# Sensitive files
*.key
*.pem
secrets/
.env

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

---

### Commit and Push

```bash
git add .
git commit -m "Initial GitOps configuration"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/k8s-argocd.git
git push -u origin main
```

---

## 🚀 Creating Argo CD Applications {#creating-applications}

### Method 1: Using Argo CD CLI (Recommended)

```bash
argocd app create register-app \
  --repo https://github.com/YOUR_USERNAME/k8s-argocd.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace register \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

**Parameters Explained:**

| Parameter | Purpose |
|-----------|---------|
| `--repo` | Git repository URL (source of truth) |
| `--path` | Folder containing Kubernetes manifests |
| `--dest-server` | Target Kubernetes cluster |
| `--dest-namespace` | Namespace where app will be deployed |
| `--sync-policy automated` | Auto-sync when Git changes |
| `--self-heal` | Revert manual changes automatically |
| `--auto-prune` | Delete resources removed from Git |

---

### Method 2: Using YAML Manifest

Create `argocd-application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: register-app
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/YOUR_USERNAME/k8s-argocd.git
    targetRevision: main
    path: k8s
  
  destination:
    server: https://kubernetes.default.svc
    namespace: register
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Apply:

```bash
kubectl apply -f argocd-application.yaml
```

---

### Method 3: Using Argo CD UI

1. Login to Argo CD UI
2. Click "+ NEW APP"
3. Fill in details:
   - **Application Name**: `register-app`
   - **Project**: `default`
   - **Sync Policy**: `Automatic`
   - **Repository URL**: `https://github.com/YOUR_USERNAME/k8s-argocd.git`
   - **Revision**: `main`
   - **Path**: `k8s`
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `register`
4. Click "CREATE"

---

## 🔍 Verify Application Deployment

### Using CLI

```bash
# List all applications
argocd app list

# Get detailed info
argocd app get register-app

# Watch sync status
argocd app wait register-app --health
```

**Expected Output:**

```
Name:               register-app
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          register
URL:                https://localhost:8080/applications/register-app
Repo:               https://github.com/YOUR_USERNAME/k8s-argocd.git
Target:             main
Path:               k8s
SyncWindow:         Sync Allowed
Sync Policy:        Automated
Sync Status:        Synced to main (abc1234)
Health Status:      Healthy
```

---

### Using kubectl

```bash
# Check namespace
kubectl get ns register

# Check all resources
kubectl get all -n register

# Check pods
kubectl get pods -n register

# Check logs
kubectl logs -n register -l app=register
```

---

### Using Argo CD UI

1. Open Argo CD UI
2. Click on "register-app"
3. View application topology
4. Check sync status: **Synced** ✅
5. Check health status: **Healthy** ✅

---

## 🔄 Complete CI/CD Workflow {#cicd-workflow}

### Full GitOps Flow Diagram

```
┌──────────────────────────────────────────────────────────┐
│  Step 1: Developer Commits Code                          │
│  ─────────────────────────────────────────────────────── │
│  Developer → Push code → GitHub (app-repo)               │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  Step 2: Jenkins CI Pipeline Triggered                   │
│  ─────────────────────────────────────────────────────── │
│  • Checkout code                                         │
│  • Run tests                                             │
│  • Build Docker image                                    │
│  • Tag: your-registry/app:v1.2.3                         │
│  • Push to Docker registry                               │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  Step 3: Jenkins Updates GitOps Repo                     │
│  ─────────────────────────────────────────────────────── │
│  • Clone k8s-argocd repo                                 │
│  • Update deployment.yaml:                               │
│    image: your-registry/app:v1.2.3                       │
│  • Commit and push                                       │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  Step 4: Argo CD Detects Change                          │
│  ─────────────────────────────────────────────────────── │
│  • Monitors k8s-argocd repo (every 3 minutes)            │
│  • Detects new commit                                    │
│  • Compares desired state (Git) vs actual (K8s)          │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  Step 5: Argo CD Syncs to Kubernetes                     │
│  ─────────────────────────────────────────────────────── │
│  • Applies new deployment.yaml                           │
│  • Rolling update:                                       │
│    - Creates new pods with v1.2.3                        │
│    - Waits for health checks                             │
│    - Terminates old pods                                 │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  Step 6: Application Running                             │
│  ─────────────────────────────────────────────────────── │
│  • New version live in Kubernetes                        │
│  • Argo CD shows: Synced ✅ Healthy ✅                    │
│  • Zero downtime deployment                              │
└──────────────────────────────────────────────────────────┘
```

---

### Jenkins Pipeline for Image Update

Create `Jenkinsfile`:

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'your-registry.io'
        IMAGE_NAME = 'register-app'
        GITOPS_REPO = 'https://github.com/YOUR_USERNAME/k8s-argocd.git'
        GITOPS_BRANCH = 'main'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/YOUR_USERNAME/app-repo.git'
            }
        }
        
        stage('Run Tests') {
            steps {
                sh 'npm install'
                sh 'npm test'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "v${env.BUILD_NUMBER}"
                    env.IMAGE_TAG = imageTag
                    sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${imageTag} ."
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-registry-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin ${DOCKER_REGISTRY}"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.IMAGE_TAG}"
                    }
                }
            }
        }
        
        stage('Update GitOps Repo') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh """
                            git clone https://${GIT_TOKEN}@github.com/YOUR_USERNAME/k8s-argocd.git
                            cd k8s-argocd
                            sed -i 's|image: .*|image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.IMAGE_TAG}|' k8s/deployment.yaml
                            git config user.email "jenkins@example.com"
                            git config user.name "Jenkins CI"
                            git add k8s/deployment.yaml
                            git commit -m "Update image to ${env.IMAGE_TAG}"
                            git push origin ${GITOPS_BRANCH}
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "✅ Deployment initiated! Argo CD will sync the changes."
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
```

---

### Testing the Workflow

1. **Make a code change** in your app repository
2. **Push to GitHub**
3. **Jenkins builds** and updates GitOps repo
4. **Argo CD syncs** automatically (within 3 minutes)
5. **Verify** in Argo CD UI or kubectl:

```bash
# Watch rollout
kubectl rollout status deployment/register-app -n register

# Check new image
kubectl get pods -n register -o jsonpath='{.items[0].spec.containers[0].image}'
```

---

## 🔐 Security Best Practices {#security}

### 1. Never Store Secrets in Git

**❌ BAD:**

```yaml
apiVersion: v1
kind: Secret
data:
  password: cGFzc3dvcmQxMjM=  # Easily decoded
```

**✅ GOOD: Use External Secrets Operator**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: register-secret
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: register-secret
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: prod/database
        property: url
```

---

### 2. Use Sealed Secrets

Install Sealed Secrets:

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml
```

Encrypt secrets:

```bash
echo -n 'mypassword' | kubectl create secret generic mysecret \
  --dry-run=client --from-file=password=/dev/stdin -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml
```

---

### 3. RBAC for Argo CD

Create `argocd-rbac-cm.yaml`:

```yaml
apiVersion: v1
