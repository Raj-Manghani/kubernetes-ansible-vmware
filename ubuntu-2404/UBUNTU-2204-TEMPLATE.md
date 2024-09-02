# Ubuntu 24.04 Ansible VMware Kubernetes VM Provisioner

## Introduction

Ubuntu 24.04 virtual machines need specific configurations before use as Kubernetes-ready nodes.  This Ansible playbook installs all configurations and dependencies required for any Ubuntu 24.04 VM to be Kubernetes-ready.  However, Ansible has its own VM configurations and requirements to access newly created VMs.  The Ansible requirements need to be hard coded into the template.  The guide below provides step-by-step instructions for creating a VMware vSphere template using vCenter.  This template, paired with the playbook, will create a Kubernetes-ready node/VM on Ubuntu 24.04.

IMPORTANT NOTICE ---> The VMs created by this template & playbook lack some security measures.  For production use, the following items need to be implemented and/or considered after the VMs are created:

  -> *Enable and configure firewall (e.g. enable UFW, control-plane ports, worker node ports, etc)*

  -> *Reduce attack surface by removing unused user accounts (e.g. Cloud User, Ubuntu, etc)*
  
  -> *Change VM's default password*
  
  -> *Set appropriate VM access priviledges for users*
  
  -> *Configure SSH Server to disallow password authentication*

  -> *Consider disallowing root user SSH*

  -> *Consider replacing VM SSH key pairs*

  -> *SElinux is set to "permissive". As of Kubernetes v1.31, this is a required.*

# Step-by-Step Guide to Creating vCenter VM Template

## Download and Create Ubuntu 24.04 VM in vCenter

### Step 1: Download the Latest Ubuntu 24.04 OVA Template
    https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.ova

### Step 2: Add OVA template to vCenter Content Library
```bash
    vCenter menu ---> "Content Libraries" ---> Select/Create Library ---> "Actions" ---> "Import item" ---> "local file" ---> "upload files"
```
### Step 3: Deploy VM Template in vCenter
```bash
vCenter ---> "Inventory" ---> "New Virtual Machine" ---> "Deploy from template"
  -> "Customize template" ---> "Default Users Password" (Ubuntu) = #Desired Password
```
### Step 4: Configure VMs
```bash
Right-Click ---> "Edit Settings"
  -> "Virtual Hardware"
        --> Add desired VM specs (e.g. cpu, memory, disk, thin provisioning, etc.)

  -> Advanced Parameters 
    --> Add Parameters (Add parameters below for persistent volumes and/or copy-paste VMRC functionality)
         >"disk.EnabledUUID = TRUE" ---> #required for Kubernetes CSIs/Persistent Volumes
         >"isolation.tools.copy.disable = FALSE" ---> #required for copy/paste functionality for VMRC
         >"isolation.tools.paste.disable = FALSE" ---> #""
         >"isolation.tools.setGUIOptions.enable = TRUE" ---> #""
```
##  Configure Ubuntu 24.04 OS
Configure OS to allow Ansible to connect and configure VMs

### Step 1: Login via vCenter VMRC
```bash
Right-click VM ---> "Power" ---> "Power On"

Select VM ---> "Summary" ---> "Launch Remote Console"
  -> Username: Ubuntu
  -> Password: #Password Previously Entered for Default User
  -> Prompted to change password ---> Change password to desired password.  #To change back to original ---> sudo passwd ubuntu
```
### Step 2: Create `root` Password (used later for `Kubeadm init`)
```bash
sudo passwd root #Enter desired password for root user
```
### Step 3: Configure Cloud-Init
```bash
sudo vi /etc/cloud/cloud.cfg
  -> disable_root: false

  -> default_user: lock_passwd: False #enables default user (ubuntu) to ssh via password

Disable password change at first log in
  -> echo -e "#cloud-config\nchpasswd:\n  expire: False" | sudo tee /etc/cloud/cloud.cfg.d/99-custom-config.cfg
```
### Step 4: Configure SSH Server
```bash
sudo vi /etc/ssh/sshd_config
  -> Uncomment & change `#PermitRootLogin prohibit-password` ---> `PermitRootLogin yes`

  -> Uncomment `#PubkeyAuthentication yes` ---> `PubkeyAuthentication yes`

  -> Uncomment `#PasswordAuthentication yes` ---> `PasswordAuthentication yes`

cd /etc/ssh/sshd_config.d ---> ls ---> sudo vi <*>-cloudimg-settings.conf
  -> Change `PasswordAuthentication no` ---> `PasswordAuthentication yes`
```
### Step 5: Configure NOPASSWD Sudo User
```bash
sudo visudo 
  -> add following line `%sudo  ALL=(ALL) NOPASSWD:ALL`

  -> save & exit
```
### Step 6: Reboot VM for Good Measure
```bash
sudo reboot
```
##  Prepare VM to Become VMware Template
Clean the VM and reset the OS with `cloud-init`.  These steps will ensure unique machines are created from template.

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
rm /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id
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
## Convert Ubuntu 24.04 VM to vCenter Template

### Step 1: Remove Virtual Hardware
```bash
Select VM ---> "Edit Settings" ---> "CD/DVD drive 1" ---> "Remove device" #if applicable
```
### Step 2: Convert VM to Template
```bash
right-click vm ---> "Template" ---> "Convert to Template" ---> "YES"
```

## Conclusion
After following the steps above, you should have a working vSphere template to use with the paired playbook.  You can use this template by placing its `vcenter name` in `vars/main.yml` file.  Check `PLAYBOOK_GUIDE.md' for more on how to configure and use the playbook.