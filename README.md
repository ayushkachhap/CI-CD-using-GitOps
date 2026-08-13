[README.md](https://github.com/user-attachments/files/31035869/README.md)
# Immutable Global Portfolio: Secure-by-Default Infrastructure

This is a fantastic project architecture. Combining declarative automation with Policy as Code (PaC) is exactly the kind of "fail-proof" methodology that stands out on a GitHub portfolio, especially when demonstrating enterprise-grade reliability and security-by-default practices. Here is a breakdown of how to structure the pipeline, the best tools for the job, and how to elevate it further.

## The Goal: "Secure by Default" Deployment

To make GitHub the single source of truth and enforce policies before any infrastructure is spun up, you will want a Continuous Integration/Continuous Deployment (CI/CD) pipeline.

### CI/CD Workflow

1.  **Version Control (Git):** Write your Terraform configuration (`main.tf`) and your PaC rules, then commit them to your repository.
2.  **Trigger (GitHub Actions):** Open a Pull Request.
3.  **Gatekeeper (Policy Check):** A policy engine runs against the code. It checks for simple requirements (e.g., "Are SSH keys encrypted?" or resource limits defined).
4.  **Approval & Merge:** If the code violates policies, the pipeline fails (cannot be merged). If it passes, it is approved.
5.  **Deployment:** Once approved, the infrastructure is deployed to the target cloud provider.

## Recommended Tooling

Because you are likely working locally on your machine, HashiCorp doesn't have a tier-one provider for this. However, you can manage this with the following tools:

*   **Policy Engine:** Open Policy Agent (OPA) / Conftest. This allows custom policies written in a language called Rego.
    *   *Example Policy:* "Deny any VM deployment that requests more than 4GB RAM."
*   **Alternative Policy Engine:** Checkov. It comes with hundreds of built-in policies tailored for major cloud environments (AWS/GCP/Azure).
*   **CI/CD Runner:** GitHub Bridge (Self-Hosted Runner). Since you are initiating from the cloud, GitHub needs to "see" a daemon running on your primary Ubuntu center. This allows securely executing deployments directly from the blueprint.

---

## Phase 1: Local Setup & Initialization

### Prerequisites (macOS)
If you haven't already, grab Homebrew from `brew.sh`. Otherwise, open your terminal and get ready.

### Step 1: Install Toolchain
Install Terraform and Python (best practice is via Homebrew). Keep them updated:

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform python
```
Verify the installation to ensure it is ready.

### Step 2: Setup Workspace
A clean, modular structure is critical for high-availability scale. Create a dedicated space for your infrastructure logic:

```bash
mkdir -p proof-of-concept
cd proof-of-concept
mkdir -p .github/workflows
touch .github/workflows/pac-1.yml main.tf variables.tf
```

### Step 3: Generate a Secure Key
You need an SSH key to access your virtual machines securely. Generate a modern `ed25519` key (it is more secure than RSA):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```
*(The `-N ""` parameter skips the passphrase prompt for automation purposes).*

### Step 4: Copy the Public Key
On macOS, use `pbcopy` to quickly grab the contents of your new public key:

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```
You will paste this (`Cmd + V`) into your Terraform configuration later.

---

## Phase 2: Configuration & Deployment (Terraform)

### Defining the Infrastructure (`main.tf`)
When designing for High Availability (HA) right from the start, you want to deploy multiple nodes.

```hcl
variable "node_count" {
  description = "Number of instances"
  default     = 2
}

variable "memory_limit" {
  default     = "2G"
}

# Example of dynamic naming and IP mapping
locals {
  instance_names = [for i in range(var.node_count) : "portfolio_node_${i + 1}"]
}
```
*Note: Ensure your sensitive files (like `.tfstate` and `.tfvars`) are protected by adding them to your `.gitignore`.*

### Cloud-Init (Bootstrap Script)
Cloud-init is industry-standard software used to configure VMs as they boot up. It uses YAML formatting.

```yaml
# cloud-config
package_upgrade: true
users:
  - name: admin
    gecos: Admin User
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: sudo
    shell: /bin/bash
    ssh_authorized_keys:
      - YOUR_PUBLIC_SSH_KEY_HERE # Paste your copied key here

ssh_pwauth: false # Force SSH key usage, disable passwords

runcmd:
  - echo "Cloud-init complete. Node is ready." >> /var/log/cloud-init.log
```

---

## Phase 3: Policy Enforcement (The Gatekeeper)

This is where the "fail-proof" aspect comes in. We use tools like Checkov to catch errors *before* deployment.

### Example Checkov Policy (Custom)
```yaml
id: "CKV_CUSTOM_1"
category: "GENERAL_SECURITY"
cond_type: "attribute"
attribute: "ssh_authorized_keys"
operator: "exists"
```
This policy ensures that *someone* hasn't accidentally deleted the SSH key requirement.

### Running the CI Pipeline
When you push your code, GitHub Actions takes over.

```yaml
# .github/workflows/pac-1.yml
name: Security & Policy Check
on: [push, pull_request]

jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
```
If the pipeline fails, you cannot merge. This protects your physical servers from bad configurations.

---

## Phase 4: Kubernetes & Chaos Testing

### K3s Installation
We use K3s, a lightweight, real-world Kubernetes distribution.

**Step 1: Install K3s**
Run this on your deployed VMs:
```bash
curl -sfL https://get.k3s.io | sh -
```

**Step 2: Deploy Workloads**
You can now deploy Nginx or other web applications using standard `kubectl` commands.

### The "Holy Trinity" of Reliability: Chaos Engineering
To prove enterprise-grade reliability, we must test the system's resilience by intentionally breaking things.

**Problem Statement:** What happens when a node fails at 3:00 AM?
**Solution:** The system should fix itself without human intervention.

**Step 1: Identify Target**
Check your running pods:
```bash
sudo k3s kubectl get pods
# Output example: web-5c8b556c8f-xyz12 1/1 Running 0 15s
```

**Step 2: Execute Chaos**
Intentionally destroy the pod:
```bash
sudo k3s kubectl delete pod web-5c8b556c8f-xyz12
```

**Step 3: Observe Auto-Recovery**
Immediately check the pods again. You should witness the orchestration loop working—the old pod is dropping, and a new one is spinning up instantly beneath it. This "Auto-Healing" demonstrates true microservice resilience.

---

## Final Note on Secrets Management
As a senior practice, *never* hardcode credentials (like AWS or Google API keys). Use encrypted secrets injected via your CI runner (e.g., `${{ secrets.AWS_ACCESS_KEY }}`). This philosophy of absolute security is critical for modern GitOps.
