# ami-service-checks

**Ansible playbook for validating service installation and state during AMI builds.**

Validates that required services are installed and in the correct state before Packer finalizes the AMI. Detects misconfigured or missing services before the image reaches production.

---

## Purpose

Detects service configuration problems during AMI build rather than after deployment.

**Validation approach:**
- Read-only checks via Ansible `service_facts` and `stat` modules
- Runs locally inside the AMI during Packer provisioning
- Fails Packer build if any service check fails
- Supports Amazon Linux 2/2023, CentOS, Debian, Rocky Linux, Ubuntu

---

## Workflow

1. Packer boots EC2 instance from base AMI
2. Packer installs Ansible on the instance
3. Packer runs playbook locally inside the instance
4. Playbook validates service installation and state
5. Packer aborts build if validation fails
6. Packer creates AMI if validation succeeds

---

## ✅ Services Validated

### All Distributions

| Service              | Expected State | Notes                           |
|----------------------|----------------|---------------------------------|
| `vault`              | inactive       | Checks service and `/bin/vault` binary |
| `datadog-agent`      | running, inactive, or stopped | Any state acceptable |
| `falcon-sensor`      | running        | Crowdstrike security agent      |

### Distribution-Specific Checks

| Distribution         | Service         | Expected State | Service Name                    |
|----------------------|-----------------|----------------|---------------------------------|
| Amazon Linux 2       | SSM Agent       | running        | `amazon-ssm-agent.service`      |
| Amazon Linux 2       | TD-Agent        | inactive       | `td-agent.service`              |
| Amazon Linux 2023    | SSM Agent       | running        | `amazon-ssm-agent.service`      |
| Amazon Linux 2023    | Fluentd         | inactive       | `fluentd.service`               |
| CentOS               | SSM Agent       | running        | `amazon-ssm-agent.service`      |
| CentOS               | TD-Agent        | inactive       | `td-agent.service`              |
| Debian               | SSM Agent       | running        | `amazon-ssm-agent.service`      |
| Debian               | Fluentd         | inactive       | `fluentd.service`               |
| Rocky Linux          | SSM Agent       | running        | `amazon-ssm-agent.service`      |
| Rocky Linux          | Fluentd         | inactive       | `fluentd.service`               |
| Ubuntu               | SSM Agent       | running        | `snap.amazon-ssm-agent.amazon-ssm-agent.service` (snap) |
| Ubuntu               | TD-Agent        | inactive       | `td-agent.service`              |

---

## Packer Integration

```hcl
provisioner "ansible-local" {
  playbook_file    = "service-check-playbook.yml"
  extra_arguments  = ["--check"]
}
```

---

## References

- [Packer Ansible Local Docs](https://developer.hashicorp.com/packer/plugins/provisioners/ansible/ansible-local)
- [Ansible Facts & Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)
- [Checkov (Security Scanning)](https://www.checkov.io/)
- [Best Practices for AMI Hardening (AWS)](https://aws.amazon.com/premiumsupport/knowledge-center/ami-hardening/)

---

## Diagrams

Visual representations of the AMI validation workflow:
- [AMI lifecycle with Ansible validation](./ami_lifecycle_ansible_validation_remixed.png)
- [Mobile-friendly flowchart](./A_flowchart_infographic_illustrates_the_process_of.png)
