# Work Instruction 4: Create Core Azure Resources

## Objective

Now that we have the infrastructure set up for our repository and the storage of the remote state of the Terraform projects, let's create the resources for a possible order processing system.

## Prerequisites

- Completion of  [Lab 3](03-create-service-principal.md) 
- Terraform is installed on your local machine.
- Azure CLI installed and authenticated on your local machine

## Steps

1. From the local repository folder created in [Lab 1](01-initializing-solution.md), create the `infra\solution` folder.

2. Add a `providers.tf` file in the folder with the following configuration:

   ```hcl
   # #############################################################################
   #                          Terraform Configuration
   # #############################################################################
   
   terraform {
     required_providers {
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~>3.0"
       }
     }
   }
   
   provider "azurerm" {
     skip_provider_registration = true
     features {
       resource_group {
         prevent_deletion_if_contains_resources = true
       }
     }
   }
   ```

3. Add a `variables.tf` file in the folder with the following configuration:

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
   
   locals {
     criticality = var.azure_environment == "dev" ? "Medium" : var.azure_environment == "qa" ? "High" : var.azure_environment == "e2e" ? "High" : var.azure_environment == "prod" ? "Mission Critical" : "Medium"
     disaster_recovery = var.azure_environment == "dev" ? "Dev" : var.azure_environment == "qa" ? "Dev" : var.azure_environment == "e2e" ? "Dev" : var.azure_environment == "prod" ? "Mission Critical" : "Dev"
     tags = {
       Product      = "ECommerce"
       Criticiality = local.criticality
       CostCenter   = "SwissArmyKnife-${var.azure_environment}"
       DR           = local.disaster_recovery
       Env          = var.azure_environment
     }
   }
   ```

4. Add a `dev.tfvars` file in the folder with the following variable values:

   ```hcl
   azure_environment    = "dev"
   azure_region         = "eastus2"
   resource_name_suffix = "<<RESOURCE_NAME_SUFFIX>>"
   ```

5. Add a `modules.tf` file in the folder with the following configuration:

   ```hcl
   # #############################################################################
   #                          Modules
   # #############################################################################
   
   module "azure_regions" {
     source       = "git::https://github.com/TaleLearnCode/terraform-azure-regions.git"
     azure_region = var.azure_region
   }
   ```

6. Add a `main.tf` file in the folder with the following configuration:

   ```hcl
   resource "azurerm_resource_group" "rg" {
     name     = "rg-SwissArmyKnife-${var.azure_environment}-${module.azure_regions.region.region_short}"
     location = module.azure_regions.region.region_cli
     tags     = local.tags
   }
   
   resource "azurerm_servicebus_namespace" "sb" {
     name                = "sbns-SwissArmyKnife-${var.azure_environment}-${module.azure_regions.region.region_short}"
     location            = azurerm_resource_group.rg.location
     resource_group_name = azurerm_resource_group.rg.name
     sku                 = "Standard"
   }
   
   resource "azurerm_servicebus_topic" "order_topic" {
     name         = "sbns-SwissArmyKnifeOrder-${var.azure_environment}-${module.azure_regions.region.region_short}"
     namespace_id = azurerm_servicebus_namespace.sb.id
   }
   
   resource "azurerm_cosmosdb_account" "cosmos" {
     name                = lower("cosno-SwissArmyKnife-${var.azure_environment}-${module.azure_regions.region.region_short}")
     location            = azurerm_resource_group.rg.location
     resource_group_name = azurerm_resource_group.rg.name
     offer_type          = "Standard"
     kind                = "GlobalDocumentDB"
   
     consistency_policy {
       consistency_level = "Session"
     }
   
     geo_location {
       location          = azurerm_resource_group.rg.location
       failover_priority = 0
     }
   }
   
   resource "azurerm_cosmosdb_sql_database" "database" {
     name                = "ecommerce"
     resource_group_name = azurerm_resource_group.rg.name
     account_name        = azurerm_cosmosdb_account.cosmos.name
   }
   
   resource "azurerm_cosmosdb_sql_container" "container" {
     name                  = "orders"
     resource_group_name   = azurerm_resource_group.rg.name
     account_name          = azurerm_cosmosdb_account.cosmos.name
     database_name         = azurerm_cosmosdb_sql_database.database.name
     partition_key_paths   = ["/orderId"]
     throughput            = 400
   }
   
   resource "azurerm_storage_account" "function" {
     name                     = lower("stsakf${var.resource_name_suffix}${var.azure_environment}${module.azure_regions.region.region_short}")
     resource_group_name      = azurerm_resource_group.rg.name
     location                 = azurerm_resource_group.rg.location
     account_tier             = "Standard"
     account_replication_type = "LRS"
   }
   
   resource "azurerm_service_plan" "function" {
     name                = "asp-SwissArmyKnifeFunction-${var.azure_environment}-${module.azure_regions.region.region_short}"
     resource_group_name = azurerm_resource_group.rg.name
     location            = azurerm_resource_group.rg.location
     os_type             = "Linux"
     sku_name            = "Y1"
   }
   
   resource "azurerm_linux_function_app" "ecommerce" {
     name                = "func-SwissArmyKnifeOrderProcessing-${var.azure_environment}-${module.azure_regions.region.region_short}"
     resource_group_name = azurerm_resource_group.rg.name
     location            = azurerm_resource_group.rg.location
   
     storage_account_name       = azurerm_storage_account.function.name
     storage_account_access_key = azurerm_storage_account.function.primary_access_key
     service_plan_id            = azurerm_service_plan.function.id
   
     identity {
       type = "SystemAssigned"
     }
     
     site_config {}
   }
   
   resource "azurerm_eventgrid_topic" "order_events" {
     name                = "evgt-SwissArmyKnifeOrder-${var.azure_environment}-${module.azure_regions.region.region_short}"
     location            = azurerm_resource_group.rg.location
     resource_group_name = azurerm_resource_group.rg.name
   }
   
   data "azurerm_client_config" "current" {}
   
   resource "azurerm_key_vault" "kv" {
     name                = lower("kv-SAK-${var.azure_environment}-${module.azure_regions.region.region_short}")
     location            = azurerm_resource_group.rg.location
     resource_group_name = azurerm_resource_group.rg.name
   
     tenant_id = data.azurerm_client_config.current.tenant_id
     sku_name  = "standard"
   }
   
   resource "azurerm_role_assignment" "function_secret_user" {
     scope                = azurerm_key_vault.kv.id
     role_definition_name = "Key Vault Secrets User"
     principal_id         = azurerm_linux_function_app.ecommerce.identity.*.principal_id[0]
   }
   ```

7. Copy the `dev.tfconfig` file from the `infra\remote-state` folder and update the key to `solution.tfstate`. Your `dev.tfconfig` file should like simliar to:

   ```hcl
   storage_account_name = "stterraformrandomdevuse2"
   container_name = "terraform-state"
   key = "solution.tfstate"
   sas_token = "<<SAS_TOKEN>>"
   ```

8. Initialize the Terraform project:

   ```sh
   cd ..\solution
   terraform init --backend-config=dev.tfconfig
   ```

9. Validate the Terraform project:

   ```hcl
   terraform validate
   ```

10. Plan the Terraform project:

    ```hcl
    terraform plan --var-file=dev.tfvars
    ```

11. Commit and push the changes to GitHub.

12. Validate that the `Continuous Deployment` GitHub action fires and completes.







In the last lab, we turned on branch protection rules, so we can no longer merge directly to the `develop` (or `main`) branch.

1. In the lower-left corner of Visual Studio Code, click the name of the branch (`develop`).
2. Select the **Create new branch** option.
3. Enter `04-core-resources` in the **Branch name** prompt.
4. Click the **Publish Branch** button.

### Step 2: Configure the providers

A Terraform **provider** is a plugin that allows Terraform to manage resources on various platforms and services. Each supported service or infrastructure platform has its provider, which defines the available resources and handles API calls to manage those resources. There are a couple of providers for Azure, but the primary one you will work with is the Azure Provider (azurerm).

When building a Terraform configuration, you must define the provider(s) used by the configuration, which is what we do in this step.

1. (If not already) open Visual Studio Code.
2. (If not already) open the repository folder.
3. In the `infra` folder, create the `providers.tf` file and add the following to the file:

```HCL
# #############################################################################
# Provider Configuration
# #############################################################################

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
  backend "azurerm" {
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
}
```

For this project, we are doing the following in this `providers.tf` file:

- Configuring the overall Terraform project indicates that.
  - We are requiring the latest 3.0 AzureRM provider from the HashiCorp register.
  - We are storing the state of the Terraform project using the AzureRM provider.
- Configuring the AzureRM provider with the following options:
  - Ensuring that any associated Azure resource groups are not deleted if they are not empty.

### Step 3: Configure the core variables

Terraform **variables** are placeholders for values that you can use to make your configurations more dynamic and reusable. They allow you to define values that can be reused across multiple resources, modules, and environments. You can parameterize your configuration files with variables, making them more flexible and adaptable.

In this step, we will configure the standard variables the Terraform configuration uses for the workshop.

1. In the `infra` folder, create the `variables.tf` file and add the following to the file:

```HCL
# #############################################################################
# Common Variables
# #############################################################################

