# Ansible Playbook for Cisco Catalyst Center Template Synchronization

Automate the synchronization of Jinja2 templates between a Git repository and Cisco Catalyst Center Template Projects using GitOps workflows.

## Features

- Leverages the official [Cisco DNA Center Ansible Collection](https://galaxy.ansible.com/cisco/dnac) (`cisco.dnac`)
- Integrates with the [Cisco DNA Center Python SDK](https://github.com/cisco-en-programmability/dnacentersdk) (`dnacentersdk`)
- Clones a specified Git repository containing Jinja2 templates
- Synchronizes template content to Cisco Catalyst Center Template Projects
- Automatically adds Git commit messages to template version descriptions
- Embeds Git diff information as Jinja comments in template payloads for traceability

## Sample Repository

A sample template repository is available for reference:  
**Repository:** [CatalystCenter-BGP-EVPN-VXLAN](https://github.com/imanassypov/CatalystCenter-BGP-EVPN-VXLAN)  
**Branch:** `composite-template`

When changes are committed to the Git repository and the playbook is executed, the updates are automatically synchronized to Cisco Catalyst Center.

## Device Targeting Hints

Templates can specify their target device type using an optional hint comment at the beginning of the `.j2` file:

```jinja
{## CATC: productFamily=Switches and Hubs, softwareType=IOS-XE ##}
```

| Parameter | Default Value | Supported Values |
|-----------|---------------|------------------|
| `productFamily` | `Switches and Hubs` | `Routers`, `Switches and Hubs`, `Wireless Controller` |
| `softwareType` | `IOS-XE` | `IOS-XE`, `IOS`, `NX-OS` |

When no hint is present, templates default to `Switches and Hubs` with `IOS-XE`. Default values can be customized in the playbook variables (`DEFAULT_PRODUCT_FAMILY`, `DEFAULT_SOFTWARE_TYPE`).

---

## Composite Template Synchronization

The playbook supports synchronization of **composite CLI templates** to Cisco Catalyst Center. Composite templates combine multiple regular templates into an ordered execution sequence, enabling modular template design and reuse.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Git Repository                                │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │  FABRIC-VRF.j2  │    │ FABRIC-EVPN.j2  │    │ FABRIC-NVE.j2   │ │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘ │
│           │                      │                      │          │
│           └──────────────────────┼──────────────────────┘          │
│                                  ▼                                  │
│                    ┌─────────────────────────┐                     │
│                    │  BGP-EVPN-BUILD.yml     │  ← Definition file  │
│                    │  (in same folder)       │    inside Git repo  │
│                    └─────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼ Ansible Playbook
┌─────────────────────────────────────────────────────────────────────┐
│                    Cisco Catalyst Center                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Composite Template: BGP-EVPN-BUILD.j2          │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │FABRIC-VRF│→│FABRIC-NVE│→│FABRIC-   │→│FABRIC-   │→ ...  │   │
│  │  │   .j2    │ │   .j2    │ │EVPN.j2   │ │OVERLAY.j2│       │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Definition File Structure

Place a YAML definition file alongside your templates in the Git repository. The playbook automatically discovers all `.yml` files in the synchronized repository:

```
CatalystCenter-BGP-EVPN-VXLAN/        # Git repository
└── BGP EVPN/                         # Template folder
    ├── FABRIC-VRF.j2
    ├── FABRIC-EVPN.j2
    ├── FABRIC-NVE.j2
    ├── ...
    └── BGP-EVPN-BUILD.yml            # Composite definition → creates BGP-EVPN-BUILD.j2
```

**Definition File Format:**

```yaml
# Optional: Override the composite template name (default: filename with .j2 extension)
# composite_name: "CUSTOM-COMPOSITE-NAME"

# Ordered list of templates to include in the composite
templates:
  - name: "FABRIC-VRF.j2"
  - name: "FABRIC-LOOPBACKS.j2"
  - name: "FABRIC-NVE.j2"
  - name: "FABRIC-MCAST.j2"
  - name: "FABRIC-EVPN.j2"
  - name: "FABRIC-OVERLAY.j2"
  - name: "FABRIC-IPSEC.j2"
  - name: "FABRIC-NAC-IOT.j2"
```

### Template Inclusion Guidelines

| Template Type | Include in Composite | Reason |
|---------------|----------------------|--------|
| `FABRIC-*.j2` | ✅ Yes | Top-level executable templates |
| `DEFN-*.j2`   | ❌ No  | Data definitions (included via Jinja2 `{% include %}`) |
| `FUNC-*.j2`   | ❌ No  | Macro libraries (included via Jinja2 `{% include %}`) |

> **Note:** Only include top-level templates that execute independently. Templates referenced via Jinja2 `{% include %}` statements are resolved at render time and should not be added to the composite.

### API Compliance

The playbook generates `containingTemplates` structures that conform to the Cisco Catalyst Center API export format:

```json
{
  "name": "FABRIC-VRF.j2",
  "id": "75ec3be5-7fd4-4110-aaef-071e1b768179",
  "composite": false,
  "language": "JINJA",
  "description": "description",
  "projectName": "CatalystCenter-BGP-EVPN-VXLAN",
  "deviceTypes": [{"productFamily": "Switches and Hubs"}],
  "templateParams": [],
  "tags": []
}
```

### Validation

The playbook performs the following validations before creating or updating composite templates:

- ✅ Verifies all referenced templates exist in Cisco Catalyst Center
- ✅ Confirms all template IDs are resolved successfully
- ⚠️ Skips composites with missing child templates (with warning)

---

## Compatibility Matrix

The `dnac_version` parameter in `credentials.yml` must match your Cisco Catalyst Center version. Use the corresponding Ansible collection and Python SDK versions:

| Cisco Catalyst Center | Ansible Collection (`cisco.dnac`) | Python SDK (`dnacentersdk`) |
|------------------|-----------------------------------|----------------------------|
| 2.3.5.3          | 6.13.3                            | 2.6.11                     |
| 2.3.7.6          | 6.25.0                            | 2.8.3                      |
| 2.3.7.7          | 6.30.2                            | 2.8.6                      |
| 2.3.7.9          | 6.33.2                            | 2.8.6                      |
| 3.1.3.0          | ≥6.36.0                           | ≥2.10.1                    |

> **Note:** For the latest compatibility information, refer to the official [Cisco DNA Center Ansible Collection documentation](https://github.com/cisco-en-programmability/dnacenter-ansible#compatibility-matrix).

## Requirements

- Python 3.9 or later
- Ansible 2.15 or later

### Installation

Create a Python virtual environment and install dependencies:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

### Configuration

1. Update `credentials.yml` with your Cisco Catalyst Center host and connection settings
2. Create `vault.yml` with your credentials and encrypt it:

   ```bash
   ansible-vault encrypt ansible-git-dnac/vault.yml
   ```

## Usage

Run the playbook with a vault password file:

```bash
ansible-playbook ansible-git-dnac/ansible-git-dnac.yml --vault-password-file .vault_pass
```

Alternatively, use an interactive vault password prompt:

```bash
ansible-playbook ansible-git-dnac/ansible-git-dnac.yml --ask-vault-pass
```

Enable debug mode for verbose output:

```bash
DEBUG=true ansible-playbook ansible-git-dnac/ansible-git-dnac.yml --vault-password-file .vault_pass
```

## Example Output

The screenshot below demonstrates the playbook's capabilities:

- Template files synchronized from the Git repository to Cisco Catalyst Center
- Automatically generated comment headers reflecting the Git diff from the previous version
- Template versioning with commit messages in human-readable format

[![Sample Run](https://github.com/imanassypov/Cisco-Catalyst-Center-Templates-Github-integration/blob/main/sample_run.png)](https://github.com/imanassypov/Cisco-Catalyst-Center-Templates-Github-integration/blob/main/sample_run.png)

## References

- [CI/CD Pipeline Demo with Cisco Catalyst Center](https://gitlab.com/oboehmer/dnac-template-as-code) by Oliver Boehmer — inspiration for this project
- [Cisco DNA Center Ansible Collection](https://cisco-en-programmability.github.io/dnacenter-ansible/main/plugins/index.html) — official Ansible Galaxy module documentation
- [Cisco Catalyst Center API Reference](https://developer.cisco.com/docs/dna-center/) — official API documentation
