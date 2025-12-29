---
layout: post
permalink: https://madlabber.wordpress.com/2020/01/09/vmware-nic-order-may-change-after-supermicro-bios-updates/
title: VMware NIC order may change after SuperMicro BIOS updates.
description: None
date: 2020-01-09 08:33:44 -0000
last_modified_at: 2020-01-09 08:33:44 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2020/01/image-2.png
categories:
- Uncategorized
tags: []
---
I encountered this issue over the holidays while doing some firmware and bios updates in the lab. A couple of my hosts are based on SuperMicro Xeon-D boards from the X10SDV line. These systems have 2x1gb and 2x10gb ports, and the original NIC order enumerated the 1gb ports first.

![](https://madlabber.wordpress.com/wp-content/uploads/2020/01/image-1.png?w=1024)

After updating the the latest BIOS (2.1), one of my ESX hosts did not come back online. I could see from the IPMI console the system had booted, but was not responding to pings. When I checked the management NICs, I discovered the order had changed.

![](https://madlabber.wordpress.com/wp-content/uploads/2020/01/image.png?w=1024)

I had to reconfigure my vSwitch uplinks to accommodate the new NIC order, by re-assigning the management uplink in the DCUI so I could get back into the hosts and fix everything else. I don't know why one system re-ordered the NICs and the other did not, but I am now left with two otherwise identical hosts with different network uplink topologies. That is a mystery for another day, but if you are running SuperMicro based VMware hosts, proceed with caution.

![](https://madlabber.wordpress.com/wp-content/uploads/2020/01/image-2.png?w=790)
