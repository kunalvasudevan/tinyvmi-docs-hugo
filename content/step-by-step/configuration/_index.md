+++
title= "Configure TinyVMI"
date= 2018-06-04T22:21:41-04:00
description = ""
weight = 2
alwaysopen = true
+++



Before configure TinyVMI, it is required to install Xen with FLASK enabled and to have a guest VM to be monitored by TinyVMI. To do so, see instructions [here]({{<ref "step-by-step/install-xen-with-flask.md" >}}).


{{% panel theme="success" header="Steps to Configure TinyVMI" %}}

+ [Update XSM FLASK Policy]({{< relref "step-by-step/configuration/update-xsm-flask-policy.md" >}})

+ [Update Makefile under xen-src/stubdom/]({{< relref "step-by-step/configuration/update-stubdom-makefile.md" >}})

+ [Get Target Guest Kernel Info]({{< relref "step-by-step/configuration/get-target-guest-info.md" >}})

+ [Configure TinyVMI with Target Guest Info]({{< relref "step-by-step/configuration/configure-tinyvmi-with-target-guest-info.md" >}})


{{% /panel %}}