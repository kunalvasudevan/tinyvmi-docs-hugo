+++
title= "Update Stubdom Makefile"
date= 2018-06-04T22:24:30-04:00
description = ""
weight = 4
+++

TinyVMI is developed as a [stubdom](https://wiki.xenproject.org/wiki/StubDom) running on top of Xen hypervisor. 

The following changes are made to ``xen-src/stubdom/Makefile`` to support the compilation of TinyVMI with the necessary targets and library dependences ([full makefile example](https://github.com/tinyvmi/tinyvmi/tree/master/docs/xen-stubdom/example_makefile)):

{{< highlight patch >}}

--- stubdom/Makefile.old	2017-12-13 06:37:59.000000000 -0500
+++ stubdom/Makefile	2018-07-02 19:46:48.999545378 -0400
@@ -61,7 +61,7 @@
 
 TARGET_LDFLAGS += -nostdlib -L$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/lib
 
-TARGETS=$(STUBDOM_TARGETS)
+TARGETS=$(STUBDOM_TARGETS) c xenstore tinyvmi
 
 STUBDOMPATH="stubdompath.sh"
 genpath-target = $(call buildmakevars2file,$(STUBDOMPATH))
@@ -101,6 +101,103 @@
 	  $(MAKE) DESTDIR= && \
 	  $(MAKE) DESTDIR= install )
 
+##############
+# Cross-json-c
+##############
+# https://s3.amazonaws.com/json-c_releases/releases/json-c-0.13.1.tar.gz
+
+JSON_C_URL="https://s3.amazonaws.com/json-c_releases/releases"
+JSON_C_VERSION=0.13
+
+# TARGET_CFLAGS += -Wno-error=implicit-fallthrough -Wno-error=declaration-after-statement 
+# TARGET_CFLAGS += -Wno-error=strict-prototypes
+# TARGET_CFLAGS += -Wno-error=implicit-function-declaration 
+TARGET_CFLAGS += -Wno-error
+
+json-c-$(JSON_C_VERSION).tar.gz:
+	$(FETCHER) $@ $(JSON_C_URL)/$@
+
+json-c-$(XEN_TARGET_ARCH): json-c-$(JSON_C_VERSION).tar.gz
+	tar xzf $<
+	mv json-c-$(JSON_C_VERSION) $@
+
+JSON_C_STAMPFILE=$(CROSS_ROOT)/$(GNU_TARGET_ARCH)-xen-elf/lib/libjson-c.a
+.PHONY: cross-json-c
+cross-json-c: $(JSON_C_STAMPFILE)
+$(JSON_C_STAMPFILE): json-c-$(XEN_TARGET_ARCH) $(NEWLIB_STAMPFILE)
+	( cd json-c-$(XEN_TARGET_ARCH) && \
+	  sh autogen.sh && \
+	  CFLAGS="$(TARGET_CPPFLAGS) $(TARGET_CFLAGS)" CC=$(CC) \
+	  ./configure --disable-shared --disable-threading \
+		  --prefix=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf \
+		  --exec-prefix=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf \
+		  --host=$(GNU_TARGET_ARCH)-xen-elf && \
+	  $(MAKE) DESTDIR= && \
+	  $(MAKE) DESTDIR= install )
+
+##############
+# Cross-jansson
+##############
+# http://www.digip.org/jansson/releases/jansson-2.11.tar.gz
+JANSSON_URL="http://www.digip.org/jansson/releases"
+JANSSON_VERSION=2.11
+
+# TARGET_CFLAGS += -Wno-error=implicit-fallthrough -Wno-error=declaration-after-statement 
+# TARGET_CFLAGS += -Wno-error=strict-prototypes
+# TARGET_CFLAGS += -Wno-error=implicit-function-declaration 
+TARGET_CFLAGS += -Wno-error
+
+jansson-$(JANSSON_VERSION).tar.gz:
+	$(FETCHER) $@ $(JANSSON_URL)/$@
+
+jansson-$(XEN_TARGET_ARCH): jansson-$(JANSSON_VERSION).tar.gz
+	tar xzf $<
+	mv jansson-$(JANSSON_VERSION) $@
+
+JANSSON_STAMPFILE=$(CROSS_ROOT)/$(GNU_TARGET_ARCH)-xen-elf/lib/libjansson.a
+.PHONY: cross-jansson
+cross-jansson: $(JANSSON_STAMPFILE)
+$(JANSSON_STAMPFILE): jansson-$(XEN_TARGET_ARCH) $(NEWLIB_STAMPFILE)
+	( cd jansson-$(XEN_TARGET_ARCH) && \
+	  CFLAGS="$(TARGET_CPPFLAGS) $(TARGET_CFLAGS)" CC=$(CC) \
+	  ./configure --prefix=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf \
+		  --host=$(GNU_TARGET_ARCH)-xen-elf && \
+	  $(MAKE) DESTDIR= && \
+	  $(MAKE) DESTDIR= install )
+
+##############
+# Cross-iconv
+##############
+# https://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.15.tar.gz
+ICONV_URL="https://ftp.gnu.org/pub/gnu/libiconv"
+ICONV_VERSION=1.15
+
+# TARGET_CFLAGS += -Wno-error=implicit-fallthrough -Wno-error=declaration-after-statement 
+# TARGET_CFLAGS += -Wno-error=strict-prototypes
+# TARGET_CFLAGS += -Wno-error=implicit-function-declaration 
+TARGET_CFLAGS += -Wno-error
+
+libiconv-$(ICONV_VERSION).tar.gz:
+	$(FETCHER) $@ $(ICONV_URL)/$@
+
+iconv-$(XEN_TARGET_ARCH): libiconv-$(ICONV_VERSION).tar.gz
+	tar xzf $<
+	mv libiconv-$(ICONV_VERSION) $@
+
+ICONV_STAMPFILE=$(CROSS_ROOT)/$(GNU_TARGET_ARCH)-xen-elf/lib/libiconv.a
+.PHONY: cross-iconv
+cross-iconv: $(ICONV_STAMPFILE)
+$(ICONV_STAMPFILE): iconv-$(XEN_TARGET_ARCH) $(NEWLIB_STAMPFILE)
+	( cd iconv-$(XEN_TARGET_ARCH) && \
+	  CFLAGS="$(TARGET_CPPFLAGS) $(TARGET_CFLAGS)" CC=$(CC) \
+	  ./configure --prefix=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf \
+		  --host=$(GNU_TARGET_ARCH)-xen-elf \
+		  --build=x86_64-linux-gnu \
+		  --enable-static \
+		  && \
+	  $(MAKE) DESTDIR= && \
+	  $(MAKE) DESTDIR= install )
+
 ############
 # Cross-zlib
 ############
@@ -493,6 +590,20 @@
 c: $(CROSS_ROOT) c-minios-config.mk
 	CPPFLAGS="$(TARGET_CPPFLAGS) $(shell cat c-minios-config.mk)" CFLAGS="$(TARGET_CFLAGS)" $(MAKE) DESTDIR= -C $@ LWIPDIR=$(CURDIR)/lwip-$(XEN_TARGET_ARCH) 
 
+
+###
+# tinyvmi based on c but no glib, 2014.10.27
+###
+
+tinyvmi-minios-config.mk: $(CURDIR)/tinyvmi/minios.cfg
+	MINIOS_CONFIG="$<" CONFIG_FILE="$(CURDIR)/$@" $(MAKE) DESTDIR= -C $(MINI_OS) config
+
+.PHONY: tinyvmi
+# tinyvmi: cross-iconv cross-jansson cross-json-c cross-newlib $(CROSS_ROOT) tinyvmi-minios-config.mk
+tinyvmi: cross-jansson cross-json-c cross-newlib $(CROSS_ROOT) tinyvmi-minios-config.mk
+	CPPFLAGS="$(TARGET_CPPFLAGS) $(shell cat c-minios-config.mk)" CFLAGS="$(TARGET_CFLAGS)" $(MAKE) DESTDIR= -C $@ LWIPDIR=$(CURDIR)/lwip-$(XEN_TARGET_ARCH) 
+
+
 ######
 # VTPM
 ######
@@ -565,6 +676,13 @@
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
@@ -604,6 +722,8 @@
 
 install-c: c-stubdom
 
+install-tinyvmi: tinyvmi-stubdom
+
 install-caml: caml-stubdom
 
 install-xenstore: xenstore-stubdom
@@ -658,6 +778,7 @@
 clean:
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-ioemu
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-c
+	rm -fr mini-os-$(XEN_TARGET_ARCH)-tinyvmi #lele, 2018.02.18
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-caml
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-grub
 	rm -fr mini-os-$(XEN_TARGET_ARCH)-xenstore

{{< /highlight >}}