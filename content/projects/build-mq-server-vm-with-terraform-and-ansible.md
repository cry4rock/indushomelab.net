---
title: "Build an IBM MQ Server VM with Terraform and Ansible on Proxmox"
date: 2026-05-24
draft: false
description: "A complete guide to automating IBM MQ server provisioning on Proxmox using Terraform for VM creation and Ansible for MQ installation and configuration."
tags:
  - homelab
  - terraform
  - ansible
  - ibm-mq
  - devops
  - infrastructure
  - automation
categories:
  - Homelab
  - DevOps
  - Infrastructure
  - Automation
cover:
  alt: "IBM MQ Server provisioned with Terraform and Ansible on Proxmox"
---

In this post I'll walk through how I fully automated the provisioning of an IBM MQ server on my Proxmox homelab using Terraform and Ansible. With a single `terraform apply`, the entire stack builds itself — VM creation, OS configuration, IBM MQ installation, queue manager setup, and systemd service registration.

---

## Architecture Overview

```text
homelab02 (Terraform + Ansible host)
    │
    │  terraform apply
    ▼
Proxmox API (192.168.1.14:8006)
    │
    ├── ① Clone Ubuntu 24.04 template
    ├── ② Apply cloud-init (kernel params, qemu-agent)
    ├── ③ Wait for VM ready
    └── ④ Run Ansible → IBM MQ install + QM1 config
```

**Tools used:**
- **Proxmox VE 8.4** — hypervisor
- **Terraform** with `bpg/proxmox` provider — VM provisioning
- **cloud-init** — first-boot configuration
- **Ansible** — IBM MQ installation and configuration
- **IBM MQ 9.3.0** — message broker

---

## Prerequisites

### On the Proxmox server

- Proxmox VE 8.x installed
- An Ubuntu 24.04 cloud-init template (VM ID 9000)
- Snippets enabled on local storage
- An API token with Administrator permissions

### On the Ansible/Terraform host (homelab02)

- Terraform installed
- Ansible installed
- `ansible.posix` collection installed
- IBM MQ binary downloaded from IBM
- SSH key pair generated

---

## Step 1 — Create Proxmox API Token

In the Proxmox web UI:

```
Datacenter → Permissions → API Tokens → Add
  User:               root@pam
  Token ID:           proxmox_api_token_id
  Privilege Separation: unchecked
```

Then assign Administrator permissions:

```
Datacenter → Permissions → Add → API Token Permission
  Path:  /
  Token: root@pam!proxmox_api_token_id
  Role:  Administrator
```

Test the token from your Ansible host:

```bash
curl -sk https://192.168.1.14:8006/api2/json/version \
  -H 'Authorization: PVEAPIToken=root@pam!proxmox_api_token_id=<your-secret>'
```

You should get a JSON response like:

```json
{"data":{"version":"8.4.19","release":"8.4"}}
```

---

## Step 2 — Create Ubuntu 24.04 Cloud-Init Template

Run these commands on the **Proxmox server**:

```bash
# Download Ubuntu 24.04 cloud image
cd /tmp
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Create base VM
qm create 9000 --name ubuntu-2404-template --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0

# Import the cloud image
qm importdisk 9000 /tmp/noble-server-cloudimg-amd64.img local-lvm

# Configure the VM
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1

# Convert to template
qm template 9000
```

---

## Step 3 — Enable Snippets on Local Storage

The Terraform provider uploads cloud-init files via SSH as snippets. Enable this in Proxmox:

```bash
pvesm set local --content backup,iso,vztmpl,snippets
```

---

## Step 4 — Project Structure

```
proxmox-terraform/
├── main.tf
├── cloud-init.yaml
└── ibmmq-ansible/
    ├── playbook.yml
    ├── inventory.ini
    ├── group_vars/
    │   └── all.yml
    └── roles/
        └── ibmmq/
            ├── tasks/
            │   ├── main.yml
            │   ├── 01_users_groups.yml
            │   ├── 02_system_config.yml
            │   ├── 03_install_mq.yml
            │   ├── 04_environment.yml
            │   ├── 05_configure_mq.yml
            │   └── 06_service.yml
            ├── templates/
            │   ├── mqsc_config.j2
            │   └── ibmmq.service.j2
            └── handlers/
                └── main.yml
```

---

## Step 5 — cloud-init.yaml

This runs on first boot and configures the VM before Ansible takes over. The key things it does are creating the user, setting kernel parameters required by IBM MQ, and installing the qemu guest agent.

