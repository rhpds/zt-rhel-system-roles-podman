# ZeroTouch Labs - Claude Instructions

This repository contains the **Red Hat Podman System Roles** RHEL online lab (`zt-rhel-system-roles-podman`) for the ZeroTouch platform. The lab teaches novice sysadmins to define, configure, install, and deploy a containerized service using the `podman` RHEL system role, deployed as a Quadlet managed by systemd. When modifying this lab, follow these patterns and conventions.

## Lab Overview

- **Audience**: Novice sysadmins. Persona: Sysadmin Nate mentors; Manager Scott sets the demands.
- **Flow**: install `rhel-system-roles` → author a system role playbook (`~/soe.yml`) that creates a Quadlet pulling a secure nginx container → deploy and validate → add automatic updates.
- **Topology**: a single RHEL 10 VM (`rhel`), one wetty root terminal tab.
- **Modules**: 5 (`module-01` … `module-05`), all with `solveButton: true`.

## Lab Structure

```
zt-rhel-system-roles-podman/
├── README.adoc                    # Lab overview documentation
├── lab-metadata.yml               # Showroom platform catalog metadata
├── site.yml                       # Antora site configuration
├── ui-config.yml                  # UI tabs and module configuration
├── config/
│   ├── instances.yaml             # VM definitions
│   ├── networks.yaml              # Network configuration
│   └── firewall.yaml              # Firewall rules
├── content/
│   ├── antora.yml                 # Antora content config
│   └── modules/ROOT/
│       ├── assets/images/         # Images for documentation
│       └── pages/                 # AsciiDoc module content files
├── setup-automation/              # Initial provisioning scripts
│   ├── setup-rhel.sh
│   ├── ansible.cfg
│   └── main.yml
├── runtime-automation/            # Per-module automation
│   ├── module-01/ … module-05/
│   │   ├── solve-rhel.sh          # Shows/performs solution
│   │   └── validation-rhel.sh     # Validates completion
│   ├── ansible.cfg
│   ├── inventory
│   └── main.yml
└── utilities/                     # Utility scripts (copy from template)
    ├── lab-build
    ├── lab-clean
    ├── lab-serve
    └── lab-stop
```

## CRITICAL Configuration Patterns

### Module Naming Convention

**CRITICAL: Module files MUST follow the `module-##` naming pattern**

- Content files: `module-01.adoc`, `module-02.adoc`, `module-03.adoc`, etc.
- Runtime directories: `module-01/`, `module-02/`, `module-03/`, etc.
- ui-config.yml references: `name: module-01`, `name: module-02`, etc.
- site.yml start_page: `modules::module-01.adoc`

❌ **DO NOT** use descriptive names like `01-introduction.adoc` or `02-install.adoc`
✅ **DO** use `module-01.adoc`, `module-02.adoc`, etc.

**Reason**: Antora and the showroom build system expect this exact naming pattern. Custom names will cause the antora-builder init container to fail, preventing the lab from deploying.

### Runtime Automation Scripts

**CRITICAL: Script naming must be exact**

Each module directory contains:
- `solve-rhel.sh` - Contains all commands the participant should run
- `validation-rhel.sh` - Validates completion (NOT `validate-rhel.sh`)

**Script requirements:**
- Use `#!/bin/sh` shebang (NOT `#!/bin/bash`)
- Must be executable (`chmod +x`)
- Log to `/tmp/progress.log` for debugging (first line: `echo "Solved module called module-XX" >> /tmp/progress.log` / `echo "Validating module called module-XX" >> /tmp/progress.log`)
- Exit with appropriate codes (0 for success, 1 for failure in validation)
- Solve scripts should run ALL participant commands, not just log a message

**Validation scripts check real conditions** (package installed, file exists, grep of expected content, service active). Every `FAIL:` message includes the exact command the participant should run to fix it. Modules with nothing to validate may omit the validation script entirely (e.g., `module-01` ships only a solve script).

**Example validation pattern:**

```bash
#!/bin/sh
echo "Validating module called module-03" >> /tmp/progress.log

if ! [ -f /root/soe.yml ]; then
    echo "FAIL: /root/soe.yml not found. Create the playbook (see module 3)."
    exit 1
fi

echo "PASS: The playbook defines the nginx quadlet and uses the podman system role"
exit 0
```

### instances.yaml Format

**ALWAYS use this exact format:**

```yaml
---
virtualmachines:
  - name: "rhel"
    image: "rhel-10-2-eus-06-08-26"
    bootloader: efi
    memory: "4G"
    cores: 1
    image_size: "40G"
    tags:
      - key: "AnsibleGroup"
        value: "bastions"
    networks:
      - default
```

