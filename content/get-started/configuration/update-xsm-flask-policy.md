+++
title= "Update XSM FLASK Policy"
date= 2018-06-04T22:21:07-04:00
description = ""
weight = 1
+++


{{% panel theme="success" header="NOTE" %}}

- Before this step, it is required to install Xen with FLASK enabled and to have a guest VM to be monitored by TinyVMI. To do so, see instructions [here]({{<ref "get-started/install-xen-with-flask.md" >}}).

- The following assumes the XSM FLASK policy is booted from file ``/boot/flask/xenpolicy``. However, the file name can be changed to other ones as shown [here]({{<ref "get-started/install-xen-with-flask.md#update-grub-linux-xen" >}}). Or can be dynamically loaded after booting via ``xl loadpolicy <xenpolicy_file>``.

{{% /panel %}}


#### 1. change policy in /tools/flask/policy/modules/

    dom0.te
    domU.te

In general, those two files defines privileges granted to dom0 and domU, respectively. In order to grant enough privileges to TinyVMI as well as dom0, those two files are tentatively changed. An example setup for Xen-4.10.0 can be found from our [github repo](https://github.com/tinyvmi/tinyvmi/blob/master/docs/xen-flask/example_policy/xen_4_10).

{{% panel theme="danger" header="DANGER"%}}

In our proof-of-concept experiment, TinyVMI and target VM are both labeled with ``domU`` for simplicity. However, this is a risky setup where all ``domUs`` have same privilege of introspecting other ``domUs`` on the same Xen hypervisor. Therefore, this should never be used for any commercial or regular usage on personal PCs. 

{{% /panel %}}


#### 2. build policy

    apt-get install checkpolicy
    cd xen-src/
    make -C tools/flask/policy
    
#### 3. load policy

You can either simply to load it dynamically via:

    sudo xl loadpolicy tools/flask/policy/xenpolicy-4.10.0

OR you can make the system boot with the newest policy by 

    sudo cp tools/flask/policy/xenpolicy-4.10.0 /boot/flask/
    
and change grub options accordingly (see [Update Grub Scripts with Xen XSM]({{<relref "get-started/install-xen-with-flask.md#update-grub" >}})).
