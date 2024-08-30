# Ansible Playbooks for VMware vSphere vCenter VM Provisioning and Kubernetes Setup

## Overview

This repository contains Ansible playbooks designed to QUICKLY provision virtual machines via VMware (vSphere) and prepare them for Kubernetes Kubeadm deployment. The playbook will take a vSphere vCenter VM template to a kubernetes ready VM.  From there, all a user needs to do is run `kubeadm init` to create a kubernetes cluster (with Containerd runtime). The playbooks support various Linux distributions, including Ubuntu 24.04 and RHEL 9.4.  If you don't see your preferred linux distro playbook, lets us know!

The playbooks provide an option to configure the new VMs for Calico CNI or Longhorn Persistent Volumes using `calico_cni: true/false` and `use_longhorn_pv: true/false` located in the `vars:` section of each playbooks.

## Prerequisites

+ vSphere 8.0 or above.  No prior versions have been tested, but may still work.  Feel free to test the playbooks out on previous versions and lets us know the outcome.

+ vSphere credentials with appropriate priviledges.
  
+ Ansible host version 2.17 or later.
  
+ `main.yml` file declaring playbook variables .  Please see example `main.yml` for reference and required variables.
  
+ Public key or public key file paths for new VMs.  Your Ansible host requires the private key counterpart to access and further configure the VMs.
    + Playbooks have the option to omit `root` user from obtaining public key embedded in `vars:` for each playbook (e.g. `add_root_ssh_key: true` or `add_root_ssh_key: false`).
    + Playbooks have an option to provide a seperate public key for `root` user embedded in `vars:` for each playbook (e.g. `root_key_path: "/root/.ssh/public_key.pub"` or `""`).

+ vSphere vCenter VM template dependencies vary between Linux distros.  Please refer to the `README` file located in specific Linux distro playbook folder for more information on how to configure a "Linux distro specific" vSphere VM template.

## VM Template Dependencies and Configuration

Specific configurations and dependencies are required on the VMware VM template.  Since Linux distros vary, VM configuration instructions are located in the `README` files of the specific distro folders.   Generally, the vm template configurations:
    + Configure ssh server for `root` and/or `user` access
    + Allow password authentication ssh access.  Since this playbook inserts public keys after the VM is created, username and password authentication is required.  This may be disabled after ssh keys are inserted. 
    + Define a password to access VMs.  This is accomplished by making changes to /etc/cloud/cloud.cfg (cloud-init).  Cloud-init may need to be installed depending on Linux distro chosen.  
    + 

## Playbooks

### 1. Ubuntu 24.04 Playbook

#### Description

This playbook provisions VMs from a template and configures them for Kubernetes. It includes tasks for network configuration, SSH key management, SELinux configuration, and Kubernetes package installation.

#### Key Features

- Clone VMs from a template
- Configure network settings using NetPlan
- Manage SSH keys for root and specified users
- Set SELinux to permissive mode and disable swap
- Install and configure containerd
- Install Kubernetes packages (kubeadm, kubectl, kubelet)
- Install and configure Calico CNI
- Configure VMs for Longhorn persistent volumes

#### Usage

To run the playbook, use the following command:

```bash
ansible-playbook -i inventory ubuntu_playbook.yml

2. RHEL 9.4 Playbook
Description
This playbook provisions VMs from a template and configures them for Kubernetes. It includes tasks for network configuration, SSH key management, SELinux configuration, and Kubernetes package installation.

Key Features
Clone VMs from a template
Configure network settings using NetworkManager
Manage SSH keys for root and specified users
Set SELinux to permissive mode and disable swap
Install and configure containerd
Install Kubernetes packages (kubeadm, kubectl, kubelet)
Install and configure Calico CNI
Configure VMs for Longhorn persistent volumes
Usage
To run the playbook, use the following command:

ansible-playbook -i inventory rhel_playbook.yml

Variables
The playbooks use various variables defined in the vars/main.yml file. Key variables include:

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
Contributing
Contributions are welcome! Please fork the repository and submit a pull request with your changes.

License
This project is licensed under the MIT License. See the LICENSE file for details.

Contact
For any questions or issues, please open an issue in the repository or contact the maintainer.


Feel free to customize this README to better fit your needs! Let me know if there's anything else you'd like to add or modify.