**Key points:**
- Top-level key is `virtualmachines` (NOT `instances`)
- Image must be `"rhel-10-2-eus-06-08-26"` (RHEL 10.2 EUS)
- Must include `bootloader: efi`
- Use `image_size` for the main disk size (do NOT define a separate root disk)
- Memory and sizes use quoted strings with units: `"4G"`, `"40G"`
- Always include the `AnsibleGroup: bastions` tag
- Networks is just `- default` (simple list format)
- This lab uses a single VM — do not add more without updating ui-config tabs and all module content

### networks.yaml Format

**ALWAYS use this minimal format:**

```yaml
---
# By default egress traffic is not allowed, define below the ports allowed.
- name: default
```

**Do NOT specify subnet, gateway, or dhcp settings** - these are handled automatically.

### firewall.yaml Format

**ALWAYS use this format:**

```yaml
---
# By default egress traffic is not allowed, define below the ports allowed.
egress:
  - ports:
      - protocol: TCP
        port: 443
#ingress:
# For Cockpit
#  - ports:
#      - protocol: TCP
#        port: 9090
```

**Key points:**
- Use `egress:` for outbound traffic rules
- Protocol should be uppercase (TCP, UDP)
- This lab only needs egress to port 443 (HTTPS) — the podman role opens the web port on the VM's own firewall

### ui-config.yml Format

**Terminal tab MUST use this format:**

```yaml
---

antora:
  name: modules
  dir: www
  modules:
    - name: module-01
      label: "Introduction"
      solveButton: true
    - name: module-02
      label: "Install the Software"
      solveButton: true
    - name: module-05
      label: "Automatic Updates"
      solveButton: true

tabs:
  - name: "rhel terminal"
    url: /wetty_rhel/ssh/root
```

