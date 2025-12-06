# Red Hat Insights: Compliance & Malware Detection Playbook

Ansible playbook for RHEL 9 systems that performs registration, compliance scanning, and malware detection using **only official Red Hat supported roles and collections**.

## What This Playbook Does

1. **Registers the system** with Red Hat Subscription Manager using activation keys
2. **Connects the system to Red Hat Insights** via the `rhc` client
3. **Installs compliance prerequisites** (OpenSCAP scanner, SCAP Security Guide)
4. **Runs a compliance scan** against CIS Level 1 Benchmark for RHEL 9
5. **Uploads compliance results** to Red Hat Insights
6. **Enables Malware Detection** with scheduled scans
7. **Assigns systems to compliance policies** (optional)

## Official Roles Used

| Role | Collection | Purpose |
|------|------------|---------|
| `rhc` | `redhat.rhel_system_roles` | RHSM registration + Insights connection |
| `compliance` | `redhat.insights` | OpenSCAP installation and scanning |

## Prerequisites

### Control Node Requirements

- Ansible Core 2.14+ or AAP 2.4+
- Network access to managed nodes
- Ansible Vault for secrets management

### Managed Node Requirements

- Red Hat Enterprise Linux 9.x
- Network access to:
  - `subscription.rhsm.redhat.com` (RHSM)
  - `cert.console.redhat.com` (Insights)
  - `cloud.redhat.com` (Insights API)
- Root or sudo privileges

### Red Hat Console Setup

Before running this playbook, complete these steps in [console.redhat.com](https://console.redhat.com):

1. **Create an Activation Key**
   - Navigate to: Insights → Connector → Activation Keys
   - Create a key with RHEL 9 content enabled
   - Note the key name and your Organization ID

2. **Create a Compliance Policy**
   - Navigate to: Security → Compliance → SCAP Policies
   - Click "Create new policy"
   - Select: "CIS Red Hat Enterprise Linux 9 Benchmark"
   - Choose profile: "Level 1 - Server" (or Level 2 as needed)
   - Copy the policy UUID from the URL

## Quick Start

### 1. Install Required Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Vault Secrets

```bash
# Copy the example vault file
cp vault.yml.example vault.yml

# Encrypt it
ansible-vault encrypt vault.yml

# Edit to add your credentials
ansible-vault edit vault.yml
```

### 3. Update Inventory

Edit `inventory.ini` and add your RHEL 9 systems:

```ini
[rhel9]
server01.example.com
server02.example.com
192.168.1.100
```

### 4. Run the Playbook

```bash
# Full run with vault password prompt
ansible-playbook insights_compliance.yml --ask-vault-pass

# Or with vault password file
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook insights_compliance.yml
```

## Configuration Variables

### Registration Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `vault_rhc_organization` | Red Hat Organization ID | Yes |
| `vault_activation_key` | Activation key name | Yes |
| `insights_environment` | Environment tag for Insights | No (default: production) |

### Compliance Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `vault_compliance_policy_id` | UUID of compliance policy | No |

### Malware Detection Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `malware_detection_enabled` | `true` | Enable malware detection |
| `malware_scan_on_playbook_run` | `true` | Run scan during playbook execution |
| `malware_schedule_enabled` | `true` | Enable scheduled scans |
| `malware_schedule_weekday` | `0` | Day of week (0=Sunday) |
| `malware_schedule_hour` | `2` | Hour (24h format) |
| `malware_schedule_minute` | `30` | Minute |

## Running Specific Phases

Use tags to run specific phases of the playbook:

```bash
# Registration only
ansible-playbook insights_compliance.yml --tags registration

# Compliance scan only (requires prior registration)
ansible-playbook insights_compliance.yml --tags compliance

# Malware detection setup only
ansible-playbook insights_compliance.yml --tags malware

# Verification only
ansible-playbook insights_compliance.yml --tags verify
```

## AAP / Automation Controller Integration

### Project Setup

1. Create a Project pointing to this repository/directory
2. Ensure "Update on Launch" is enabled
3. Add collection requirements path: `requirements.yml`

### Credential Setup

Create a "Vault" type credential with your vault password.

### Job Template

| Setting | Value |
|---------|-------|
| Inventory | Your RHEL 9 inventory |
| Project | This project |
| Playbook | `insights_compliance.yml` |
| Credentials | Machine + Vault |
| Extra Variables | Override defaults as needed |

### Workflow Integration

This playbook works well in workflows:

```
[Provision VM] → [This Playbook] → [Application Deploy] → [Validate]
```

## Viewing Results

After successful execution:

1. **Registration Status**: Insights → Inventory
2. **Compliance Results**: Security → Compliance → Reports
3. **Malware Scans**: Security → Malware

## Troubleshooting

### Registration Failures

```bash
# Check RHSM status
subscription-manager status

# Check Insights connection
insights-client --test-connection

# View registration logs
journalctl -u rhcd
```

### Compliance Scan Issues

```bash
# Verify SSG installation
rpm -q scap-security-guide

# Check available profiles
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Manual compliance check
insights-client --compliance
```

### Malware Detection Issues

```bash
# Verify YARA installation
rpm -q yara

# Manual malware scan
insights-client --collector malware-detection

# Check scan logs
cat /var/log/insights-malware-detection.log
```

## File Structure

```
insights-compliance-role/
├── ansible.cfg              # Ansible configuration
├── insights_compliance.yml  # Main playbook
├── inventory.ini            # Inventory file
├── requirements.yml         # Collection requirements
├── vault.yml.example        # Vault template
├── vault.yml                # Your encrypted secrets (create this)
└── README.md                # This file
```

## Security Considerations

- Store all credentials in Ansible Vault
- Use activation keys instead of username/password
- Restrict access to vault password files (`chmod 600`)
- Review compliance policies before enabling auto-remediation
- Malware detection logs may contain sensitive path information

## References

- [RHEL System Roles Documentation](https://access.redhat.com/articles/3050101)
- [rhc Role README](https://github.com/linux-system-roles/rhc)
- [Red Hat Insights Compliance](https://docs.redhat.com/en/documentation/red_hat_insights/)
- [Insights Malware Detection](https://access.redhat.com/documentation/en-us/red_hat_insights/)
- [CIS Benchmarks](https://www.cisecurity.org/benchmark/red_hat_linux)

## License

MIT - Use freely in your environment.

## Author

Generated for CMMC Level 2 compliance automation.
