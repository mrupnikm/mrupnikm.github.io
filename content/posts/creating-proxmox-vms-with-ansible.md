---
author: "Matic Rupnik"
authorLink: matic-rupnik
date: 2024-09-17    
title: Creating Proxmox VMs with Ansible
tags: [
  "Ansible",
  "Proxmox",
  "Ubuntu",
]
---

# Creating Proxmox VM templates with Ansible

## Overview

This post describes an automated way of creating Proxmox VM templates using Ansible. There are some options of how to automate creation of VMs in Proxmox but most relly on having an already present [VM template](https://pve.proxmox.com/wiki/VM_Templates_and_Clones) to be than cloned.
Creating these templates by hand is not exactly ideal if you wish to write down your Infrastructure as code, so I have created a simple playbook to make it easy to setup a cloud image (Ubuntu 24.4. in this case) template.


## How It Works

### 1. Defining variables
So let's have a look at the following YAML file containing the variables:
```yaml
#
# ubuntu cloud image template
#
proxmox_vm_template_id: 8002
proxmox_vm_template_memory: 4096
proxmox_vm_template_core: 2
proxmox_vm_template_name: ubuntu-cloud-noble
proxmox_vm_template_net0: "virtio,bridge=vmbr0"
proxmox_vm_template_image_url: "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
proxmox_vm_template_image_name: "noble-server-cloudimg-amd64.img"
proxmox_vm_template_storage: "lvm"
proxmox_vm_template_scsihw: "virtio-scsi-pci"
proxmox_vm_template_disk_path: "/vm-{{ proxmox_vm_template_id }}-disk-0.raw"
proxmox_vm_template_ipconfig0: "ip=192.168.1.20/24 gw=192.168.1.1"
proxmox_vm_template_boot_order: "c"
proxmox_vm_template_bootdisk: "scsi0"
proxmox_vm_template_serial0: "socket"
proxmox_vm_template_vga: "serial0"
proxmox_vm_template_cloudinit: "cloudinit"
proxmox_vm_template_disk_size: "32G"
```
### Main information:
- **proxmox_vm_template_image_url**: download path to the desired cloud image file 
- **proxmox_vm_template_name**: this is the name/ID Proxmox registers the template under
- **proxmox_vm_template_storage**: registered storage on your Proxmox server
- **proxmox_vm_template_ipconfig0**: cloud-init defined static IP and gateway

> NOTE: configure the hardware options to your liking

### 2. Reading the playbook

```yaml
- name: Create VM template
  hosts: proxmox
  become: yes
  become_user: root
  vars_files:
    - vars.yml
  tasks:
    - name: Download the Ubuntu Cloud Image
      get_url:
        url: "{{ proxmox_vm_template_image_url }}"
        dest: "/tmp/{{ proxmox_vm_template_image_name }}"

    - name: Create the VM
      command: >
        qm create {{ proxmox_vm_template_id }}
        --memory {{ proxmox_vm_template_memory }}
        --core {{ proxmox_vm_template_core }}
        --name {{ proxmox_vm_template_name }}
        --net0 {{ proxmox_vm_template_net0 }}

    - name: Import the disk image
      command: >
        qm disk import {{ proxmox_vm_template_id }} /tmp/{{ proxmox_vm_template_image_name }} {{ proxmox_vm_template_storage }}
        
    - name: Resize disk
      command: >
        qemu-img resize {{ mount_point }}/images/{{ proxmox_vm_template_id }}{{ proxmox_vm_template_disk_path }} {{ proxmox_vm_template_disk_size }}

    - name: Set the SCSI hardware and disk
      command: >
        qm set {{ proxmox_vm_template_id }} 
        --scsihw {{ proxmox_vm_template_scsihw }} 
        --scsi0 {{ proxmox_vm_template_storage }}:{{ proxmox_vm_template_id }}{{ proxmox_vm_template_disk_path }}

    - name: Attach the CloudInit disk
      command: >
        qm set {{ proxmox_vm_template_id }} 
        --ide2 {{ proxmox_vm_template_storage }}:{{ proxmox_vm_template_cloudinit }}    

    - name: Configure boot settings
      command: >
        qm set {{ proxmox_vm_template_id }}
        --boot {{ proxmox_vm_template_boot_order }}
        --bootdisk {{ proxmox_vm_template_bootdisk }}

    - name: Set serial and VGA options
      command: >
        qm set {{ proxmox_vm_template_id }}
        --serial0 {{ proxmox_vm_template_serial0 }}
        --vga {{ proxmox_vm_template_vga }}

    - name: Convert the VM to a template
      command: qm template {{ proxmox_vm_template_id }}
```

Since there are a lot of variables here it might not be very easy to read so lets just go over the main steps defined.

### Steps:
1. Pulls the desired image
2. Creates core VM with hardware defined
3. Imports cloud image
4. Resizes the work partition to desired size
5. Set SCSI partition
6. Attach cloud init disk
7. Condigure boot settings
8. Set serial and VGA
9. Transform VM into a template

## Result

This will create an Ubuntu VM template that is visible in the GUI along with your running VMs. From here you can create clones of this vm and pass cloud-init configuration to it so you have a clean way of managing your machines.
For provisioning these VMs from templates like created from this article checkout [Terraform Proxmox Provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)

## Resources
- [My general Proxmox IoC with the above example](https://github.com/mrupnikm/infrastructure/tree/main/proxmox/ansible)
- [Example of creating Kubernetes nodes with Terraform](https://github.com/mrupnikm/infrastructure/tree/main/proxmox/terraform) 
- 

