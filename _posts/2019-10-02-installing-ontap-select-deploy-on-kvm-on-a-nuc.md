---
layout: post
permalink: https://madlabber.wordpress.com/2019/10/02/installing-ontap-select-deploy-on-kvm-on-a-nuc/
title: Installing ONTAP Select Deploy on KVM (on a NUC)
description: None
date: 2019-10-03 04:46:24 -0000
last_modified_at: 2019-10-03 04:46:24 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2019/10/selectdeploy1.png
categories:
- Uncategorized
tags: []
---
In a previous series of posts I built an ESX host on a NUC and used it to run ONTAP Select. This time around I'll do it on KVM. This is one of those 'prove it actually works' posts, because I keep hearing it doesn't work. That may have been true at one time, but with a quarterly release cadence this is a product that evolves fairly quickly. This post will cover installing KVM and the ONTAP deploy utility, the next post will cover the actual ONTAP Select deployment.

ONTAP Select is supported on KVM so this is mostly just a matter of following the instructions, but the NUC platform brings a few challenges. It only has 4 cores, and it only has 1 NIC, which just like on VMware is a little below the documented system requirements. Unlike VMware, there is no "standalone eval" image. This time I'll build it the proper way, using the ONTAP Deploy utility VM. But first, I need to get KVM up and running.

The hardware specifications are the same as the VMware build:

NUC8i5BEH, (4 cores, 8 threads)  
64GB RAM  
512GB NVME drive  
1TB SSD drive  
Note: To deploy a licensed instance of ONTAP Select a 2TB SSD would be needed.

For this build I chose Centos 7.6 and these install options, based entirely on personal preference:  
Server with GUI  
\+ Virtualization Client  
\+ Virtualization Hypervisor  
\+ Virtualization Tools  
\+ System Administration Tools  
  
I installed Centos to the NVME drive, saving the SATA SSD for later.

During setup I created a local account called 'admin' and set the password for root.

Later in the process I will be creating a bridge for openvswitch and adding the sole 1gb NIC, which will drop the wired network connectivity to the KVM host. So to be able to do that work over SSH I will use the Wifi adapter for host management, and assign the wired interface to a link-local only address.

Time to build KVM. Start by opening an ssh session into the host as admin, and switch to root:
  
    su

Next use yum to install all the dependancies:
  
    yum install -y qemu-kvm libvirt openvswitch virt-install lshw lsscsi lsof

If openvswitch is missing from the repo, you can either build it from source or grab it from the community build service. This post is long enough without a build-from-source detour, so I'll grab it from the CBS.
  
    wget https://cbs.centos.org/kojifiles/packages/openvswitch/2.7.3/1.1fc27.el7/x86_64/openvswitch-2.7.3-1.1fc27.el7.x86_64.rpm
    yum install openvswitch-2.7.3-1.1fc27.el7.x86_64.rpm
  
Next create a storage pool using that data SSD, which on this platform is /dev/sda
  
    virsh pool-define-as select_pool logical --source-dev /dev/sda --target=/dev/select_pool
    virsh pool-build select_pool
    virsh pool-start select_pool
    virsh pool-autostart select_pool
  
Now setup openvswitch
  
    systemctl start openvswitch
    systemctl enable openvswitch
    ovs-vsctl add-br br0
    ifdown eno1
    ovs-vsctl add-port br0 eno1
    ifup eno1
  
Set the queue length rules required for ONTAP Select
  
    echo "SUBSYSTEM=="net", ACTION=="add", KERNEL=="ontapn*", ATTR{tx_queue_len}="5000"" > /etc/udev/rules.d/99-ontaptxqueuelen.rules
    cat /etc/udev/rules.d/99-ontaptxqueuelen.rules
  
Thats it for KVM. Now for the ONTAP Deploy VM. ONTAP Deploy is part deployment utility, part HA mediator, and part license server. It is the standard supported way to deploy ONTAP Select regardless of the hypervisor. Deploy does not have to run on the same host as Select. One deploy instance can manage about 100 instances of Select in an enterprise environment.

A raw image is available for running the Deploy VM on KVM, which you can get from the evaluation section of the Netapp support site, or you can get a 90day eval here: https://www.netapp.com/us/forms/tools/90-day-trial-of-ontap-select.aspx . Start by downloading the ONTAPdeploy raw.tgz file on your local machine and copying it over with scp:
  
    scp ~/Downloads/ONTAPdeploy2.12.1.raw.tgz admin@192.168.123.59:/home/admin

And now back over on the ssh session to the KVM host...  
Extract the tgz:
  
    cd /home/admin
    tar -xzvf ONTAPdeploy2.12.1.raw.tgz
  
Give it a home:
  
    mkdir /home/ontap
    mv ONTAPdeploy.raw /home/ontap

And use virt-install to build a VM around it:
  
    virt-install --name=ontapdeploy --vcpus=2 --ram=4096 --os-type=linux --controller=scsi,model=virtio-scsi --disk path=/home/ontap/ONTAPdeploy.raw,device=disk,bus=scsi,format=raw --network "type=bridge,source=br0,model=virtio,virtualport_type=openvswitch" --console=pty --import --wait 0

Set it to autostart:
  
    virsh autostart ontapdeploy

Next use the virsh console to complete the VMs setup script:
  
    virsh console ontapdeploy

The setup script will look something like this:
  
    Connected to domain ontapdeploy
    Escape character is ^]
    That does not appear to be a valid hostname
    Host name            : ontapdeploy
    Use DHCP to set networking information? [n]: n
    Net mask             : 255.255.255.0
    Gateway              : 192.168.123.1
    Primary DNS address  : 192.168.123.21
    Secondary DNS address:
    Please enter in all search domains separated by spaces (can be left blank):
    Selected IP           : 192.168.123.58
    Selected net mask     : 255.255.255.0
    Selected gateway      : 192.168.123.1
    Selected primary DNS  : 192.168.123.21
    Selected secondary DNS:
    Search domains        :
    Calculated network    : 192.168.123.0
    Calculated broadcast  : 192.168.123.255
    Are these values correct? [y]: y
    Applying network configuration. Please wait...
    Continuing system startup. Please wait...
    Debian GNU/Linux 9 ontapdeploy ttyS0
    ontapdeploy login:
  
The GUI should be available now over https on the specified address. The default credentials are:  
username: admin  
password: admin123  
Log in once now to change the default password, and the system will be ready to deploy ONTAP Select.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/10/selectdeploy1.png?w=1024)
