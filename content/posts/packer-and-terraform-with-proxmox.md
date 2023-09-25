---
title: "Packer and Terraform With Proxmox"
date: 2023-09-22T11:40:09+02:00
summary: |
  Building system images with packer & deploying them with Terraform
tags:
  - proxmox
  - terraform
  - packer
---

I've recently started to use [proxmox](https://www.proxmox.com/en/) on my previous desktop system for testing purpose (my homelab is not using VMs), and to be able to install some stuff on it, I used `packer` to build system imates/templates & `terraform` to deploy them.

Source code is here: [proxmox-infra](https://github.com/mycroft/proxmox-infra).

## Packer

Building the `Packer` image with [proxmox-iso](https://developer.hashicorp.com/packer/integrations/hashicorp/proxmox/latest/components/builder/iso) is pretty much straightforward:


```hcl
source "proxmox-iso" "ubuntu-basic" {
    vm_name = "${var.vm_name}"

    [... snap ...]

    # That's the proxmox node name to generate VM on. 
	node = "night-flight"

    iso_file = "local:iso/ubuntu-22.04.3-live-server-amd64.iso"
    unmount_iso = true
    iso_storage_pool = "local"

    [... snap ...]

    # Install the qemu agent
    qemu_agent = true

    # do not add an empty cloud_init iso after generation, as it will be enabled in "children" VMs.
    cloud_init = false

    # PACKER Boot Commands for ubuntu
    boot_command = [
    	"c",
    	"linux /casper/vmlinuz --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/' ",
    	"<enter><wait>",
    	"initrd /casper/initrd<enter><wait>",
    	"boot<enter>"
    ]
    boot = "c"
    boot_wait = "5s"

    http_directory = "http" 

    ssh_username = "mycroft"
    ssh_agent_auth = true
    # public key is created thanks to cloud-init
    ssh_keypair_name = "id_ed25519"

    ssh_timeout = "20m"
}
```

[source](https://github.com/mycroft/proxmox-infra/blob/main/packer/ubuntu-basic/ubuntu-basic.pkr.hcl)


## Terraform

Once the image is created, VMs can be deployed with `terraform` using the [proxmox provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs).

My current configuration for 4 Kubernetes nodes is:

```hcl
resource "proxmox_vm_qemu" "kubernetes_server" {
  count       = 4
  name        = "kube-vm-${count.index + 1}"
  desc        = "Kubernetes node"
  target_node = var.proxmox_host
  clone       = var.template_name

  agent       = 1
  os_type     = "cloud-init"
  cores       = 2
  sockets     = 1
  cpu         = "host"
  memory      = 8192
  scsihw      = "virtio-scsi-pci"
  bootdisk    = "scsi0"
  qemu_os     = "l26"

  cloudinit_cdrom_storage = "local-lvm"

  disk {
    size     = "32G"
    type     = "scsi"
    storage  = "local-lvm"
  }

  network {
    model  = "virtio"
    bridge = "vmbr0"
  }

  ipconfig0 = "ip=10.2.1.2${count.index + 1}/8,gw=10.0.0.1"

  lifecycle {
    ignore_changes = [
      network,
    ]
  }

  sshkeys = <<EOF
  ${var.ssh_key}
  EOF
}
```

[source](https://github.com/mycroft/proxmox-infra/blob/main/terraform/kube-servers.tf)

