# Complete Step-by-Step Guide: Production Kubernetes on AWS

### What you will build:

```
Your Laptop
    │
    ▼ SSH
Bastion Host ── (public subnet) ──────────────────────────────────┐
    │                                                             │
    │ ProxyJump SSH                                               │
    ▼                                                             │
┌─────────────────────────────────────────────────────────────┐   │
│                   PRIVATE SUBNETS                           │   │
│                                                             │   │
│  Internal NLB (port 6443) ◄─── kubectl from bastion         │   │
│       │                                                     │   │
│       ├──────────────┬──────────────┐                       │   │
│       ▼              ▼              ▼                       │   │
│  control-plane-1  control-plane-2  control-plane-3          │   │
│  (etcd leader)    (etcd follower)  (etcd follower)          │   │
│                                                             │   │
│  worker-1                worker-2                           │   │
│  ├── Calico (CNI)         ├── Calico (CNI)                  │   │
│  ├── Envoy Gateway        └── Envoy Gateway                 │   │
│  └── EBS CSI Driver                                         │   │
└─────────────────────────────────────────────────────────────┘   │
                                                                  │
Internet → AWS Public NLB (80/443) → Envoy Gateway → Apps ────────┘
```

### IP Address Plan (substitute your real IPs as you go)

```
VPC CIDR:             10.0.0.0/16
Public Subnet 1a:     10.0.0.0/24   → Bastion host
Public Subnet 1b:     10.0.1.0/24   → Reserved
Private Subnet 1a:    10.0.10.0/24  → CP1, CP2, Worker-1
Private Subnet 1b:    10.0.11.0/24  → CP3, Worker-2

Pod Network CIDR:     192.168.0.0/16  (Calico — no overlap with VPC)
Service Network:      10.96.0.0/12    (Kubernetes default)
```

### Time & Cost Estimate

```
Time:  3-4 hours first time
Cost:  ~$14-16/day if running 24x7
       ~$5/day if you stop EC2s when not using

ALWAYS stop EC2 instances from AWS Console when done for the day.
```

### PHASE 1 — AWS Infrastructure

**Step 1.1 — Create VPC (Everything in One Click)**

Go to: AWS Console → VPC → Create VPC
At the top, select "VPC and more" (not just "VPC only").
Fill in exactly:

```
Name tag auto-generation:  k8s-prod
IPv4 CIDR:                 10.0.0.0/16
IPv6 CIDR:                 No IPv6 CIDR block
Tenancy:                   Default
Number of AZs:             2
First AZ:                  us-east-1a
Second AZ:                 us-east-1b
Public subnets:            2
Private subnets:           2
NAT gateways:              In 1 AZ   ← (saves cost vs 1 per AZ)
VPC endpoints:             None
DNS hostnames:             Enable ✓
DNS resolution:            Enable ✓
```

Click **Create VPC**. Wait 3 minutes for NAT Gateway to provision.
After creation:
Go to **VPC → Subnets**, write down the names AWS gave your subnets:

```
Public subnet in 1a:   ________________________  (call it public-1a)
Public subnet in 1b:   ________________________  (call it public-1b)
Private subnet in 1a:  ________________________  (call it private-1a)
Private subnet in 1b:  ________________________  (call it private-1b)
```

**Step 1.2 — Create IAM Roles (So Nodes Can Talk to AWS APIs)**

Role 1 — **Control Plane Role**
Go to: **IAM → Roles → Create role**

```
Trusted entity:    AWS service
Use case:          EC2
Click Next

Search and add these policies:
  ✓ AmazonEC2FullAccess
  ✓ ElasticLoadBalancingFullAccess

Click Next
Role name:   k8s-control-plane-role
Create role
```

Role 2 — **Worker Node Role**
Go to: **IAM → Roles → Create role**

```
Trusted entity:    AWS service
Use case:          EC2
Click Next

Search and add:
  ✓ AmazonEC2ContainerRegistryReadOnly
  ✓ AmazonEBSCSIDriverPolicy

Click Next
Role name:   k8s-worker-role
Create role
```

