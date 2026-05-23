# ⚙️ AWS-infrastructure-automation-Ansible

> A production-grade Ansible project demonstrating infrastructure provisioning, secure credential management, passwordless SSH configuration, and targeted multi-OS configuration management using loops and conditionals.

![Ansible](https://img.shields.io/badge/Ansible-2.x-EE0000?style=flat-square&logo=ansible&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS-EC2-FF9900?style=flat-square&logo=amazon-aws&logoColor=white)
![Ansible Vault](https://img.shields.io/badge/Ansible_Vault-Encrypted-EE0000?style=flat-square&logo=ansible&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-AMI-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![RHEL](https://img.shields.io/badge/RHEL-AMI-EE0000?style=flat-square&logo=redhat&logoColor=white)
![WSL2](https://img.shields.io/badge/WSL2-Control_Node-0078D6?style=flat-square&logo=windows&logoColor=white)

---

## 📋 Table of Contents

- [📁 Project Structure](#-project-structure)
- [🔧 Prerequisites](#-prerequisites)
- [🔐 Task 1 — Secure Credential Management (Ansible Vault)](#-task-1--secure-credential-management-ansible-vault)
- [🔑 Task 2 — Passwordless Authentication (WSL2 Fix)](#-task-2--passwordless-authentication-wsl2-fix)
- [☁️ Task 3 — Multi-OS Dynamic Provisioning (Loops & Idempotency)](#️-task-3--multi-os-dynamic-provisioning-loops--idempotency)
- [🎯 Task 4 — Targeted Configuration Management (Conditionals)](#-task-4--targeted-configuration-management-conditionals)
- [▶️ Full Execution Guide](#️-full-execution-guide)
- [💡 Concepts Covered](#-concepts-covered)

---

## 📁 Project Structure

```
ansible-ec2-automation/
│
├── ec2_create.yml              # Playbook — provisions 3 EC2 instances via AWS API
├── ec2_stop.yml                # Playbook — targeted shutdown using conditionals
├── inventory.ini               # Global inventory tracking all managed nodes
├── vault.pass                  # ⚠️ Local vault password file (added to .gitignore)
│
└── group_vars/
    └── all/
        └── pass.yml            # 🔒 Vault-encrypted AWS credentials
```

> ⚠️ `vault.pass` is **never committed to version control**. It is listed in `.gitignore` and exists only on the local control node.

---

## 🔧 Prerequisites

### Required Packages

Install the following on your WSL2/Ubuntu control node before running any playbook:

```bash
# Install Ansible
sudo apt update && sudo apt install -y ansible

# Install Python AWS SDK (required by the amazon.aws collection)
pip install boto3 botocore --break-system-packages

# Install the official AWS Ansible collection
ansible-galaxy collection install amazon.aws
```

### AWS Security Group — Inbound Rules

Ensure the Security Group attached to your EC2 instances has the following inbound rule configured:

| Type       | Protocol | Port Range | Source        | Purpose                        |
|------------|----------|------------|---------------|--------------------------------|
| SSH        | TCP      | 22         | 0.0.0.0/0     | Ansible SSH access             |
| Custom TCP | TCP      | Your App   | 0.0.0.0/0     | Application traffic (optional) |

> **Note:** For production environments, restrict SSH source to your specific IP address instead of `0.0.0.0/0`.

### Local SSH Key Generation

Generate an RSA key pair on your WSL2 control node if one does not already exist:

```bash
ssh-keygen -t rsa -b 4096
# Accept all defaults — key saved to ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub
```

---

## 🔐 Task 1 — Secure Credential Management (Ansible Vault)

### Overview

AWS credentials (`aws_access_key` and `aws_secret_key`) are sensitive values that must never be stored in plaintext inside a repository. This project uses **Ansible Vault** to encrypt these credentials inside `group_vars/all/pass.yml`, making the file safe to track in version control.

Think of it as a **safe and a key**:

| Element | Analogy | In Practice |
|---|---|---|
| `group_vars/all/pass.yml` | 🔒 The **Safe** | Encrypted file containing AWS credentials |
| `vault.pass` | 🔑 The **Key** | Plaintext password file used to unlock the safe |
| `--vault-password-file vault.pass` | Using the key | Passed at runtime — no manual prompt required |

### `group_vars/all/pass.yml` (before encryption)

```yaml
aws_access_key: "AKIAIOSFODNN7EXAMPLE"
aws_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

### Setup Steps

**Step 1 — Create the vault password file**
```bash
echo "your-secure-vault-password" > vault.pass
```

**Step 2 — Add it to `.gitignore` immediately**
```bash
echo "vault.pass" >> .gitignore
```

**Step 3 — Encrypt the credentials file**
```bash
ansible-vault encrypt group_vars/all/pass.yml --vault-password-file vault.pass
```

**Step 4 — Verify, view, or edit the encrypted file**
```bash
# Verify it is encrypted
cat group_vars/all/pass.yml

# View decrypted contents safely (nothing written to disk)
ansible-vault view group_vars/all/pass.yml --vault-password-file vault.pass

# Edit the encrypted file in place
ansible-vault edit group_vars/all/pass.yml --vault-password-file vault.pass
```

After encryption, `group_vars/all/pass.yml` becomes unreadable ciphertext beginning with `$ANSIBLE_VAULT;1.1;AES256` — completely safe to commit.

---

## 🔑 Task 2 — Passwordless Authentication (WSL2 Fix)

### Overview

For Ansible to connect to managed nodes without password prompts, the control node's public SSH key must be present in each EC2 instance's `~/.ssh/authorized_keys` file. This is the foundation of passwordless authentication.

### The WSL2 Bug — Why Standard Commands Hang

The standard `ssh-copy-id` command uses a space-separated `-o` flag:

```bash
# ❌ This causes terminal hangs in WSL2/Ubuntu due to command-parsing timeout bug
ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o "IdentityFile ~/my-aws-key.pem" ec2-user@<EC2_PUBLIC_IP>
```

WSL2's SSH client has a known timeout issue when parsing space-separated configuration options inside `-o` flags, causing the command to hang indefinitely.

### ✅ The Fix — Use `=` Sign, No Spaces

```bash
# ✅ Bulletproof version — works reliably in WSL2
ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o IdentityFile=~/my-aws-key.pem ec2-user@<EC2_PUBLIC_IP>
```

> **Key difference:** `IdentityFile=~/my-aws-key.pem` uses `=` with **no spaces** — this prevents the WSL2 terminal from hanging during key copy.

### Targeting Different AMI Types

The default EC2 user differs between AMI distributions:

| AMI Type | Default SSH User | Example Command |
|---|---|---|
| Amazon Linux / RHEL | `ec2-user` | `ssh-copy-id ... ec2-user@<IP>` |
| Ubuntu | `ubuntu` | `ssh-copy-id ... ubuntu@<IP>` |

```bash
# Copy key to Amazon Linux / RHEL instance
ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o IdentityFile=~/my-aws-key.pem ec2-user@<RHEL_EC2_IP>

# Copy key to Ubuntu instance
ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o IdentityFile=~/my-aws-key.pem ubuntu@<UBUNTU_EC2_IP>
```

**Verify passwordless access:**
```bash
ssh ubuntu@<UBUNTU_EC2_IP>     # Should connect with no password prompt
ssh ec2-user@<RHEL_EC2_IP>     # Should connect with no password prompt
```

---

## ☁️ Task 3 — Multi-OS Dynamic Provisioning (Loops & Idempotency)

### Overview

`ec2_create.yml` provisions **3 EC2 instances** in the `ap-south-1` region in a single playbook run — 2 Ubuntu nodes and 1 RHEL node — using an Ansible `loop` block for clean, DRY iteration.

### `ec2_create.yml`

```yaml
---

- name: Provision Multi-OS EC2 Infrastructure
  hosts: localhost
  connection: local

  tasks:

    - name: Launch EC2 Instances
      amazon.aws.ec2_instance:
        name: "{{ item.name }}"
        instance_type: t2.micro
        image_id: "{{ item.ami }}"
        region: ap-south-1
        key_name: my-aws-key
        access_key: "{{ aws_access_key }}"
        secret_key: "{{ aws_secret_key }}"
        state: running
        tags:
          Name: "{{ item.name }}"
          OS: "{{ item.os }}"
      loop:
        - { name: "ubuntu-node-1", ami: "ami-07a00cf447dbbc844", os: "Ubuntu" }
        - { name: "ubuntu-node-2", ami: "ami-07a00cf447dbbc844", os: "Ubuntu" }
        - { name: "rhel-node-1",   ami: "ami-00a3ff4223e36738",  os: "RHEL"   }
```

### AMI Reference

| Instance | AMI ID | OS | Region |
|---|---|---|---|
| ubuntu-node-1 | `ami-07a00cf447dbbc844` | Ubuntu 22.04 LTS | ap-south-1 |
| ubuntu-node-2 | `ami-07a00cf447dbbc844` | Ubuntu 22.04 LTS | ap-south-1 |
| rhel-node-1 | `ami-00a3ff4223e36738` | Red Hat Enterprise Linux | ap-south-1 |

### 📌 Interview Focus — Idempotency

> **Idempotency** means running the same playbook multiple times always produces the same result — no duplicates, no unintended changes.

When `ec2_create.yml` is run a second time, Ansible checks whether an EC2 instance matching the defined `name` tag and specifications already exists in AWS. If it does:

```
TASK [Launch EC2 Instances] ****************************
ok: [localhost] => (item=ubuntu-node-1)   ← already exists, skipped
ok: [localhost] => (item=ubuntu-node-2)   ← already exists, skipped
ok: [localhost] => (item=rhel-node-1)     ← already exists, skipped
```

Ansible reports `ok` and takes **no action** instead of launching duplicate instances. This is a core principle of infrastructure-as-code and what separates Ansible from simple shell scripts.

---

## 🎯 Task 4 — Targeted Configuration Management (Conditionals)

### Overview

After provisioning, all three EC2 public IPs are added to `inventory.ini` under a single `[all]` group. `ec2_stop.yml` then automates a remote shutdown — but **only on Ubuntu nodes** — using Ansible's `when` conditional.

### `inventory.ini`

```ini
[all]
<UBUNTU_NODE_1_IP>   ansible_user=ubuntu
<UBUNTU_NODE_2_IP>   ansible_user=ubuntu
<RHEL_NODE_1_IP>     ansible_user=ec2-user
```

### `ec2_stop.yml`

```yaml
---

- name: Targeted Shutdown — Ubuntu Nodes Only
  hosts: all
  become: true

  tasks:

    - name: Gather system facts
      ansible.builtin.gather_facts:

    - name: Shutdown Ubuntu instances only
      ansible.builtin.command: /sbin/shutdown -h now
      when: ansible_facts['os_family'] == "Debian"
```

### How the Conditional Works

Before executing any task, Ansible automatically runs a **Gathering Facts** step. This interrogates each managed node and collects detailed system information — including `os_family`.

| OS | `ansible_facts['os_family']` | Shutdown Executed? |
|---|---|---|
| Ubuntu 22.04 LTS | `"Debian"` | ✅ Yes |
| Red Hat Enterprise Linux | `"Redhat"` | ❌ Skipped |

```
TASK [Shutdown Ubuntu instances only] *************************
changed: [<UBUNTU_NODE_1_IP>]   ← os_family == "Debian" → executed
changed: [<UBUNTU_NODE_2_IP>]   ← os_family == "Debian" → executed
skipping: [<RHEL_NODE_1_IP>]    ← os_family == "Redhat" → skipped
```

> The `when` clause evaluates the automatically gathered `os_family` fact. Ubuntu belongs to the `"Debian"` family, RHEL belongs to the `"Redhat"` family. This allows surgical, per-OS task execution across a mixed-OS infrastructure — all from a single playbook targeting `hosts: all`.

---

## ▶️ Full Execution Guide

Follow these steps in order for a complete end-to-end run.

### Step 1 — Encrypt credentials
```bash
ansible-vault encrypt group_vars/all/pass.yml --vault-password-file vault.pass
```

### Step 2 — Provision the EC2 infrastructure
```bash
ansible-playbook ec2_create.yml --vault-password-file vault.pass
```

### Step 3 — Update `inventory.ini` with new public IPs
```bash
# Retrieve public IPs from AWS Console or CLI, then add to inventory.ini
nano inventory.ini
```

### Step 4 — Copy your public key to all instances (run once per node)
```bash
# Ubuntu nodes
ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o IdentityFile=~/my-aws-key.pem ubuntu@<UBUNTU_NODE_1_IP>
ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o IdentityFile=~/my-aws-key.pem ubuntu@<UBUNTU_NODE_2_IP>

# RHEL node
ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o IdentityFile=~/my-aws-key.pem ec2-user@<RHEL_NODE_1_IP>
```

### Step 5 — Verify connectivity across all nodes
```bash
ansible all -i inventory.ini -m ping
```

Expected output:
```
<UBUNTU_NODE_1_IP> | SUCCESS => { "ping": "pong" }
<UBUNTU_NODE_2_IP> | SUCCESS => { "ping": "pong" }
<RHEL_NODE_1_IP>   | SUCCESS => { "ping": "pong" }
```

### Step 6 — Dry run the configuration playbook
```bash
ansible-playbook ec2_stop.yml -i inventory.ini --check
```

### Step 7 — Execute targeted shutdown on Ubuntu nodes only
```bash
ansible-playbook ec2_stop.yml -i inventory.ini
```

---

## 💡 Concepts Covered

| Concept | Description |
|---|---|
| **Ansible Vault** | Encrypts `group_vars/all/pass.yml` so AWS credentials are safe to commit |
| **group_vars** | Variables automatically loaded for all hosts — no manual `vars_files` needed |
| **vault.pass + `--vault-password-file`** | Eliminates manual password prompts on every playbook run |
| **Passwordless SSH** | RSA public key pushed to managed nodes via `ssh-copy-id` |
| **WSL2 Fix** | Using `IdentityFile=` (no spaces) to prevent terminal hangs in WSL2 |
| **Ansible Loop** | Iterates over a list to provision multiple EC2 instances cleanly and concisely |
| **Idempotency** | Re-running `ec2_create.yml` reports `ok` — never creates duplicate instances |
| **Gathering Facts** | Ansible auto-collects system data (`os_family`, etc.) before executing tasks |
| **`when` Conditional** | Filters task execution by `os_family` — targets Ubuntu, skips RHEL automatically |
| **`become: true`** | Escalates to sudo on remote nodes — required for system shutdown command |
| **Mixed-OS Inventory** | Single `[all]` group managing both Ubuntu and RHEL nodes simultaneously |
| **hosts: localhost** | EC2 provisioning runs from the control node via AWS API — no SSH to target |
| **amazon.aws collection** | Official Ansible collection providing the `ec2_instance` module |
| **boto3 / botocore** | Python AWS SDK required by the `amazon.aws` Ansible collection |

---

<div align="center">
  <sub>Built for DevOps learning · Ansible · Ansible Vault · AWS EC2 · Loops · Conditionals · WSL2</sub>
</div>
