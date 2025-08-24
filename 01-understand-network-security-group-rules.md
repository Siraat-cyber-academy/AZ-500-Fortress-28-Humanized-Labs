# Understand Network Security Group rules

📝 This is the placeholder file for **Understand Network Security Group rules**.

---

# 🛡️ Lab: Understand Network Security Group (NSG) Rules

## Overview

Learn how NSGs control inbound/outbound traffic at **subnet** and **NIC** level; how **priority & rule evaluation** work; how to use **service tags** and **ASGs**; how to **validate** with real VMs; and how to read **Effective Security Rules** and **NSG Flow Logs**.

## Learning Objectives

* Explain NSG **default rules** and **priority processing**
* Create & associate NSGs with **subnets** (optionally NICs)
* Use **service tags** (Internet, VirtualNetwork) and **source filters**
* Validate traffic (allow/deny) between Internet ⇄ Web VM ⇄ App VM
* Inspect **Effective Security Rules** and (optional) **Flow Logs**

## Reference Topology

* **VNet**: `nsg-lab-vnet` (10.10.0.0/16)

  * `web-subnet` (10.10.1.0/24)
  * `app-subnet` (10.10.2.0/24)
* **NSGs**: `nsg-web`, `nsg-app` (associated to subnets)
* **VMs**:

  * `web-vm` (Ubuntu, **public IP** in `web-subnet`)
  * `app-vm` (Ubuntu, **no public IP** in `app-subnet`)

## What You’ll Build (Policies)

* **Allow HTTP (80) from Internet → web-vm**
* **Allow SSH (22) only from your client public IP → web-vm**
* **Allow App HTTP (8080) only from web-subnet → app-vm**
* All else denied by defaults

---

## Prerequisites

* Azure subscription + Owner/Contributor on target RG
* **Azure CLI** logged in: `az login`
* Your **public IP** (for SSH restriction): visit `ifconfig.me` or similar

---

## 1) Deploy Core Infra (CLI)

> Create RG, VNet, subnets, and NSGs.

```bash
# Variables (edit your initials/location)
LOC=westeurope
RG=nsg-lab-rg
VNET=nsg-lab-vnet

az group create -n $RG -l $LOC

# VNet + subnets
az network vnet create \
  -g $RG -n $VNET --address-prefix 10.10.0.0/16 \
  --subnet-name web-subnet --subnet-prefix 10.10.1.0/24

az network vnet subnet create \
  -g $RG --vnet-name $VNET -n app-subnet --address-prefix 10.10.2.0/24

# NSGs
az network nsg create -g $RG -n nsg-web
az network nsg create -g $RG -n nsg-app
```

### Associate NSGs to Subnets

```bash
az network vnet subnet update \
  -g $RG --vnet-name $VNET -n web-subnet \
  --network-security-group nsg-web

az network vnet subnet update \
  -g $RG --vnet-name $VNET -n app-subnet \
  --network-security-group nsg-app
```

---

## 2) Author NSG Rules

> NSG rule evaluation: **lowest priority wins** (100–4096 custom). Defaults (65000+): AllowVNet, AllowLoadBalancer, then **DenyAll**.

### `nsg-web` (Internet → Web)

```bash
MYIP=$(curl -s ifconfig.me)  # your client public IP

# Allow HTTP from Internet to web-subnet
az network nsg rule create -g $RG --nsg-name nsg-web -n allow-http \
  --priority 100 --access Allow --direction Inbound --protocol Tcp \
  --source-address-prefixes Internet --source-port-ranges "*" \
  --destination-address-prefixes "*" --destination-port-ranges 80

# Allow SSH only from your IP
az network nsg rule create -g $RG --nsg-name nsg-web -n allow-ssh-from-myip \
  --priority 110 --access Allow --direction Inbound --protocol Tcp \
  --source-address-prefixes ${MYIP}/32 --source-port-ranges "*" \
  --destination-address-prefixes "*" --destination-port-ranges 22
```

### `nsg-app` (Web → App)

```bash
# Allow App HTTP on 8080 only from web-subnet
az network nsg rule create -g $RG --nsg-name nsg-app -n allow-8080-from-web \
  --priority 100 --access Allow --direction Inbound --protocol Tcp \
  --source-address-prefixes 10.10.1.0/24 --source-port-ranges "*" \
  --destination-address-prefixes "*" --destination-port-ranges 8080

# Optional: allow SSH from web-subnet (for admin hops)
az network nsg rule create -g $RG --nsg-name nsg-app -n allow-ssh-from-web \
  --priority 110 --access Allow --direction Inbound --protocol Tcp \
  --source-address-prefixes 10.10.1.0/24 --source-port-ranges "*" \
  --destination-address-prefixes "*" --destination-port-ranges 22
```

> **Tip**: We rely on **subnet-level NSGs** only. Avoid creating NIC-level NSGs in this lab to keep rule evaluation simple.

---

## 3) Create VMs

> Create one **public** web VM and one **private** app VM.
> To avoid auto–NIC NSGs, we’ll provision NICs manually.

