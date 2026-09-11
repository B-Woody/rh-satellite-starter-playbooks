# Red Hat Satellite Automation Playbooks

This is a set of Ansible playbooks for a simple automated installation and configuration of a very basic Red Hat Satellite. Nothing crazy, but it serves as a good starting point for setting up a new Satellite deployment with config-as-code approach. Deliberately kept simple so we can clone it and build on it.

Playbooks use the `redhat.satellite` and `redhat.satellite_operations` Ansible collections from https://github.com/RedHatSatellite/ for as many tasks as possible.

You can run these playbooks individually, or all at once with `run_all.yml`

If need be, comment out lines in `run_all.yml` to skip steps or add more playbooks to expand. 

## Prerequisites

1. Ansible is installed on the control node with `redhat.satellite` and `redhat.satellite_operations` collections.
2. The target Satellite server is reachable via SSH.
3. An RHN manifest file is available at `/tmp/manifest.zip` (or set `MANIFEST_PATH` env).
4. SSH key-based authentication is configured for the target host.
5. For an offline installation, matching RHEL 9 and Satellite 6.19 binary DVD ISO images are available on the Ansible control node.
6. For Capsule deployment, a RHEL 9 host with working forward and reverse DNS is reachable from both Ansible and Satellite.

## Collection Installation

Install the required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

If you're running this from Fedora or this doesn't work, you might need to get the collections from an actual RHEL host that has them installed via RPM. 

## Variable Configuration

### Group Variables

Group variables (shared across all hosts):

**`group_vars/satellite_servers.yml`:**

- `satellite_hostname`: The hostname or FQDN of the Satellite server.
- `satellite_organization`: The organization name in Satellite (default: `Default_Organization`).
- `satellite_admin_password`: The admin password for Satellite (default: `changeme`).
- `offline_install_rhel_iso_source`: Controller-side path to the RHEL binary DVD ISO.
- `offline_install_satellite_iso_source`: Controller-side path to the Satellite binary DVD ISO.
- `offline_install_iso_directory`: Directory in which the ISOs are stored on the Satellite server.
- `offline_install_repositories`: Local repository IDs and paths. Override these if a DVD has a different directory layout.

**`group_vars/all.yml`:**

- `capsule_repository_sets`: Repository sets needed to install Capsule 6.19.
- `capsule_content_repositories`: Generated Satellite repository names and parent products.
- `capsule_content_view`: Content View used by Capsule hosts before installation.
- `capsule_activation_key`: Activation key used to register the ready RHEL hosts.
- `capsule_lifecycle_environments`: Content synchronized to each installed Capsule.
- `capsule_download_policy`: Capsule content download policy.
- `capsule_content_force_publish`: Publish a fresh installation Content View version after synchronization.
- `capsule_certificates_regenerate`: Explicitly replace existing Capsule certificate bundles.
- `capsule_installer_extra_options`: Additional `satellite-installer` arguments.
- `capsule_expected_features`: Features required by post-install verification.
- `capsule_manage_firewall`: Manage the packaged `RH-Satellite-6-capsule` firewalld service.

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
- Add each Capsule to the `capsule_servers` group. Use its DNS FQDN for `ansible_host`, or set `capsule_fqdn` in host variables.

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

### Optional Step 00: Prepare Offline Installation Repositories

Set the two controller-side ISO paths in `group_vars/satellite_servers.yml`, host variables, or on the command line. Then copy the images and configure persistent ISO mounts and local DNF repositories:

```bash
ansible-playbook -i inventory.yml 00_copy_offline_media.yml \
  -e offline_install_rhel_iso_source=/path/to/rhel-9.iso \
  -e offline_install_satellite_iso_source=/path/to/satellite-6.19.iso

ansible-playbook -i inventory.yml 00_configure_offline_repositories.yml \
  -e offline_install_rhel_iso_source=/path/to/rhel-9.iso \
  -e offline_install_satellite_iso_source=/path/to/satellite-6.19.iso
```

The configuration playbook verifies repository metadata and confirms that `hostname` and `satellite-installer` are available using only the DVD repositories. It creates `/etc/yum.repos.d/satellite-offline.repo`. The default DVD paths are `BaseOS` and `AppStream` on the RHEL image and `Satellite` and `Maintenance` on the Satellite image.

These playbooks configure repositories needed to install Satellite itself; they do not import operating-system content into Satellite. Use Playbooks 08 and 09 to transfer Content Views to a disconnected Satellite after installation. They are intentionally excluded from `run_all.yml`, so connected installation behavior does not change.

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

### Step 07: Configure Remote Execution (REX) and Compliance Scanning (OpenSCAP)

```bash
ansible-playbook -i inventory.yml 07_rex_and_scap.yml
```

This configures REX and OpenSCAP features with some basic examples. Most uses direct API calls with `ansible.builtin.uri` and needs some tidy up to be more re-usable.

### Step 08: Export a Content View

Run this against the connected Satellite after repository synchronization:

```bash
ansible-playbook -i inventory.yml 08_export_content_view.yml
```

The playbook publishes a new version of `content_transfer_content_view` and exports it in `syncable` format. The first export for a `content_transfer_destination_server` is complete. Later runs use the latest syncable export history for that destination and create an incremental export. This assumes every generated incremental is transferred and imported in order because the connected Satellite cannot inspect import history on the disconnected Satellite.

To select a known imported baseline explicitly, pass its source export history ID:

