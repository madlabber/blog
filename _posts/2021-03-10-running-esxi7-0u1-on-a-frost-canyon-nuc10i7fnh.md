---
layout: post
permalink: https://madlabber.wordpress.com/2021/03/10/running-esxi7-0u1-on-a-frost-canyon-nuc10i7fnh/
title: Running ESXi7.0U1 on a Frost Canyon NUC10I7FNH
description: None
date: 2021-03-11 07:47:05 -0000
last_modified_at: 2021-03-11 07:47:05 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2021/03/nuc10i7front.jpg
categories:
- Uncategorized
tags:
- Intel Nuc
- vmware
---
Since my last series of NUC host build posts several new NUCs have come on the market, and we already have visibility into the models in the pipeline, but of the 4x4x2 models current and foreseeable, the NUC10i7FNH is the only 6 core part in that form factor. With this model we get:

6 core / 12 thread i7 CPU  
64GB RAM is actually supported  
1x onboard gigabit NIC, supported in the 7.0U1 build of ESXi  
1x TB3 port, that can be leveraged for additional networking  
1x M.2 slot for NVME storage  
1x SATA 2.5" bay for additional storage

For this build I'll be running ESX on a USB stick. I'll be using an NFS datastore for now. I'll circle back to this later with some internal SSDs for different lab scenarios, but for now it has a very short BOM:

1 x NUC10i7FNH  
2 x 32GB SoDIMM (64gb total)  
1 x 32gb USB stick

Physical assembly is as trivial as ever, and this build of ESXi works without customizations, but there are a couple of things in the BIOS worth looking at.

First, we need to disable the TPM. The NUC doesn't actually TPM chip, but if we don't disable the imaginary TPM chip ESX will keep warning us that the phantom chip isn't responding. This setting is buried in a submenu of Security. Navigate to the Security menu, then click "Security Features", and un-check "Intel Platform Trust Technology". This option was broken in older BIOS versions, but was fixed in 0.44. My unit arrived with the patched version installed, but if yours did not, you'll need to update it to make the warning go away.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/securitymenu.jpg?w=836)Click on 'Security Features' ![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/tpmsetting.jpg?w=1024)Uncheck 'Intel Platform Trust Technology'

The other setting to check is "After Power Failure", which is again buried in a submenu. Navigate to the Boot menu, then click Secondary Power Settings, and scroll down to "After Power Failure". The default value is least likely to be the one you want. I prefer "Power On", but set it to your preferred server behavior.

Now we can proceed with installing ESXi. As long as you use the 7.0U1 iso, everything should just work.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/platform-screen.jpg?w=1024)

After logging in we can see 12 logical CPUs and 64GB ram. I can work with that.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/hardware-info.jpg?w=1024)

**Setting My Preferences:**  
Next I apply my personal lab host preferences in the advanced settings:

![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/settings0.jpg?w=926)

Mem.Share.ForceSalting = 0, to re-enable transparent page sharing for all VMs.  
Misc.BlueScreenTimeout = 30, to allow the host to reboot if it encounters a PSOD  
UserVars.SupressShellWarning = 1, because I keep ssh enabled and don't want it bugging me.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/settings1.jpg?w=1024)

* ![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/settings2.jpg?w=1024)

And then I enable the TSM and TSM-SSH services. Select the server from the list, and use the Actions button to set the service to start and stop with host, and start the service.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/services.jpg?w=1024)

**Networking options:**  
The onboard NIC works with this build of ESX using the inbox driver, but so does the Apple Thunderbolt NIC, when plugged into the thunderbolt 3 port via the apple thunderbolt 2 to thunderbolt 3 adapter. The Apple NIC uses the ntg3 inbox driver. There are several thunderbolt and USB3 networking options available using either vendor supplied drivers for some 10gbe adapters, or the native USB3 network driver fling from VMware.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/03/nuc10i7front.jpg?w=1024)

**Conclusion:**  
With the upcoming 4x4x2 form factor NUCs all set to deliver only 4 cores, this little 6 core box appears to be the one to have. It has been around long enough now to have good driver support, and ESXi7.0u1 works out of the box. It doesn't get any simpler for a small form factor lab enthusiast.
