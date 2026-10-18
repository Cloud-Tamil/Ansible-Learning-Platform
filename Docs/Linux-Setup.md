# Ansible Hands-on Lab: Two Ubuntu VMs

A beginner-friendly lab that uses two Ubuntu Server VMs in VirtualBox to learn Ansible basics: SSH key authentication, inventory, ad-hoc commands, and a first playbook that installs Nginx.

> **Note:** All IP addresses in this guide (`192.168.56.X`, `192.168.56.X`) are examples. Always use the IPs actually assigned to your VMs.

---

## Table of Contents

1. [Lab Architecture](#1-lab-architecture)
2. [What You Will Learn](#2-what-you-will-learn)
3. [Prerequisites](#3-prerequisites)
4. [Step 1: Create the Virtual Machines](#step-1-create-the-virtual-machines)
5. [Step 2: Configure the Network](#step-2-configure-the-network)
6. [Step 3: Install Ansible (Control Node)](#step-3-install-ansible-control-node)
7. [Step 4: Create the Automation User](#step-4-create-the-automation-user)
8. [Step 5: Configure Passwordless SSH](#step-5-configure-passwordless-ssh)
9. [Step 6: Configure Passwordless sudo](#step-6-configure-passwordless-sudo)
10. [Step 6: Configure Passwordless sudo](#step-6-configure-privillage-sudo)
11. [Step 7: Create the Inventory](#step-7-create-the-inventory)
12. [Step 8: Test Connectivity](#step-8-test-connectivity)
13. [Step 9: Write Your First Playbook](#step-9-write-your-first-playbook)
14. [Troubleshooting](#troubleshooting)
15. [Cleanup](#cleanup)

---

## 1. Lab Architecture

```
                 Your laptop (Oracle VirtualBox)
        ┌──────────────────────────────────────────┐
        │                                          │
        │   VM 1: ansible-control   VM 2: ansible-node1
        │   ┌──────────────────┐    ┌──────────────────┐
        │   │ Control node     │    │ Managed node     │
        │   │ Ansible + SSH    │───▶│ SSH server +     │
        │   │ client           │ SSH│ Python           │
        │   │ 192.168.56.X    │    │ 192.168.56.X    │
        │   └──────────────────┘    └──────────────────┘
        │          Host-only network (192.168.56.X/24)
        └──────────────────────────────────────────┘
```

| Node | Role | Purpose |
|------|------|---------|
| **VM 1: Control node** | Automation server | Runs Ansible, stores inventory and playbooks, connects to VM 2 over SSH |
| **VM 2: Managed node** | Target server | Runs an SSH server; Ansible executes commands and installs software (e.g., Nginx) on it |

> Ansible is installed **only** on the control node. The managed node only needs SSH and Python.

---

## 2. What You Will Learn

| Step | Topic | Outcome |
|------|-------|---------|
| 1 | Create two Ubuntu VMs | Two working servers |
| 2 | Configure the network | Both VMs can communicate |
| 3 | Install Ansible | Control node ready |
| 4 | Create an automation user | Dedicated SSH user |
| 5 | Configure SSH keys | Passwordless SSH login |
| 6 | Configure sudo | Privileged automation |
| 7 | Create an inventory | Register managed hosts |
| 8 | Run Ansible commands | Verify connectivity |
| 9 | Create a playbook | Install and start Nginx |

---

## 3. Prerequisites

- A computer with [Oracle VirtualBox](https://www.virtualbox.org/) installed
- [Ubuntu Server 24.04 LTS ISO](https://ubuntu.com/download/server)
- Basic comfort with the Linux command line
- At least 4 GB of free RAM and 40 GB of free disk space for both VMs

---

## Step 1: Create the Virtual Machines

> Already have two Ubuntu VMs? Skip to [Step 2](#step-2-configure-the-network).

Recommended resources:

| Resource | VM 1 | VM 2 |
|----------|------|------|
| VM name | `ansible-control` | `ansible-node1` |
| OS | Ubuntu Server 24.04 LTS | Ubuntu Server 24.04 LTS |
| RAM | 2 GB | 2 GB |
| CPU | 2 cores | 2 cores |
| Disk | 20 GB | 20 GB |

In VirtualBox:

1. Click **New** and create `ansible-control`.
2. Select the Ubuntu ISO and complete the installation.
3. Repeat for `ansible-node1`.
4. Set a username and password during installation, and keep these initial admin credentials handy.

---

## Step 2: Configure the Network

The VMs need to talk to each other and also reach the internet to install packages. The simplest setup is **NAT + Host-only**:

- **NAT** (Adapter 1): internet access
- **Host-only** (Adapter 2): private network between the two VMs

For **each** VM:

1. Shut down the VM.
2. Open **VirtualBox → select the VM → Settings → Network**.
3. **Adapter 1:** enable it and select **NAT**.
4. **Adapter 2:** enable it and select **Host-only Adapter** (or the equivalent Host-only Network option in your VirtualBox version).
5. Choose the **same** Host-only network for both VMs.
6. Save and start the VMs.

If no Host-only network exists, create one in VirtualBox's **Network Manager** (in some versions under **Tools**).

### Check the IP addresses

In each VM, run:

```bash
ip -br addr
```

Look for the Host-only interface (often `enp0s8`). Example:

```
enp0s8    UP    192.168.56.X/24
```

Record the Host-only IP of each VM:l 

- Control VM: `CONTROL_IP` `Replace actual IP`
- Managed VM: `NODE1_IP`   `Replace actual IP`

### Test connectivity

From the control VM:

```bash
ping -c 4 NODE1_IP
```

Replace `NODE1_IP` with the real address, for example `192.168.56.11`. If the ping fails, confirm both VMs use the same Host-only network and have valid IPs before continuing (see [Troubleshooting](#troubleshooting)).

---

## Step 3: Install Ansible (Control Node)

Run on **ansible-control** using your initial Ubuntu admin account:

```bash
sudo apt update
sudo apt install -y ansible openssh-client
```

Verify:

```bash
ansible --version
ssh -V
```

---

## Step 4: Create the Automation User

We use a dedicated user named `ansible` on both machines.

### On VM 1: Control node

```bash
sudo adduser ansible
sudo usermod -aG sudo ansible
```

Choose a password when prompted (the personal info fields can be left blank). Then switch to the new account:

```bash
su - ansible
whoami
```

Expected output: `ansible`

### On VM 2: Managed node

Log in with your initial admin account and run:

```bash
sudo apt update
sudo apt install -y openssh-server python3 sudo
sudo systemctl enable --now ssh
```

Create the automation user:

```bash
sudo adduser ansible
sudo usermod -aG sudo ansible
```

Check that SSH is running (press `q` to exit):

```bash
sudo systemctl status ssh
```

---

## Step 5: Configure Passwordless SSH

Ansible connects over SSH. Key-based authentication means no password prompt for every connection.

### 5.1 Generate an SSH key on VM 1

As the `ansible` user on the control node:

```bash
ssh-keygen -t ed25519
```

Press **Enter** to accept the default location, and **Enter** again to leave the passphrase empty (acceptable for this lab only). Keys are created at:

```
cd .ssh
/home/ansible/.ssh/id_ed25519       # private key (never share)
/home/ansible/.ssh/id_ed25519.pub   # public key
```

### 5.2 Copy the public key to VM 2

```bash
ssh-copy-id ansible@NODE1_IP
```

Example:

```bash
ssh-copy-id ansible@192.168.56.X
```

Enter the `ansible` user's password on VM 2 when prompted.

### 5.3 Test the login

```bash
ssh ansible@NODE1_IP
hostname
whoami
exit
```

You should land in the managed node's shell with no password prompt, and `whoami` should print `ansible`.

---

## Step 6: Configure Passwordless sudo

Ansible needs elevated privileges to install packages and manage services. On the **managed node (VM 2)**, using an admin account:

```bash
echo 'ansible ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible
sudo chmod 440 /etc/sudoers.d/ansible
sudo visudo -cf /etc/sudoers.d/ansible
```

Expected output:

```
/etc/sudoers.d/ansible: parsed OK
```

> ⚠️ **Security warning:** This gives the `ansible` user unrestricted root access without a password. Use it **only** on disposable lab VMs. Production systems should use tightly scoped sudo rules.

---

## Step 7: Create the Inventory

An inventory tells Ansible which machines to manage. On VM 1, as the `ansible` user:

```bash
mkdir -p ~/ansible-lab
cd ~/ansible-lab
nano inventory.ini
```

Add the following, replacing the IP with your managed node's real address:

```ini
[webservers]
node1 ansible_host=192.168.56.11 ansible_user=ansible
```

Save in Nano with `Ctrl+O`, `Enter`, then `Ctrl+X`.

| Part | Meaning |
|------|---------|
| `[webservers]` | Inventory group name |
| `node1` | Friendly alias for the managed machine |
| `ansible_host` | Real IP address of the managed node |
| `ansible_user` | Linux account Ansible uses to connect |

Check the inventory:

```bash
ansible-inventory -i inventory.ini --list
```

---

## Step 8: Test Connectivity

On VM 1:

```bash
cd ~/ansible-lab
ansible all -i inventory.ini -m ping
```

Expected output:

```
node1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

`SUCCESS` means Ansible connected to the managed node over SSH, ran its `ping` module, and got a reply. This is an Ansible connectivity test, **not** an ICMP network ping.

Try a few ad-hoc commands:

```bash
ansible all -i inventory.ini -m command -a "hostname"
ansible all -i inventory.ini -m command -a "cat /etc/os-release"
```

---

## Step 9: Write Your First Playbook

A playbook describes the desired state of your servers in YAML. This one installs Nginx and makes sure it is running.

Create `install_nginx.yml` in `~/ansible-lab`:

```yaml
---
- name: Install and start Nginx
  hosts: webservers
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure Nginx is started and enabled at boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Optionally check the syntax first:

```bash
ansible-playbook -i inventory.ini install_nginx.yml --syntax-check
```

Run the playbook:

```bash
ansible-playbook -i inventory.ini install_nginx.yml
```

Verify from the control node:

```bash
curl http://NODE1_IP
```

You should see the HTML of the default "Welcome to nginx!" page.

### Idempotency

Run the playbook a second time. Tasks should now report `ok` instead of `changed`, because Ansible only makes changes when the system differs from the desired state.

---

## Troubleshooting

| Problem | What to check |
|---------|---------------|
| **Host-only interface has no IP** | Confirm Adapter 2 is enabled and set to the same Host-only network on both VMs. If the adapter was added after installation, configure it in `/etc/netplan/` (e.g., enable DHCP for `enp0s8`) and run `sudo netplan apply`. |
| **`ping` fails between VMs** | Verify both VMs are on the same Host-only network and the IPs are in the same subnet. |
| **`ssh-copy-id` fails** | Confirm the SSH server is running on VM 2 (`sudo systemctl status ssh`) and the `ansible` user exists with a password set. |
| **Ansible `UNREACHABLE`** | Test `ssh ansible@NODE1_IP` manually. Check `ansible_host` and `ansible_user` in `inventory.ini`. |
| **`sudo: a password is required`** | Re-check [Step 6](#step-6-configure-passwordless-sudo) on the managed node. |
| **`/usr/bin/python3: not found`** | Install Python on the managed node: `sudo apt install -y python3`. |
| **Host key verification prompt** | Type `yes` the first time you connect, or connect once manually with `ssh` so the host key is saved. |

---

## Cleanup

To reset the lab, remove the Nginx package from the managed node:

```bash
ansible all -i inventory.ini -b -m ansible.builtin.apt -a "name=nginx state=absent purge=true"
```

When you are finished, you can delete the VMs from VirtualBox. Since the `NOPASSWD` sudo rule is for lab use only, do not reuse these VMs for anything sensitive.

---

## Next Steps

- Add more managed nodes to `inventory.ini`
- Use variables and templates (Jinja2)
- Organize automation into roles
- Learn `ansible-vault` for secrets
- Explore the [official Ansible documentation](https://docs.ansible.com/)
