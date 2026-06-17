# Module 05 - Lab
## Build and use a local module

**Duration:** ~25 minutes  
**Provider:** `hashicorp/azurerm`  
**You will:** Write a reusable Azure storage module from scratch, call it from a root configuration, deliberately hit the scope boundary, and call it twice to create two independent storage accounts from one module definition.

---

## Prerequisites

> **First time?** These labs require **Terraform** (≥ 1.5) and the **Azure CLI**. If the `terraform` or `az` commands below aren't recognised, install them first: [Terraform](https://developer.hashicorp.com/terraform/install) · [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

> **Windows users:** Run this lab in **Git Bash** (bundled with [Git for Windows](https://git-scm.com/download/win)) or **WSL**. Every command below is written for a bash-style shell and runs as-is in Git Bash. CMD and PowerShell differ on a few commands (`export`, `mkdir -p`, inspecting files), so they aren't recommended for these labs.

```bash
terraform -version    # should be ≥ 1.5
az account show       # confirm your training Azure subscription is active
```

You will create one Azure Resource Group and **two** Storage Accounts.

---

## Setup

```bash
mkdir ~/module05_lab && cd ~/module05_lab
```

This lab uses a subdirectory structure. Terraform treats each directory independently — the root configuration calls the child module by its path.

---

## Step 1 — Write the module

### What you'll learn
- How to structure a local module using the three standard files.
- How module variables are the module's *input interface* — callers must pass values for required variables.
- How outputs define what the module exposes to callers — everything else is private.

### Step 1a - Create the module directory

```bash
mkdir -p modules/storage_account
```

Your project structure will look like this:

```
module05_lab/
├── modules/
│   └── storage_account/
│       ├── main.tf        ← the module's resources
│       ├── variables.tf   ← what callers must/can pass in
│       └── outputs.tf     ← what callers can read back
├── main.tf                ← root config (you'll create this in Step 2)
├── variables.tf           ← root variables (Step 2)
├── terraform.tfvars       ← root variable values (Step 2)
└── outputs.tf             ← root outputs (Step 3)
```

> **A module is nothing more than a directory of `.tf` files.** Let's move a storage account pattern into one.

### Step 1b - Create `modules/storage_account/variables.tf`

```hcl
variable "resource_group_name" {
  type        = string
  description = "Name of the resource group to deploy into."
}

variable "location" {
  type        = string
  description = "Azure region."
}

variable "storage_account_name_suffix" {
  type        = string
  description = "Short suffix appended to the storage account name (e.g. 'logs', 'data')."
}

variable "initials" {
  type        = string
  description = "Your initials, used to make the storage account name unique (e.g. 'fdv')."

  validation {
    condition     = can(regex("^[a-z]{2,4}$", var.initials))
    error_message = "initials must be 2–4 lowercase letters."
  }
}

variable "environment" {
  type        = string
  description = "Environment name (development, test, acceptance, production)."
  default     = "development"

  validation {
    condition     = contains(["development", "test", "acceptance", "production"], var.environment)
    error_message = "environment must be one of: development, test, acceptance or production."
  }
}

variable "tags" {
  type        = map(string)
  description = "Additional tags applied to the storage account. The environment tag is always derived from var.environment."
  default     = {}
}
```

> **What's happening here?**  
> These variables are the module's **input interface**. The caller (the root config) must provide a value for every variable that has no `default`. Notice that the module doesn't know what resource group exists in the root — the caller has to tell it via `var.resource_group_name`.

### Step 1c - Create `modules/storage_account/main.tf`

```hcl
locals {
  storage_account_tiers = {
    development = "Standard"
    test        = "Standard"
    acceptance  = "Standard"
    production  = "Premium"
  }

  tags = merge(var.tags, { environment = var.environment })
}

resource "azurerm_storage_account" "st" {
  name                     = "st${var.initials}wedidlab05${var.storage_account_name_suffix}"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = lookup(local.storage_account_tiers, var.environment, "Standard")
  account_replication_type = "LRS"
  tags                     = local.tags
}
```

> **What's happening here?**  
> Inside the module, all configuration uses `var.*` — the module is completely self-contained. It has no visibility into the root config's resources or variables. The `lookup()` map and `merge()` tag pattern from Lab 04 live here once, inside the module, rather than repeated in every caller.
>
> **Naming:** `stfdvwedidlab05logs`, `stfdvwedidlab05data` (for initials `fdv`). The name follows the Microsoft naming convention: `st` + initials + `we` (westeurope) + `d` (development) + `id` (workload) + `lab05` (instance) + suffix. All under 24 characters.

### Step 1d - Create `modules/storage_account/outputs.tf`

```hcl
output "endpoint" {
  description = "Primary blob endpoint for this storage account."
  value       = azurerm_storage_account.st.primary_blob_endpoint
}

output "name" {
  description = "Name of the created storage account."
  value       = azurerm_storage_account.st.name
}
```

> **What's happening here?**  
> These two outputs are the **only** things a caller can access from this module. The `account_tier`, `location`, and everything inside `local.*` are all private implementation details. Outputs are the module's public interface.

### ✅ Checkpoint

- [ ] `modules/storage_account/variables.tf` has 6 variables
- [ ] `modules/storage_account/main.tf` creates `azurerm_storage_account.st` using `var.initials` in the name
- [ ] `modules/storage_account/outputs.tf` exposes `endpoint` and `name`

---

## Step 2 — Call the module from root

### What you'll learn
- How the `module` block calls a child module and maps arguments to the module's variables.
- How `terraform init` handles local modules and what it records.
- How the resource address in state is fully namespaced by the module name.

### Step 2a - Create root `variables.tf`

```hcl
variable "initials" {
  type        = string
  description = "Your initials, used in resource names to keep them unique (e.g. 'fdv')."

  validation {
    condition     = can(regex("^[a-z]{2,4}$", var.initials))
    error_message = "initials must be 2–4 lowercase letters."
  }
}
```

### Step 2b - Create `terraform.tfvars`

```hcl
initials = "fdv" # REPLACE: your own lowercase initials (2–4 letters)
```

> `terraform.tfvars` is auto-loaded by Terraform — no `-var-file` flag needed. This is the one exception to the explicit-loading rule.

### Step 2c - Create root `main.tf`

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "lab" {
  name     = "rg-${var.initials}-we-d-id-lab05"
  location = "westeurope"
  tags = {
    managed_by = "terraform"
  }
}

module "storage_account_logs" {
  source = "./modules/storage_account"

  resource_group_name         = azurerm_resource_group.lab.name
  location                    = azurerm_resource_group.lab.location
  storage_account_name_suffix = "logs"
  initials                    = var.initials
  environment                 = "development"
  tags = {
    managed_by = "terraform"
  }
}
```

> **What's happening here?**  
> `source = "./modules/storage_account"` tells Terraform where to find the module. Every other argument (`resource_group_name`, `location`, etc.) **must match a `variable` block** inside `modules/storage_account/variables.tf`. If you pass an argument with no matching variable, Terraform errors: "An argument named X is not expected here."

### Step 2d - Initialise and preview

```bash
terraform init
```

> **Notice:** Terraform created `.terraform/modules/`. Open `modules.json` inside it — it records the local path to the module. For Registry modules this file would show the download URL and version.

```bash
terraform validate
terraform plan
```

> **Habit:** Same as Lab 03 and 04 — `terraform validate` before every `terraform plan`. With a module in play it now also checks that the arguments you pass match the module's declared variables.

Look carefully at the resource address:

```
# module.storage_account_logs.azurerm_storage_account.st will be created
```

> **What's happening here?**  
> The resource address is fully namespaced: `module.<local_name>.<resource_type>.<resource_name>`. This is how Terraform keeps multiple module instances from conflicting in state.

### ✅ Checkpoint

- [ ] Root `variables.tf` declares `var.initials`
- [ ] `terraform.tfvars` exists with your own initials set
- [ ] Root `main.tf` has `module "storage_account_logs"` with all required arguments
- [ ] `terraform init` completed — `.terraform/modules/modules.json` exists
- [ ] `terraform plan` shows `module.storage_account_logs.azurerm_storage_account.st` will be created

---

## Step 3 — Explore scope

### What you'll learn
- What the scope boundary means in practice — by hitting it deliberately.
- How to correctly reference module outputs from the root.
- The distinction between internal implementation and the public interface.

### Step 3a - Deliberately break scope to see the error

Create root `outputs.tf` with a reference to something that is **not** a declared module output:

```hcl
# This will fail — account_tier is NOT declared as an output in the module
output "storage_tier" {
  value = module.storage_account_logs.account_tier
}
```

```bash
terraform plan
```

You should see:

```
│ Error: Unsupported attribute
│
│ This object does not have an attribute named "account_tier".
```

> **What's happening here?**  
> `account_tier` is an attribute of `azurerm_storage_account.st` inside the module — but the module doesn't expose it. The root config cannot reach inside a module to read its resources' attributes directly. This is encapsulation: the caller only sees what the module author chose to expose.

### Step 3b - Fix the output to use a declared module output

Replace the content of `outputs.tf`:

```hcl
output "logs_endpoint" {
  description = "Blob endpoint for the logs storage account."
  value       = module.storage_account_logs.endpoint
}

output "logs_storage_name" {
  description = "Name of the logs storage account."
  value       = module.storage_account_logs.name
}
```

```bash
terraform plan
```

> **Notice:** The outputs now show `(known after apply)` — Terraform accepted the references as valid and will populate the values once the resources are created.

> **Discussion:** What other attributes would you expose if you were publishing this module for your team? Think about what a caller would need: the resource ID for role assignments, the primary access key for apps, the connection string?

### ✅ Checkpoint

- [ ] The intentional scope error (`account_tier`) triggered a clear error message
- [ ] `outputs.tf` now references `module.storage_account_logs.endpoint` and `.name`
- [ ] `terraform plan` succeeds with `(known after apply)` on both outputs

---

## Step 4 — Two instances and destroy

### What you'll learn
- How to create multiple independent resource sets from a single module definition.
- How module outputs are accessed per-instance.
- How to clean up a modular configuration.

### Step 4a - Add a second module call

Add to the end of root `main.tf`:

```hcl
module "storage_account_data" {
  source = "./modules/storage_account"

  resource_group_name         = azurerm_resource_group.lab.name
  location                    = azurerm_resource_group.lab.location
  storage_account_name_suffix = "data"
  initials                    = var.initials
  environment                 = "development"
  tags = {
    managed_by = "terraform"
  }
}
```

### Step 4b - Update `outputs.tf`

Replace the content of `outputs.tf`:

```hcl
output "logs_endpoint" {
  value = module.storage_account_logs.endpoint
}

output "logs_storage_name" {
  value = module.storage_account_logs.name
}

output "data_endpoint" {
  value = module.storage_account_data.endpoint
}

output "data_storage_name" {
  value = module.storage_account_data.name
}
```

```bash
terraform init
terraform plan
```

> **Why `init` again?** You added a *new* module call (`storage_account_data`). Even though it points at the same source path, Terraform has to register the new instance in `.terraform/modules/` before it can plan it. Skipping `init` here gives you a "Module not installed" error.

> **Notice:** Three resources to add — the resource group and two storage accounts, each in their own module namespace. Same module definition, two completely separate resource instances.

### Step 4c - Apply

```bash
terraform apply
```

Type `yes`. Open the **Azure Portal** and navigate to `rg-<your-initials>-we-d-id-lab05`. Verify both storage accounts exist with names containing your initials.

### Step 4d - Query outputs and inspect state

```bash
terraform output
```

```bash
terraform state list
```

The state list shows:

```
azurerm_resource_group.lab
module.storage_account_data.azurerm_storage_account.st
module.storage_account_logs.azurerm_storage_account.st
```

> **Notice:** `terraform state list` sorts addresses alphabetically, so `data` appears before `logs`. Both storage accounts are in different module namespaces — same module definition, two fully independent resource instances in state. The resource group is a root-level resource, not inside any module namespace.

### Step 4e - Destroy

```bash
terraform destroy
```

Type `yes`. Verify in the Azure Portal that `rg-<your-initials>-we-d-id-lab05` is gone.

### ✅ Checkpoint

- [ ] `terraform apply` created 3 resources (1 RG + 2 storage accounts) visible in Azure Portal
- [ ] `terraform output` shows four distinct values (two endpoints, two names)
- [ ] `terraform state list` shows module-namespaced resource addresses
- [ ] `terraform destroy` removed all resources

---

## Summary

| What you did | What you learned |
|---|---|
| Created `modules/storage_account/` with 3 files | A module is a directory of `.tf` files — the structure is the convention |
| Declared variables in the module | Variables are the input interface — only declared variables can be passed |
| Used `lookup()` and `merge()` inside the module | Lab 04 patterns live here once, shared by every caller |
| Declared outputs in the module | Outputs are the only thing callers can read — everything else is private |
| Created root `variables.tf` with `var.initials` | Root variables feed into module calls to keep resource names unique |
| Created `terraform.tfvars` | Auto-loaded values without explicit `-var-file` |
| Called the module from root with `source = "./modules/storage_account"` | The `module` block maps caller arguments to module variables |
| Ran `terraform init` and inspected `.terraform/modules/` | Terraform registers local modules by path, downloads remote ones |
| Deliberately referenced a non-output attribute | The scope boundary is real and enforced — it's not just a convention |
| Called the same module twice with different suffixes | One module definition → multiple independent resource sets in state |
| Read `module.storage_account_logs.endpoint` from root outputs | `module.<local_name>.<output_name>` is the access pattern |

**Key habits from this lab:**
- Always declare outputs for anything a caller might reasonably need
- Module variable names become the argument names in `module {}` — keep them clear and descriptive
- The scope boundary exists to protect callers from implementation details — embrace it
- Pass `var.initials` from root into the module so trainees only change one place

---

## Full file reference

### `modules/storage_account/variables.tf`

```hcl
variable "resource_group_name" {
  type        = string
  description = "Name of the resource group to deploy into."
}

variable "location" {
  type        = string
  description = "Azure region."
}

variable "storage_account_name_suffix" {
  type        = string
  description = "Short suffix appended to the storage account name (e.g. 'logs', 'data')."
}

variable "initials" {
  type        = string
  description = "Your initials, used to make the storage account name unique (e.g. 'fdv')."

  validation {
    condition     = can(regex("^[a-z]{2,4}$", var.initials))
    error_message = "initials must be 2–4 lowercase letters."
  }
}

variable "environment" {
  type        = string
  description = "Environment name (development, test, acceptance, production)."
  default     = "development"

  validation {
    condition     = contains(["development", "test", "acceptance", "production"], var.environment)
    error_message = "environment must be one of: development, test, acceptance or production."
  }
}

variable "tags" {
  type        = map(string)
  description = "Additional tags applied to the storage account. The environment tag is always derived from var.environment."
  default     = {}
}
```

### `modules/storage_account/main.tf`

```hcl
locals {
  storage_account_tiers = {
    development = "Standard"
    test        = "Standard"
    acceptance  = "Standard"
    production  = "Premium"
  }

  tags = merge(var.tags, { environment = var.environment })
}

resource "azurerm_storage_account" "st" {
  name                     = "st${var.initials}wedidlab05${var.storage_account_name_suffix}"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = lookup(local.storage_account_tiers, var.environment, "Standard")
  account_replication_type = "LRS"
  tags                     = local.tags
}
```

### `modules/storage_account/outputs.tf`

```hcl
output "endpoint" {
  description = "Primary blob endpoint for this storage account."
  value       = azurerm_storage_account.st.primary_blob_endpoint
}

output "name" {
  description = "Name of the created storage account."
  value       = azurerm_storage_account.st.name
}
```

### Root `variables.tf`

```hcl
variable "initials" {
  type        = string
  description = "Your initials, used in resource names to keep them unique (e.g. 'fdv')."

  validation {
    condition     = can(regex("^[a-z]{2,4}$", var.initials))
    error_message = "initials must be 2–4 lowercase letters."
  }
}
```

### Root `terraform.tfvars`

```hcl
initials = "fdv" # REPLACE: your own lowercase initials (2–4 letters)
```

### Root `main.tf`

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "lab" {
  name     = "rg-${var.initials}-we-d-id-lab05"
  location = "westeurope"
  tags = {
    managed_by = "terraform"
  }
}

module "storage_account_logs" {
  source = "./modules/storage_account"

  resource_group_name         = azurerm_resource_group.lab.name
  location                    = azurerm_resource_group.lab.location
  storage_account_name_suffix = "logs"
  initials                    = var.initials
  environment                 = "development"
  tags = {
    managed_by = "terraform"
  }
}

module "storage_account_data" {
  source = "./modules/storage_account"

  resource_group_name         = azurerm_resource_group.lab.name
  location                    = azurerm_resource_group.lab.location
  storage_account_name_suffix = "data"
  initials                    = var.initials
  environment                 = "development"
  tags = {
    managed_by = "terraform"
  }
}
```

### Root `outputs.tf`

```hcl
output "logs_endpoint" {
  value = module.storage_account_logs.endpoint
}

output "logs_storage_name" {
  value = module.storage_account_logs.name
}

output "data_endpoint" {
  value = module.storage_account_data.endpoint
}

output "data_storage_name" {
  value = module.storage_account_data.name
}
```

---

## Further reading

- [Modules overview](https://developer.hashicorp.com/terraform/tutorials/modules/module)
- [Use registry modules in configuration](https://developer.hashicorp.com/terraform/tutorials/modules/module-use)
- [Build and use a local module](https://developer.hashicorp.com/terraform/tutorials/modules/module-create)
- [module block reference](https://developer.hashicorp.com/terraform/language/v1.12.x/block/module)
- [Module composition](https://developer.hashicorp.com/terraform/language/v1.12.x/modules/develop/composition)
