+++
title= "Install Xen With XSM FLASK Policy"
date= 2018-06-04T14:22:24-04:00
description = ""
weight = 1
#toc = true
+++


## Overview

The following steps are tested with Xen 4.10.0, other Xen versions with 4.8 and onward should also be fine. The tested host OSes include Ubuntu 16.04, and 18.04. Other Linux based system should be OK as long as they are compatitable with Xen hypervisor.

{{% panel theme="success" header="Steps"%}}
+ [1. Install Prerequest Packages of Xen]({{< relref "#prerequest" >}})

+ [2. Install Xen]({{< relref "#install-xen" >}})

+ [3. Install XSM Policy]({{< relref "#install-xsm" >}})

+ [4. Update GRUB Scripts with Xen XSM ]({{< relref "#update-grub" >}})

+ [5. Setup Xen Guest Network with WiFi or Ethernet]({{< relref "#xen-network" >}})

+ [6. Install Xen Guest]({{< relref "#xen-guest" >}})

+ [7. Usefull Links]({{<relref "#links">}})

{{%/panel %}}

#### 1. Install Prerequest Packages of Xen {#prerequest}

Install prerequested packages of Xen. Official list of packages can be found [here](https://wiki.xenproject.org/wiki/Compiling_Xen_From_Source#Build_Dependencies). For a quick install, there is a handy script for [Ubuntu 18.04](https://gist.githubusercontent.com/cnlelema/5f14675364a47c6ffa7e34bb6d3ad470/raw/41cffdbc8d0c689e8d9ba78d886a215125d833d9/install-pre-ubu18-xen4.10.0.sh) and [Ubuntu 16.04](https://gist.githubusercontent.com/cnlelema/ca366be63573dbdaa14938107c611897/raw/ff3a4e268c1db8397ba116c695c004ac10821736/install-pre-ubu16-xen4.10.0.sh). You can run it to install all the dependences directly:
```
#
wget https://gist.githubusercontent.com/cnlelema/5f14675364a47c6ffa7e34bb6d3ad470/raw/41cffdbc8d0c689e8d9ba78d886a215125d833d9/install-pre-ubu18-xen4.10.0.sh
sudo bash install-pre-ubu18-xen4.10.0.sh
```
or 
```
#
wget https://gist.githubusercontent.com/cnlelema/ca366be63573dbdaa14938107c611897/raw/ff3a4e268c1db8397ba116c695c004ac10821736/install-pre-ubu16-xen4.10.0.sh
sudo bash install-pre-ubu16-xen4.10.0.sh
```

#### 2. Install Xen {#install-xen}

##### 2.1 configure Xen

Open a terminal in the source code of Xen. Run:

    cd xen-src/
    ./configure --enable-systemd --enable-stubdom  
    
##### 2.2 enable XSM

Run the following command in the terminal:

    make -C xen menuconfig
    
There will be a graphical interactive interface shown in the terminal. Then choose ``Common Features -> enable XSM``, with no other sub options.

##### 2.3 patch qemu-xen (only for Ubuntu 18.04) {#patch-qemu-xen}

Due to the upgraded libc in ubuntu 18.04, some errors would come out when compiling qemu-xen in the old release of Xen. [More details](https://tinyvmi.github.io/gsoc-blog/post/02-build-ubu18/). We need to upgrade qemu-xen to the lastest version to match 

    cd xen-src/tools/
    rm -r qemu-xen/
    git clone https://xenbits.xen.org/git-http/qemu-xen.git
    cd -
    
##### 2.4 build & install Xen
    
    cd xen-src/
    make dist -j4
    # sometimes can fail with -j4, so try without it if error occurs.
    
    sudo make install

##### 2.5 post-install operations

    sudo ldconfig

    sudo systemctl enable xen-qemu-dom0-disk-backend.service
    sudo systemctl enable xen-init-dom0.service
    sudo systemctl enable xenconsoled.service

    sudo systemctl enable xencommons
    sudo systemctl enable xen-watchdog.service


#### 3. Install XSM Policy {#install-xsm}

##### 3.1 compile FLASK policy
    
    cd xen-src/
    make -C tools/flask/policy
        
##### 3.2 copy policy to boot dir
    
    sudo mkdir /boot/flask/
    sudo cp tools/flask/policy/xenpolicy-4.10.0 /boot/flask/
    cd /boot/flask
    sudo ln -s xenpolicy-4.10.0 xenpolicy
       
#### 4. Update Grub Scripts with Xen XSM {#update-grub}

{{% panel %}}

WARNING 1: The following steps are only tested on Ubuntu systems. Other Linux distributions should not follow these commands unless you know what you are doing.

WARNING 2: The following commands are changing your system booting parameters. Mistake operations can make your system unbootable. If you are not sure what you are doing, please first backup the files before change them. 

{{% /panel %}}

##### 4.1 update /etc/default/grub
_We need to enable XSM flask policy in boot command parameters._
Add the following line to the file ``/etc/default/grub`` (backup before change: ``cd /etc/default && cp grub grub.backup``) :

    GRUB_CMDLINE_XEN_DEFAULT="dom0_mem=3096M,max:3096M flask=enforcing"

Change values of ``dom0_mem`` and ``max`` accordingly. They are the actual main memory you want to allocate to Domain 0 on Xen. Make sure they are the same. For the reason, see [ Why should I dedicate fixed amount of memory for Xen dom0](https://wiki.xenproject.org/wiki/Xen_Project_Best_Practices#Why_should_I_dedicate_fixed_amount_of_memory_for_Xen_Project_dom0.3F). 

##### 4.2 update /etc/grub.d/20_linux_xen {#update-grub-linux-xen}

_This step will change grub auto generation script to load XSM policy module automatically._ 
Under Debian systems, you can find the grub auto-generate script at /etc/grub.d/20_linux_xen. **Find the correct place** you need to insert the line of 
```module /boot/flask/<xenpolicy_name>```

where ``<xenpolicy_filename>`` is the XSM FLASK policy file we just compiled.

For example, on ubuntu system, the following line is changed: 

{{< highlight patch >}}
--- /etc/grub.d/backup/20.linux.xen	2018-04-30 16:41:32.885068197 -0400
+++ /etc/grub.d/20_linux_xen	2018-04-30 16:42:02.273066593 -0400
@@ -138,6 +138,7 @@
     sed "s/^/$submenu_indentation/" << EOF
 	echo	'$(echo "$message" | grub_quote)'
 	module	--nounzip   ${rel_dirname}/${initrd}
+	module /boot/flaskpolicy/xenpolicy
 EOF
   fi
   sed "s/^/$submenu_indentation/" << EOF
{{< / highlight >}}

##### 4.3 generate grub files

_Run 'sudo update-grub'._ This will update your grub files (/boot/grub/grub.cfg). 

##### 4.4 check grub file
A successful update of grub should contain at least two changes in the menuentry of file `` /boot/grub/grub.cfg``: 

**a)** the xen kernel loading command line should have options of ``dom0_mem`` and ``flask``. For example, 

    multiboot /boot/xen.gz  placeholder  dom0_mem=3096M,max:3096M flask=enforcing ${xen_rm_opts}


**b).** last line of the menuentry should have xenpolicy loaded. For example, 

    module /boot/flask/xenpolicy

The following is a complete example of a menu entry definition in ``/boot/grub/grub.cfg`` auto-generated by ``update-grub``:

{{< highlight cfg "hl_lines=18 23" >}}

### BEGIN /etc/grub.d/20_linux_xen ###
menuentry 'Ubuntu GNU/Linux, with Xen hypervisor' --class ubuntu --class gnu-linux --class gnu --class os --class xen $menuentry_id_option '    xen-gnulinux-simple-030e940e-4663-4857-bf79-682485d1507f' {
    insmod part_msdos
    insmod ext2
    set root='hd0,msdos1'
    if [ x$feature_platform_search_hint = xy ]; then
    search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos1 --hint-efi=hd0,msdos1 --hint-baremetal=ahci0,msdos1  030e940e-4663-4857-bf79-682485d1507f
    else
    search --no-floppy --fs-uuid --set=root 030e940e-4663-4857-b
f79-682485d1507f
    fi
    echo	'Loading Xen xen ...'
        if [ "$grub_platform" = "pc" -o "$grub_platform" = "" ]; then
            xen_rm_opts=
        else
            xen_rm_opts="no-real-mode edd=off"
        fi
    multiboot	/boot/xen.gz placeholder  dom0_mem=3096M,max:3096M flask=enforcing ${xen_rm_opts}
    echo	'Loading Linux 4.15.0-22-generic ...'
    module	/boot/vmlinuz-4.15.0-22-generic placeholder root=UUID=    030e940e-4663-4857-bf79-682485d1507f ro  quiet splash
    echo	'Loading initial ramdisk ...'
    module	--nounzip   /boot/initrd.img-4.15.0-22-generic
    module /boot/flask/xenpolicy
}

{{< / highlight >}}


##### 4.5 Reboot

Now you can reboot the system to see whether Xen is successfully installed. If successfull, you will be able to boot with Xen hypervisor on grub menu during booting. 

After booted, run ``xl list -Z`` to see the result. A successful install should output something like this:

{{< highlight console >}}
root@ubu18:~# xl list -Z
Name        ID  Mem     VCPUs   State       Time(s)     Security Label
Domain-0    0   3096    4       r-----      14753.8     system_u:system_r:dom0_t
{{< / highlight >}}

#### 5. Setup Xen Guest Network with WiFi or Ethernet {#xen-network}

If WiFi on host, follow [5.1]({{<relref "#wifi-bridge">}}) ~ [5.3]({{<relref "#wifi-guest-if">}}) to set up a NAT network for guest to access Internet.
If Ethernet, please jump to follow [5.4]({{relref "#eth"}}).
##### 5.1 Create a network bridge {#wifi-bridge}

A bridge could be created manually by running:

    brctl addbr natBr
    ifconfig natBr 10.0.0.1 up
         
``natBr`` is the name of the bridge, which is defined by yourself. To automatically create the bridge at boot time in the dom0, you can use following in your /etc/network/interfaces:

    auto natBr
    iface natBr inet static
    bridge_ports none
    bridge_stp no
    address 10.0.0.1
    netmask 255.255.255.0
    network 10.0.0.0
    broadcast 10.0.0.255

The above code will create a bridge named ``natBr`` during system boot. The ip address of the bridge is ``10.0.0.1``. Guest VMs connected to this bridge could get IP addresses with the prefix ``10.0.0.*``.

Once succeed, run command ``brctl show``. The output will contain a line with defined bridge, such as:

    bridge name	    bridge id		    STP enabled 	interfaces
    natBr		    8000.000000000000	no		

Furthermore, ``ifconfig`` will list the bridge info as following:


    natBr: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 10.0.0.1  netmask 255.255.255.0  broadcast 10.0.0.255
            inet6 fe80::dc2a:40ff:fe7e:566d  prefixlen 64  scopeid 0x20<link>
            ether de:2a:40:7e:56:6d  txqueuelen 1000  (Ethernet)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 134  bytes 18848 (18.8 KB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

If you see outputs above, a bridge is successfully created. Then go to next step.

##### 5.2 Enable forwarding and NAT {#wifi-nat-forward}

change file /etc/sysctl.conf, add (or uncomment) the line:

    net.ipv4.ip_forward = 1

run
    $ sudo sysctl -p /etc/sysctl.conf


Enable forwarding and NAT(postrouting & masquerade) with iptables

    $ sudo iptables -A FORWARD --in-interface natBr -j ACCEPT
        
    $ sudo iptables --table nat -A POSTROUTING --out-interface <wifi_interface> -j MASQUERADE
    
replace ``<wifi_interface>`` with the name of your WiFi interface, such as ``wlps0``, or ``lan0``, etc.

##### 5.3 In guest: {#wifi-guest-if}

Update ( adding the bridge configs to VM). To connect the VM to the bridge ddit VM (DomU) cfg and modify the vif line (virtual interface).

    
    vif=['bridge=natBr,ip=10.0.0.111']   # This will connect guest to natBr with IP address 10.0.0.111

you can also leave out the ip= part and set that in the domU:

    # set static ip address in domU, file /etc/network/interfaces
    auto eth0
    iface eth0 inet static
    address 10.0.0.111
    netmask 255.255.255.0
    gateway 10.0.0.1
    dns-nameservers 8.8.8.8 8.8.4.4
        

##### 5.4 If Ethernet on host: {#eth}

Please refer to [Network Configuration Examples -- Debian stype bridge configuration](https://wiki.xen.org/wiki/Network_Configuration_Examples_(Xen_4.1%2B)#Example_Debian-style_bridge_configuration_.28e.g._Debian.2C_Ubuntu.29)


## 6. Install Xen Guest {#xen-guest}

On Ubuntu, you can follow these steps [here](https://help.ubuntu.com/community/Xen#Creating_vms)

## 7. Useful links {#links}

+ Offical Documentation of Xen: [ Xen Seucrity Modules: XSM-FLASK](https://wiki.xenproject.org/wiki/Xen_Security_Modules_:_XSM-FLASK)

+ Setup Xen on Ubuntu: [Xen -- Community Help Wiki](https://help.ubuntu.com/community/Xen)  

+ Network configuration on Xen: [Network Configuration Examples](https://wiki.xen.org/wiki/Network_Configuration_Examples_(Xen_4.1%2B))

+ WiFi Network setup on Xen Dom0:
    * [Manually configuring NAT networking in Xen](http://blog.manula.org/2012/04/manually-configuring-nat-networking-in.html)
    * [Network Configuration Examples -- Network Address Translation](https://wiki.xenproject.org/wiki/Network_Configuration_Examples_(Xen_4.1%2B)#Network_Address_Translation_.28NAT.29)

