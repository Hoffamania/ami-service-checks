# Project Configuration

## Style Guide

Follow `~/.claude/style-guide.md` for all documentation, commit messages, and code comments.

**Key requirement:** Be specific, not vague.
- Not: "Improving playbook"
- Use: "Adding SSHD validation for Amazon Linux 2023"
- Not: "Fixing documentation"
- Use: "Correcting service state table for Vault on Rocky Linux"

---

## Project Overview

Ansible playbook that validates service installation and state during Packer AMI builds. Runs locally inside the AMI being built and fails the build if any service check fails.

**Execution context:** Local Ansible run via Packer's `ansible-local` provisioner with `become: true`

**Target environments:** Amazon Linux 2/2023, CentOS, Debian, Rocky Linux, Ubuntu

---

## Playbook Structure

### Service Validation Pattern

Each service follows this pattern:

```yaml
- name: "Gather service facts"
  service_facts:

- name: "Display service state"
  debug:
    msg: "{{ansible_facts.services['service.service'].state}}"
  when: <distribution conditionals>

- name: "Validate service state"
  assert:
    that:
      - "'{{ansible_facts.services['service.service'].state}}' == 'running'"
    fail_msg: "Service validation failed"
    success_msg: "Service validated"
  when: <distribution conditionals>
```

### Ansible Facts Used

**Distribution detection:**
- `ansible_facts['distribution']` - Returns: "Amazon", "Ubuntu", "Debian", "Rocky", "CentOS"
- `ansible_facts['distribution_major_version']` - Returns: "2", "2023", "20", "22", etc.

**Service state:**
- `ansible_facts.services['name.service'].state` - Returns: "running", "inactive", "stopped", "failed"

**Example conditionals:**

```yaml
# Amazon Linux 2
when: ansible_facts['distribution'] == "Amazon" and
      ansible_facts['distribution_major_version'] == "2"

# Multiple distributions
when: (ansible_facts['distribution'] == "Rocky") or
      (ansible_facts['distribution'] == "Debian")

# Exclude specific distribution
when: ansible_facts['distribution'] != "Ubuntu"
```

---

## Current Service Checks

### All Distributions (No Conditionals)

**Crowdstrike Falcon Sensor** (lines 51-63)
- Service: `falcon-sensor.service`
- Expected state: `running`
- Purpose: Security agent for threat detection

**Datadog Agent** (lines 65-79)
- Service: `datadog-agent.service`
- Expected states: `running`, `inactive`, or `stopped` (any acceptable)
- Purpose: Monitoring and observability

**HashiCorp Vault** (lines 124-147)
- Service: `vault.service`
- Expected state: `inactive`
- Binary check: `/bin/vault` must exist
- Purpose: Secrets management (configured but not started until runtime)

### Distribution-Specific Checks

**Amazon SSM Agent (systemd)** (lines 17-39)
- Service: `amazon-ssm-agent.service`
- Expected state: `running`
- Distributions: CentOS, Debian, Rocky, Amazon Linux 2, Amazon Linux 2023
- Purpose: AWS Systems Manager agent for remote management

**Amazon SSM Agent (snap)** (lines 43-49)
- Service: `snap.amazon-ssm-agent.amazon-ssm-agent.service`
- Expected state: `running`
- Distribution: Ubuntu only
- Purpose: SSM agent via snap package manager
- Reference: https://docs.aws.amazon.com/systems-manager/latest/userguide/agent-install-ubuntu-64-snap.html

**TD-Agent (Treasure Data)** (lines 81-99)
- Service: `td-agent.service`
- Expected state: `inactive`
- Distributions: CentOS, Ubuntu, Amazon Linux 2
- Purpose: Log collection for Sumo Logic

**Fluentd** (lines 101-120)
- Service: `fluentd.service`
- Expected state: `inactive`
- Distributions: Amazon Linux 2023, Debian, Rocky
- Purpose: Log collection for Sumo Logic (replaces td-agent on newer distros)

---

## Adding New Service Checks

### Step 1: Determine Service Scope

**Option A: All distributions**
- Add task without `when:` conditional
- Example: Vault, Datadog, Falcon Sensor

**Option B: Specific distributions**
- Add `when:` conditional with distribution checks
- Example: SSM agent, td-agent, fluentd

