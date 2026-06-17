# Terraform Workshop — Labs

Hands-on labs for learning **Terraform** on **Azure**. You'll start from the core workflow, move through the configuration language, and finish by building a reusable module — deploying (and cleaning up) real Azure resources at every step.

> **Before you start:** you need **Terraform ≥ 1.5** ([install](https://developer.hashicorp.com/terraform/install)), the **Azure CLI** ([install](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)), and access to an Azure subscription. On **Windows**, run the commands in **Git Bash** or **WSL**. Each lab restates its own prerequisites.

## The labs

Do them **in order** — each builds on the last (Lab 04 reuses the resource group you create in Lab 03).

| # | Lab | Duration | What you'll practise |
|---|-----|----------|----------------------|
| **03** | [Core Terraform workflow](module03/README.md) | ~30 min | write & format config · init · validate · plan · apply · inspect state · detect drift · destroy |
| **04** | [Terraform configuration language](module04/README.md) | ~30 min | variables · `.tfvars` · locals & `lookup()` · data sources · `for_each` · pre/postconditions & checks · outputs & secrets |
| **05** | [Build and use a local module](module05/README.md) | ~25 min | author a module · call it · the scope boundary · two instances from one definition |

## How it works

- Read each lab right here on GitHub, or `git clone` / **Download ZIP** to keep a local copy.
- You'll create files in a working folder (e.g. `~/module03_lab`) and run Terraform commands — just follow along step by step.
- Every lab ends by destroying what it created, so nothing is left running on your subscription.

---

🌍 Happy Terraforming!
