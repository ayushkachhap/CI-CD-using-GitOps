[README.md](https://github.com/user-attachments/files/31028690/README.md)
# Secure-by-Default Deployment Architecture

![Build Status](https://img.shields.io/badge/build-passing-brightgreen)
![Version](https://img.shields.io/badge/version-1.0.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)

## Description

This project demonstrates an enterprise-grade, "secure-by-default" deployment architecture. By combining **declarative automation** (Terraform) with **Policy as Code (PaC)**, it establishes a fail-proof methodology for cloud infrastructure. GitHub serves as the single source of truth for the CI/CD pipeline, ensuring that all code is validated against strict security policies and compliance rules before it can be merged or deployed. 

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation/Setup](#installationsetup)
- [Usage](#usage)
- [Contributing](#contributing)
- [License](#license)

## Features

* **Declarative Infrastructure:** Manages resources using Terraform (`main.tf`).
* **Policy as Code (PaC):** Uses Open Policy Agent (OPA) / Conftest and Rego to enforce constraints (e.g., denying VMs with >4GB RAM).
* **Automated CI/CD Pipeline:** GitHub Actions workflow (`pac-pipeline.yml`) acts as a gatekeeper for Pull Requests.
* **Security Scanning:** Out-of-the-box integration with tools like Checkov / tfsec to prevent insecure configurations (e.g., unencrypted SSH keys).
* **Drift Detection:** Scheduled cron jobs to detect and alert on manual CLI changes that drift from the source of truth.
* **Highly Available (HA) Cluster:** Utilizes lightweight Kubernetes (K3s) with a containerized Python watcher script for self-healing and resilience.

## Prerequisites

Before you begin, ensure you have the following installed:
* **OS:** macOS or Linux
* **Package Manager:** Homebrew (for macOS)
* **Tools:** Terraform, Open Policy Agent (OPA), Checkov
* **Cloud Provider:** AWS, GCP, or Azure account with appropriate Role-Based Access Control (RBAC) permissions.

## Installation/Setup

Follow these steps to set up the proof-of-concept environment on macOS:

1. **Install Terraform via Homebrew:**
   ```bash
   brew tap hashicorp/tap
   brew install hashicorp/tap/terraform
   ```

2. **Initialize the Project Directory:**
   ```bash
   mkdir -p proof-of-concept && cd proof-of-concept
   touch main.tf
   mkdir -p .github/workflows
   touch .github/workflows/pac-pipeline.yml
   ```

3. **Initialize Terraform:**
   ```bash
   terraform init
   ```

## Usage

### CI/CD Workflow Execution
The standard workflow follows this path:
1. **Write Configuration:** Update `main.tf` and commit to the repository.
2. **Trigger CI:** Open a Pull Request.
3. **Gatekeeper Validation:** OPA and Checkov run automatically. If policies (e.g., `CKV_CUSTOM_1`) fail, the merge is blocked.
4. **Apply:** Once approved and merged, Terraform applies the infrastructure to the cloud.

### Setting Up the K3s Cluster
To deploy the Highly Available K3s cluster on the provisioned VMs:
```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

### Configuring Secure Access (Cloud-init)
Ensure secure access by provisioning SSH keys during the boot sequence using YAML:
```yaml
users:
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-rsa YOUR_PUBLIC_SSH_KEY_HERE
```
> **Note:** Password authentication should remain disabled to protect against brute-force attacks.

## Contributing

Contributions make the open-source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

Distributed under the MIT License. See `LICENSE` for more information.
