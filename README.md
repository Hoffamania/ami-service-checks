<<<<<<< HEAD
# 🧪 ami-service-checks

**A reusable Ansible playbook for validating AMI readiness during Packer builds.**  
Make sure your golden images are actually ready — without brittle shell scripts.

---

## 🔍 Why This Exists

Golden images are great — until your Vault agent doesn’t start, SSHD fails silently, or your observability stack is missing.  
This playbook gives you **early failure visibility** by validating runtime services *before the AMI is ever shared*.

✅ Lightweight  
✅ Read-only  
✅ Deterministic  
✅ CI/CD friendly

---

## 🧭 Lifecycle Diagram

![AMI Validation Flowchart](./ami_lifecycle_ansible_validation_remixed.png)

---

## 📸 Illustrated Version

Prefer more visuals? Here's a mobile-friendly version too:

👉 [Download alternate mobile-friendly version](./A_flowchart_infographic_illustrates_the_process_of.png)
=======

# 🧪 ami-service-checks

**A reusable Ansible playbook for validating AMI readiness during Packer builds.**  
Make sure your golden images are actually ready — without brittle shell scripts.

---

## 🔍 Why This Exists

Golden images are great — until your Vault agent doesn’t start, SSHD fails silently, or your observability stack is missing.  
This playbook gives you **early failure visibility** by validating runtime services *before the AMI is ever shared*.

✅ Lightweight  
✅ Read-only  
✅ Deterministic  
✅ CI/CD friendly

---

## 🧭 Lifecycle Diagram (Mermaid.js)

```mermaid
flowchart TD
  A[Packer CLI] --> B[Loads Packer Template (HCL)]
  B --> C[Authenticates with AWS via IAM Role]
  C --> D[Launch EC2 Builder Instance]
  D --> E[Provision with Ansible (Local)]
  E --> F[Run service-check-playbook.yml]

  subgraph F [Run service-check-playbook.yml]
    F1[Gather facts (always)]
    F2[If Amazon Linux 2 → Check amazon-ssm-agent]
    F3[If Amazon Linux 2023 → Check vault & sshd]
    F4[If Rocky → Check vault & sshd]
    F5[If Debian → Check vault & sshd]
    F6[If Ubuntu → Check vault & sshd]
    F7[All → Check datadog-agent (if present)]
    F8[All → Check td-agent (if present)]
  end

  F --> G{All checks passed?}
  G -->|Yes| H[📦 Create Golden AMI]
  G -->|No| I[❌ Abort Build – Fail Early]
  H --> J[Use AMI in CI/CD or Autoscaling Pipelines]

  style H fill:#c6f6d5,stroke:#38a169
  style I fill:#fed7d7,stroke:#e53e3e
  style A fill:#f0f9ff,stroke:#2b6cb0
  style F fill:#ebf8ff,stroke:#4299e1
```

---

## 📸 Illustrated Version

Prefer visuals? Here's a full-color version with icons and metaphors:

👉 [View `ami_lifecycle_ansible_validation_remixed.png`](./ami_lifecycle_ansible_validation_remixed.png)  
👉 [Or download mobile-friendly version](./A_flowchart_infographic_illustrates_the_process_of.png)
>>>>>>> main

---

## 🛠 How It Works

1. Packer boots an EC2 instance
2. Ansible runs **locally inside** the image being built
3. The playbook performs **read-only validation** of services and OS traits
4. Failing checks abort the build early

---

## ✅ Services Validated

| OS Condition         | Validations Performed                   |
|----------------------|-----------------------------------------|
| Amazon Linux 2       | Check for `amazon-ssm-agent`            |
| Amazon Linux 2023    | Check `vault`, `sshd`                   |
| Rocky Linux          | Check `vault`, `sshd`                   |
| Debian               | Check `vault`, `sshd`                   |
| Ubuntu               | Check `vault`, `sshd`                   |
| All Distros (if present) | Check `datadog-agent`, `td-agent`   |

---

## 🧾 Example Packer Config

```hcl
provisioner "ansible-local" {
  playbook_file    = "service-check-playbook.yml"
  extra_arguments  = ["--check"]
}
```

---

## 📚 References

- [Packer Ansible Local Docs](https://developer.hashicorp.com/packer/plugins/provisioners/ansible/ansible-local)
- [Ansible Facts & Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)
- [Checkov (Security Scanning)](https://www.checkov.io/)
- [Best Practices for AMI Hardening (AWS)](https://aws.amazon.com/premiumsupport/knowledge-center/ami-hardening/)

---

*Brought to you by an SRE who prefers certainty over surprise.* 🧠🔐
