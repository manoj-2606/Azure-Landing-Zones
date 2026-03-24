# ALZ Day 7 -- Writing Real Terraform ALZ Code

> **The single goal of Day 7:**  
> Write actual Terraform code for a landing zone spoke using AVM modules --  
> not copy-paste, not reading someone else's code --  
> writing it yourself, running terraform plan, and being able to explain  
> every single line to a code reviewer or a client.
>
> **Time Required:** 2.5 hours  
> **Split:** 30 min setup | 90 min writing code | 30 min running plan and reading output
>
> **Hard requirement:** Terraform must be installed and `terraform version` must work  
> before you open this file further. If it is not installed yet, stop here and install it.  
> Download: https://developer.hashicorp.com/terraform/install

---

## What You Are Building Today

A single ALZ landing zone spoke -- expressed entirely in Terraform using AVM modules.

By the end of Day 7 you will have written and successfully planned:

```
alz-spoke-day7/
├── main.tf          # All AVM module calls
├── variables.tf     # Input variables
├── outputs.tf       # Exported values
├── providers.tf     # AzureRM provider configuration
└── terraform.tfvars # Actual values for this landing zone
```

This is a real, deployable landing zone spoke.  
On Day 10 you will deploy it to an actual Azure subscription.  
Today you write it and run terraform plan.

---

## Section 1 -- Environment Setup (Do This First)

### Step 1: Install Azure CLI and Login

```bash
# Verify Azure CLI is installed
az --version

# Login to Azure
az login

# Set your target subscription
az account set --subscription "your-subscription-id"

# Verify you are in the right subscription
az account show
```

### Step 2: Verify Terraform Installation

```bash
terraform version
# Should return: Terraform v1.x.x
```

### Step 3: Create Your Working Directory

```bash
mkdir alz-spoke-day7
cd alz-spoke-day7
```

### Step 4: Create the Required Files

```bash
touch main.tf variables.tf outputs.tf providers.tf terraform.tfvars
```

---

## Section 2 -- providers.tf (Write This First)

The providers file tells Terraform which provider to use and configures authentication.

```hcl
terraform {
  required_version = ">= 1.3.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  # Authentication uses Azure CLI login for local development
  # In the pipeline this uses OIDC or Service Principal
}
```

**What each block does:**
- `required_version` -- minimum Terraform version needed to run this code
- `required_providers` -- declares which providers to download from the registry
- `provider "azurerm"` -- configures the Azure provider
- `features {}` -- required block for AzureRM provider, controls optional behaviours

---

## Section 3 -- variables.tf (Write This Second)

Variables make your code reusable. The same code deploys any landing zone  
by changing the values in terraform.tfvars.

```hcl
variable "location" {
  description = "Azure region where all resources will be deployed"
  type        = string
  default     = "eastus"
}

variable "landing_zone_name" {
  description = "Short name for this landing zone -- used in resource naming"
  type        = string
  # Example: "a1", "a2", "p1"
}

variable "vnet_address_space" {
  description = "Address space for the spoke VNet -- must not overlap with hub or other spokes"
  type        = list(string)
  # Example: ["10.1.0.0/24"]
}

variable "app_subnet_prefix" {
  description = "CIDR prefix for the application tier subnet"
  type        = string
  # Example: "10.1.0.0/26"
}

variable "db_subnet_prefix" {
  description = "CIDR prefix for the database tier subnet"
  type        = string
  # Example: "10.1.0.64/26"
}

variable "pe_subnet_prefix" {
  description = "CIDR prefix for the private endpoint subnet"
  type        = string
  # Example: "10.1.0.128/27"
}

variable "hub_firewall_private_ip" {
  description = "Private IP of the Azure Firewall in the hub -- used for UDR next hop"
  type        = string
  # Example: "10.0.0.4"
}

variable "app_team_group_object_id" {
  description = "Object ID of the Entra ID group for the application team"
  type        = string
}

variable "tags" {
  description = "Tags applied to all resources in this landing zone"
  type        = map(string)
  default = {
    managed-by  = "terraform"
    environment = "production"
  }
}
```

---

## Section 4 -- terraform.tfvars (Write This Third)

This file provides actual values for this specific landing zone.  
In a real ALZ, each landing zone has its own tfvars file.  
Replace the placeholder values with real values from your Azure subscription.

