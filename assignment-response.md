# Assignment Response — Three-Tier Cloud Deployment with Terraform
**Full Name:** Dowusu Bekoe
**Date:** March 31, 2026
**Course:** DevOps Journey — Week 10: Terraform

---

## Task 1 — Infrastructure Report

### What I Did

I deployed a three-tier Book Review application on **Microsoft Azure** using Terraform to provision and manage all infrastructure from scratch. The architecture consists of a Next.js frontend VM (Nginx), a Node.js/Express backend VM (PM2), and an Azure MySQL Flexible Server — connected through an Application Gateway that performs path-based routing, directing `/api/*` traffic to the backend and all other requests to the frontend. I organised the Terraform code into separate files by concern (`main.tf`, `providers.tf`, `variables.tf`, `locals.tf`, `outputs.tf`, `network.tf`, `nsg.tf`, `compute.tf`, `database.tf`, `lb.tf`, `backend.tf`) following Terraform conventions. Remote state was configured in a dedicated Azure Storage Account in an isolated `tfstate-rg` resource group, and secrets were managed exclusively through environment variables (`TF_VAR_db_admin_password`) — never stored in version control.

---

### Notes

#### Cloud Platform
**Microsoft Azure**

---

#### Terraform Code Structure

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

**Key design decisions:**

| File | Purpose |
|---|---|
| `locals.tf` | Derives `location_short` from the full Azure region name via a map — eliminates a redundant variable that could drift out of sync |
| `backend.tf` | Remote state in a dedicated `tfstate-rg` resource group, isolated so `terraform destroy` cannot delete the state file |
| `compute.tf` | Backend VM uses a full cloud-init `custom_data` script — installs Node.js 20.x, clones the repo, writes `.env`, starts app via PM2 with systemd persistence |
| `outputs.tf` | Exposes `application_url`, `appgw_public_ip`, `mysql_fqdn`, and VM IPs for post-deployment verification |

**Provider version pinning:**
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
- Database password never written to `.tfvars` or version control
- Injected at runtime via `export TF_VAR_db_admin_password="..."`
- `terraform.tfvars`, `terraform.tfstate`, and `.mcp.json` are all gitignored

---

#### Architecture Diagram

```
                    ┌──────────────────────────────────────────────────┐
                    │            AZURE VIRTUAL NETWORK                  │
   Internet         │            10.0.0.0/16                            │
       │            │                                                    │
       ▼            │  ┌────────────────────────────────────────────┐   │
 ┌──────────┐       │  │  snet-appgw (10.0.4.0/24)                  │   │
 │  AppGW   │◄──────┼──│  Application Gateway Standard_v2           │   │
 │  Public  │       │  │  /api/* → Internal LB → Backend VM         │   │
 │  IP      │       │  │  /*    → Frontend VM                       │   │
 │20.87.49.x│       │  └──────────┬───────────────┬─────────────────┘   │
 └──────────┘       │             │               │                      │
                    │             ▼               ▼                      │
                    │  ┌──────────────────┐ ┌────────────────────┐      │
                    │  │ snet-app          │ │ snet-web            │      │
                    │  │ (10.0.2.0/24)     │ │ (10.0.1.0/24)      │      │
                    │  │ Internal LB       │ │ Frontend VM         │      │
                    │  │ 10.0.2.100:3001   │ │ Next.js + Nginx     │      │
                    │  │ Backend VM        │ │ Public IP (SSH only)│      │
                    │  │ Node.js + PM2     │ │ 4.221.174.x         │      │
                    │  │ port 3001 (private│ └────────────────────┘      │
                    │  └────────┬──────────┘                             │
                    │           ▼                                         │
                    │  ┌──────────────────────────────────────────────┐  │
                    │  │ snet-db (10.0.3.0/24) — Private              │  │
                    │  │ MySQL Flexible Server (SSL enforced)          │  │
                    │  │ Private DNS Zone: .mysql.database.azure.com   │  │
                    │  └──────────────────────────────────────────────┘  │
                    │  NAT Gateway — outbound internet for web + app      │
                    └──────────────────────────────────────────────────────┘
```

**NSG Rules Summary:**

