---
layout: post
title:  "Proxmox cloud init"
date:   2025-07-22 23:29:59 +0200
categories: Guides
tags: [howto, proxmox]
---
The main idea is to have a guide to generate a template for Proxmox using a cloud init image from ubuntu.
Also how to create virtual machines using them.

But first of all, what's cloud-init? From [RedHat][redhat-cloud-init]: _"The cloud-init utility automates the initialization of cloud instances during system boot."_ Or in other words, it's a configuration that is loaded when the system starts. That configuration can be users, hostname, scripts...

This sounds great for a Proxmox server, as all the tedious part of configuring users, public keys etc will be automated. And the best part, is that cloud-init can be mixed with templates, so setting a VM up and running with the expected configuration is a matter of just a couple of clicks!


```mermaid!
architecture-beta
    group internet(cloud)[Ubuntu Servers]
    group proxmox(server)[Proxmox]
    group template(server)[Template] in proxmox

    service disk_ubuntu(disk)[Cloud Init image] in internet
    
    service cloudinit(database)[Cloud Init database] in template

    service templateDisk(server)[Local Cloud Init Template VM] in template
    service vm(server)[Virtual Machine] in proxmox

    disk_ubuntu:R --> L:templateDisk
    templateDisk:T <-- B:cloudinit

    templateDisk:R --> T:vm
    cloudinit:R --> T:vm

```

# Prepare the template disk 
There are plenty of cloud images available, but I want to use the latest x86-64 from [ubuntu cloud images][ubuntu-images]

Once selected, the image must be downloaded in Proxmox commandline. In this example, I will be using Ubuntu 25.04 (Plucky Puffin).

> [!WARNING]
> Here is important to pay attention to the architecture. Here is x86-64 (amd64)

{% highlight bash %}
$ wget -q https://cloud-images.ubuntu.com/releases/plucky/release/ubuntu-25.04-server-cloudimg-amd64.img
{% endhighlight %}

For bonus points, lets analyze the image we just downloaded. As it can be inferred from the name, its img file, but to know what kind of image, lets run the `file` command. 

{% highlight bash %}
$ file ubuntu-25.04-server-cloudimg-amd64.img 
ubuntu-25.04-server-cloudimg-amd64.img: QEMU QCOW Image (v3), 3758096384 bytes (v3), 3758096384 bytes{% endhighlight %}

So it's a QEMU QCOW Image version 3. Then to know the virtual size of the image, `qemu-img info` command can be run.

{% highlight bash %}
$ qemu-img info ubuntu-25.04-server-cloudimg-amd64.img 
image: ubuntu-25.04-server-cloudimg-amd64.img
file format: qcow2
virtual size: 3.5 GiB (3758096384 bytes)
disk size: 656 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
Child node '/file':
    filename: ubuntu-25.04-server-cloudimg-amd64.img
    protocol type: file
    file length: 656 MiB (688117760 bytes)
    disk size: 656 MiB
{% endhighlight %}

So the virtual size of the image is 3.5GB, this means that the disk is very small once it's up and running. So in order to increase the size, we can use the `qemu-img resize IMAGE SIZE` command. In this example, the image will be increased to 128GB.

{% highlight bash %}
$ qemu-img resize ubuntu-25.04-server-cloudimg-amd64.img 128G
Image resized.
{% endhighlight %}

Now if we check again the virtual size, it must be 128GB:

{% highlight bash %}
$ qemu-img info ubuntu-25.04-server-cloudimg-amd64.img 
image: ubuntu-25.04-server-cloudimg-amd64.img
file format: qcow2
virtual size: 128 GiB (137438953472 bytes)
disk size: 656 MiB
...
{% endhighlight %}

# Create the Template Virtual Machine
Let's create the template. This template will use the recently configured disk and also a cloud-init disk. This VM will act as the base for every one created.

The next command will create a VM named `ubuntu-2504-cloudinit-template`, with the VM ID 900 (change as needed) based on a `Linux Linux 2.6 - 6.X Kernel` with 4GB of ram, with the QEMU Guest Agent activated (will be installed later on), the disk will be located in `dir-nvme` so change it as you need. For more info about the parameters see [Proxmox manual][promox-manual].

{% highlight bash %}
$ qm create 900 --name "ubuntu-2504-cloudinit-template" --ostype l26 \
    --memory 4096 --agent 1 --bios seabios --machine q35 --cpu host \
    --socket 1 --cores 2 --vga serial0 --serial0 socket  \
    --net0 virtio,bridge=vmbr4

{% endhighlight %}

Now let's import the ubuntu disk into this Virtual Machine:

{% highlight bash %}
$ qm importdisk 900 ubuntu-25.04-server-cloudimg-amd64.img dir-nvme
Formatting '/mnt/pve/dir-nvme/images/900/vm-900-disk-1.raw', fmt=raw size=137438953472 preallocation=off
transferred 0.0 B of 128.0 GiB (0.00%)
transferred 1.3 GiB of 128.0 GiB (1.00%)
transferred 2.6 GiB of 128.0 GiB (2.02%)
...
transferred 128.0 GiB of 128.0 GiB (100.00%)
transferred 128.0 GiB of 128.0 GiB (100.00%)
unused0: successfully imported disk 'dir-nvme:900/vm-900-disk-0.raw'
{% endhighlight %}

