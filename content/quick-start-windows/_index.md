
+++
title="Quick Start with Windows Guest"
date=2018-07-17T12:56:10-04:00
description=""
weight=1
alwaysopen = true
toc = true
+++


{{% panel theme="success" header="NOTE" %}}
This post shows all the comands and configuration files you need to build and run TinyVMI with Windows guest. 
Most of them can be used directly in your terminal on host machine (tested with Ubuntu 16.04, 18.04 as Dom0), while some fields need to be adjusted according to your build environment. 
For example, your working directory **WORKDIR**, logical volume group name **vg0**, iso image file path, etc.

Acknowledgment:
This post is customized for TinyVMI based on instructions at [DRAKVUF](https://drakvuf.com/).
{{% /panel %}}

#### 1. Install Xen
Compile and install Xen from given source code of xen-4.10.0, with XSM enabled.

```bash
mkdir WOKRDIR
cd WORKDIR/
# install prerequest for Ubuntu
wget https://gist.githubusercontent.com/cnlelema/5f14675364a47c6ffa7e34bb6d3ad470/raw/41cffdbc8d0c689e8d9ba78d886a215125d833d9/install-pre-ubu18-xen4.10.0.sh
sudo bash install-pre-ubu18-xen4.10.0.sh

# download Xen source
git clone --recurse-submodules https://github.com/tinyvmi/xen.git
cd xen


# configure xen
./configure --enable-stubdom --enable-systemd

# enable XSM support
make -C xen menuconfig
# Mark option 
#   'Common Features -> Xen Security Modules support', and 
#   suboption 'Compile Xen with a built-in security policy'. 
# leave other option as is.

# compile & install Xen
make dist -jN  #set N to be number of cores/threads on your machine.
sudo make install 

sudo systemctl enable xen-qemu-dom0-disk-backend.service
sudo systemctl enable xen-init-dom0.service
sudo systemctl enable xenconsoled.service

sudo ldconfig

# enable XSM flask in grub entry options

# backup 
sudo cp /etc/default/grub /etc/default/grub-backup

# Add the following line to the file /etc/default/grub:
#     GRUB_CMDLINE_XEN_DEFAULT="dom0_mem=3096M,max:3096M flask=enforcing"
# This will be appended to the option of Linux kernel in the grub entry
sudo sed -i '/GRUB_CMDLINE_LINUX=/aGRUB_CMDLINE_XEN_DEFAULT=\"dom0_mem=3096M,max:3096M flask=enforcing\"' /etc/default/grub

sudo update-grub

# Finally reboot and choose Xen entry upon grub menu
sudo reboot
```

Select Xen to boot upon grub entry. After login, you can verify installation via:

```bash
sudo xl list -Z
```

The output should looks like:
```bash
 
Name                                        ID   Mem VCPUs	State	Time(s)   Security Label
Domain-0                                     0 10240     4     r-----     800.2 system_u:system_r:dom0_t
```

#### 2. Install Windows VM and get Windows symbols. {#install_guest}
You can either create a [Windows VM]({{<relref "quick-start-windows/_index.md#install_guest">}}) or [Linux VM]({{<relref "quick-start-linux/_index.md#install_guest">}}) as target VM to intropect from TinyVMI.

##### Install Windows as guest {#windows-vm}

Create logical volumn for Windows VM:

```bash
# create logical volumn
#   with 
#   size 20G, name 'lvwin7', in volumn group 'vg0'
sudo lvcreate -L20G -n lvwin7 vg0
# verify the lv with its name and size
sudo lvdisplay
# outputs should include
    --- Logical volume ---
    LV Path                /dev/vg0/lvwin7
    LV Name                lvwin7
    VG Name                vg0
    LV UUID                tSN9Kk-****-****-****-****-****-afwDhx
    LV Write Access        read/write
    LV Creation host, time ruisrv, 2018-06-28 13:09:38 -0400
    LV Status              available
    # open                 0
    LV Size                20.00 GiB
    Current LE             5120
    Segments               1
    Allocation             inherit
    Read ahead sectors     auto
    - currently set to     256
    Block device           252:8

```

Create a configuration file use the following template:

```conf
# file windows7.cfg
# ====
# HVM guest created from Windows ISO file
# ====

type = "hvm"
serial='pty'

# Guest name
name = "win7"


# Initial memory allocation (MB)
memory = 4096

# Maximum memory (MB)
# If this is greater than `memory' then the slack will start ballooned
# (this assumes guest kernel support for ballooning)
maxmem = 4096

# Number of VCPUS
vcpus = 2

# Network devices
vif = [ 'type=ioemu, bridge=netbr0' ]

# Disk Devices
# change to your lv path and iso file.
disk = [ '/dev/vg0/lvwin7,raw,xvda,w','/media/iso/en_windows_7_AIO_sp1_x64_x86_dvd.ISO,,hdc,cdrom' ]

# Guest VGA console configuration, either SDL or VNC
#sdl = 1
vnc = 1

vncconsole=1

boot = "dc"
# uncomment this after installation, to avoid boot to iso
#boot = "cd"

# fix mouse cursor
usbdevice='tablet'

# XSM label
seclabel='system_u:system_r:domU_t'

```
Start VM to install Windows:

```bash
xl create windows7.cfg
```

If no iso file for Windows, VM image could be downloaded [here](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/). You will need to convert the Windows VM image from Virtualbox (VMWare) format to Xen Image.


Next, we need to get Windows symbols by getting Rekall profile from Dom0:

```bash
cd WORKDIR/
git clone https://github.com/libvmi/libvmi
cd libvmi
./autogen.sh
./configure --disable-kvm
```

Make sure the output is like the following:
```bash
Feature         | Option
----------------|---------------------------
Xen Support     | --enable-xen=yes
KVM Support     | --enable-kvm=no
File Support    | --enable-file=yes
Shm-snapshot    | --enable-shm-snapshot=no
Rekall profiles | --enable-rekall-profiles=yes
----------------|---------------------------

OS              | Option
----------------|---------------------------
Windows         | --enable-windows=yes
Linux           | --enable-linux=yes


Tools           | Option                    | Reason
----------------|---------------------------|----------------------------
Examples        | --enable-examples=yes
VMIFS           | --enable-vmifs=yes        | yes
```

Build LibVMI:

```bash
make
```

Now create Rekall profile for Windows 7 guest VM:

```bash
# check VM is running
$ sudo xl list
Name                                        ID   Mem VCPUs	StateTime(s)
Domain-0                                     0 10239     4     r-----    2479.8
win7                                         6  4096     2     -b----      84.1
$ cd WORKDIR/libvmi/examples/
$ sudo ./vmi-win-guid name win7
Windows Kernel found @ 0xc04d000
	Version: 64-bit Windows 7
	PE GUID: 4ce7951a5ea000
	PDB GUID: 3844dbb920174967be7aa4a2c20430fa2
	Kernel filename: ntkrnlmp.pdb
	Multi-processor without PAE
	Signature: 17744.
	Machine: 34404.
	# of sections: 24.
	# of symbols: 0.
	Timestamp: 1290245402.
	Characteristics: 34.
	Optional header size: 240.
	Optional header type: 0x20b
	Section 1: .text
	Section 2: INITKDBG
	Section 3: POOLMI
	Section 4: POOLCODE
	Section 5: RWEXEC
	Section 6: .rdata
	Section 7: .data
	Section 8: .pdata
	Section 9: ALMOSTRO
	Section 10: SPINLOCK
	Section 11: PAGELK
	Section 12: PAGE
	Section 13: PAGEKD
	Section 14: PAGEVRFY
	Section 15: PAGEHDLS
	Section 16: PAGEBGFX
	Section 17: PAGEVRFB
	Section 18: .edata
	Section 19: PAGEDATA
	Section 20: PAGEVRFC
	Section 21: PAGEVRFD
	Section 22: INIT
	Section 23: .rsrc
	Section 24: .reloc
```
Important fields are:

```bash
PDB GUID: 3844dbb920174967be7aa4a2c20430fa2
Kernel filename: ntkrnlmp.pdb
```

If vmi-win-guid fails to find the Windows kernel in memory, you can use Rekall to examine ntoskrnl.exe on the disk:

```bash
$ sudo kpartx -l /dev/vg0/lvwin7  # This will show all the partiions in the volumn, e,g
vg0-lvwin7p1 : 0 204800 /dev/vg0/lvwin7 2048
vg0-lvwin7p2 : 0 41734144 /dev/0/lvwin7 206848
$ sudo kpartx -a /dev/vg0/lvwin7
$ sudo mount -o ro /dev/mapper/vg0-lvwin7p2 /mnt  # might change partion to find the Windows/ folder
$ sudo rekal peinfo -f /mnt/Windows/System32/ntoskrnl.exe > WORKDIR/peinfo.txt
$ sudo umount /mnt
$ sudo kpartx -d /dev/vg0/lvwin7
```

The generated WORKDIR/peinfo.txt file will contain the required PDB filename and GUID. 

Given PDB file name and GUID, we can generate the Rekall profile:

```bash
# create a target configuration directory in TinyVMI source tree. 
mkdir WORKDIR/xen/stubdom/tinyvmi/tiny-vmi/config/target_conf_examples/example_target_windows7_sp1_x64_new
cd WORKDIR/xen/stubdom/tinyvmi/tiny-vmi/config/target_conf_examples/example_target_windows7_sp1_x64_new/

rekall fetch_pdb ntkrpamp 3844dbb920174967be7aa4a2c20430fa2
rekall parse_pdb ntkrpamp > windows7-sp1.rekall.json

xxd -i windows7-sp1.rekall.json > target_libvmi_sym.c

```

Create a C header file named "target_libvmi_sym.h" with the following content:

```c
#ifndef TARGET_LIBVMI_SYM_H
#define TARGET_LIBVMI_SYM_H


#ifdef SYM_FILE_FROM_STRING
#undef SYM_FILE_FROM_STRING
#endif

#ifndef REKALL_FILE_FROM_STRING
#define REKALL_FILE_FROM_STRING
#endif  //REKALL_FILE_FROM_STRING


#define guest_rekall_string windows7_sp1_rekall_json
#define guest_rekall_string_len windows7_sp1_rekall_json_len

extern unsigned char windows7_sp1_rekall_json[];
extern unsigned int windows7_sp1_rekall_json_len;

#define guest_rekall_string_SRC_FILE "/tiny-vmi/config/target_libvmi_s
ym.c"


#endif // TARGET_LIBVMI_COMMON_H
```

Under the same directory, create configuration file named libvmi.conf for Windows:

```conf
win7 { 
    ostype = "Windows"; 
    rekall_profile = "string"; 
}
```

And convert it into C string:

```
xxd -i libvmi.conf > target_libvmi_conf_file.c
```

Then create symblic link in TinyVMI source:

```bash
# go to dir WORKDIR/xen/stubdom/tinyvmi/tiny-vmi/config/target_conf 
# and remove old links
cd ../../target_conf
rm *
# create symbolic 
ln -s ../target_conf_examples/example_target_windows7_sp1_x64_new/* .
```

Update ``DOMAIN_NAME`` in file ``xen/stubdom/tinyvmi/include/domain_id.h``. For example:

```c
#ifndef DOMAIN_ID_H
#define DOMAIN_ID_H

#define DOMAIN_NAME "win7"

#endif // DOMAIN_ID_H
```

Then compile and run TinyVMI 

Adjust the Xen configuration file use the template at ``WORKDIR/xen/stubdom/tinyvmi/domain_config``:

```conf
# file xen-src/stubdom/tinyvmi/domain_config
# Change fileds accordingly

# Kernel image file.
kernel = "mini-os.gz"

# Initial memory allocation (in megabytes) for the new domain.
memory = 64

# A name for your domain. All domains must have different names.
name = "TinyVMI"

# network connection
vif = ['ip=10.0.0.100,bridge=netbr0' ]

on_crash = 'destroy'

seclabel='system_u:system_r:domU_t'
```

You might need to adjust the network configuration accordingly, i.e. giving a valid IP address and network bridge name on your domain 0.

```bash
cd WORKDIR/xen/stubdom/tinyvmi
sudo bash run.tiny.sh buildrun
```

Then you'll get outputs like:

```
Xen Minimal OS!
    start_info: 0xf6000(VA)
    nr_pages: 0x2000
    shared_inf: 0xa0278000(MA)
        pt_base: 0xf9000(VA)

....

[main] IP 0 netmask 0 gateway 0.
[main] TCP/IP bringup begins.
Thread "tcpip_thread": pointer: 0x0x200000002170, stack: 0x0x2f0000
[tcpip_thread] TCP/IP bringup ends.
[main] Network is ready.
"main" 
VMI_TEST: Hello, world!
VMI_TEST: main: TimeStamp: 1531894244 s 473686 us
VMI_TEST: main: TimeStamp: 1531894246 s 483747 us
evtchn_open() -> 4
xenevtchn_bind_interdomain(1, 6) = 0
VMI_TEST: LibVMI init succeeded!
VMI_TEST: Waiting for events...
```


Create a file with the following C code in Windows 7:

```c
// int3.c in windows guest
#include <stdio.h>
#include <windows.h>

void loopasm(){
    __asm__("int $3");
}

int main(){
    printf("INT 3 in ASM\n");
    for (int i = 0; i< 10; i++){
        Sleep(2000);
        loopasm();
        printf("INT 3 LOOP %d\n", i);
    }
    printf("loop done\n");
    return 0;
}
```

Compile and run it in Windows guest using CodeBlock (or other alternatives).

![win7_int3](/screenshot/win7_int3.png?height=600&classes=border,shadow)

Then on the output of TinyVMI, should like:

![win7_int3_tinyvmi](/screenshot/win7_int3_tinyvmi.png?height=661&classes=border,shadow)


We can see the int 3 instruction running in Windows guest is caught by TinyVMI.
