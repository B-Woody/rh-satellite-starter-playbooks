# Red Hat Satellite Automation Playbooks

This directory contains a set of Ansible playbooks for automated installation and configuration of Red Hat Satellite.

Playbooks use the `redhat.satellite` and `redhat.satellite_operations` Ansible collections from https://github.com/RedHatSatellite/ for as many tasks as possible.

These playbooks are a starting point for automated installation and deployment of Red Hat Satellite. They are designed to be run in sequence, starting with `01_install_satellite.yml`.

## Prerequisites

1. Ansible is installed on the control node.
2. The target Satellite server is reachable via SSH.
3. An RHN manifest file is available at `/tmp/manifest.zip` (or set `MANIFEST_PATH` env).
4. SSH key-based authentication is configured for the target host.

## Collection Installation

Install the required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Variable Configuration

### Group Variables

Group variables (shared across all hosts):

**`group_vars/satellite.yml`:**

- `satellite_hostname`: The hostname or FQDN of the Satellite server.
- `satellite_organization`: The organization name in Satellite (default: `Default_Organization`).
- `satellite_admin_password`: The admin password for Satellite (default: `changeme`).

### Host Variables

Variables specific to each host:

**`host_vars/<hostname>.yml`:**

- Variables specific to individual hosts can be defined here.
- If a host has no corresponding file, group variables are used.

### Inventory Configuration

**`inventory.yml`:**

- Update `ansible_host` with the IP or FQDN of the target Satellite server.
- Update `ansible_user` with the SSH username.
- Update `ansible_ssh_private_key_file` with the path to your private key.

### Playbook-Specific Variables

Passed via `-e` or defined in vars blocks:

**`02_import_manifest.yml`:**

- `MANIFEST_PATH`: Environment variable or set to the path of the manifest zip file.

## SSH Connection Setup

All playbooks use SSH to connect to the target host. Configure SSH access:

1. Ensure passwordless SSH key authentication works:

```bash
ssh <ansible_user>@<satellite_host>
```

2. Update `inventory.yml` with the correct host, user, and key path.

## Playbook Overview

### Step 01: Install Satellite

```bash
ansible-playbook -i inventory.yml 01_install_satellite.yml
```

Installs Red Hat Satellite 6.19 using the `redhat.satellite_operations.installer` role. This role installs the `satellite-installer` package and runs `satellite-installer` with the satellite scenario. Requires root privileges (`become: true`).

### Step 02: Import Subscription Manifest

```bash
ansible-playbook -i inventory.yml 02_import_manifest.yml
```

Uploads the Red Hat subscription manifest to Satellite, enabling access to Red Hat content repositories. Connects to the Satellite API via HTTP.

### Step 03: Enable Repositories

```bash
ansible-playbook -i inventory.yml 03_enable_repos.yml
```

Enables RHEL 9 BaseOS and AppStream repository sets using the `repository_set` module. This marks the repositories as available for sync. Connects to the Satellite API via HTTP.

### Step 04: Create Lifecycle Environment Path

```bash
ansible-playbook -i inventory.yml 04_create_lce.yml
```

Creates a lifecycle environment hierarchy: `Library -> Development -> QA -> Production` using the `lifecycle_environment` module. Connects to the Satellite API via HTTP.

### Step 05: Create Content View

```bash
ansible-playbook -i inventory.yml 05_create_cv.yml
```

Creates a content view containing the RHEL 9 BaseOS and AppStream repositories using the `content_view` module. Connects to the Satellite API via HTTP.

### Step 06: Sync Repositories

```bash
ansible-playbook -i inventory.yml 06_sync_repos.yml
```

Synchronizes the RHEL 9 repositories to Satellite using the `repository_sync` module. Connects to the Satellite API via HTTP.

## Running All Playbooks

If you want to run all playbooks in sequence, use the `run_all.yml` master playbook:

```bash
ansible-playbook -i inventory.yml run_all.yml
```

## Notes

- All playbooks use the `inventory.yml` file as the default inventory.
- All playbooks use SSH to connect to the target host.
- Playbook 01 requires root privileges (`become: true`).
- Playbooks 02–06 connect to Satellite's API via HTTP and do not need root privileges.
- The manifest import (Playbook 02) requires a valid Red Hat subscription manifest.
- Repository names in Playbooks 03–06 should match the actual repository names available in your Satellite organization after manifest import.
- This is a starting point. Adapt the playbooks to your organization's needs.

## Ansible Vault

### Quick Reference

The playbooks store secrets in plain text by default. Use Ansible Vault to encrypt sensitive files:

```bash
ansible-vault encrypt group_vars/satellite.yml
ansible-vault edit inventory.yml
```

Run playbooks with a vault password prompt:

```bash
ansible-playbook -i inventory.yml --ask-vault-pass
```

Or use a stored key for CI/CD:

```bash
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_key
ansible-playbook -i inventory.yml
```

### Full Guide

#### 1. Encrypt Your Files

```bash
ansible-vault encrypt group_vars/satellite.yml inventory.yml
```

This rewrites them as `group_vars/satellite.yml.enc` and `inventory.yml.enc`.

#### 2. Edit Encrypted Files

```bash
ansible-vault edit group_vars/satellite.yml
```

#### 3. Decrypt Files

```bash
ansible-vault decrypt group_vars/satellite.yml
```

#### 4. View Encrypted Files

```bash
ansible-vault view group_vars/satellite.yml
```

#### 5. Using a Password File

Create a file containing your vault password (chmod 600):

```bash
echo "your-vault-password" > ~/.vault_key
chmod 600 ~/.vault_key
```

Then use `--vault-password-file ~/.vault_key` on every command.

### Version Control

Add encrypted vault files to `.gitignore` so they are not committed:

```gitignore
# Ansible Vault — provide these externally
*.yml.enc
group_vars/*vault*
host_vars/*vault*
```

Then commit the plain-text files only after they are encrypted.