```hcl
location          = "eastus"
landing_zone_name = "a1"

vnet_address_space = ["10.1.0.0/24"]
app_subnet_prefix  = "10.1.0.0/26"
db_subnet_prefix   = "10.1.0.64/26"
pe_subnet_prefix   = "10.1.0.128/27"

hub_firewall_private_ip = "10.0.0.4"
# Replace with your actual hub firewall IP if you have one
# Use "10.0.0.4" as a placeholder if you do not have a hub yet

app_team_group_object_id = "00000000-0000-0000-0000-000000000000"
# Replace with an actual Entra ID group object ID from your tenant

tags = {
  managed-by    = "terraform"
  environment   = "production"
  landing-zone  = "a1"
  team          = "app-team-a"
  cost-centre   = "CC-1001"
}
```

---

## Section 5 -- main.tf (The Main Event)

This is where you call AVM modules to build the landing zone.  
Write each block one at a time. Understand it before moving to the next.

### Block 1: Resource Group

```hcl
# Standard azurerm resource -- not an AVM module
# Resource groups are simple enough not to need a module wrapper
resource "azurerm_resource_group" "spoke" {
  name     = "rg-lz-${var.landing_zone_name}-prod"
  location = var.location
  tags     = var.tags
}
```

### Block 2: Spoke Virtual Network (AVM Module)

```hcl
module "spoke_vnet" {
  source  = "Azure/virtual-network/azurerm"
  version = "~> 1.0"

  name                = "vnet-lz-${var.landing_zone_name}-prod"
  resource_group_name = azurerm_resource_group.spoke.name
  location            = var.location
  address_space       = var.vnet_address_space

  subnets = {
    app_subnet = {
      name             = "snet-app-${var.landing_zone_name}"
      address_prefixes = [var.app_subnet_prefix]
    }
    db_subnet = {
      name             = "snet-db-${var.landing_zone_name}"
      address_prefixes = [var.db_subnet_prefix]
    }
    pe_subnet = {
      name             = "snet-pe-${var.landing_zone_name}"
      address_prefixes = [var.pe_subnet_prefix]
    }
  }

  tags = var.tags
}
```

**Why each parameter exists:**
- `source` -- tells Terraform to download the AVM module from the registry
- `version` -- pins the module version so updates do not break deployments
- `address_space` -- the CIDR range for the entire VNet (/24 from variables)
- `subnets` -- carves the VNet into named subnets with their CIDR slices
- `tags` -- applied to the VNet and all child resources

### Block 3: Route Table (UDR) for App Subnet

```hcl
resource "azurerm_route_table" "app_subnet_udr" {
  name                = "rt-snet-app-${var.landing_zone_name}"
  resource_group_name = azurerm_resource_group.spoke.name
  location            = var.location

  route {
    name                   = "default-to-firewall"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = var.hub_firewall_private_ip
  }

  tags = var.tags
}

# Associate the route table with the app subnet
resource "azurerm_subnet_route_table_association" "app_subnet" {
  subnet_id      = module.spoke_vnet.subnets["app_subnet"].resource_id
  route_table_id = azurerm_route_table.app_subnet_udr.id
}
```

**Why each part exists:**
- `address_prefix = "0.0.0.0/0"` -- matches ALL outbound traffic
- `next_hop_type = "VirtualAppliance"` -- routes to a specific IP (the firewall)
- `next_hop_in_ip_address` -- the Azure Firewall private IP from variables
- The association block -- attaches the route table to the specific subnet

### Block 4: NSG for App Subnet

```hcl
resource "azurerm_network_security_group" "app_subnet_nsg" {
  name                = "nsg-snet-app-${var.landing_zone_name}"
  resource_group_name = azurerm_resource_group.spoke.name
  location            = var.location

  # Allow inbound HTTPS from anywhere within the VNet
  security_rule {
    name                       = "allow-inbound-https-vnet"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "VirtualNetwork"
    destination_address_prefix = "*"
  }

  # Deny all other inbound traffic
  security_rule {
    name                       = "deny-all-inbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}

# Associate NSG with app subnet
resource "azurerm_subnet_network_security_group_association" "app_subnet" {
  subnet_id                 = module.spoke_vnet.subnets["app_subnet"].resource_id
  network_security_group_id = azurerm_network_security_group.app_subnet_nsg.id
}
```

**Why priority numbers matter:**
- Rules are evaluated lowest number first
- Priority 100 evaluates before priority 4096
- The allow rule fires for HTTPS -- the deny-all catches everything else
- Azure has default rules at 65000+ that you cannot delete

### Block 5: NSG for DB Subnet (More Restrictive)

