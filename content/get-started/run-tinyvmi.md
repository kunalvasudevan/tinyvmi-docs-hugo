+++
title= "Build and Run TinyVMI"
date= 2018-06-04T14:43:03-04:00
description = ""
weight = 3
+++

    
{{% panel theme="success" header="Tips"%}}
You can directly run a handy bash script in tinyvmi/ directory:

To build and run TinyVMI altogether in a single command:

```bash
cd xen-src/stubdom/tinyvmi
sudo bash run.tiny.sh build 
```

Or to only build TinyVMI without creating VM:

```bash
cd xen-src/stubdom/tinyvmi
sudo bash run.tiny.sh make
```

Or to only run TinyVMI without rebuild:

```bash
cd xen-src/stubdom/tinyvmi
sudo bash run.tiny.sh make #This will only Run TinyVMI without re-build.
```

For more details, see the following steps.

{{%/panel%}}


#### Build TinyVMI {#build-tinyvmi}

    # cd ./stubdom/
    # make tinyvmi-stubdom
    
#### Grant Xenstore Permission to LibVMI {#grant-xenstore}

Command ``xenstore-chmod`` can be used to change xenstore permissions. ( ``xenstore-ls -p`` to see the current permissions)

[Permission of a Xenstore directory/key](https://xenbits.xen.org/docs/4.6-testing/man/xenstore-chmod.1.html) is defined by format of ``LD``, where ``L`` is a letter for the type of permission and ``D`` is the corresponding domain ID. In order to get the domain ID of TinyVMI, you can first create the VM and pause it immediately (within 2 seconds as the [main function](https://github.com/tinyvmi/tinyvmi/blob/master/main.c) will wait for 2 seconds before start executing its actual workload codes), then get the ID via ``xl list`` or ``xl domid TinyVMI``. For example:

```bash
# start VM and pause immediately
cd ./stubdom/minios-x86_64-tinyvmi
xl create ../tinyvmi/domain_config & && \
xl pause TinyVMI &&
echo "Domain ID of TinyVMI: $(xl domid 'TinyVMI')"
```
Then, you could run

```bash
# change permission of the directory '/local/domain' recursively
$ xenstore-chmod -r '/local/domain' rN
```
This will allow TinyVMI to read the entire directory '/local/domain' in Xenstore.

#### Run TinyVMI {#run-tinyvmi}

Now you can resume the TinyVMI by running ``xl unpause TinyVMI``.