**Option C: Different service names per distribution**
- Create separate tasks for each variant
- Example: SSM agent (systemd vs snap)

### Step 2: Define Expected State

**running** - Service must be active
- Use for: Monitoring agents, security agents, system services
- Examples: SSM agent, Falcon sensor

**inactive** - Service installed but not started
- Use for: Runtime services configured during build
- Examples: Vault, Fluentd, td-agent

**any state** - Service present, state doesn't matter
- Use for: Services with variable runtime states
- Examples: Datadog (may be running or inactive)

### Step 3: Add Tasks to Playbook

```yaml
- name: "Check status of <service-name>"
  service_facts:

- name: "Display <service-name> state"
  debug:
    msg: "{{ansible_facts.services['<service>.service'].state}}"
  when: <conditionals>

- name: "Validate <service-name> state"
  assert:
    that:
      - "'{{ansible_facts.services['<service>.service'].state}}' == 'running'"
    fail_msg: "<Service> not in expected state"
    success_msg: "<Service> validated successfully"
  when: <conditionals>
```

### Step 4: Update README.md

Add service to appropriate table in README.md:

**For all-distribution services:**
```markdown
| `service-name`       | running        | Purpose description             |
```

**For distribution-specific services:**
```markdown
| Distribution Name    | Service Name    | running        | `service-name.service`          |
```

### Step 5: Test Changes

```bash
# Syntax validation
ansible-playbook --syntax-check service-check-playbook.yml

# Dry run (requires target system)
ansible-playbook --check service-check-playbook.yml

# Full execution (requires target system)
ansible-playbook service-check-playbook.yml
```

---

## Binary Path Validation

For services installed via non-package methods, validate binary location.

**Pattern:**
```yaml
- name: "Register <binary> installation"
  ansible.builtin.stat:
    path: /path/to/binary
  register: binary_check

- name: "Validate <binary> exists"
  fail:
    msg: "Binary not found at /path/to/binary"
  when: binary_check.stat.path != '/path/to/binary'
```

**Example:** Vault validation (line 139-147)
```yaml
- name: "Register Vault installation status"
  ansible.builtin.stat:
    path: /bin/vault
  register: vault_installed
  tags: register-vault

- name: "Check vault installation status"
  fail:
    msg: "Vault binary is not installed"
  when: vault_installed.stat.path != '/bin/vault'
```

---

## Distribution-Specific Service Names

Some services use different names across distributions.

**Example: SSM Agent**
- Most: `amazon-ssm-agent.service`
- Ubuntu: `snap.amazon-ssm-agent.amazon-ssm-agent.service`

**Implementation:**
```yaml
# Standard systemd service
- name: "Validate SSM agent on systemd distributions"
  assert:
    that:
      - "'{{ansible_facts.services['amazon-ssm-agent.service'].state}}' == 'running'"
  when: ansible_facts['distribution'] != "Ubuntu"

# Ubuntu snap service
- name: "Validate SSM agent on Ubuntu via snap"
  assert:
    that:
      - "'{{ansible_facts.services['snap.amazon-ssm-agent.amazon-ssm-agent.service'].state}}' == 'running'"
  when: ansible_facts['distribution'] == "Ubuntu"
```

---

## Common Issues

### Issue: Undefined Variable Error

**Symptom:** `"ansible_facts.services['service.service']" is undefined`

**Cause:** Missing `service_facts:` call before accessing service data

**Solution:** Add `service_facts:` task before any service state checks

### Issue: Service Not Found

**Symptom:** Task fails with service not found error

**Cause:** Service name doesn't match systemd unit name

**Solution:**
1. SSH to target system
2. Run: `systemctl list-units --type=service | grep <name>`
3. Use exact systemd unit name in playbook

### Issue: Wrong State Value

**Symptom:** Assertion fails despite service running

**Cause:** Expected state doesn't match actual reported state

**Solution:** Add debug task to display actual state before writing assertion

### Issue: Distribution Version Mismatch

**Symptom:** Conditional fails on correct distribution

**Cause:** `distribution_major_version` is string not integer

**Solution:** Use string comparison: `== "2023"` not `== 2023`

### Issue: Task Runs on Wrong Distribution

**Symptom:** Task fails on distribution where service doesn't exist

**Cause:** Missing or incorrect `when:` conditional

