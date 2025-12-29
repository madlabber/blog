---
layout: post
permalink: https://madlabber.wordpress.com/2019/10/07/deploying-ontap-select-on-kvm-on-a-nuc/
title: Deploying ONTAP Select on KVM (on a NUC)
description: None
date: 2019-10-08 06:13:35 -0000
last_modified_at: 2019-10-08 06:13:35 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy09.png
categories:
- Uncategorized
tags: []
---
In my last post I went through the process for getting KVM installed and installing the ONTAP deploy VM. Deploying ONTAP Select is mostly a matter of stepping through a nice wizard, but I will have to make one adjustment in the swagger interface to deploy it on the NUC. Everything in here could be done with RESTful API calls, but unfamiliar things are easier to learn in a GUI.

After logging into a fresh install of the deploy utility you land at this workflow. If you bought a license this is where you would upload the license file. I don't have any licenses to add, so I'll run it as an eval cluster. Click Next.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy01a.png?w=640)

The next step is to add the hypervisor hosts to the inventory, which in this case is just my KVM box. Fill in the form, click add, and wait for it to show up in the list on the right. Next.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy02.png?w=1024)

This page defines the ONTAP Select cluster. In this example, its a single node cluster running 9.6 on KVM. Fill in the form and click Done.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy03.png?w=1024)

Done doesn't really mean done. It just advances to node setup. Under licenses pick Evaluation Mode, then fill out the hypervisor particulars.

Undersized hosts like the NUC may not appear in the the Hosts drop list. It can still be assigned to a host from the cli by connecting to the deploy instance over ssh:
  
    (ONTAPdeploy) node modify -cluster-name otskvm -name otskvm-01 -host-name 192.168.123.59

Under storage, pick the Storage Pool from the drop list and assign some of its capacity to ONTAP. Don't try to assign the entire capacity of the storage pool. ONTAP Select needs about 266GB for its system disks, which are not included in the information presented on this panel. Also to use a license type other than evaluation, the storage pool capacity needs to be set to at least 1tb. Factoring in the system disks the storage pool needs about 1.3TB available to accept a licensed instance of ONTAP Select. Here I am deploying in eval mode, and only assigning 500GB.  
To move on, click Done.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy04.png?w=1024)

The Next button will become enabled, and the final fields before 'create cluster' are the cluster admin password. If you are on a host with 6 cores, click create cluster and you're done. If you are following along on a quad core host like a NUC, we need to use the swagger interface to change a setting that is not exposed in the GUI.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy05.png?w=1024)

When deploy creates the VM, it will reserve a full 4 cores worth of CPU. This creates a VM with optimal performance, but on a host that only has 4 cores we need to dial that back a bit. Note that this should not usually be done in production. If you need to do this in production check in with your account team first to make sure your scenario can be supported.

To access the swagger interface, select "API Documentation" from the help menu. This is where you can access all of the API documentation and try out API calls along the way.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger00.png?w=248)

In the swagger interface, scroll down toward the bottom and expand the clusters section.

Find the GET /clusters section, and click "Try it out!"

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger01.png?w=732) ![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger02.png?w=720)

Record the cluster's id. It becomes an input on the next API call. Scroll down to GET /clusters/{cluster_id}/nodes. Fill in the cluster ID from the first API call, and click "Try it out!". The output returned will have the id of the node.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger04.png?w=727) ![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger06.png?w=722)

Now that we have both the cluster id, and the node id, we can adjust the reservations setting on the node. Scroll on down to:  
PATCH /clusters/{cluster_id}/nodes/{node_id}

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger07.png?w=726)

Fill in the cluster id, the node id, and the changes shown here. Valid values for cpu_reservation_percentage are 25, 50,75, 100, with 100 being the default.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger08.png?w=722)

Once again click "Try it out!", but this time look for {} in the response body and a response code of 200.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/swagger09.png?w=721)

Now switch back to the deploy GUI, pick a cluster admin password, and click create cluster. It will take a while to deploy, but eventually should end in a successful deployment:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy09.png?w=1024)

It will take several minutes for the cluster's ONTAP System Manager web interface to become available on the cluster management IP you specified. Be patient and remember to connect over https. There is even a link to it on the clusters tab of the ONTAP deploy UI.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/09/kvmdeploy10.png?w=422)

Once you have access to the ONTAP system manager, provisioning storage services is the same as it is in any other ONTAP system. For a walkthrough of setting up CIFS services, [see this post](https://madlabber.wordpress.com/2019/08/15/running-an-ontap-select-eval-cluster-on-a-nuc/).