```bash
ansible-playbook -i inventory.yml 08_export_content_view.yml \
  -e content_transfer_from_history_id=123
```

Copy the complete export directory, including `metadata.json` and all repository content, to the disconnected Satellite. Preserve the directory layout and place it beneath `/var/lib/pulp/imports`.

### Step 09: Import a Content View

Run this against the disconnected Satellite after transferring the export:

```bash
ansible-playbook -i inventory.yml 09_import_content_view.yml \
  -e content_transfer_import_path=/var/lib/pulp/imports/my-export
```

By default, the metadata file is `metadata.json` inside `content_transfer_import_path`; override `content_transfer_metadata_file` if your export layout differs. Satellite determines whether the import is complete or incremental from this metadata. For an incremental import, the playbook verifies that the exact predecessor version has an import history before starting the import. Import each incremental export in order and do not remove the baseline content.

These playbooks use the `redhat.satellite` collection and do not invoke Hammer. They are not included in `run_all.yml` because export and import normally target different Satellite servers.

## Capsule Deployment

The Capsule workflow is separate from `run_all.yml` and assumes the main Satellite installation, manifest import, organization, and location are operational. It deploys a Satellite 6.19 content Capsule with Remote Execution enabled by default.

Before running it, review `capsule_repository_sets` and `capsule_content_repositories` in `group_vars/all.yml`. Repository names generated from a Red Hat manifest can differ. The configured names must exactly match those shown by Satellite.

Configure at least one Capsule host in `inventory.yml`:

```yaml
capsule_servers:
  hosts:
    capsule01:
      ansible_host: capsule01.example.com
      ansible_user: cloud-user
```

Override site-specific assignments in `host_vars/capsule01.yml` when needed:

```yaml
---
capsule_fqdn: capsule01.example.com
capsule_location: Remote Site
capsule_lifecycle_environments:
  - Library
  - Production
capsule_download_policy: on_demand
```

Run the complete workflow:

```bash
ansible-playbook -i inventory.yml run_capsule.yml
```

The individual playbooks are:

1. `10_prepare_capsule_content.yml` enables and synchronizes the RHEL and Capsule repositories, creates and publishes a Content View, and creates an activation key.
2. `11_register_capsule_host.yml` uses `redhat.satellite.registration_command` to register the ready RHEL host and verifies that `satellite-capsule` is available.
3. `12_generate_capsule_certificates.yml` uses `redhat.satellite_operations.capsule_certs_generate` and securely stages one certificate bundle per Capsule under `.artifacts/`.
4. `13_install_capsule.yml` uses `redhat.satellite_operations.installer` with the `capsule` scenario and enables Remote Execution.
5. `14_configure_capsule.yml` uses `redhat.satellite.smart_proxy` and `smart_proxy_refresh` to register the Capsule and assign its organization, location, lifecycle environments, and download policy.
6. `15_sync_capsule_content.yml` starts Capsule content synchronization through the Katello API and waits with `redhat.satellite.wait_for_task`.
7. `16_verify_capsule.yml` verifies the local proxy service and the features advertised to Satellite.

Repository synchronization and Capsule content synchronization are operational actions and run whenever their playbooks run. Content View publication is idempotent by default: the first version is published automatically. Set `capsule_content_force_publish=true` when synchronized repository changes must be published for later Capsule installations.

Certificate artifacts are ignored by Git, stored with restrictive permissions, and removed from the controller and Capsule after a successful installation by default. The protected source bundle remains on Satellite so later complete runs can stage it again. Set `capsule_certificates_regenerate=true` only when certificates must be replaced.

The default configuration disables Satellite TLS certificate validation to match the existing repository defaults. For production, install the Satellite CA on the Ansible execution hosts and set both `satellite_validate_certs` and `capsule_validate_certs` to `true`.

## Running All Playbooks

If you want to run all playbooks in sequence, use the `run_all.yml` master playbook:

```bash
ansible-playbook -i inventory.yml run_all.yml --become-ask-pass
```

## Notes

- All playbooks use the `inventory.yml` file as the default inventory.
- All playbooks use SSH to connect to the target host.
- Playbook 01 requires root privileges (`become: true`).
- Capsule Playbooks 11-13 and 16 require root privileges on Capsule hosts.
- The optional Step 00 playbooks require root privileges on the Satellite server and enough free space for both ISO images.
- Playbooks 02–06 connect to Satellite's API via HTTP and do not need root privileges.
- The manifest import (Playbook 02) requires a valid Red Hat subscription manifest.
- Repository names in Playbooks 03–06 should match the actual repository names available in your Satellite organization after manifest import.
- **This is a starting point. Adapt the playbooks to your needs**

## Ansible Vault

If you use this for a production deployment, please strongly consider using Ansible vault so your creds stay secure.

### Quick Reference

The playbooks store secrets in plain text by default. Use Ansible Vault to encrypt sensitive files:

```bash
ansible-vault encrypt group_vars/satellite_servers.yml
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

### Version Control

Add encrypted vault files to `.gitignore` so they are not committed:

```gitignore
# Ansible Vault — provide these externally
*.yml.enc
group_vars/*vault*
host_vars/*vault*
```

## ToDo:

- [ ] Acitvation Key Automation
- [ ] Host Configuration ( firewall rules, storage verification etc. )
- [x] Offline repositories for disconnected Satellite installation

---

I hope this helps people get a head start!

-Woody