```hcl
resource "azurerm_network_security_group" "db_subnet_nsg" {
  name                = "nsg-snet-db-${var.landing_zone_name}"
  resource_group_name = azurerm_resource_group.spoke.name
  location            = var.location

  # Only allow inbound from app subnet -- nothing else
  security_rule {
    name                       = "allow-inbound-from-app-subnet"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "1433"
    source_address_prefix      = var.app_subnet_prefix
    destination_address_prefix = "*"
  }

  # Deny everything else inbound
  security_rule {
    name                       = "deny-all-inbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}

resource "azurerm_subnet_network_security_group_association" "db_subnet" {
  subnet_id                 = module.spoke_vnet.subnets["db_subnet"].resource_id
  network_security_group_id = azurerm_network_security_group.db_subnet_nsg.id
}
```

**Why the DB NSG is more restrictive:**
- Port 1433 is SQL Server -- only the app subnet can reach it
- Source is `var.app_subnet_prefix` not `VirtualNetwork` -- explicit subnet-level control
- This is the principle of least privilege applied to network rules

### Block 6: RBAC Assignment for App Team (AVM Module)

```hcl
module "rbac_app_team_contributor" {
  source  = "Azure/authorization/azurerm//modules/role_assignment"
  version = "~> 0.1"

  principal_id         = var.app_team_group_object_id
  role_definition_name = "Contributor"
  scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"

  # principal_type prevents unnecessary Graph API lookups
  principal_type = "Group"
}

# Data source to get current subscription and tenant details
data "azurerm_client_config" "current" {}
```

---

## Section 6 -- outputs.tf (Write This Last)

Outputs expose values from this landing zone for other modules or pipelines to consume.

```hcl
output "resource_group_name" {
  description = "Name of the spoke resource group"
  value       = azurerm_resource_group.spoke.name
}

output "spoke_vnet_id" {
  description = "Resource ID of the spoke VNet -- used for peering configuration"
  value       = module.spoke_vnet.resource_id
}

output "spoke_vnet_name" {
  description = "Name of the spoke VNet"
  value       = module.spoke_vnet.name
}

output "app_subnet_id" {
  description = "Resource ID of the app tier subnet"
  value       = module.spoke_vnet.subnets["app_subnet"].resource_id
}

output "db_subnet_id" {
  description = "Resource ID of the DB tier subnet"
  value       = module.spoke_vnet.subnets["db_subnet"].resource_id
}
```

---

## Section 7 -- Running Terraform Plan

### Step 1: Initialise

```bash
terraform init
```

This downloads the AVM modules from the Terraform Registry  
and the AzureRM provider from HashiCorp.  
You will see it downloading modules -- this is expected.

### Step 2: Validate

```bash
terraform validate
```

Checks for syntax errors before running plan.  
Fix any errors shown before proceeding.

### Step 3: Plan

```bash
terraform plan
```

Read every line of the output carefully.  
You should see:

```
Terraform will perform the following actions:

  # azurerm_resource_group.spoke will be created
  + resource "azurerm_resource_group" "spoke" {
      + location = "eastus"
      + name     = "rg-lz-a1-prod"
      + tags     = {
          + "cost-centre"  = "CC-1001"
          + "environment"  = "production"
          + "landing-zone" = "a1"
          + "managed-by"   = "terraform"
          + "team"         = "app-team-a"
        }
    }

  # module.spoke_vnet.azurerm_virtual_network.this will be created
  + resource "azurerm_virtual_network" "this" {
      + address_space = ["10.1.0.0/24"]
      + location      = "eastus"
      + name          = "vnet-lz-a1-prod"
      ...
    }

Plan: 12 to add, 0 to change, 0 to destroy.
```

### Step 4: Read the Plan Output

For every resource in the plan ask yourself:
1. Why is this resource being created?
2. What ALZ concept does it implement?
3. What would happen if this resource was missing?

