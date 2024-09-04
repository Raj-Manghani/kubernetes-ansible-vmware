# Ansible Playbooks for VMware vSphere vCenter VM Provisioning and Kubernetes Setup
*** These playbooks are designed to streamline the creation of Kubernetes clusters on VMware VMs using Kubeadm and Containerd. ***
## Overview

“This repository features Ansible playbooks that swiftly provision virtual machines via VMware (vSphere) and prepare them for Kubernetes cluster deployment using Kubeadm. The playbooks transform a vCenter VM template into a Kubernetes-ready VM. From there, users simply need to run `kubeadm init` to create a Kubernetes cluster with the Containerd runtime. The playbooks support various Linux distributions, including Ubuntu 24.04 and RHEL 9.4. If your preferred Linux distribution isn’t listed, let us know!”

Additionally, the playbooks provide an option to configure the new VMs for Calico CNI and Longhorn Persistent Volumes using `calico_cni: true/false` and `use_longhorn_pv: true/false` located in the `vars:` section of each playbooks.

## Prerequisites

+ vSphere 8.0 or above.  No prior versions have been tested, but may still work. (Feel free to test the playbooks out on previous versions and share your outcome.)

+ vSphere credentials with appropriate priviledges
  
+ Ansible host version 2.17 or later
  
+ `main.yml` file declaring playbook variables.  Refer to the example 'main.yml' for the required variables.
  
+ Public key or public key file paths for new VMs.  Your Ansible host requires the private key counterpart to access and further configure the VMs when executing playbooks.
    + Playbooks have the option to omit `root` user from obtaining public keys. See `vars:` for any playbook (e.g. `add_root_ssh_key: true` or `add_root_ssh_key: false`).
    + Playbooks have an option to provide a seperate public keys for `root` user. See `vars:` for any playbook (e.g. `root_key_path: "/root/.ssh/public_key.pub"` or `""` to use user public key).

+ vSphere vCenter VM template provisioning for Kubernetes vary by Linux distro. Refer to the `README` file located in specific Linux distro playbook folder for more information on how to configure vSphere VM template for a particular Linux distribution.

## VM Template Dependencies and Configuration

Specific configurations are required for VMware VM templates to create Kubernetes cluster nodes.  The Ansible playbooks in this repository have their own VM template requirements which allow Ansible to make these necessary configurations.  Some of these configurations are not suitable for productions environments and should be reversed after the VMs are created.  Generally, the VM template configurations will:

+ Configure ssh server for `root` and `user` access

+ Allow password authenticated ssh access.  Since playbooks insert public keys after VMs are created, username/password authentication is required for provisioning.

+ Define a single password via vSphere/Customize Template.

+ Configure VMs with static IPs, DNS servers and search domain.

+ Set SELinux to `permissive` mode. (Kubernetes Required)

*PLEASE NOTE: Since Linux distros vary, VM template configuration instructions are located in the `<LINUX DISTRO>-<distro version>-TEMPLATE.md` files of the specific distro folders.*

## Current Linux-based Playbooks

### 1. Ubuntu 24.04 Playbook

#### Description

This playbook provisions VMs from a vSphere template and configures them for Kubernetes. It includes tasks for network configuration, SSH key management, SELinux configuration, Kubernetes package installation and, other Kubernetes package dependencies (optional: package and configurations for Calico CNI and Longhorn).

#### Key Features

- Clone VMs from a template (For template creation see `UBUNTU-2404-TEMPLATE.md` located in ubuntu-2404 folder)
- Configure network settings using NetPlan
- Manage SSH keys for root and specified users (With option to omit or apply seperate key for `root` user)
- Set SELinux to permissive mode and disable swap
- Install and configure containerd
- Install Kubernetes packages (kubeadm, kubectl, kubelet)
- Install and configure Calico CNI (optional: Set boolean `true/false` directly in playbook)
- Configure VMs for Longhorn persistent volumes (optional: Set boolean `true/false` directly in playbook)
- Versioning: Select desired version of Kubernetes and Calico CNI

### 2. RHEL 9.4 Playbook

#### Description

This playbook provisions VMs from a vSphere template and configures them for Kubernetes. It includes tasks for network configuration, SSH key management, SELinux configuration, Kubernetes package installation and, other Kubernetes package dependencies (optional: package and configurations for Calico CNI and Longhorn).

#### Key Features

- Clone VMs from a template (For template creation see `RHEL-94-TEMPLATE.md` located in ubuntu-2404 folder)
- Configure network settings using NetworkManager
- Manage SSH keys for root and specified users (With option to omit or apply seperate key for `root` user)
- Set SELinux to permissive mode and disable swap
- Install and configure containerd
- Install Kubernetes packages (kubeadm, kubectl, kubelet)
- Install and configure Calico CNI (optional: Set boolean `true/false` directly in playbook)
- Configure VMs for Longhorn persistent volumes (optional: Set boolean `true/false` directly in playbook)
- Versioning: Select desired version of Kubernetes and Calico CNI

### 3. Variables

The playbooks use various variables defined in the `vars/main.yml` and in the `vars:` section in the playbook files.  VMware vSphere specific variables are located in `main.yml`.  While VM specific variables are located in the `vars:' section inside the playbook yaml files.

The `vm_list` variable located in playbook yaml files determines the desired quantity of VMs, their hostnames, and their static IP addresses.  Add VM(s) to the list by add another entry to the list.
```bash
  vm_list:
      - { name: kubernetes1, item_ip: 10.11.0.32 }
      - { name: kubernetes2, item_ip: 10.11.0.33 }
      - { name: kubernetes3, item_ip: 10.11.0.34 }
      - { name: <add-VM-hostname>, item_ip: <desired static IP>}
```

#### vSphere Variables 

Variables are located in `vars/main.yml`.  By default, playbooks will look for the vars file at as defined in playbook:
```bash
vars_files: 
  - /root/playbooks/vars/main.yml
```
vSphere Variables in main.yml include:
```bash
vcenter_ip: "<IP Address or FQDN>"
vcenter_username: "<administrator or other @vsphere.local>"
vcenter_password: "<username's password>"
datacenter_name: "<Datacenter-name>"
cluster_name: "<vSphere-Cluster-Name>" 
folder_name: "<desired-vSphere-folder-for-provisioned-VMs>"
vcenter_hostname: "<IP or FQDN of vCenter>"
template_name_rhel-94-1: "<name-of-vSphere-RHEL-9.4-template>"
template_name_rhel-94-2: "<"">"
template_name_rhel-94-3: "<''>"
template_name_ubuntu-2404-1: "<name-of-vSphere-Ubuntu-template>"
template_name_ubuntu-2404-2: "<''>"
template_name_ubuntu-2404-3: "<''>"
```

vm_list: List of VMs to be provisioned
dns_server_list: List of DNS servers
search_domain_name: DNS search domain
user: User for SSH access
vm_password: Password for VMs
public_key_path: Path to the public SSH key
root_key_path: Path to the root SSH key
kubernetes_version: Version of Kubernetes to install
calicoctl_version: Version of Calicoctl to install
use_longhorn_pv: Flag to enable Longhorn persistent volumes

### 4. Contributing
Contributions are welcome! Please fork the repository and submit a pull request with your changes.

### 5. License
This project is licensed under the MIT License. See the LICENSE file for details.

### 6. Contact
For any questions or issues, please open an issue in the repository or contact the maintainer.