# RHEL 9.4 Ansible VMware Kubernetes VM Provisioner

## Introduction

RHEL 9.4 virtual machines need specific configurations before use as Kubernetes-ready nodes.  This repository's Ansible playbook installs all configurations and dependencies required for any RHEL 9.4 VM to be Kubernetes-ready.  However, Ansible has its own VM configurations and requirements needed to access newly created VMs.  These Ansible requirements need to be hard coded into the template.  The guide below provides step-by-step instructions for creating a VMware vSphere template using vCenter.  This template, along with the playbook, will create a Kubernetes-ready node/VM on RHEL 9.4.

IMPORTANT NOTICE ---> The VMs created by this template & playbook lack some security measures.  For production use, the following items need to be implemented and/or considered after the VMs are created:
  
  -> *Enable and configure firewall (e.g. enable firewalld, control-plane ports, worker node ports, etc)*

  -> *Reduce attack surface by removing unused user accounts (e.g. Cloud User, etc)*
  
  -> *Change VM's default password*
  
  -> *Set appropriate VM access priviledges for users*
  
  -> *Configure SSH Server to disallow password authentication*

  -> *Consider disallowing root user SSH*

  -> *Consider replacing VM SSH key pairs*

  -> *SElinux is set to "permissive". As of Kubernetes v1.31, this is a required.*

# Step-by-Step Guide to Creating vCenter VM Template

## Download and Create RHEL 9.4 VM in vCenter

### Step 1: Download the Latest RHEL 9.4 ISO
https://developers.redhat.com/products/rhel/overview

### Step 2: Add RHEL 9.4 ISO to vCenter Content Library
```bash
vCenter menu ---> "Content Libraries" ---> Select/Create Library ---> "Actions" ---> "Import item" ---> "local file" ---> "upload files"
```
### Step 3: Configure VM and Add RHEL 9.4 ISO
```bash
vCenter ---> "Inventory" ---> "New Virtual Machine" ---> "Create a new virtual machine" ---> "Customize hardware" 
  -> "Virtual Hardware" 
    --> Add desired VM specs (e.g. cpu, memory, disk, thin provisioning, etc.)
    --> "New CD/DVD Drive" ---> "Content Library ISO File" ---> Select "rhel-9.4-xxx_xxx-boot"
    --> Check box ---> "Connect At Power On"
  
  -> "Advanced Parameters" 
    --> "Add Parameters" (Add parameters below for persistent volumes and/or copy-paste VMRC functionality)
         >"disk.EnabledUUID = TRUE" ---> #required for Kubernetes CSIs/Persistent Volumes
         >"isolation.tools.copy.disable = FALSE" ---> #required for copy/paste functionality for VMRC
         >"isolation.tools.paste.disable = FALSE" ---> #""
         >"isolation.tools.setGUIOptions.enable = TRUE" ---> #""
```

### Step 4: Power on VM and Install RHEL 9.4
```bash
Select VM in inventory list and power on

Launch Remote Console/VMRC
  -> Select "Install Red Hat Enterprise Linux"

  -> Along with other settings, set the following in "Installation Summary"
    --> "Software Selection"
          > "Base Environment" 
            > "Minimal Install"
          
          > "Additional software for Selected Environment"
            >"Standard"
            > "System Tools"
            > "Container Management"
            > "Development Tools"
            > "Security Tools"
            > "Headless Management"

  -> "Root Password" ---> Enable ---> "Allow root SSH login with password"

  -> "User Creation" ---> Create user Account ---> check "Make this user administrator"

  -> "Begin Installation" ---> "Complete!" ---> "Reboot System"
```
##  Configure RHEL 9.4 OS
Configure OS to allow Ansible to connect and configure VMs.

