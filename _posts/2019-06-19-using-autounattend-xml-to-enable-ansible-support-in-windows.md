---
layout: post
permalink: https://madlabber.wordpress.com/2019/06/19/using-autounattend-xml-to-enable-ansible-support-in-windows/
title: Using autounattend.xml to enable Ansible support in Windows
description: None
date: 2019-06-20 03:33:32 -0000
last_modified_at: 2019-06-20 03:58:06 -0000
publish: true
pin: false
categories:
- Uncategorized
tags:
- Ansible
- Automation
- virtualization
- vmware
- Windows
---
Recently I needed to automate the installation of a Windows Server VM to create a domain controller on a greenfield, standalone ESXi host, and I wanted to do it within an Ansible playbook. This posed a number of challenges. I wanted to be able to easily share the playbook, so I wanted to avoid building a custom Windows OVA. I knew that if I could get either VMware tools or WinRM working I could take control of the system with an Ansible playbook, but even with an unattended installation of Windows both the VMware tools and WinRM listener would require manual intervention. I needed to find a way to perform an unattended install of Windows that could also do all of these things automatically, with just an untouched install ISO and a network connection. That is where the autounattend.xml file comes into play.

When windows boots from the install disc, setup looks for an autounattend.xml answer file on the attached disks. If it finds one it will skip the interactive setup and configure itself per the xml file. Officially these are created using the Windows ADK, but I decided to take a shortcut, and use an online tool to generate the file. The tool I used was the Windows Answer File Generator (AFG) at <https://www.windowsafg.com/> . I had to make some modifications to its output, so you may be better off using the ADK if you need to do any further customizations.

One of the sections in that xml file can be used to define a set of commands to run at first login, called FirstLogonCommands, which can be leveraged to install the VMware tools and enable WinRM, provided the VM gets a working network connection at first boot. I had to add this section by hand because the AFG doesn't have a way to add FirstLogonCommands.

I settled on the following sequence:

  1. Download and Run ConfigureRemotingForAnsible.ps1 from github
  2. Download the VMware tools installer from VMware
  3. Silently install the VMware tools

I made a choice here to pull these resources directly from the internet but I could have easily hosted them on an internal webserver if I needed this to work without internet access. I did not use the normal mechanism for installing VMware tools because I couldn't cleanly automate that on an unconfigured Windows VM.

After a lot of trial and error, I arrived at this set of commands:
  
    powershell -ExecutionPolicy ByPass Invoke-Expression (Invoke-WebRequest -Uri http://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1)
    powerShell "Invoke-WebRequest -Uri http://packages.vmware.com/tools/esx/6.7u2/windows/x64/VMware-tools-10.3.5-10430147-x86_64.exe -OutFile C:\Windows\temp\VMware-tools-10.3.5-10430147-x86_64.exe"
    C:\Windows\temp\VMware-tools-10.3.5-10430147-x86_64.exe /S /v"/qn REBOOT=R"   
  
Here are those commands formatted in XML ready to insert into the autounattend.xml file:
  
    <FirstLogonCommands>
      <SynchronousCommand wcm:action="add">
        <Order>1</Order>
        <CommandLine>powershell -ExecutionPolicy ByPass Invoke-Expression (Invoke-WebRequest -Uri http://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1)</CommandLine>
      </SynchronousCommand>              
      <SynchronousCommand wcm:action="add">
        <Order>2</Order>
        <CommandLine>powerShell "Invoke-WebRequest -Uri http://packages.vmware.com/tools/esx/6.7u2/windows/x64/VMware-tools-10.3.5-10430147-x86_64.exe -OutFile C:\Windows\temp\VMware-tools-10.3.5-10430147-x86_64.exe"</CommandLine>
      </SynchronousCommand>
      <SynchronousCommand wcm:action="add">
        <Order>3</Order>
        <CommandLine>C:\Windows\temp\VMware-tools-10.3.5-10430147-x86_64.exe /S /v"/qn REBOOT=R"</CommandLine>
      </SynchronousCommand>
    </FirstLogonCommands> 
  
The Answer File Generator didn't set a logoncount, so I added that to prevent the instance from always automatically logging in as administrator:
  
    <AutoLogon>
    ...
        <Username>Administrator</Username>
        <LogonCount>1</LogonCount>
    </AutoLogon>
  
I am using the [evaluation edition of server 2016](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2016?filetype=ISO), which has multiple installation options on the ISO. As a result, I have to add an InstallFrom section so it can auto-install the correct one:
  
    <InstallFrom>
        <MetaData wcm:action="add">
            <Key>/IMAGE/NAME</Key>
            <Value>Windows Server 2016 SERVERSTANDARD</Value>
        </MetaData>
    </InstallFrom>
  
And because that is an evaluation ISO, I need to remove the <productkey> section entirely.

I posted the full xml file and a ready-to-use floppy image on github:  
<https://github.com/madlabber/win-unattend-ansible>  
If you use this floppy as-is, the Windows password will be Changeme!  
Be certain you do.

You should also review the [Ansible documentation for setting up windows hosts](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html), since this script may not configure things to your liking.

You can try this out by building a 2016 VM (non-EFI boot), attaching the [Server 2016 Eval ISO](http://software-download.microsoft.com/download/pr/Windows_Server_2016_Datacenter_EVAL_en-us_14393_refresh.ISO) and [this floppy image](https://github.com/madlabber/win-unattend-ansible/blob/master/floppy.flp), and letting it boot. The configuration in the XML is minimal, because I plan to do guest customizations with either the Ansible Windows modules or the Ansible VMware modules.

With the xml file tested in a VM, I am ready to get back to automating new VM creation with Ansible. That is not as straightforward as it should be, but more on that in a future post.