**Key points:**
- Must start with `---` followed by blank line, and end with a blank line
- One terminal tab: `rhel terminal` at `/wetty_rhel/ssh/root` (single-VM lab)
- Module names match the content file names (e.g., `module-01` matches `module-01.adoc`)
- `solveButton: true` for ALL modules (this lab's convention — every module has a solve script)

### site.yml Format

**ALWAYS use this complete format:**

```yaml
---
site:
  title: "Managing Services with the Podman System Role"
  url: https://github.com/rhpds/showroom_template_zero
  start_page: modules::module-01.adoc

content:
  sources:
    - url: ./
      start_path: content

ui:
  bundle:
    url: https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip
    snapshot: true
  supplemental_files: ./supplemental-ui

output:
  dir: ./www/www

runtime:
  cache_dir: ./.cache/antora
```

**Key points:**
- Must include `url: https://github.com/rhpds/showroom_template_zero` in the `site:` section
- **CRITICAL**: Use `https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip` for the bundle (NOT course-ui)
- Must include `snapshot: true`, `supplemental_files: ./supplemental-ui`, `output:` with `dir: ./www/www`, and `runtime:` with `cache_dir: ./.cache/antora`
- start_page should reference `module-01.adoc`

### antora.yml Format

**Keep this minimal:**

```yaml
---

name: modules
version: master
```

**Key points:**
- Located in `content/antora.yml`
- Must have blank line after `---`
- Keep it simple - just name and version
- Do NOT add asciidoc attributes section

### runtime-automation/inventory Format

**Must contain actual inventory content:**

```ini
[local]
localhost

[local:vars]
ansible_connection = local
```

**Do NOT** just have comments - the file needs actual inventory definitions.

### Ansible Configuration Files

**setup-automation/ansible.cfg:**
```ini
[defaults]
host_key_checking = False
```

**runtime-automation/ansible.cfg:**
```ini
[defaults]
#inventory                 = inventory
retry_files_enabled       = false
```

**Key points:**
- Different configurations for setup vs runtime
- setup-automation needs `host_key_checking = False`
- runtime-automation needs commented inventory and retry_files setting

### Setup Automation Pattern

**setup-automation/setup-rhel.sh must follow this pattern:**

```bash
#!/bin/bash

echo "Lab setup starting" > /tmp/progress.log
chmod 666 /tmp/progress.log

usermod -aG wheel rhel

# Lab-specific setup tasks here

echo "Lab setup complete" >> /tmp/progress.log
```

**This lab's specific setup**: the image ships `dnf-automatic` enabled. The setup script stops/disables/masks `dnf-automatic-install.timer` and `dnf-automatic.timer` and sets `apply_updates = no` / `download_updates = no` in `/etc/dnf/automatic.conf`, so module-05's `podman-auto-update.timer` is the only auto-update mechanism participants see. Do not remove this — it keeps the lab deterministic.

**Critical elements:**
- Write to `/tmp/progress.log` and set permissions (`chmod 666`)
- Add `rhel` user to wheel group
- Do NOT use `set -e` - this can cause premature exit
- Avoid long-running operations like pulling container images (the playbook run in module-04 does that, on purpose, where participants can watch)

### main.yml Files

**Use the standard template `main.yml` playbooks** (both `setup-automation/` and `runtime-automation/`):

These playbooks handle:
- Creating dynamic inventory with bastion host connections (from `BASTION_*` env vars)
- Checking for and executing module scripts (setup/solve/validation)
- Proper logging and error handling
- Outputting results to the platform

**Do NOT create a simplified version** - always copy from `zt-rhel-tmm-template` to ensure proper functionality with the platform's ansible-runner API.

### utilities/ Directory

**Always copy the entire utilities directory from `zt-rhel-tmm-template/utilities/`:**

- `lab-build` - Builds the lab documentation
- `lab-serve` - Serves the lab locally for preview
- `lab-stop` - Stops the local server
- `lab-clean` - Cleans build artifacts

**Copy the entire directory** - do not create these scripts manually.

**Local builds without podman/docker** (e.g., on the lab author's Mac): `npx -y antora site.yml` builds the site directly. The `www/` directory is gitignored build output.

### lab-metadata.yml Format

**ALWAYS include this file (Showroom platform catalog metadata):**

```yaml
lab:
  name: "Red Hat Podman System Roles"
  shortname: "zt-rhel-system-roles-podman"
  maintainer: "Matthew Yee (myee@redhat.com)"

  description:
    summary: "Learn how to use the podman RHEL system role to deploy containers as systemd services."
    goal: "Use the podman RHEL system role to define, deploy, and manage a containerized web service as code."
    concepts:
     - "Podman"
     - "RHEL System Roles"
    use_case: "Manage containerized services at scale with the podman RHEL system role"
    time_to_complete: "30"

  git_ref:
    production: "main"
    development: "main"
```

**Key points:**
- `shortname` must match the repository name
- `git_ref` values must match the branches on GitHub (this repo: `main`)
- Local commits are pushed to `github.com/rhpds/zt-rhel-system-roles-podman`

### Required Files

**Must include:**
- `README.adoc` - Lab overview documentation
- `lab-metadata.yml` - Platform catalog metadata
- `content/modules/ROOT/assets/images/` - Images referenced by module content

**Do NOT include:**
- `README.md` - Use README.adoc instead

## Content Guidelines

### AsciiDoc Module Files

- Place in `content/modules/ROOT/pages/`
- Name with module prefix: `module-01.adoc`, `module-02.adoc`, etc.
- Use proper AsciiDoc formatting:

```asciidoc
= Module Title

== Section Heading

Narrative text content goes here.

[source,bash,run]
----
commands that users will run
----

[NOTE]
====
Important notes in callout boxes
====

[TIP]
====
Helpful tips
====

[quote, Person Name]
____
Quotes for engagement or humor
____
```

### Collapsible Sections in Module Content

**Basic collapsible block (no nesting):**

```asciidoc
.Title of the collapsible step
[%collapsible]
====
Step content, instructions, images, etc.
====
```

**Wrapping a collapsible with narrative text (the standard pattern in this lab):** when a collapsible step is preceded by its own intro sentence and you want to group them as one visual unit, wrap both in an outer delimited block one level larger than the collapsible's own delimiter:

```asciidoc
=====
Narrative text introducing this step.

.Title of the collapsible step
[%collapsible]
====
Step content here.
====
=====
```

**Nesting a `[NOTE]`, `[TIP]`, etc. inside a collapsible:** AsciiDoc requires each nesting level to use a different delimiter length. If a collapsible's content includes another delimited block (like `[NOTE]`), bump the collapsible's own delimiter up one level (to `=====`), and bump any outer wrapper up another level (to `======`):

```asciidoc
======
Narrative text introducing this step.

.Title of the collapsible step
[%collapsible]
=====
Step content here.

[NOTE]
====
Callout content.
====
=====
======
```

**Key points:**
- The innermost delimited block always uses `====` (4 equals signs).
- Each level of nesting outward adds one more equals sign (`=====`, `======`, etc.).
- Always close blocks in reverse order of opening (innermost first).
- Forgetting to bump the delimiter length when nesting is a common source of broken rendering — the inner block's closing delimiter gets matched to the wrong opening block.
- Rebuild with `npx -y antora site.yml` after content changes to catch rendering errors.

### Real-World Context and Personas

Use the standard fictitious elements:

**The Company: Super-Business**
- A fictitious company where all work is "super-businessey"
- Use when the lab needs a realistic business context

**The Persona: Manager Scott**
- An IT middle-manager at Super-Business who has lost most technical abilities
- Makes ridiculous demands with unclear or changing requirements
- In this lab, Scott sets the scenario: "the web server must be a container, managed by systemd, defined as code — and it needs automatic updates"

**The Persona: Sysadmin Nate**
- An experienced systems administrator at Super-Business, Scott's long-suffering colleague
- Leaves helpful notes that guide without giving away the complete solution
- Adds engagement, humor, and practical wisdom

**Usage Guidelines:**
- Use Scott to set demands and create the "why"
- Use Nate's notes (e.g., `[NOTE]` blocks signed *From: Nate (Senior Sysadmin)*) to reinforce what the participant just learned

### Script Requirements

**All runtime scripts must:**
- Use `#!/bin/sh` shebang
- Be executable (`chmod +x`)
- Log to `/tmp/progress.log` for tracking
- Exit with appropriate codes: `exit 0` for success, `exit 1` for failure (validation scripts)

## Podman Lab Playbook Conventions

**CRITICAL: The lab's teaching content uses the standard RHEL system roles playbook shape.** The playbook participants write is `~/soe.yml` (in the home directory, like the sibling `zt-rhel-system-roles` lab):

```yaml
---
- name: Deploy a secure nginx web server
  hosts: localhost
  remote_user: root
  vars:
    podman_quadlet_specs:
      - name: nginx
        type: container
        file_content: |
          [Container]
          Image=registry.access.redhat.com/ubi10/nginx-126:10.0
          PublishPort=8080:8080
          PodmanArgs=--security-opt no-new-privileges
          Exec=/usr/libexec/s2i/run
          Volume=/opt/web/index.html:/opt/app-root/src/index.html:Z
    podman_firewall:
      - port: 8080/tcp
        state: enabled
  pre_tasks:
    - name: Create the web content directory
      ansible.builtin.file:
        path: /opt/web
        state: directory
        mode: '0755'
    - name: Create the web content
      ansible.builtin.copy:
        content: |
          <h1>Super-Business internal web server</h1>
          <p>Served by a systemd-managed nginx container.</p>
        dest: /opt/web/index.html
        mode: '0644'
  roles:
    - role: rhel-system-roles.podman
```

**Key conventions:**
- Play-level `vars:` + the `roles:` keyword — the standard system roles playbook shape (NOT `tasks:` + `include_role`)
- `remote_user: root` (NOT `become: true`)
- Role name `rhel-system-roles.podman` — the role path name as installed under `/usr/share/ansible/roles/` (NOT the collection FQN)
- Web content tasks go in `pre_tasks:` so they run before the role starts the container
- Quadlet specs use `podman_quadlet_specs` with `name`/`type`/`file_content`
- The image is fully qualified AND pinned (`registry.access.redhat.com/ubi10/nginx-126:10.0`) — required for `podman auto-update` to work
- module-05 adds `AutoUpdate=registry` in the `[Container]` section and enables `podman-auto-update.timer`
- Keep `~/soe.yml` paths consistent across module content, solve scripts, and validation scripts

**Hard-won lessons (do not regress):**

1. **The nginx image's default command just prints usage and exits.** `registry.access.redhat.com/ubi10/nginx-126` is an s2i builder image with `CMD $STI_SCRIPTS_PATH/usage`. The Quadlet MUST set `Exec=/usr/libexec/s2i/run` or the container starts and immediately dies. (Symptom: `systemctl status nginx` shows inactive/failed right after the playbook succeeds.)
2. **The image's docroot `/opt/app-root/src` ships EMPTY** — the page must be mounted in via `Volume=`.
3. **`ansible.builtin.copy` does not create parent directories** — `/opt/web` must be created (pre_tasks) before writing `/opt/web/index.html`.
4. **The podman system role installs the podman package itself** ("Ensure required packages are installed" in the role's `tasks/main.yml`). Do not require `podman --version` before the playbook runs — module-02 checks `rpm -q podman` and expects *not installed*; module-04 checks `podman --version` after the role ran.
5. **Setup masks the image's own `dnf-automatic` timers** so module-05's `podman-auto-update.timer` is the only update mechanism in play.

## Modifying This Lab

When asked to change this lab:

1. **Keep module-## naming** for all content files and runtime directories
2. **Update all three places together** when a module's commands change: the module content heredocs, `runtime-automation/{module}/solve-rhel.sh`, and `runtime-automation/{module}/validation-rhel.sh`
3. **Use `~/soe.yml`** as the playbook path everywhere (content, solve scripts, validation hints)
4. **Rebuild and validate locally**: `npx -y antora site.yml`, extract and YAML-parse heredoc playbooks from solve scripts, `sh -n` all scripts
5. **Commit and push** to `github.com/rhpds/zt-rhel-system-roles-podman` (branch `main`)

## Common Pitfalls to Avoid

❌ **DON'T:**
- Use custom module names like `01-introduction.adoc` - must be `module-01.adoc`
- Name validation scripts `validate-rhel.sh` - must be `validation-rhel.sh`
- Use `#!/bin/bash` in runtime scripts - use `#!/bin/sh`
- Create solve scripts that only log - they must contain actual participant commands
- Use `instances:` as top-level key (use `virtualmachines:`)
- Define a separate "root" disk (use `image_size:` instead)
- Specify subnet/gateway/dhcp in networks.yaml
- Use `port:` and `path:` for terminal tabs (use `url: /wetty_rhel/ssh/root`)
- Use wrong UI bundle URL (must be `nookbag-bundle` not `course-ui`)
- Forget the `AnsibleGroup: bastions` tag or `bootloader: efi`
- Forget blank lines in ui-config.yml and antora.yml
- Use `set -e` in setup scripts (can cause premature exit)
- Create simplified versions of main.yml files
- Write the system role playbook with `include_role`/`become: true` - use the `roles:` + play-level `vars:` shape
- Check `podman --version` before the playbook has run

✅ **DO:**
- Follow the `module-##` naming pattern exactly
- Use `validation-rhel.sh` for validation scripts
- Use `#!/bin/sh` for runtime scripts
- Put all participant commands in solve scripts for testing purposes
- Reference existing working labs for patterns
- Use quoted strings for memory/size values
- Keep networks.yaml minimal
- Make scripts executable
- Use clear pass/fail messages in validation scripts, with the fix command in every FAIL
- Include `lab-metadata.yml` with `shortname` matching the repo name
- Log to /tmp/progress.log in all scripts

## Troubleshooting Deployment Failures

If a lab fails to deploy with the showroom pod stuck in "PodInitializing":

1. **Check module naming** - Must be `module-01.adoc` not custom names
2. **Check script filenames** - Must be `validation-rhel.sh` not `validate-rhel.sh`
3. **Check shebangs** - Runtime scripts need `#!/bin/sh`
4. **Check ansible.cfg** - Different for setup vs runtime
5. **Check inventory file** - Must have actual content, not just comments
6. **Check UI bundle URL** - Must be `nookbag-bundle` not `course-ui`
7. **Check setup script** - Should match template pattern exactly
8. **Check permissions** - All scripts must be executable
9. **Compare to working lab** - Use `diff -r` to compare with a working lab

## Reference Labs

- **zt-rhel-tmm-template** - The authoritative template (utilities and main.yml always come from here)
- **zt-satellite-compliance** - Reference for accordion/collapsible content format and lab conventions
- **zt-rhel-system-roles** - Reference for the `soe.yml` playbook shape (`hosts: localhost`, `remote_user: root`, play-level `vars:`, `roles:` list)

## Critical Lessons Learned (2026-10)

1. **Module naming breaks Antora** - Custom names like `01-introduction.adoc` cause antora-builder to fail silently. Always use `module-##.adoc`.

2. **Script filename typo** - Using `validate-rhel.sh` instead of `validation-rhel.sh` causes runtime failures. The system looks for the exact filename.

3. **Wrong shebang causes issues** - Using `#!/bin/bash` instead of `#!/bin/sh` in runtime scripts can cause unexpected behavior in the container environment.

4. **Empty inventory fails** - The `runtime-automation/inventory` file must contain actual inventory content, not just comments.

5. **Wrong UI bundle** - Using `course-ui` bundle instead of `nookbag-bundle` causes the showroom pod to fail initialization.

6. **The podman role installs podman** - participants must not be asked to check for podman before the playbook has run (see Podman Lab Playbook Conventions).

7. **nginx s2i image needs `Exec=/usr/libexec/s2i/run`** - the default CMD prints usage and exits; without `Exec=` the container dies immediately after the playbook succeeds.

8. **Playbook conventions matter** - the `roles:` + play-level `vars:` shape with `rhel-system-roles.podman` is the documented standard; `include_role` with per-task vars is not how system role playbooks are written in this lab family.

These issues all result in the same symptom class: the showroom pod gets stuck at "PodInitializing", or the lab's playbook succeeds while the service silently fails. Always compare against a working lab when debugging.
