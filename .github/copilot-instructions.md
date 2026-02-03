# Copilot Instructions for ansible-git-catc

## Project Overview
GitOps automation for Cisco Catalyst Center templates. Syncs Jinja2 templates from Git → Cisco Catalyst Center Template Projects, auto-versioning with Git commit metadata.

## Architecture (Single Playbook)
[ansible-git-catc/ansible-git-catc.yml](ansible-git-catc/ansible-git-catc.yml) executes this flow:
1. Clone Git repo → 2. Find `*.j2` files → 3. Extract commit/diff metadata → 4. Create/update Cisco Catalyst Center templates → 5. Version with commit message

Key data structure pattern—lists merged incrementally by `name` key:
```yaml
template_id_list: "{{ template_git_list | community.general.lists_mergeby(template_dnac_list, 'name') }}"
```

## File Structure
| File | Purpose |
|------|---------|
| `ansible-git-catc/ansible-git-catc.yml` | Main playbook (all logic in one file) |
| `ansible-git-catc/credentials.yml` | Cisco Catalyst Center host, Git repo URL, `dnac_version` |
| `ansible-git-catc/vault.yml` | Encrypted credentials (`dnac_username`, `dnac_password`) |
| `requirements.txt` / `requirements.yml` | Python + Ansible collection deps |
| `ENSEVT-TEMPLATES/`, `CatalystCenter-BGP-EVPN-VXLAN/` | Sample template repos |

## Cisco Catalyst Center Module Pattern
All `cisco.dnac.*` calls require these 7 parameters—copy exactly:
```yaml
dnac_host: "{{ dnac_host }}"
dnac_username: "{{ dnac_username }}"
dnac_password: "{{ dnac_password }}"
dnac_verify: "{{ dnac_verify }}"
dnac_port: "{{ dnac_port }}"
dnac_version: "{{ dnac_version }}"
dnac_debug: "{{ dnac_debug }}"
```

## Template Conventions
- Extension: `.j2` (configured via `template_extension` in credentials.yml)
- Language: `JINJA` (not Velocity)—hardcoded in playbook
- Diff header: Git diff auto-prepended as `{## ... ##}` Jinja comments

### Device Targeting Hints
Templates can specify device type via optional hint comment (first line recommended):
```jinja
{## CATC: productFamily=Switches and Hubs, softwareType=IOS-XE ##}
```
| Parameter | Default | Common Values |
|-----------|---------|---------------|
| `productFamily` | `Switches and Hubs` | `Routers`, `Switches and Hubs`, `Wireless Controller` |
| `softwareType` | `IOS-XE` | `IOS-XE`, `IOS`, `NX-OS` |

If no hint is present, defaults from playbook vars are used.

## Commands

### Setup
```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

### Run (with Vault)
```bash
ansible-playbook ansible-git-catc/ansible-git-catc.yml --vault-password-file .vault_pass
```

### Debug Mode
```bash
DEBUG=true ansible-playbook ansible-git-catc/ansible-git-catc.yml --vault-password-file .vault_pass
```

## Constraints
- **Summary limit**: 1024 chars max (`CATC_TEMPLATE_SUMMARY_MAXCHAR`)
- **Project naming**: Auto-extracted from Git URL via regex `.*\/([^\/]*).git`
- **Idempotency**: Updates only when `diff_msg + payload != dnac_template_content`
- **Version compatibility**: Match `dnac_version` in credentials.yml to SDK/collection versions (see README.md compatibility matrix)
