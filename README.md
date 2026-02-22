# linux_lcm

# 🚀 Libvirt Lifecycle Manager (LCM)

A declarative Ansible-based automation suite designed to manage the full lifecycle of virtual infrastructure on a remote hypervisor ("Debbie"). This project allows you to define a **desired state** (e.g., "I want 3 web servers") and automatically reconciles the environment by adding or removing VMs to match.

## 🏗 Key Features
* **Declarative Scaling:** Simply provide the `guest_count` and the playbook calculates whether to provision new nodes or decommission excess ones.
* **Cloud-Native Provisioning:** Uses standard Cloud-Init (NoCloud) images. The automation dynamically generates a custom configuration (hostname, SSH keys, user data) and packages it into a CI-Data ISO.
* **Automated CD-ROM Attachment:** Every VM is provisioned with a secondary virtual CD-ROM drive containing the Cloud-Init ISO, ensuring the OS is fully initialized and reachable via SSH on the very first boot.
* **Optimized Storage (Backing Stores):** Utilizes QCOW2 Linked Clones with lightweight backing store (Copy-on-Write) for near-instant provisioning and reduced storage footprint on the hypervisor.
* **Intelligent Reconciliation:** Uses set-theory logic (`difference` filters) to determine exactly which VMs need action.
* **SSH Tunneling:** Automatically handles complex `ProxyCommand` and `StrictHostKeyChecking` logic to reach VMs hidden inside private NAT/Bridge networks.

---

## 📋 Prerequisites

### Control Node
* **Ansible 2.15+**
* `community.libvirt` collection installed: 
    ```bash
    ansible-galaxy collection install community.libvirt
    ```

### Hypervisor 
* Libvirt/KVM installed and running.
* `python3-libvirt` and `qemu-utils` packages.
* SSH Key access for the Ansible user.

---

## ⚙️ Configuration (`group_vars/all/vars.yml`)

The project is driven by centralized variables. Change these defaults to match your environment.

| Variable | Default | Description |
| :--- | :--- | :--- |
| `guest_count` | `1` | The **Desired State** number of VMs to be running. |
| `vm_name_prefix` | `ubuntu-web` | Prefix used to identify and manage specific VM sets. |
| `libvirt_uri` | `qemu:///system` | The connection URI for the remote hypervisor. |
| `base_image_source` | (Ubuntu 24.04 Path) | Path to the cloud-image `.img` file on the hypervisor. |

---

## 🚀 Usage

### 1. Provision / Scale Up / Scale Down
To ensure a specific number of VMs exist (e.g., 3):
```
<control_node> $ ansible-playbook provision.yml -e "guest_count=3" -K
```
results in:
```
<hypervisor_node> $ virsh list
 Id   Name            State
-------------------------------
 76   ubuntu-web-02   running
 77   ubuntu-web-03   running
 78   ubuntu-web-01   running
```
To add 2 more VMs for a total of 5:
```
<control_node> $ ansible-playbook provision.yml -e "guest_count=5" -K
```
results in:
```
<hypervisor_node> $ virsh list
 Id   Name            State
-------------------------------
 76   ubuntu-web-02   running
 77   ubuntu-web-03   running
 78   ubuntu-web-01   running
 79   ubuntu-web-05   running
 80   ubuntu-web-04   running
```
To remove all VMs for a total of 0:
```
<control_node> $ ansible-playbook provision.yml -e "guest_count=0" -K
```
results in:
```
<hypervisor_node> $ virsh list
 Id   Name            State
-------------------------------
```

### 2. Expanded Usage
To add 2 VMs with the name prefix of "rhel10" and the location of the backing store at "/data/sdc1_vol1/iso/rhel10.0_serial_console.qcow2":
```
<control_node> $ ansible-playbook provision.yml -e "vm_count=2 name_prefix=rhel10 vm_image=/data/sdc1_vol1/iso/rhel10.0_serial_console.qcow2" -K
```
results in:
```
<hypervisor_node> $ virsh list
 Id   Name            State
-------------------------------
 81   rhel10-01       running
 82   rhel10-02       running
```
To reduce the number of VMs you do not need to provide the name_prefix:
```
<control_node> $ ansible-playbook provision.yml -e "vm_count=1 name_prefix=rhel10" -K
```
results in:
```
<hypervisor_node> $ virsh list
 Id   Name            State
-------------------------------
 81   rhel10-01       running
```

## Useful links
Redhat Enterprise Linux Image builder:
https://console.redhat.com/insights/image-builder#tags=

## Future plans
* **Expose additional settings** Many settings , such as VM Memory, Number of CPUs, and Newroking are currently hard coded and should be configurable.
* **Integrate with Jenkins** Implement a Jenkins-driven CI/CD pipeline.