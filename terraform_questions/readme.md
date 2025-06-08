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
   ```bash
   gsutil ls -a gs://terraform-state-bucket/terraform.tfstate*
2. If a backup exists, restore it:
```
gsutil cp gs://terraform-state-bucket/terraform.tfstate#1234567890 ./terraform.tfstate
```
3. If no backup is available, create a new empty state file:
```
terraform init
```
4. Use gcloud commands to inventory existing resources:
``
# List Compute Instances
gcloud compute instances list --project=my-project

# List Cloud SQL instances
gcloud sql instances list --project=my-project
``
5. Systematically import existing resources back into the state file:
``
# Import a Compute Instance
terraform import google_compute_instance.web_server projects/my-project/zones/us-central1-a/instances/web-server-01

# Import a Cloud SQL instance
terraform import google_sql_database_instance.main my-project:my-sql-instance
``
6. After recovery, implement proper state file backup procedures by configuring a GCS backend with versioning:
```
terraform {
  backend "gcs" {
    bucket  = "terraform-state-bucket"
    prefix  = "terraform/state"
  }
}
```