**Step 1.3 — Create Security Groups (4 Groups)**
Go to: EC2 → Security Groups → Create security group

⚠️ Read this before starting: Two SGs reference each other (control-plane and workers). Create all 4 SGs with just a name first, then go back and add rules after all 4 exist.

Strategy:

Create all 4 SGs with just names/VPC — no rules yet
Then edit each one and add the inbound rules

**Create SG 1: k8s-sg-bastion**

```
Security group name:  k8s-sg-bastion
Description:          Bastion host SSH access
VPC:                  k8s-prod-vpc
Create (no rules yet)
```

**Create SG 2: k8s-sg-nlb**

```
Security group name:  k8s-sg-nlb
Description:          Internal NLB for API server
VPC:                  k8s-prod-vpc
Create (no rules yet)
```

**Create SG 3: k8s-sg-control-plane**

```
Security group name:  k8s-sg-control-plane
Description:          Kubernetes control plane nodes
VPC:                  k8s-prod-vpc
Create (no rules yet)
```

**Create SG 4: k8s-sg-workers**

```
Security group name:  k8s-sg-workers
Description:          Kubernetes worker nodes
VPC:                  k8s-prod-vpc
Create (no rules yet)
```

Now go back and edit each SG to add inbound rules. 
Click the **SG name → Edit inbound rules**.

**k8s-sg-bastion — Inbound Rules:**

```
Type    Protocol    Port    Source
SSH     TCP         22      Custom → YOUR_HOME_IP/32 (e.g. 203.0.113.5/32)
```

**k8s-sg-nlb — Inbound Rules:**

```
Type	    Protocol	Port	Source
CustomTCP	TCP	6443	Custom → k8s-sg-bastion
CustomTCP	TCP	6443	Custom → k8s-sg-workers
CustomTCP	TCP	6443	Custom → k8s-sg-control-plane
```

**k8s-sg-control-plane — Inbound Rules:**

```
Type	    Protocol	Port	    Source
Custom TCP	TCP	        6443	    Custom → k8s-sg-nlb
Custom TCP	TCP	        6443	    Custom → k8s-sg-workers
Custom TCP	TCP	        6443	    Custom → k8s-sg-control-plane
Custom TCP	TCP	        2379-2380	Custom → k8s-sg-control-plane
Custom TCP	TCP	        10250	    Custom → k8s-sg-control-plane
Custom TCP	TCP	        10250	    Custom → k8s-sg-workers
Custom TCP	TCP	        10259	    Custom → k8s-sg-control-plane
Custom TCP	TCP	        10257	    Custom → k8s-sg-control-plane
Custom TCP	TCP	        5473	    Custom → k8s-sg-control-plane
Custom TCP	TCP	        5473	    Custom → k8s-sg-workers
Custom TCP	TCP	        179	        Custom → k8s-sg-control-plane
Custom TCP	TCP	        179	        Custom → k8s-sg-workers
Custom UDP	UDP	        4789	    Custom → k8s-sg-control-plane
Custom UDP	UDP	        4789	    Custom → k8s-sg-workers
SSH	        TCP	        22	        Custom → k8s-sg-bastion
```

**k8s-sg-workers — Inbound Rules:**

```
Type	    Protocol	Port	        Source
Custom TCP	TCP	        10250	        Custom → k8s-sg-control-plane
Custom TCP	TCP	        10250	        Custom → k8s-sg-workers
Custom TCP	TCP	        10256	        Custom → k8s-sg-control-plane
Custom TCP	TCP	        30000-32767	    Custom → k8s-sg-nlb
Custom UDP	UDP	        30000-32767	    Custom → k8s-sg-nlb
Custom TCP	TCP	        5473	        Custom → k8s-sg-control-plane
Custom TCP	TCP	        179	            Custom → k8s-sg-control-plane
Custom UDP	UDP	        4789	        Custom → k8s-sg-control-plane
Custom UDP	UDP	        4789	        Custom → k8s-sg-workers
SSH	        TCP	        22	            Custom → k8s-sg-bastion
```