variable "azure_region" {
  type        = string
  description = "Location of the resource group."
}

variable "azure_environment" {
  type        = string
  description = "The environment component of an Azure resource name."
}

variable "resource_name_suffix" {
  type        = string
  description = "The suffix to append to the resource names."
}
```

2. In the `infra` folder, create the `dev.tfvars` file and add the following to the file:

```HCL
azure_region         = "centralus"
azure_environment    = "dev"
resource_name_suffix = "{random-value}"
```

A **tfvars** file is used to manage variable assignments systematically. These files have either the extension `.tfvars` or `tfvars.json`.

3. Update the `resource_name_suffix` value to match what was returned during [Lab 2](02-initialize-terraform-remote-state.md).

> If you used a different region or environment name during Lab 2, update those values here.

### Step 4: Configure modules

A Terraform **module** is a collection of Terraform configuration files in a single directory that encapsulates Terraform configuration. It reduces the code you must develop for similar infrastructure components and helps promote the **DRY** (Don't Repeat Yourself) concept.

In this step, we will configure the modules used by the Terraform project to the point of completing this lab.

1. In the `infra` folder, create the `modules.tf` file and add the following to the file:

```HCL
# #############################################################################
# Modules
# #############################################################################

module "azure_regions" {
  source       = "git::https://github.com/TaleLearnCode/terraform-azure-regions.git"
  azure_region = var.azure_region
}

