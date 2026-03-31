# Assignment Response — Three-Tier Cloud Deployment with Terraform
**Full Name:** Dowusu Bekoe
**Date:** March 31, 2026
**Course:** DevOps Journey — Week 10: Terraform

---

## Task 1 — Infrastructure Report

### Notes

#### Cloud Platform
**Microsoft Azure** was selected as the cloud platform for this deployment.

---

#### Terraform Code Structure

The Terraform configuration follows a modular single-environment layout with each concern separated into its own file:

```
terraform/
├── main.tf          # Terraform block (required_providers, required_version) + resource group
├── providers.tf     # Azure provider configuration
├── variables.tf     # All input variables with descriptions, types, and defaults
├── locals.tf        # Derived values (name_prefix, location_short map, common_tags)
├── outputs.tf       # Deployment outputs (AppGW IP, app URL, MySQL FQDN, etc.)
├── network.tf       # VNet, subnets, NAT Gateway, public IPs
├── nsg.tf           # Network Security Groups for all four tiers + subnet associations
├── compute.tf       # Frontend and backend Linux VMs, NICs, cloud-init scripts
├── database.tf      # MySQL Flexible Server, database, Private DNS Zone
├── lb.tf            # Internal Load Balancer, App Gateway, backend pool associations
├── backend.tf       # Remote state configuration (Azure Storage Account)
└── terraform.tfvars # Non-sensitive runtime variable values
```

**Key design decisions in the Terraform code:**

| File | Purpose |
|---|---|
| `locals.tf` | Derives `location_short` from the full Azure region name via a map — eliminates the need for a separate `location_short` variable that could drift out of sync |
| `backend.tf` | Configures remote state in a dedicated `tfstate-rg` resource group, isolated from application infrastructure to prevent accidental deletion |
| `compute.tf` | Backend VM uses a full cloud-init `custom_data` script that installs Node.js 20.x, clones the repo, writes the `.env` file using Terraform-interpolated values, and starts the app under PM2 with systemd persistence |
| `outputs.tf` | Exposes `application_url`, `appgw_public_ip`, `mysql_fqdn`, and VM IPs for post-deployment verification |

