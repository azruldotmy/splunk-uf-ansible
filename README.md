# Splunk Universal Forwarder 10.6.0.5 — Ansible Deployment

This project is intended to deploy Splunk Universal Forwarder to managed Linux servers from a central Ansible control node.

## Deployment settings

| Item | Setting |
|---|---|
| Ansible control node | `172.16.1.210` |
| Splunk deployment server | `172.16.1.200:8089` |
| Universal Forwarder version | `10.6.0.5` |
| Package | `splunkforwarder-10.6.0.5-86587d4e3b27-linux-amd64.tgz` |
| Installation directory | `/opt/splunkforwarder` |
| Additional Splunk application admin | `splunk` |
| Linux service user/group | Use an existing, approved account and group on the clients; the example below assumes both are named `splunk` |

The Linux service identity and the Splunk application account are separate identities, even if both have the name `splunk`.

> **Before production use:** Update the variables and task file as described below. The original task file creates a `splunkfwd` user/group and uses the previous `siemsupport` account and deployment-server address; those settings must be changed for the environment above.

## 1. Prepare the control node

On the Ubuntu control node (`172.16.1.210`), install Ansible and SSH utilities:

```bash
sudo apt update
sudo apt install -y ansible-core openssh-client python3 ca-certificates
ansible --version
```

Run Ansible from a dedicated, approved automation account on the control node where possible. Keep its SSH private key and Vault password protected. Do not copy the private key to managed servers.

## 2. Arrange access to managed clients

Ansible is agentless, but it still requires remote access and privileges. Before deploying, arrange an approved SSH account (for example, `svc_ansible`) on each client. That account must be able to use `sudo` for the tasks that install files, adjust ownership, configure systemd, and start the forwarder.

The SSH account used by Ansible is **not** the Splunk service account and is **not** the Splunk application admin account.

Check these network paths with the infrastructure/network team:

- Control node to clients: TCP `22` (or the configured SSH port).
- Clients to `download.splunk.com`: TCP `443`, if clients download the package directly.
- Clients to the deployment server `172.16.1.200`: TCP `8089`.
- Clients to the configured Splunk receiving indexers: the receiving port configured in your environment (commonly TCP `9997`). Setting the deployment server does not configure indexer outputs by itself.

Managed Linux hosts need Python available for the Ansible modules used by this playbook. If clients cannot reach the public download site, stage the installer in an approved internal repository or copy it to the clients through an approved process and adjust the download task accordingly.

## 3. Inventory

For a single supported Linux group, `inventory.ini` can look like this:

```ini
[linux]
client-01 ansible_host=192.0.2.10 ansible_user=svc_ansible ansible_ssh_private_key_file=/home/ansible/.ssh/id_ed25519
```

Replace the example host and IP address with actual managed clients. `192.0.2.10` is documentation-only and is not a real target.

For a mixed Linux fleet, group clients by operating system while retaining a parent group for shared tasks:

```ini
[linux:children]
ubuntu
rhel

[ubuntu]
ubuntu-client-01 ansible_host=REPLACE_WITH_UBUNTU_IP ansible_user=svc_ansible

[rhel]
rhel-client-01 ansible_host=REPLACE_WITH_RHEL_IP ansible_user=svc_ansible
```

These inventory entries are examples only. Use the actual hostnames/IP addresses, SSH account, and approved SSH authentication method. Ensure the play in `site.yml` targets `hosts: linux` if using the parent group.

Test connectivity before attempting installation:

```bash
ansible linux -i inventory.ini -m ping
ansible linux -i inventory.ini -b -m command -a 'id -u'
```

The second command should return `0` on each host if privilege escalation works. If sudo requires a password, add `--ask-become-pass`.

## 4. Configure the variables

Move shared variables to `group_vars/linux/splunk.yml` when the play targets the parent `linux` group. If you only deploy to Ubuntu and keep the inventory group named `ubuntu`, `group_vars/ubuntu/splunk.yml` can be used instead.

Set the relevant values in the variables file:

```yaml
splunk_version: "10.6.0.5"
splunk_package: "splunkforwarder-10.6.0.5-86587d4e3b27-linux-amd64.tgz"
splunk_download_url: "https://download.splunk.com/products/universalforwarder/releases/10.6.0.5/linux/splunkforwarder-10.6.0.5-86587d4e3b27-linux-amd64.tgz"
splunk_download_path: "/var/tmp/splunkforwarder-10.6.0.5-86587d4e3b27-linux-amd64.tgz"
splunk_home: "/opt/splunkforwarder"

# These must match an existing, approved Linux account and group on every target.
# This example assumes the Linux user and group are both named "splunk".
splunk_service_user: "splunk"
splunk_service_group: "splunk"

# This is a Splunk application account, separate from the Linux service account.
splunk_admin_user: "splunk"
splunk_deployment_server: "172.16.1.200:8089"
splunk_phone_home_interval: 60
```

Do not assume that the service user/group exists just because the Splunk deployment team uses the name `splunk`. Confirm the actual Linux account and group with the server team first.

## 5. Reuse an existing Linux service account and group

If the clients already have the approved service account and group, the playbook should **validate** them rather than create or modify them. In `chapter1/tasks/main.yml`, remove the tasks named:

- `Ensure the Splunk service group exists`
- `Ensure the least-privileged Splunk service account exists`