module "resource_group" {
  source        = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "resource-group"
}

module "api_management" {
  source        = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "api-management-service-instance"
}

module "service_bus_namespace" {
  source        = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "service-bus-namespace"
}

module "cosmos_account" {
  source        = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "azure-cosmos-db-for-nosql-account"
}

module "log_analytics_workspace" {
  source = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "log-analytics-workspace"
}

module "application_insights" {
  source = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "application-insights"
}

module "key_vault" {
  source = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "key-vault"
}

module "app_config" {
  source = "git::https://github.com/TaleLearnCode/azure-resource-types.git"
  resource_type = "app-configuration-store"
}
```

There are two different modules here, with one of those being used multiple times:

- **terraform-azure-regions**: This module provides Azure region information for the specified Azure region.
- **azure-resource-types**: This module helps to keep the naming of Azure resources consistent. The goal of this module is to provide the resource type abbreviation to be used within the name of an Azure resource. You need to declare this module for each resource type you need. At this point, we are using it for the following types of Azure resources:
  - Resource Group
  - API Management
  - Service Bus namespace
  - Log Analytics Workspace
  - Application Insights
  - Key Vault
  - App Configuration Store

## Step 5: Add references to global resources

In Terraform, a **data** element references information retrieved from external sources or other configurations.

In this step, we will define the global resources for the Terraform project (at this stage of its development).

1. In the `infra` folder, create the `referenced-resources.tf` file and add the following to the file:

```HCL
# #############################################################################
# Referenced Resoruces
# #############################################################################

