# Copilot Instructions for ansible-git-catc (Updated Architecture)

## Project Overview
GitOps automation for Cisco Catalyst Center (CatC) templates. Syncs Jinja2 templates from Git → CatC Template Projects with auto-versioning using Git commit metadata. **Refactored to use `cisco.dnac.template_workflow_manager`** for streamlined, declarative template management.

## Architecture
**Main playbook**: [ansible-git-catc/ansible-git-catc.yml](ansible-git-catc/ansible-git-catc.yml)  
**Configuration**: [ansible-git-catc/inventory.yml](ansible-git-catc/inventory.yml)  
**Task modules**: 
- [ansible-git-catc/process-template.yml](ansible-git-catc/process-template.yml) - Individual template processing
- [ansible-git-catc/process-composite.yml](ansible-git-catc/process-composite.yml) - Composite template processing

**Flow**: Clone Git repo → Sort templates by dependency order → Process templates → Build workflow configs → Sync to CatC via `template_workflow_manager` → Process composites → Sync composites

## Key Changes from Legacy Architecture

### Template Workflow Manager Benefits
- **Declarative configuration** - Define desired state, module handles creation/updates
- **Idempotent operations** - Automatically detects changes and updates only when needed
- **Batch processing** - More efficient API calls for multiple templates
- **Built-in state management** - `state: merged` handles create/update logic
- **Better error handling** - Improved validation and error messages

### Inventory-Based Configuration
Replaces the old `credentials.yml` with Ansible inventory pattern:
```yaml
all:
  hosts:
    catalyst_center:
      dnac_host: dnac.example.com
      dnac_version: 2.3.7.9
      git_repo: "https://github.com/org/repo.git"
      git_branch: "main"
```

**Benefits**:
- Multi-environment support (dev/test/prod inventory files)
- Variable inheritance and precedence
- Centralized configuration management
- Separation of concerns (config vs. credentials)

### Template Processing Order
**CRITICAL**: Templates must be processed in dependency order to avoid CatC errors.

The playbook uses **dynamic ordering** based on composite template definitions:

```yaml
# Composite definitions (.yml files) automatically define processing order:
# 1. Templates referenced in composites are processed first (priority)
# 2. Templates not in composites are processed second (regular)
# 3. Composites are processed last

# Example: COMPOSITE-BUILD.yml references these templates
templates:
  - name: "Template-A.j2"      # Priority template (in composite)
  - name: "Template-B.j2"      # Priority template (in composite)
  - name: "Template-C.j2"      # Priority template (in composite)

# HelperLibrary.j2 is not in any composite → Regular template (processed after priority)
```

**Benefits**:
- No static `template_order` dictionary to maintain
- Add/remove templates without updating playbook code
- Composite definitions serve dual purpose: define composites AND dependencies
- Automatic fallback: if no composites exist, all templates processed as regular

## File Structure
| File | Purpose |
|------|---------|
| `ansible-git-catc/ansible-git-catc.yml` | Main playbook using template_workflow_manager |
| `ansible-git-catc/process-template.yml` | Included task to process individual templates |
| `ansible-git-catc/process-composite.yml` | Included task to process composite templates |
| `ansible-git-catc/inventory.yml` | Inventory file with CatC connection & Git config |
| `ansible-git-catc/vault.yml` | Encrypted credentials (username/password) |
| `ansible-git-catc/vault.yml.example` | Template for vault.yml |
| `CatC Templates/*/` | Cloned template repos (auto-populated at runtime) |

**Removed files** (from old architecture):
- ~~`ansible-git-catc/credentials.yml`~~ → Replaced by `inventory.yml`
- ~~`ansible-git-catc/composite-resolve-ids.yml`~~ → Logic integrated into workflow manager

## Configuration Files

### inventory.yml
Centralized configuration for Catalyst Center connection and Git settings:

```yaml
all:
  hosts:
    catalyst_center:
      ansible_host: localhost
      ansible_connection: local
      
      # Cisco Catalyst Center Connection
      dnac_host: dnac.example.com
      dnac_username: admin  # Defined in vault.yml
      dnac_port: 443
      dnac_version: 2.3.7.9
      dnac_verify: false
      dnac_debug: true
      dnac_log_level: INFO
      dnac_log: true
      
      # Git Repository
      git_repo: "https://github.com/org/repo.git"
      git_branch: "main"
      git_local_dest: "{{ playbook_dir }}/../CatC Templates/RepoName"
      
      # Template Settings
      template_extension: "j2"
      include_diff_header: false
      default_software_type: "IOS"
      default_software_variant: "XE"
      default_template_version: "1.0"
      default_device_types:
        - product_family: "Switches and Hubs"
          product_series: "Cisco Catalyst 9500 Series Switches"
      catc_template_summary_maxchar: 1024
```

