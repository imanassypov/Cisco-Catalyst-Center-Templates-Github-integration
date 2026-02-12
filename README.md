# Ansible Playbook for Cisco Catalyst Center Template Synchronization

Automate the synchronization of Jinja2 templates between a Git repository and Cisco Catalyst Center Template Projects using GitOps workflows.

## Features

- Leverages the official [Cisco DNA Center Ansible Collection](https://galaxy.ansible.com/cisco/dnac) (`cisco.dnac`)
- Integrates with the [Cisco DNA Center Python SDK](https://github.com/cisco-en-programmability/dnacentersdk) (`dnacentersdk`)
- Clones a specified Git repository containing Jinja2 templates
- Synchronizes template content to Cisco Catalyst Center Template Projects
- Automatically adds Git commit messages to template version descriptions
- Embeds Git diff information as Jinja comments in template payloads for traceability
- **NEW**: Updated playbook using `cisco.dnac.template_workflow_manager` for streamlined template management
- **NEW**: Ansible inventory-based configuration for better organization and multi-environment support
- **NEW**: Dynamic template ordering based on composite definitions (no static ordering required)

## Refactored Architecture

The updated playbook (`ansible-git-catc.yml`) introduces several improvements:

### Benefits of Template Workflow Manager

- **Simplified Configuration**: Uses declarative configuration model instead of imperative API calls
- **Idempotent Operations**: Automatically handles create/update logic based on template existence
- **Batch Processing**: More efficient handling of multiple templates
- **Better Error Handling**: Improved error messages and validation
- **State Management**: Built-in state tracking (merged, deleted, etc.)

### Dynamic Template Ordering

- **No Static Lists**: Template processing order determined automatically from composite definitions
- **Dependency-Driven**: Templates referenced in composites are prioritized automatically
- **Flexible**: Works with any repository structure without configuration changes
- **Maintainable**: Add/remove templates without updating playbook code

### Inventory-Based Configuration

- **Centralized Settings**: All connection parameters and configuration in `inventory.yml`
- **Multi-Environment Support**: Easy to manage dev/test/prod environments with separate inventory files
- **Variable Inheritance**: Leverage Ansible's inventory variable precedence
- **Better Security**: Credentials separated in vault file, configuration in inventory

## Sample Repository

A sample template repository is available for reference:  
**Repository:** [CatalystCenter-BGP-EVPN-VXLAN](https://github.com/imanassypov/CatalystCenter-BGP-EVPN-VXLAN)  
**Branch:** `composite-template`

> **Note:** This is an example implementation. The playbook works with any Git repository containing Jinja2 templates.

When changes are committed to the Git repository and the playbook is executed, the updates are automatically synchronized to Cisco Catalyst Center.

## Device Targeting Hints

Templates can specify their target device type using an optional hint comment at the beginning of the `.j2` file:

```jinja
{## CATC: productFamily=Switches and Hubs, softwareType=IOS-XE, deviceType=Cisco Catalyst 9300 Switch ##}
```

| Parameter | Default Value | Supported Values |
|-----------|---------------|------------------|
| `productFamily` | `Switches and Hubs` | `Routers`, `Switches and Hubs`, `Wireless Controller` |
| `softwareType` | `IOS-XE` | `IOS-XE`, `IOS`, `NX-OS` |
| `deviceType` | *(see below)* | `Cisco Catalyst 9300 Switch`, `Cisco Catalyst 9500 Switch`, `Cisco ASR 1000 Series`, etc. |

**Default Device Types:** When no `deviceType` hint is specified, templates target multiple device series:
- Cisco Catalyst 9500 Series Switches
- Cisco Catalyst 9000 Series Virtual Switches
- Cisco Catalyst 9300 Series Switches
- Cisco Catalyst 9400 Series Switches

Default values can be customized in the inventory file (`default_device_types`, `default_software_type`). The updated playbook supports multiple device types per template:

```yaml
default_device_types:
  - product_family: "Switches and Hubs"
    product_series: "Cisco Catalyst 9500 Series Switches"
  - product_family: "Switches and Hubs"
    product_series: "Cisco Catalyst 9000 Series Virtual Switches"
  - product_family: "Switches and Hubs"
    product_series: "Cisco Catalyst 9300 Series Switches"
  - product_family: "Switches and Hubs"
    product_series: "Cisco Catalyst 9400 Series Switches"
```

## Git Diff Header

The playbook can optionally prepend Git diff information as Jinja comments at the top of each template. This provides traceability by showing the commit hash and changes from the previous version.

**Example diff header:**
```jinja
{## a1b2c3d4 ##}
{## diff --git a/Template-A.j2 b/Template-A.j2 ##}
{## @@ -1,5 +1,6 @@ ##}
{## +new line added ##}
```

**Configuration:** Set `INCLUDE_DIFF_HEADER` in the playbook vars:

| Value | Behavior |
|-------|----------|
| `false` (default) | Template content only, no diff header |
| `true` | Prepend Git diff as Jinja comments |

---

## Composite Template Synchronization

The playbook supports synchronization of **composite CLI templates** to Cisco Catalyst Center. Composite templates combine multiple regular templates into an ordered execution sequence, enabling modular template design and reuse.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Git Repository                                │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │  Template-A.j2  │    │  Template-B.j2  │    │  Template-C.j2  │ │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘ │
│           │                      │                      │          │
│           └──────────────────────┼──────────────────────┘          │
│                                  ▼                                  │
│                    ┌─────────────────────────┐                     │
│                    │  COMPOSITE-BUILD.yml    │  ← Definition file  │
│                    │  (in same folder)       │    inside Git repo  │
│                    └─────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼ Ansible Playbook
┌─────────────────────────────────────────────────────────────────────┐
│                    Cisco Catalyst Center                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Composite Template: COMPOSITE-BUILD.j2         │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │Template-A│→│Template-B│→│Template-C│→│Template-D│→ ...  │   │
│  │  │   .j2    │ │   .j2    │ │   .j2    │ │   .j2    │       │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Definition File Structure

Place a YAML definition file alongside your templates in the Git repository. The playbook automatically discovers all `.yml` files in the synchronized repository:

```
MyTemplateRepository/                  # Git repository
└── Templates/                         # Template folder
    ├── Template-A.j2
    ├── Template-B.j2
    ├── Template-C.j2
    ├── ...
    └── COMPOSITE-BUILD.yml            # Composite definition → creates COMPOSITE-BUILD.j2
```

**Definition File Format:**

```yaml
# Optional: Override the composite template name (default: filename with .j2 extension)
# composite_name: "CUSTOM-COMPOSITE-NAME"

# Ordered list of templates to include in the composite
templates:
  - name: "Template-A.j2"
  - name: "Template-B.j2"
  - name: "Template-C.j2"
  - name: "Template-D.j2"
  - name: "Template-E.j2"
  - name: "Template-F.j2"
```

### Template Inclusion Guidelines

| Template Type | Include in Composite | Reason |
|---------------|----------------------|--------|
| Top-level config templates | ✅ Yes | Executable templates that run independently |
| Definition files | ❌ No  | Data definitions (included via Jinja2 `{% include %}`) |
| Macro libraries | ❌ No  | Function libraries (included via Jinja2 `{% include %}`) |

> **Note:** Only include top-level templates that execute independently. Templates referenced via Jinja2 `{% include %}` statements are resolved at render time and should not be added to the composite.

### API Compliance

The playbook generates `containingTemplates` structures that conform to the Cisco Catalyst Center API export format:

```json
{
  "name": "Template-A.j2",
  "id": "75ec3be5-7fd4-4110-aaef-071e1b768179",
  "composite": false,
  "language": "JINJA",
  "description": "description",
  "projectName": "MyTemplateProject",
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

### Composite Payload Creation Flow

The playbook creates composite template payloads through the following stages:

1. **Discovery** – Recursively finds all `.yml` definition files in the synced Git repository

2. **Load Definitions** – Parses each YAML file to extract the composite name and ordered template list:
   ```yaml
   composite_definitions: [{
     'name': 'COMPOSITE-BUILD.j2',
     'definition_path': '/path/to/COMPOSITE-BUILD.yml',
     'template_names': ['Template-A.j2', 'Template-B.j2', ...]
   }]
   ```

3. **Prepare API List** – Builds initial structure with template names and determines if composite exists:
   ```yaml
   composite_api_list: [{
     'name': 'COMPOSITE-BUILD.j2',
     'existing_id': '...',  # or '' if new
     'is_new': true/false,
     'containingTemplates': [{'name': 'Template-A.j2', 'composite': false}, ...]
   }]
   ```

4. **Resolve IDs** – The included `composite-resolve-ids.yml` resolves each template name to its Catalyst Center ID:
   ```yaml
   resolved_templates: [{
     'name': 'Template-A.j2',
     'id': '75ec3be5-7fd4-4110-aaef-071e1b768179',
     'composite': false,
     'language': 'JINJA',
     ...
   }]
   ```

5. **API Call** – Creates or updates the composite template with fully resolved `containingTemplates`

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

#### 1. Configure Ansible Inventory

Update `ansible-git-catc/inventory.yml` with your Cisco Catalyst Center host and Git repository settings:

```yaml
all:
  hosts:
    catalyst_center:
      ansible_host: your-dnac-host.example.com
      dnac_host: your-dnac-host.example.com
      dnac_port: 443
      dnac_version: 2.3.7.6
      git_repo: "https://github.com/your-org/your-template-repo.git"
      git_branch: "main"
```

#### 2. Configure Credentials

Create `vault.yml` from the example template and add your credentials:

```bash
cp ansible-git-catc/vault.yml.example ansible-git-catc/vault.yml
# Edit vault.yml with your credentials
ansible-vault encrypt ansible-git-catc/vault.yml
```

## Usage

### Using the Updated Playbook

```bash
ansible-playbook -i ansible-git-catc/inventory.yml ansible-git-catc/ansible-git-catc.yml --vault-password-file .vault_pass
```

Alternatively, use an interactive vault password prompt:

```bash
ansible-playbook -i ansible-git-catc/inventory.yml ansible-git-catc/ansible-git-catc.yml --ask-vault-pass
```

Enable debug mode for verbose output:

```bash
DEBUG=true ansible-playbook -i ansible-git-catc/inventory.yml ansible-git-catc/ansible-git-catc.yaml --vault-password-file .vault_pass
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

## Author

**Igor Manassypov**  
Solutions Architect, Cisco Systems  
📧 [imanassy@cisco.com](mailto:imanassy@cisco.com)