```bash
# Public IP for web
az network public-ip create -g $RG -n web-pip --sku Basic --allocation-method Dynamic

# NICs (no NSGs at NIC level)
az network nic create -g $RG -n web-nic \
  --vnet-name $VNET --subnet web-subnet --public-ip-address web-pip

az network nic create -g $RG -n app-nic \
  --vnet-name $VNET --subnet app-subnet

# SSH key (if you don't already have one)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/azure_nsg_lab -N "" 2>/dev/null || true

# VMs
az vm create -g $RG -n web-vm --image Ubuntu2204 \
  --size Standard_B1s --admin-username azureuser \
  --ssh-key-values ~/.ssh/azure_nsg_lab.pub \
  --nics web-nic

az vm create -g $RG -n app-vm --image Ubuntu2204 \
  --size Standard_B1s --admin-username azureuser \
  --ssh-key-values ~/.ssh/azure_nsg_lab.pub \
  --nics app-nic
```

Fetch the web VM public IP:

```bash
az vm show -g $RG -n web-vm -d --query publicIps -o tsv
```

---

## 4) Configure Simple Services

### On `web-vm`: NGINX (port 80)

```bash
ssh -i ~/.ssh/azure_nsg_lab azureuser@<WEB_PUBLIC_IP>
sudo apt update && sudo apt -y install nginx
curl -I http://localhost
```

### On `app-vm`: Simple HTTP server (port 8080)

From `web-vm`, hop privately:

```bash
# Find app private IP
APPIP=$(hostname -I | awk '{print $1}')  # (run on app-vm to see it)
# From web-vm:
ssh -i ~/.ssh/azure_nsg_lab azureuser@10.10.2.4   # (replace with app-vm's IP)
# On app-vm:
sudo apt update && sudo apt -y install python3
nohup python3 -m http.server 8080 >/dev/null 2>&1 &
curl -I http://localhost:8080
```

---

## 5) Validate NSG Behavior

**From your laptop (Internet → web-vm):**

* ✅ `curl http://<WEB_PUBLIC_IP>` → **200 OK** (allowed by `allow-http`)
* ✅ `ssh azureuser@<WEB_PUBLIC_IP>` → **works only from your IP**
* ❌ Try any other port (e.g., 3389/445) → **blocked by defaults**

**From web-vm (web → app):**

* ✅ `curl http://<APP_PRIVATE_IP>:8080` → **200 OK** (allowed by `allow-8080-from-web`)
* ❌ `curl http://<APP_PRIVATE_IP>:80` → **blocked** (no allow rule → hits DenyAllInBound)

---

## 6) See “Effective Security Rules” (Portal)

Azure Portal → **Virtual machines** → `web-vm` → **Networking** → select NIC → **Effective security rules**

* Observe merged rules (subnet-level NSG + defaults)
* Confirm why HTTP/22 allowed, others denied

Repeat for `app-vm`.

---

## 7) (Optional) Enable NSG Flow Logs

* **Network Watcher** → NSG Flow Logs → enable on `nsg-web` / `nsg-app`
* Storage account required
* Inspect flows (allowed/denied) for validation & learning

---

## 8) Priority Drill (Rule Evaluation Demo)

Create a **higher-priority deny** on `nsg-web` to block HTTP:

```bash
az network nsg rule create -g $RG --nsg-name nsg-web -n deny-http-temp \
  --priority 90 --access Deny --direction Inbound --protocol Tcp \
  --source-address-prefixes Internet --destination-port-ranges 80 --source-port-ranges "*" \
  --destination-address-prefixes "*"
```

* Now `curl http://<WEB_PUBLIC_IP>` from your laptop → **fails** (deny at priority 90 beats allow at 100).
* **Delete** the deny rule to restore service:

```bash
az network nsg rule delete -g $RG --nsg-name nsg-web -n deny-http-temp
```

---

## 9) Clean Up

```bash
az group delete -n $RG --yes --no-wait
```

---

## Key Concepts (Interview-Grade Recap)

* **Default Inbound**: AllowVNetIn (65000), AllowLoadBalancerIn (65001), **DenyAllIn (65500)**
* **Default Outbound**: AllowVNetOut (65000), AllowInternetOut (65001), **DenyAllOut (65500)**
* **Lower priority number = evaluated first**
* **Association**: NSG can attach to **subnet** and/or **NIC** (both apply; most restrictive wins)
* **Service Tags** (e.g., Internet, VirtualNetwork) simplify source/destination scopes
* **ASGs** group NICs logically for scalable rules

---

### Troubleshooting Tips

* SSH blocked? Confirm your **public IP** in rule matches current IP.
* HTTP to app-vm from web-vm failing? Check **port (8080)** and **source prefix (web-subnet)**.
* Still confused? Use **Effective Security Rules** first, then **NSG Flow Logs**.


---

✒️ **Closing Signature**  
✍️ Created & Curated by  
**Muhammad Naveed Ishaque (Eks2)**  
Content Creator | AI Writer | Narrative Simplifier  
With the inner voice of Eks2 — the whisper behind the work.  

🕊️ **Siraat AI Academy**  
_“The Straight Path — Empowering minds with clarity, illuminating paths with purpose.”_  
