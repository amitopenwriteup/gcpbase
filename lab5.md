# GCP Compute Instance – Terraform Setup Guide

## Overview

This guide documents the steps to provision a **Google Cloud Compute Instance** using Terraform with a **GCS remote backend** for state management. The instance network details are fetched via **Terraform Remote State** from a previously deployed network module.

---

## Prerequisites

- Terraform installed (`>= 1.0`)
- GCP Service Account JSON key file
- GCS bucket created for Terraform state
- Network remote state already deployed at prefix `student.01-network-state`

---

## Step 1: Clone the Repository

```bash
cd ..
git clone https://github.com/amitopenwriteup/gcpinstance.git
cd gcpinstance
```

---

## Step 2: Copy Provider Configuration

Copy the shared provider configuration from the system folder:

```bash
cp ../system/provider.tf .
```

---

## Step 3: Configure `backend.tf`

Update the GCS backend with your credentials and bucket name:

```hcl
terraform {
  backend "gcs" {
    credentials = "location of json file"   # Path to your GCP service account JSON
    bucket      = "name of bucket"          # Your GCS bucket name
    prefix      = "instance"
  }
}
```

---

## Step 4: Configure `instance.tf`

Update credentials, bucket name, and instance details:

```hcl
data "terraform_remote_state" "network_details" {
  backend = "gcs"
  config = {
    bucket = "provide your bucket name"          # GCS bucket name
    prefix = "networking"        # Path to the remote state file
  }
}

resource "google_compute_instance" "my_vm" {
  name         = "amit-instance"
  machine_type = "e2-micro"
  zone         = "us-east1-b"

  boot_disk {
    initialize_params {
      image = "projects/ubuntu-os-cloud/global/images/4751156868206452390"
    }
  }

  network_interface {
    network    = data.terraform_remote_state.network_details.outputs.network_name
    subnetwork = data.terraform_remote_state.network_details.outputs.subnetwork_name
    access_config {}
  }

  tags = ["amit-instance"]
}

output "instance_public_ip" {
  value       = google_compute_instance.my_vm.network_interface[0].access_config[0].nat_ip
  description = "The public IP address of the instance"
}
```

---

## Step 5: Initialize and Apply Terraform

```bash
terraform init
terraform apply
```

- `terraform init` initializes the backend and downloads required providers.
- `terraform apply` provisions the compute instance on GCP.

---

## Step 6: Git Commit and Push

After making all changes, push the configuration to GitHub:

```bash
git add .
git commit -m "Add GCP compute instance Terraform configuration"
git push origin main
```

---

## File Structure

```
gcpinstance/
├── provider.tf        # GCP provider configuration (copied from system/)
├── backend.tf         # GCS remote backend configuration
└── instance.tf        # Compute instance + remote state data source
```

---

## Outputs

| Output Name          | Description                          |
|----------------------|--------------------------------------|
| `instance_public_ip` | The public NAT IP of the VM instance |

---

## Notes

- The instance is deployed in zone **`us-east1-b`** with machine type **`e2-micro`** (free-tier eligible).
- Network and subnetwork values are dynamically fetched from the **network module's remote state** — ensure the network stack is deployed before applying this module.
- The boot image used is **Ubuntu** (image ID: `4751156868206452390`).
