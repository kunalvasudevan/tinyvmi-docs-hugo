
+++
title="Quick Start"
date=2018-07-17T12:56:10-04:00
description=""
weight=1
alwaysopen = true
toc = true
+++


1, compile and install Xen from given source code of xen-4.10.0, with XSM enabled.

    # install prerequest for Ubuntu
    wget https://gist.githubusercontent.com/cnlelema/5f14675364a47c6ffa7e34bb6d3ad470/raw/41cffdbc8d0c689e8d9ba78d886a215125d833d9/install-pre-ubu18-xen4.10.0.sh
    sudo bash install-pre-ubu18-xen4.10.0.sh

    # download Xen source
    git clone --recurse-submoduleshttps://github.com/tinyvmi/xen.git
    cd xen


    # configure xen
    ./configure --enable-stubdom --enable-systemd

    # enable XSM support
    make -C xen menuconfig
    # choose 'Common Features -> Xen Security Modules support', no other sub options. 
    
    # compile & install Xen
    make dist -jN  #set N to be number of cores/threads on your machine.
    sudo make install 

    sudo systemctl enable xen-qemu-dom0-disk-backend.service
    sudo systemctl enable xen-init-dom0.service
    sudo systemctl enable xenconsoled.service

    sudo ldconfig

    # compile & install XSM policy
    make -C tools/flask/policy
    sudo mkdir /boot/flaskpolicy
    sudo cp tools/flask/policy/xenpolicy-4.10.0 /boot/flaskpolicy/

    # backup
    sudo cp /etc/grub.d/20_linux_xen /etc/grub.d/back_up_20_linux_xen

    # Add the following line to /etc/grub.d/20_linux_xen
    #     module /boot/flaskpolicy/xenpolicy-4.10.0
    # This will append this line as the last command of Xen grub entry
    sudo sed -i '/--nounzip/a\\tmodule /boot/flaskpolicy/xenpolicy-4.10.0' /etc/grub.d/20_linux_xen

    # backup 
    sudo cp /etc/default/grub /etc/default/grub-backup

    # Add the following line to the file /etc/default/grub:
    #     GRUB_CMDLINE_XEN_DEFAULT="dom0_mem=3096M,max:3096M flask=enforcing"
    # This will be appended to the option of Linux kernel in the grub entry
    sudo sed -i '/GRUB_CMDLINE_LINUX=/aGRUB_CMDLINE_XEN_DEFAULT=\"dom0_mem=3096M,max:3096M flask=enforcing\"' /etc/default/grub

    sudo update-grub

    sudo reboot

    # boot to Xen


2, install target guest VM on Xen. Note: -- Both target guest vm and TinyVMI are set with FLASK label: seclabel='system_u:system_r:domU_t'. A windows VM image could be downloaded [here](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/). You will need to convert the Windows VM image from Virtualbox (VMWare) format to Xen Image. Get target VM symbols according to the following two posts:

- [Get Target Guest Kernel Info]({{<relref "step-by-step/configuration/get-target-guest-info.md" >}});
- [Configure TinyVMI with Target Guest Info]({{<relref "step-by-step/configuration/configure-tinyvmi-with-target-guest-info.md">}}).

3, compile and run tinyvmi as follows:


    # cd ./stubdom/tinyvmi
    # bash run.tiny.sh buildrun
    
Then you'll get outputs like:

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

