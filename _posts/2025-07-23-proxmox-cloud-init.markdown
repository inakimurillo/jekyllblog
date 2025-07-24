---
layout: post
title:  "Proxmox cloud init"
date:   2025-07-22 23:29:59 +0200
categories: Guides
tags: [howto, proxmox]
---
The main idea is to have a guide to generate a template for proxmox using a cloud init image from ubuntu.
Also how to create virtual machines using them.

# Creating the template 
There are plenty of cloud images available, but I want to use the latest x86-64 from [ubuntu cloud images][ubuntu-images]

Once selected, the image must be downloaded in promox commandline. In this example, I will be using Ubuntu 25.04 (Plucky Puffin).

> [!WARNING]
> Here is important to pay attention to the architecture. Here is x86-64 (amd64)

{% highlight bash %}
wget -q https://cloud-images.ubuntu.com/releases/plucky/release/ubuntu-25.04-server-cloudimg-amd64.img
{% endhighlight %}




[ubuntu-images]: https://cloud-images.ubuntu.com/releases/

