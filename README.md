# linux_lcm

# 🚀 Libvirt Lifecycle Manager (LCM)

A declarative Ansible-based automation suite designed to manage the full lifecycle of virtual infrastructure on a remote hypervisor ("hypervisor"). This project allows you to define a **desired state** (e.g., "I want 3 web servers") and automatically reconciles the environment by adding or removing VMs to match.

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
Ubuntu Cloud Images: https://cloud-images.ubuntu.com/
Redhat Enterprise Linux Image builder: https://console.redhat.com/insights/image-builder#tags=

## Future plans
* **Expose additional settings:** Many settings, such as VM Memory, Number of CPUs, and Networking are currently hard coded and should be configurable.
* **Additional VM Templates:** Add additional templates to allow further customization of provisioned VMs.
* **Integrate with Jenkins:** Implement a Jenkins-driven CI/CD pipeline.

## Example run
This example shows the complete run when provisioning a VM.
```
jarkko@control:~/linux_lcm$ ansible-playbook provision.yml -e "vm_count=1 name_prefix=rhel10 vm_image=/data/sdc1_vol1/iso/rhel10.0_serial_console.qcow2" -K
BECOME password:

PLAY [Declarative VM Management] ************************************************************************************************************************************

TASK [Audit: Get list of existing VMs] ******************************************************************************************************************************
ok: [hypervisor]

TASK [Filter: Identify current lab VMs] *****************************************************************************************************************************
ok: [hypervisor]

TASK [Initialize Desired VM list] ***********************************************************************************************************************************
ok: [hypervisor]

TASK [Calculate: Desired VM list (only if count > 0)] ***************************************************************************************************************
ok: [hypervisor]

TASK [Calculate: Add/Remove lists] **********************************************************************************************************************************
ok: [hypervisor]

TASK [Action: Create missing VMs] ***********************************************************************************************************************************
included: /home/jarkko/linux_lcm/lifecycle_steps.yml for hypervisor => (item=rhel10-01)

TASK [Debug: Processing VM rhel10-01] *******************************************************************************************************************************
ok: [hypervisor] => {
    "msg": "State: present | Target: rhel10-01"
}

TASK [Shut down VM rhel10-01 from Libvirt] **************************************************************************************************************************
skipping: [hypervisor]

TASK [Undefine VM rhel10-01 from Libvirt] ***************************************************************************************************************************
skipping: [hypervisor]

TASK [Delete Disk Images for rhel10-01] *****************************************************************************************************************************
skipping: [hypervisor]

TASK [Delete Cloud-Init ISO for rhel10-01] **************************************************************************************************************************
skipping: [hypervisor]

TASK [Run Provisioning Role] ****************************************************************************************************************************************

TASK [libvirt_provision : Create Cloud-Init config directory] *******************************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Render user-data template] ****************************************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Create empty meta-data file] **************************************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Render network-config template] ***********************************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Remove old ISO if it exists] **************************************************************************************************************
ok: [hypervisor]

TASK [libvirt_provision : Generate Cloud-Init ISO] ******************************************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Cleanup: Remove temporary Cloud-Init source files] ****************************************************************************************
changed: [hypervisor] => (item=/tmp/rhel10-01_config/user-data)
changed: [hypervisor] => (item=/tmp/rhel10-01_config/meta-data)
changed: [hypervisor] => (item=/tmp/rhel10-01_config/network-config)
changed: [hypervisor] => (item=/tmp/rhel10-01_config)

TASK [libvirt_provision : Create VM disk using backing file (Copy-on-Write)] ****************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Define the VM from the XML template] ******************************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Ensure the VM is started] *****************************************************************************************************************
changed: [hypervisor]

TASK [libvirt_provision : Wait for VM to get an IP address] *********************************************************************************************************
FAILED - RETRYING: [hypervisor]: Wait for VM to get an IP address (10 retries left).
FAILED - RETRYING: [hypervisor]: Wait for VM to get an IP address (9 retries left).
FAILED - RETRYING: [hypervisor]: Wait for VM to get an IP address (8 retries left).
changed: [hypervisor]

TASK [Wait for rhel10-01 to get an IP] ******************************************************************************************************************************
changed: [hypervisor]

TASK [Add rhel10-01 to dynamic inventory] ***************************************************************************************************************************
changed: [hypervisor]

TASK [Action: Remove excess VMs] ************************************************************************************************************************************
skipping: [hypervisor]

PLAY RECAP **********************************************************************************************************************************************************
hypervisor                     : ok=22   changed=12   unreachable=0    failed=0    skipped=5    rescued=0    ignored=0
```