Replace them with:

```yaml
- name: Verify the existing Splunk service account
  ansible.builtin.getent:
    database: passwd
    key: "{{ splunk_service_user }}"

- name: Verify the existing Splunk service group
  ansible.builtin.getent:
    database: group
    key: "{{ splunk_service_group }}"
```

If either identity is absent, the play should stop at validation rather than creating a new identity. Make sure the existing account is suitable for running Splunk and can read the log sources you plan to collect. Avoid reusing a broadly privileged Ansible automation account as the forwarder runtime account.

## 6. Configure credentials with Ansible Vault

Keep the built-in Splunk `admin` seed password and the additional Splunk account password in an encrypted Vault file. For a mixed fleet using `hosts: linux`, use `group_vars/linux/vault.yml`. Create the file with these variables:

```yaml
vault_splunk_seed_password: "REPLACE_WITH_A_STRONG_UNIQUE_PASSWORD"
vault_splunk_admin_password: "REPLACE_WITH_A_DIFFERENT_STRONG_PASSWORD"
```

Use two different, strong passwords; do not use the example strings. Encrypt the file:

```bash
ansible-vault encrypt group_vars/linux/vault.yml
```

For an Ubuntu-only inventory, place the file under `group_vars/ubuntu/vault.yml` instead and keep the variable path consistent with the playbook. Confirm that the validation task's `fail_msg` also refers to the chosen Vault path.

The built-in Splunk `admin` account is used for initial CLI authentication. `splunk_admin_user: "splunk"` is the additional Splunk application account the playbook creates. These are not Linux user accounts.

## 7. Rename the task directory and wire up `site.yml`

If using `chapter1` for the Splunk tasks, rename the directory from `chapter8` to `chapter1`. Keep the template under `chapter1/templates/deploymentclient.conf.j2`.

In `site.yml`, use the appropriate inventory group and import path. For a mixed Linux fleet:

```yaml
---
- name: Install and configure Splunk Universal Forwarder
  hosts: linux
  become: true
  gather_facts: true

  tasks:
    - name: Install and configure Splunk Universal Forwarder
      ansible.builtin.import_tasks: chapter1/tasks/main.yml

  handlers:
    - name: Restart Splunk Universal Forwarder
      ansible.builtin.systemd:
        name: SplunkForwarder
        daemon_reload: true
        state: restarted
```

If this file is being integrated into another playbook, merge the task import and handler into the appropriate existing play rather than creating conflicting duplicate plays or replacing unrelated tasks unintentionally.

In the deployment-client `template` task in `chapter1/tasks/main.yml`, use an explicit template path to match the renamed directory:

```yaml
src: "{{ playbook_dir }}/chapter1/templates/deploymentclient.conf.j2"
```

Ensure the template sets the deployment target to `172.16.1.200:8089` through the `splunk_deployment_server` variable.

## 8. Linux distribution and architecture compatibility

The current installer URL is for the Linux AMD64/x86-64 tarball. The current assertion checks for Linux, `x86_64`, and systemd, but does not explicitly allowlist distribution names or versions.

Before targeting different distributions, verify that each OS version and architecture is supported by this specific Universal Forwarder release. Group hosts by distribution and architecture as needed, and use the appropriate installer for ARM64 or other supported architectures. Do not remove compatibility checks just to make an incompatible host pass. Test each supported OS family before broad rollout.

Because this is a tarball-based installation, the task file does not use `apt` or `dnf` to install the forwarder package. Nevertheless, runtime dependencies, service behaviour, permissions, and supported OS versions must still be checked on each Linux family.

## 9. Validate and deploy in stages

Run a syntax check first:

```bash
ansible-playbook -i inventory.ini site.yml --syntax-check --ask-vault-pass
```

Check mode can be useful for inspection, but it cannot fully simulate extraction, the first Splunk startup, or Splunk CLI account creation:

```bash
ansible-playbook -i inventory.ini site.yml --check --diff --ask-vault-pass
```

Deploy to one approved test client first:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass --limit client-01
```

After validating the service, deployment-server check-in, and expected log collection, expand the rollout in controlled batches. For example:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass --limit linux --forks 10
```

The `--forks 10` option allows up to ten hosts to be processed concurrently; adjust it to suit your network, server capacity, and change window.

## 10. Verify a deployed client

On a client, verify the service and installed version:

```bash
sudo systemctl status SplunkForwarder
sudo /opt/splunkforwarder/bin/splunk version
sudo cat /opt/splunkforwarder/etc/system/local/deploymentclient.conf
```

Also verify from the deployment server that the client checks in and receives the expected deployment apps. Confirm that the relevant `inputs.conf` and `outputs.conf` settings are deployed, either by deployment-server apps or by an explicit configuration-management workflow.

## Operational guardrails

- Use `get_url` rather than invoking `wget` directly from a shell task.
- Obtain and configure the trusted package checksum before production rollout; the current workflow does not pin a checksum.
- The playbook is intended to avoid reinstalling the same version and to stop if a different version is already installed. Treat upgrades as a separate, tested change.
- The deployment-client configuration tells the forwarder where to check for apps. It does not, by itself, define monitored log sources or indexer output destinations.
- Use `no_log: true` on tasks whose arguments or output could expose secrets.
- Confirm ownership and permissions before recursively changing the ownership of an existing Splunk installation.
