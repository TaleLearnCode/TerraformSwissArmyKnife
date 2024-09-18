# Work Instruction 1: Initializing Solution

## Objective

We normally only think about Terraform for cloud resources in AWS, Azure, or Google Cloud Platform, but Terraform can be used to manage all sorts of infrastructure and services to include GitHub repositories.

In this work instruction, you will prepare your demonstration environment by setting up the GitHub repository for the demo. 

## Prerequisites

- Git is installed on your local machine.
- A GitHub account.
- Visual Studio Code installed on your local machine.

## Steps

### Part 1: Create a GitHub Token

You will need a GitHub Token to allow the Terraform project to manage your GitHub resources.

1. Go to [GitHub](https://github.com) and log in to your account.
2. Click your profile photo in the upper-right corner of any GitHub page, then click **Settings**.
3. In the left sidebar, click **Developer settings**.
4. Click **Personal access tokens** > **Tokens (classic)** in the left sidebar.
5. Click **Generate new token** > **Generate new token (classic)**.
6. Configure the token:
   - Give your token a descriptive name in the **Note** field (e.g., `Terraform GitHub Token`).
   - Set an expiration date for the token. Choose a duration that matches your security policies.
   - Select the scopes (permissions) that your token needs. For managing repositories with Terraform, you will need
     - `repo` (Full control of private repositories)
     - `workflow` (Update GitHub Actions workflows)
     - `admin:repo_hook` (Full control of repository hooks)
     - `delete_repo` (Delete repositories)
7. Scroll to the bottom of the page and click **Generate token**.
8. Copy the generated token now. Once you leave the page, you will not be able to see it again.

### Part 2: Create the GitHub Repository using Terraform

1. From the folder on your machine where you keep your local repositories (e.g., `C: Repos`), create a new directory for your Terraform project and navigate to it.

   ```sh
   mkdir terraform-swiss-army-knife-demo
   cd terraform-swiss-army-knife-demo
   
   mkdir infra
   cd infra
   
   mkdir github
   cd github
   ```

   > You can name the repository folder whatever makes sense for you. However, it is highly recommended that you keep the `infra\github` folder structure name within your repository folder.

2. Define the variables to the Terraform project by creating a file named `variables.tf` and add the following configuration:

   ```hcl
   variable "github_token" {
     type        = string
     sensitive   = true
     description = "The GitHub token with repo and admin:repo_hook permissions"
   }
   
   variable "github_repository_name" {
     type        = string
     default     = "swissarmy-ecommerce-app"
     description = "The name of the Azure Static Web App GitHub repository."
    }
   
    variable "github_repository_description" {
     type = string
     default = "Swiss Army Knife E-Commerce App"
     description = "The description of the Azure Static Web App GitHub repository."
    }
   ```

3. Define the providers for the Terraform project by creating a file named `providers.tf` and add the following configuration:

   ```hcl
   terraform {
     required_providers {
       github = {
         source  = "integrations/github"
         version = "~> 6.0"
       }
     }
   }
   
   provider "github" {
     token = var.github_token
   }
   ```

4. Azure Static Web App GitHub repository nameDefine the resources to be managed by the Terraform GitHub project by creating a file named `main.tf` and add the following configuration:

   ```hcl
   resource "github_repository" "ecommerce_repo" {
     name               = var.github_repository_name
     description        = var.github_repository_description
     visibility         = "public"
     gitignore_template = "Terraform"
     license_template   = "mit"
     auto_init          = true
   }
   
   resource "github_branch" "develop" {
     repository = github_repository.ecommerce_repo.name
     branch     = "develop"
   }
   
   resource "github_branch_default" "default"{
     repository = github_repository.ecommerce_repo.name
     branch     = github_branch.develop.branch
   }
   ```

5. Define the parameter inputs (variable values) by creating a file named `dev.tfvars` and add the following:

   ```hcl
   github_token = "<<Token copied in Part 1>>"
   ```

   

6. Initialize the Terraform configuration:

   ```sh
   terraform init
   ```

   You should receive a response: "**Terraform has been successfully installed!**" If not, correct any errors and rerun the `terraform init`.

7. Apply the Terraform configuration to create the GitHub repository:

   ```hcl
   terraform apply --var-file=dev.tfvars
   ```

   Terraform will generate a plan of what it needs to do. You should see a message indicating `Plan: 3 to add, 0 to change, and 0 to destroy`.

   Review the plan and then confirm the action by typing `yes` when prompted.

   After the application process, you will see the message `Apply complete! Resources: 3 added, 0 changed, 0 destroyed.` This will create the repository named `swissarmy-ecommerce-app` on your GitHub account with a `.gitignore` template for Terraform, an MIT license, and an initial commit.

### Part 3: Pushing the Terraform Project into the GitHub Repository

1. Navigate to the root directory of your local repository:

   ```sh
   cd ../..
   ```

2. Initialize the Git Repository locally:

   ```sh
   git init -b develop
   ```

3. Navigate back to the GitHub Terraform project directory:

   ```sh
   cd infra/github
   ```

4. Add the Terraform configuration files to the Git repository:

   ```sh
   git add variables.tf
   git add providers.tf
   git add main.tf
   ```

   > Do not perform a `git add .` as it will commit files that you do not want and would normally be ignored if the .gitignore file was present.

5. Commit the added files with an appropriate commit message:

   ```sh
   git commit -m "Initial commit with Terraform configuration for GitHub repository"
   ```

6. Link the local repository to GitHub:

   ```sh
   git remote add origin <<repository_url>>
   ```

   Replace `<<repository_url>>` with the actual URL of your GitHub repository.

7. Push the committed changes to the remote repository:

   ```sh
   git push -u origin develop
   ```

8. Pull the remote repository:

   ```sh
   git pull origin develop --allow-unrelated-histories
   ```

9. Push the merged repository:

   ```sh
   git push -u origin develop
   ```

   

## Summary

You have successfully created a GitHub repository using Terraform with the specified settings and pushed your Terraform project into that repository. You can verify the repository and its contents on your GitHub account.