**Step 1.4 — Create an SSH Key Pair**
Go to: **EC2 → Key Pairs → Create key pair**

```
Name:          k8s-key
Key pair type: RSA
File format:   .pem
Create key pair
```

**Step 1.5 — Launch All EC2 Instances (6 total)**

Go to: EC2 → Instances → Launch Instances
Launch them one at a time with these exact settings:

**Bastion Host**

```
Name:                    bastion
AMI:                     Ubuntu Server 24.04 LTS (64-bit x86)
                         (Search "Ubuntu 24" in the AMI search box)
Instance type:           t3.micro
Key pair:                k8s-key

Network settings (click Edit):
  VPC:                   k8s-prod-vpc
  Subnet:                public-1a
  Auto-assign public IP: Enable ✓
  Security group:        k8s-sg-bastion

Storage:
  8 GB   gp3

Advanced details:
  IAM instance profile:  (leave blank)

Launch
```

**Control Plane 1**

```
Name:                    control-plane-1
AMI:                     Ubuntu Server 24.04 LTS
Instance type:           m7i-flex.large
Key pair:                k8s-key

Network settings:
  VPC:                   k8s-prod-vpc
  Subnet:                private-1a
  Auto-assign public IP: Disable ✗
  Security group:        k8s-sg-control-plane

Storage:
  50 GB  gp3

Advanced details:
  IAM instance profile:  k8s-control-plane-role

Launch
```

**Control Plane 2 — same as CP1, same private-1a subnet**

```
Name:     control-plane-2
Subnet:   private-1a
All other settings: identical to control-plane-1
```

**Control Plane 3 — different subnet for AZ diversity**

```
Name:     control-plane-3
Subnet:   private-1b           ← different AZ
All other settings: identical to control-plane-1
```

**Worker 1**

```
Name:                    worker-1
AMI:                     Ubuntu Server 24.04 LTS
Instance type:           c7i-flex.large
Key pair:                k8s-key

Network settings:
  VPC:                   k8s-prod-vpc
  Subnet:                private-1a
  Auto-assign public IP: Disable ✗
  Security group:        k8s-sg-workers

Storage:
  100 GB  gp3

Advanced details:
  IAM instance profile:  k8s-worker-role

Launch
```

**Worker 2 — different subnet**

```
Name:     worker-2
Subnet:   private-1b           ← different AZ
All other settings: identical to worker-1
```

**Write down all Private IPs now**. Go to EC2 → Instances, click each instance:

```
Bastion Public IP:    ______________________  (visible as Public IPv4)
control-plane-1:      ______________________  (Private IPv4)
control-plane-2:      ______________________  (Private IPv4)
control-plane-3:      ______________________  (Private IPv4)
worker-1:             ______________________  (Private IPv4)
worker-2:             ______________________  (Private IPv4)
```

**Step 1.6 — Create the Network Load Balancer (HA API Server)**

First, create the **Target Group**:
Go to: **EC2 → Target Groups → Create target group**

```
Target type:           Instances
Target group name:     k8s-api-tg
Protocol:              TCP
Port:                  6443
VPC:                   k8s-prod-vpc

Health check settings:
  Protocol:            TCP
  Port:                traffic port (6443)
  Healthy threshold:   2
  Unhealthy threshold: 2
  Interval:            10 seconds

Click Next

Register targets:
  ✓ Select control-plane-1
  ✓ Select control-plane-2
  ✓ Select control-plane-3
  Click "Include as pending below"

Create target group
```

****⚠️ Targets will show Unhealthy until kubeadm init runs. This is normal.****

**Now create the NLB:**

Go to: **EC2 → Load Balancers → Create load balancer → Network Load Balancer**

```
Load balancer name:    k8s-api-nlb
Scheme:                Internal           ← NOT internet-facing
IP address type:       IPv4

Network mapping:
  VPC:                 k8s-prod-vpc
  Mappings:
    ✓ us-east-1a → private-1a subnet
    ✓ us-east-1b → private-1b subnet

Security groups:       k8s-sg-nlb

Listeners and routing:
  Protocol: TCP   Port: 6443   Forward to: k8s-api-tg

Create load balancer
```

