![kubernetes-ansible-vmare](https://github.com/user-attachments/assets/557d72a8-3177-4d3f-8487-3fbd5f1fcf62)
<sub>*The repository's contents are not officially endorsed by VMware, Ansible, Kubernetes, Red Hat or Canonical*</sub>
# Ansible Playbooks for VMware vSphere vCenter VM Provisioning and Kubernetes Setup
These playbooks are designed to streamline the creation of Kubernetes clusters on VMware (vSphere) VMs using Kubeadm and ContainerD.

## Overview

This repository features Ansible playbooks that swiftly provision virtual machines via VMware (vSphere) and prepare them for Kubernetes cluster deployment using Kubeadm. The playbooks transform a vCenter VM template into a Kubernetes-ready VM. From there, users simply need to run `kubeadm init` to create a Kubernetes cluster that uses ContainerD runtime. The playbooks support various Linux distributions, including Ubuntu 24.04 and RHEL 9.4. If your preferred Linux distribution isn’t listed, let us know! Additionally, the playbooks provide an option to configure the new VMs for Calico CNI and Longhorn persistent volumes. 

## Prerequisites

+ vSphere 8.0 or above.  (No prior vSphere versions have been tested, but may still work. If you do try the playbooks on previous versions, please share the outcome with us.)

+ vSphere credentials with appropriate priviledges
  
+ Ansible host version 2.17 or later (Later versions may not be backward compatible).  See requirements.txt for additional packages and versions.  It is recommended to run these playbooks and additional packages in `python3 venv`.
  
+ `main.yml` file declaring vSphere variables.  Refer to the example 'main.yml' for the required variables.
  
+ 1 to 2 public ssh keys or public ssh key file paths for the new VMs.  Your Ansible host requires the private key counterpart to access and further configure the VMs when executing playbooks.
    + Playbooks have the option to omit `root` user from obtaining public keys. See `vars:` for any playbook (e.g. `add_root_ssh_key: true` or `add_root_ssh_key: false`).
    + Playbooks have an option to provide a seperate public keys for `root` user. See `vars:` for any playbook (e.g. `root_key_path: "/root/.ssh/public_key.pub"` or `""` to use 'user public key').

+ vSphere vCenter Linux VM template.  (Refer to the `README` file located in specific Linux distro playbook folder for more information on how to configure vSphere VM template for a particular Linux distribution.)

+ These playbooks are designed to run without declaring a `ansible.cfg` file.  If the playbook fails for any reason, remove `ansible.cfg` during runtime.

## VM Template Dependencies and Configuration

Specific configurations are required for VMware VM templates to create Kubernetes cluster nodes.  The Ansible playbooks in this repository have their own VM template dependencies. These template dependencies allow Ansible to make the necessary configurations to the VMs that result in a Kubernetes-ready VM.  

Some of these configurations are not suitable for productions environments and should be reversed after the VMs are created.  Generally, the VM template configurations will:

+ Configure ssh server for `root` and `user` access

+ Allow password authenticated ssh access.  Since playbooks insert public keys after VMs are created, username/password authentication is required for provisioning.

+ Define a single password via vSphere-Customize Template. 

+ Configure VMs with static IPs, DNS servers and search domain.

+ Set SELinux to `permissive` mode. (Kubernetes Required)

+ On applicable distros will set `%sudo  ALL=(ALL) NOPASSWD:ALL`

*PLEASE NOTE: Since Linux distros vary, VM template configuration instructions are located in a readme file in the specific distro folders (e.g.`<LINUX DISTRO>-<distro version>-TEMPLATE.md`).*

## Ansible Host Requirements

These playbooks were tested with Ansible Host version 2.17.  To ensure all modules work as intended, specific packages are required before running this playbook.  Refer to `requirements.txt` for more information on the packages required.  It is highly recommended you use a python virtual environment when running playbooks.  If you are unsure how to set this up, there are many online video walkthrus/tutorials.  Additionally, `python3` path needs to be set accordingly in `vars:` section as interpreter in your playbook for it to work as intended (e.g vars.ansible_python_interpreter: /usr/bin/python3).

## Current Linux-based Playbooks

![ansible-vmware-kubernetes-playbooks](https://github.com/user-attachments/assets/3d78de53-0788-448d-aa0b-45d34ac02fa2)
<sub>*Playbooks are user generated and not endorsed by Canonical*</sub>
### 1. Ubuntu 24.04 Playbook

#### Description

This playbook provisions VMs from a vSphere template and configures them for Kubernetes. It includes tasks for network configuration, SSH key management, SELinux configuration, Kubernetes package installation and, other Kubernetes package dependencies (optional: package and configurations for Calico CNI and Longhorn).

#### Key Features

- Clone VMs from a template (For template creation see `UBUNTU-2404-TEMPLATE.md` located in ubuntu-2404 folder)
- Configure network settings using NetPlan
- Manage SSH keys for root and specified users (With option to omit or apply seperate key for `root` user)
- Set SELinux to permissive mode and disable swap
- Install and configure ContainerD
- Install Kubernetes packages (kubeadm, kubectl, kubelet)
- Install and configure Calico CNI (optional: Set boolean `true/false` directly in playbook)
- Configure VMs for Longhorn persistent volumes (optional: Set boolean `true/false` directly in playbook)
- Versioning: Select desired version of Kubernetes and Calico CNI (Set directly in playbook)

![ansible-vmware-kubernetes-playbooks (1)](https://github.com/user-attachments/assets/a23b76eb-9ca4-4974-869f-787fdfe02e97)
<sub>*Playbooks are user generated and not endorsed by Red Hat.*</sub>
### 2. RHEL 9.4 Playbook

#### Description

This playbook provisions VMs from a vSphere template and configures them for Kubernetes. It includes tasks for network configuration, SSH key management, SELinux configuration, Kubernetes package installation and, other Kubernetes package dependencies (optional: package and configurations for Calico CNI and Longhorn).

#### Key Features

- Clone VMs from a template (For template creation see `RHEL-94-TEMPLATE.md` located in ubuntu-2404 folder)
- Configure network settings using NetworkManager
- Manage SSH keys for root and specified users (With option to omit or apply seperate key for `root` user)
- Set SELinux to permissive mode and disable swap
- Install and configure ContainerD
- Install Kubernetes packages (kubeadm, kubectl, kubelet)
- Install and configure Calico CNI (optional: Set boolean `true/false` directly in playbook)
- Configure VMs for Longhorn persistent volumes (optional: Set boolean `true/false` directly in playbook)
- Versioning: Select desired version of Kubernetes with cooresponding pause image version, and Calico CNI version (set directly in playbook)

## Playbook Variables

The playbooks' variables are defined in 2 places; in `vars/main.yml` and, the `vars:` section of the playbook file.  VMware vSphere specific variables are located in `main.yml`.  VM specific and all other variables are located in the `vars:' section inside the playbook.

### Playbook Variables (`vars:`)

The variables below are located in the `vars` section in every playbook.
```bash
vars:
    ansible_ssh_pass: "{{ vm_password }}"
    ansible_python_interpreter: /usr/bin/python3
    vm_list:
      - { name: kubernetes-cp-1, item_ip: 10.11.0.35, ram: 8192, disk: 140, cpu: 2}
      - { name: kubernetes-cp-2, item_ip: 10.11.0.36, ram: 4096, disk: 140, cpu: 2 }
      - { name: kubernetes-cp-3, item_ip: 10.11.0.37, ram: 4096, disk: 140, cpu: 2 }
      - { name: kubernetes-wk-1, item_ip: 10.11.0.38, ram: 16384, disk: 140, cpu: 2 }
      - { name: kubernetes-wk-2, item_ip: 10.11.0.39, ram: 16384, disk: 140, cpu: 2 }
      - { name: kubernetes-wk-3, item_ip: 10.11.0.40, ram: 16384, disk: 140, cpu: 2 }
    dns_server_list:
      - 192.168.50.246
      - 192.168.50.1
      - 10.11.0.1
    search_domain_name: "<mydomain.com>"
    user: "<user or root>"
    vm_password: "<password entered vSphere --> Customize Template>"
    default_gateway: "<gateway IP address>"
    public_key_path: "<path to ssh public key (i.e./root/.ssh/u-pub.pub)>"
    root_key_path: "<path to ssh public key (i.e./root/.ssh/u-pub.pub)>"
    add_root_ssh_key: true/false 
    users_to_add_key: [<non-root user account (i.e. myLinuxUser, etc)>] #[myuser, otherUser]
    kubernetes_version: "v1.31"    # Desired version in format: "v1.31"
    pause_image_version: "3.10"    # Pause Image Version should coorelate to with Kubernetes Version.  Format: "3.9"
    calico_cni: true               # Installs Calico CNI dependencies if true.
    calicoctl_version: "v3.28.1"   # Used to set Calico CNI version.  Format: "v3.28.1"
    use_longhorn_pv: true          # Installs Longhorn dependencies if true.
```
The `vm_list` variable determines the desired quantity of VMs, their hostnames, static IP address, desired RAM (GBs), desired disk storage (GBs), and desired CPU(s).  Add/delete VM(s) to the list by inserting/removing an entry as seen in the example below.
```bash
  vm_list:
      - { name: kubernetes-cp-1, item_ip: 10.11.0.35, ram: 8192, disk: 140, cpu: 2}
      - { name: kubernetes-cp-2, item_ip: 10.11.0.36, ram: 4096, disk: 140, cpu: 2 }
      - { name: kubernetes-cp-3, item_ip: 10.11.0.37, ram: 4096, disk: 140, cpu: 2 }
      - { name: kubernetes-wk-1, item_ip: 10.11.0.38, ram: 16384, disk: 140, cpu: 2 }
      - { name: kubernetes-wk-2, item_ip: 10.11.0.39, ram: 16384, disk: 140, cpu: 2 }
      - { name: kubernetes-wk-3, item_ip: 10.11.0.40, ram: 16384, disk: 140, cpu: 2 }
      - { name: <add-VM-hostname>, item_ip: <desired static IP>, ram: <desired VM RAM>, disk: <desired VM disk space>, cpu: <desired VM CPUs>}
```
### vSphere Variables (`main.yml`) 

vSphere variables are located in `main.yml`.  Playbooks will look for the vars file as defined in playbook `vars_files`.  In the example below, the main.yml should be places in `root/playbooks/vars` folder:
```bash
vars_files: 
  - /root/playbooks/vars/main.yml
```
vSphere Variables in `main.yml` include:
```bash
vcenter_ip: "<IP Address or FQDN>"
vcenter_username: "<administrator or other @vsphere.local>"
vcenter_password: "<username's password>"
datacenter_name: "<Datacenter-name>"
cluster_name: "<vSphere-Cluster-Name>"
datastore_name: "<Datastore-Name>" 
folder_name: "<desired-vSphere-folder-for-provisioned-VMs>"
vcenter_hostname: "<IP or FQDN of vCenter>"
template_name_rhel-94-1: "<name-of-vSphere-RHEL-9.4-template>"
template_name_rhel-94-2: "<"">"
template_name_rhel-94-3: "<''>"
template_name_ubuntu-2404-1: "<name-of-vSphere-Ubuntu-template>"
template_name_ubuntu-2404-2: "<''>"
template_name_ubuntu-2404-3: "<''>"
```


## Kubernetes Version, Pause Image Version, Calico CNI Version
Playbooks allow you to declare versions of kubernetes, pause images and Calicoctl (used with Calico CNI) directly inside the playbook (e.g. `vars: kubernetes_version: "v1.31"` to install version 1.31 kubeadm, kubectl and kubelet).  The playbook default versions are compatible with each other.  Please check vendor specific version requirements if you plan to change these values.

Note: Playbooks will install the latest stable version of ContainerD for Kubernetes. 

## Calico CNI and Longhorn Dependencies
If you plan to use Calico or Longhorn, the playbooks install required dependencies when `vars: use_calico_cni: true` and `vars: use_longhorn_pv: true` 

## Contributing
Contributions are welcome! Please fork the repository and submit a pull request with your changes.

## License
You are free to use this code however you please.  Credit to this repository is always nice.

## Contact
For any questions or issues, please open an issue in the repository.