**Solution:** Add distribution check to restrict task scope

---

## Testing Strategy

### Local Testing (Without Packer)

1. Launch EC2 instance from target base AMI
2. SSH to instance: `ssh ec2-user@<ip>`
3. Install Ansible: `sudo yum install ansible` or `sudo apt install ansible`
4. Copy playbook: `scp service-check-playbook.yml ec2-user@<ip>:~`
5. Run playbook: `ansible-playbook -i localhost, -c local service-check-playbook.yml`
6. Review output for failures

### Packer Integration Testing

1. Add playbook to Packer template provisioners section
2. Run Packer build: `packer build template.pkr.hcl`
3. Monitor Packer output for Ansible task results
4. Verify build aborts if checks fail
5. Verify AMI created if checks pass

### CI/CD Validation

```bash
# Syntax check
ansible-playbook --syntax-check service-check-playbook.yml

# Ansible lint
ansible-lint service-check-playbook.yml

# Packer validation
packer validate template.pkr.hcl

# Full build
packer build template.pkr.hcl
```

---

## Packer Configuration

### HCL Format

```hcl
provisioner "ansible-local" {
  playbook_file    = "service-check-playbook.yml"
  extra_arguments  = ["--check"]
}
```

### JSON Format

```json
{
  "type": "ansible-local",
  "playbook_file": "service-check-playbook.yml",
  "extra_arguments": ["--check"]
}
```

### Execution Flow

1. Packer launches EC2 instance from base AMI
2. Packer installs Ansible on instance
3. Packer copies playbook to instance
4. Packer executes: `ansible-playbook service-check-playbook.yml`
5. Playbook gathers facts and validates services
6. Build aborts if any assertion fails
7. Packer creates AMI if all assertions pass
8. Packer terminates instance

---

## Debugging Service Checks

### Display All Services

```yaml
- name: "Show all detected services"
  debug:
    var: ansible_facts.services
```

### Display Specific Service

```yaml
- name: "Show service details"
  debug:
    msg: "{{ ansible_facts.services['service.service'] }}"
```

### Display Distribution Info

```yaml
- name: "Show distribution facts"
  debug:
    msg: "Distribution: {{ ansible_facts['distribution'] }} Version: {{ ansible_facts['distribution_major_version'] }}"
```

### Enable Verbose Output

```bash
# Local execution
ansible-playbook -v service-check-playbook.yml

# Packer execution
provisioner "ansible-local" {
  playbook_file = "service-check-playbook.yml"
  extra_arguments = ["-v"]
}
```

---

## Commit Message Format

Follow `~/.claude/style-guide.md` conventions.

**Good examples:**

```
feat: Add SSHD validation for Amazon Linux 2023

Check that sshd.service is running on Amazon Linux 2023.
Validates SSH access available after AMI launch.

Lines: 150-165
```

```
fix: Correct Falcon Sensor conditional for Debian

Add missing distribution check to restrict Falcon
validation to distributions where it's installed.

Previously ran on all distros causing failures.
```

```
docs: Update service table with Fluentd entry

Add Fluentd service to distribution-specific table.
Document that it's checked on AL2023, Debian, Rocky.

Previously documented only td-agent.
```

**Bad examples:**

```
fix: Improving playbook
# Too vague - what specifically improved?
```

```
docs: Updating README
# What changed in README?
```

```
refactor: Clean up code
# What code? What cleanup?
```

---

## File Structure

```
ami-service-checks/
├── CLAUDE.md                           # This file - project instructions
├── README.md                           # User documentation
├── service-check-playbook.yml          # Ansible playbook
├── .gitignore                          # Git exclusions
├── ami_lifecycle_ansible_validation_remixed.png
└── A_flowchart_infographic_illustrates_the_process_of.png
```

---

## Related Documentation

- [Ansible service_facts module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_facts_module.html) - Service state gathering
- [Ansible assert module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html) - Validation assertions
- [Ansible stat module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/stat_module.html) - File/binary checks
- [Ansible conditionals](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html) - When statements
- [Packer Ansible Local](https://developer.hashicorp.com/packer/plugins/provisioners/ansible/ansible-local) - Provisioner docs
- [systemd service states](https://www.freedesktop.org/software/systemd/man/systemctl.html) - Service state reference

---

**Last updated:** 2024-11-18