data "azurerm_client_config" "current" {}
```

In this file, we are creating a data source to access the configuration of the AzureRM provider. This will give us details about your Azure account, which we will use to create resources later.

### Step 6: Create the global configuration file

Now that we have all the ancillary configuration elements, we can add elements to manage Azure resources. We will start by creating the configuration for assigning tags to the Azure resources we will make.

In Azure, **tags** are metadata elements that consist of key-value pairs. They help you identify resources based on settings relevant to your organization.

We will define some variables for parameterization, but this section introduces **locals**. Some named expressions will enable you to assign values within your code. Think of them as internal variables inaccessible from outside the configuration.

1. In the `infra` folder, create the `global.tf` file and add the following to the file:

```HCL
# #############################################################################
# Core Remanufacturing resources
# #############################################################################

# -----------------------------------------------------------------------------
#                             Tags
# -----------------------------------------------------------------------------

variable "global_tag_product" {
  type        = string
  default     = "Remanufacturing"
  description = "The product or service that the resources are being created for."
}

variable "global_tag_cost_center" {
  type        = string
  default     = "Remanufacturing"
  description = "Accounting cost center associated with the resource."
}

variable "global_tag_criticality" {
  type        = string
  default     = "Medium"
  description = "The business impact of the resource or supported workload. Valid values are Low, Medium, High, Business Unit Critical, Mission Critical."
}

variable "global_tag_disaster_recovery" {
  type        = string
  default     = "Dev"
  description = "Business criticality of the application, workload, or service. Valid values are Mission Critical, Critical, Essential, Dev."
}

locals {
  global_tags = {
    Product     = var.global_tag_product
    Criticality = var.global_tag_criticality
    CostCenter  = "${var.global_tag_cost_center}-${var.azure_environment}"
    DR          = var.global_tag_disaster_recovery
    Env         = var.azure_environment
  }
}
```



> [!TIP]
>
> Terraform allows you to put the different configuration elements anywhere within the Terraform project folder you desire as long as they are in a **Tf** file. A good practice for larger projects (this will become) is to create separate **TF** files for each resource/resource type. But to make the labs a little easier, we are grouping resources by area/microservice.

### Step 7: Add the global resource group to the configuration

In the `global.tf` file, add the following:

```HCL
# -----------------------------------------------------------------------------
# Resource Group
# -----------------------------------------------------------------------------

resource "azurerm_resource_group" "global" {
  name     = "${module.resource_group.name.abbreviation}-CoolRevive-${var.azure_environment}-${module.azure_regions.region.region_short}"
  location = module.azure_regions.region.region_cli
  tags     = local.global_tags
}
```

### Step 8: Add the API Management service instance to the configuration

1. Add the following to the `global.tf` file:

```HCL
# -----------------------------------------------------------------------------
# API Management
# -----------------------------------------------------------------------------

variable "apim_publisher_name" {
  type        = string
  description = "The name of the publisher of the API Management instance."
}

variable "apim_publisher_email" {
  type        = string
  description = "The email address of the publisher of the API Management instance."
}

variable "apim_sku_name" {
  type        = string
  description = "The SKU of the API Management instance."
}