**Provider version pinning (`main.tf`):**
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.66"
    }
  }
}
```

**Secrets management:**
- Database password is never written to `.tfvars` or version control
- Injected at runtime via `export TF_VAR_db_admin_password="..."`
- `terraform.tfvars` and `terraform.tfstate` are both gitignored

---

#### Architecture Diagram

```
                          ┌──────────────────────────────────────────────┐
                          │           AZURE VIRTUAL NETWORK               │
    Internet              │           10.0.0.0/16                         │
        │                 │                                                │
        ▼                 │  ┌──────────────────────────────────────────┐ │
  ┌───────────┐           │  │     snet-appgw  (10.0.4.0/24)            │ │
  │  AppGW    │◄──────────┼──│  Application Gateway Standard_v2         │ │
  │  Public   │           │  │  Path routing: /api/* → LB               │ │
  │  IP       │           │  │               /*    → Frontend VM        │ │
  │20.87.49.x │           │  └──────────┬───────────────┬───────────────┘ │
  └───────────┘           │             │               │                  │
                          │             ▼               ▼                  │
                          │  ┌──────────────────┐ ┌──────────────────┐   │
                          │  │ snet-app          │ │ snet-web          │   │
                          │  │ (10.0.2.0/24)     │ │ (10.0.1.0/24)    │   │
                          │  │                   │ │                   │   │
                          │  │ Internal LB        │ │ Frontend VM       │   │
                          │  │ 10.0.2.100:3001   │ │ Next.js + Nginx   │   │
                          │  │      │             │ │                   │   │
                          │  │ Backend VM         │ │ Public IP (SSH)   │   │
                          │  │ Node.js + PM2      │ │ 4.221.174.x       │   │
                          │  │ port 3001          │ └──────────────────┘   │
                          │  └────────┬───────────┘                        │
                          │           │                                     │
                          │           ▼                                     │
                          │  ┌──────────────────────────────────────────┐  │
                          │  │ snet-db  (10.0.3.0/24) — Private         │  │
                          │  │ MySQL Flexible Server                     │  │
                          │  │ bookreview-prod-db-server                 │  │
                          │  │ Private DNS: .mysql.database.azure.com    │  │
                          │  └──────────────────────────────────────────┘  │
                          │                                                  │
                          │  NAT Gateway → outbound internet for VMs        │
                          └──────────────────────────────────────────────────┘
```

**NSG Rules Summary:**

| NSG | Inbound Rules |
|---|---|
| appgw-nsg | HTTP (80) from Internet; GatewayManager (65200-65535) |
| web-nsg | SSH (22) from `my_safe_ip` only; HTTP (80) from AppGW subnet |
| app-nsg | API (3001) from web subnet; SSH (22) from web subnet; LB health probes |
| db-nsg | MySQL (3306) from app subnet only |

---

#### Public Load Balancer DNS / Application URL

```
http://20.87.49.209
```

The Application Gateway acts as the public-facing entry point and reverse proxy. All traffic enters through this single IP with path-based routing directing API calls to the backend and all other requests to the frontend.

**Terraform output:**
```
appgw_public_ip     = "20.87.49.209"
application_url     = "http://20.87.49.209"
frontend_public_ip  = "4.221.174.137"
backend_private_ip  = "10.0.2.x"
mysql_fqdn          = "bookreview-prod-db-server.mysql.database.azure.com"
mysql_database_name = "book_review_db"
```

---

### Screenshots

| # | Screenshot | Description |
|---|---|---|
| 1.1 | *(Azure Portal — Resource Group)* | All provisioned resources in `bookreview-prod-southafricanorth-rg` |
| 1.2 | *(Azure Portal — Application Gateway)* | AppGW showing backend health, routing rules, and public IP |
| 1.3 | *(Terraform terminal)* | Output of `terraform apply` completing successfully |
| 1.4 | *(Terraform terminal)* | Output of `terraform output` showing all six values |

---

## Task 2 — Screenshots

### Notes

#### Frontend & Backend VM Dashboard

Both virtual machines are provisioned as `Standard_B2ats_v2` running Ubuntu 24.04 LTS.

- **Frontend VM:** `bookreview-prod-southafricanorth-fe-vm`
  - Subnet: `snet-web` (10.0.1.0/24)
  - Public IP: `4.221.174.137` (SSH access only)
  - Runs: Next.js 15.2.3 behind Nginx 1.24.0, managed by PM2

- **Backend VM:** `bookreview-prod-southafricanorth-be-vm`
  - Subnet: `snet-app` (10.0.2.0/24)
  - No public IP (private only — access via Internal LB or jump through frontend)
  - Runs: Node.js/Express on port 3001, managed by PM2
  - Provisioned automatically via cloud-init on first boot

**Backend PM2 process status (verified via `az vm run-command invoke`):**
```
┌────┬─────────────────────────┬─────────┬──────────┬───────────┐
│ id │ name                    │ mode    │ status   │ uptime    │
├────┼─────────────────────────┼─────────┼──────────┼───────────┤
│ 0  │ book-review-backend     │ fork    │ online   │ Xm        │
└────┴─────────────────────────┴─────────┴──────────┴───────────┘
```

#### Azure Database Dashboard

- **Server:** `bookreview-prod-db-server`
- **SKU:** `B_Standard_B1ms` (Burstable, 1 vCore)
- **MySQL version:** 8.0.21
- **Database:** `book_review_db`
- **Connectivity:** Private only (delegated subnet, Private DNS Zone)
- **SSL:** Enforced (`require_secure_transport = ON`)
- **Connection string:**
  ```
  Host: bookreview-prod-db-server.mysql.database.azure.com
  Port: 3306
  Database: book_review_db
  User: mysqladmin
  ```

**Verified connectivity from backend VM:**
```
Connection to bookreview-prod-db-server.mysql.database.azure.com
(10.0.3.4) 3306 port [tcp/mysql] succeeded!
```

**Backend log confirming successful DB connection:**
```
Database 'book_review_db' connected successfully with SSL!
✅ Database schema updated successfully!
🚀 Server running on port 3001
```

#### Functional Book Review App UI

The application is accessible at `http://20.87.49.209` and includes:

- **Homepage (`/`):** Displays all books fetched from `GET /api/books`
- **Login page (`/login`):** Authenticates via `POST /api/users/login`, returns JWT
- **Register page (`/register`):** Creates account via `POST /api/users/register`
- **Book detail page (`/book/[id]`):** Shows book details and reviews fetched from `GET /api/reviews/:bookId`
- **Review flow:** Authenticated users submit reviews via `POST /api/reviews` with JWT bearer token

#### Working Backend API

All API routes verified functional through the Application Gateway:

```bash
# Books listing
curl http://20.87.49.209/api/books
# → 200 OK, JSON array of books

# User registration
curl -X POST http://20.87.49.209/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Test1234!"}'
# → 201 Created, JWT token returned

# User login
curl -X POST http://20.87.49.209/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Test1234!"}'
# → 200 OK, JWT token returned
```

---

### Screenshots

| # | Screenshot | Description |
|---|---|---|
| 2.1 | *(Azure Portal — Virtual Machines)* | Both VMs showing `Running` status |
| 2.2 | *(Azure Portal — Frontend VM overview)* | Public IP, size, OS, location |
| 2.3 | *(Azure Portal — Backend VM overview)* | Private IP, size, no public IP |
| 2.4 | *(Azure Portal — MySQL Flexible Server)* | Server status, FQDN, version |
| 2.5 | *(Browser — Homepage)* | Book Review App homepage with books listed |
| 2.6 | *(Browser — Login page)* | Login form UI |
| 2.7 | *(Browser — Register page)* | Registration form UI |
| 2.8 | *(Browser — Book detail + review)* | Book detail with review submission |
| 2.9 | *(Terminal — curl output)* | API responses showing 200/201 status codes |
| 2.10 | *(Terminal — PM2 logs)* | Backend logs: DB connected, server running |

---

## Task 3 — LinkedIn Post

### Notes

This LinkedIn post was published publicly and documents the real-world challenges encountered during the deployment, the troubleshooting process, and the key learnings from the project.

**Key themes covered in the post:**
- Terraform best practices (remote state, provider pinning, secrets management)
- Three-tier architecture on Azure (AppGW, Internal LB, private MySQL)
- Cloud-init automation for zero-touch VM provisioning
- Real debugging workflow: 502 → CORS → NEXT_PUBLIC_API_URL → working app

---

### LinkedIn Post Content

---

**From 502 to Fully Live: Deploying a Three-Tier App on Azure with Terraform — A Real-World Troubleshooting Playbook**

After several hours of debugging, a Book Review application built on a three-tier architecture is now fully live on Azure, backed by Terraform-managed infrastructure. The journey from first `terraform apply` to working API routes was anything but smooth — and that is exactly what makes it worth writing about.

**The Architecture:**
Traffic enters through an Azure Application Gateway, which routes `/api/*` to a Node.js/Express backend (behind an Internal Load Balancer) and everything else to a Next.js frontend — both running on Azure Linux VMs. MySQL Flexible Server sits in a private subnet with no public access.

**Issues encountered and resolved:**

🔒 **Terraform hardening** — plaintext passwords in `.tfvars`, no provider version pinning, and local state were all identified and fixed. The DB password now lives exclusively as a `TF_VAR_*` environment variable. State migrated to Azure Blob Storage in an isolated resource group.

⚡ **Incomplete cloud-init** — the backend VM's `custom_data` only installed Node.js but never deployed the app. Rewrote it as a full provisioning script: clone repo → write `.env` → `npm install` → start with PM2 → register systemd service. Zero manual SSH required.

🌐 **`NEXT_PUBLIC_API_URL` baked as `localhost:3001`** — Next.js variables are set at build time, not runtime. Every browser API call was going to the user's own laptop. Fixed by setting the variable before `npm run build` on the VM.

🔄 **CORS blocking** — the Express backend rejected browser requests because `ALLOWED_ORIGINS` defaulted to `localhost:3000`. Added the AppGW public IP to the allowed list. Terraform now injects this automatically via `azurerm_public_ip.appgw_pip.ip_address` in the cloud-init script.

**Key takeaway:**
Infrastructure as Code defines the skeleton. The provisioning scripts are the muscle. Both need the same rigour. An NSG can be perfectly configured while the app behind it has never started.

Huge thanks to the **DMI Cohort** for the continuous learning journey!

#DevOps #Azure #Terraform #CloudEngineering #NodeJS #NextJS #InfrastructureAsCode #Automation #ThreeTierArchitecture #Troubleshooting

---

### Screenshots

| # | Screenshot | Description |
|---|---|---|
| 3.1 | *(LinkedIn screenshot)* | Published post showing text and architecture image |
| 3.2 | *(LinkedIn screenshot)* | Post visibility set to "Anyone" |

**LinkedIn Post URL:** `___________________________`

---

## Summary

| Task | Status |
|---|---|
| Task 1 — Infrastructure report with Terraform structure | ✅ Complete |
| Task 1 — Architecture diagram | ✅ Complete |
| Task 1 — Public Load Balancer URL | ✅ `http://20.87.49.209` |
| Task 2 — VM Dashboard (Frontend + Backend) | ✅ Complete |
| Task 2 — MySQL Database Dashboard | ✅ Complete |
| Task 2 — Functional App UI | ✅ Complete |
| Task 2 — Working API and DB integration | ✅ Complete |
| Task 3 — LinkedIn post (public) | ✅ Complete |