After creation, go to the **NLB → Details tab** → copy the **DNS name**:

```
NLB DNS:   k8s-api-nlb-xxxxxxxxxxxxxxxx.elb.us-east-1.amazonaws.com

Save this. You'll need it in Phase 4.
```

**VPC Endpoint**

Pods call AWS APIs privately within the VPC — no internet, no NAT needed.

Go to: **AWS Console → VPC → Endpoints → Create Endpoint**
**Endpoint 1: EC2 API (required for EBS CSI)**

```
Service category:  AWS services
Service name:      com.amazonaws.us-east-1.ec2  (search "ec2" and select it)
VPC:               k8s-prod-vpc
Subnets:           ✓ private-1a   ✓ private-1b
Security group:    Create new SG:
                     Inbound: HTTPS 443 from 10.0.0.0/16
                              HTTPS 443 from 192.168.0.0/16  ← pod CIDR
                     Outbound: All traffic
Policy:            Full access
Create endpoint
```

**Endpoint 2: STS (required for IAM auth)**

```
Service category:  AWS services
Service name:      com.amazonaws.us-east-1.sts  (search "com.amazonaws.us-east-1.sts" and select it)
VPC:               k8s-prod-vpc
Subnets:           ✓ private-1a   ✓ private-1b
Security group:    Create new SG:
                     Inbound: HTTPS 443 from 10.0.0.0/16
                              HTTPS 443 from 192.168.0.0/16  ← pod CIDR
                     Outbound: All traffic
Policy:            Full access
Create endpoint
```

**Endpoint 3: S3 (recommended — free gateway endpoint)**

```
Service category:  AWS services
Service name:      com.amazonaws.us-east-1.s3
Type:              Gateway  ← different type, no cost
VPC:               k8s-prod-vpc
Route tables:      ✓ select both private route tables
Create endpoint
```

**PHASE 2 — Set Up SSH Access**

Step 2.1 — Configure SSH on Your Laptop
Set permissions on your key:

```
chmod 400 ~/Downloads/k8s-key.pem

# Create SSH config (~/.ssh/config) — open with any text editor:

nano ~/.ssh/config

# Paste this, replacing all the IP placeholders with your real IPs:

# Bastion — direct access from laptop
Host bastion
  HostName <BASTION_PUBLIC_IP>
  User ubuntu
  IdentityFile ~/Downloads/k8s-key.pem
  StrictHostKeyChecking no

# Control Plane nodes — jump through bastion
Host control-plane-1
  HostName <CP1_PRIVATE_IP>
  User ubuntu
  IdentityFile ~/Downloads/k8s-key.pem
  ProxyJump bastion
  StrictHostKeyChecking no

Host control-plane-2
  HostName <CP2_PRIVATE_IP>
  User ubuntu
  IdentityFile ~/Downloads/k8s-key.pem
  ProxyJump bastion
  StrictHostKeyChecking no

Host control-plane-3
  HostName <CP3_PRIVATE_IP>
  User ubuntu
  IdentityFile ~/Downloads/k8s-key.pem
  ProxyJump bastion
  StrictHostKeyChecking no

# Worker nodes
Host worker-1
  HostName <W1_PRIVATE_IP>
  User ubuntu
  IdentityFile ~/Downloads/k8s-key.pem
  ProxyJump bastion
  StrictHostKeyChecking no

Host worker-2
  HostName <W2_PRIVATE_IP>
  User ubuntu
  IdentityFile ~/Downloads/k8s-key.pem
  ProxyJump bastion
  StrictHostKeyChecking no
```

**Save the file.**

**Step 2.2 — Test All Connections**

```
ssh bastion            # should give ubuntu@bastion prompt
ssh control-plane-1    # should jump through bastion to CP1
ssh control-plane-2
ssh control-plane-3
ssh worker-1
ssh worker-2
```

**Step 2.3 — Install kubectl on Bastion**

