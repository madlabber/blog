---
layout: post
permalink: https://madlabber.wordpress.com/2019/07/13/running-esxi-6-7-on-a-bean-canyon-intel-nuc-nuc8i5beh/
title: Running ESXi 6.7 on a Bean Canyon Intel NUC NUC8i5BEH.
description: None
date: 2019-07-14 01:16:32 -0000
last_modified_at: 2019-07-14 17:44:49 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2019/07/nuc8i5beh.jpg
categories:
- Uncategorized
tags:
- ESXi
- Homelab
- Intel Nuc
- vmware
---
With 4 cores, 8 logical CPUs and up to 64 GB of ram the 8th generation i5 and i7 Intel NUCs make nice little home lab virtualization hosts. This week I rebuilt one of mine and documented the build process.

#### Hardware:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/nuc-parts.jpg)

In terms of components, there is not much to it. The NUC includes everything except ram and storage on the motherboard. The components I chose for this build are listed below.

* NUC8i5BEH
* 64GB (2x32GB) SoDIMM (M471A4G43MB1)
* A 32GB USB stick for the ESXi boot disk
* Local Storage: (optional)
  * Samsung 970 PRO NVME M.2 drive, 512GB
  * Samsung 960 EVO SSD drive, 1TB

Everything is easily accessible for installation. Loosen 4 screws to remove the bottom cover and everything can be assembled in minutes.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/nuc-assembly.jpg)

#### Bios settings:

Next boot into the BIOS and update it if needed. The BIOS hotkey is F2. If the NUC doesn't detect a monitor at boot the video out may not work, so plug in and turn on the monitor before powering up the NUC. I have already updated the BIOS on this one, but it is easy to do. Just put the bios file on a USB stick and install it from the Visual BIOS.

There are a few BIOS settings that should be adjusted to make things go smoothly. First, to reliably boot ESXi from the USB stick, both UEFI boot and Legacy Boot should be enabled.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/boot-priority.jpg)

Next, on the boot configuration tab, enable "Boot USB devices first":

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/boot-options.jpg)

Next head over to the Security tab and uncheck "Intel Platform Trust Technology". The NUC doesn't have a TPM chip, so if you don't disable this you'll get a persistent warning in vCenter: " TPM 2.0 device detected but a connection cannot be established. "

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/tpm-disable.jpg)

On the Power tab you'll find the setting that controls what happens after a power failure. By default it will stay powered off. For lab hosts I set it to 'last state', for appliance hosts like my pfSense firewall, I set it to always power on.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/afterpowerfailure.jpg)

#### ESXi Installation:

ESXi 6.7U1 works out of the box, with no special vibs or image customization required. There is really nothing unique to see here so I'll skip on to configuration.

#### ESXi Configuration:

Once ESX is up and running you can see the 64GB ram kit is working despite the 32GB limit in Intel's documentation.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/esxi-hardware.jpg)

Because I have internal storage, I create a datastore 'datastore1' on the NVME drive. I'm saving the SATA SSD for a later project so I am leaving it alone for now.

Next there are a few settings in ESXi that are worth pointing out. First, set the swap location to a datastore. This avoids some situations where a patch may fail to install due to lack of swap.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/esxi-swap.jpg)

Similarly the logs should be moved to a persistent location, here I'll put them on datastore1. These settings are found in the "Advanced settings" on the system tab shown above. Note that I had to pre-create the directory structure on datastore1 before applying this setting.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/logdir.jpg)

The next few settings are less conventional, and not recommended for production, but make life easier in the lab. I'll explain my reasoning for each and you can decide for yourself how you like your systems configured.

First up is salting. Salting is used to deliberately break transparent page sharing (TPS) in an effort to improve the default security posture of the host. But this isn't production, it's a home lab. I fully expect to over-commit memory on this little host, so if I can gain any efficiency by re-enabling transparent page sharing and letting it de-dupe the ram across VMs, I'll take it.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/salting.jpg)

Next is the BlueScreenTmeout. By default if an ESXi host panics (a PSOD), it will sit on the panic screen forever so you can diagnose the error and so the host doesn't go back into service until you've had a chance to address the problem. But for these little NUCs I run them headless, and they don't have IPMI or even vPro. I would have to plug in a monitor and reboot it anyway to get at the console, so I would rather it just reboot so I can access it over the network. On this setting 0=never reboot, >0=seconds to wait before reboot. I'm going with 30 seconds:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/bsod.jpg)

And finally I will enable SSH and disable the resulting shell warning. I frequently connect to my lab hosts over SSH, so I prefer to leave SSH enabled. Again, this isn't something you would do in production. It is purely a lab convenience.

For both of the TSM services, I set the policy to "Start and Stop with Host", then start the service.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/tsm-policy.jpg)

The UI will continue to warn that these services are running. This setting disables that warning:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/suppressshellwarning.jpg)

#### Networking Options:

The built-in 1gb network adapter may not be enough for every lab scenario, but network connectivity is not limited to the single onboard gigabit NIC. The Apple Thunderbolt adapter and Apple Thunderbolt NIC are fully functional:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/tb-nic.jpg)

Or you can install the [USB Network driver fling](https://labs.vmware.com/flings/usb-network-native-driver-for-esxi), and add some USB 3 gigabit nics, with some caveats around jumbo frame support and a few other things you can read about while you download the driver.

Here's a screenshot from a NUC I was testing with 4x1gb connectivity provided by 2 USB3 adapters, the Apple dongles, and the onboard NIC.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/nucw4nics1.jpg)

10GB is also option, with a working Thunderbolt3 adapter and driver. William Lam over [virtuallyghetto.com](https://www.virtuallyghetto.com/) has been testing some of the [10GB options](https://www.virtuallyghetto.com/2019/04/new-thunderbolt-3-to-10gbe-options-for-esxi.html).

#### Conclusion:

The current generation of NUCs are surprisingly capable and configurable little ESXi lab hosts. This is how I build mine, but if you've got other ideas share them in the comments.
