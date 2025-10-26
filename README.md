# CloudStack Nested Virtualization with Ansible

[![CloudStack](https://img.shields.io/badge/CloudStack-4.21-orange?logo=apache&logoColor=white)](https://cloudstack.apache.org/)
[![Ansible](https://img.shields.io/badge/Ansible-2.10+-blue?logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Platform](https://img.shields.io/badge/Platform-Ubuntu%2022.04%2B%20%7C%20Rocky%209-green)](https://github.com/aloonj/nested-cloudstack-ansible)
[![Hypervisor](https://img.shields.io/badge/Hypervisor-KVM%20(Nested)-red?logo=linux&logoColor=white)](https://www.linux-kvm.org/)
[![Purpose](https://img.shields.io/badge/Purpose-Testing%20%2F%20Development-yellow)](https://github.com/aloonj/nested-cloudstack-ansible)
[![License](https://img.shields.io/badge/License-Unlicense-lightgrey)](LICENSE)

Ansible playbook for deploying Apache CloudStack on nested VMs. Useful for testing and demos without needing multiple physical hosts.

**⚙️ Simple configuration:**
- `inventory/group_vars/all/main.yml` - Network settings, zone config, VM specs, etc.
- `inventory/group_vars/all/secrets.yml` - Passwords, API keys (gitignored)

## TL;DR

```bash
# 1. Configure secrets (passwords, SSH key)
cp inventory/group_vars/all/secrets.yml.example inventory/group_vars/all/secrets.yml
vim inventory/group_vars/all/secrets.yml  # Set passwords and add your SSH public key

# 2. Deploy infrastructure (30-60 minutes)
ansible-playbook -i inventory/hosts.yml deploy-cloudstack.yml --ask-become-pass

# 3. Generate API keys in UI at http://192.168.100.10:8080/client (admin/password)
#    Add them to secrets.yml, then configure the zone
ansible-playbook -i inventory/hosts.yml configure-zone.yml
```

## What This Does

Deploys a complete CloudStack environment on nested VMs in a single KVM host:

**Phase 1-2: Infrastructure Setup**
- Creates 4 nested VMs with cloud-init configuration
  - `cs-mgmt` (192.168.100.10) - Management server + MySQL
  - `cs-nfs` (192.168.100.11) - NFS storage
  - `cs-kvm01` (192.168.100.20) - KVM compute host
  - `cs-kvm02` (192.168.100.21) - KVM compute host
- Configures SSH keys, hostnames, networking, and system updates

**Phase 3-4: Storage & Database**
- Sets up NFS server with primary and secondary storage exports
- Installs and configures MariaDB with CloudStack databases

**Phase 5: CloudStack Management**
- Installs CloudStack management server packages
- Initializes CloudStack database schemas
- Configures and starts management server (accessible on port 8080)

**Phase 6-7: Compute Hosts**
- Mounts NFS storage on KVM compute hosts
- Installs and configures CloudStack KVM agent on compute nodes
- Sets up SSH key authentication between management and compute hosts

**Optional: Zone Configuration** (via `configure-zone.yml`)
- Programmatically creates zone, pod, and cluster via CloudStack API
- Adds KVM compute hosts to the cluster
- Configures primary and secondary storage
- Enables the zone and downloads system VM templates
- Creates ready-to-use CloudStack environment

## Network Configuration

All infrastructure and guest VMs share the **192.168.100.0/24** network:
- **192.168.100.1** - Gateway (csbr0 bridge on physical host)
- **192.168.100.10-24** - Infrastructure VMs (management, NFS, compute hosts)
- **192.168.100.25-99** - Reserved for pod IPs (System VMs: SSVM, CPVM, Virtual Routers)
- **192.168.100.100-254** - Guest VMs launched by CloudStack users

The KVM compute hosts use `cloudbr0` bridge (attached to `enp1s0`) for CloudStack guest networking.

## Requirements

**Physical Host:**
- Ubuntu 22.04+ or Debian 11+ with KVM
- AMD or Intel CPU with nested virtualization enabled
- 20GB+ RAM (32GB recommended)
- 200GB+ free disk space in `/var/lib/libvirt/images`

**Check nested virtualization:**
```bash
# AMD
cat /sys/module/kvm_amd/parameters/nested  # Should show 'Y' or '1'

# Intel
cat /sys/module/kvm_intel/parameters/nested  # Should show 'Y' or '1'
```

**Install dependencies:**
```bash
sudo apt update
sudo apt install -y ansible python3-libvirt libvirt-daemon-system \
                     libvirt-clients virtinst qemu-kvm qemu-utils \
                     genisoimage cloud-utils
ansible-galaxy collection install community.libvirt
```

## Setup

1. **Create secrets file:**
   ```bash
   cp inventory/group_vars/all/secrets.yml.example inventory/group_vars/all/secrets.yml
   vim inventory/group_vars/all/secrets.yml  # Set passwords and SSH public key (required)
   ```

2. **(Optional) Edit `inventory/group_vars/all/main.yml`:**
   - Change hypervisor platform (`hypervisor_type` - currently only `kvm_libvirt` supported)
   - Change VM guest OS (Rocky 9 or Ubuntu 22.04)
   - Adjust network settings, zone config, etc.

3. **Deploy infrastructure:**
```bash
ansible-playbook -i inventory/hosts.yml deploy-cloudstack.yml --ask-become-pass
```

Takes 30-60 minutes. Access UI at http://192.168.100.10:8080/client (admin/password)

4. **Generate API keys and configure zone automatically:**
   - Login to CloudStack UI: http://192.168.100.10:8080/client (admin/password)
   - Navigate to: Accounts → admin → View Users → Click on admin user → Generate Keys
   - Copy the API Key and Secret Key
   - Edit `inventory/group_vars/all/secrets.yml` and add your keys:
     ```yaml
     cloudstack_api_key: "YOUR_API_KEY_HERE"
     cloudstack_secret_key: "YOUR_SECRET_KEY_HERE"
     ```
   - Run the zone configuration playbook:
     ```bash
     ansible-playbook -i inventory/hosts.yml configure-zone.yml
     ```

The zone configuration will automatically:
- Create zone, pod, and cluster
- Add KVM hosts (using SSH key authentication - automatically configured during deployment)
- Configure primary and secondary storage
- Enable the zone and trigger system VM template downloads (15-30 minutes)

**Note:** SSH key authentication is automatically set up during deployment. The management server's `cloud` user can connect to KVM hosts as `root` without a password.

## Guest OS Options

Edit `inventory/group_vars/all.yml` to change the guest OS for nested VMs.

**Rocky Linux 9 (default):**
```yaml
vm_os: "rocky9"
vm_base_image_url: "https://download.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2"
vm_base_image_name: "rocky-9-cloudimg.qcow2"
```

**Ubuntu 22.04:**
```yaml
vm_os: "ubuntu22"
vm_base_image_url: "https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img"
vm_base_image_name: "ubuntu-22.04-cloudimg.qcow2"
```

## Complete Teardown (Destroy Everything)

**⚠️ WARNING: These commands will permanently destroy all VMs, disks, and networks. Only run this to completely remove the deployment.**

```bash
for vm in cs-mgmt cs-nfs cs-kvm01 cs-kvm02; do
  virsh destroy $vm 2>/dev/null
  virsh undefine $vm --nvram
done
rm -rf /var/lib/libvirt/images/cloudstack/
rm -rf /var/lib/libvirt/qemu/nvram/cs-*_VARS.fd
virsh net-destroy cloudstack-net
virsh net-undefine cloudstack-net
```

## Notes

- **Testing only** - Not for production use
- **Simple configuration** - Non-sensitive config in `main.yml`, secrets in `secrets.yml` (gitignored)
- **Basic networking** - Uses NAT with all infrastructure, system VMs, and guest VMs on the same flat 192.168.100.0/24 network. Simplified setup ideal for nested virtualization testing.
- **CloudStack 4.21** by default (change `cloudstack_version` in group_vars)
- **UEFI boot** - Nested VMs use UEFI (required for Rocky 9)