resource "azurerm_api_management" "global" {
  name                = lower("${module.api_management.name.abbreviation}-CoolRevive${var.resource_name_suffix}-${var.azure_environment}-${module.azure_regions.region.region_short}")
  location            = azurerm_resource_group.global.location
  resource_group_name = azurerm_resource_group.global.name
  publisher_name      = var.apim_publisher_name
  publisher_email     = var.apim_publisher_email
  sku_name            = var.apim_sku_name
  tags                = local.global_tags
}
```

> [!TIP]
>
> Azure API Management (like many products of its type) is extensive and encompassing. In the Cool Revive scenario, we will set up just one API Management instance for the whole organization. For many organizations, this will make sense. Large companies might want to consider running different instances based on organizational divisions. This will depend on the organization's needs.

2. Add the following variable values to the `dev.tfvars` file:

```HCL
apim_publisher_name  = "Cool Revive"
apim_publisher_email = "{Your_Email_Address}"
apim_sku_name        = "Developer_1"
```

3. Replace `{Your_Email_Address}` with your email address.

### Step 9: Setup the backend configuration information

1. Copy the `infra\remote-state\dev.tfconfig` file

> The tfconfig file name uses the `azure_environment` variable value, so if you change that value, the file will be named accordingly.

2. Paste the file into the `infra` folder.

3. Open the `infra\dev.tfconfig` file
4. Update the key value to `remanufacturing.tfstate`

Your `infra\dev.tfconfig` file should look similar to this:

```hcl
storage_account_name = "stterraform000devcus"
container_name = "terraform-state"
key = "remanufacturing.tfstate"
sas_token = "The SAS token"
```

### Step 10: Initialize the Terraform project

In the Visual Studio Code Terminal window, execute the following command:

```sh
cd ..
terraform init --backend-config="dev.tfconfig"
```

> [!IMPORTANT] 
>
> Be sure that you are in the correct directory within the Terminal window. We previously ran commands within the `infra\remote-state` directory, and now we need to execute the commands in the `infra` folder.

Hopefully, you will receive the following message: **Terraform has been successfully installed!** However, part of the initialization process is scanning the project for errors, so you might need to correct something and rerun the `terraform init` command.

### Step 11: Validate the Terraform project

1. In the Visual Studio Code Terminal window, execute the following command:

```hcl
terraform validate
```

The `terraform validate` command runs checks that verify whether a configuration is syntactically valid and internally consistent, regardless of any provided variables or existing state. It is primarily helpful for verifying reusable modules, including the correctness of attribute names and value types.

> [!TIP]
>
> It is best practice to execute the `terraform validate` command whenever you make changes to your Terraform project to ensure that you didn't make any apparent mistakes.

2. Correct any validation errors and rerun the `terraform validate` command until the validation is successful.

### Step 12: Plan the Terraform project

1. In the Visual Studio Code Terminal window, execute the following command:

```hcl
terraform plan --var-file=dev.tfvars
```

> The `terraform plan` command creates an execution plan, letting you preview the changes Terraform will make to your infrastructure.

2. Inspect the Terraform plan to verify it will make the changes you expect.

> The plan results should look like this:
>
> **Plan:** 3 to add, 0 to change, 0 to destroy

### Step 13: Commit and push the Terraform project to your central Git repository

1. Click on the `Source Control` tab within Visual Studio code.
2. Add an appropriate commit message:

```
Adding the primary Terraform configuration project.
```

3. Click the **Commit** button (this will perform a `git commit` command).
4. Click the **Sync Changes** button (this will perform a `git push` command).

### Step 14: Create a pull request and merge it into develop

#### Using GitHub

1. Navigate to your GitHub repository.
2. Click the **Pull requests** tab.
3. Click the **New pull request** button.
4. Change the **compare** to `features/04-core-resources`.
5. Click the **Create pull request** button.
6. Click the **Create pull request** button.
7. Once all of the checks have passed, click the **Merge pull request** button.
8. Click the **Confirm merge** button.
9. Open the **Actions** tab in a separate browser tab and validate that the `Terraform (CD)` pipeline is running.

> [!NOTE]
>
> Creating the API Management instance typically takes 20 to 25 minutes to execute. We can proceed to the next lab.



## Conclusion

In this lab, you initialize the primary Terraform configuration project, which will manage the resources for the Cool Revive Remanufacturing system.

## Next Steps

In the next lab, you will build the software development platform pipeline to automatically apply the Terraform configuration changes when pushed to the development branch.