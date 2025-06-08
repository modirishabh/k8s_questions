# Top 10 Terraform Scenario-Based Questions and Answers for GCP

This document provides solutions to common Terraform scenarios when working with Google Cloud Platform (GCP).

## Table of Contents
1. [Managing Infrastructure Changes](#1-managing-infrastructure-changes)
2. [Handling State File Corruption](#2-handling-state-file-corruption)
3. [Multi-Environment Deployment](#3-multi-environment-deployment)
4. [Implementing CI/CD with Terraform](#4-implementing-cicd-with-terraform)
5. [Managing Large-Scale Infrastructure](#5-managing-large-scale-infrastructure)
6. [Handling Provider API Rate Limiting](#6-handling-provider-api-rate-limiting)
7. [Implementing Custom Validation](#7-implementing-custom-validation)
8. [Migrating Existing Infrastructure](#8-migrating-existing-infrastructure)

---

## 1. Managing Infrastructure Changes

**Scenario:**  
Your team has made manual changes to a GCP production environment that was originally deployed using Terraform. Now Terraform plan shows many resources will be destroyed and recreated, which would cause downtime.

**Question:**  
How would you handle this situation to minimize disruption while bringing the infrastructure back under Terraform management?

**Answer:**  
To handle this scenario with minimal disruption in GCP:
1. First, use `terraform refresh` to update the state file with the current infrastructure state.
2. For resources that have drifted, use terraform import to bring them back under Terraform management without recreation:
```
# Example: Importing a GCP Compute Instance
terraform import google_compute_instance.app_server projects/my-project/zones/us-central1-a/instances/app-server-01
```
3. For resources that cannot be imported easily, I would:
- Use `gcloud` commands to document the current configuration.
- Update the Terraform code to match the current state.
- Use the `terraform state rm` command to remove resources from state that need special handling.
4. Apply changes gradually in maintenance windows if needed.
5. Implement IAM roles that prevent direct console changes to enforce infrastructure as code practices.
6. Consider using Google Cloud's Config Connector as a complementary tool to help reconcile existing resources.

---

## 2. Handling State File Corruption

**Scenario:**  
Your team discovers that the Terraform state file is corrupted or has been accidentally deleted, but the GCP infrastructure is still running in production.

**Question:**  
What steps would you take to recover from this situation?

**Answer:**  
To recover from a corrupted or deleted state file in GCP:

1. First, check if state file backups exist (if using a GCS backend with versioning enabled):
```
   gsutil ls -a gs://terraform-state-bucket/terraform.tfstate*
```

2. If a backup exists, restore it:
```
gsutil cp gs://terraform-state-bucket/terraform.tfstate#1234567890 ./terraform.tfstate
```

3. If no backup is available, create a new empty state file:
```
terraform init
```

4. Use gcloud commands to inventory existing resources:
```
gcloud compute instances list --project=my-project # List Compute Instances

gcloud sql instances list --project=my-project # List Cloud SQL instances

```

5. Systematically import existing resources back into the state file:

```

terraform import google_compute_instance.web_server projects/my-project/zones/us-central1-a/instances/web-server-01 # Import a Compute Instance

terraform import google_sql_database_instance.main my-project:my-sql-instance # Import a Cloud SQL instance


```

6. After recovery, implement proper state file backup procedures by configuring a GCS backend with versioning:
```
terraform {
  backend "gcs" {
    bucket  = "terraform-state-bucket"
    prefix  = "terraform/state"
  }
}
```

## 3. Multi-Environment Deployment

**Scenario:**  
You need to manage deployments of your infrastructure across multiple environments (development, staging, production) in GCP, with slight variations in configuration for each environment.

**Question:**  
How would you structure your Terraform configurations to accommodate multiple environments, and how would you switch between them?

**Answer:**  
To manage multiple environments in Terraform, I would:
1. I would use one of these approaches depending on the organization's needs: a. Workspaces approach.
```
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
```
With conditional configuration:
```
locals {
  env_config = {
    dev     = { machine_type = "e2-small", instance_count = 1, project = "my-dev-project" }
    staging = { machine_type = "e2-medium", instance_count = 2, project = "my-staging-project" }
    prod    = { machine_type = "e2-standard-4", instance_count = 5, project = "my-prod-project" }
  }
  config = local.env_config[terraform.workspace]
}

provider "google" {
  project = local.config.project
  region  = "us-central1"
}
```
b. Directory structure approach with separate GCP projects:
```
├── modules/
│   ├── vpc/
│   ├── gke/
│   └── cloud_sql/
├── environments/
│   ├── dev/
│   │   └── terraform.tfvars   # dev-project settings
│   ├── staging/
│   │   └── terraform.tfvars   # staging-project settings
│   └── prod/
│       └── terraform.tfvars   # prod-project settings
```
2. Use environment-specific variable files with GCP project details:
```
terraform apply -var-file=environments/prod/terraform.tfvars
```

3. Implement a CI/CD pipeline that applies the correct configuration for each environment, using separate service accounts for each environment.
4. Use remote state with path separation for each environment:
```
terraform {
  backend "gcs" {
    bucket = "terraform-state-bucket"
    prefix = "terraform/state/prod"  # Different prefix per environment
  }
}
```
---

## 4. Implementing CI/CD with Terraform

**Scenario:**  
You are tasked with setting up a CI/CD pipeline for your Terraform code to manage GCP resources, ensuring that changes are tested and applied automatically.

**Question:**  
What steps and best practices would you follow to set up a robust CI/CD pipeline for your Terraform configurations?

**Answer:**  
In setting up a CI/CD pipeline for Terraform, I would:
- Use a version control system such as Git to manage the Terraform code.
- Configure a CI/CD platform like Jenkins, GitLab CI, GitHub Actions, or Google Cloud Build to run pipeline jobs.
- Define pipeline stages for linting (e.g., `terraform fmt`), validation (`terraform validate`), planning (`terraform plan`), and applying (`terraform apply`) the Terraform code.
- Ensure that the Terraform state is stored in a remote backend such as GCS, with state locking enabled.
- Implement manual approval steps before applying changes, especially for production.
- Securely manage secrets and credentials using a secret management solution like GCP Secret Manager.
- Integrate automated testing and policy enforcement using tools like OPA (Open Policy Agent) or Terraform Sentinel.

---

## 5. Managing Large-Scale Infrastructure

**Scenario:**  
Your Terraform configurations have grown in complexity as you manage a large-scale infrastructure on GCP. This has led to longer plan and apply times and difficulty in understanding the impact of changes.

**Question:**  
How would you optimize and manage large-scale Terraform configurations to improve performance and maintainability?

**Answer:**  
To manage large-scale Terraform configurations, I would:
1. Break down monolithic state:
Split infrastructure into logical components based on GCP services (networking, compute, GKE, Cloud SQL, etc.)
Use separate state files for different components:
```
# networking/backend.tf
terraform {
  backend "gcs" {
    bucket = "terraform-state-bucket"
    prefix = "terraform/networking"
  }
}

# compute/backend.tf
terraform {
  backend "gcs" {
    bucket = "terraform-state-bucket"
    prefix = "terraform/compute"
  }
}
```
2. Implement a module strategy:
Create reusable modules for common GCP patterns:

```
modules/
├── gcp-vpc/
├── gcp-gke/
├── gcp-cloud-sql/
└── gcp-load-balancer/
```
Version modules using Git tags or Terraform Registry
Use the GCP Architecture Center reference architectures as inspiration
3. State management across components:
Use remote state with references between components:
```
data "terraform_remote_state" "network" {
  backend = "gcs"
  config = {
    bucket = "terraform-state-bucket"
    prefix = "terraform/networking"
  }
}

resource "google_compute_instance" "app_server" {
  network = data.terraform_remote_state.network.outputs.network_self_link
}
```
4. Optimize for performance:
Use -target flag for specific resource updates
Implement parallel execution where possible
Consider using Terraform Cloud's remote operations

5. Testing and validation:
Use the GCP Config Validator for policy enforcement
Implement automated testing with tools like Kitchen-Terraform
Create a test GCP project for infrastructure testing

---

## 6. Handling Provider API Rate Limiting

**Scenario:**  
While managing a large number of GCP resources, you encounter API rate limiting errors that cause your Terraform apply to fail intermittently.

**Question:**  
What strategies would you use to mitigate API rate limiting issues when using Terraform with GCP?

**Answer:**  
To mitigate API rate limiting issues in Terraform with GCP:
- Use the `provider` block to configure retry logic and backoff settings for the GCP provider.
```
provider "google" {
  project = "my-project"
  region  = "us-central1"
  
  # Configure request retry behavior
  request_timeout = "60s"
  batching {
    enable_batching = true
    send_after = "2s"
  }
}
```
- Stagger Terraform `apply` operations by using the `-parallelism` flag to limit the number of concurrent operations.
- Break up large sets of resource changes into smaller batches to reduce the number of API calls made in a short time.
```
terraform apply -target=module.networking
terraform apply -target=module.compute
terraform apply -target=module.database
```
- Implement exponential backoff in your CI/CD pipeline scripting to retry failed operations with increasing delays.
```
 Use a wrapper script for terraform apply
MAX_RETRIES=5
RETRY_DELAY=10

for i in $(seq 1 $MAX_RETRIES); do
  terraform apply -auto-approve
  if [ $? -eq 0 ]; then
    echo "Apply succeeded"
    exit 0
  fi
  
  echo "Apply failed, retrying in $RETRY_DELAY seconds..."
  sleep $((RETRY_DELAY * i))
done

echo "Apply failed after $MAX_RETRIES retries"
exit 1
```
- Contact GCP support to request a rate limit increase quotas if your legitimate use case exceeds the default limits.
- Monitor API usage and rate limit errors using GCP monitoring tools to better understand and manage your API consumption.

---

## 7. Implementing Custom Validation

**Scenario:**  
You need to ensure that Terraform configurations adhere to certain custom rules and constraints specific to your organization's policies for GCP resources.

**Question:**  
How can you introduce custom validation into your Terraform workflow to enforce these organizational policies?

**Answer:**  
To introduce custom validation into Terraform:
- Use Terraform's `validation` block within variable definitions to enforce rules on the input values.
```
variable "machine_type" {
  description = "GCP machine type for the instance"
  type        = string
  
  validation {
    condition     = can(regex("^(e2|n2|c2)-", var.machine_type))
    error_message = "Machine type must be from the e2, n2, or c2 family for compliance reasons."
  }
}

variable "disk_size_gb" {
  description = "Boot disk size in GB"
  type        = number
  
  validation {
    condition     = var.disk_size_gb >= 20 && var.disk_size_gb <= 100
    error_message = "Disk size must be between 20 and 100 GB."
  }
}
```
- Write custom policy definitions using a policy-as-code tool like OPA (Open Policy Agent) and integrate policy checks into your CI/CD pipeline.
```
Use Policy-as-Code tools such as:
Open Policy Agent (OPA) with Conftest
Google Cloud's Policy Intelligence tools
Terraform Sentinel policies (if using Terraform Cloud)
```
- Utilize Terraform's `external` data source to run scripts that perform custom validation logic.
- Integrate pre-commit hooks in your version control system to check for policy compliance before code is even committed.
```
  #!/bin/bash
# pre-commit hook for GCP compliance validation

# Check for public buckets
if grep -r "public_access_prevention = \"inherited\"" --include="*.tf" .; then
  echo "ERROR: Public buckets are not allowed. Set public_access_prevention to 'enforced'"
  exit 1
fi

# Ensure all projects have labels
if grep -r "resource \"google_project\"" --include="*.tf" . | grep -v "labels"; then
  echo "ERROR: All projects must have labels"
  exit 1
fi
```
- Regularly review and update your validation rules and policies to adapt to changes in organizational requirements or GCP features.

---

## 8. Migrating Existing Infrastructure

**Scenario:**  
You have existing GCP resources that were created manually or using another infrastructure-as-code tool, and you want to bring them under Terraform management without causing any service disruptions.

**Question:**  
What process would you follow to safely migrate the existing GCP resources into Terraform?

**Answer:**  
To migrate existing GCP resources to Terraform:
- Perform an inventory of your existing GCP resources using the GCP Console, CLI, or APIs.
- Create Terraform configurations that represent the current state of your resources.
- Use `terraform import` to map existing resources to your Terraform configurations.
- Gradually import resources into Terraform management, starting with less critical resources and testing thoroughly before proceeding with more critical ones.
- Use Terraform's `plan` to ensure that the imported resources match the expected state and no unintended changes will occur.
- Establish a clear rollback plan in case any issues arise during the migration process.
- Document the migration process and any discrepancies found for future reference and possible process improvements.

---