```yaml
#cloud-config
ssh_pwauth: true

users:
  - name: indunil
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
    groups: sudo
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-public-key

chpasswd:
  list: |
    indunil:yourpassword
  expire: false

package_update: true
package_upgrade: true

packages:
  - qemu-guest-agent
  - curl
  - git
  - python3

write_files:
  - path: /etc/sysctl.d/99-ibmmq.conf
    owner: root:root
    permissions: '0644'
    content: |
      # IBM MQ Kernel Parameters
      kernel.shmmax=268435456
      kernel.shmall=65536
      kernel.shmmni=4096
      kernel.sem=500 256000 250 1024

runcmd:
  - systemctl enable qemu-guest-agent
  - systemctl start qemu-guest-agent
  - sysctl -p /etc/sysctl.d/99-ibmmq.conf
```

> **Important:** The kernel shared memory parameters are critical for IBM MQ. Without `kernel.shmmax=268435456`, the queue manager creation fails with `AMQ8101S error 893`.

---

## Step 6 — main.tf

The Terraform configuration handles VM creation, waits for cloud-init to complete, then hands off to Ansible.

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.78"
    }
  }
}

provider "proxmox" {
  endpoint  = "https://192.168.1.14:8006"
  api_token = "root@pam!proxmox_api_token_id=<your-secret>"
  insecure  = true

  ssh {
    agent       = false
    username    = "root"
    private_key = file("~/.ssh/id_ed25519")
  }
}

# Upload cloud-init snippet to Proxmox
resource "proxmox_virtual_environment_file" "cloud_init" {
  content_type = "snippets"
  datastore_id = "local"
  node_name    = "homelabsvr"

  source_raw {
    data      = file("cloud-init.yaml")
    file_name = "cloud-init.yaml"
  }
}

# Create the VM
resource "proxmox_virtual_environment_vm" "ubuntu_vm" {
  name      = "terraform-ubuntu-01"
  node_name = "homelabsvr"

  agent {
    enabled = true
    trim    = true
  }

  clone {
    vm_id = 9000
  }

  cpu {
    cores = 4
  }

  memory {
    dedicated = 4096
  }

  disk {
    datastore_id = "local-lvm"
    interface    = "scsi0"
    size         = 50
  }

  network_device {
    bridge = "vmbr0"
  }

  initialization {
    user_data_file_id = proxmox_virtual_environment_file.cloud_init.id

    ip_config {
      ipv4 {
        address = "192.168.1.21/24"
        gateway = "192.168.1.1"
      }
    }

    user_account {
      username = "indunil"
      password = "yourpassword"
      keys     = ["ssh-ed25519 AAAA... your-public-key"]
    }
  }

  operating_system {
    type = "l26"
  }

  timeout_create = 1800
}

# Wait for cloud-init to complete
resource "null_resource" "wait_for_vm" {
  depends_on = [proxmox_virtual_environment_vm.ubuntu_vm]

  provisioner "local-exec" {
    command = <<EOT
      echo "Removing old SSH host key..."
      ssh-keygen -f '/home/indunil/.ssh/known_hosts' -R '192.168.1.21' || true

      echo "Waiting for SSH..."
      until ssh -o StrictHostKeyChecking=no \
                -o ConnectTimeout=5 \
                -i ~/.ssh/id_ed25519 \
                indunil@192.168.1.21 'echo SSH_OK' 2>/dev/null | grep -q 'SSH_OK'; do
        echo "SSH not ready yet, retrying in 10s..."
        sleep 10
      done
      echo "SSH is up!"

      echo "Waiting for cloud-init to complete..."
      until ssh -o StrictHostKeyChecking=no \
                -o ConnectTimeout=5 \
                -i ~/.ssh/id_ed25519 \
                indunil@192.168.1.21 \
                'cloud-init status 2>/dev/null' | grep -q 'done'; do
        echo "cloud-init still running, retrying in 15s..."
        sleep 15
      done
      echo "VM cloud-init complete!"
    EOT
  }

  triggers = {
    vm_id = proxmox_virtual_environment_vm.ubuntu_vm.id
  }
}

# Run Ansible to install IBM MQ
resource "null_resource" "ansible_mq_setup" {
  depends_on = [null_resource.wait_for_vm]

  provisioner "local-exec" {
    command = <<EOT
      ansible-playbook \
        -i '192.168.1.21,' \
        -u indunil \
        --private-key ~/.ssh/id_ed25519 \
        --ssh-extra-args='-o StrictHostKeyChecking=no' \
        ibmmq-ansible/playbook.yml
    EOT
  }

  triggers = {
    vm_id = proxmox_virtual_environment_vm.ubuntu_vm.id
  }
}