```
# SSH to bastion first
ssh bastion

# Install kubectl
curl -LO "https://dl.k8s.io/release/v1.33.1/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Create .kube directory
mkdir -p ~/.kube

# Exit bastion
exit
```

## PHASE 3 — Prepare All Kubernetes Nodes

**Run EVERY command in Phase 3 on ALL 5 K8s nodes** (3 control planes + 2 workers).

Open 5 terminal windows. SSH into each node in a separate window. Run commands in all 5 simultaneously.

**Step 3.1 — Set Hostnames**

Run the command **matching the node** you're on:

```
# On control-plane-1:
sudo hostnamectl set-hostname control-plane-1

```

**Step 3.2 — Update /etc/hosts on ALL 5 Nodes**
Replace the IPs below with your actual private IPs, then run on every node:

```
sudo tee -a /etc/hosts << 'EOF'
# Kubernetes cluster nodes
<CP1_PRIVATE_IP>   control-plane-1
<CP2_PRIVATE_IP>   control-plane-2
<CP3_PRIVATE_IP>   control-plane-3
<W1_PRIVATE_IP>    worker-1
<W2_PRIVATE_IP>    worker-2
EOF
```

Now, set variable called NLB_DNS with the NLB copied from AWS Console.

```
export NLB_DNS=<NLB DNS pasted from AWS console>

Copy Private IP of Control Plane 1 using command "hostname -I | awk '{print $1}'". Then run,
echo "<Private IP>  <NLB_DNS>" | sudo tee -a /etc/hosts

grep "k8s-api-nlb" /etc/hosts
```

**Step 3.3 — Disable Swap**

```
# Disable immediately
sudo swapoff -a

# Disable permanently (survives reboot)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify — Swap row must show 0
free -h
```

**Step 3.4 — Load Kernel Modules**

```
# Write module names to auto-load config
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load them right now (no reboot needed)
sudo modprobe overlay
sudo modprobe br_netfilter

# Verify — both must return output
lsmod | grep overlay
lsmod | grep br_netfilter
```

**Step 3.5 — Configure Kernel Networking Parameters**

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply immediately
sudo sysctl --system

# Verify — all 3 must show = 1
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

**Step 3.6 — Install containerd (Container Runtime)**

```
# Download
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.28/containerd-1.7.28-linux-amd64.tar.gz

# Extract to /usr/local
sudo tar Cxzvf /usr/local containerd-1.7.28-linux-amd64.tar.gz

# Get the systemd service file
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup (required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Set correct pause image
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.10"#g' /etc/containerd/config.toml

# Start and enable containerd
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Verify — must show Active: active (running)
systemctl status containerd
```

**Step 3.7 — Install runc**

```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.2.5/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# Verify
runc --version
```

**Step 3.8 — Install CNI Plugins**

```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.2.tgz
```

**Step 3.9 — Install kubeadm, kubelet, kubectl**

```
# Dependencies
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes GPG key
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repo
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install
sudo apt-get update
sudo apt-get install -y \
  kubelet=1.33.1-1.1 \
  kubeadm=1.33.1-1.1 \
  kubectl=1.33.1-1.1 \
  --allow-change-held-packages

# Lock versions (prevent accidental upgrade)
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet

# Verify
kubeadm version
kubelet --version
kubectl version --client
containerd --version
runc --version
```

**Step 3.10 — Configure crictl**

```
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock

# Verify — should return containerd info JSON
sudo crictl info
```

**Step 4.1 — Initialize First Control Plane**

Run kubeadm init:

```
sudo kubeadm init \
  --control-plane-endpoint "$NLB_DNS:6443" \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=$NODE_IP \
  --apiserver-cert-extra-sans="$NLB_DNS,$NODE_IP,127.0.0.1" \
  --upload-certs \
  --node-name control-plane-1
```

**Copy both join commands into a text file on your laptop right now. You need them for the next steps.**

**Step 4.2 — Set Up kubeconfig on control-plane-1**


```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Quick check — status will be NotReady (no CNI yet, that's fine)
kubectl get nodes
```

**Now, install Calico (v3.30.2):**