Now its time to set this disk as the primary hard disk and boot from it.

{% highlight bash %}
$ qm set 900 --scsihw virtio-scsi-pci --virtio0 dir-nvme:900/vm-900-disk-0.raw,discard=on
update VM 900: -scsihw virtio-scsi-pci -virtio0 dir-nvme:900/vm-900-disk-0.raw,discard=on
{% endhighlight %}

{% highlight bash %}
$ qm set 900 --boot order=virtio0
update VM 900: -boot order=virtio0
{% endhighlight %}

Finally lets add a cloud init disk

{% highlight bash %}
$ qm set 900 --ide2 dir-nvme:cloudinit
update VM 900: -ide2 dir-nvme:cloudinit
Formatting '/mnt/pve/dir-nvme/images/900/vm-900-cloudinit.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off preallocation=metadata compression_type=zlib size=4194304 lazy_refcounts=off refcount_bits=16
ide2: successfully created disk 'dir-nvme:900/vm-900-cloudinit.qcow2,media=cdrom'
generating cloud-init ISO
{% endhighlight %}

# Prepare the cloud init disk
Now its time to configure the cloud init disk. But before, make sure you have snippets active in the storage. See image below:

Open datacenter → storage → select the need storage → add snippets to contents list

![image tooltip here]({{site.baseurl}}/assets/2025-07-23-proxmox-cloud-init/image10.jpg)

Let's add a cloud init script in yaml format (`ubuntuCloudInit.yaml`). This script will install `qemu-guest-agent` and will reboot the system. This script only be executed once.

> [!NOTE]
> /mnt/pve/dir-nvme/snippets can be changed to /var/lib/vz/snippets/ to use local Storage

{% highlight bash %}
$ cat << EOF | tee /mnt/pve/dir-nvme/snippets/ubuntuCloudInit.yaml
#cloud-config
runcmd:
    - apt update
    - apt install -y qemu-guest-agent
    - systemctl start qemu-guest-agent
    - reboot
EOF
{% endhighlight %}

Let's recap, we have a virtual machine with a cloud-init enabled image, and cloud-image disk added. But we have to configure the cloud-image disk. 

This command will add the previously created yaml script, will add `template` and `cloudinit` tags, add a user with username `user` with password `greatPassword` (please change this), will do an automatic package upgrade after the first boot, will add ssh authorized key form the host and finaly config the network with dhcp. For more information see [proxmox-manual-qm]. 

{% highlight bash %}
$ qm set 900 --cicustom "vendor=dir-nvme:snippets/ubuntuCloudInit.yaml" 
$ qm set 900   --tags template,cloudinit 
$ qm set 900   --ciuser user --ciupgrade 1
$ qm set 900  --cipassword $(openssl passwd -6 "greatPassword")
$ qm set 900   --sshkeys ~/.ssh/authorized_keys
$ qm set 900   --ipconfig0 ip=dhcp
{% endhighlight %}

We must not forget to update the cloud-inti configuration:
{% highlight bash %}
$ qm cloudinit update 900
generating cloud-init ISO
{% endhighlight %}

And now the moment we all have been waiting for, convert the VM to a template:

{% highlight bash %}
$ qm template 900
{% endhighlight %}

# Create the Virtual Machine

To use this template, we must clone the template machine. There are two types of clones, `Linked`and `Full` clones. The first ones needs the template virtual disk to be in the same host, on the other hand, the `Full` ones are a copy of the original template.

In this case, a linked one is going to be created, as occupy less size and fits my use case.

> [!NOTE]
> In case a -full clone is used, the storage can be specified. For example: -full -storage local-lvm
> Linked clones are created by default

{% highlight bash %}
$ qm clone 900 201 --name ubuntu-test
create full clone of drive ide2 (dir-nvme:900/vm-900-cloudinit.qcow2)
Formatting '/mnt/pve/dir-nvme/images/201/vm-201-cloudinit.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off preallocation=metadata compression_type=zlib size=4194304 lazy_refcounts=off refcount_bits=16
create linked clone of drive virtio0 (dir-nvme:900/base-900-disk-0.raw)
clone 900/base-900-disk-0.raw: images, vm-201-disk-0.qcow2, 201 to vm-201-disk-0.qcow2 (base=../900/base-900-disk-0.raw)
Formatting '/mnt/pve/dir-nvme/images/201/vm-201-disk-0.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=137438953472 backing_file=../900/base-900-disk-0.raw backing_fmt=raw lazy_refcounts=off refcount_bits=16

{% endhighlight %}

And now lets start the VM: (To see the start up sequence is recommended to use the GUI)

{% highlight bash %}
$ qm start 201 
{% endhighlight %}


# TLDR;

[ubuntu-images]: https://cloud-images.ubuntu.com/releases/

[redhat-cloud-init]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_cloud-init_for_rhel_9/introduction-to-cloud-init_cloud-content

[promox-manual]: https://pve.proxmox.com/wiki/Manual:_qm.conf
[proxmox-manual-qm]: https://pve.proxmox.com/pve-docs/qm.1.html

