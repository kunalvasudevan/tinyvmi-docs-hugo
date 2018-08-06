+++
title= "Get Target Guest Info (Linux)"
date= 2018-06-04T22:24:45-04:00
description = ""
weight = 6
+++


#### Start Target VM with FLASK label {#guest-start}

According to our setup in [Update XSM FLASK Policy]({{<ref "step-by-step/configuration/update-xsm-flask-policy.md">}}), a VM labeled with ``domU`` has priviledges to intropsect another domU's physical memory. Now we can label TinyVMI with ``domU`` to grant the VM of TinyVMI with those privileges. 

Change xl config file for the target VM. Add the following line to this file:

    seclabel='system_u:system_r:domU_t'

An example configuration file:

```
# Kernel image file.
kernel = "mini-os.gz"

# Initial memory allocation (in megabytes) for the new domain.
memory = 64

# A name for your domain. All domains must have different names.
name = "TinyVMI"

on_crash = 'destroy'

seclabel='system_u:system_r:domU_t'
```

Now you can start target VM by running ``xl create <domain_config_file>``

#### Get Guest Kernel Offsets {#guest-offset}

{{% panel theme="success" header="NOTE" %}}
This method is totally derived from [how LibVMI get info from guest kernel](https://github.com/libvmi/libvmi/tree/master/tools/linux-offset-finder).
{{% /panel %}}

Download the [linux-offset-finder](https://github.com/libvmi/libvmi/tree/master/tools/linux-offset-finder) from LibVMI repo. Then you can use this program to get the offset values needed for
the ~/etc/libvmi.conf file.  To use, follow the steps below.

1. Copy the files in this directory to the target VM. 
2. Log into the target VM as root.
3. cd into the directory with this program, then run make.
4. insmod findoffsets.ko
5. if you are logged into the console, you will see the output. Otherwise, see /var/log/syslog or dmesg for the output.
6. rmmod findoffsets
7. copy the output into your ~/etc/libvmi.conf file in dom0, be sure to update the domain name and sysmap location.


#### Get Guest Kernel Sysmap {#guest-sysmap}

Copy the system map file to dom0. For example, on a guest VM running with Ubuntu 16.04, you can find system map file at /boot/Sysmap.map-\<kernel\_version>, then copy it in ~/etc/Sysmap.map-\<kernel\_version>.  The current kernel_verion can be got by run ``uname -r``.

{{%panel theme="success" header="Summary"%}}
Now we have the following info from the target guest VM:

1. target VM name and ID.
2. ~/etc/libvmi.conf. Acquired by ``linux-offset-finder`` in target VM.
3. ~/etc/System.map-\<kernel_version>. Copied from /boot/System.map-* on target VM.

Next, we will use those info to configure TinyVMI.
{{%/panel %}}

