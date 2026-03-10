# Ansible GitOps for Cisco Catalyst Center Templates

Automates synchronization of Jinja2 templates from a GitHub repository directly into Cisco Catalyst Center (CatC) Template Projects — with composite template support, automatic dependency ordering, and Git commit metadata in version descriptions.

---

## Table of Contents

1. [Features](#features)
2. [Architecture Overview](#architecture-overview)
3. [File Structure](#file-structure)
4. [Detailed Processing Flow](#detailed-processing-flow)
   - [Stage 1 — Fetch Repository Tree](#stage-1--fetch-repository-tree)
   - [Stage 2 — Fetch File Contents & Commit Metadata](#stage-2--fetch-file-contents--commit-metadata)
   - [Stage 3 — Dynamic Template Ordering](#stage-3--dynamic-template-ordering)
   - [Stage 4 — Build Workflow Configurations](#stage-4--build-workflow-configurations)
   - [Stage 5 — Sync to Catalyst Center](#stage-5--sync-to-catalyst-center)
5. [process-template.yml Logic](#process-templateyml-logic)
6. [process-composite.yml Logic](#process-compositeyml-logic)
7. [Template Conventions](#template-conventions)
8. [Composite Template Definitions](#composite-template-definitions)
9. [Configuration](#configuration)
10. [Setup & Usage](#setup--usage)
11. [Compatibility Matrix](#compatibility-matrix)
12. [Troubleshooting](#troubleshooting)
13. [References](#references)

---

## Features

- **No local clone required** — fetches all content via GitHub REST API and raw URLs at runtime
- **Declarative sync** via `cisco.dnac.template_workflow_manager` — idempotent create/update
- **Dynamic dependency ordering** — template processing order derived automatically from composite definitions; no static lists to maintain
- **Composite template support** — syncs ordered multi-template composites with full `containingTemplates` resolution
- **Git metadata in descriptions** — commit timestamp, message, and author written to template description field
- **Optional diff headers** — Git patch embedded as Jinja2 comments for traceability
- **Inventory-based configuration** — `inventory.yml` centralizes all connection and repo settings; supports multiple environments
- **Vault-encrypted credentials** — username/password stored and encrypted separately in `vault.yml`

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         GitHub Repository                            │
│                                                                      │
│   BGP EVPN/                                                          │
│   ├── DEFN-VRF.j2          ← Data definition (Jinja include)        │
│   ├── FUNC-OBJECT-MACROS.j2← Macro library (Jinja include)          │
│   ├── FABRIC-NVE.j2        ← Top-level config template              │
│   ├── FABRIC-VRF.j2        ← Top-level config template              │
│   ├── ...                                                            │
│   └── BGP-EVPN-BUILD.yml   ← Composite definition file              │
└─────────────────────────────┬────────────────────────────────────────┘
                              │  GitHub REST API + raw.githubusercontent.com
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    ansible-git-catc.yml (Ansible)                    │
│                                                                      │
│  1. Fetch file tree via GitHub API                                   │
│  2. Download .j2 content + .yml composite definitions                │
│  3. Fetch commit metadata per file                                   │
│  4. Order templates: [regular] → [priority] → [composites]          │
│  5. Build template_workflow_configs + composite_workflow_configs     │
│  6. Sync via cisco.dnac.template_workflow_manager                    │
└─────────────────────────────┬────────────────────────────────────────┘
                              │  HTTPS REST API
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Cisco Catalyst Center                             │
│                                                                      │
│   Project: CLUS26-Site1                                              │
│   ├── DEFN-VRF.j2          (regular template)                       │
│   ├── FUNC-OBJECT-MACROS.j2(regular template)                       │
│   ├── FABRIC-NVE.j2        (priority template)                      │
│   ├── FABRIC-VRF.j2        (priority template)                      │
│   ├── ...                                                            │
│   └── BGP-EVPN-BUILD.j2    (composite template)                     │
│       └── containingTemplates: [FABRIC-VRF, FABRIC-NVE, ...]        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## File Structure

```
.
├── ansible.cfg                # Auto-loads inventory.yml (no -i flag needed)
├── requirements.txt           # Python dependencies (dnacentersdk, ansible)
├── requirements.yml           # Ansible Galaxy collection dependencies (cisco.dnac, community.general)
├── ansible-git-catc.yml       # Main playbook
├── process-template.yml       # Included task: builds one regular template config object
├── process-composite.yml      # Included task: builds one composite template config object
├── inventory.yml              # CatC connection parameters + Git repo configuration
├── vault.yml                  # Encrypted credentials — gitignored, never commit
└── vault.yml.example          # Vault template — safe to commit
```

---

## Detailed Processing Flow

### Full Playbook Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ansible-git-catc.yml                                                           │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ STAGE 1 — Fetch Repository Tree                                          │   │
│  │                                                                          │   │
│  │  Parse git_repo URL → git_repo_slug                                      │   │
│  │  Build optional Authorization header (git_token)                        │   │
│  │  GET /repos/{slug}/git/trees/{branch}?recursive=1                       │   │
│  │       │                                                                  │   │
│  │       ├─ filter path starts with git_repo_subfolder/                    │   │
│  │       ├─ filter *.j2  → api_template_files[]                            │   │
│  │       └─ filter *.yml → api_composite_files[]                           │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                  │                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ STAGE 2 — Fetch File Contents & Commit Metadata                         │   │
│  │                                                                          │   │
│  │  For each .j2 file:                                                      │   │
│  │    GET raw.githubusercontent.com/.../file.j2   → template content       │   │
│  │    GET /repos/{slug}/commits?path=file.j2      → last commit info       │   │
│  │    (optional) GET /repos/{slug}/commits/{sha}  → diff patch             │   │
│  │                                                                          │   │
│  │  For each .yml file:                                                     │   │
│  │    GET raw.githubusercontent.com/.../file.yml  → composite definition   │   │
│  │                                                                          │   │
│  │  Assembled into:                                                         │   │
│  │    enriched_template_files[]  +  enriched_composite_files[]             │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                  │                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ STAGE 3 — Dynamic Template Ordering                                     │   │
│  │                                                                          │   │
│  │  Parse each .yml composite definition → extract referenced template names│  │
│  │    composite_referenced_templates = [FABRIC-VRF.j2, FABRIC-NVE.j2, …]  │   │
│  │                                                                          │   │
│  │  For each enriched template:                                             │   │
│  │    name IN composite_referenced_templates  → priority_template_list[]   │   │
│  │    name NOT IN composite_referenced_templates → regular_template_list[] │   │
│  │                                                                          │   │
│  │  sorted_template_files = regular_template_list + priority_template_list │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                  │                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ STAGE 4 — Build Workflow Configurations                                  │   │
│  │                                                                          │   │
│  │  include_tasks: process-template.yml  (loop over sorted_template_files) │   │
│  │    → builds template_workflow_configs[]                                  │   │
│  │                                                                          │   │
│  │  include_tasks: process-composite.yml (loop over enriched_composite_files)│  │
│  │    → builds composite_workflow_configs[]                                 │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                  │                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ STAGE 5 — Sync to Catalyst Center                                       │   │
│  │                                                                          │   │
│  │  cisco.dnac.template_workflow_manager                                    │   │
│  │    state: merged                                                         │   │
│  │    config: template_workflow_configs   ← individual templates first     │   │
│  │                                                                          │   │
│  │  cisco.dnac.template_workflow_manager                                    │   │
│  │    state: merged                                                         │   │
│  │    config: composite_workflow_configs  ← composites after               │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Stage 1 — Fetch Repository Tree

```
inventory.yml
  git_repo: "https://github.com/org/repo.git"
  git_branch: "main"
  git_repo_subfolder: "BGP EVPN"           ← optional subfolder filter
  git_token: (from vault, optional)
         │
         ▼
  regex_replace → git_repo_slug = "org/repo"
         │
         ▼
  GET https://api.github.com/repos/org/repo/git/trees/main?recursive=1
         │
         ▼
  repo_tree_response.json.tree[]
         │
         ├── selectattr path starts with "BGP EVPN/"
         ├── selectattr path matches "\.j2$"   → api_template_files[]
         └── selectattr path matches "\.yml$"  → api_composite_files[]
```

---

### Stage 2 — Fetch File Contents & Commit Metadata

```
api_template_files[]                    api_composite_files[]
       │                                         │
       ▼                                         ▼
GET raw.githubusercontent.com           GET raw.githubusercontent.com
    /{slug}/{branch}/{path}                 /{slug}/{branch}/{path}
       │                                         │
       ▼                                         ▼
template_content_results[]              composite_content_results[]
       │
       ▼
GET /repos/{slug}/commits
    ?path={file}&per_page=1&sha={branch}
       │
       ▼
template_commit_results[]
  └── json[0].commit.author.date
  └── json[0].commit.message
  └── json[0].commit.author.name
       │
       ▼ (if include_diff_header: true)
GET /repos/{slug}/commits/{sha}
       │
       ▼
template_diff_results[]
  └── json.files[].patch

       │                                         │
       ▼                                         ▼
enriched_template_files[]               enriched_composite_files[]
  - name: "FABRIC-NVE.j2"                 - name: "BGP-EVPN-BUILD.yml"
  - path: "BGP EVPN/FABRIC-NVE.j2"        - path: "BGP EVPN/BGP-EVPN-BUILD.yml"
  - content: "..."                         - content: "templates:\n  - name: ..."
  - commit_message: "2026-01-01 | ..."
  - diff_content: "@@ -1,5 +1,6 ..."
```

---

### Stage 3 — Dynamic Template Ordering

The playbook avoids any static `template_order` dictionary. Processing order is derived entirely from composite definitions found in the repository:

```
enriched_composite_files[]
  └── BGP-EVPN-BUILD.yml content (YAML parsed)
        templates:
          - name: "FABRIC-VRF.j2"       ┐
          - name: "FABRIC-LOOPBACKS.j2"  │
          - name: "FABRIC-NVE.j2"        ├─ composite_referenced_templates[]
          - name: "FABRIC-MCAST.j2"      │
          - name: "FABRIC-EVPN.j2"       │
          - name: "FABRIC-OVERLAY.j2"    │
          - name: "FABRIC-NAC-IOT.j2"    ┘

enriched_template_files[]   (all 19 templates)
           │
           ▼
  For each template:
  ┌─────────────────────────────────────────────┐
  │  name IN composite_referenced_templates?    │
  │                                             │
  │  YES → priority_template_list[]             │
  │   (FABRIC-VRF, FABRIC-NVE, etc.)           │
  │                                             │
  │  NO  → regular_template_list[]             │
  │   (DEFN-*, FUNC-*)                         │
  └─────────────────────────────────────────────┘
           │
           ▼
  sorted_template_files =
    regular_template_list[]    ← processed FIRST (DEFN-*, FUNC-*)
    + priority_template_list[] ← processed SECOND (FABRIC-*)

  WHY THIS ORDER:
  - DEFN-* and FUNC-* templates must exist in CatC before FABRIC-* templates
    can include them via {% include "ProjectName/DEFN-VRF.j2" %}
  - FABRIC-* templates must exist before the composite can reference them
  - Composites are processed entirely separately after both lists
```

---

### Stage 4 — Build Workflow Configurations

```
sorted_template_files[]
  └── loop → include_tasks: process-template.yml
                    │
                    ▼
        template_workflow_configs[] (grows each iteration)

enriched_composite_files[]
  └── loop → include_tasks: process-composite.yml
                    │
                    ▼
        composite_workflow_configs[] (grows each iteration)
```

---

### Stage 5 — Sync to Catalyst Center

```
template_workflow_configs[]                composite_workflow_configs[]
  [{                                          [{
    configuration_templates: {                 configuration_templates: {
      template_name: "DEFN-VRF.j2"              template_name: "BGP-EVPN-BUILD.j2"
      project_name:  "CLUS26-Site1"             project_name:  "CLUS26-Site1"
      language:      "JINJA"                    composite:     true
      template_content: "..."                   containing_templates: [
      device_types:  [...]                        {name: "FABRIC-VRF.j2", ...},
      software_type: "IOS"                        {name: "FABRIC-NVE.j2", ...},
      ...                                         ...
    }                                           ]
  }, ...]                                    }
       │                                   }]
       │                                       │
       ▼  (call 1)                             ▼  (call 2 — after call 1 completes)
cisco.dnac.template_workflow_manager    cisco.dnac.template_workflow_manager
  state: merged                           state: merged
       │                                       │
       ▼                                       ▼
  Creates or updates each              Creates or updates composite
  individual template in CatC          (child templates guaranteed to
                                        exist from call 1)
```

---

## process-template.yml Logic

Called once per template in the sorted loop:

```
template_file (loop variable)
  ├── .name          "FABRIC-NVE.j2"
  ├── .content       raw Jinja2 content from GitHub
  ├── .commit_message "2026-01-15 | Fix NVE config [Igor M]"
  └── .diff_content  "@@ -10,5 +10,6 @@..." (or empty string)
         │
         ▼
  1. REPLACE {{ TEMPLATE_PROJECT_NAME }} → projectName
     (resolves cross-project Jinja include paths at build time)

  2. (if include_diff_header: true)
     wrap diff lines in {## ... ##} Jinja comments
     → diff_content_raw

  3. Assemble:
     template_content = diff_content_raw + template_content_raw

  4. Build template_config object:
     {
       configuration_templates: {
         template_name:        "FABRIC-NVE.j2"
         project_name:         "CLUS26-Site1"
         language:             "JINJA"
         template_content:     <assembled content>
         template_description: "Template synced from Git ... | commit message"
         device_types:         [{ product_family, product_series }, ...]
         software_type:        "IOS"
         software_variant:     "XE"
         software_version:     null
         template_params:      []
         failure_policy:       "ABORT_TARGET_ON_ERROR"
         version:              "1.0"
         tags:                 []
       }
     }

  5. Append to template_workflow_configs[]
```

---

## process-composite.yml Logic

Called once per composite definition file:

```
composite_file (loop variable)
  ├── .name     "BGP-EVPN-BUILD.yml"
  └── .content  YAML text
         │
         ▼
  1. Parse YAML content → composite_def
     {
       # composite_name: "CUSTOM-NAME.j2"  ← optional override
       templates:
         - name: "FABRIC-VRF.j2"
         - name: "FABRIC-NVE.j2"
         ...
     }

  2. Resolve composite name:
     composite_name = composite_def.composite_name
                   ?? filename with .yml → .j2
                   = "BGP-EVPN-BUILD.j2"

  3. Build containing_templates_list[]
     For each template name in composite_def.templates:
       {
         name:             "FABRIC-VRF.j2"
         composite:        false
         project_name:     "CLUS26-Site1"
         language:         "JINJA"
         description:      "description"
         device_types:     [...]
         software_type:    "IOS"
         software_variant: "XE"
         templateParams:   []
         tags:             []
       }

  4. Build composite_config object:
     {
       configuration_templates: {
         template_name:         "BGP-EVPN-BUILD.j2"
         project_name:          "CLUS26-Site1"
         composite:             true
         language:              "JINJA"
         template_content:      ""
         template_description:  "Composite template synced from Git repository"
         device_types:          [...]
         software_type:         "IOS"
         software_variant:      "XE"
         containing_templates:  <containing_templates_list>
         version:               "1.0"
         tags:                  []
       }
     }

  5. Append to composite_workflow_configs[]

  6. Reset containing_templates_list = []
     (prepare for next composite iteration)
```

---

## Template Conventions

### File Naming

| Prefix | Purpose | Include in Composite Definition? |
|--------|---------|----------------------------------|
| `DEFN-*.j2` | Data definitions — sets Jinja2 variables | ❌ No — included via `{% include %}` at render time |
| `FUNC-*.j2` | Macro/function libraries | ❌ No — included via `{% include %}` at render time |
| `FABRIC-*.j2` | Top-level executable config templates | ✅ Yes — listed in composite `.yml` |

### Device Targeting Hint

Optional first-line comment parsed by the playbook:

```jinja
{## CATC: productFamily=Switches and Hubs, softwareType=IOS-XE, productSeries=Cisco Catalyst 9300 Series Switches ##}
```

When absent, `default_device_types` from `inventory.yml` is used.

### Cross-Template Includes

Templates reference others using the project name as a path prefix. The playbook substitutes the actual project name at build time:

```jinja
{% include "{{ TEMPLATE_PROJECT_NAME }}/DEFN-VRF.j2" %}
```

`{{ TEMPLATE_PROJECT_NAME }}` is replaced with `projectName` from `inventory.yml` (e.g., `CLUS26-Site1`).

---

## Composite Template Definitions

A `.yml` file placed anywhere inside `git_repo_subfolder` defines one composite template:

```yaml
# BGP-EVPN-BUILD.yml

# Optional: override the composite name in CatC
# composite_name: "CUSTOM-COMPOSITE.j2"

# List top-level FABRIC-* templates only.
# DEFN-* and FUNC-* are resolved via Jinja {% include %} — do NOT add them here.
templates:
  - name: "FABRIC-VRF.j2"
  - name: "FABRIC-LOOPBACKS.j2"
  - name: "FABRIC-L3OUT.j2"
  - name: "FABRIC-NVE.j2"
  - name: "FABRIC-MCAST.j2"
  - name: "FABRIC-EVPN.j2"
  - name: "FABRIC-OVERLAY.j2"
  - name: "FABRIC-NAC-IOT.j2"
```

The composite is created in CatC as `BGP-EVPN-BUILD.j2` (filename `.yml` → `.j2`).

---

## Configuration

### ansible.cfg

Located in the playbook directory — eliminates the need for `-i` on every run:

```ini
[defaults]
inventory = inventory.yml
```

> **Note:** The directory must not be world-writable (`chmod o-w .`) or Ansible will ignore `ansible.cfg` for security reasons.

### inventory.yml

All connection and repository parameters:

```yaml
all:
  hosts:
    catalyst_center:
      ansible_host: localhost
      ansible_connection: local
      ansible_python_interpreter: "{{ ansible_playbook_python }}"

      # Catalyst Center connection
      dnac_host: dnac.example.com
      dnac_port: 443
      dnac_version: 2.3.7.6
      dnac_verify: false
      dnac_debug: true
      dnac_log_level: INFO
      dnac_log: true

      # Git repository
      git_repo: "https://github.com/org/repo.git"
      git_branch: "main"
      git_repo_subfolder: "BGP EVPN"   # subfolder within repo (leave blank for root)
      # git_token: (define in vault.yml for private repos or to raise API rate limits)

      # CatC project name (overrides name derived from folder path)
      projectName: "MyProject"

      # Template defaults
      template_extension: "j2"
      include_diff_header: false
      default_software_type: "IOS"
      default_software_variant: "XE"
      default_template_version: "1.0"
      catc_template_summary_maxchar: 1024

      default_device_types:
        - product_family: "Switches and Hubs"
          product_series: "Cisco Catalyst 9500 Series Switches"
        - product_family: "Switches and Hubs"
          product_series: "Cisco Catalyst 9300 Series Switches"
```

### vault.yml

Store credentials only — encrypt with `ansible-vault`:

```yaml
dnac_username: "admin"
dnac_password: "your_password"
# git_token: "ghp_..."   # optional, for private repos
```

---

## Setup & Usage

### Prerequisites

```bash
# Install Python dependencies
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Install Ansible Galaxy collections
ansible-galaxy collection install -r requirements.yml
```

### Credentials

```bash
# Create vault file from template and populate credentials
cp vault.yml.example vault.yml

# Edit vault.yml with your CatC credentials (and optional git_token for private repos)
# then encrypt with Ansible Vault:
ansible-vault encrypt vault.yml

# Store the vault password in a local file (gitignored):
echo 'your_vault_password' > .vault_pass
chmod 600 .vault_pass
```

### Fix Directory Permissions (one-time)

Ansible ignores `ansible.cfg` if the directory is world-writable:

```bash
chmod o-w .
```

### Run

```bash
# Standard run — inventory loaded automatically via ansible.cfg
ansible-playbook ansible-git-catc.yml --vault-password-file .vault_pass

# Interactive vault password prompt
ansible-playbook ansible-git-catc.yml --ask-vault-pass

# Debug mode — shows sorted template list, workflow configs, and sync results
DEBUG=true ansible-playbook ansible-git-catc.yml --vault-password-file .vault_pass
```

---

## Compatibility Matrix

Match `dnac_version` in `inventory.yml` to your Catalyst Center version.
This playbook suite ships `requirements.yml` pinned to `cisco.dnac 6.46.0`, which is verified compatible with CatC 2.3.7.6 and 2.3.7.9.

| Cisco Catalyst Center | `cisco.dnac` Collection | `dnacentersdk` | Notes |
|-----------------------|-------------------------|----------------|-------|
| 2.3.5.3 | 6.13.3 | 2.6.11 | Legacy |
| 2.3.7.6 | 6.25.0 – **6.46.0** | 2.8.3 – 2.8.6 | Lab baseline; **6.46.0 verified** |
| 2.3.7.9 | 6.33.2 – **6.46.0** | 2.8.6 | Recommended; **6.46.0 verified** |
| 3.1.3.0 | ≥ 6.36.0 | ≥ 2.10.1 | Latest |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `No inventory was parsed` | `ansible.cfg` ignored | Run `chmod o-w .` on the project directory |
| `NCTP10073: syntax errors` | CatC Jinja2 parser limitation (e.g., `not in`) | Rewrite to `not X in Y` in source template |
| Composite fails with missing templates | Individual templates not synced first | Ensure templates in composite `.yml` exist in CatC; the playbook handles ordering automatically |
| `403` from GitHub API | Rate limit or missing token for private repo | Add `git_token` to `vault.yml` |
| Wrong templates picked up | `git_repo_subfolder` not set or wrong | Verify subfolder path matches repository structure |

---

## References

- [Cisco DNA Center Ansible Collection](https://cisco-en-programmability.github.io/dnacenter-ansible/main/plugins/index.html)
- [Cisco Catalyst Center API Reference](https://developer.cisco.com/docs/dna-center/)
- [Sample Template Repository — CatalystCenter-BGP-EVPN-VXLAN](https://github.com/imanassypov/CatalystCenter-BGP-EVPN-VXLAN)
- [CI/CD Pipeline inspiration](https://gitlab.com/oboehmer/dnac-template-as-code) by Oliver Boehmer

---

## Author

**Igor Manassypov**  
Systems Engineer, Cisco Systems  
[imanassy@cisco.com](mailto:imanassy@cisco.com)  
Copyright © 2024–2026 Cisco Systems, Inc. All rights reserved.
