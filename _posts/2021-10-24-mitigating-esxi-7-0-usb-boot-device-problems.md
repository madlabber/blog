---
layout: post
permalink: https://madlabber.wordpress.com/2021/10/24/mitigating-esxi-7-0-usb-boot-device-problems/
title: Mitigating ESXi 7.0 USB boot device problems
description: None
date: 2021-10-24 23:37:38 -0000
last_modified_at: 2021-10-24 23:37:38 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2021/10/screen-shot-2021-10-24-at-3.22.48-pm.png
categories:
- Uncategorized
tags:
- Intel Nuc
- virtualization
- vmware
---
If you've upgraded to ESXi 7 and you use USB or SD card boot devices, you've likely [experienced some host failures](https://kb.vmware.com/s/article/83963). In my case, all of my lab hosts are affected. I had to put the brakes on upgrading past 6.7 after my early host upgrades started burning through boot drives. Like most of the community I was expecting a fix in U3. Instead, VMware chose to [deprecate support for USB/SD card boot devices](https://kb.vmware.com/s/article/85685) with plans to remove it completely in a future release. This means a full lab rebuild is inevitable, and I'll get back to that in a future post, but there are some things we can do in the meantime that may stabilize 7.0 and get a little more life out of those boot devices.

First, [upgrade to 7.0u2c](https://docs.vmware.com/en/VMware-vSphere/7.0/rn/vsphere-esxi-70u2c-release-notes.html) or later. Although [7.0u3](https://docs.vmware.com/en/VMware-vSphere/7.0/rn/vsphere-esxi-703-release-notes.html) is out, I'm staying on [7.0u2d](https://docs.vmware.com/en/VMware-vSphere/7.0/rn/vsphere-esxi-70u2d-release-notes.html) for the time being. 7.0u3 has been [taking down entire clusters](https://kb.vmware.com/s/article/86100) when a thin provisioned VM is powered on, so I'm waiting for some of that dust to settle.

Next, apply the mitigations from VMware KB [article 85685](https://kb.vmware.com/s/article/85685), nonchalantly titled: "Removal of SD card/USB as a standalone boot device option". These changes reduce the IO to the boot device, and hopefully, help them last a bit longer between reinstalls.

The first mitigation is an advanced setting introduced in 7.0u2c that moves the VMware tools to a RAM disk. Apparently [excessive reads to the VMware tools are contributing to the premature boot device failures](https://kb.vmware.com/s/article/2149257). The RAM disk workaround should be easy to implement, by setting the host's Advanced setting "ToolsRamdisk" to 1. But even though support was added for moving the tools to a Ramdisk, they [forgot to add the actual option to the list of advanced settings](https://kb.vmware.com/s/article/83782).

![](https://madlabber.wordpress.com/wp-content/uploads/2021/10/screen-shot-2021-10-24-at-3.22.48-pm.png?w=1024)

Let's start by adding that missing option. SSH into the ESXi host and run the following command:

`esxcfg-advcfg -A ToolsRamdisk --add-desc "Use VMware Tools repository from /tools ramdisk" --add-default "0" --add-type 'int' --add-min "0" --add-max "1"`

Then set it to enabled with this command:

`esxcli system settings advanced set -o /UserVars/ToolsRamdisk -i 1`

Neither of those commands will take effect until the next reboot, so shut down all those VMs and...

`reboot`

After a reboot, the new setting is in the list where it belongs:

![](https://madlabber.wordpress.com/wp-content/uploads/2021/10/screen-shot-2021-10-24-at-4.14.34-pm.png?w=1024)

The second mitigation is to move the scratch area. This part is thankfully automated. If there are any local disks it will pick one to host the scratch area, if there are no local disks it will use a Ramdisk. If you have shared storage that is not VSAN, you can manually configure scratch to live on shared storage by [following this kb](https://kb.vmware.com/s/article/1033696).

The last thing on the list from article 85685 is to make sure your USB stick supports 100MB/s and 128TBW of write endurance. mmmmkay. good luck with that.

How effective these mitigations are remains to be seen. I have seen anecdotal reports of people continuing to lose boot devices, and I've personally had one fail again already. Maybe the damage had been done before u2c came out, or maybe the USB sticks available to me just can't take this kind of abuse. Either way, the only way forward appears to be a rolling nuke&pave to get my whole lab moved off of USB boot devices. It won't be fun, or fast, or easy, but it looks like it must be done.
