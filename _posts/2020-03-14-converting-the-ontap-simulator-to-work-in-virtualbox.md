---
layout: post
permalink: https://madlabber.wordpress.com/2020/03/14/converting-the-ontap-simulator-to-work-in-virtualbox/
title: Converting the ONTAP Simulator to work in VirtualBox
description: None
date: 2020-03-15 02:51:52 -0000
last_modified_at: 2020-03-15 17:48:33 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2020/03/vsim-vbox.jpg
categories:
- Uncategorized
tags:
- NetApp
- VirtualBox
- virtualization
---
This is a topic that comes up after just about every ONTAP release. The ONTAP simulator is only supported on VMware hypervisors, but with a little effort it can run in VirtualBox. Simulate ONTAP, also known as the ONTAP simulator, or the vsim, is distributed as virtual appliance in OVA format, but VirtualBox fails to import the OVA. Instead, it throws some variation of this error:

![](https://madlabber.wordpress.com/wp-content/uploads/2020/03/vbox-error.jpg?w=768)

The underlying problem is that virtual box doesn't connect IDE devices to the correct IDE ports unless they are presented in just the right order within the OVF xml file. This can be overcome by modifying the xml by to avoid the bug in VirtualBox, but that is tedious. There is another way. VirtualBox has a nice 'vboxmanage' command line interface that can be used to rebuild the VM, and then export it to a VirtualBox friendly OVA file. It is this more scriptable approach that will be used here.

**Obtaining the Simulator**

The simulator is available to NetApp customers, most partners, and employees. It is not available to guest accounts. Guest accounts can access the ONTAP Select evaluation as an alternative.

For ONTAP 9.7, the download is available from the beta support site at:  
<https://mysupport-beta.netapp.com/site/tools/tool-eula/5e31797415040d3cce0033d3>

Previous versions can be downloaded from the tool chest at:  
<https://mysupport.netapp.com/tools/info/ECMLP2538456I.html?productID=61970>

**Scripting the OVA conversion**

This example script is in bash, and uses the ONTAP 9.7 version of the simulator. It runs as-is on OS X but some adjustments may be needed for other operating systems or other versions of the simulator.

Grab the complete script from this GitHub repo, then follow along with the rest of the blog.  
<https://github.com/madlabber/vsim2vbox>

Lets start by defining some variable. These variables will need to be adjusted if working with a different version of the simulator. name is the base name of the ova file, without the extension. memory is the amount of ram to assign to the VM. The VMware version only needs about 5gb, but on VirtualBox a bit more ram is needed. The IDExx variables hold the names of the corresponding VMDK files within OVA archive.
  
    #!/bin/bash
    name="vsim-netapp-DOT9.7-cm_nodar"
    memory=6192
    IDE00="vsim-NetAppDOT-simulate-disk1.vmdk"
    IDE01="vsim-NetAppDOT-simulate-disk2.vmdk"
    IDE10="vsim-NetAppDOT-simulate-disk3.vmdk"
    IDE11="vsim-NetAppDOT-simulate-disk4.vmdk"

And do a little sanity check to make sure the vboxmanage command is available
  
    if [ -z "$(which vboxmanage)" ];then echo "vboxmanage not found";exit;fi

Now we can extract the contents of the OVA, which is just a tar file with some specific formatting constraints.
  
    tar -xzvf "$name".ova

Some older versions of the simulator also gzip the VMDK files, so if you are working with an older version be sure to decompress the VMDK files as well.

Now use vboxmanage to create a new VM
  
    vboxmanage createvm --name "$name" --ostype "FreeBSD_64" --register
    vboxmanage modifyvm "$name" --ioapic on
    vboxmanage modifyvm "$name" --vram 16
    vboxmanage modifyvm "$name" --cpus 2
    vboxmanage modifyvm "$name" --memory "$memory"

Next add the NICs. Here the internal network, intnet, is used because it makes the conversion predictably scriptable. When the final ova is re-imported, adjust the networking to suite your needs.
  
    vboxmanage modifyvm "$name" --nic1 intnet --nictype1 82545EM --cableconnected1 on
    vboxmanage modifyvm "$name" --nic2 intnet --nictype2 82545EM --cableconnected2 on
    vboxmanage modifyvm "$name" --nic3 intnet --nictype3 82545EM --cableconnected3 on
    vboxmanage modifyvm "$name" --nic4 intnet --nictype4 82545EM --cableconnected4 on

Next add 2 serial ports. These are not needed on VMware, but ONTAP may hang on VirtualBox if these are not present.
  
    vboxmanage modifyvm "$name" --uart1 0x3F8 4
    vboxmanage modifyvm "$name" --uart2 0x2F8 3

The VirtualBox bios enumerate disks a little differently, so to maintain the original device order presented to the VM add an empty virtual floppy drive.
  
    vboxmanage storagectl "$name" --name floppy --add floppy --controller I82078 --portcount 1
    vboxmanage storageattach "$name" --storagectl floppy --device 0 --medium emptydrive

And now we can add the IDE controller and attach those VMDK files.
  
    vboxmanage storagectl "$name" --name IDE    --add ide    --controller PIIX4  --portcount 2
    vboxmanage storageattach "$name" --storagectl IDE --port 0 --device 0 --type hdd --medium "$IDE00"
    vboxmanage storageattach "$name" --storagectl IDE --port 0 --device 1 --type hdd --medium "$IDE01"
    vboxmanage storageattach "$name" --storagectl IDE --port 1 --device 0 --type hdd --medium "$IDE10"
    vboxmanage storageattach "$name" --storagectl IDE --port 1 --device 1 --type hdd --medium "$IDE11"

Now that the VM is built, vboxmanage can export it to an OVA file that VirtualBox can understand.
  
    vboxmanage export "$name" -o "$name"-vbox.ova

And finally, remove the temporary VM from VirtualBox.
  
    vboxmanage unregistervm "$name" --delete

The end result should be a -vbox.ova that can be cleanly imported into VirtualBox. But there are a couple of considerations when running the simulator on this platform.

First, this isn't VMware, so the VMware tools service will throw errors at startup. That can be avoided by setting a variable at the loader prompt that disables that service.
  
    setenv bootarg.vm.run_vmtools false

Second, the NICs are not enumerated in the order one would expect. Here is the mapping of VirtualBox network adapters to ONTAP ports:

Adapter 1: e0d  
Adapter 2: e0a  
Adapter 3: e0b  
Adapter 4: e0c

And finally, beware the nat network. ONTAP has multiple ethernet ports that expect to be able to communicate with each other over the network, but the VirtualBox NAT network isolates each port to its own private broadcast domain. As a result the NAT network will not work.

Aside from that you should now be able to run a completely unsupported instance of the ONTAP Simulator in VirtualBox.

![](https://madlabber.wordpress.com/wp-content/uploads/2020/03/vsim-vbox.jpg?w=1024)
