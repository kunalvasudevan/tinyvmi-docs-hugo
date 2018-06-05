+++
title= "Update Stubdom Makefile"
date= 2018-06-04T22:24:30-04:00
description = ""
weight = 2
+++

The following changes are made to ``xen-src/stubdom/Makefile`` to support the compilation of TinyVMI in Xen source tree ([full makefile](https://github.com/tinyvmi/tinyvmi/tree/master/docs/xen-stubdom/example_makefile)):

{{< highlight patch >}}

--- stubdom/Makefile.old	2017-12-13 06:37:59.000000000 -0500
+++ stubdom/Makefile	2018-05-10 16:44:54.053957803 -0400
@@ -61,7 +61,7 @@
 
 TARGET_LDFLAGS += -nostdlib -L$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/lib
 
-TARGETS=$(STUBDOM_TARGETS)
+TARGETS=$(STUBDOM_TARGETS) c xenstore tinyvmi
 
 STUBDOMPATH="stubdompath.sh"
 genpath-target = $(call buildmakevars2file,$(STUBDOMPATH))
@@ -493,6 +493,16 @@
 c: $(CROSS_ROOT) c-minios-config.mk
 	CPPFLAGS="$(TARGET_CPPFLAGS) $(shell cat c-minios-config.mk)" CFLAGS="$(TARGET_CFLAGS)" $(MAKE) DESTDIR= -C $@ LWIPDIR=$(CURDIR)/lwip-$(XEN_TARGET_ARCH) 
 
+
+###
+# tinyvmi based on c but no glib, 2014.10.27
+###
+
+.PHONY: tinyvmi
+tinyvmi: cross-newlib
+	CPPFLAGS="$(TARGET_CPPFLAGS) $(shell cat c-minios-config.mk)" CFLAGS="$(TARGET_CFLAGS)" $(MAKE) DESTDIR= -C $@ LWIPDIR=$(CURDIR)/lwip-$(XEN_TARGET_ARCH) 
+
+
 ######
 # VTPM
 ######
@@ -565,6 +575,13 @@
 c-stubdom: mini-os-$(XEN_TARGET_ARCH)-c lwip-$(XEN_TARGET_ARCH) libxc c
 	DEF_CPPFLAGS="$(TARGET_CPPFLAGS)" DEF_CFLAGS="$(TARGET_CFLAGS)" DEF_LDFLAGS="$(TARGET_LDFLAGS)" MINIOS_CONFIG="$(CURDIR)/c/minios.cfg" $(MAKE) DESTDIR= -C $(MINI_OS) OBJ_DIR=$(CURDIR)/$< LWIPDIR=$(CURDIR)/lwip-$(XEN_TARGET_ARCH) APP_OBJS=$(CURDIR)/c/main.a
 
+
+## added 20180305, lele
+.PHONY: tinyvmi-stubdom
+tinyvmi-stubdom: mini-os-$(XEN_TARGET_ARCH)-tinyvmi lwip-$(XEN_TARGET_ARCH) libxc xenstore tinyvmi
+	@echo "LELE:target $@ :L563"
+	DEF_CPPFLAGS="$(TARGET_CPPFLAGS)" DEF_CFLAGS="$(TARGET_CFLAGS)" DEF_LDFLAGS="$(TARGET_LDFLAGS)" MINIOS_CONFIG="$(CURDIR)/tinyvmi/minios.cfg" $(MAKE) DESTDIR= -C $(MINI_OS) OBJ_DIR=$(CURDIR)/$< LWIPDIR=$(CURDIR)/lwip-$(XEN_TARGET_ARCH) APP_OBJS=$(CURDIR)/tinyvmi/main.a
+
 .PHONY: vtpm-stubdom
 vtpm-stubdom: mini-os-$(XEN_TARGET_ARCH)-vtpm vtpm
 	DEF_CPPFLAGS="$(TARGET_CPPFLAGS)" DEF_CFLAGS="$(TARGET_CFLAGS)" DEF_LDFLAGS="$(TARGET_LDFLAGS)" MINIOS_CONFIG="$(CURDIR)/vtpm/minios.cfg" $(MAKE) -C $(MINI_OS) OBJ_DIR=$(CURDIR)/$< APP_OBJS="$(CURDIR)/vtpm/vtpm.a" APP_LDLIBS="-ltpm -ltpm_crypto -lgmp -lpolarssl"
@@ -604,6 +621,8 @@
 
 install-c: c-stubdom
 
+install-tinyvmi: tinyvmi-stubdom
+
 install-caml: caml-stubdom
 
 install-xenstore: xenstore-stubdom
@@ -658,6 +677,7 @@
 clean:
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-ioemu
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-c
+	rm -fr mini-os-$(XEN_TARGET_ARCH)-tinyvmi #lele, 2018.02.18
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-caml
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-grub
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-xenstore

{{< /highlight >}}