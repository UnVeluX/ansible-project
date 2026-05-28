# ansible-project

A collection of Ansible playbooks for Linux infrastructure automation — covering system provisioning, user management, package deployment, patching, and SSH key distribution. Designed to be used with **Ansible AWX / Automation Platform** or run directly via the `ansible-playbook` CLI.

---

## Table of Contents

- [Playbooks](#playbooks)
- [Requirements](#requirements)
- [User Conventions](#user-conventions)
- [Getting Started](#getting-started)
- [AWX Usage](#awx-usage)
  - [Setting up the deploy\_ssh\_key playbook (Custom Credential Type)](#setting-up-the-deploy_ssh_key-playbook-custom-credential-type)
- [Playbook Reference](#playbook-reference)
- [Maintainer](#maintainer)

---

## Playbooks

| File | Description |
|---|---|
| `ping-update.yml` | Ping all hosts and update the package cache (Debian & RedHat) |
| `update-grade_reboot.yml` | Full dist-upgrade with automatic reboot if required (Debian) |
| `install_tools.yml` | Install common CLI tools: vim, nano, curl, git, htop, net-tools (Debian) |
| `create_user.yml` | Create the `ansible` user, add to sudo, force password change on first login |
| `deploy_ssh_key.yml` | Deploy an SSH public key to the `ansible` user's `authorized_keys` |
| `backup_home_dir.yml` | Archive `/home` to `/tmp/home_backup.tar.gz` on each target host |
| `sys_info.yml` | Display a full system report: OS, CPU, RAM, disk, and IP address |

---

## Requirements

- Ansible 2.10+
- Python 3 on the control node
- Target hosts: Debian/Ubuntu (most playbooks) or RedHat/CentOS (ping-update)

---

## User Conventions

> **Important:** Several playbooks assume specific user accounts exist on target hosts.

- `create_user.yml` creates a user named **`ansible`** with sudo privileges.
- `deploy_ssh_key.yml` deploys keys to the **`ansible`** user by default (`ssh_user: ansible`).
- Some playbooks use `become: yes` and therefore expect the connection user (configured in your inventory or AWX credentials) to have sudo access.

Make sure the target machines have the expected user present (or run `create_user.yml` first as a provisioning step).

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/UnVeluX/ansible-project.git
cd ansible-project
```

### 2. Configure your inventory

Create an `inventory` file or use an existing one:

```ini
[servers]
192.168.1.10
192.168.1.11

[servers:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### 3. Run a playbook

```bash
# Ping hosts and update package cache
ansible-playbook -i inventory ping-update.yml

# Full upgrade with reboot if needed
ansible-playbook -i inventory update-grade_reboot.yml

# Create the ansible user on all hosts
ansible-playbook -i inventory create_user.yml

# Deploy your SSH public key
ansible-playbook -i inventory deploy_ssh_key.yml

# Get a full system report
ansible-playbook -i inventory sys_info.yml
```

---

## AWX Usage

Add this repository as a **Project** in AWX (source: Git, URL: `https://github.com/UnVeluX/ansible-project`), then create a **Job Template** for each playbook you want to run.

### Setting up the `deploy_ssh_key` playbook (Custom Credential Type)

`deploy_ssh_key.yml` requires the `ssh_public_key` variable to be passed in at runtime. The cleanest way to handle this in AWX is to create a **Custom Credential Type** so the key can be stored and injected securely.

#### Step 1 — Create the Custom Credential Type

Navigate to **Settings → Credential Types → Add**.

**Input Configuration:**

```yaml
fields:
  - id: ssh_public_key
    type: string
    label: SSH Public Key
    secret: false
    multiline: true
required:
  - ssh_public_key
```

**Injector Configuration:**

```yaml
extra_vars:
  ssh_public_key: '{{ ssh_public_key }}'
```

Save the credential type.

#### Step 2 — Create a Credential

Navigate to **Credentials → Add**, select the custom type you just created, and paste your SSH public key into the **SSH Public Key** field.

#### Step 3 — Attach to the Job Template

When creating or editing the Job Template for `deploy_ssh_key.yml`, add the credential created above in the **Credentials** section. AWX will automatically inject `ssh_public_key` as an extra variable at runtime.

---

## Playbook Reference

### `ping-update.yml`
Pings all hosts to verify connectivity, then updates the package cache. Supports both Debian (`apt`) and RedHat (`dnf`) families.

### `update-grade_reboot.yml`
Runs a full `dist-upgrade` on Debian/Ubuntu hosts. Checks for `/var/run/reboot-required` and reboots automatically if the flag is present.

### `install_tools.yml`
Installs a base set of tools on Debian/Ubuntu hosts: `vim`, `nano`, `curl`, `git`, `htop`, `net-tools`. Skipped on non-Debian systems.

### `create_user.yml`
Creates a user named **`ansible`** (customizable via `username` var), adds them to the `sudo` group, sets a pre-hashed password, and forces a password change on first login. Also injects a `.bashrc` message prompting the user to reconnect their SSH session after the password change.

> **Note:** The default password hash in the playbook should be replaced with your own before use.
> Original password : TempPassword123!
> 
> Or generate one with:
> ```bash
> python3 -c "from passlib.hash import sha512_crypt; print(sha512_crypt.using(rounds=5000).hash('yourpassword'))"
> ```

### `deploy_ssh_key.yml`
Deploys an SSH public key to the `authorized_keys` of the **`ansible`** user on all target hosts. Requires the `ssh_public_key` variable (see [AWX setup](#setting-up-the-deploy_ssh_key-playbook-custom-credential-type) above or pass via `-e`).

### `backup_home_dir.yml`
Archives the entire `/home` directory into `/tmp/home_backup.tar.gz` on each host. The backup stays local to the target machine.

### `sys_info.yml`
Outputs a formatted system report to the Ansible output/log, including hostname, OS distribution and kernel, IP address, CPU model and usage, RAM total/free, and per-mount disk usage.

---

## Maintainer

**UnVeluX** — [github.com/UnVeluX](https://github.com/UnVeluX)

Issues and pull requests are welcome.
