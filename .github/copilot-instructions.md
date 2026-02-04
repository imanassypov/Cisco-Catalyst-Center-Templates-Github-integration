# Copilot Instructions for ansible-git-catc

## Project Overview
GitOps automation for Cisco Catalyst Center (CatC) templates. Syncs Jinja2 templates from Git → CatC Template Projects with auto-versioning using Git commit metadata.

## Architecture
**Main playbook**: [ansible-git-catc/ansible-git-catc.yml](ansible-git-catc/ansible-git-catc.yml)

**Flow**: Clone Git repo → Find `*.j2` files → Extract commit/diff → Create/update CatC templates → Version with commit → Process composite templates

**Key pattern**—lists merged incrementally by `name` key using `community.general.lists_mergeby`:
```yaml
template_id_list: "{{ template_git_list | community.general.lists_mergeby(template_dnac_list, 'name') }}"
```

## File Structure
| File | Purpose |
|------|---------|
| `ansible-git-catc/ansible-git-catc.yml` | Main playbook |
| `ansible-git-catc/composite-resolve-ids.yml` | Included task to resolve template IDs for composites |
| `ansible-git-catc/credentials.yml` | CatC host, Git repo URL, `dnac_version`, `git_branch` |
| `ansible-git-catc/vault.yml` | Encrypted `dnac_username`, `dnac_password` |
| `CatC Templates/*/` | Cloned template repos (auto-populated at runtime) |

## Cisco Catalyst Center Module Pattern
The playbook uses `module_defaults` with YAML anchors. When adding new `cisco.dnac.*` tasks, reference existing anchor:
```yaml
cisco.dnac.configuration_template_info:
  <<: *dnac_connection  # or explicitly include all 7 params below
```
Required parameters: `dnac_host`, `dnac_username`, `dnac_password`, `dnac_verify`, `dnac_port`, `dnac_version`, `dnac_debug`

## Template Conventions

### File Naming
- `FABRIC-*.j2` – Top-level executable templates (included in composites)
- `DEFN-*.j2` – Data definitions (included via Jinja `{% include %}`)
- `FUNC-*.j2` – Macro libraries (included via Jinja `{% include %}`)

### Device Targeting Hint
First line of `.j2` files—parsed via regex:
```jinja
{## CATC: productFamily=Switches and Hubs, softwareType=IOS-XE, productSeries=Cisco Catalyst 9000 Series Virtual Switches ##}
```
Defaults: `DEFAULT_PRODUCT_FAMILY`, `DEFAULT_SOFTWARE_TYPE`, `DEFAULT_PRODUCT_SERIES` in playbook vars.

### Include Pattern
Templates reference others via project-relative includes:
```jinja
{% include "{{ TEMPLATE_PROJECT_NAME }}/DEFN-VRF.j2" %}
```

## Composite Templates
Defined via `.yml` files in the Git repo (e.g., [BGP-EVPN-BUILD.yml](CatC%20Templates/CatalystCenter-BGP-EVPN-VXLAN/BGP%20EVPN/BGP-EVPN-BUILD.yml)):
```yaml
templates:
  - name: "FABRIC-VRF.j2"
  - name: "FABRIC-LOOPBACKS.j2"
```
**Only list `FABRIC-*` templates**—`DEFN-*` and `FUNC-*` are resolved at render time.

## Commands

```bash
# Setup
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

# Run
ansible-playbook ansible-git-catc/ansible-git-catc.yml --vault-password-file .vault_pass

# Debug mode (verbose output)
DEBUG=true ansible-playbook ansible-git-catc/ansible-git-catc.yml --vault-password-file .vault_pass
```

## Constraints
- **Summary limit**: 1024 chars (`CATC_TEMPLATE_SUMMARY_MAXCHAR`)
- **Idempotency**: Updates only when `payload != dnac_template_content` (or deviceType changes)
- **Version compatibility**: Match `dnac_version` in credentials.yml to SDK versions:
  | CatC Version | `cisco.dnac` | `dnacentersdk` |
  |--------------|--------------|----------------|
  | 2.3.7.6 | 6.25.0 | 2.8.3 |
  | 2.3.7.9 | 6.33.2 | 2.8.6 |
  | 3.1.3.0 | ≥6.36.0 | ≥2.10.1 |
