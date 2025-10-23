# HashiCorp Vault — Production Architecture & Deployment Guide.

##  Architecture Overview
![Uploading image.png…]()


##Overview
This production-grade HashiCorp Vault deployment is designed for high availability, fault tolerance, and end-to-end security in AWS. It leverages a 5-node Vault cluster using Raft Integrated Storage, distributed across three Availability Zones (2-2-1 node setup) to tolerate up to two node failures and survive an entire AZ outage.

Vault nodes reside in private subnets with no direct internet access, while an Application Load Balancer (ALB) handles external HTTPS traffic. The architecture includes:
Redundant NAT Gateways for high availability
Automatic unsealing via AWS KMS
Synchronous Raft replication with automatic leader election
Secure inter-node communication via mTLS
Storage & Backup:
EBS volumes provide encrypted storage for data and audit logs
Daily snapshots can be pushed to S3 for disaster recovery.

###  Design Goals
1. **N+2 Redundancy:** 5 Vault nodes tolerate 2 node failures  
2. **Multi-AZ Distribution:** Survive entire Availability Zone (AZ) loss  
3. **Zero Single Points of Failure:** Each layer is redundant  
4. **Auto-Recovery:** Self-healing via Auto Scaling and Raft Autopilot  
5. **End-to-End Security:** TLS + AWS KMS for unseal  


---

## ☁️ AWS Network Design

### 🏗️ VPC Overview
**VPC CIDR:** `10.0.0.0/16`  
Region: **us-west-2 (Oregon)**  
The VPC is divided across **three Availability Zones (AZs)** for high availability and fault tolerance.

---

### 🌐 Subnet Layout

| Tier | Subnet CIDR | Availability Zone | Purpose | Components |
|------|---------------|-------------------|----------|-------------|
| **Public Subnet 1** | 10.0.101.0/24 | us-west-2a | Ingress / NAT | ALB, NAT Gateway |
| **Public Subnet 2** | 10.0.102.0/24 | us-west-2b | Ingress / NAT | ALB, NAT Gateway |
| **Public Subnet 3** | 10.0.103.0/24 | us-west-2c | Ingress / NAT | ALB, NAT Gateway |
| **Private Subnet 1** | 10.0.1.0/24 | us-west-2a | Application Layer | Vault Node 1, Node 2 |
| **Private Subnet 2** | 10.0.2.0/24 | us-west-2b | Application Layer | Vault Node 3, Node 4 |
| **Private Subnet 3** | 10.0.3.0/24 | us-west-2c | Application Layer | Vault Node 5 |

---

### 🧭 Routing Configuration

| Route Table | Destination | Target | Associated Subnets |
|--------------|-------------|---------|---------------------|
| Public Route Table | `0.0.0.0/0` | Internet Gateway (IGW) | All Public Subnets |
| Private Route Table (AZ A) | `0.0.0.0/0` | NAT Gateway (10.0.101.x) | 10.0.1.0/24 |
| Private Route Table (AZ B) | `0.0.0.0/0` | NAT Gateway (10.0.102.x) | 10.0.2.0/24 |
| Private Route Table (AZ C) | `0.0.0.0/0` | NAT Gateway (10.0.103.x) | 10.0.3.0/24 |

---

### 🔐 Network Design Highlights

1. **Multi-AZ Deployment** — Vault cluster spread across three AZs (2-2-1 setup).  
2. **Public Layer** — Only ALB and NAT Gateways have internet access.  
3. **Private Layer** — Vault nodes live in private subnets with no direct internet access.  
4. **Redundant NATs** — Each AZ has its own NAT Gateway for failover.  
5. **Encrypted Traffic** — All communication (ALB ↔ Vault, Vault ↔ Vault) is TLS secured.  
6. **Scalability** — Each tier can independently scale within its subnet CIDR.  

---

### 🧱 Security Summary

