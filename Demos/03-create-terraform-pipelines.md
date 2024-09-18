# Lab 3: Create Terraform pipelines

## Objective

While you can apply Terraform configuration projects locally, there are many benefits to setting up continuous integration/continuous development (CI/CD) pipelines to handle this task automatically when you push changes into the repository. We will create the CI and CD Terraform pipelines for your workshop project in this lab.

## Prerequisites

- Completion of  [Lab 2](02-initiaize-terraform-remote-state.md) 
- Azure CLI installed and authenticated on your local machine

## Steps

### Part 1: Create an Azure service principal for the CI/CD pipelines to use

In a future lab, we will set up continuous integration/continuous development (CI/CD) pipelines to deploy Azure infrastructure changes when the Terraform configuration changes. GitHub/Azure DevOps will require an Azure service principal to execute the Terraform apply command to your Azure subscription.

1. Open a terminal or command prompt.
2. If not already logged into your Azure account, log in to your Azure account:

``` sh
az login
```

- Follow the instructions to log in. This step is not needed if you are using a cloud shell.

3. Get your Azure subscription identifier:

```sh
az account show
```

This will return several pieces of information about your Azure subscription. Please take note of the `ID` value, as you will need it in the next step.

4. Create a service principal:

```sh
az ad sp create-for-rbac --name "TerraformServicePrincipal" --role Contributor --scopes /subscriptions/YOUR_SUBSCRIPTION_ID
```

- Replace `YOUR_SUBSCRIPTION_ID` with your actual Azure subscription identifier.
- This command will output a JSON object containing the necessary credentials.

Here is an example of what the JSON output might look like:

```json
{
  "appId": "YOUR_APP_ID",
  "displayName": "myServicePrincipal",
  "password": "YOUR_PASSWORD",
  "tenant": "YOUR_TENANT_ID"
}
```

Save the outputted JSON somewhere for future use.

> [!CAUTION] 
>
> Do not save the outputted JSON in your Git repository. This contains secrets that should not be stored in a repository.

### Part 2: Add service principal to GitHub

1. In your local `infra\github` folder, add the following configuration to the `variables.tf` file:

   ```hcl
   variable "client_secret" {
     type    = string
   }
   
   variable "subscription_id" {
     type    = string
   }
   
   variable "tenant_id" {
     type    = string
   }
   
   variable "client_id" {
     type    = string
   }
   
   # Create a JSON object for the Azure credentials
   locals {
     azure_credentials = jsonencode({
       clientSecret     = var.client_secret
       subscriptionId   = var.subscription_id
       tenantId         = var.tenant_id
       clientId         = var.client_id
     })
   }
   ```

2. In your local `infra\github` folder, add the following variable assignments to the `dev.tfvars` file:

   ```hcl
   client_secret   = "'password' from above"
   subscription_id = "YOUR_SUBSRCRIPTION_ID"
   tenant_id       = "'tenant' from above"
   client_id       = "'appId' from above"
   ```

3. In your local `infra\github` folder, add the following configuration to the `main.tf` file:

   ```hcl
   resource "github_actions_secret" "azure_credentials" {
     repository       = github_repository.ecommerce_repo.name
     secret_name      = "AZURE_CREDENTIALS"
     plaintext_value = local.azure_credentials
   }
   ```

4. In a command or Terminal window and from the `infra\github` folder, validate the configuration changes:

   ```hcl
   terraform validate
   ```

5. In a command or Terminal window and from the `infra\github` folder, plan the configuration changes:

   ```hcl
   terraform apply --var-file=dev.tfvars
   ```

### Part 3: Permit the service principal to assign roles

