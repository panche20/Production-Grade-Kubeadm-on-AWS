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
