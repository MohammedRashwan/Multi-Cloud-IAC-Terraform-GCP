GCP Infrastructure as Code (IaC) with Terraform
This project demonstrates how to use Terraform to provision and manage infrastructure on Google Cloud Platform (GCP). It includes a Google Compute Engine VM instance, a VPC network with a subnet and firewall rules, and a Google Cloud Storage bucket.

Project Overview
The Terraform code provided in this repository automates the creation of the following GCP resources:

Google Compute Engine VM Instance: A virtual machine running Debian Linux, with Apache installed.
VPC Network: A custom virtual private cloud (VPC) network with a subnet and firewall allowing HTTP/HTTPS traffic.
Google Cloud Storage Bucket: A storage bucket used for storing files.
Project Structure
graphql
Copy code
gcp-terraform-iac/
├── main.tf                 # Root module for calling all other modules
├── providers.tf            # GCP provider configuration
├── variables.tf            # Input variables for the root module
├── outputs.tf              # Output definitions for the root module
├── modules/                # Submodules for different GCP resources
│   ├── compute/            # Module for Compute Engine VM
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── network/            # Module for VPC network
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── storage/            # Module for Cloud Storage bucket
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── README.md               # Documentation (this file)
Requirements
Terraform (v0.13+)
GCP account with an active project
Service account with appropriate permissions
Service account key
Setup and Usage
1. Clone the Repository
bash
Copy code
git clone https://github.com/yourusername/gcp-terraform-iac.git
cd gcp-terraform-iac
2. Configure Google Cloud Authentication
Set the path to your GCP service account key file and define the project ID:

bash
Copy code
export GOOGLE_CLOUD_KEYFILE_JSON="<path_to_your_service_account_key>.json"
export TF_VAR_project_id="your-gcp-project-id"
3. Initialize Terraform
Run the following command to initialize Terraform, which downloads the required providers and sets up your backend:

bash
Copy code
terraform init
4. Plan the Infrastructure
Preview the changes Terraform will apply by running:

bash
Copy code
terraform plan
5. Apply the Changes
Deploy the infrastructure by running:

bash
Copy code
terraform apply -auto-approve
6. Destroy the Infrastructure
To clean up and remove all the resources that were created, run:

bash
Copy code
terraform destroy -auto-approve
Terraform Code Overview
main.tf
This file contains the core Terraform configuration, which calls individual modules for compute, networking, and storage.

hcl
Copy code
module "network" {
  source  = "./modules/network"
  vpc_name = var.vpc_name
}

module "compute" {
  source       = "./modules/compute"
  machine_type = var.machine_type
  network      = module.network.network_name
}

module "storage" {
  source      = "./modules/storage"
  bucket_name = var.bucket_name
}

terraform {
  backend "gcs" {
    bucket  = "my-terraform-state"
    prefix  = "terraform/state"
    project = var.project_id
  }
}
providers.tf
Defines the Google Cloud provider configuration.

hcl
Copy code
provider "google" {
  credentials = file("<path_to_your_service_account_key>.json")
  project     = var.project_id
  region      = var.region
}
variables.tf
Defines the input variables for the project, such as the project ID, region, VPC name, machine type, and bucket name.

hcl
Copy code
variable "project_id" {
  type        = string
  description = "The GCP project ID"
}

variable "region" {
  type        = string
  description = "The GCP region"
  default     = "us-central1"
}

variable "vpc_name" {
  type        = string
  description = "VPC network name"
  default     = "my-vpc"
}

variable "machine_type" {
  type        = string
  description = "GCP machine type for VM instance"
  default     = "e2-medium"
}

variable "bucket_name" {
  type        = string
  description = "Name of the GCP Storage bucket"
}
outputs.tf
Defines the outputs that Terraform will provide after applying the infrastructure, such as the VM instance IP and Cloud Storage bucket URL.

hcl
Copy code
output "instance_ip" {
  value = google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip
}

output "bucket_url" {
  value = google_storage_bucket.bucket.url
}
Modules Overview
Compute Module (modules/compute/)
This module provisions a Google Compute Engine VM instance with Apache installed and configured to run on HTTP.

main.tf
hcl
Copy code
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = var.machine_type
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = var.network
    access_config {
      // Ephemeral public IP
    }
  }

  metadata_startup_script = <<-EOT
    #! /bin/bash
    sudo apt-get update
    sudo apt-get install -y apache2
    sudo systemctl start apache2
  EOT
}
variables.tf
hcl
Copy code
variable "machine_type" {
  type        = string
  description = "Machine type for the GCP VM instance"
}

variable "network" {
  type        = string
  description = "VPC network name"
}
Network Module (modules/network/)
This module provisions a VPC network, a subnet, and firewall rules to allow HTTP/HTTPS traffic.

main.tf
hcl
Copy code
resource "google_compute_network" "vpc_network" {
  name = var.vpc_name
}

resource "google_compute_subnetwork" "subnet" {
  name          = "subnet"
  ip_cidr_range = "10.0.0.0/24"
  network       = google_compute_network.vpc_network.name
}

resource "google_compute_firewall" "firewall" {
  name    = "allow-http"
  network = google_compute_network.vpc_network.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
}
variables.tf
hcl
Copy code
variable "vpc_name" {
  type        = string
  description = "Name of the VPC network"
}
Storage Module (modules/storage/)
This module provisions a Google Cloud Storage bucket.

main.tf
hcl
Copy code
resource "google_storage_bucket" "bucket" {
  name     = var.bucket_name
  location = "US"
}
variables.tf
hcl
Copy code
variable "bucket_name" {
  type        = string
  description = "Name of the storage bucket"
}
Optional: GitHub Actions for CI/CD
To automate Terraform operations, you can use the following GitHub Actions workflow to perform Terraform init, plan, and apply on every push to the main branch.

Add this file at .github/workflows/terraform.yml:

yaml
Copy code
name: Terraform

on:
  push:
    branches:
      - main

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan

    - name: Terraform Apply
      run: terraform apply -auto-approve
Conclusion
This project provides a simple yet effective demonstration of how to use Terraform for managing Google Cloud infrastructure. You can extend it further by adding more modules or services like Google Cloud SQL, Google Kubernetes Engine, or Pub/Sub.

Feel free to fork this repository and experiment with your own GCP setups!