If you cannot answer all three for every resource -- go back and re-read the relevant section.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Error: Module not found` | terraform init not run | Run `terraform init` first |
| `Error: Invalid value for variable` | tfvars value wrong type | Check variable type in variables.tf |
| `Error: Address space overlaps` | CIDR conflict | Check your subnet CIDRs do not overlap |
| `Error: Subscription not found` | Not logged in or wrong subscription | Run `az login` and `az account set` |
| `Error: Insufficient permissions` | No Contributor on subscription | Check your RBAC in the portal |
| `Module has no outputs matching` | Wrong output reference | Check AVM module outputs tab on registry |

---

## Hands-On Checklist -- Complete in Order

- [ ] Terraform installed -- `terraform version` works
- [ ] Azure CLI installed -- `az --version` works
- [ ] Logged into Azure -- `az login` completed
- [ ] Correct subscription set -- `az account show` shows right subscription
- [ ] Working directory created -- `alz-spoke-day7/`
- [ ] providers.tf written and understood
- [ ] variables.tf written -- all 9 variables with descriptions
- [ ] terraform.tfvars written with real or placeholder values
- [ ] main.tf written -- all 6 blocks written and understood
- [ ] outputs.tf written -- all 5 outputs
- [ ] `terraform init` run successfully -- modules downloaded
- [ ] `terraform validate` passes -- no syntax errors
- [ ] `terraform plan` runs successfully -- 12 resources to add
- [ ] Every resource in the plan explained -- why it exists, what it implements

---

## Self-Test Questions and Correct Answers

---

### Question 1: Why does the UDR use 0.0.0.0/0 as the address prefix?

**Correct Answer:**  
0.0.0.0/0 is the default route -- it matches every possible destination address.  
Using this prefix means every packet leaving the subnet, regardless of destination,  
is forced through the Azure Firewall.  
Traffic to the internet, traffic to other subnets, traffic to on-premises --  
all of it hits the firewall first.  
If you used a more specific prefix like 10.0.0.0/8, only traffic to that range  
would go through the firewall. Internet-bound traffic would bypass it entirely.  
0.0.0.0/0 ensures nothing escapes inspection.

---

### Question 2: Why is the DB subnet NSG more restrictive than the app subnet NSG?

**Correct Answer:**  
The DB subnet contains databases -- the most sensitive data in the landing zone.  
The app subnet NSG allows inbound HTTPS from anywhere within the VNet.  
The DB subnet NSG allows inbound SQL traffic on port 1433  
from the app subnet IP range only -- no other source is permitted.  
This implements the principle of least privilege at the network level.  
Even if a VM in another subnet is compromised, it cannot reach the database  
because the NSG blocks it before the traffic arrives.  
The database is accessible only to the specific subnet that needs it.

---

### Question 3: Why do you use a Terraform module version pin like `version = "~> 1.0"` instead of just using the latest?

**Correct Answer:**  
The `~> 1.0` constraint means "use any version from 1.0 up to but not including 2.0."  
Minor versions (1.1, 1.2) are allowed -- they add features without breaking changes.  
Major versions (2.0) are blocked -- they may introduce breaking changes.  
Without a version pin, `terraform init` downloads the latest version every time.  
If a module releases a breaking change, your next pipeline run fails unexpectedly  
in production without you changing any code.  
Version pinning makes your deployment reproducible and prevents silent breakage  
from upstream module updates.

---

### Question 4: What does terraform init actually do?

**Correct Answer:**  
Terraform init performs three things:  
First -- it downloads the required providers declared in required_providers  
(in this case the AzureRM provider from HashiCorp).  
Second -- it downloads the AVM modules declared with source in module blocks  
from the Terraform Registry into a local .terraform directory.  
Third -- it initialises the backend -- if remote state is configured,  
it connects to the Azure Storage Account and verifies access.  
Nothing is deployed. Nothing in Azure is changed.  
Init only sets up the local Terraform working environment.  
It must be run before plan or apply can work.

---

### Question 5: A colleague looks at your code and asks why you have both an NSG and a UDR on the same subnet. Is that not redundant?

**Correct Answer:**  
No -- they operate at different layers and serve different purposes.  
The NSG filters traffic at the subnet boundary -- it decides what is allowed  
to enter or leave the subnet based on IP address and port.  
The UDR controls where traffic goes next after leaving the subnet --  
it forces all outbound traffic through the Azure Firewall for deep inspection.  
The NSG is the doorman at the floor of the apartment building.  
The UDR is the sign at reception saying all visitors must pass through security.  
Both are needed because the NSG cannot do URL filtering or threat intelligence,  
and the UDR without an NSG means traffic is inspected at the firewall  
but nothing enforces subnet-level isolation locally.  
Removing either one creates a security gap.

---

## Day 7 Completion Checklist

- [ ] All files written from scratch -- not copy-pasted from this file
- [ ] terraform validate passes
- [ ] terraform plan succeeds -- 12 resources to add
- [ ] Every resource in the plan explained out loud
- [ ] All 5 self-test questions answered without notes
- [ ] Score yourself -- 80 and above means proceed to Day 8

---

## Day 8 Preview -- Adding Hub Peering and Testing the Full Flow

On Day 8 you extend your Day 7 code to add:
- VNet peering from spoke to hub (with all three peering settings)
- Private DNS Zone and link to the spoke VNet
- A basic Defender for Cloud configuration
- Running terraform plan on the extended code
- Understanding the full network flow from your code to Azure

Day 8 is where your landing zone becomes genuinely connected  
to the hub and on-premises path.

---

*Day 7 of 21 | Azure Landing Zone Study Track -- Terraform Track | March 2026*