### vault.yml
Credentials only (encrypted with ansible-vault):
```yaml
dnac_username: "admin"
dnac_password: "your_password_here"
```

## Module Pattern - Template Workflow Manager

The playbook uses `module_defaults` to set connection parameters for all `template_workflow_manager` calls:

```yaml
module_defaults:
  cisco.dnac.template_workflow_manager:
    dnac_host: "{{ dnac_host }}"
    dnac_username: "{{ dnac_username }}"
    dnac_password: "{{ dnac_password }}"
    dnac_verify: "{{ dnac_verify }}"
    dnac_port: "{{ dnac_port }}"
    dnac_version: "{{ dnac_version }}"
    dnac_debug: "{{ dnac_debug }}"
    dnac_log_level: "{{ dnac_log_level }}"
    dnac_log: "{{ dnac_log }}"
```

When using the workflow manager, simply call:
```yaml
- name: Synchronize templates
  cisco.dnac.template_workflow_manager:
    state: merged
    config: "{{ template_workflow_configs }}"
```

## Template Workflow Configuration Structure

Templates are processed into workflow manager configuration format:

```yaml
template_workflow_configs:
  - configuration_templates:
      template_name: "Template-A.j2"
      project_name: "MyTemplateProject"
      language: "JINJA"
      template_content: "{{ template_content }}"
      template_description: "Template synced from Git repository"
      device_types: "{{ default_device_types }}"
      software_type: "{{ default_software_type }}"
      software_variant: "{{ default_software_variant }}"
      software_version: null
      template_params: []
      failure_policy: "ABORT_TARGET_ON_ERROR"
      version: "{{ default_template_version }}"
      tags: []
```

## Template Conventions

### File Naming
- Top-level config templates (e.g., `Config-*.j2`) – Executable templates (included in composites)
- Definition templates (e.g., `Def-*.j2`) – Data definitions (included via Jinja `{% include %}`)
- Function libraries (e.g., `Lib-*.j2`) – Macro libraries (included via Jinja `{% include %}`)

### Device Targeting Hint
First line of `.j2` files—parsed via regex:
```jinja
{## CATC: productFamily=Switches and Hubs, softwareType=IOS-XE, deviceType=Cisco Catalyst 9300 Switch ##}
```

Defaults configured in `inventory.yml`:
- `default_software_type`: `"IOS"`
- `default_software_variant`: `"XE"`
- `default_device_types`: List of device type objects

### Include Pattern
Templates reference others via project-relative includes:
```jinja
{% include "{{ TEMPLATE_PROJECT_NAME }}/HelperTemplate.j2" %}
```

The playbook replaces `{{ TEMPLATE_PROJECT_NAME }}` with the actual project name.

## Composite Templates

Defined via `.yml` files in the Git repo:

```yaml
# Optional: Override composite name (default: filename with .j2 extension)
# composite_name: "CUSTOM-NAME.j2"

templates:
  - name: "Template-A.j2"
  - name: "Template-B.j2"
  - name: "Template-C.j2"
```

**Only list top-level executable templates**—definition and function library templates are resolved at render time.

### Composite Workflow Configuration

```yaml
composite_workflow_configs:
  - configuration_templates:
      template_name: "COMPOSITE-BUILD.j2"
      project_name: "MyTemplateProject"
      composite: true
      language: "JINJA"
      template_content: ""
      template_description: "Composite template synced from Git repository"
      device_types: "{{ default_device_types }}"
      software_type: "{{ default_software_type }}"
      software_variant: "{{ default_software_variant }}"
      containing_templates: "{{ containing_templates_list }}"
      version: "{{ default_template_version }}"
```

## Commands

```bash
# Setup
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

# Create vault file from example
cp ansible-git-catc/vault.yml.example ansible-git-catc/vault.yml
# Edit vault.yml with credentials, then encrypt:
ansible-vault encrypt ansible-git-catc/vault.yml

# Run with inventory
ansible-playbook -i ansible-git-catc/inventory.yml ansible-git-catc/ansible-git-catc.yml --vault-password-file .vault_pass

# Or with interactive vault prompt
ansible-playbook -i ansible-git-catc/inventory.yml ansible-git-catc/ansible-git-catc.yml --ask-vault-pass

# Debug mode (verbose output)
DEBUG=true ansible-playbook -i ansible-git-catc/inventory.yml ansible-git-catc/ansible-git-catc.yml --vault-password-file .vault_pass
```