| NSG | Inbound Rules |
|---|---|
| appgw-nsg | HTTP (80) from Internet; GatewayManager (65200–65535) |
| web-nsg | SSH (22) from `my_safe_ip` only; HTTP (80) from AppGW subnet |
| app-nsg | API (3001) from web subnet; SSH (22) from web subnet; LB health probes |
| db-nsg | MySQL (3306) from app subnet only |

---

#### Public Load Balancer DNS / Application URL

```
http://20.87.49.209
```

**Terraform outputs:**
```
appgw_public_ip     = "20.87.49.209"
application_url     = "http://20.87.49.209"
frontend_public_ip  = "4.221.174.137"
backend_private_ip  = "10.0.2.x"
mysql_fqdn          = "bookreview-prod-db-server.mysql.database.azure.com"
mysql_database_name = "book_review_db"
```

---

### What I Learned

Separating Terraform code by concern (one file per resource group) makes the configuration far easier to read, debug, and review in pull requests than placing everything in a single `main.tf`. I also learned that the Terraform state backend resource group must be created manually and kept outside Terraform's own management — if Terraform managed its own state storage, `terraform destroy` would delete the state file along with everything else, leaving orphaned resources with no way to manage them.

---

### Issues Faced and How I Solved Them

**Issue 1 — Terraform state lock after interrupted migration**
When migrating local state to the Azure Storage backend, the process was interrupted and left the state blob locked. Any subsequent `terraform plan` failed with `state blob is already locked`. Resolved by running `terraform force-unlock <lock-id>` using the ID shown in the error message — safe to do because the lock info confirmed no other operation was actively running on the same machine.

**Issue 2 — Provider version unpinned and secrets in `.tfvars`**
The initial code had no `required_providers` block, meaning any `terraform init` could pull a breaking provider version. The database password was also stored in plain text in `terraform.tfvars`. Fixed by adding a `terraform {}` block pinning `azurerm ~> 4.66`, removing the password from the file entirely, and documenting the `TF_VAR_db_admin_password` environment variable pattern for all future runs.

---

### Screenshots

| # | Screenshot |
|---|---|
| 1.1 | Azure Portal — Resource Group showing all provisioned resources |
| 1.2 | Azure Portal — Application Gateway with routing rules and public IP |
| 1.3 | Terminal — `terraform apply` completing successfully |
| 1.4 | Terminal — `terraform output` showing all six values |

---

## Task 2 — Application Deployment & Verification

### What I Did

I verified the full three-tier deployment end-to-end after `terraform apply` completed. The frontend VM (`bookreview-prod-southafricanorth-fe-vm`) runs Next.js 15.2.3 behind Nginx 1.24.0 and is managed by PM2 with a systemd service for reboot persistence. The backend VM (`bookreview-prod-southafricanorth-be-vm`) runs the Node.js/Express API on port 3001, also managed by PM2, provisioned automatically via cloud-init on first boot with no manual SSH required. The MySQL Flexible Server is accessible only from within the private `snet-db` subnet with SSL enforced and a Private DNS Zone resolving its FQDN internally. I confirmed the full registration, login, and book review flow works through the Application Gateway — with API calls correctly routing to `/api/users/register`, `/api/users/login`, `/api/books`, and `/api/reviews/:bookId`.

---

### Notes

#### Frontend & Backend VM Details

- **Frontend VM:** `bookreview-prod-southafricanorth-fe-vm`
  - Size: `Standard_B2ats_v2` | OS: Ubuntu 24.04 LTS
  - Subnet: `snet-web` (10.0.1.0/24)
  - Public IP: `4.221.174.137` (SSH access only)
  - Runs: Next.js 15.2.3 behind Nginx 1.24.0, managed by PM2

- **Backend VM:** `bookreview-prod-southafricanorth-be-vm`
  - Size: `Standard_B2ats_v2` | OS: Ubuntu 24.04 LTS
  - Subnet: `snet-app` (10.0.2.0/24)
  - No public IP — private only, accessible via Internal LB
  - Runs: Node.js/Express on port 3001, managed by PM2

**Backend PM2 process (confirmed via `az vm run-command invoke`):**
```
┌────┬──────────────────────┬─────────┬──────────┐
│ id │ name                 │ mode    │ status   │
├────┼──────────────────────┼─────────┼──────────┤
│ 0  │ book-review-backend  │ fork    │ online   │
└────┴──────────────────────┴─────────┴──────────┘
```