### Step 1: Login to RHEL 9.4 VM via vCenter VMRC
```bash
Select VM ---> "Summary" ---> "Lauch Remote Console" / / OR SSH with password authentication
  -> Username: root / / or other administrator account
  -> Password: #Password previously entered for root or user, during installation process
```
### Step 3: Install and Configure Cloud-Init
```bash
sudo yum install cloud-init -y

sudo vi /etc/cloud/cloud.cfg
  -> disable_root: true ---> disable_root: false

  -> ssh_pwauth: false ---> ssh_pwauth: true

  -> ssh_deletekeys: true ---> ssh_deletekeys: false

  -> disable_vmware_customization: false

  -> Default_user #add new user created during installation
    --> name: myuser
    --> lock_passwd: True ---> default_user: lock_passwd: False #enables default user to ssh via password
    --> gecos: #Fullname of user

Prevent password change at first log in
  -> echo -e "#cloud-config\nchpasswd:\n  expire: False" | sudo tee /etc/cloud/cloud.cfg.d/99-custom-config.cfg
```
### Step 4: Configure SSH Server
```bash
sudo vi /etc/ssh/sshd_config
  -> Uncomment & change `#PermitRootLogin prohibit-password` ---> `PermitRootLogin yes`

  -> Uncomment `#PubkeyAuthentication yes` ---> `PubkeyAuthentication yes`

  -> Uncomment `#PasswordAuthentication yes` ---> `PasswordAuthentication yes`
```
### Step 5: Configure NOPASSWD Sudo User
```bash
sudo visudo 
  -> Uncomment line `%wheel  ALL=(ALL) NOPASSWD:ALL`

  -> save & exit
```
### Step 6: Stop & Disable Firewalld
```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
```
### Step 7: Reboot VM for good measure
```bash
sudo reboot
```
##  Prepare VM to Become VMware Template
Clean the VM and reset the OS with `cloud-init`.  These steps will ensure unique machines are created from template.  Root login recommended.

### Step 1: Clean Logs
```bash
if [ -f /var/log/audit/audit.log ]; then
  cat /dev/null > /var/log/audit/audit.log
fi
if [ -f /var/log/wtmp ]; then
  cat /dev/null > /var/log/wtmp
fi
if [ -f /var/log/lastlog ]; then
  cat /dev/null > /var/log/lastlog
fi
```
### Step 2: Clean udev Rules
```bash
if [ -f /etc/udev/rules.d/70-persistent-net.rules ]; then
  rm /etc/udev/rules.d/70-persistent-net.rules
fi
```
### Step 3: Clean /tmp Directories
```bash
rm -rf /tmp/*
rm -rf /var/tmp/*
```
### Step 4: Clean SSH Host Keys
```bash
rm -f /etc/ssh/ssh_host_*
```
### Step 5: Clean Machine-ID
```bash
truncate -s 0 /etc/machine-id
rm /var/lib/dbus/machine-id #Directory or file may not exist
ln -s /etc/machine-id /var/lib/dbus/machine-id #Directory or file may not exist
```
### Step 6: Clean Shell History
```bash
unset HISTFILE
history -cw
echo > ~/.bash_history
rm -fr /root/.bash_history
```
### Step 7: Truncate hostname, hosts, resolv.conf and set hostname to localhost
```bash
truncate -s 0 /etc/{hostname,hosts,resolv.conf}
hostnamectl set-hostname localhost
```
### Step 8: Reset Cloud-Init
```bash
cloud-init clean -s -l
```
### Step 9: Shutdown VM via OS
```bash
shutdown -h now
```
## Convert RHEL 9.4 VM to vCenter Template

### Step 1: Remove Virtual Hardware
```bash
Select VM ---> "Edit Settings" ---> "CD/DVD drive 1" ---> "Remove device"
```
### Step 2: Convert VM to Template
```bash
right-click vm ---> "Template" ---> "Convert to Template" ---> "YES"
```

## Conclusion
After following the steps above, you should have a working vSphere template to use with the paired playbook.  You can use this template by placing its `vcenter name` in `vars/main.yml` file.  Check `PLAYBOOK_GUIDE.md' for more on how to configure and use the playbook.