output "vm_ip"        { value = "192.168.1.21" }
output "vm_name"      { value = proxmox_virtual_environment_vm.ubuntu_vm.name }
output "mq_connection" {
  value = "Channel: DEV.APP.SVRCONN | Host: 192.168.1.21 | Port: 1414"
}
```

---

## Step 7 — Ansible Variables (group_vars/all.yml)

```yaml
ibmmq_version:      "9.3.0"
ibmmq_binary_name:  "IBM_MQ_9.3.0_LINUX_X86-64.tar.gz"
ibmmq_binary_src:   "~/mq-binaries/{{ ibmmq_binary_name }}"
ibmmq_install_dir:  "/opt/mqm"
ibmmq_tmp_dir:      "/tmp/mqinstall"

ibmmq_qmgr_name:    "QM1"
ibmmq_qmgr_port:    1414
ibmmq_qmgr_channel: "DEV.APP.SVRCONN"

ibmmq_group:        "mqm"
ibmmq_user:         "mqm"
ibmmq_user_uid:     1001
ibmmq_group_gid:    1001
ibmmq_user_home:    "/home/mqm"
ibmmq_user_shell:   "/bin/bash"

ibmmq_app_user:     "mqapp"
ibmmq_app_user_uid: 1002
ibmmq_app_user_home: "/home/mqapp"

ibmmq_nofile_limit: 10240
ibmmq_nproc_limit:  4096
ibmmq_data_dir:     "/var/mqm"
ibmmq_log_dir:      "/var/mqm/log"
```

---

## Step 8 — Ansible Role Tasks

The Ansible role is split into six task files run in order.

### 01 — Users and Groups

Creates the `mqm` administrator user, `mqapp` application user, and their groups.

```yaml
- name: Create mqm group
  ansible.builtin.group:
    name: "{{ ibmmq_group }}"
    gid: "{{ ibmmq_group_gid }}"
    state: present

- name: Create mqm user
  ansible.builtin.user:
    name: "{{ ibmmq_user }}"
    uid: "{{ ibmmq_user_uid }}"
    group: "{{ ibmmq_group }}"
    home: "{{ ibmmq_user_home }}"
    shell: "{{ ibmmq_user_shell }}"
    create_home: true
    state: present

- name: Create mqapp user
  ansible.builtin.user:
    name: "{{ ibmmq_app_user }}"
    uid: "{{ ibmmq_app_user_uid }}"
    groups: "{{ ibmmq_group }}"
    home: "{{ ibmmq_app_user_home }}"
    shell: /bin/bash
    create_home: true
    state: present
```

### 02 — System Configuration

Sets ulimits for the MQ users. The kernel parameters are already applied by cloud-init, so this step just verifies and sets PAM limits.

```yaml
- name: Set system limits for mqm user
  ansible.builtin.pam_limits:
    domain: "{{ ibmmq_user }}"
    limit_type: "{{ item }}"
    limit_item: nofile
    value: "{{ ibmmq_nofile_limit }}"
  loop: [soft, hard]
```

### 03 — Install IBM MQ

Copies the binary, extracts it, accepts the license, and installs packages in strict dependency order using `dpkg` with `--configure -a` after each to avoid pre-dependency errors.

```yaml
- name: Check if IBM MQ is already installed
  ansible.builtin.command:
    cmd: "/opt/mqm/bin/dspmqver"
  register: mq_already_installed
  changed_when: false
  failed_when: false

- name: Install packages in strict dependency order
  ansible.builtin.shell:
    cmd: "dpkg -i {{ ibmmq_tmp_dir }}/MQServer/{{ item }} && dpkg --configure -a"
  loop:
    - "ibmmq-runtime_*.deb"
    - "ibmmq-gskit_*.deb"
    - "ibmmq-server_*.deb"
    - "ibmmq-client_*.deb"
    - "ibmmq-sdk_*.deb"
    - "ibmmq-jre_*.deb"
    - "ibmmq-java_*.deb"
    - "ibmmq-man_*.deb"
    - "ibmmq-samples_*.deb"
  when: mq_already_installed.rc != 0
