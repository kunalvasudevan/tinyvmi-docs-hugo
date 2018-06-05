+++
title= "Build and Run TinyVMI"
date= 2018-06-04T14:43:03-04:00
description = ""
weight = 3
+++


#### Build TinyVMI {#build-tinyvmi}

    # cd ./stubdom/
    # make tinyvmi-stubdom
    
#### Label VM of TinyVMI

Change xl config file for the VM of TinyVMI. A sample could be found in ``xen-src/extras/mini-os/domain_config``. Add the following line to this file:

    seclabel='system_u:system_r:domU_t'

#### Run TinyVMI {#run-tinyvmi}

Now you can start vm by running ``xl create <domain_config_file>``.
For example, 

    # cd minios-x86_64-tinyvmi
    # xl create -c ../../extras/mini-os/domain_config
    

{{% panel theme="success" header="Tips"%}}
You can also directly run a handy bash script in tinyvmi/ directory:

    cd xen-src/stubdom/tinyvmi
    sudo bash run.tiny.sh

{{%/panel%}}