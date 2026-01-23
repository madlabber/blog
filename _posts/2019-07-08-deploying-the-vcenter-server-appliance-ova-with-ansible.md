---
permalink: /2019/07/08/deploying-the-vcenter-server-appliance-ova-with-ansible/
title: Deploying the vCenter Server Appliance OVA with Ansible
description: None
date: 2019-07-09 03:36:36 -0000
last_modified_at: 2019-07-09 03:36:36 -0000
publish: true
pin: false
categories:
- Uncategorized
tags:
- Ansible
- Automation
- OVA
- OVF
- vcenter
- virtualization
- vmware
---
The goal of this playbook is to deploy the VCSA virtual appliance without any additional user interaction, but the strategy used here will work with any OVA that leverages OVF properties as part of its deployment.

The Ansible module that does the bulk of the work is vmware_deploy_ovf. The outline is pretty straightforward:
  
      - vmware_deploy_ovf:
          hostname: '{{ esxi_address }}'
          username: '{{ esxi_username }}'
          password: '{{ esxi_password }}'
          name: '{{ vcenter_hostname }}'
          ovf: '{{ vcsa_ova_file }}' 
          wait_for_ip_address: true
          validate_certs: no
          inject_ovf_env: true
          properties:
            property: value
            property: value
            ...
        delegate_to: localhost

Setting 'inject_ovf_env' to 'true' will pass the properties to the VM at power on. We just have to know what properties the virtual appliance is expecting. To get those properties, we have to pull apart the OVA and examine its ovf xml file.

In the case of the VCSA, first we need to grab the OVA file. It's located in the vcsa folder within the VCSA ISO.

![](/https:/madlabber.wordpress.com/wp-content/uploads/2019/07/vcsa-location.png)

Next we need to extract the OVA to get at the ovf file. OVA files are tar archives with a specific set of constraints, so anything that can extract a tar can extract an OVA.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/ovf-file.png)

The OVF file defines the virtual machine(s), deployment options, and properties that make up the virtual appliance. It is really just an xml file, and it can be viewed in any text editor.

The first thing to look for in a OVF file is the DeploymentOptionSection. This section is not always present, but if an OVF supports multiple deployment options they are defined here. In the case of the VCSA, there are a lot of deployment options to choose from. Most of the time for my homelab or nested lab scenarios the one I want is 'tiny'.

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/deploymentoptions.png)

This becomes the first value in the property list in the playbook:
  
          properties:
            DeploymentOption.value: 'tiny'

Next move on to the property section of the xml:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/ovfproperties.png)

The property name is the value of ovf:key, and the default value is defined in ovf:value. Expected values are usually explained by the <description> text, which is what the OVF deployment UI in vCenter would present to a user performing this task interactively. Notice that the value of ovf:userConfigurable is not always true. Some values are not intended to be user configurable. Also some value are not applicable to all deployment options. In the case of the VCSA several properties are specific to upgrades, so I'll ignore those.

The VCSA has one property that has userConfigurable:false that we need to configure anyway to make the deployment fully automatic:

![](https://madlabber.wordpress.com/wp-content/uploads/2019/07/autoconfig.png)

Currently, Ansible allows properties to be configured in the playbook regardless of the userConfigurable setting, at least when the deployment target is an ESXi host. If you ever need to flip userConfigurable to true for some property, the OVF file itself can be edited, but doing so invalidates the hashes in the .mf (manifest) file within the OVA. In this case just delete the .mf file and deploy the edited ovf file instead of the original ova file.

Here are the relevant properties extracted from the .ovf, formatted for the vmware_deploy_ovf module:
  
          inject_ovf_env: true
          properties:
            DeploymentOption.value: '{{ vcsa_size }}' 
            guestinfo.cis.appliance.net.addr.family: 'ipv4' 
            guestinfo.cis.appliance.net.mode: 'static' 
            guestinfo.cis.appliance.net.addr: '{{ vcenter_address }}' 
            guestinfo.cis.appliance.net.pnid: '{{ vcenter_fqdn }}'
            guestinfo.cis.appliance.net.prefix: '{{ net_prefix }}' 
            guestinfo.cis.appliance.net.gateway: '{{ net_gateway }}' 
            guestinfo.cis.appliance.net.dns.servers: '{{ dns_servers }}' 
            guestinfo.cis.appliance.root.passwd: '{{ vcenter_password }}' 
            guestinfo.cis.ceip_enabled: "False"
            guestinfo.cis.deployment.autoconfig: 'True' 
            guestinfo.cis.vmdir.password: '{{ vcenter_password }}' 
            domain: '{{ domain }}'
            searchpath: '{{ searchpath }}'

I've substituted Ansible variable names so I can define these in the vars file associated with this playbook.

Once the OVF deploys, the module can wait for an IP address before continuing. Sometimes this is good enough, but the VCSA may take another 20 minutes or so before it is actually usable. To make sure it is at least ready to take API calls before continuing, I will run vmware_about_facts once a minute until it succeeds.
  
      - name: Wait for vCenter
        vmware_about_facts:
          hostname: '{{ vcenter_address }}'
          username: 'administrator@vsphere.local'
          password: '{{ vcenter_password }}'
          validate_certs: no
        delegate_to: localhost
        retries: 30
        delay: 60
        register: result           
        until: result is succeeded 

In my lab this playbook takes about 20 minutes to run, and the web client takes a few additional minutes to be ready for interactive use.

The complete playbook and vars file for deploying vCenter can be found in this [github repo](https://github.com/madlabber/Deploy-VCSA-Ansible). This was developed on vCenter 6.7U1, but new versions may bring new properties, or existing properties may be renamed. By following the process outlined in this post, you can adapt the playbook as the VCSA evolves over time, or apply this approach to automating the deployment of other virtual appliances in your environment.