```

> **Gotcha:** The `ibmmq-*` package names differ from the older `MQSeries*` naming convention. Also, packages must be installed one at a time with `dpkg --configure -a` between each — installing them all at once causes pre-dependency failures because `ibmmq-runtime` must be fully configured before any other package can install.

### 04 — Environment

Sets `PATH`, `LD_LIBRARY_PATH`, and useful MQ aliases in `.bashrc` for both `mqm` and `mqapp` users, and writes a global `/etc/profile.d/ibmmq.sh`.

### 05 — Configure Queue Manager

Creates the queue manager, starts it, and applies the MQSC configuration using `su -l mqm` (the `-l` flag loads the full login environment which is required for MQ commands to work).

```yaml
- name: Create queue manager
  ansible.builtin.shell:
    cmd: "su -l mqm -c 'crtmqm -p {{ ibmmq_qmgr_port }} {{ ibmmq_qmgr_name }}'"
  when: qmgr_check.rc != 0

- name: Apply MQSC configuration
  ansible.builtin.shell:
    cmd: "su -l mqm -c 'runmqsc {{ ibmmq_qmgr_name }} < /tmp/mqsc_config.mqsc'"
```

The MQSC script creates:
- TCP listener on port 1414
- Server connection channel `DEV.APP.SVRCONN`
- Dead letter queue
- Local queues: `DEV.QUEUE.1`, `DEV.QUEUE.2`, `DEV.QUEUE.3`, `ADMIN.QUEUE`
- Channel authentication rules

### 06 — Systemd Service

Stops the queue manager gracefully, registers it with systemd, and starts it back up under systemd control so it auto-starts on every reboot.

```yaml
- name: Stop queue manager before handing to systemd
  ansible.builtin.shell:
    cmd: "su -l mqm -c 'endmqm -w {{ ibmmq_qmgr_name }} 2>/dev/null || true'"

- name: Enable and start IBM MQ service via systemd
  ansible.builtin.systemd:
    name: "ibmmq-{{ ibmmq_qmgr_name }}"
    enabled: true
    state: started
    daemon_reload: true
```

---

## Step 9 — Run It

```bash
cd /mnt/storage/proxmox-terraform

# Install dependencies
sudo apt install -y ansible
ansible-galaxy collection install ansible.posix

# Place your MQ binary
mkdir -p ~/mq-binaries
# copy IBM_MQ_9.3.0_LINUX_X86-64.tar.gz to ~/mq-binaries/

# Deploy everything
terraform init
terraform apply
```

The full run takes about **7 minutes**:

| Phase | Time |
|---|---|
| VM clone + boot | ~3m 42s |
| cloud-init wait | ~4s |
| Ansible (56 tasks) | ~3m 28s |

---

## Verification

After `terraform apply` completes, SSH into the VM and verify:

```bash
ssh indunil@192.168.1.21

# Check MQ version
sudo su -l mqm -c 'dspmqver'

# Check queue manager status
sudo su -l mqm -c 'dspmq'
# QMNAME(QM1)   STATUS(Running)

# Check systemd service
systemctl status ibmmq-QM1

# Check listener is up
sudo su -l mqm -c 'runmqsc QM1 <<< "DISPLAY LSSTATUS(*)"'
```

---

## Key Lessons Learned

**1. Proxmox API token permissions must be explicit**
Even with root@pam tokens, storage and SDN paths need explicit ACL entries. Easiest fix: grant Administrator at `/`.

**2. IBM MQ package install order matters**
The `ibmmq-*` packages have strict pre-dependencies. Install `ibmmq-runtime` first, configure it with `dpkg --configure -a`, then install `ibmmq-gskit`, then the rest in order.

**3. Kernel shared memory is critical**
The default `kernel.shmall=4096` on Ubuntu equals only 16MB of shared memory — far too low for IBM MQ. Setting it to `kernel.shmmax=268435456` (256MB) in cloud-init ensures it's ready before Ansible runs.

**4. Use `su -l` not `su -` for MQ commands**
`su -l mqm` loads the full login environment including PATH to `/opt/mqm/bin`. Without it, MQ commands fail.

**5. Ansible `become_user` doesn't work without ACL support**
Ubuntu 24.04 doesn't support the ACL chmod mode Ansible uses for privilege escalation to non-root users. Use `su -l mqm -c 'command'` in shell tasks instead.

---

## Teardown

```bash
terraform destroy
```

This removes the VM, its disks, and the cloud-init snippet from Proxmox cleanly.

---

## What's Next

- Add SSH key-based auth only (disable password)
- Add TLS to the MQ listener
- Create multiple queue managers with different configs using Terraform variables
- Integrate with a monitoring stack (Prometheus + Grafana)
