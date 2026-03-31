# Book Review App — Azure Deployment Guide

This guide walks through deploying the Book Review application on Azure using Terraform. The infrastructure is fully automated: a single `terraform apply` provisions the network, virtual machines, database, load balancer, and application gateway, and a cloud-init script configures each VM on first boot.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Bootstrap the Terraform State Backend](#step-1--bootstrap-the-terraform-state-backend)
4. [Step 2 — Configure Variables](#step-2--configure-variables)
5. [Step 3 — Deploy the Infrastructure](#step-3--deploy-the-infrastructure)
6. [Step 4 — Verify the Deployment](#step-4--verify-the-deployment)
7. [Step 5 — Access the Application](#step-5--access-the-application)
8. [Updating the Application](#updating-the-application)
9. [Troubleshooting](#troubleshooting)
10. [Destroying the Infrastructure](#destroying-the-infrastructure)

---

## Architecture Overview

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│   Application Gateway (AppGW)   │  ← Public entry point, path-based routing
│   Standard_v2  │  port 80       │
└────────┬────────────────────────┘
         │ /api/*              │ /*
         ▼                     ▼
┌─────────────────┐   ┌─────────────────┐
│  Internal LB    │   │  Frontend VM    │  ← Next.js + Nginx (web subnet)
│  port 3001      │   │  10.0.1.x       │
└────────┬────────┘   └─────────────────┘
         │
         ▼
┌─────────────────┐
│  Backend VM     │  ← Node.js/Express + PM2 (app subnet)
│  10.0.2.x       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  MySQL Flexible │  ← Azure Database for MySQL (db subnet, private)
│  Server         │
└─────────────────┘
```

**Subnets:**

| Subnet | CIDR | Purpose |
|---|---|---|
| snet-appgw | 10.0.4.0/24 | Application Gateway |
| snet-web | 10.0.1.0/24 | Frontend VM |
| snet-app | 10.0.2.0/24 | Backend VM + Internal LB |
| snet-db | 10.0.3.0/24 | MySQL Flexible Server (delegated) |

**Key design decisions:**
- The backend VM has no public IP — all internet traffic routes through the AppGW
- The database is isolated in a private subnet with a Private DNS Zone
- A NAT Gateway provides outbound internet access for both web and app subnets
- PM2 manages the Node.js process and restarts it on VM reboot via systemd

---

## Prerequisites

### Required tools

| Tool | Minimum version | Install |
|---|---|---|
| Terraform | >= 1.5.0 | https://developer.hashicorp.com/terraform/install |
| Azure CLI | >= 2.50 | `curl -sL https://aka.ms/InstallAzureCLIDeb \| sudo bash` |
| Git | any | `sudo apt install git` |

### Required accounts and permissions

- **Azure subscription** with Contributor or Owner role
- **GitHub account** with access to the `book-review-app` repository
- **SSH key pair** on your local machine (`~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`)

Generate an SSH key if you do not have one:

```bash
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"
```

### Authenticate to Azure

```bash
az login
az account set --subscription "<your-subscription-id>"

# Confirm the correct subscription is active
az account show --query "{name:name, id:id}" -o table
```

---

## Step 1 — Bootstrap the Terraform State Backend

Terraform state must be stored remotely before the first `terraform init`. The state storage account is created manually and lives outside Terraform's own management.

> **Why separate?** If Terraform managed its own state storage, `terraform destroy` would delete the state file — leaving orphaned resources with no way to manage them.

**1a. Create a dedicated resource group for state:**

```bash
az group create \
  --name tfstate-rg \
  --location "South Africa North"
```

**1b. Create the storage account (name must be globally unique, 3–24 lowercase alphanumeric chars, no hyphens):**

```bash
az storage account create \
  --name <your-unique-tfstate-sa> \
  --resource-group tfstate-rg \
  --location "South Africa North" \
  --sku Standard_LRS \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2
```

**1c. Create the blob container:**

```bash
az storage container create \
  --name tfstate \
  --account-name <your-unique-tfstate-sa>
```

**1d. Update `terraform/backend.tf` with your storage account name:**

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "<your-unique-tfstate-sa>"   # ← change this
    container_name       = "tfstate"
    key                  = "book-review-app/prod/terraform.tfstate"
  }
}
```

---

## Step 2 — Configure Variables

### 2a. Find your public IP address

The NSG restricts SSH access to your IP only:

```bash
curl -s ifconfig.me
# Example output: 203.0.113.5
```

### 2b. Update `terraform/terraform.tfvars`

```hcl
repo_url    = "https://github.com/dowusubekoe-dev/book-review-app.git"
my_safe_ip  = "203.0.113.5/32"       # your public IP with /32 suffix
location    = "South Africa North"
environment = "prod"
```

### 2c. Set the database password as an environment variable

The password is never written to disk or version control:

```bash
export TF_VAR_db_admin_password="<your-strong-password>"
```

**Password requirements (Azure MySQL):**
- Minimum 8 characters
- Must include uppercase, lowercase, number, and special character
- Example format: `MyP@ssw0rd2026!`

Verify it is set:

```bash
echo $TF_VAR_db_admin_password
```

> **Note:** `export` only persists for the current shell session. Re-run the export if you open a new terminal before applying.

---

## Step 3 — Deploy the Infrastructure

```bash
cd terraform

# Initialise — downloads provider, configures remote state backend
terraform init

# Preview all resources that will be created
terraform plan

# Deploy (takes approx. 10–15 minutes)
terraform apply
```

Type `yes` when prompted.

### What gets created

| Resource | Count | Notes |
|---|---|---|
| Resource Group | 1 | All app resources |
| Virtual Network | 1 | 10.0.0.0/16 |
| Subnets | 4 | web, app, db, appgw |
| NSGs | 4 | One per subnet with tiered rules |
| NAT Gateway | 1 | Outbound internet for web + app subnets |
| Frontend VM | 1 | Next.js + Nginx |
| Backend VM | 1 | Node.js + PM2 (auto-provisioned via cloud-init) |
| Internal Load Balancer | 1 | Stable private IP for AppGW → backend |
| Application Gateway | 1 | Standard_v2, path-based routing |
| MySQL Flexible Server | 1 | Private, SSL enforced |
| MySQL Database | 1 | `book_review_db` |
| Private DNS Zone | 1 | For MySQL private resolution |
| Public IPs | 3 | AppGW, frontend VM, NAT Gateway |

### Backend VM provisioning (automatic)

The backend VM runs a cloud-init script on first boot that:

1. Installs Node.js 20.x via NodeSource
2. Installs PM2 globally
3. Clones the repository from `var.repo_url`
4. Writes the `.env` file with Terraform-interpolated DB credentials
5. Runs `npm install --omit=dev`
6. Starts the app under PM2 as `azureuser`
7. Registers PM2 as a systemd service for reboot persistence

Monitor provisioning progress via the Azure portal (VM → Boot diagnostics → Serial log) or:

```bash
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-be-vm \
  --command-id RunShellScript \
  --scripts "tail -50 /var/log/backend-init.log" \
  --query "value[0].message" -o tsv
```

---

## Step 4 — Verify the Deployment

### 4a. Check Terraform outputs

```bash
terraform output
```

Expected output:

```
appgw_public_ip     = "20.x.x.x"
application_url     = "http://20.x.x.x"
backend_private_ip  = "10.0.2.x"
frontend_public_ip  = "4.x.x.x"
mysql_database_name = "book_review_db"
mysql_fqdn          = "bookreview-prod-db-server.mysql.database.azure.com"
```

### 4b. Verify the backend API is responding

```bash
curl -s http://$(terraform output -raw appgw_public_ip)/api/books | head -c 200
```

Expected: a JSON array of books.

### 4c. Verify the backend VM process

```bash
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-be-vm \
  --command-id RunShellScript \
  --scripts "sudo -u azureuser pm2 list" \
  --query "value[0].message" -o tsv
```

Expected: `book-review-backend` with status `online`.

### 4d. Test user registration

```bash
curl -s -w "\nHTTP %{http_code}" \
  -X POST http://$(terraform output -raw appgw_public_ip)/api/users/register \
  -H "Content-Type: application/json" \
  -H "Origin: http://$(terraform output -raw appgw_public_ip)" \
  -d '{"email":"test@example.com","password":"Test1234!"}'
```

Expected: HTTP 201 with a JWT token in the response body.

---

## Step 5 — Access the Application

Open a browser and navigate to:

```
http://<appgw_public_ip>
```

Use the `application_url` output for the exact address:

```bash
terraform output application_url
```

---

## Updating the Application

### Update backend code

After pushing code changes to GitHub, re-deploy on the backend VM:

```bash
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-be-vm \
  --command-id RunShellScript \
  --scripts "cd /home/azureuser/app && git pull && cd backend && npm install --omit=dev && sudo -u azureuser pm2 restart book-review-backend" \
  --query "value[0].message" -o tsv
```

### Update frontend code

After pushing code changes to GitHub, rebuild and restart on the frontend VM:

```bash
# Pull latest code
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-fe-vm \
  --command-id RunShellScript \
  --scripts "cd /home/azureuser/book-review-app && git pull" \
  --query "value[0].message" -o tsv

# Rebuild Next.js (takes 2-3 minutes)
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-fe-vm \
  --command-id RunShellScript \
  --scripts "cd /home/azureuser/book-review-app/frontend && npm run build" \
  --query "value[0].message" -o tsv

# Restart the frontend process
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-fe-vm \
  --command-id RunShellScript \
  --scripts "sudo -u azureuser pm2 restart frontend" \
  --query "value[0].message" -o tsv
```

### Update infrastructure

```bash
cd terraform
terraform plan   # review changes first
terraform apply
```

---

## Troubleshooting

### State lock error

```
Error: state blob is already locked
```

**Cause:** A previous Terraform operation was interrupted without releasing the lock.

**Fix:**
```bash
terraform force-unlock <lock-id>
# The lock ID is shown in the error message
```

---

### SSH connection refused or timed out

**Check 1 — Confirm your current public IP matches `my_safe_ip`:**

```bash
curl -s ifconfig.me
# Compare with my_safe_ip in terraform.tfvars
```

If different, update `terraform.tfvars` and run `terraform apply`.

**Check 2 — Confirm the VM is running:**

```bash
az vm get-instance-view \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-fe-vm \
  --query "instanceView.statuses[1].displayStatus" -o tsv
```

**Check 3 — Test port 22 reachability:**

```bash
nc -zv <frontend-public-ip> 22
```

**Check 4 — After VM recreation, remove stale host key:**

```bash
ssh-keygen -f '~/.ssh/known_hosts' -R '<vm-ip>'
```

---

### Website loads but no books displayed (502 Bad Gateway)

**Cause:** The backend app is not running on port 3001.

**Diagnose:**

```bash
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-be-vm \
  --command-id RunShellScript \
  --scripts "ss -tlnp | grep 3001" \
  --query "value[0].message" -o tsv
```

If no output, the app is not running. Check the init log:

```bash
az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-be-vm \
  --command-id RunShellScript \
  --scripts "tail -30 /var/log/backend-init.log" \
  --query "value[0].message" -o tsv
```

---

### API calls return 404

**Cause:** The frontend is calling the wrong API path.

Correct API paths routed through the AppGW (`/api/*`):

| Operation | Method | Path |
|---|---|---|
| Register | POST | `/api/users/register` |
| Login | POST | `/api/users/login` |
| List books | GET | `/api/books` |
| Get book | GET | `/api/books/:id` |
| Get reviews | GET | `/api/reviews/:bookId` |
| Submit review | POST | `/api/reviews` |

---

### API calls return 500 (CORS error)

**Cause:** `ALLOWED_ORIGINS` in the backend `.env` does not include the AppGW public IP.

**Fix:**

```bash
APPGW_IP=$(cd terraform && terraform output -raw appgw_public_ip)

az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-be-vm \
  --command-id RunShellScript \
  --scripts "echo 'ALLOWED_ORIGINS=http://${APPGW_IP}' >> /home/azureuser/app/backend/.env && sudo -u azureuser pm2 restart book-review-backend" \
  --query "value[0].message" -o tsv
```

---

### Frontend shows `localhost:3001` errors in browser console

**Cause:** `NEXT_PUBLIC_API_URL` was not set when the Next.js app was built.

**Fix — set the env var and rebuild:**

```bash
APPGW_IP=$(cd terraform && terraform output -raw appgw_public_ip)

az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-fe-vm \
  --command-id RunShellScript \
  --scripts "echo 'NEXT_PUBLIC_API_URL=http://${APPGW_IP}' > /home/azureuser/book-review-app/frontend/.env.local" \
  --query "value[0].message" -o tsv

az vm run-command invoke \
  --resource-group bookreview-prod-southafricanorth-rg \
  --name bookreview-prod-southafricanorth-fe-vm \
  --command-id RunShellScript \
  --scripts "cd /home/azureuser/book-review-app/frontend && npm run build && sudo -u azureuser pm2 restart frontend" \
  --query "value[0].message" -o tsv
```

> **Important:** `NEXT_PUBLIC_*` variables are baked into the JavaScript bundle at build time. The app must be rebuilt whenever this value changes.

---

## Destroying the Infrastructure

```bash
cd terraform
terraform destroy
```

Type `yes` when prompted. This removes all resources in the application resource group.

> **Note:** The Terraform state storage account (`tfstate-rg`) is **not** managed by Terraform and will **not** be destroyed. Delete it manually if no longer needed:
>
> ```bash
> az group delete --name tfstate-rg --yes
> ```