## Version Compatibility

Match `dnac_version` in `inventory.yml` to your Catalyst Center version:

| CatC Version | `cisco.dnac` Collection | `dnacentersdk` | Notes |
|--------------|-------------------------|----------------|-------|
| 2.3.5.3 | 6.13.3 | 2.6.11 | Legacy |
| 2.3.7.6 | 6.25.0 | 2.8.3 | Stable |
| 2.3.7.9 | 6.33.2 - 6.46.0 | 2.8.6 | Current recommended |
| 3.1.3.0 | ≥6.36.0 | ≥2.10.1 | Latest |

**Current setup** (updated_playbook branch):
- `cisco.dnac`: 6.46.0
- Compatible with CatC 2.3.7.9

## Workflow Patterns

### Processing Individual Templates
`process-template.yml` is included with loop:
```yaml
- name: Process regular templates in correct order
  include_tasks: process-template.yml
  with_items: "{{ sorted_template_files }}"
  loop_control:
    loop_var: template_file
    index_var: template_idx
```

Each iteration:
1. Extracts template metadata (name, content, commit message)
2. Optionally prepends Git diff header
3. Builds workflow configuration object
4. Appends to `template_workflow_configs` list

### Processing Composite Templates
`process-composite.yml` is included with loop:
```yaml
- name: Process composite templates
  include_tasks: process-composite.yml
  with_items: "{{ composite_definition_files.files }}"
  loop_control:
    loop_var: composite_file
```

Each iteration:
1. Loads YAML definition
2. Extracts composite name and template list
3. Builds `containing_templates` list with metadata
4. Appends to `composite_workflow_configs` list
5. Resets `containing_templates_list` for next composite

### Synchronization
Templates and composites are synced in separate calls:
```yaml
# Sync regular templates first
- name: Synchronize all templates
  cisco.dnac.template_workflow_manager:
    state: merged
    config: "{{ template_workflow_configs }}"

# Then sync composites (which reference the regular templates)
- name: Synchronize composite templates
  cisco.dnac.template_workflow_manager:
    state: merged
    config: "{{ composite_workflow_configs }}"
```

## Key Variables

| Variable | Set In | Purpose |
|----------|--------|---------|
| `projectName` | Runtime (from Git repo structure) | CatC project name |
| `sorted_template_files` | Runtime (sorted by `template_order`) | Ordered template list |
| `template_workflow_configs` | Built incrementally | List of template configs |
| `composite_workflow_configs` | Built incrementally | List of composite configs |
| `containing_templates_list` | Per-composite (reset after each) | Child templates for composite |

## Debug Mode
Set `DEBUG=true` environment variable or set `DEBUG: true` in playbook vars:
```yaml
vars:
  DEBUG: "{{ lookup('env', 'DEBUG') | default(false) | bool }}"
```

Debug blocks show:
- Execution timestamp
- Project name
- Sorted template list
- Template workflow configurations
- Composite workflow configurations
- Sync results

## Best Practices

1. **Use composite definitions** - Let `.yml` files define template dependencies
2. **Test with DEBUG=true** - Review configurations before sync
3. **Use inventory for environments** - Create separate inventory files for dev/test/prod
4. **Encrypt vault files** - Always use `ansible-vault encrypt` for credentials
5. **Version pinning** - Match collection versions to CatC version in requirements.yml
6. **Meaningful commits** - Commit messages appear in template descriptions
7. **Composites last** - Always sync individual templates before composites

## Troubleshooting

### Template sync fails
- Enable DEBUG mode to see configuration payloads
- Check API version compatibility (dnac_version in inventory)
- Verify templates referenced in composites exist individually in CatC

### Composite templates missing references
- Ensure referenced templates exist in CatC before syncing composite
- Check `containing_templates_list` in debug output
- Verify template names match exactly (case-sensitive)

### Connection errors
- Verify `dnac_host`, `dnac_port`, `dnac_version` in inventory.yml
- Check credentials in vault.yml
- Test with `dnac_verify: false` for self-signed certificates

### Version compatibility issues
- Check compatibility matrix - match collection version to CatC version
- Verify `dnacentersdk` version in requirements.txt
- Update requirements.yml if needed