```
# Install Tigera operator:
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

# Download custom resources:
curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/custom-resources.yaml -O

# Apply:
kubectl apply -f custom-resources.yaml
```

**Step 4.3 — Copy kubeconfig to Bastion**

From your laptop (not from CP1):

```
# Copy kubeconfig from CP1 through to bastion
ssh control-plane-1 "cat ~/.kube/config" | \
  ssh bastion "mkdir -p ~/.kube && cat > ~/.kube/config"

# Verify kubectl works from bastion
ssh bastion "kubectl get nodes"
```

**Step 4.4 — Join Control Plane 2**

SSH to control-plane-2:

```
ssh control-plane-2
```

**Step 4.4.1 - Set Hostnames**

```
sudo hostnamectl set-hostname control-plane-2
```

**Step 4.4.2 - Update /etc/hosts**

```
sudo tee -a /etc/hosts << 'EOF'
# Kubernetes cluster nodes
<CP1_PRIVATE_IP>   control-plane-1
<CP2_PRIVATE_IP>   control-plane-2
<CP3_PRIVATE_IP>   control-plane-3
<W1_PRIVATE_IP>    worker-1
<W2_PRIVATE_IP>    worker-2
EOF
```

**Step 4.4.3 - Disable Swap**

```
# Disable immediately
sudo swapoff -a

# Disable permanently (survives reboot)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify — Swap row must show 0
free -h
```

**Step 4.4.4 - Load Kernel Modules**

```
# Write module names to auto-load config
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load them right now (no reboot needed)
sudo modprobe overlay
sudo modprobe br_netfilter

# Verify — both must return output
lsmod | grep overlay
lsmod | grep br_netfilter
```

**Step 4.4.5 - Configure Kernel Networking Parameters**

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply immediately
sudo sysctl --system

# Verify — all 3 must show = 1
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

**Step 4.4.6 - Install containerd (Container Runtime)**

```
# Download
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.28/containerd-1.7.28-linux-amd64.tar.gz

# Extract to /usr/local
sudo tar Cxzvf /usr/local containerd-1.7.28-linux-amd64.tar.gz

# Get the systemd service file
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup (required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Set correct pause image
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.10"#g' /etc/containerd/config.toml

# Start and enable containerd
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Verify — must show Active: active (running)
systemctl status containerd
```

**Step 4.4.7 - Install runc**

```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.2.5/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# Verify
runc --version
```

**Step 4.4.8 - Install kubeadm, kubelet, kubectl**

```
# Dependencies
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes GPG key
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repo
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install
sudo apt-get update
sudo apt-get install -y \
  kubelet=1.33.1-1.1 \
  kubeadm=1.33.1-1.1 \
  kubectl=1.33.1-1.1 \
  --allow-change-held-packages

# Lock versions (prevent accidental upgrade)
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet

# Verify
kubeadm version
kubelet --version
kubectl version --client
containerd --version
runc --version
```

**Step 4.4.9 - Configure crictl**

```
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock

# Verify — should return containerd info JSON
sudo crictl info
```

**Step 4.4.10 - Test Connectivity to CP1 BEFORE Re-joining**

```
# Can CP2 reach CP1's API server directly?
curl -k https://10.0.10.10:6443/livez
# Expected: "ok"

# Can CP2 reach CP1 via NLB?
curl -k https://k8s-api-nlb-201ba974294070a9.elb.us-east-1.amazonaws.com:6443/livez
# Expected: "ok"
# If this times out — Security Group issue (CP2 can't reach NLB on 6443)
```

If both return **ok**, proceed. If the NLB check times out, check that **CP2's security group (k8s-sg-control-plane)** has inbound **TCP 6443** from itself.

**Step 4.4.11 - Run Kubeadm join**

```
sudo kubeadm join <NLB_DNS>:6443 \
  --token <TOKEN_RECEIVED_FROM_KUBEADM_JOIN_COMMAND_RUN_ON_CP1> \
  --discovery-token-ca-cert-hash sha256:<HASH_RECEIVED_FROM_KUBEADM_JOIN_COMMAND_RUN_ON_CP1> \
  --control-plane \
  --certificate-key <CERT_KEY_RECEIVED_FROM_KUBEADM_JOIN_COMMAND_RUN_ON_CP1> \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}') \
  --node-name control-plane-2
```

