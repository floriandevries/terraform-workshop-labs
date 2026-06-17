# Module 03 - Lab
## Core Terraform workflow

**Duration:** ~30 minutes  
**Provider:** `hashicorp/azurerm`  
**You will:** Practice the complete workflow end-to-end — write and format a configuration, initialise the workspace, validate it, plan and apply changes to Azure, inspect state, detect drift, and clean everything up.

## Prerequisites

> **First time?** These labs require **Terraform** (≥ 1.5) and the **Azure CLI**. If the `terraform` or `az` commands below aren't recognised, install them first: [Terraform](https://developer.hashicorp.com/terraform/install) · [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

> **Windows users:** Run this lab in **Git Bash** (bundled with [Git for Windows](https://git-scm.com/download/win)) or **WSL**. Every command below is written for a bash-style shell and runs as-is in Git Bash. CMD and PowerShell differ on a few commands (`export`, `mkdir -p`, inspecting files), so they aren't recommended for these labs.

```bash
terraform -version    # should be ≥ 1.5
az account show       # confirm your training Azure subscription is active
```

If `az account show` returns an error, run `az login` first.

You will create an **Azure Resource Group** and an **Azure Storage Account**. Make sure you have permission to do this in the Azure subscription shown.

## Setup

```bash
mkdir ~/module03_lab && cd ~/module03_lab
```

All files you create in this lab go in this folder.

## Step 1 — Write and format

### What you'll learn
- How to write a minimal but complete Terraform configuration for Azure.
- What `terraform fmt` does and why it matters.

### Step 1a - Create `main.tf` intentionally messy

Copy the following into a new file called `main.tf`. The inconsistent indentation and misaligned `=` signs are intentional.

```hcl
terraform {
  required_providers {
      azurerm = {
        source = "hashicorp/azurerm"
          version   = "~> 4.0"
    }
  }
}

provider "azurerm" {
features {}
}

resource "azurerm_resource_group" "lab" {
name = "rg-<your-initials>-we-d-mgt-lab03"
    location =  "westeurope"
}

resource "azurerm_storage_account" "lab" {
  name                     = "st<your-initials>wedmgtlab03"
  resource_group_name           = azurerm_resource_group.lab.name
  location = azurerm_resource_group.lab.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

> **Replace `<your-initials>`** in both resource names, use **lowercase letters only**. Storage account names must be 3–24 characters, lowercase, alphanumeric, and globally unique across all of Azure.
>
> The names follow Microsoft's naming convention: `<type>-<initials>-<region>-<env>-<workload>-<instance>`. Here `we` = westeurope, `d` = development, `mgt` = management workload, `lab03` = this lab.
>
> Example with initials `fdv`: resource group `rg-fdv-we-d-mgt-lab03`, storage account `stfdvwedmgtlab03`.

### Step 1b - Format with `terraform fmt`

```bash
terraform fmt
```

Open `main.tf` again. The file has been rewritten in place with consistent two-space indentation and aligned `=` signs.

> **What's happening here?**  
> `terraform fmt` applies the canonical HCL style. It only changes whitespace, never logic. The configuration is identical, just readable and consistent.

> **Notice:** `terraform fmt` prints the filename of every file it changes. If nothing is printed, the file was already correctly formatted.

### ✅ Checkpoint

- [ ] `main.tf` exists with a resource group and a storage account, both using your initials
- [ ] `terraform fmt` reformatted the file — aligned `=` signs and consistent two-space indentation

## Step 2 — Init, validate and plan

### What you'll learn
- What `terraform init` does and why it must run before `terraform validate`.
- How `terraform validate` catches errors before you reach the Azure API.
- How to read a `terraform plan` output and save a plan for safe application.

### Step 2a - Initialise the workspace

```bash
terraform init
```

> **What's happening here?**  
> `terraform init` does three things: downloads the `azurerm` provider plugin into `.terraform/`, configures the backend (local by default), and creates `.terraform.lock.hcl` to pin the exact provider version.
>
> **Why init before validate?** `terraform validate` checks argument names against the provider schema, the schema lives in the provider binary that `terraform init` downloads. Without `init`, validate can only catch basic syntax errors, not argument-level mistakes like typos in argument names.

Check the lock file:

```bash
cat .terraform.lock.hcl
```

Find the `version =` line. Anyone who runs `terraform init` in this directory will get exactly this provider version, as long as the lock file is committed to Git.

### Step 2b - Introduce a deliberate typo

Edit `main.tf`: find the `resource_group_name` argument on the storage account block and change it to `resourc_group_name` (remove the `e` in `resource`).

### Step 2c - Validate and catch the typo

```bash
terraform validate
```

You should see:

```
│ Error: Unsupported argument
│
│ An argument named "resourc_group_name" is not expected here.
│ Did you mean "resource_group_name"?
```

> **What's happening here?**  
> `terraform validate` checks your `.tf` files offline, no API calls, no Azure credentials needed. It caught the wrong argument name before any plan or apply. This is the habit: validate before you touch the cloud.

### Step 2d - Fix the typo and validate again

Correct `resourc_group_name` back to `resource_group_name`, then run:

```bash
terraform validate
```

```
Success! The configuration is valid.
```

> **Habit:** Run `terraform fmt` before committing. Run `terraform validate` before every `terraform plan`.

### Step 2e - Preview with `terraform plan`

```bash
terraform plan
```

Read the output carefully:

- Lines starting with `+` will be **created**.
- The summary line: `Plan: 2 to add, 0 to change, 0 to destroy`.
- The resource group appears before the storage account, Terraform detected the implicit dependency and will create them in the right order.

### Step 2f - Save the plan to a file

```bash
terraform plan -out=tfplan
```

> **What's happening here?**  
> This saves the exact plan to a binary file called `tfplan`. When you apply a saved plan, Terraform applies *exactly* what was reviewed, there is no risk of drift between the review and the execution. This is the production-safe pattern.

### ✅ Checkpoint

- [ ] `terraform init` downloaded the `azurerm` provider and created `.terraform.lock.hcl`
- [ ] `terraform validate` caught the `resourc_group_name` typo, then passed after the fix
- [ ] `terraform plan` showed `Plan: 2 to add, 0 to change, 0 to destroy`
- [ ] The plan was saved to a `tfplan` file

## Step 3 — Apply and inspect state

### What you'll learn
- How `terraform apply` uses the dependency graph to create resources in the right order.
- How to inspect what Terraform knows about your resources with `terraform state`.
- How the state file links Terraform's config to real Azure resource IDs.

### Step 3a - Apply the saved plan

```bash
terraform apply tfplan
```

> **Notice:** Applying a saved plan skips the `yes` confirmation, the plan was already reviewed. Watch the output as resources are created.

While it runs, open the **Azure Portal** and navigate to Resource Groups. Watch `rg-<your-initials>-we-d-mgt-lab03` appear, then the storage account appear inside it.

### Step 3b - List tracked resources

```bash
terraform state list
```

Expected output:

```
azurerm_resource_group.lab
azurerm_storage_account.lab
```

### Step 3c - Inspect a specific resource

```bash
terraform state show azurerm_storage_account.lab
```

Find and note:
- `account_tier` — should be `"Standard"`
- `account_replication_type` — should be `"LRS"`
- `id` — the full Azure ARM resource ID that links Terraform's state to Azure

### Step 3d - Open the raw state file

Open `terraform.tfstate` in your editor. Find the `"resources"` array and locate your storage account. The `"id"` field is the ARM resource ID, this is the link between Terraform's world and Azure's world.

> **Important:** Never manually edit `terraform.tfstate`. If you need to modify state, use `terraform state` commands.

### ✅ Checkpoint

- [ ] `terraform apply tfplan` created both resources — visible in the Azure Portal
- [ ] `terraform state list` shows `azurerm_resource_group.lab` and `azurerm_storage_account.lab`
- [ ] `terraform state show` displayed the storage account's `id` — the full ARM resource ID

## Step 4 — Drift and destroy

### What you'll learn
- What infrastructure drift is and how Terraform detects it.
- How to reconcile drift with `terraform apply`.
- How `terraform destroy` cleans up infrastructure safely.

### Step 4a - Simulate drift

1. Open the **Azure Portal** → navigate to your storage account.
2. Click **Tags** in the left menu.
3. Add a new tag: key `owner`, value `<your-initials>` (e.g. `fdv`).
4. Click **Apply**.

This simulates what happens when someone changes infrastructure outside of Terraform.

### Step 4b - Detect drift with `terraform plan`

```bash
terraform plan
```

Find this in the output:

```
  ~ update in-place
  ~ tags = {
      - "owner" = "fdv" -> null
    }
```

> **What's happening here?**  
> Terraform compared your config (no `owner` tag) against the real Azure resource (which now has one) and found a difference. The `~` symbol means an in-place update — Terraform is proposing to revert the manual change to match what the config declares.

### Step 4c - Reconcile the drift

```bash
terraform apply
```

Type `yes`. Verify in the Azure Portal that the `owner` tag has been removed.

### Step 4d - Preview the destroy plan

```bash
terraform plan -destroy
```

Review the output — `Plan: 0 to add, 0 to change, 2 to destroy`. Notice that the storage account appears before the resource group in the destroy order. Terraform reverses the dependency graph automatically: it cannot remove the resource group while the storage account still lives inside it.

> **Notice:** Terraform works out the reverse dependency order from the same graph it used to create. You never have to specify what to destroy first.

### Step 4e - Destroy the storage account

**Lab 04 will reference your resource group as a data source**, so keep it alive. Destroy only the storage account:

```bash
terraform destroy -target=azurerm_storage_account.lab
```

Type `yes`. Verify in the Portal that the storage account is gone but `rg-<your-initials>-we-d-mgt-lab03` remains.

> **Notice:** Terraform prints a yellow warning that *"Resource targeting is in effect"* and is *"not for routine use"*. That is expected here — we are deliberately keeping the resource group for Lab 04. In normal day-to-day work you destroy everything, so you rarely need `-target`.

> You will destroy the resource group at the end of Lab 04.

### ✅ Checkpoint

- [ ] The manually added `owner` tag showed up as an in-place update in `terraform plan`
- [ ] `terraform apply` reconciled the drift — the tag is gone from the Portal
- [ ] `terraform plan -destroy` previewed the storage account being destroyed before the resource group
- [ ] `terraform destroy -target=azurerm_storage_account.lab` removed the storage account; the resource group remains for Lab 04

## Summary

| Command | What it did in this lab |
|---|---|
| `terraform fmt` | Fixed inconsistent formatting in `main.tf` |
| `terraform init` | Downloaded the `azurerm` provider and created the lock file |
| `terraform validate` | Caught a typo in an argument name before touching Azure |
| `terraform plan` | Previewed 2 resources to create |
| `terraform plan -out=tfplan` | Saved the plan for a safe, reproducible apply |
| `terraform apply tfplan` | Created the resource group and storage account |
| `terraform state list` | Listed all resources tracked in state |
| `terraform state show` | Showed the full attributes of the storage account |
| `terraform plan -destroy` | Previewed the destroy order (storage account before resource group) |
| `terraform destroy -target=azurerm_storage_account.lab` | Removed only the storage account, keeping the resource group for Lab 04 |

**Habits to take away:**
- Run `terraform fmt` before committing. Run `terraform validate` before every `terraform plan`.
- Always run `terraform init` first, `terraform validate` needs the provider schemas to catch argument-level errors.
- Always read the plan before typing `yes`.
- Drift happens. `terraform plan` always tells you about it.
- Terraform reverses the dependency graph for destroys automatically — you never have to specify the order.
- Never manually edit `terraform.tfstate`.

---

## Full `main.tf` reference

If you got stuck, here is the complete, correctly formatted `main.tf`:

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
  name     = "rg-<your-initials>-we-d-mgt-lab03"
  location = "westeurope"
}

resource "azurerm_storage_account" "lab" {
  name                     = "st<your-initials>wedmgtlab03"
  resource_group_name      = azurerm_resource_group.lab.name
  location                 = azurerm_resource_group.lab.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

---

## Further reading

- [The Core Terraform Workflow](https://developer.hashicorp.com/terraform/intro/v1.12.x/core-workflow)
- [terraform fmt](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/fmt)
- [terraform validate](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/validate)
- [terraform init](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/init)
- [terraform plan](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/plan)
- [terraform apply](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/apply)
- [terraform destroy](https://developer.hashicorp.com/terraform/cli/v1.12.x/commands/destroy)
- [Manage resources in Terraform state](https://developer.hashicorp.com/terraform/tutorials/state/state-cli)
