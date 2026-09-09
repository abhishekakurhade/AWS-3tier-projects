# AWS 3-Tier Architecture Project 🏗️

A highly available, scalable **3-tier web application** deployed on AWS, built using core AWS networking, compute, and database services — no third-party infra tools, pure console/native AWS setup.

The project demonstrates a real-world production-style deployment pattern: a public-facing web tier, an internal application tier, and an isolated database tier, spread across multiple Availability Zones for fault tolerance.

---

## 📐 Architecture Overview


**Traffic flow:** Internet → Internet Gateway → Internet-facing ALB → Web Tier (public subnets) → Internal ALB → App Tier (private subnets) → RDS (private DB subnets). Private subnets reach the internet **outbound-only** (for patches/updates) via a NAT Gateway.

---

## ⚙️ AWS Services Used

| Service | Purpose |
|---|---|
| **VPC** | Custom isolated network (`192.168.0.0/16`) spanning 2 Availability Zones |
| **Subnets** | 6 subnets — public, private (app), and private (DB) tiers, 2 per AZ |
| **Internet Gateway** | Allows inbound/outbound internet access for public subnets |
| **NAT Gateway** | Gives private subnets outbound-only internet access |
| **Route Tables** | 5 route tables controlling traffic between public, private, and DB tiers |
| **Application Load Balancer (ALB)** | 2 ALBs — one internet-facing (web tier), one internal (app tier) |
| **EC2** | Hosts the web tier and app tier instances |
| **Auto Scaling Group** | Automatically scales EC2 instances based on demand/health |
| **Security Groups** | Enforces least-privilege access between tiers |
| **RDS** | Managed relational database in an isolated private DB subnet |

---

## 🌐 Network Design

**VPC:** `3iter-application` — `192.168.0.0/16`

| Availability Zone | Subnet | CIDR | Tier |
|---|---|---|---|
| ap-south-1a | public-subnet-az1 | 192.168.0.0/22 | Public (Web) |
| ap-south-1a | private-subnet-az1 | 192.168.8.0/22 | Private (App) |
| ap-south-1a | private-db-az1 | 192.168.16.0/22 | Private (DB) |
| ap-south-1b | public-subnet-az3 | 192.168.4.0/22 | Public (Web) |
| ap-south-1b | private-subnet-az3 | 192.168.12.0/22 | Private (App) |
| ap-south-1b | private-db-az3 | 192.168.20.0/22 | Private (DB) |

- **Internet Gateway** (`3tier-app-gateway`) — routes internet traffic to the 2 public subnets.
- **NAT Gateway** (`3tier-app-nat-gateway`) — single public NAT gateway (1 EIP) giving private subnets outbound internet access.
- **Route Tables** — `public-RT` (public subnets → IGW), `private-RT` (app subnets → NAT), `private-db-RT` (DB subnets, local-only, no internet route).

![VPC Network Architecture](screenshots/vpc-network-architecture.jpg)

---

## ⚖️ Load Balancing

Two Application Load Balancers were provisioned in the VPC, both spanning 2 Availability Zones for high availability:

| Load Balancer | Scheme | Purpose |
|---|---|---|
| `Loadbalcer-frontend` | Internet-facing | Routes public user traffic to the web tier EC2 instances |
| `app-internal-LB` | Internal | Routes internal traffic from the web tier to the app tier EC2 instances |

![Load Balancer Dashboard](screenshots/load-balancer-dashboard.jpg)

---

## 🖥️ Application UI

Sample deployed application — a registration form that writes to the RDS-backed data tier and lists registered records.

![Application UI](screenshots/app-registration-ui.jpg)

---

## ✨ Features

- 🔒 Multi-tier network isolation (public / private-app / private-db)
- 📈 Auto Scaling for both web and app tiers to handle variable load
- ⚖️ Load balancing at two layers (public + internal)
- 🌍 Multi-AZ deployment for high availability and fault tolerance
- 🛡️ Security Groups enforcing tier-to-tier least-privilege access
- 🗄️ Managed database layer (RDS) isolated from direct internet access
- 🚪 NAT Gateway for secure outbound-only access from private subnets

---

## 📁 Repository Structure

```
AWS-3tier-projects/
├── README.md
└── screenshots/
    ├── vpc-network-architecture.jpg
    ├── load-balancer-dashboard.jpg
    └── app-registration-ui.jpg
```

> Add your application source code (web tier / app tier) and any IaC scripts as additional folders, e.g. `web-tier/`, `app-tier/`, `scripts/`.

---

## 🚀 Deployment Steps (Summary)

1. **Create the VPC** — `192.168.0.0/16` with 6 subnets (2 public, 2 private-app, 2 private-db) across 2 AZs.
2. **Attach an Internet Gateway** and set up a **NAT Gateway** in a public subnet.
3. **Configure route tables** for public, private-app, and private-db subnets.
4. **Create Security Groups** for each tier (ALB → Web, Web → App, App → RDS).
5. **Launch EC2 instances** for the web and app tiers behind their respective **Auto Scaling Groups**.
6. **Create two ALBs** — internet-facing (web tier) and internal (app tier) — with health checks and target groups.
7. **Provision RDS** in the private DB subnets (multi-AZ recommended).
8. **Deploy the application code** to the web/app tier instances and connect the app tier to RDS.
9. **Test end-to-end** — access the app via the ALB DNS name.

---

## 🔮 Future Improvements

- Convert this manual setup into **Infrastructure as Code** (Terraform / CloudFormation)
- Add a **CI/CD pipeline** (CodePipeline / GitHub Actions) for automated deployments
- Enable **CloudWatch alarms** and dashboards for proactive monitoring
- Add **HTTPS** via ACM certificates on the public ALB
- Introduce a **CDN (CloudFront)** in front of the web tier

---

## 👤 Author

**Abhishek**
Project: AWS 3-Tier Architecture Deployment

---

*If you found this project helpful, consider giving it a ⭐!*
