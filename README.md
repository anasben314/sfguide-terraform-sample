# Overview
Terraform is an open-source Infrastructure as Code (IaC) tool created by HashiCorp. It is a declarative Infrastructure as Code tool, meaning instead of writing step-by-step imperative instructions like with SQL, JavaScript, or Python, you can declare what you want using a YAML-like syntax. Terraform is also stateful, meaning it keeps track of your current state and compares it with your desired state. A reconciliation process between the two states generates a plan that Terraform can execute to create new resources or update/delete existing resources. This plan is implemented as an acyclic graph, which is what allows Terraform to understand and handle dependencies between resources. Using Terraform is a great way to manage account-level Snowflake resources like Warehouses, Databases, Schemas, Tables, and Roles/Grants, among many other use cases.

A Terraform provider is available for Snowflake, allowing Terraform to integrate with Snowflake.

Example Terraform use-cases:

- Set up storage in your cloud provider and add it to Snowflake as an external stage
- Add storage and connect it to Snowpipe
- Create a service user and push the key into the secrets manager of your choice or rotate keys

Many Snowflake customers use Terraform to comply with security controls, maintain consistency, and support similar engineering workflows for infrastructure at scale.

This tutorial demonstrates how you can use Terraform to manage your Snowflake configurations in an automated and source-controlled way. We show you how to install and use Terraform to create and manage your Snowflake environment, including how to create a database, schema, warehouse, multiple roles, and a service user.

This introductory tutorial does not cover specific cloud providers or how to integrate Terraform into specific CI/CD tools. Those topics may be covered in future labs.

## Prerequisites
- Familiarity with Git, Snowflake, and Snowflake objects
- ACCOUNTADMIN role access to a Snowflake account

## What You'll Need
- A GitHub account
- A git command-line client
- Text editor of your choice
- Terraform installed

## What You'll Build
- A repository containing a Terraform project that manages Snowflake account objects through code

## [Create a Service User for Terraform](create_tf_user.md)
## [Setup Terraform Authentication](setup_terraform_authentication.md)
## Declaring Resources
Add a file to your project in the base directory named main.tf. In main.tf we set up the provider and define the configuration for the database and the warehouse that we want Terraform to create.

Copy the contents of the following block to your *main.tf*

    terraform {
      required_providers {
        snowflake = {
          source  = "Snowflake-Labs/snowflake"
          version = "~> 0.76"
        }
      }
    }
    
    provider "snowflake" {
      role = "SYSADMIN"
    }
    
    resource "snowflake_database" "db" {
      name = "TF_DEMO"
    }
    
    resource "snowflake_warehouse" "warehouse" {
      name           = "TF_DEMO"
      warehouse_size = "large"
      auto_suspend   = 60
    }

This is all the code needed to create these resources.

## Preparing the Project to Run
To set up the project to run Terraform, you first need to initialize the project.

Run the following from a shell in your project folder:

    $ terraform init

The dependencies needed to run Terraform are downloaded to your computer.

In this demo, we use a local backend for Terraform, which means that state file are stored locally on the machine running Terraform. Because Terraform state files are required to calculate all changes, its critical that you do not lose this file (*.tfstate, and *.tfstate.* for older state files), or Terraform will lose track of what resources it is managing. For any kind of multi-tenancy or automation workflows, it is highly recommended to use Remote Backends, which is outside the scope of this lab.

The .terraform folder is where all provider and modules depenencies are downloaded. There is nothing important in this folder and it is safe to delete. You should add this folder, and the Terraform state files (since you do not want to be checking in sensitive information to VCS) to .gitignore.

### Create a file named .gitignore in your project root, then add the following text to the file and save it:

    *.terraform*
    *.tfstate
    *.tfstate.*
    
## Changing and Adding Resources

