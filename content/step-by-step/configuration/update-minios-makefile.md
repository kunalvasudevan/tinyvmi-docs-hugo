+++
title= "Update Mini-OS Makefile"
date= 2018-07-17T13:18:54-04:00
description = "Update Mini-OS Makefile"
weight = 5
+++

Mini-OS is compiled with a minimal number of libraries essentially for its limited kernel functions. TinyVMI requires more libraries, including **[json-c](https://github.com/json-c/json-c)**, **[jansson](http://www.digip.org/jansson/)**. 

All libraries in Mini-OS need to be statically compiled and linked into the tiny kernel. Under Xen development environment, we could statically compile the library in stubdom's makefile as shown in [the previous step]({{<relref "step-by-step/configuration/update-stubdom-makefile.md">}}). To link the library, the following changes need to be made in the Makefile of Mini-OS source code:

{{< highlight patch >}}

--- extras/mini-os/Makefile.old	2017-10-20 06:50:35.000000000 -0400
+++ extras/mini-os/Makefile	2018-07-02 19:48:24.655456301 -0400
@@ -142,6 +142,9 @@
 LIBS += $(XEN_ROOT)/stubdom/libxc-$(MINIOS_TARGET_ARCH)/libxenctrl.a
 LIBS += $(XEN_ROOT)/stubdom/libxc-$(MINIOS_TARGET_ARCH)/libxenguest.a
 endif
+# APP_LDLIBS += -liconv
+APP_LDLIBS += -ljansson
+APP_LDLIBS += -ljson-c
 APP_LDLIBS += -lpci
 APP_LDLIBS += -lz
 APP_LDLIBS += -lm

{{< /highlight >}}