**Backend log confirming DB connectivity:**
```
Database 'book_review_db' connected successfully with SSL!
✅ Database schema updated successfully!
🚀 Server running on port 3001
```

#### Azure Database Dashboard

- **Server:** `bookreview-prod-db-server`
- **SKU:** `B_Standard_B1ms` | **MySQL version:** 8.0.21
- **Database:** `book_review_db`
- **Connectivity:** Private only (delegated subnet + Private DNS Zone)
- **SSL:** Enforced
- **FQDN:** `bookreview-prod-db-server.mysql.database.azure.com`

**Verified reachability from backend VM:**
```
Connection to bookreview-prod-db-server.mysql.database.azure.com
(10.0.3.4) 3306 port [tcp/mysql] succeeded!
```

#### API Routes Verified

```bash
curl http://20.87.49.209/api/books
# → 200 OK, JSON array of books

curl -X POST http://20.87.49.209/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Test1234!"}'
# → 201 Created, JWT token returned
```

---

### What I Learned

Running `az vm run-command invoke` is a powerful way to diagnose and interact with a VM without needing SSH — it was invaluable for checking PM2 process status, reading init logs, and patching environment files when SSH access was unstable. I also learned that Azure Application Gateway's 502 response specifically means the gateway itself is healthy but the backend pool is not responding, which immediately narrows the problem to the application layer rather than the network.

---

### Issues Faced and How I Solved Them

**Issue 1 — 502 Bad Gateway: backend app not running**
After `terraform apply`, the homepage loaded but all API calls returned 502. The root cause was an incomplete `custom_data` script — it installed Node.js but never cloned the repo, wrote the `.env`, or started the app. Rewrote the script to fully automate provisioning: install Node.js 20.x via NodeSource → clone repo → write `.env` with Terraform-interpolated values → `npm install --omit=dev` → start under PM2 → register systemd service for reboot persistence. Zero manual SSH required.

**Issue 2 — CORS rejecting all browser requests (500 error)**
After fixing the 502, API calls from the browser returned 500. PM2 logs revealed `Error: CORS policy: Not allowed by server`. The backend reads allowed origins from `ALLOWED_ORIGINS` in `.env`, which defaulted to `localhost:3000`. The AppGW IP (`http://20.87.49.209`) was not in the list. Appended `ALLOWED_ORIGINS=http://20.87.49.209` to the backend `.env` via `az vm run-command invoke` and restarted PM2. Updated `compute.tf` to inject this value automatically using `${azurerm_public_ip.appgw_pip.ip_address}` so it is set correctly on all future VM provisioning.

**Issue 3 — Frontend API calls targeting `localhost:3001`**
The homepage loaded but showed no books. The frontend `api.js` used `process.env.NEXT_PUBLIC_API_URL || "http://localhost:3001"` as the base URL. Since `NEXT_PUBLIC_API_URL` was not set at build time, every browser request went to the user's own machine. Fixed by writing `.env.local` with the correct AppGW IP on the frontend VM and triggering a fresh `npm run build`. The fallback in `api.js` was also updated from `"http://localhost:3001"` to `""` (empty string) so calls use relative paths behind any reverse proxy.

**Issue 4 — SSH host key mismatch after VM recreation**
When Terraform recreated the backend VM due to a `custom_data` change, SSH threw `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`. Expected behaviour — a new VM generates a new host key. Resolved with:
```bash
ssh-keygen -f '~/.ssh/known_hosts' -R '10.0.2.5'
```

---

### Screenshots

| # | Screenshot |
|---|---|
| 2.1 | Azure Portal — Virtual Machines dashboard showing both VMs as Running |
| 2.2 | Azure Portal — Frontend VM overview (public IP, OS, size) |
| 2.3 | Azure Portal — Backend VM overview (private IP only, no public IP) |
| 2.4 | Azure Portal — MySQL Flexible Server (FQDN, version, private access) |
| 2.5 | Browser — App homepage with book list displayed |
| 2.6 | Browser — Login page |
| 2.7 | Browser — Register page |
| 2.8 | Browser — Book detail page with review submission |
| 2.9 | Terminal — curl output showing 200/201 API responses |
| 2.10 | Terminal — PM2 logs showing DB connected and server running |

---

## Task 3 — LinkedIn Post

### What I Did

