---
permalink: /2019/09/08/creating-a-new-active-directory-forest-with-ansible/
title: Creating a new Active Directory Forest with Ansible
description: None
date: 2019-09-09 05:10:54 -0000
last_modified_at: 2019-09-12 14:36:35 -0000
publish: true
pin: false
image:
  path: https://madlabber.wordpress.com/wp-content/uploads/2019/09/ad_create.jpg
categories:
- Uncategorized
tags: []
---
Building new AD forests isn't something most of us do often enough to need to automate it, but recently I was talking to a good friend and a fellow homelabber who needed to provision some new domains in his lab. I do this a lot because every lab environment I build gets its own AD forest. When I told him I'd been automating it with Ansible he suggested I write it up for the blog.

[This playbook](https://github.com/madlabber/examples/blob/master/AD_create.yml)creates a new domain in a new forest from a freshly provisioned VM, like the one built in my previous post on [building windows VMs with Ansible.](https://madlabber.wordpress.com/2019/06/23/how-to-build-a-windows-vm-from-scratch-with-ansible/)

The beginning of the playbook defines all the variables needed to provision the new AD Forest. In practice I keep them in a vars file, but to simplify the example playbook I put them in-line.
  
    ---
    - name: Create new Active-Directory Domain & Forest
      hosts: localhost
      vars:
        temp_address: 172.16.108.144
        dc_address: 172.16.108.11
        dc_netmask_cidr: 24
        dc_gateway: 172.16.108.2
        dc_hostname: 'dc01'
        domain_name: "demo.lab"
        local_admin: '.\administrator'
        temp_password: 'Changeme!'
        dc_password: 'P@ssw0rd'
        recovery_password: 'P@ssw0rd'
        upstream_dns_1: 8.8.8.8
        upstream_dns_2: 8.8.4.4
        reverse_dns_zone: "172.16.108.0/24"
        ntp_servers: "0.us.pool.ntp.org,1.us.pool.ntp.org"
      gather_facts: no

Part of the process of preparing this VM to become a domain controller involves setting a static IP, changing its hostname, and changing its password, so I use Ansible's dynamic inventory rather than a static inventory file.

First I add it to inventory using the VMs original IP and password:
  
      tasks:
      - name: Add host to Ansible inventory
        add_host:
          name: '{{ temp_address }}'
          ansible_user: '{{ local_admin }}'
          ansible_password: '{{ temp_password }}'
          ansible_connection: winrm
          ansible_winrm_transport: ntlm
          ansible_winrm_server_cert_validation: ignore
          ansible_winrm_port: 5986
      - name: Wait for system to become reachable over WinRM
        wait_for_connection:
          timeout: 900
        delegate_to: '{{ temp_address }}'

Next set the static IP. This task does not have windows Ansible coverage, so it uses win_shell, which in turn runs the command under Powershell.
  
      - name: Set static IP address
        win_shell: "(new-netipaddress -InterfaceAlias Ethernet0 -IPAddress {{ dc_address }} -prefixlength {{dc_netmask_cidr}} -defaultgateway {{ dc_gateway }})"
        delegate_to: '{{ temp_address }}'  
        ignore_errors: True 

This command will always return failed because once the IP changes Ansible can't check the results of the task. Just set ignore_errors: true and let it time out. Next Add the host back in to inventory under its new IP address:
  
      - name: Add host to Ansible inventory with new IP
        add_host:
          name: '{{ dc_address }}'
          ansible_user: '{{ local_admin }}'
          ansible_password: '{{ temp_password }}'
          ansible_connection: winrm
          ansible_winrm_transport: ntlm
          ansible_winrm_server_cert_validation: ignore
          ansible_winrm_port: 5986 
      - name: Wait for system to become reachable over WinRM
        wait_for_connection:
          timeout: 900
        delegate_to: '{{ dc_address }}'

Next set the local administrator password. This password will become the domain admin password later when the system is promoted to a domain controller.
  
      - name: Set Password
        win_user:
          name: administrator
          password: "{{dc_password}}"
          state: present
        delegate_to: '{{ dc_address }}'
        ignore_errors: True  

Once again re-add it to inventory using its new IP address:
  
      - name: Add host to Ansible inventory with new Password
        add_host:
          name: '{{ dc_address }}'
          ansible_user: '{{ local_admin }}'
          ansible_password: '{{ dc_password }}'
          ansible_connection: winrm
          ansible_winrm_transport: ntlm
          ansible_winrm_server_cert_validation: ignore
          ansible_winrm_port: 5986 
      - name: Wait for system to become reachable over WinRM
        wait_for_connection:
          timeout: 900
        delegate_to: '{{ dc_address }}'

Next set the upstream DNS servers. These will become the DNS forwarders once the AD integrated DNS server is installed.
  
      - name: Set upstream DNS server 
        win_dns_client:
          adapter_names: '*'
          ipv4_addresses:
          - '{{ upstream_dns_1 }}'
          - '{{ upstream_dns_2 }}'
        delegate_to: '{{ dc_address }}'

Next set the upstream NTP servers. Domain controllers should reference an authoritative time source.
  
      - name: Stop the time service
        win_service:
          name: w32time
          state: stopped
        delegate_to: '{{ dc_address }}'
      - name: Set NTP Servers
        win_shell: 'w32tm /config /syncfromflags:manual /manualpeerlist:"{{ntp_servers}}"'
        delegate_to: '{{ dc_address }}'  
      - name: Start the time service
        win_service:
          name: w32time
          state: started  
        delegate_to: '{{ dc_address }}'

Now before proceeding disable the windows firewall. Otherwise the domain firewall policy will prevent later tasks from succeeding after the system reboots. You can re-enable it and set rules to your liking once the playbook is complete.
  
      - name: Disable firewall for Domain, Public and Private profiles
        win_firewall:
          state: disabled
          profiles:
          - Domain
          - Private
          - Public
        tags: disable_firewall
        delegate_to: '{{ dc_address }}'

Before promoting a system to a DC, you should set its hostname. Its much simpler to rename it before it becomes a domain controller. These tasks update the hostname, and reboot if required.
  
      - name: Change the hostname 
        win_hostname:
          name: '{{ dc_hostname }}'
        register: res
        delegate_to: '{{ dc_address }}'
      - name: Reboot
        win_reboot:
        when: res.reboot_required   
        delegate_to: '{{ dc_address }}'

Now you are ready to install active directory and create the domain.
  
      - name: Install Active Directory
        win_feature: >
             name=AD-Domain-Services
             include_management_tools=yes
             include_sub_features=yes
             state=present
        register: result
        delegate_to: '{{ dc_address }}'
      - name: Create Domain
        win_domain: >
           dns_domain_name='{{ domain_name }}'
           safe_mode_password='{{ recovery_password }}'
        register: ad
        delegate_to: "{{ dc_address }}"
      - name: reboot server
        win_reboot:
         msg: "Installing AD. Rebooting..."
         pre_reboot_delay: 15
        when: ad.changed
        delegate_to: "{{ dc_address }}"

Once the system reboots there are a few little cleanup tasks. First domain controllers should use themselves as the DNS server. This should get set during dc_promo, but I like to be sure it gets set:
  
      - name: Set internal DNS server 
        win_dns_client:
          adapter_names: '*'
          ipv4_addresses:
          - '127.0.0.1'
        delegate_to: '{{ dc_address }}'

Next create the reverse lookup zone for the local subnet. The forward lookup zones get created automatically, but the reverse zones do not. Note the retries on this step. At this point in the process the system has rebooted after becoming a domain controller. It takes a while for it to really be ready to continue.
  
      - name: Create reverse DNS zone
        win_shell: "Add-DnsServerPrimaryZone -NetworkID {{reverse_dns_zone}} -ReplicationScope Forest"
        delegate_to: "{{ dc_address }}"    
        retries: 30
        delay: 60
        register: result           
        until: result is succeeded

And the final step in my process is to make sure RDP is enabled so I can remote in and do any one-off customizations:
  
      - name: Check for xRemoteDesktopAdmin Powershell module
        win_psmodule:
          name: xRemoteDesktopAdmin
          state: present
        delegate_to: "{{ dc_address }}"
      - name: Enable Remote Desktop
        win_dsc:
          resource_name: xRemoteDesktopAdmin
          Ensure: present
          UserAuthentication: NonSecure
        delegate_to: "{{ dc_address }}"

Thats the process end to end from newly installed windows server to newly provisioned Active Directory forest. The complete playbook is in the [examples repo](https://github.com/madlabber/examples) on my github.