**Step 4.4.12 - AFTER Successful Join: Then Add /etc/hosts on CP2**

```
# Only run this AFTER kubeadm join finishes successfully
CP2_IP=$(hostname -I | awk '{print $1}')
echo "$CP2_IP  <NLB_DNS>" | sudo tee -a /etc/hosts

# Set up kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## For 3rd Control Plane, follow the same steps followed for Control Plane 2

**Step 5 - Set up Worker Node 1**

```
ssh worker-1
```

**Step 5.1 — Set Hostnames**

```
sudo hostnamectl set-hostname worker-1
```

**Step 5.2 — Update /etc/hosts**

```
sudo tee -a /etc/hosts << 'EOF'
# Kubernetes cluster nodes
<CP1_PRIVATE_IP>   control-plane-1
<CP2_PRIVATE_IP>   control-plane-2
<CP3_PRIVATE_IP>   control-plane-3
<W1_PRIVATE_IP>    worker-1
<W2_PRIVATE_IP>    worker-2
EOF
```

**Step 5.3 — Disable Swap**

```
# Disable immediately
sudo swapoff -a

# Disable permanently (survives reboot)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify — Swap row must show 0
free -h
```

**Step 5.4 — Load Kernel Modules**

```
# Write module names to auto-load config
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load them right now (no reboot needed)
sudo modprobe overlay
sudo modprobe br_netfilter

# Verify — both must return output
lsmod | grep overlay
lsmod | grep br_netfilter
```

**Step 5.5 — Configure Kernel Networking Parameters**

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply immediately
sudo sysctl --system

# Verify — all 3 must show = 1
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

**Step 5.6 — Install containerd (Container Runtime)**

```
# Download
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.28/containerd-1.7.28-linux-amd64.tar.gz

# Extract to /usr/local
sudo tar Cxzvf /usr/local containerd-1.7.28-linux-amd64.tar.gz

# Get the systemd service file
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup (required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Set correct pause image
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.10"#g' /etc/containerd/config.toml

# Start and enable containerd
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Verify — must show Active: active (running)
systemctl status containerd
```

**Step 5.7 — Install runc**

```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.2.5/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# Verify
runc --version
```

**Step 5.8 — Install CNI Plugins**

```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.2.tgz
```

**Step 5.9 — Install kubeadm, kubelet, kubectl**

```
# Dependencies
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes GPG key
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repo
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install
sudo apt-get update
sudo apt-get install -y \
  kubelet=1.33.1-1.1 \
  kubeadm=1.33.1-1.1 \
  kubectl=1.33.1-1.1 \
  --allow-change-held-packages

# Lock versions (prevent accidental upgrade)
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet

# Verify
kubeadm version
kubelet --version
kubectl version --client
containerd --version
runc --version
```

**Step 5.10 — Configure crictl**

```
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock

# Verify — should return containerd info JSON
sudo crictl info
```

**Step 5.11 - Run Kubeadm join command received from CP1 when kubeadm init was run**

```
sudo kubeadm join <NLB_DNS>:6443 \
  --token <TOKEN_FROM_CP1_JOIN_COMMAND> \
  --discovery-token-ca-cert-hash sha256:<CA_CERT_FROM_CP1_JOIN_COMMAND> \
  --node-name worker-1
```

**Step 5.12 - Update CoreDNS Upstream DNS**

```
# Check what's on the node
cat /etc/resolv.conf
# If it shows 127.0.0.53 — that's the problem

# Fix: Point to the real VPC DNS on all 5 nodes
# Run this on each node (control-plane-1, 2, 3, worker-1, worker-2):
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
cat /etc/resolv.conf  # should now show 10.0.0.2 or 169.254.169.253
```

**Follow the same steps followed from Worker-1 on Worker-2 node.**

