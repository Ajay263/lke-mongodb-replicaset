# MongoDB ReplicaSet with Mongo Express on Linode Kubernetes Engine

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.34+-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![MongoDB](https://img.shields.io/badge/MongoDB-ReplicaSet-47A248?style=flat&logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![Helm](https://img.shields.io/badge/Helm-3.16+-0F1689?style=flat&logo=helm&logoColor=white)](https://helm.sh/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A production-ready guide for deploying a highly available MongoDB ReplicaSet on Linode Kubernetes Engine (LKE) with Mongo Express web UI and NGINX Ingress Controller.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Part 1: Setting Up Linode Kubernetes Cluster](#part-1-setting-up-linode-kubernetes-cluster)
- [Part 2: Local Environment Setup](#part-2-local-environment-setup)
- [Part 3: Deploy MongoDB ReplicaSet](#part-3-deploy-mongodb-replicaset)
- [Part 4: Deploy Mongo Express Web UI](#part-4-deploy-mongo-express-web-ui)
- [Part 5: Install NGINX Ingress Controller](#part-5-install-nginx-ingress-controller)
- [Part 6: Configure Ingress for External Access](#part-6-configure-ingress-for-external-access)
- [Part 7: Access and Verify](#part-7-access-and-verify)
- [Troubleshooting](#troubleshooting)
- [Production Considerations](#production-considerations)
- [Clean Up](#clean-up)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This project provides a comprehensive guide for deploying a production-grade MongoDB ReplicaSet on Linode Kubernetes Engine. The deployment includes:

- **High Availability**: 3-node MongoDB ReplicaSet with automatic failover
- **Quorum Management**: MongoDB Arbiter for election consensus
- **Web Interface**: Mongo Express for database administration
- **External Access**: NGINX Ingress Controller with LoadBalancer
- **Persistent Storage**: Linode Block Storage for data durability

---

## Architecture

![Architecture Diagram](img/k8_architecture.png)

**What will be Deployed?:**
- 3-node MongoDB ReplicaSet with automatic failover
- 1 MongoDB Arbiter for quorum
- Mongo Express web UI for database administration
- NGINX Ingress Controller for external access
- Persistent storage using Linode Block Storage

---

## Prerequisites

- A Linode account ([Sign up here](https://www.linode.com/) - $100 free credit available)
- Credit card for Linode account (required even with free credits)
- Git Bash (Windows), Terminal (macOS/Linux), or PowerShell
- Basic understanding of Kubernetes concepts

**Estimated Monthly Cost:**
- 2x Dedicated 4GB nodes
- 3x Block Storage (10GB each)
- 1x NodeBalancer
- **Total: ~$72/month**

*Note: You can use smaller instances for testing (Shared 2GB at $12/month for 3 nodes = $36 total)*

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/Ajay263/lke-mongodb-replicaset
cd lke-mongodb-replicaset

# Add Helm repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Deploy MongoDB
helm install mongodb --values helm-mongodb.yaml bitnami/mongodb

# Deploy Mongo Express
kubectl apply -f mongo-express.yaml

# Install NGINX Ingress
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --set controller.publishService.enabled=true

# Configure Ingress (update with your LoadBalancer IP)
kubectl apply -f mongo-express-ingress.yaml
```

> **Note**: For detailed step-by-step instructions, continue reading below.

---

## Part 1: Setting Up Linode Kubernetes Cluster

### Step 1.1: Access the Linode Cloud Manager

1. Log in to [https://cloud.linode.com/](https://cloud.linode.com/)
2. You'll see the Linode Cloud Manager dashboard
3. Familiarize yourself with the navigation menu on the left

### Step 1.2: Create a Kubernetes Cluster

#### 1. Navigate to Kubernetes Section:
- In the left sidebar, click **Kubernetes**
- Click the blue **Create Cluster** button

#### 2. Configure Cluster Label and Region:

| Setting | Value | Description |
|---------|-------|-------------|
| **Cluster Label** | `mongodb-cluster` | Or any name you prefer - just a friendly name for identification |
| **Region** | Your choice | Select a region close to you (Examples: `Frankfurt, DE (eu-central)`, `Newark, NJ (us-east)`, `Singapore, SG (ap-south)`) |
| **Kubernetes Version** | `1.34` (latest stable) | Use the default recommended version unless you have specific requirements |

**Tip**: Choose region based on your location for lower latency

#### 3. Add Node Pools:

**What are Node Pools?**
- Groups of worker nodes (virtual machines) that run your containers
- All nodes in a pool have the same specifications

**Recommended Configuration for this project:**

**Option A: Production Setup (Recommended)**
- Click **Add Node Pool**
- Under **Dedicated CPU**, select **Dedicated 4GB**
  - 2 CPU cores
  - 4GB RAM
  - 80GB Storage
- Set **Quantity**: `3`

**Why 2 nodes?**
- Distributes MongoDB pods across different physical machines
- Provides high availability
- Allows for zero-downtime updates
- This might not be the recommended value but for our use case its fine

#### 4. Review Your Configuration:

You should see:
```
Cluster Summary
├── Cluster Label: mongodb-cluster
├── Region: useast
├── Version: 1.34.x
└── Node Pools:
    └── Dedicated 4GB: 2 nodes ($72/month)

Estimated Monthly Cost: $72/month
(includes NodeBalancer costs)
```

#### 5. Create the Cluster:
- Review the monthly cost estimate
- Click the blue **Create Cluster** button at the bottom
- You'll see a progress indicator

#### 6. Wait for Provisioning:

- **Status will show**: "Provisioning"
- This takes approximately **5-10 minutes**
- You'll see each node being created:
  - `lke123456-123456-abcdef123456` - Provisioning → Running
  - `lke123456-123456-abcdef789012` - Provisioning → Running
  - `lke123456-123456-abcdef345678` - Provisioning → Running

**What's happening behind the scenes:**
- Linode is creating 3 virtual machines
- Installing Kubernetes on each node
- Configuring networking between nodes
- Setting up the cluster control plane
- Installing the Container Storage Interface (CSI) for block storage

#### 7. Verify Cluster is Ready:
- Wait until **Status** changes to "Ready"
- All 3 nodes should show **Status: Ready**

### Step 1.3: Download Kubeconfig File

**What is a kubeconfig file?**
- Configuration file that contains:
  - Cluster API endpoint
  - Authentication certificates
  - User credentials
- Required for `kubectl` to connect to your cluster

**Steps:**

1. **From the Kubernetes page**, click on your cluster name (`mongodb-cluster`)

2. **You'll see the cluster details page showing:**
   - Cluster endpoint URL
   - Node pool information
   - Kubernetes version
   - Created date

3. **Download the kubeconfig:**
   - Look for the **Kubeconfig** section at the top
   - Click the **Download kubeconfig** button
   - Your browser will download a file named something like:
     - `mongodb-cluster-kubeconfig.yaml`
     - `kubeconfig.yaml`
   - **Save this file to your Downloads folder**

4. **Important Security Note:**
   - This file contains cluster admin credentials
   - **Keep it secure** - don't share it or commit to Git
   - Anyone with this file has full access to your cluster

**Screenshot reference locations:**
```
Cloud Manager
├── Kubernetes (left sidebar)
│   └── mongodb-cluster
│       ├── Kubeconfig [Download button here]
│       ├── API Endpoint
│       ├── Node Pools
│       │   └── pool-1 (3 nodes)
│       └── Summary
```

---

## Part 2: Local Environment Setup

### Step 2.1: Install kubectl

**What is kubectl?**
- Command-line tool to interact with Kubernetes clusters
- Sends commands to the cluster's API server
- Required for all cluster operations

<details>
<summary><b>Windows Installation (Git Bash)</b></summary>

1. **Open Git Bash**

2. **Navigate to Downloads:**
   ```bash
   cd ~/Downloads
   ls  # Verify you can see your kubeconfig file
   ```

3. **Download kubectl binary:**
   ```bash
   curl.exe -LO "https://dl.k8s.io/release/v1.34.0/bin/windows/amd64/kubectl.exe"
   ```

4. **Optional but recommended - Verify the download:**
   ```bash
   curl.exe -LO "https://dl.k8s.io/v1.34.0/bin/windows/amd64/kubectl.exe.sha256"
   ```
   
   Then open PowerShell and run:
   ```powershell
   cd Downloads
   $(Get-FileHash -Algorithm SHA256 .\kubectl.exe).Hash -eq $(Get-Content .\kubectl.exe.sha256)
   # Should return: True
   ```

5. **Create a bin directory:**
   ```bash
   mkdir -p ~/bin
   ```

6. **Move kubectl to bin:**
   ```bash
   mv kubectl.exe ~/bin/
   ```

7. **Add to Windows PATH**

8. **Close and reopen Git Bash**

9. **Verify installation:**
   ```bash
   kubectl version --client
   
   # Expected output:
   # Client Version: v1.34.0
   ```
</details>

<details>
<summary><b>macOS Installation</b></summary>

```bash
# Using Homebrew (easiest)
brew install kubectl

# Verify
kubectl version --client
```
</details>

<details>
<summary><b>Linux Installation (Ubuntu/Debian)</b></summary>

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```
</details>

### Step 2.2: Configure kubectl to Access Your Cluster

1. **Navigate to Downloads folder:**
   ```bash
   cd ~/Downloads
   ls  # You should see your kubeconfig file
   ```

2. **Set appropriate permissions (Unix-like systems):**
   ```bash
   # For macOS/Linux
   chmod 400 mongodb-cluster-kubeconfig.yaml
   
   # For Windows Git Bash (optional, but recommended)
   chmod 400 mongodb-cluster-kubeconfig.yaml
   ```
   
   **Why?** Restricts file access to only you (owner read-only)

3. **Set KUBECONFIG environment variable:**
   ```bash
   export KUBECONFIG=mongodb-cluster-kubeconfig.yaml
   ```
   
   **What this does:**
   - Tells kubectl which cluster configuration to use
   - Points to your specific cluster credentials
   
   **Important:** This only works for the current terminal session!

4. **Test connection to your cluster:**
   ```bash
   kubectl get nodes
   ```
   
   **Expected output:**
   ```
   NAME                            STATUS   ROLES    AGE   VERSION
   lke123456-123456-abcdef123456   Ready    <none>   15m   v1.28.3
   lke123456-123456-abcdef789012   Ready    <none>   15m   v1.28.3
   lke123456-123456-abcdef345678   Ready    <none>   15m   v1.28.3
   ```
   
   **If you see this, congratulations!** You're connected to your cluster! 🎉

5. **Check cluster info:**
   ```bash
   kubectl cluster-info
   
   # Output shows:
   # Kubernetes control plane is running at https://xxxxx.linodelke.net:443
   # CoreDNS is running at https://xxxxx.linodelke.net:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
   ```

6. **View default namespaces:**
   ```bash
   kubectl get namespaces
   
   # Expected output:
   # NAME              STATUS   AGE
   # default           Active   20m
   # kube-node-lease   Active   20m
   # kube-public       Active   20m
   # kube-system       Active   20m
   ```

### Step 2.3: Install Helm

**What is Helm?**
- Package manager for Kubernetes (like apt for Ubuntu, brew for macOS)
- Manages complex applications with "Charts"
- Simplifies deployment and upgrades
- We'll use it to install MongoDB

<details>
<summary><b>Windows Installation</b></summary>

**Option 1: Manual Download (Recommended for Git Bash)**

```bash
cd ~/Downloads

# Download Helm
curl -fsSL -o helm.zip https://get.helm.sh/helm-v3.16.2-windows-amd64.zip

# Extract (if unzip is available in Git Bash)
unzip helm.zip

# Move helm.exe to bin directory
mv windows-amd64/helm.exe ~/bin/

# Clean up
rm -rf windows-amd64 helm.zip

# Verify
helm version
```

**Option 2: Using Chocolatey (if installed)**

```bash
choco install kubernetes-helm
helm version
```

**Option 3: Using Scoop (if installed)**

```bash
scoop install helm
helm version
```
</details>

<details>
<summary><b>macOS Installation</b></summary>

```bash
brew install helm
helm version
```
</details>

<details>
<summary><b>Linux Installation</b></summary>

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```
</details>

### Step 2.4: Verify Your Setup

Run these commands to confirm everything is working:

```bash
# 1. Check kubectl
kubectl version --client

# 2. Check cluster connection
kubectl get nodes

# 3. Check Helm
helm version

# 4. Check current context
kubectl config current-context

# 5. View all cluster resources
kubectl get all --all-namespaces
```

**If all commands work, you're ready to deploy!** ✅

---

## Part 3: Deploy MongoDB ReplicaSet

### Step 3.0: Clone the Repository

```bash
git clone https://github.com/Ajay263/lke-mongodb-replicaset
cd lke-mongodb-replicaset
```

### Step 3.1: Add Bitnami Helm Repository

**What is Bitnami?**
- Company providing pre-packaged, production-ready applications
- Maintains high-quality Helm charts
- MongoDB chart includes best practices and security configurations

```bash
# Add the repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Step 3.2: Create MongoDB Configuration File

**Why use a values file?**
- Customizes default Helm chart settings
- Makes configuration reusable and version-controllable
- Easier to track changes

1. **Create the file:**
   ```bash
   # Using nano (Git Bash/Linux/macOS)
   nano helm-mongodb.yaml
   ```
   
   Or use your preferred text editor (VS Code, Notepad++, etc.)

2. **Add this configuration:**
   ```yaml
   architecture: replicaset
   replicaCount: 3
   persistence:
     storageClass: "linode-block-storage"
   auth:
     rootPassword: secret-root-pwd
   ```

**Configuration Explained:**

| Setting | Value | Purpose |
|---------|-------|---------|
| `architecture` | `replicaset` | Deploys MongoDB in high-availability mode with multiple replicas |
| `replicaCount` | `3` | Creates 3 MongoDB data nodes for redundancy |
| `storageClass` | `linode-block-storage` | Uses Linode's persistent block storage (survives pod restarts) |
| `rootPassword` | `secret-root-pwd` | Sets MongoDB admin password (⚠️ change for production!) |

### Step 3.3: Deploy MongoDB Using Helm

```bash
# Deploy with your custom values
helm install mongodb --values helm-mongodb.yaml bitnami/mongodb
```

**Command breakdown:**
- `helm install` - Install a new Helm release
- `mongodb` - Name of your release (you choose this)
- `--values helm-mongodb.yaml` - Use your custom configuration
- `bitnami/mongodb` - Chart to install (repository/chartname)

### Step 3.4: Monitor the Deployment

**Watch pods being created (real-time updates):**
```bash
kubectl get pods --watch
```

**You'll see this progression:**

```
# Initial state (0-30 seconds)
NAME                READY   STATUS     RESTARTS   AGE
mongodb-0           0/1     Init:0/1   0          5s
mongodb-arbiter-0   0/1     Pending    0          5s

# Containers starting (30-60 seconds)
mongodb-0           0/1     Running    0          35s
mongodb-arbiter-0   0/1     Running    0          35s

# First replica ready (60-90 seconds)
mongodb-0           1/1     Running    0          70s
mongodb-arbiter-0   1/1     Running    0          70s
mongodb-1           0/1     Pending    0          1s

# Second replica starting (90-120 seconds)
mongodb-1           0/1     Init:0/1   0          20s
mongodb-1           0/1     Running    0          45s
mongodb-1           1/1     Running    0          80s

# Third replica starting (120-180 seconds)
mongodb-2           0/1     Pending    0          1s
mongodb-2           0/1     Init:0/1   0          15s
mongodb-2           0/1     Running    0          40s
mongodb-2           1/1     Running    0          75s
```

**Press `Ctrl+C` when all pods show `1/1 Running`**

**What's happening during deployment:**

1. **Init:0/1** - Running initialization containers
   - Setting up data directories
   - Configuring MongoDB settings
   - Preparing for replication

2. **Running** - Main container started
   - MongoDB process starting
   - Loading configuration
   - Joining replica set

3. **1/1 Running** - Pod is healthy and ready
   - MongoDB accepting connections
   - Replication configured
   - Health checks passing

### Step 3.5: Verify MongoDB Deployment

```bash
# View all resources created by Helm
kubectl get all

# Expected output:
NAME                    READY   STATUS    RESTARTS   AGE
pod/mongodb-0           1/1     Running   0          3m
pod/mongodb-1           1/1     Running   0          2m
pod/mongodb-2           1/1     Running   0          1m
pod/mongodb-arbiter-0   1/1     Running   0          3m

NAME                               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/kubernetes                 ClusterIP   10.128.0.1   <none>        443/TCP     20m
service/mongodb-arbiter-headless   ClusterIP   None         <none>        27017/TCP   3m
service/mongodb-headless           ClusterIP   None         <none>        27017/TCP   3m

NAME                               READY   AGE
statefulset.apps/mongodb           3/3     3m
statefulset.apps/mongodb-arbiter   1/1     3m
```

**Understanding the resources:**

- **Pods**: Individual running containers
  - `mongodb-0`, `mongodb-1`, `mongodb-2`: Data-storing replicas
  - `mongodb-arbiter-0`: Voting member (no data)

- **Services**: Network endpoints
  - `mongodb-headless`: DNS for individual pods
  - No load balancing (ClusterIP: None)

- **StatefulSets**: Manages pod lifecycle
  - Ensures ordered deployment
  - Provides stable network identities
  - Manages persistent storage

---

## Part 4: Deploy Mongo Express Web UI

This uses the file named `mongo-express.yaml`

**Configuration Explanation:**
- **Deployment**: Runs 1 replica of Mongo Express
- **Environment Variables**:
  - `ME_CONFIG_MONGODB_ADMINUSERNAME`: MongoDB username (root)
  - `ME_CONFIG_MONGODB_SERVER`: Points to mongodb-0 pod via headless service
  - `ME_CONFIG_MONGODB_ADMINPASSWORD`: Securely references password from Kubernetes Secret
- **Service**: Creates internal endpoint on port 8081

### Step 4.1: Deploy Mongo Express

```bash
kubectl apply -f mongo-express.yaml
```

### Step 4.2: Verify Deployment

```bash
# Check if Mongo Express is running
kubectl get pods | grep mongo-express

# Expected output:
# mongo-express-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

# Check service
kubectl get svc mongo-express-service

# Expected output:
# NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# mongo-express-service    ClusterIP   10.128.44.229   <none>        8081/TCP   1m
```

---

## Part 5: Install NGINX Ingress Controller

### Step 5.1: Add NGINX Ingress Helm Repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Step 5.2: Install NGINX Ingress Controller

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --set controller.publishService.enabled=true
```

### Step 5.3: Wait for LoadBalancer IP

```bash
# Watch for external IP (takes 2-3 minutes)
kubectl get service nginx-ingress-ingress-nginx-controller --watch

# Expected progression:
# NAME                                     TYPE           EXTERNAL-IP   PORT(S)
# nginx-ingress-ingress-nginx-controller   LoadBalancer   <pending>     80:31596/TCP,443:31134/TCP
# nginx-ingress-ingress-nginx-controller   LoadBalancer   170.x.x.x     80:31596/TCP,443:31134/TCP

# Press Ctrl+C once you see the EXTERNAL-IP
```

### Step 5.4: Verify All Services

```bash
kubectl get services

# Expected output:
# NAME                                               TYPE           EXTERNAL-IP       PORT(S)
# kubernetes                                         ClusterIP      10.128.0.1        443/TCP
# mongo-express-service                              ClusterIP      10.128.44.229     8081/TCP
# mongodb-arbiter-headless                           ClusterIP      None              27017/TCP
# mongodb-headless                                   ClusterIP      None              27017/TCP
# nginx-ingress-ingress-nginx-controller             LoadBalancer   170.187.131.161   80:31596/TCP,443:31134/TCP
# nginx-ingress-ingress-nginx-controller-admission   ClusterIP      10.128.78.109     443/TCP
```

---

## Part 6: Configure Ingress for External Access

### Step 6.1: Create Ingress Configuration

Create a file named `mongo-express-ingress.yaml`:

Replace `170-187-131-161.ip.linodeusercontent.com` with your actual LoadBalancer IP in the format shown.

### Step 6.2: Apply Ingress Configuration

```bash
kubectl apply -f mongo-express-ingress.yaml
```

### Step 6.3: Verify Ingress

```bash
# Check Ingress status
kubectl get ingress

# Expected output:
# NAME            CLASS    HOSTS                                        ADDRESS           PORTS   AGE
# mongo-express   <none>   170-187-131-161.ip.linodeusercontent.com     170.187.131.161   80      30s

# Get detailed information
kubectl describe ingress mongo-express
```

---

## Part 7: Access and Verify

### Step 7.1: Open Mongo Express in Browser

**Using hostname:**
```
http://170-187-131-161.ip.linodeusercontent.com
```

*(Replace with your actual LoadBalancer IP)*

### Step 7.2: Explore Your MongoDB Cluster

You should see the Mongo Express dashboard with:
- Database list
- Connection information
- Ability to create/view/edit databases and collections

### Step 7.3: Test MongoDB Connection

In Mongo Express, you can:
1. Create a new database (click "Create Database")
2. Create collections
3. Insert documents
4. Run queries

---

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods

# View pod logs
kubectl logs <pod-name>

# Describe pod for events
kubectl describe pod <pod-name>
```

### Connection Issues

```bash
# Test MongoDB connectivity from within cluster
kubectl run -it --rm debug --image=mongo:latest --restart=Never -- \
  mongosh mongodb://root:secret-root-pwd@mongodb-0.mongodb-headless:27017

# Check service endpoints
kubectl get endpoints
```

### Ingress Not Working

```bash
# Check Ingress status
kubectl describe ingress mongo-express

# Check NGINX Ingress logs
kubectl logs -n default -l app.kubernetes.io/name=ingress-nginx

# Verify LoadBalancer service
kubectl get svc nginx-ingress-ingress-nginx-controller
```

### Storage Issues

```bash
# Check Persistent Volume Claims
kubectl get pvc

# Describe PVC for details
kubectl describe pvc <pvc-name>

# Check Persistent Volumes
kubectl get pv
```

---

## Production Considerations

### Best Practices for Production

1. **Change Default Password**: Never use `secret-root-pwd` in production
2. **Enable TLS/HTTPS**: Use cert-manager with Let's Encrypt
3. **Add Authentication**: Configure basic auth for Mongo Express
4. **Set Resource Limits**: Define CPU/memory limits for all containers
5. **Enable Monitoring**: Install Prometheus and Grafana
6. **Configure Backups**: Set up automated MongoDB backups
7. **Use NetworkPolicies**: Restrict pod-to-pod communication
8. **Update Images**: Use specific version tags, not `latest`
9. **Implement RBAC**: Configure proper Role-Based Access Control
10. **Test Failover**: Regularly test replica set failover scenarios

### Architecture Components

#### MongoDB ReplicaSet
- **Primary Node**: Accepts all write operations
- **Secondary Nodes**: Replicate data, can serve reads
- **Arbiter**: Participates in elections, doesn't store data
- **Automatic Failover**: If primary fails, a secondary is elected

#### Services
- **Headless Services**: Provide direct DNS to individual pods (mongodb-headless)
- **ClusterIP Services**: Internal load-balanced endpoints (mongo-express-service)
- **LoadBalancer Service**: External access with Linode NodeBalancer (nginx-ingress)

#### Storage
- **Persistent Volumes**: Backed by Linode Block Storage
- **Stateful Deployment**: Data survives pod restarts
- **Per-Pod Storage**: Each MongoDB pod has its own persistent volume

#### Ingress
- **NGINX Controller**: Routes HTTP/HTTPS traffic
- **Host-based Routing**: Direct traffic based on hostname
- **Path-based Routing**: Route to different services based on URL path

### Security Considerations

⚠️ **Important Security Notes:**

1. **Mongo Express is publicly accessible** - Consider:
   - Adding basic authentication
   - Restricting by IP address
   - Only exposing temporarily for admin tasks
   - Using a VPN for access

2. **MongoDB is not directly exposed** - It's only accessible within the cluster via:
   - `mongodb-0.mongodb-headless.default.svc.cluster.local:27017`
   - This is good! Keep it this way.

3. **Passwords in Kubernetes Secrets** - While better than plain text:
   - Secrets are base64 encoded, not encrypted by default
   - Consider using external secret management (HashiCorp Vault, AWS Secrets Manager)
   - Enable encryption at rest in etcd

4. **Network Policies** - Consider implementing:
   ```bash
   # Example: Only allow Mongo Express to access MongoDB
   kubectl apply -f network-policy.yaml
   ```

---

## Clean Up

### Remove Application Components

```bash
# Delete Ingress
kubectl delete ingress mongo-express

# Delete Mongo Express
kubectl delete -f mongo-express.yaml

# Delete NGINX Ingress Controller
helm uninstall nginx-ingress

# Delete MongoDB
helm uninstall mongodb

# Delete Persistent Volume Claims (data will be lost!)
kubectl delete pvc --all

# Verify all resources are deleted
kubectl get all
```

### Delete Kubernetes Cluster

To delete the entire Linode Kubernetes cluster:
1. Go to Linode Cloud Manager
2. Navigate to Kubernetes
3. Click on your cluster
4. Click "Delete Cluster"
5. Confirm deletion

**Warning:** This will permanently delete all data and cannot be undone!

---

## Learning Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [MongoDB ReplicaSet Documentation](https://docs.mongodb.com/manual/replication/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Linode Kubernetes Engine](https://www.linode.com/products/kubernetes/)

---

## Contributing

Contributions are welcome! Please follow these guidelines:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Guidelines

- Follow Kubernetes best practices
- Test changes in a development cluster
- Update documentation for new features
- Add troubleshooting steps for common issues

---

## Support

For issues or questions:
- **Linode Support**: https://www.linode.com/support/
- **Kubernetes Community**: https://kubernetes.io/community/
- **MongoDB Community**: https://community.mongodb.com/
- **GitHub Issues**: [Report an issue](https://github.com/Ajay263/lke-mongodb-replicaset/issues)

---

## License

This guide is provided as-is for educational purposes.

---

<div align="center">

**⭐ If you find this project helpful, please consider giving it a star!**

**Built with ❤️ by someone learning Kubernetes hands-on!**

*Last Updated: November 2025*

</div>