| Component | Access Type | Description |
|------------|--------------|-------------|
| **ALB** | Public | Handles HTTPS (443) traffic from clients |
| **Vault Nodes** | Private | Accessible via ALB or bastion host only |
| **NAT Gateway** | Public | Enables outbound internet access for private subnets |
| **Security Groups** | Restricted | Only necessary inbound/outbound ports opened (443, 8200) |

---

### 🗺️ Example Deployment

- **5 Vault Nodes (Raft storage backend)** deployed in private subnets  
- **1 Application Load Balancer (ALB)** for external client access  
- **3 NAT Gateways** for high availability  
- **1 Internet Gateway (IGW)** for public routing  
- **3 Route Tables** (per AZ for private subnets)

---

## ⚖️ Load Balancer (ALB)

- **Type:** Application Load Balancer (HTTPS 8200)  
- **Health Check:** `/v1/sys/health?standbyok=true&perfstandbyok=true`  
- **Codes:** `200=Active`, `429=Standby`, `472=DR`, `473=Perf Standby`  
- **Stickiness:** Enabled (1 hour cookie)  
- **TLS:** End-to-end encryption  

---

## 🔐 Vault Cluster (5-Node Raft)

| Node | AZ | Role | Subnet |
|------|----|------|--------|
| vault-1 | us-west-2a | Leader/Follower | 10.0.1.0/24 |
| vault-2 | us-west-2a | Follower | 10.0.1.0/24 |
| vault-3 | us-west-2b | Follower | 10.0.2.0/24 |
| vault-4 | us-west-2b | Follower | 10.0.2.0/24 |
| vault-5 | us-west-2c | Follower | 10.0.3.0/24 |

**Raft Consensus:**
- Quorum = 3 of 5 nodes  
- Leader election = automatic (~10s)  
- Log replication = synchronous  

---

## 💾 Storage and Audit Setup

- **Storage Type:** Integrated Storage (Raft)  
- **Data Path:** `/opt/vault/data` (100 GB gp3 EBS)  
- **IOPS:** 3000 | **Throughput:** 125 MB/s  
- **Encryption:** EBS at rest  

**Audit Logs:**
- Path: `/var/log/vault-audit` (50 GB volume)  
- Format: JSON (one entry per line)  

---

## 🔑 Auto-Unseal (AWS KMS)

### First Boot
1. Vault starts sealed  
2. Run `vault operator init`  
3. Master key encrypted via AWS KMS  
4. Recovery keys generated  

### Restart (Auto Unseal)
1. Vault authenticates with IAM role  
2. Uses AWS KMS to decrypt key  
3. Vault auto-unseals and joins cluster  

✅ No manual unseal  
✅ Safe for Auto Scaling and restarts  

---

## 🧱 Security Controls

### TLS
- Client → ALB → Vault: TLS 1.2+  
- Vault → Vault (Raft): mTLS (8201)  

### IAM Role Permissions
- `kms:Decrypt`, `kms:DescribeKey`  
- `ec2:DescribeInstances`, `ec2:DescribeTags`  
- `autoscaling:DescribeAutoScalingGroups`  
- `logs:CreateLogStream`, `logs:PutLogEvents`  

### Security Groups
| Port | Source | Purpose |
|------|---------|----------|
| 8200 | ALB SG | Vault API |
| 8201 | Self | Raft Traffic |
| 22 | Bastion SG | Admin SSH |

---

## ☁️ Backup & Recovery

### Backup (Manual)
```bash
vault operator raft snapshot save backup-$(date +%Y%m%d).snap
```

### Restore
```bash
vault operator raft snapshot restore backup.snap
```

Stored snapshots can be pushed to **AWS S3** for Disaster recovery.

---

## ✅ Summary

This setup delivers:
- Production-grade **Vault HA cluster**
- **5 nodes** using **Raft Integrated Storage**
- **AWS KMS Auto-Unseal**
- **Multi-AZ fault tolerance**
- **End-to-end TLS**
- **Daily snapshot → S3 backup**
- **Self-healing + quorum-based consistency**
