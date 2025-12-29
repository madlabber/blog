---
permalink: /2021/11/11/iscsi-boot-on-an-intel-nuc/
title: iSCSI Boot on an Intel NUC
description: None
date: 2021-11-11 08:10:33 -0000
last_modified_at: 2021-11-11 08:10:33 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2021/11/nucpod.jpg
categories:
- Uncategorized
tags: []
---
This isn't something I ever thought I would need to try, but VMware's deprecation of the usual homelab boot devices in 7.x left me in a bind. To illustrate my problem, here's the layout of my homelab's management cluster:

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/mgmt-cluster.jpg?w=318)

Every host is booting from USB, and ESX50 which provides the shared storage for the management cluster by way of an ONTAP Select virtual storage appliance, is using both of its internal drives to provide shared storage. I'll have to wipe it out to reinstall ESX on an internal disk. ESX51-54 are the (diskless) hosts within the management cluster, also booting from USB. Solving the ESX50 problem is a challenge for another day. For now I need to do something about my management hosts's dependency on USB boot.

Now admittedly I could have bought drives for these hosts and populated the m.2 bay, but that would have limited my flexibility for running lab scenarios with these hosts, and I had always intended to convert them to a VSAN cluster someday, or a 4node OTS cluster, or a 4 node StorageGrid. So... how could I avoid sacrificing my NVME slots to being mere boot devices, and maybe keep my options open to the possibility of spinning up a VSAN lab down the road? The least-bad idea seemed to be iSCSI boot. but would it even work?

To get these options to light up in the BIOS, I first had to factory reset. Whatever optimizations I had applied to get USB boot working in the first place were preventing the NIC from coming up in the Add-In config options.

In the Visual Bios, navigate to Devices -> Add-in Config. If you see both the iSCSI Configuration option and the onboard NIC, then you can proceed to configuring iSCSI.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/visualbios1.jpg?w=1024)Clicking on iSCSI Configuration takes you into this page, where the first thing you have to do is set the initiator IQN. Any valid IQN will work, and I tested several variants, but I chose to stick with an typical Intel IQN: iqn.1987-05.com.intel:esx54 ![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/iscsi-iqn.jpg?w=1024)

The next step is to add a boot attempt, but before I could do that I had to save and exit, and re-open the iscsi configuration page. Perhaps it was a fluke of my particular BIOS revision, but I mention it in case you also cannot add a boot attempt after setting the IQN.

To add a boot attempt, navigate down to Add Attempt and press enter.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/boot-attempt1.jpg?w=1024)

The values I set for my boot attempt are:  
`iSCSI Mode: Enabled  
Internet Protocol: IP4  
Connection Retry Count: 4  
Connection Establishment Timeout: 10000 (milliseconds)  
OUI-Format ISID: (Default)  
Configure ISID: (Default)  
Enable DHCP: X (enabled)  
Get target info via DHCP: disabled  
Target Name: (Target IQN from my NetApp)  
Target Address: 192.168.123.64 (the target IP on my NetApp)  
Target Port: 3260 (iscsi default port)  
Boot Lun: 0  
Authentication Type: None`

As you can see there is no option for a VLAN tag, so the iscsi target must be reachable by the native VLAN of the onboard NIC. I also tried routing my iSCSI boot traffic, and that also worked, but pushing iSCSI over a router is rarely a good idea.

Next I needed a boot lun. How you provision your boot LUNs will vary from SAN to SAN, but on the NetApp ONTAP based systems it is pretty simple in the GUI:

I had already enabled iSCSI on a StorageVM when I did the original install, so just had to add and map a LUN in System Manager

Here I fill out the basic LUN details:

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/add-lun.jpg?w=1024)VMware requires at least 32GB for boot LUNs in vSphere 7, but when I installed onto a local disk it actually consumed 128gb, so I'm making my boot LUNs super-sized. Don't click save just yet, we need to add the iSCSI initiator IQN to match the BIOS settings used earlier.  
Click 'More Options', scroll down to the 'Host Mapping' section, pick 'Host Initiators', then click add-initiator, and fill in the IQN. Give the new Initiator Group a name, and only select this one host's IQN. ![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/igroup.jpg?w=1024)

I needed the target IQN earlier to complete the boot attempt settings in the BIOS. That can be found by navigating to Storage VMs, click the iscsi enabled Storage VM, then on the settings tab scroll down to the iSCSI settings card:

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/iscsi-settings.jpg?w=634)

Now, if everything goes according to plan, you can boot the NUC from the ESXi installation media, and it should discover the iSCSI target LUN:

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/install-to-lun.jpg?w=1024)

Installation was otherwise uneventful, and after it rebooted, it eventually found its boot lun and here we go:

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/boot-lun-info.jpg?w=1024)The 'Normal/Degraded' status indicates there is only a single path to the LUN, which on this clean install is accurate. There is plenty of additional configuration to do before I can put this back into my management cluster. But, at least I won't be feeding it new USB sticks every couple of months.

So now with iSCSI boot, from a dedicated host providing shared storage, I have a tiny little converged infrastructure. I'm nicknaming it the 'NUCPod', at least until I upset someone in marketing. It may yet run VSAN someday, but I'll have to bootstrap it from an actual SAN.

![](https://madlabber.wordpress.com/wp-content/uploads/2021/11/nucpod.jpg?w=1024)