### These are the commands we want to run in snowflake

    ```sh
    # Add Required Provider
    terraform init
    
    # Create Snowflake Database
    CREATE DATABASE TF_DEMO;
    
    # Create Snowflake Warehouse
    CREATE WAREHOUSE TF_DEMO WAREHOUSE_SIZE='SMALL' AUTO_SUSPEND=60;
    
    # Create Snowflake Role
    CREATE ROLE TF_DEMO_SVC_ROLE;
    
    # Grant Privileges to Role on Database
    GRANT USAGE ON DATABASE TF_DEMO TO ROLE TF_DEMO_SVC_ROLE;
    
    # Create Snowflake Schema
    CREATE SCHEMA TF_DEMO;
    
    # Grant Privileges to Role on Schema
    GRANT USAGE ON SCHEMA TF_DEMO TO ROLE TF_DEMO_SVC_ROLE;
    
    # Grant Privileges to Role on Warehouse
    GRANT USAGE ON WAREHOUSE TF_DEMO TO ROLE TF_DEMO_SVC_ROLE;
    
    # Generate RSA Key Pair (Use a tool or library to generate an RSA key pair with 2048 bits)
    
    # Create Snowflake User
    CREATE USER tf_demo_user DEFAULT_WAREHOUSE = TF_DEMO DEFAULT_ROLE = TF_DEMO_SVC_ROLE DEFAULT_NAMESPACE = TF_DEMO.TF_DEMO RSA_PUBLIC_KEY = 'your_rsa_public_key_here';
    
    # Grant MONITOR Privileges to Role on User
    GRANT MONITOR ON USER tf_demo_user TO ROLE TF_DEMO_SVC_ROLE;
    
    # Grant Role to User
    GRANT ROLE TF_DEMO_SVC_ROLE TO USER tf_demo_user;

### Terrafrom equivalent

All databases need a schema to store tables, so we'll add that and a service user so that our application/client can connect to the database and schema. The syntax is very similar to the database and warehouse you already created. By now you have learned everything you need to know to create and update resources in Snowflake. We'll also add privileges so the service role/user can use the database and schema.

You'll see that most of this is what you would expect. The only new part is creating the private key. Because the Terraform TLS private key generator doesn't export the public key in a format that the Terraform provider can consume, some string manipulations are needed.

Change the warehouse size in the main.tf file from xsmall to small.
Add the following resources to your *main.tf* file:

    provider "snowflake" {
      alias = "security_admin"
      role  = "SECURITYADMIN"
    }
    
    resource "snowflake_role" "role" {
      provider = snowflake.security_admin
      name     = "TF_DEMO_SVC_ROLE"
    }
    
    resource "snowflake_grant_privileges_to_role" "database_grant" {
      provider   = snowflake.security_admin
      privileges = ["USAGE"]
      role_name  = snowflake_role.role.name
      on_account_object {
        object_type = "DATABASE"
        object_name = snowflake_database.db.name
      }
    }
    
    resource "snowflake_schema" "schema" {
      database   = snowflake_database.db.name
      name       = "TF_DEMO"
      is_managed = false
    }
    
    resource "snowflake_grant_privileges_to_role" "schema_grant" {
      provider   = snowflake.security_admin
      privileges = ["USAGE"]
      role_name  = snowflake_role.role.name
      on_schema {
        schema_name = "\"${snowflake_database.db.name}\".\"${snowflake_schema.schema.name}\""
      }
    }
    
    resource "snowflake_grant_privileges_to_role" "warehouse_grant" {
      provider   = snowflake.security_admin
      privileges = ["USAGE"]
      role_name  = snowflake_role.role.name
      on_account_object {
        object_type = "WAREHOUSE"
        object_name = snowflake_warehouse.warehouse.name
      }
    }
    
    resource "tls_private_key" "svc_key" {
      algorithm = "RSA"
      rsa_bits  = 2048
    }
    
    resource "snowflake_user" "user" {
        provider          = snowflake.security_admin
        name              = "tf_demo_user"
        default_warehouse = snowflake_warehouse.warehouse.name
        default_role      = snowflake_role.role.name
        default_namespace = "${snowflake_database.db.name}.${snowflake_schema.schema.name}"
        rsa_public_key    = substr(tls_private_key.svc_key.public_key_pem, 27, 398)
    }
    
    resource "snowflake_grant_privileges_to_role" "user_grant" {
      provider   = snowflake.security_admin
      privileges = ["MONITOR"]
      role_name  = snowflake_role.role.name
      on_account_object {
        object_type = "USER"
        object_name = snowflake_user.user.name
      }
    }
    
    resource "snowflake_role_grants" "grants" {
      provider  = snowflake.security_admin
      role_name = snowflake_role.role.name
      users     = [snowflake_user.user.name]
    }

To get the public and private key information for the application, use Terraform output values.
Add the following resources to a new file named *outputs.tf*
    output "snowflake_svc_public_key" {
        value = tls_private_key.svc_key.public_key_pem
    }
    
    output "snowflake_svc_private_key" {
        value     = tls_private_key.svc_key.private_key_pem
        sensitive = true
    }

Replace `'your_rsa_public_key_here'` with the actual RSA public key generated. 
