---
title: "How to create floppy disk images on MacOS"
categories:
  - Blog
  - Automation
  - Virtualization
tags:
  - automation
  - virtualization
---

I know what you're thinking. Why.

The floppy disk may be obsolete in the real world, but sometimes you still need to add one to a virtual machine.  Recently,  I needed to inject an autounattend.xml file into a windows VM so I could bring up a Windows instance from scratch with Ansible.  The best of the bad ideas on the whiteboard was to use a virtual floppy disk.

The usual approach to creating .flp images is to make an empty file with a .flp extension, attach it to a VM, and then format it from inside the VM.  After all, thats what VMware says to do in [kb1002195](https://kb.vmware.com/s/article/1002195).  That method was much too interactive as I tested and iterated on my autounattend.xml file.  What I needed was a scriptable method to create virtual floppy images and inject files without using a helper VM to do it.  As it turns out, hdiutil can create them, it's just not very well documented.  

Here's how you can use the MacOS terminal command line to create floppy disk images from scratch.

First, create a folder containing the contents of the floppy disk you are about to create.  For example:

    mkdir floppyroot
    cp autounattend.xml floppyroot/

Next, use the hdiutil command to create the floppy image from that folder

    hdiutil create -size 1440k -fs "MS-DOS FAT12" -layout NONE -srcfolder floppyroot -format UDRW -ov floppy.dmg 

A few of those hdiutil options could use some context:

    -fs "MS-DOS FAT12" : Forces FAT12. "MS-DOS" alone would default to FAT32
    -layout NONE :  uses the whole disk instead of partitioning it.
    -srcfolder <folder>: This can be omitted if you just want an empty floppy
    -format UDRW: This format option creates an uncompressed image file. This option is only required if -srcfolder is specified.
    -ov: Overwrite the output file if it already exists

And finally, rename the file to .flp so VMware will recognize it.

    mv floppy.dmg floppy.flp

Later, if you need to mount that .flp on MacOS, you can also do that with hdiutil:

    hdiutil attach floppy.flp

The output will include the device path and the mount path.  Either of which can be used to eject the disk:

    hdiutil eject /Volumes/FLOPPYROOT
or

    hdiutil eject /dev/disk2

With the floppy image out of the way, I can move on to crafting an autounattend.xml that enables a newly created Windows VM to be controlled by Ansible. That will be a topic for another day.
