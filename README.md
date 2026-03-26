# Secure Azure Infrastructure Deployment

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=flat&logo=microsoft-azure&logoColor=white)
![Status](https://img.shields.io/badge/Status-Complete-success)

## Overview

A secure cloud infrastructure environment built on Microsoft Azure, demonstrating real-world implementation of network security, identity and access management, secrets management, and security monitoring. This project mirrors enterprise-grade security architecture and was built entirely using the Azure Portal and Azure CLI.

## Technologies Used

- Microsoft Azure
- Azure Virtual Network + Subnets
- Azure Virtual Machines (Ubuntu 22.04)
- Network Security Groups
- Azure Key Vault
- Role-Based Access Control (RBAC)
- Managed Identities
- Azure Monitor + Log Analytics
- Azure CLI (PowerShell)
- SSH Key Authentication

## Architecture
```
Resource Group: secure-infra-rg (East US)
│
├── Virtual Network (secure-vnet) 10.0.0.0/16
│     └── Subnet (default-subnet) 10.0.1.0/24
│           └── Network Security Group (secure-nsg)
│                 └── Inbound Rule: SSH allowed from admin IP only
│
├── Virtual Machine (secure-vm)
│     ├── Ubuntu 22.04 LTS
│     ├── SSH key authentication
│     └── System-assigned managed identity
│
├── Key Vault (secure-kv-brayden)
│     ├── RBAC authorization enabled
│     ├── Secrets stored (db-password)
│     ├── Role: Key Vault Administrator → Brayden Miller
│     └── Role: Key Vault Secrets User → secure-vm managed identity
│
└── Monitoring
      ├── Log Analytics Workspace (secure-logs)
      ├── Diagnostic settings → Key Vault + VM
      ├── Alert: KeyVault-Secret-Access (fires on SecretGet events)
      └── Alert: VM-Deallocation-Alert (fires on VM shutdown)
```

## Implementation Details

### Day 1 — Network Foundation
Built the core network infrastructure using Azure CLI. Created a Virtual Network with a /16 address space providing 65,000+ available IP addresses, and carved out a /24 subnet within it. Attached a Network Security Group to the subnet with default deny-all inbound rules, ensuring no traffic reaches resources unless explicitly permitted.

**Key decision:** Used NSGs as a cost-effective alternative to Azure Firewall, demonstrating the same network traffic filtering principles at zero cost.

### Day 2 — Virtual Machine Deployment
Deployed an Ubuntu 22.04 LTS VM inside the private subnet using spot pricing to minimize cost. Configured SSH key pair authentication and added a custom NSG inbound rule restricting SSH access on port 22 to a single authorized IP address. Successfully connected to the VM from a local machine via SSH to verify security controls.

**Key decision:** Restricted SSH access to a specific IP address rather than using Azure Bastion, implementing least privilege access at the network layer.

### Day 3 — Key Vault and RBAC
Created an Azure Key Vault with RBAC authorization enabled — the modern best practice over legacy access policies. Assigned the Key Vault Administrator role to the admin user and the Key Vault Secrets User role to the VM's managed identity. Enabled a system-assigned managed identity on the VM and verified end-to-end secret retrieval from inside the VM using only the managed identity — no hardcoded credentials anywhere in the system.

**Key decision:** Used managed identities instead of service principals with secrets, eliminating credential management overhead and reducing attack surface.

### Day 4 — Monitoring and Alerts
Created a Log Analytics workspace as the central log collection point. Configured diagnostic settings on both the Key Vault and VM to stream logs into the workspace. Built two alert rules — one that fires on Key Vault secret access events using a custom KQL query, and one that fires on VM deallocation. Verified the pipeline end-to-end by triggering a real secret access event and receiving the email notification within 5 minutes.

**Key decision:** Used log query based alerts with custom KQL rather than relying solely on built-in metrics, enabling detection of specific security-relevant events.

## CLI Command Reference

### Resource Group
```bash
az group create --name secure-infra-rg --location eastus
```

### Virtual Network and Subnet
```bash
az network vnet create --resource-group secure-infra-rg --name secure-vnet --address-prefix 10.0.0.0/16 --subnet-name default-subnet --subnet-prefix 10.0.1.0/24
```

### Network Security Group
```bash
az network nsg create --resource-group secure-infra-rg --name secure-nsg

az network vnet subnet update --resource-group secure-infra-rg --vnet-name secure-vnet --name default-subnet --network-security-group secure-nsg
```

### Virtual Machine
```bash
az vm create --resource-group secure-infra-rg --name secure-vm --image Ubuntu2204 --size Standard_B2ps_v2 --admin-username azureuser --generate-ssh-keys --vnet-name secure-vnet --subnet default-subnet --nsg secure-nsg --public-ip-sku Standard
```

### NSG Rule — Restrict SSH to Admin IP
```bash
az network nsg rule create --resource-group secure-infra-rg --nsg-name secure-nsg --name allow-ssh-myip --protocol Tcp --priority 100 --destination-port-range 22 --source-address-prefix <YOUR-IP> --access Allow --direction Inbound
```

### Key Vault
```bash
az keyvault create --name secure-kv-brayden --resource-group secure-infra-rg --location eastus --enable-rbac-authorization true
```

### RBAC Role Assignments
```bash
# Assign Key Vault Administrator to admin user
az role assignment create --role "Key Vault Administrator" --assignee <USER-ID> --scope /subscriptions/<SUB-ID>/resourceGroups/secure-infra-rg/providers/Microsoft.KeyVault/vaults/secure-kv-brayden

# Assign Key Vault Secrets User to VM managed identity
az role assignment create --role "Key Vault Secrets User" --assignee <VM-IDENTITY-ID> --scope /subscriptions/<SUB-ID>/resourceGroups/secure-infra-rg/providers/Microsoft.KeyVault/vaults/secure-kv-brayden
```

### Store a Secret
```bash
az keyvault secret set --vault-name secure-kv-brayden --name "db-password" --value "SecureP@ssword123!"
```

### Enable VM Managed Identity
```bash
az vm identity assign --resource-group secure-infra-rg --name secure-vm
```

### Log Analytics Workspace
```bash
az monitor log-analytics workspace create --resource-group secure-infra-rg --workspace-name secure-logs --location eastus
```

### Deallocate VM (stop billing)
```bash
az vm deallocate --resource-group secure-infra-rg --name secure-vm
```

## Security Decisions

| Decision | What Was Used | Production Alternative | Reason Not Used |
|---|---|---|---|
| Network filtering | Network Security Group | Azure Firewall | $900/month cost |
| Remote access | SSH restricted to admin IP | Azure Bastion | $140/month cost |
| DDoS protection | Azure basic (free) | Azure DDoS Standard | $2,900/month cost |
| VM authentication | SSH key pair | Same | Best practice at any scale |
| Secret management | Key Vault + managed identity | Same | Best practice at any scale |
| Disk encryption | Platform-managed keys | Customer-managed keys in Key Vault | Sufficient for lab scope |

## What I Would Add in Production

- **Azure Firewall** with application and network rules for centralized traffic inspection
- **Azure Bastion** to eliminate the public IP on the VM entirely
- **Azure DDoS Protection Standard** for volumetric attack mitigation
- **Customer-managed encryption keys** stored in Key Vault for compliance requirements
- **Multiple subnets** segmented by tier — web, application, and data layers
- **Azure Private Endpoint** for Key Vault to remove public network access entirely
- **Microsoft Defender for Cloud** for continuous security posture assessment
- **Privileged Identity Management** for just-in-time access to Key Vault
- **Azure Policy** to enforce security standards across all resources automatically
- **Geo-redundant deployment** across multiple regions for high availability

## Screenshots

### Resource Group
![Resource Group](screenshots/resource-group.png)

### Virtual Network and Subnet
![Virtual Network](screenshots/vnet-subnet.png)

### Network Security Group Rules
![NSG Rules](screenshots/nsg-rules.png)

### Virtual Machine
![Virtual Machine](screenshots/vm-overview.png)

### Key Vault
![Key Vault](screenshots/keyvault.png)

### RBAC Role Assignments
![RBAC](screenshots/rbac-roles.png)

### SSH Connection
![SSH](screenshots/ssh-connection.png)

### Secret Retrieval via Managed Identity
![Secret Retrieval](screenshots/secret-retrieval.png)

### Alert Rules
![Alerts](screenshots/alerts.png)

### Alert Email Notification
![Alert Email](screenshots/alert-email.png)

## Lessons Learned

- Azure for Students subscriptions have significant compute quota restrictions — working through quota requests and capacity issues is a real cloud engineering skill
- PowerShell and Bash have different line continuation syntax — `\` works in Bash but not PowerShell
- Managed identities eliminate credential management entirely — no passwords, no keys, no rotation required
- Even as the resource owner, explicit RBAC role assignments are required to interact with Key Vault contents — zero implicit trust
- Spot pricing can reduce VM costs by 60-90% and is viable for non-critical workloads

## Author

**Brayden Miller**  
[LinkedIn](https://www.linkedin.com/in/brayden-miller13/) | [GitHub](https://github.com/BraydenMiller-CloudSec)

---
*Built as part of a hands-on Azure cloud security portfolio. See my other projects on GitHub.*
