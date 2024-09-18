# Work Instruction 2: Initialize Terraform Remote State

## Objective

Terraform uses **Terraform state** to determine which changes need to be made to your infrastructure. The state file keeps track of resources created by your configuration and maps to real-world resources. Before any operation, Terraform refreshes the state with the actual infrastructure. The primary purpose of the Terraform state is to store bindings between objects in a remote system and resource instances declared in your configuration.

By default, the Terraform state is stored locally. **Terraform remote state** allows you to store the state data in a remote data store, which can be shared among team members and automated processes.

In this work instruction, you will create the Terraform remote state for your project.

## Prerequisites

- Completion of [Lab 1](01-initializing-solution.md).
- Terraform is installed on your local machine.
- Azure CLI is installed on your local machine.

## Steps

### Part 1: Initialize the remote state Terraform configuration project

1. Open Visual Studio Code.

2. Open the local repository folder created in [Lab 1](01-initializing-solution.md).

3. In the `infra` folder, create a `remote-state` folder.

4. In the `infra\remote-state` folder, create the `main.tf` file and add the following:

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
       random = {
         source = "hashicorp/random"
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
   
   provider "local" {
   
   }
   ```

   Here, we have defined that we will use the `azurerm`, `random`, and `local` providers.

5. Add the following module declarations tot he `main.tf` file:

   ```hcl
   # #############################################################################
   #                          Modules
   # #############################################################################
   
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

   We are using two different modules, with one of those modules being used twice with a different input parameter:

   - **azure_regions** - Provides name information for the Azure regions.
   - **resource_group** - Provides naming details for resource groups.
   - **storage_account** - Provides naming details for storage accounts.

6. Add the following variable declarations to the `main.tf` file:

   ```hcl
   # #############################################################################
   #                          Variables
   # ############################################################################
   
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
     default     = ""
     description = "The suffix to append to the resource names."
   }
   ```

7. Add the following to the `main.tf` to create a random name suffix:

   ```hcl
   # #############################################################################
   #                           Resource Name Suffix
   # #############################################################################
   
   resource "random_integer" "identifier" {
     min = 100
     max = 999
     keepers = {
       test_name = "${var.azure_environment}_${var.azure_region}"
     }
   }
   
   locals {
     resource_suffix = var.resource_name_suffix == "random" ? random_integer.identifier.result : var.resource_name_suffix
   }
   ```

   > Many Azure resources require globally unique names. To prevent collision with others using these work instructions, we add a random 3-digital suffix to the resource names to help ensure their uniqueness.

8. Add the following to the `main.tf` file to initialize the tag information for the Azure resources the configuration will manage:

   ```hcl
   # #############################################################################
   #                          Define the Tags
   # #############################################################################
   
   locals {
     criticality = var.azure_environment == "dev" ? "Medium" : var.azure_environment == "qa" ? "High" : var.azure_environment == "e2e" ? "High" : var.azure_environment == "prod" ? "Mission Critical" : "Medium"
     disaster_recovery = var.azure_environment == "dev" ? "Dev" : var.azure_environment == "qa" ? "Dev" : var.azure_environment == "e2e" ? "Dev" : var.azure_environment == "prod" ? "Mission Critical" : "Dev"
     tags = {
       Product      = "InfrastructureAsCode"
       Criticiality = local.criticality
       CostCenter   = "InfrastructureAsCode-${var.azure_environment}"
       DR           = local.disaster_recovery
       Env          = var.azure_environment
     }
   }
   ```

   > Azure resources can be decorated with custom tags to help with sorting, searching, and reporting.

9. Add the following `main.tf` to retrieve the current client (you), that is planning/applying the Terraform project:

   ```hcl
   # #############################################################################
   #                       AzureRM Provider Configuration
   # #############################################################################
   
   data "azurerm_client_config" "current" {}
   ```

10. Add the following to the `main.tf` file to create the resource group where the resources for the remote state will be grouped:

    ```hcl
    # #############################################################################
    #                           Resource Group
    # #############################################################################
    
    resource "azurerm_resource_group" "rg" {
      name     = "${module.resource_group.name.abbreviation}-SwissArmyKnifeInfrastructure-${var.azure_environment}-${module.azure_regions.region.region_short}"
      location = module.azure_regions.region.region_cli
      tags     = local.tags
    }
    ```

11. Add the following configuration to the `main.tf` file to manage an Azure storage account:

    ```hcl
    # #############################################################################
    #                           Storage Account
    # #############################################################################
    
    resource "azurerm_storage_account" "st" {
      name                            = lower("${module.storage_account.name.abbreviation}Terraform${var.resource_name_suffix}${var.azure_environment}${module.azure_regions.region.region_short}")
      resource_group_name             = azurerm_resource_group.rg.name
      location                        = azurerm_resource_group.rg.location
      account_tier                    = "Standard"
      account_replication_type        = "LRS"
      allow_nested_items_to_be_public = false
    
      tags = local.tags
    }
    
    resource "azurerm_storage_container" "remote_state" {
      name                 = "terraform-state"
      storage_account_name = azurerm_storage_account.st.name
    }
    
    data "azurerm_storage_account_sas" "state" {
      connection_string = azurerm_storage_account.st.primary_connection_string
      https_only        = true
    
      resource_types {
        service   = true
        container = true
        object    = true
      }
    
      services {
        blob  = true
        queue = false
        table = false
        file  = false
      }
    
      start  = timestamp()
      expiry = timeadd(timestamp(), "17520h")
    
      permissions {
        read    = true
        write   = true
        delete  = true
        list    = true
        add     = true
        create  = true
        update  = false
        process = false
        tag     = false
        filter  = false
      }
    }
    ```

    > This storage account will be used to store the Terraform remote state.

12. Add the following to the `main.tf` file to create a tfconfig file:

    ```hcl
    # #############################################################################
    #                        Generate the TFConfig File
    # #############################################################################
    
    resource "local_file" "post-config" {
      depends_on = [azurerm_storage_container.remote_state]
    
      filename = "${path.module}/${var.azure_environment}.tfconfig"
      content  = <<EOF
    storage_account_name = "${azurerm_storage_account.st.name}"
    container_name = "terraform-state"
    key = "iac.tfstate"
    sas_token = "${data.azurerm_storage_account_sas.state.sas}"
    
      EOF
    }
    ```

    > [!NOTE]
    >
    > The `TFConfig` file extension is not a standard defined by Hashicorp, but something Chad Green came up with to help distinguish the file to be able to ignore it from git commits.

13. Add the following to the `main.tf` file to output the resource name suffix we will use in other work instructions:

    ```hcl
    # #############################################################################
    # Outputs
    # #############################################################################
    
    output "resource_name_suffix" {
      value = local.resource_suffix
    }
    ```

14. Save the `main.tf` file.

### Part 2: Ignore the TFConfig file

The `TFConfig` generated in part one contains secrets (the SAS token to access the managed storage account). As such, that file should not be checked into your Git repository.

1. Open the `.gitignore` file located in the root directory of your local repository folder.

2. Add the following to the file:

   ```yml
   # Ignore TFConfig files
   *.tfconfig
   
   # Ignore Terraform lock files
   .terraform.lock.hcl
   ```

### Part 3: Apply the Terraform configuration to Azure

1. Open a Terminal window to execute the Terraform configuration project.

   - In Visual Studio Code, click on the `Terminal` menu item and select the `New Terminal Window` option.

2. From the Terminal window you just opened, connect to your Azure account:

   ```sh
   az login
   ```

   Follow the prompts to log into your Azure Account and select the appropriate subscription (if you have multiple subscriptions).

3. Navigate to the `infra\remote-state` directory:

   ```sh
   cd infra\remote-state
   ```

4. Initialize the Terraform configuration project:

   ```sh
   terraform init
   ```

5. Validate that the Terraform configuration project is syntactically correct:

   ```sh
   terraform validate
   ```

   Correct any validation errors (there should not be any).

6. Plan the Terraform configuration project:

   ```sh
   terraform plan -var 'azure_environment=dev' -var 'azure_region=eastus2' -var 'resource_name_suffix=random'
   ```

   Three variables are being defined in the `terraform plan` command:

   - **azure_environment**: This is the system environment (i.e., dev, QA, stage, prod, etc.) you are deploying to. Multiple deployment environments are out of the scope of this workshop. Still, we are simulating deploying to `dev` since that would generally be the first environment to deploy to.
   - **azure_environment**: This is the name of the Azure region where Azure resources are deployed. When creating Azure resources, you should pick the region closest to the target users.
   - **resource_name_suffix**: This Terraform configuration project can use a random generator to help ensure the created resources are globally unique. Specifying `random` will ensure that a three-digit random number is added to the end of the resource names. You can also select anything else or leave the value blank; no suffix will be added.

   > [!NOTE] 
   >
   > The `azure_environment` and `resource_name_suffix` values together cannot be longer than **13** characters. Azure Storage accounts are limited to **24** characters, and the resource name's static part uses **11** characters.

   >  [!NOTE]  
   >
   >  While the `terraform validate` command performs a lint of the configuration, the terraform plan can find additional syntactical issues that must be resolved.

   Review the outputted plan to ensure what was expected to be built will be built.

7. Apply the Terraform configuration project to Azure:

   ```hcl
   terraform apply -var 'azure_environment=dev' -var 'azure_region=eastus2' -var 'resource_name_suffix=random' -auto-approve
   ```

   > [!Caution]
   >
   > The `terraform apply` command above uses the `-auto-apply` option, which means you will not be prompted to view and approve the plan before it is applied.

   After the apply is done, you should see the following at the end of the output:

   ```sh
   Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
   ```
   
   Make note of the `resource_name_suffix` output value as you will need this later.In

### Part 3: Migrate state to the remote state

At this point, we have created a repository within Azure to store the Terraform remote state. However, the state for configuring this repository is not in that remote state. This step will migrate the local state to the remote state.

1. In the `infra/remote-state` folder, create a file named `backend.tf` and add the following HCL:

   ```hcl
   terraform {
     backend "azurerm" {
     }
   }
   ```

2. Re-initialize the Terraform project. Enter **yes** when prompted, "Do you want to copy the existing state to the new backend?".

   ```hcl
   terraform init --backend-config=dev.tfconfig
   ```

### Step 4: Migrate GitHub Terraform state to the remote state

1. Navigate to the `infra\github` folder:

   ```sh
   cd ..\github
   ```

2. Create a file named `backend.tf` and add the following HCL:

   ```sh
   terraform {
     backend "azurerm" {
     }
   }
   ```

3. Re-initialize the Terraform project. Enter **yes** when prompted, "Do you want to copy existing state to the new backend?".

   ```sh
   terraform init --backend-config=dev.tfconfig
   ```

### Part 5: Push the remote state Terraform project to your central Git repository

1. Click on the `Source Control` tab within Visual Studio Code.

2. Add an appropriate commit message:

   ```
   Adding the Terraform remote state configuration project.
   ```

3. Click the **Commit** button.

4. Click the **Sync Changes** button.

## Conclusion

You have now used Terraform to create the Azure resources needed to implement Terraform's remote state.

## Next Steps

In the next work instruction, we will setup a GitHub Action to automatically apply solution configuration changes to Azure.