I wrote and published a LinkedIn post documenting the real-world deployment journey — covering the architecture decisions, the specific issues encountered during debugging, and the key engineering lessons learned. The post was structured as a practical troubleshooting playbook rather than a project announcement, walking through each failure (502 → CORS → localhost URL → working app) with the exact diagnosis and resolution for each. The post was published publicly with "Anyone" visibility.

---

### Notes

**LinkedIn Post:**

---

**From 502 to Fully Live: Deploying a Three-Tier App on Azure with Terraform — A Real-World Troubleshooting Playbook**

After several hours of debugging, a Book Review application built on a three-tier architecture is now fully live on Azure, backed by Terraform-managed infrastructure. The journey from first `terraform apply` to working API routes was anything but smooth — and that is exactly what makes it worth writing about.

**The Architecture:**
Traffic enters through an Azure Application Gateway, which routes `/api/*` to a Node.js/Express backend (behind an Internal Load Balancer) and everything else to a Next.js frontend — both running on Azure Linux VMs. MySQL Flexible Server sits in a private subnet with no public access.

**Issues encountered and resolved:**

🔒 **Terraform hardening** — plaintext passwords in `.tfvars`, no provider version pinning, and local state were all fixed. The DB password now lives exclusively as a `TF_VAR_*` environment variable. State migrated to Azure Blob Storage in an isolated resource group.

⚡ **Incomplete cloud-init** — the backend VM's `custom_data` only installed Node.js but never deployed the app. Rewrote it as a full provisioning script: clone repo → write `.env` → `npm install` → start with PM2 → register systemd service. Zero manual SSH required.

🌐 **`NEXT_PUBLIC_API_URL` baked as `localhost:3001`** — Next.js variables are set at build time, not runtime. Every browser API call was going to the user's own laptop. Fixed by setting the variable before `npm run build` on the VM.

🔄 **CORS blocking** — the Express backend rejected browser requests because `ALLOWED_ORIGINS` defaulted to `localhost:3000`. Added the AppGW public IP to the allowed list. Terraform now injects this automatically at provisioning time.

**Key takeaway:** Infrastructure as Code defines the skeleton. The provisioning scripts are the muscle. Both need the same rigour.

Huge thanks to the **DMI Cohort** for the continuous learning journey!

#DevOps #Azure #Terraform #CloudEngineering #NodeJS #NextJS #InfrastructureAsCode #Automation #ThreeTierArchitecture

---

### What I Learned

Writing about what went wrong is more valuable than writing about what worked. Documenting the debugging process — the wrong assumptions, the diagnostic commands, the root causes — creates a reference that is useful both for the author and for anyone else hitting the same issues. Summarising the issues for the post also reinforced a clear troubleshooting methodology: read the error, identify the layer (network → app → config), isolate before fixing.

---

### Issues Faced and How I Solved Them

**Issue — AWS and Azure credentials accidentally committed**
When running `git push origin main` after staging all files, GitHub's push protection blocked the push because `.mcp.json` contained live AWS access keys and an Azure Service Principal secret. Immediately redacted all secret values in `.mcp.json`, added `.mcp.json` to `.gitignore`, and amended the commit with `git commit --amend --no-edit` before pushing successfully. The exposed credentials were then rotated: the AWS IAM key was deactivated and deleted, and the Azure AD application secret was deleted and regenerated.

---

### Screenshots

| # | Screenshot |
|---|---|
| 3.1 | LinkedIn — Published post showing text and architecture image |
| 3.2 | LinkedIn — Post visibility set to "Anyone" |

**LinkedIn Post URL:** `___________________________`

---

## Submission Summary

| Task | Item | Status |
|---|---|---|
| Task 1 | Cloud platform identified | ✅ Microsoft Azure |
| Task 1 | Terraform code structure documented | ✅ 11 files described |
| Task 1 | Architecture diagram included | ✅ |
| Task 1 | Public Load Balancer URL | ✅ `http://20.87.49.209` |
| Task 2 | Frontend & Backend VM Dashboard | ✅ |
| Task 2 | MySQL Database Dashboard | ✅ |
| Task 2 | Functional App UI (Homepage + Login + Review) | ✅ |
| Task 2 | Working API and DB integration | ✅ |
| Task 3 | LinkedIn post written | ✅ |
| Task 3 | LinkedIn post published (public) | ⬜ Paste URL above |
