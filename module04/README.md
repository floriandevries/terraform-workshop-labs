# Module 04 - Lab
## Terraform configuration language

**Duration:** ~30 minutes  
**Provider:** `hashicorp/azurerm`  
**You will:** Build a parameterised, multi-resource configuration using variables, `.tfvars` files, `locals` with `lookup()`, a data source, `for_each`, custom conditions (`precondition`, `postcondition`, `check`), and outputs — with a secret handled safely throughout.

---

## Prerequisites

> **First time?** These labs require **Terraform** (≥ 1.5) and the **Azure CLI**. If the `terraform` or `az` commands below aren't recognised, install them first: [Terraform](https://developer.hashicorp.com/terraform/install) · [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

> **Windows users:** Run this lab in **Git Bash** (bundled with [Git for Windows](https://git-scm.com/download/win)) or **WSL**. Every command below is written for a bash-style shell and runs as-is in Git Bash. CMD and PowerShell differ on a few commands (`export`, `mkdir -p`, inspecting files), so they aren't recommended for these labs.

```bash
terraform -version    # should be ≥ 1.5
az account show       # confirm your training Azure subscription is active
```

Set the database password before you start — this variable has no default, so Terraform needs it before it can evaluate the config:

```bash
export TF_VAR_database_password="labpassword123"
```

The resource group you created in Lab 03 (`rg-<your-initials>-we-d-mgt-lab03`) is still running — you will look it up here as a **data source**. Terraform reads it but does not manage its lifecycle in this configuration.

---

## Setup

```bash
mkdir ~/module04_lab && cd ~/module04_lab
```

This lab uses multiple `.tf` files. Terraform loads all `.tf` files in the working directory as one unified configuration.

---

## Step 1 — Variables, `.tfvars` and locals with `lookup()`

### What you'll learn
- How to declare typed variables with defaults, descriptions, and validation rules.
- How to use a `.tfvars` file to manage environment-specific values.
- How `locals` computes derived values, including `lookup()` for multi-value logic.

### Step 1a - Create `variables.tf`

```hcl
variable "initials" {
  type        = string
  description = "Your initials, used to make resource names unique (e.g. 'fdv'). Lowercase letters only."

  validation {
    condition     = can(regex("^[a-z]{2,4}$", var.initials))
    error_message = "initials must be 2–4 lowercase letters."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment. Must be development, test, acceptance or production."
  default     = "development"

  validation {
    condition     = contains(["development", "test", "acceptance", "production"], var.environment)
    error_message = "environment must be one of: development, test, acceptance or production."
  }
}

variable "storage_accounts" {
  type        = list(string)
  description = "Short names for each storage account to create."
  default     = ["logs", "data"]
}

variable "tags" {
  type        = map(string)
  description = "Additional tags applied to all resources. The environment tag is always derived from var.environment."
  default = {
    managed_by = "terraform"
  }
}

variable "database_password" {
  type        = string
  description = "Simulated database password. Pass via TF_VAR_database_password — never hardcode."
  sensitive   = true
}
```

> **What's happening here?**  
> Each variable has a `type` — Terraform rejects values of the wrong type at plan time. The `validation` block on `var.environment` enforces allowed values with a clear error message before any API call. `database_password` has no `default` — it **must** be supplied at runtime. The `sensitive = true` flag masks its value in terminal output (we'll show why that's not enough security in Step 4).

### Step 1b - Create `development.tfvars`

```hcl
environment = "development"
initials    = "fdv" # REPLACE: your own lowercase initials (2–4 letters)
tags = {
  managed_by = "terraform"
}
storage_accounts = ["logs", "data"]
```

> **What's happening here?**  
> A `.tfvars` file holds variable values for a specific environment without needing CLI flags on every command. You'd have a separate `production.tfvars` with different values. Terraform does **not** auto-load arbitrary `.tfvars` files — you pass them explicitly with `-var-file=development.tfvars`. The exceptions are `terraform.tfvars` and `*.auto.tfvars`, which load automatically.
>
> **Important:** `database_password` is intentionally absent here. Secrets must not live in `.tfvars` files committed to Git — they come from environment variables (`TF_VAR_database_password`) or a secrets manager.

### Step 1c - Create `locals.tf`

```hcl
locals {
  storage_account_tiers = {
    development = "Standard"
    test        = "Standard"
    acceptance  = "Standard"
    production  = "Premium"
  }

  storage_tier = lookup(local.storage_account_tiers, var.environment, "Standard")

  tags = merge(var.tags, { environment = var.environment })
}
```

> **What's happening here?**  
> Instead of `var.environment == "production" ? "Premium" : "Standard"` — which breaks when you add a third value — `lookup()` queries a map. The third argument is the default if the key isn't found. `local.tags` uses `merge()` to derive the `environment` tag from `var.environment`, so switching to production automatically updates the tag without any manual editing of the tfvars file.

Test it in `terraform console`:

```bash
terraform init
echo 'lookup({"development":"Standard","test":"Standard","acceptance":"Standard","production":"Premium"}, "production", "Standard")' | terraform console
```

> **What's happening here?**  
> `terraform console` evaluates HCL expressions interactively without touching your config or cloud. Use it constantly when testing functions. Try `toset(["a","b","a"])` to see deduplication, or `length("westeurope")` for string functions.

### ✅ Checkpoint

- [ ] `variables.tf` has all 5 variables — `initials`, `environment`, `storage_accounts`, `tags`, `database_password`
- [ ] `development.tfvars` exists with your own initials set and no secrets
- [ ] `locals.tf` uses `lookup()` and `merge()`
- [ ] `terraform console` returned `"Premium"` for the lookup expression above

---

## Step 2 — Data source, `depends_on` and `for_each`

### What you'll learn
- How `data` blocks read existing infrastructure without managing its lifecycle.
- When and how to use `depends_on` for explicit ordering.
- How `for_each = toset()` creates multiple named resource instances.

### Step 2a - Create `data.tf`

```hcl
data "azurerm_resource_group" "existing" {
  name = "rg-${var.initials}-we-d-mgt-lab03"
}
```

> **What's happening here?**  
> This tells Terraform to look up the resource group you created in Lab 03. It will **never** create, update, or destroy it — Terraform only reads it. Its attributes (name, location, tags) become available as `data.azurerm_resource_group.existing.location`, etc. Using `var.initials` in the name means each trainee automatically looks up their own resource group.

### Step 2b - Create `main.tf`

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

resource "azurerm_storage_account" "lab" {
  for_each = toset(var.storage_accounts)

  name                     = "st${var.initials}wedmgtlab04${each.key}"
  resource_group_name      = data.azurerm_resource_group.existing.name
  location                 = data.azurerm_resource_group.existing.location
  account_tier             = local.storage_tier
  account_replication_type = "LRS"
  tags                     = local.tags

  depends_on = [data.azurerm_resource_group.existing]
}
```

> **What's happening here?**  
> Five things to point out.
>
> **Data source** — `data.azurerm_resource_group.existing` reads the resource group you created in Lab 03. Terraform fetches its attributes but does not manage its lifecycle. Your storage accounts will be deployed *into* that resource group.
>
> **`for_each`** — instead of two identical storage account blocks, `toset(var.storage_accounts)` creates one named instance per element. Each has its own address: `azurerm_storage_account.lab["logs"]` and `["data"]`.
>
> **`var.initials`** — embedding your initials in the name (`stfdvwedmgtlab04logs`) keeps the storage account names consistent with the Lab 03 naming convention. The name follows the Microsoft naming convention: `st` + initials + `we` (westeurope) + `d` (development) + `mgt` (workload) + `lab04` (instance) + suffix.
>
> **`local.storage_tier`** — the `lookup()` result drives the tier across all instances from a single map.
>
> **`depends_on`** — the `location` reference already creates an implicit dependency, so this is redundant here. It is shown deliberately so you see the syntax and understand when it *is* needed: role assignments that must propagate before a VM can access Key Vault, DNS records that must exist before a certificate can be issued. In those cases, `depends_on` is the **only** way to express the ordering.

### Step 2c - Initialise and preview

```bash
terraform init
terraform validate
terraform plan -var-file="development.tfvars"
```

> **What's happening here?**  
> You ran `terraform init` back in Step 1c, but that was before `main.tf` existed. Now that `main.tf` declares the `azurerm` provider in its `required_providers` block, `init` runs again to download it. Then `terraform validate` checks the configuration against the provider schema — the same habit from Lab 03: always validate before you plan.

Verify in the plan output:
- The data source is **read** (not created) — it appears before any resource
- `azurerm_storage_account.lab["logs"]` and `["data"]` will be created
- Both show `account_tier = "Standard"` — from `lookup()` on the `"development"` key
- Storage account names contain your initials

Now test the production override:

```bash
terraform plan -var-file="development.tfvars" -var="environment=production"
```

> **Notice:** `account_tier = "Premium"` on both accounts. The `-var` flag overrides the `.tfvars` value. Also notice the `environment` tag now shows `"production"` — because it's derived in `local.tags` via `merge()`, not hardcoded in the tfvars file.

> **Discussion:** When would `depends_on` be truly necessary rather than redundant like here?

### ✅ Checkpoint

- [ ] `data.tf` and `main.tf` exist
- [ ] `terraform init` completed successfully and `terraform validate` reported success
- [ ] `terraform plan -var-file="development.tfvars"` shows `Plan: 2 to add`
- [ ] `account_tier` is `"Standard"` for development and `"Premium"` for production
- [ ] Storage account names in the plan include your initials

---

## Step 3 — Custom conditions and outputs

### What you'll learn
- How `precondition` validates assumptions BEFORE a resource is created.
- How `postcondition` validates the actual result AFTER creation using `self`.
- How `check` blocks provide ongoing compliance assertions without blocking applies.
- How outputs expose values after apply, including the key difference between `sensitive` masking and real security.

### Step 3a - Add a `lifecycle` block with `precondition`

Update `azurerm_storage_account.lab` in `main.tf` to add the lifecycle block:

```hcl
resource "azurerm_storage_account" "lab" {
  for_each = toset(var.storage_accounts)

  name                     = "st${var.initials}wedmgtlab04${each.key}"
  resource_group_name      = data.azurerm_resource_group.existing.name
  location                 = data.azurerm_resource_group.existing.location
  account_tier             = local.storage_tier
  account_replication_type = "LRS"
  tags                     = local.tags

  depends_on = [data.azurerm_resource_group.existing]

  lifecycle {
    precondition {
      condition     = length("st${var.initials}wedmgtlab04${each.key}") <= 24
      error_message = "Storage account name 'st${var.initials}wedmgtlab04${each.key}' exceeds the 24-character limit."
    }
  }
}
```

> **What's happening here?**  
> The `precondition` runs at **plan time**, before Terraform attempts to create anything in Azure. You get a clear, specific error message rather than a cryptic Azure API error about invalid names.

**Test it:** Temporarily add `"applicationlogs"` to the `storage_accounts` list in your `development.tfvars` file and run:

```bash
terraform plan -var-file="development.tfvars"
```

The name `st<your-initials>wedmgtlab04applicationlogs` will exceed 24 characters for any initials — the precondition catches it at plan time before any API call. Remove `"applicationlogs"` from the list before continuing.

### Step 3b - Add a `postcondition`

Extend the `lifecycle` block:

```hcl
  lifecycle {
    precondition {
      condition     = length("st${var.initials}wedmgtlab04${each.key}") <= 24
      error_message = "Storage account name 'st${var.initials}wedmgtlab04${each.key}' exceeds the 24-character limit."
    }

    postcondition {
      condition     = self.account_replication_type == "LRS"
      error_message = "Expected LRS replication but got ${self.account_replication_type}. Check account configuration."
    }
  }
```

> **What's happening here?**  
> The `postcondition` runs **after** the resource is created or updated. `self` refers to the resource itself — giving you access to its actual attributes as returned by Azure. If the cloud provider set up the resource differently than intended, `postcondition` catches it immediately. This is the key difference from `precondition`: you're validating the *result*, not the *input*.

### Step 3c - Add a `check` block

Append to the end of `main.tf` (outside the resource block):

```hcl
check "storage_https_only" {
  assert {
    condition     = alltrue([for acct in azurerm_storage_account.lab : acct.https_traffic_only_enabled])
    error_message = "Warning: all storage accounts should have HTTPS-only traffic enabled."
  }
}
```

> **What's happening here?**  
> A `check` block is fundamentally different from `precondition`: it **never blocks an apply** — it only raises a warning. Use `check` blocks for compliance assertions you want to monitor on every plan and apply without blocking operations. Think of it as a continuous monitor for the lifetime of the infrastructure.
>
> - `precondition` / `postcondition` = hard rules in `lifecycle` — they **block**
> - `check` block = soft rules at top level — they **warn**
>
> Azure enables HTTPS-only by default, so this check always passes in our config. To see the warning fire, add `https_traffic_only_enabled = false` to the resource block and run `terraform plan`.

### Step 3d - Create `outputs.tf`

```hcl
output "storage_endpoints" {
  description = "Primary blob endpoints for all storage accounts."
  value = {
    for name, acct in azurerm_storage_account.lab :
    name => acct.primary_blob_endpoint
  }
}

output "storage_names_list" {
  description = "All storage account names as a flat list."
  value       = [for acct in azurerm_storage_account.lab : acct.name]
}

output "environment_tier" {
  description = "The storage tier selected for this environment via lookup()."
  value       = local.storage_tier
}

output "database_password_hint" {
  description = "Confirms the password variable is set — the value is always masked in output."
  value       = var.database_password
  sensitive   = true
}
```

### Step 3e - Verify with `terraform plan`

```bash
terraform plan -var-file="development.tfvars"
```

The plan should succeed — conditions pass and outputs show `(known after apply)`.

### ✅ Checkpoint

- [ ] `lifecycle { precondition { ... } }` added — the `applicationlogs` test triggered a clear error
- [ ] `lifecycle { postcondition { ... } }` validates `self.account_replication_type`
- [ ] `check` block added at the top level of `main.tf`
- [ ] `outputs.tf` defines all four outputs including the sensitive one
- [ ] `terraform plan -var-file="development.tfvars"` succeeds with `Plan: 2 to add`

---

## Step 4 — Apply, `terraform console` and secrets

### What you'll learn
- How to apply and verify a multi-resource configuration.
- How `terraform console` lets you explore functions and resources interactively after apply.
- How to query outputs in different formats.
- Why `sensitive = true` does **not** protect your secrets in state.

### Step 4a - Apply the configuration

```bash
terraform apply -var-file="development.tfvars"
```

Type `yes`. Open the **Azure Portal** and navigate to your Lab 03 resource group (`rg-<your-initials>-we-d-mgt-lab03`). Verify both storage accounts appear with names containing your initials.

### Step 4b - Explore with `terraform console`

```bash
terraform console -var-file="development.tfvars"
```

Inside the console, try:

```hcl
# The tier selected by lookup() for the current environment
local.storage_tier

# All storage account names from the for_each resources
[for acct in azurerm_storage_account.lab : acct.name]

# Test the check assertion directly
alltrue([for acct in azurerm_storage_account.lab : acct.https_traffic_only_enabled])

# String functions
upper(var.environment)
join("-", ["rg", "tf", "lab"])

# Exit
exit
```

> **What's happening here?**  
> Inside `terraform console` you have access to all variables, locals, resources, and data sources from your current state — it reads live state, not just config. It's the best tool for debugging expressions before committing them.

### Step 4c - Query outputs

```bash
# All outputs in human-readable format
terraform output

# As a JSON map — useful for CI/CD pipeline consumption
terraform output -json storage_endpoints

# As a raw string — useful for shell scripting
terraform output -raw environment_tier
```

> **Notice:** `storage_endpoints` returns a map keyed by storage name. `storage_names_list` returns a flat list. `environment_tier` returns just the string `"Standard"`. The `database_password_hint` output shows `<sensitive>` — the value is masked.

### Step 4d - Show why `sensitive = true` is not security

```bash
cat terraform.tfstate | grep -A3 "database_password"
```

> **Important:** `sensitive = true` masks the value in **terminal output only**. The actual value is stored in `terraform.tfstate` as plain text. This is why production state must use an encrypted remote backend (Azure Blob Storage with customer-managed keys, or HCP Terraform) with restricted access controls. Never commit `terraform.tfstate` to Git.

### Step 4e - Destroy

```bash
terraform destroy -var-file="development.tfvars"
```

Type `yes`. Verify in the Azure Portal that your storage accounts are gone. The resource group remains — Terraform only destroys what it manages, and the data source is read-only.

Now switch to the Lab 03 folder and destroy the resource group:

```bash
cd ~/module03_lab
terraform destroy
```

Type `yes`. Verify in the Portal that `rg-<your-initials>-we-d-mgt-lab03` is gone.

### ✅ Checkpoint

- [ ] `terraform apply` succeeded — 2 storage accounts visible in Azure Portal with your initials in the names
- [ ] `terraform console` evaluated `local.storage_tier` and the `for` expression successfully
- [ ] `terraform output -json storage_endpoints` returns valid JSON
- [ ] `terraform.tfstate` shows `database_password` in plain text — the security lesson
- [ ] `terraform destroy` removed your 2 storage accounts, the Lab 03 resource group remains
- [ ] Lab 03 resource group (`rg-<your-initials>-we-d-mgt-lab03`) destroyed via `cd ~/module03_lab && terraform destroy`

---

## Summary

| What you did | What you learned |
|---|---|
| Declared `variable "environment"` with `validation` | Variables enforce business rules before any API call |
| Added `variable "initials"` with regex validation | Pattern-validated variables guarantee safe, globally-unique storage account names |
| Created `development.tfvars` (no secrets) | Environment-specific values without CLI flags |
| Used `lookup(local.storage_account_tiers, ...)` | Multi-value logic from a map — cleaner than ternary chains |
| Used `merge(var.tags, { environment = var.environment })` | Derived tags always stay consistent with the environment variable |
| Ran `terraform console` | Test HCL expressions interactively without touching cloud resources |
| Used `data "azurerm_resource_group"` | Data sources read existing infra — Terraform never owns them |
| Added `depends_on` | Explicit ordering for dependencies Terraform can't infer from attributes |
| Used `for_each = toset(var.storage_accounts)` | Named instances with stable identity — no index drift like `count` |
| Added `lifecycle { precondition { ... } }` | Validates inputs BEFORE resource creation at plan time |
| Added `lifecycle { postcondition { ... } }` | Validates results AFTER creation using `self.attribute` |
| Added `check "name" { assert { ... } }` | Ongoing compliance assertion — warns, never blocks |
| Set `sensitive = true` on `database_password` | Masks in terminal only — plain text in state, always use encrypted backends |
| Used `TF_VAR_database_password` env var | Pass secrets without files, shell history, or Git |
| Wrote `for` expressions in outputs | Collect attributes from `for_each` resources into maps and lists |

**Key habits from this lab:**
- Always type your variables — `map(string)`, not `any`
- Use `lookup()` when you have more than two conditional values
- Use `terraform console` to test expressions before committing them
- `depends_on` is for real but uninferable dependencies — not just for show
- Prefer `precondition` for blocking input validation, `check` for non-blocking ongoing compliance
- Never put secrets in `.tfvars` files committed to Git — use `TF_VAR_` or a secrets manager
- `sensitive = true` is UI masking, not encryption — encrypted remote state is mandatory in production

---

## Full file reference

### `variables.tf`

```hcl
variable "initials" {
  type        = string
  description = "Your initials, used to make resource names unique (e.g. 'fdv'). Lowercase letters only."

  validation {
    condition     = can(regex("^[a-z]{2,4}$", var.initials))
    error_message = "initials must be 2–4 lowercase letters."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment. Must be development, test, acceptance or production."
  default     = "development"

  validation {
    condition     = contains(["development", "test", "acceptance", "production"], var.environment)
    error_message = "environment must be one of: development, test, acceptance or production."
  }
}

variable "storage_accounts" {
  type        = list(string)
  description = "Short names for each storage account to create."
  default     = ["logs", "data"]
}

variable "tags" {
  type        = map(string)
  description = "Additional tags applied to all resources. The environment tag is always derived from var.environment."
  default = {
    managed_by = "terraform"
  }
}

variable "database_password" {
  type        = string
  description = "Simulated database password. Pass via TF_VAR_database_password — never hardcode."
  sensitive   = true
}
```

### `development.tfvars`

```hcl
environment = "development"
initials    = "fdv" # REPLACE: your own lowercase initials (2–4 letters)
tags = {
  managed_by = "terraform"
}
storage_accounts = ["logs", "data"]
```

### `locals.tf`

```hcl
locals {
  storage_account_tiers = {
    development = "Standard"
    test        = "Standard"
    acceptance  = "Standard"
    production  = "Premium"
  }

  storage_tier = lookup(local.storage_account_tiers, var.environment, "Standard")

  tags = merge(var.tags, { environment = var.environment })
}
```

### `data.tf`

```hcl
data "azurerm_resource_group" "existing" {
  name = "rg-${var.initials}-we-d-mgt-lab03"
}
```

### `main.tf` (complete — after all steps)

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

resource "azurerm_storage_account" "lab" {
  for_each = toset(var.storage_accounts)

  name                     = "st${var.initials}wedmgtlab04${each.key}"
  resource_group_name      = data.azurerm_resource_group.existing.name
  location                 = data.azurerm_resource_group.existing.location
  account_tier             = local.storage_tier
  account_replication_type = "LRS"
  tags                     = local.tags

  depends_on = [data.azurerm_resource_group.existing]

  lifecycle {
    precondition {
      condition     = length("st${var.initials}wedmgtlab04${each.key}") <= 24
      error_message = "Storage account name 'st${var.initials}wedmgtlab04${each.key}' exceeds the 24-character limit."
    }

    postcondition {
      condition     = self.account_replication_type == "LRS"
      error_message = "Expected LRS replication but got ${self.account_replication_type}. Check account configuration."
    }
  }
}

check "storage_https_only" {
  assert {
    condition     = alltrue([for acct in azurerm_storage_account.lab : acct.https_traffic_only_enabled])
    error_message = "Warning: all storage accounts should have HTTPS-only traffic enabled."
  }
}
```

### `outputs.tf`

```hcl
output "storage_endpoints" {
  description = "Primary blob endpoints for all storage accounts."
  value = {
    for name, acct in azurerm_storage_account.lab :
    name => acct.primary_blob_endpoint
  }
}

output "storage_names_list" {
  description = "All storage account names as a flat list."
  value       = [for acct in azurerm_storage_account.lab : acct.name]
}

output "environment_tier" {
  description = "The storage tier selected for this environment."
  value       = local.storage_tier
}

output "database_password_hint" {
  description = "Confirms the password variable is set — value is always masked in output."
  value       = var.database_password
  sensitive   = true
}
```

---

## Further reading

- [Customize Terraform configuration with variables](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables)
- [Output data from Terraform](https://developer.hashicorp.com/terraform/tutorials/configuration-language/outputs)
- [Query data sources](https://developer.hashicorp.com/terraform/tutorials/configuration-language/data-sources)
- [Create resource dependencies](https://developer.hashicorp.com/terraform/tutorials/configuration-language/dependencies)
- [Perform dynamic operations with functions](https://developer.hashicorp.com/terraform/tutorials/configuration-language/functions)
- [Validate modules with custom conditions](https://developer.hashicorp.com/terraform/tutorials/configuration-language/custom-conditions)
- [Use checks to validate infrastructure](https://developer.hashicorp.com/terraform/tutorials/configuration-language/checks)
