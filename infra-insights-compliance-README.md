# infra-insights-compliance

Red Hat Insights compliance scanning, dynamic inventory, and patching report
automation for CMMC Level 2 RHEL environments.

## Collection Dependency

- `redhat.insights` >= 1.3.1 (provides inventory plugin + compliance role)
- `redhat.rhel_system_roles` (for `rhc` role if onboarding new hosts)

## Prerequisites

1. **Insights client** installed and registered on all target RHEL hosts
   (managed via Satellite 6.17 content views or the `rhc` system role).
2. **Compliance policies** configured in the
   [Insights portal](https://console.redhat.com/insights/compliance)
   and assigned to hosts.
3. **Service account** created at
   [console.redhat.com](https://console.redhat.com/iam/service-accounts)
   with the `Inventory Hosts Viewer` and `Patch viewer` roles.
4. **AAP 2.6** credential for the Insights service account (type:
   custom credential or injected via environment variables).

## Quick Start

```bash
# Install collection dependencies
ansible-galaxy collection install -r collections/requirements.yml

# Test inventory connectivity
ansible-inventory -i inventory/insights.yml --list

# Run full compliance cycle (scan + report)
ansible-playbook -i inventory/insights.yml playbooks/full-compliance-cycle.yml
```

## Playbooks

| Playbook | Purpose |
|---|---|
| `playbooks/compliance-scan.yml` | Install prereqs + run OpenSCAP scan via Insights |
| `playbooks/compliance-install.yml` | Install OpenSCAP prereqs only (no scan) |
| `playbooks/patch-report.yml` | Generate HTML patching/advisory report from Insights inventory data |
| `playbooks/full-compliance-cycle.yml` | Orchestrate: scan all hosts, then generate report |

## Inventory

The dynamic inventory (`inventory/insights.yml`) pulls hosts directly from
console.redhat.com and auto-creates groups based on patching status, advisory
counts, and Insights tags.

### Auto-generated groups

- `patching` — hosts with patching enabled
- `stale` — hosts with stale patching data
- `security_patch` — hosts with outstanding RHSA advisories
- `bug_patch` — hosts with outstanding RHBA advisories
- `enhancement_patch` — hosts with outstanding RHEA advisories
- Tag-based keyed groups prefixed with `insights_`

## AAP 2.6 Integration

1. Create an **Inventory** of type "Red Hat Insights" or import as a
   sourced inventory using the `insights.yml` file.
2. Add the project as an SCM source.
3. Create Job Templates pointing to each playbook.
4. Schedule `full-compliance-cycle.yml` weekly for continuous compliance.

## Vault-Protected Variables

Sensitive values in `inventory/group_vars/all.yml` should be encrypted:

```bash
ansible-vault encrypt_string 'your-client-id' --name 'insights_client_id'
ansible-vault encrypt_string 'your-client-secret' --name 'insights_client_secret'
ansible-vault encrypt_string 'your-proxy-password' --name 'insights_proxy_password'
```

## Proxy Configuration

If your managed hosts require a proxy to reach `console.redhat.com`, the
project supports proxy settings at two levels:

**1. Managed hosts (insights-client):**
Set these in `inventory/group_vars/all.yml`:

```yaml
insights_proxy_enabled: true
insights_proxy_url: "http://proxy.corp.example.com:8080"
insights_proxy_username: "proxy_user"        # optional, for auth proxies
insights_proxy_password: !vault |             # vault-encrypt this
  $ANSIBLE_VAULT;1.1;AES256
  ...
insights_no_proxy: "localhost,127.0.0.1,.corp.ritcsusa.com"
```

When `insights_proxy_enabled: true`, the playbooks will:
- Configure `/etc/insights-client/insights-client.conf` with the proxy URL
- Set proxy auth credentials if provided (managed via `blockinfile`, `no_log: true`)
- Write `HTTPS_PROXY`/`HTTP_PROXY`/`NO_PROXY` to `/etc/sysconfig/insights-client`
- Inject proxy environment variables into all `insights-client` CLI commands

When `insights_proxy_enabled: false` (default), all proxy tasks are skipped.

**2. Inventory plugin (API calls from controller):**
The Insights inventory plugin uses Python `requests` which respects standard
environment variables. Set these in your AAP credential or shell:

```bash
export HTTPS_PROXY=http://proxy.corp.example.com:8080
export HTTP_PROXY=http://proxy.corp.example.com:8080
export NO_PROXY=localhost,127.0.0.1,.corp.ritcsusa.com
```

In AAP 2.6, add these as environment variables in the Custom Credential Type
injector configuration for the Insights service account credential.

## Directory Structure

```
infra-insights-compliance/
├── README.md
├── ansible.cfg
├── collections/
│   └── requirements.yml
├── inventory/
│   ├── insights.yml                 # Dynamic inventory plugin
│   └── group_vars/
│       ├── all.yml                  # Global vars (credentials, proxy, settings)
│       ├── security_patch.yml       # Overrides for RHSA hosts
│       └── patching.yml             # Overrides for patching-enabled hosts
├── tasks/
│   ├── configure_insights_proxy.yml # Shared proxy config for managed hosts
│   └── handlers.yml                 # Shared handlers (insights-client restart)
├── playbooks/
│   ├── compliance-scan.yml
│   ├── compliance-install.yml
│   ├── patch-report.yml
│   └── full-compliance-cycle.yml
└── roles/
    └── insights_report/
        ├── defaults/main.yml
        ├── tasks/main.yml
        └── templates/compliance_report.html.j2
```
