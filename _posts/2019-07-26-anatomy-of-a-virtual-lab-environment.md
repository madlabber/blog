---
layout: post
permalink: https://madlabber.wordpress.com/2019/07/26/anatomy-of-a-virtual-lab-environment/
title: Anatomy of a Virtual Lab Environment
description: None
date: 2019-07-26 07:34:52 -0000
last_modified_at: 2019-07-26 07:34:52 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2019/07/virtuallabsarchitecture.jpg
categories:
- Uncategorized
tags: []
---
Virtual Labs are everywhere. VMware has HOL (Hands on labs), Microsoft has Hands-on Labs, Cisco has dCloud, NetApp has labondemand, and on and on. They're great for making complete lab environments available for demos, training, and study. But how do they work, and how can they be scaled down to run in a homelab?

Virtual Labs generally have a few things in common. They have isolated network(s) internal to the lab, they contain a collection of pre-configured VMs, and they are accessed via some sort of jump host. Virtual labs are typically cloned into multiple instances, with every lab instance containing an identical set of VMs and networks.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/virtuallabsarchitecture.jpg)

In this simple example, each lab instance has an identical set of VMs, with identical IP addresses. Each lab's gateway connects it to the transit network, and lab users connect to their lab instance through a remote display protocol.

For this scheme to work, each lab needs an isolated internal network. In fact the VMs within these labs should be completely identical, down to the mac addresses of their NICs. There are lots of ways this could be accomplished, with VXLAN and NSX at the top of the list, but those are heavyweight solutions at homelab scale. Instead I'll take a simpler approach, and just use portgroups on an isolated vSwitch to achieve network isolation between lab instances.

Here is a diagram of what those 3 lab instances might look like from a vSwitch perspective:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/virtual-lab-vswitch-example.jpg)

Each lab instance's internal lab network is backed by an individual port group, with a unique vlan assignment. A virtual router acts as the lab gateway, with the router's LAN port connected to the instance's network, and the router's WAN port connected to the VM Network. The router provides NAT to the lab instance, and a simple RDP port forward to the jump host facilitates remote access to the lab environment.

One caveat of using this strategy is that lab instances cannot span hosts. If there was a need for an individual instance to span hosts, the networks would need to be VLAN backed, or provisioned with an overlay technology. In practice there are other reasons to keep the VMs within a given instance running on the same host, so this isn't really a limitation, but it does mean there needs to be a way to group these VMs together so they always run on the same host. I use two VMware features to accomplish this, vApps, and affinity rules.

Here I've grouped my lab instance VMs into vApp containers.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/vlabs.jpg)

vApps can also be cloned using the vCenter UI, providing an easy way to provision more instances. Many of my lab topologies are too complex to survive vApp cloning, but its a good way to get started. If you pre-provision a network for the new lab clone, you can map the VMs to that network as part of the New vApp wizard.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/map-networks.jpg)

Next I can create an affinity rule, to keep all the child VMs of that vApp on the same host. But since I have cloned my vApps, all of the child VM names are the same, and the vCenter UI for creating affinity rules cannot distinguish one from another. In this case, its much easier to just create the rule with a little snippet of PowerCLI:
  
    New-DrsRule -Cluster "Lab Cluster" -Name "VirtualLab-Instance1" -KeepTogether $true -VM (get-vapp "VirtualLab-Instance1" | get-vm)

So far I've covered the general anatomy of a virtual lab, with an emphasis on the networking aspects, and an approach to implementing them on a small scale suitable for a home lab. There is a lot more to cover on this topic. The configuration of the virtual router serving as the gateway, strategies for configuring VMs to survive this kind of cloning, ways to optimize active directory for cloning and long term storage, and strategies for automated provisioning are all important topics. I also have a [project on my github](https://github.com/madlabber/vlab) with my virtual lab automation and provisioning portal, if you want to see how I really do things in my home lab. It's a perpetual work in progress, but for now I'll leave off with a screenshot of the dashboard.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/dashboard.jpg)