1. Go to the [Azure Portal](https://portal.azure.com).
2. Click on **Subscriptions**.
3. Click on the subscription you are working with for the workshop.
4. Click on **Access control (IAM)**.
5. In the **Create a custom role** card, click the **Add** button.
6. Enter `Terraform Service Principal` in the **Custom role name** field.
7. Click the **Next** button.
8. Click the **Add permissions** button.
9. Search for `Microsoft.Authorization/roleAssignments`. Click the **Microsoft Authorization** card.
10. Select all three (Read, Write, Delete) permissions under **Microsoft.Authozization/roleAssignments**.
11. Click the **Add** button.
12. Go to the **Review + create** tab.
13. Click the **Create** button.
14. After returning to the **Access control (IAM)** page, click **Add** > **Add role assignment**.
15. Go to the **Privileged administrator roles** tab.
16. Select **Terraform Service Principal** and click the **Next** button.
17. With the **User, group, or service principal** option selected, click the **+ Select members** link.
18. Search for and select `TerraformServicePrincipal` and click the **Select** button.
19. Click the **Next** button.
20. ON the **Conditions** tab, select **Allow user to assign all roles except privileged administrator roles Owner, RBAC (Recommended)**.
21. Click the **Review + assign** button.
22. Click the **Review + assign** button (again).

### Part 4: Add Secrets

To connect to the Terraform remote state, the `terraform init` command will need details on the storage account, including the storage account key. We do not want that key in plain text in the repository, so we will add a secret.

1. In your local `infra\github` folder, add the azurerm provider to the `providers.tf` file. The full file should look like:

   ```hcl
   terraform {
     required_providers {
       github = {
         source  = "integrations/github"
         version = "~> 6.0"
       }
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~>3.0"
       }
     }
   }
   
   provider "github" {
     token = var.github_token
   }
   
   provider "azurerm" {
     features {}
   }
   ```

   

2. In your local `infra\github` folder, add the following variable assignments to the `dev.tfvars` file:

   ```hcl
   variable "azure_region" {
   	type        = string
   	description = "Location of the resource group."
   }
   
   variable "azure_environment" {
   	type        = string
   	description = "The environment component of an Azure resource name. Valid values are dev, qa, e2e, core, and prod."
   }
   
   variable "resource_name_suffix" {
     type        = string
     default     = ""
     description = "The suffix to append to the resource names."
   }
   ```

3. In your local `infra\github` folder, add a `modules.tf` file with the following configuration:

   ```hcl
   module "azure_regions" {
     source       = "git::https://github.com/TaleLearnCode/terraform-azure-regions.git"
     azure_region = var.azure_region
   }
   
   module "resource_group" {
     source        = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
     resource_type = "resource-group"
   }
   
   module "storage_account" {
     source        = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
     resource_type = "storage-account"
   }
   ```

4. In your local `infra\github` folder, add a `referenced-resources.tf` file with the following configuration:

   ```hcl
   data "azurerm_resource_group" "terraform" {
     name = "${module.resource_group.name.abbreviation}-SwissArmyKnifeInfrastructure-${var.azure_environment}-${module.azure_regions.region.region_short}"
   }
   
   data "azurerm_storage_account" "terraform" {
     name                = lower("${module.storage_account.name.abbreviation}Terraform${var.resource_name_suffix}${var.azure_environment}${module.azure_regions.region.region_short}")
     resource_group_name = data.azurerm_resource_group.terraform.name
   }
   ```

5. In your local `infra\github` folder, add the following configuration to the `dev.tfvars` file:

   ```hcl
   azure_environment    = "dev"
   azure_region         = "eastus2"
   resource_name_suffix = "<<RESOURCE_NAME_SUFFIX>>"
   ```

6. In your local `infra\github` folder, add the following configuration to the `main.tf` file:

   ```
   resource "github_actions_secret" "azure_ad_client_id" {
     repository       = github_repository.ecommerce_repo.name
     secret_name      = "AZURE_AD_CLIENT_ID"
     plaintext_value = var.client_id
   }
   
   resource "github_actions_secret" "azure_ad_client_secret" {
     repository       = github_repository.ecommerce_repo.name
     secret_name      = "AZURE_AD_CLIENT_SECRET"
     plaintext_value = var.client_secret
   }
   
   resource "github_actions_secret" "azure_subscription_id" {
     repository       = github_repository.ecommerce_repo.name
     secret_name      = "AZURE_SUBSCRIPTION_ID"
     plaintext_value = var.subscription_id
   }
   
   resource "github_actions_secret" "azure_ad_tenant_id" {
     repository       = github_repository.ecommerce_repo.name
     secret_name      = "AZURE_AD_TENANT_ID"
     plaintext_value = var.tenant_id
   }
   
   resource "github_actions_secret" "terraform_storage_account_name" {
     repository       = github_repository.ecommerce_repo.name
     secret_name      = "TERRAFORM_STORAGE_ACCOUNT_NAME"
     plaintext_value = data.azurerm_storage_account.terraform.name
   }
   
   resource "github_actions_secret" "terraform_resource_group" {
     repository       = github_repository.ecommerce_repo.name
     secret_name      = "TERRAFORM_RESOURCE_GROUP"
     plaintext_value = data.azurerm_resource_group.terraform.name
   }
   ```

7. In a command or Terminal window and from the `infra\github` folder, initialize the Terraform project:

   ```hcl
   terraform init --backend-config=dev.tfconfig
   ```

8. In a command or Terminal window and from the `infra\github` folder, validate the configuration changes:

   ```hcl
   terraform validate
   ```

9. In a command or Terminal window and from the `infra\github` folder, plan the configuration changes:

   ```hcl
   terraform apply --var-file=dev.tfvars
   ```

### Part 5: Create the CD infrastructure pipeline

**Continuous Deployment (CD)** is a software development approach in which application code changes are automatically deployed into the target environment. In this step, we will create the pipeline to publish infrastructure changes automatically once they are pushed into the repository.

1. In the `.github\workflows` folder, create a file named `cd-infra.yml` and add the following to the file:

   ```yaml
   name: Continuous Deployment
   
   on:
     push:
       branches:
         - develop
       paths:
         - 'infra/solution/**'
   
   jobs:
     paths-filter:
       runs-on: ubuntu-latest
       outputs:
         terraform: ${{ steps.filter.outputs.terraform }}
         production-schedule-facade: ${{ steps.filter.outputs.production-schedule-facade }}
       steps:
         - uses: actions/checkout@v4
         - uses: dorny/paths-filter@v3
           id: filter
           with:
             filters: |
               terraform:
                 - 'infra/solution**'
   
     terraform:
       name: 'Terraform CI'
       needs: paths-filter
       env:
         ARM_CLIENT_ID: ${{ secrets.AZURE_AD_CLIENT_ID }}
         ARM_CLIENT_SECRET: ${{ secrets.AZURE_AD_CLIENT_SECRET }}
         ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
         ARM_TENANT_ID: ${{ secrets.AZURE_AD_TENANT_ID }}
       runs-on: ubuntu-latest
       defaults:
         run:
           shell: bash
       if: needs.paths-filter.outputs.terraform == 'true'
       steps:
         # Checkout the repository to the GitHub Actions runner
         - name: Checkout
           uses: actions/checkout@v4
     
         # Install the latest version of Terraform
         - name: Setup Terraform
           uses: hashicorp/setup-terraform@v1
     
         # Initialize the Terraform project
         - name: Terraform Init
           run: terraform init -backend-config="resource_group_name=${{ secrets.TERRAFORM_RESOURCE_GROUP }}" -backend-config="storage_account_name=${{ secrets.TERRAFORM_STORAGE_ACCOUNT_NAME }}" -backend-config="container_name=terraform-state" -backend-config="key=remanufacturing.tfstate"
           working-directory: ./infra
     
         # Validate the Terraform configuration
         - name: Terraform Validate
           run: terraform validate
           working-directory: ./infra
     
         # Generate and show an execution plan
         - name: Terraform Plan
           run: terraform plan --var-file=gh.tfvars -out=tfplan
           working-directory: ./infra
     
         - name: Terraform Apply
           run: terraform apply -auto-approve tfplan
           working-directory: ./infra
   ```

2. Click on the `Source Control` tab within Visual Studio Code.

3. Add an appropriate commit message:

   ```
   Adding infrastructure pipeline.
   ```

4. Click the **Commit** button.

5. Click the **Sync Changes** button.

## Conclusion

In this work instruction, we have prepared the GitHub repository to automatically apply Terraform changes within the `infra/solution` folder.

## Next Steps

In the next work instruction, we will create the Azure resources for the order processing system.