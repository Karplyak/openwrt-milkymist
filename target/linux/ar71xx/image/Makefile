#
# Copyright (C) 2008-2010 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

define imgname
$(BIN_DIR)/$(IMG_PREFIX)-$(2)-$(patsubst jffs2-%,jffs2,$(patsubst squashfs-%,squashfs,$(1)))
endef

VMLINUX:=$(BIN_DIR)/$(IMG_PREFIX)-vmlinux
UIMAGE:=$(BIN_DIR)/$(IMG_PREFIX)-uImage
fs_squash:=squashfs-only
fs_all:=all
fs_4k:=4k
fs_64k:=64k
fs_128k:=128k
ifeq ($(CONFIG_TARGET_ROOTFS_INITRAMFS),y)
	fs_squash:=initramfs
	fs_all:=initramfs
	fs_4k:=initramfs
	fs_64k:=initramfs
	fs_128k:=initramfs
	VMLINUX:=$(BIN_DIR)/$(IMG_PREFIX)-vmlinux-initramfs
	UIMAGE:=$(BIN_DIR)/$(IMG_PREFIX)-uImage-initramfs
endif

define CompressLzma
  $(STAGING_DIR_HOST)/bin/lzma e $(1) -lc1 -lp2 -pb2 $(2)
endef

define PatchKernelLzma
	cp $(KDIR)/vmlinux $(KDIR)/vmlinux-$(1)
	$(STAGING_DIR_HOST)/bin/patch-cmdline $(KDIR)/vmlinux-$(1) '$(strip $(2))'
	$(call CompressLzma,$(KDIR)/vmlinux-$(1),$(KDIR)/vmlinux-$(1).bin.lzma)
endef

define PatchKernelGzip
	cp $(KDIR)/vmlinux $(KDIR)/vmlinux-$(1)
	$(STAGING_DIR_HOST)/bin/patch-cmdline $(KDIR)/vmlinux-$(1) '$(strip $(2))'
	gzip -9 -c $(KDIR)/vmlinux-$(1) > $(KDIR)/vmlinux-$(1).bin.gz
endef

define MkImageLzma
	mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma \
		-e 0x80060000 -n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
		-d $(1) $(2)
endef

define MkImageGzip
	mkimage -A mips -O linux -T kernel -a 0x80060000 -C gzip \
		-e 0x80060000 -n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
		-d $(1) $(2)
endef

define Image/BuildKernel
	cp $(KDIR)/vmlinux.elf $(VMLINUX).elf
	cp $(KDIR)/vmlinux $(VMLINUX).bin
	gzip -9 -c $(KDIR)/vmlinux > $(KDIR)/vmlinux.bin.gz
	$(call CompressLzma,$(KDIR)/vmlinux,$(KDIR)/vmlinux.bin.lzma)
	dd if=$(KDIR)/vmlinux.bin.lzma of=$(VMLINUX).lzma bs=65536 conv=sync
	dd if=$(KDIR)/vmlinux.bin.gz of=$(VMLINUX).gz bs=65536 conv=sync
	$(call MkImageGzip,$(KDIR)/vmlinux.bin.gz,$(UIMAGE)-gzip.bin)
	$(call MkImageLzma,$(KDIR)/vmlinux.bin.lzma,$(UIMAGE)-lzma.bin)
	$(call Image/Build/Initramfs)
endef

define Image/Build/WRT400N
	$(call PatchKernelLzma,$(2),$(3))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.lzma"` -gt 1310720 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 6488064 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma $(KDIR)/vmlinux-$(2).uImage; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=1280k conv=sync; \
			dd if=$(KDIR)/root.$(1) \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
		wrt400n $(KDIR)/vmlinux-$(2).uImage $(KDIR)/root.$(1) $(call imgname,$(1),$(2))-factory.bin; \
	fi; fi
endef

dir825b1_mtdlayout=mtdparts=spi0.0:256k(uboot)ro,64k(config)ro,1024k(kernel),5184k(rootfs),64k(caldata)ro,1600k(unknown)ro,6208k@0x50000(firmware)
define Image/Build/DIR825B1
	$(call PatchKernelLzma,$(2),$(3) $(dir825b1_mtdlayout))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.lzma"` -gt 1048576 ]; then \
	        echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 5308416 ]; then \
	        echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma $(KDIR)/vmlinux-$(2).uImage; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=1024k conv=sync; \
			dd if=$(KDIR)/root.$(1) \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
		( \
			dd if=$(call imgname,$(1),$(2))-sysupgrade.bin; \
			echo -n "01AP94-AR7161-RT-080619-00"; \
		) > $(call imgname,$(1),$(2))-backup-loader.bin; \
		if [ `stat -c%s $(call imgname,$(1),$(2))-sysupgrade.bin` -gt 4194304 ]; then \
			echo "Warning: $(call imgname,$(1),$(2))-sysupgrade.bin is too big"; \
		else \
			( \
				dd if=$(call imgname,$(1),$(2))-sysupgrade.bin bs=4096k conv=sync; \
				echo -n "00AP94-AR7161-RT-080619-00"; \
			) > $(call imgname,$(1),$(2))-factory.bin; \
		fi; \
	fi; fi
endef

define Image/Build/WZRHPG30XNH
	$(call PatchKernelLzma,$(2),$(3))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.lzma"` -gt 1048576 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 31850496 ]; then \
	        echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma $(KDIR)/vmlinux-$(2).uImage; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=1024k conv=sync; \
			dd if=$(KDIR)/root.$(1) bs=128k conv=sync; \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
		( \
			echo -n -e "# Airstation Public Fmt1\x00\x00\x00\x00\x00\x00\x00\x00"; \
			dd if=$(call imgname,$(1),$(2))-sysupgrade.bin; \
		) > $(call imgname,$(1),$(2))-tftp.bin; \
		buffalo-enc -p $(4) -v 1.76 \
			-i $(call imgname,$(1),$(2))-sysupgrade.bin \
			-o $(KDIR)/$(2).enc; \
		buffalo-tag -b $(4) -p $(4) -a ath -v 1.76 -m 1.01 -l mlang8 \
			-w 3 -c 0x80041000 -d 0x801e8000 -f 1 -r M_ \
			-i $(KDIR)/$(2).enc \
			-o $(call imgname,$(1),$(2))-factory.bin; \
	fi; fi
endef

cameo_mtdlayout=mtdparts=spi0.0:128k(u-boot)ro,64k(config)ro,896k(kernel),2944k(rootfs),64k(art)ro,3840k@0x30000(firmware)
define Image/Build/Cameo
	$(call PatchKernelLzma,$(2),$(3) $(cameo_mtdlayout))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.lzma"` -gt 917504 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 2949120 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma $(KDIR)/vmlinux-$(2).uImage; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=896k conv=sync; \
			dd if=$(KDIR)/root.$(1) \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=896k conv=sync; \
			dd if=$(KDIR)/root.$(1) bs=2880k conv=sync; \
			echo -n $(4); \
		) > $(call imgname,$(1),$(2))-factory.bin; \
	fi; fi
endef

cameo7240_mtdlayout=mtdparts=spi0.0:192k(u-boot)ro,64k(nvram)ro,896k(kernel),2816k(rootfs),64k(mac)ro,64k(art)ro,3712k@0x40000(firmware)
define Image/Build/Cameo7240
	$(call PatchKernelLzma,$(2),$(3) $(cameo7240_mtdlayout))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.lzma"` -gt 917504 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 2818048 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma $(KDIR)/vmlinux-$(2).uImage; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=896k conv=sync; \
			dd if=$(KDIR)/root.$(1) \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=896k conv=sync; \
			dd if=$(KDIR)/root.$(1) bs=2752k conv=sync; \
			echo -n $(4); \
		) > $(call imgname,$(1),$(2))-factory.bin; \
	fi; fi
endef

define Image/Build/AP83
	$(call PatchKernelGzip,$(2),$(3))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.gz"` -gt 1310720 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.gz is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 6619136 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C gzip -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.gz \
			$(KDIR)/vmlinux-$(2).uImage; \
		dd if=$(KDIR)/vmlinux-$(2).uImage \
			of=$(call imgname,kernel,$(2)).bin bs=64k conv=sync; \
		dd if=$(KDIR)/root.$(1) \
			of=$(call imgname,$(1),$(2)-rootfs).bin bs=128k conv=sync; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=1280k conv=sync; \
			dd if=$(KDIR)/root.$(1); \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
	fi; fi
endef

define Image/Build/PB92
	$(call PatchKernelLzma,$(2),$(3))
	if [ `stat -c%s $(KDIR)/vmlinux-$(2).bin.lzma` -gt 917504 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	elif [ `stat -c%s $(KDIR)/root.$(1)` -gt 2818048 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma \
			$(KDIR)/vmlinux-$(2).uImage; \
		dd if=$(KDIR)/vmlinux-$(2).uImage \
			of=$(call imgname,kernel,$(2)).bin bs=64k conv=sync; \
		dd if=$(KDIR)/root.$(1) \
			of=$(call imgname,$(1),$(2)-rootfs).bin bs=128k conv=sync; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=896k conv=sync; \
			dd if=$(KDIR)/root.$(1); \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
	fi
endef

define Image/Build/PB4X
	$(call PatchKernelLzma,$(2),$(3))
	dd if=$(KDIR)/vmlinux-$(2).bin.lzma \
	   of=$(call imgname,kernel,$(2)).bin bs=64k conv=sync
	dd if=$(KDIR)/root.$(1) \
	   of=$(call imgname,$(1),$(2)-rootfs).bin bs=128k conv=sync
	-sh $(TOPDIR)/scripts/combined-image.sh \
		"$(call imgname,kernel,$(2)).bin" \
		"$(call imgname,$(1),$(2)-rootfs).bin" \
		$(call imgname,$(1),$(2))-sysupgrade.bin
endef

define Image/Build/DB120
	$(call PatchKernelLzma,$(2),$(3))
	if [ `stat -c%s $(KDIR)/vmlinux-$(2).bin.lzma` -gt 1441792 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	elif [ `stat -c%s $(KDIR)/root.$(1)` -gt 6488064 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma \
			$(KDIR)/vmlinux-$(2).uImage; \
		dd if=$(KDIR)/vmlinux-$(2).uImage \
			of=$(call imgname,kernel,$(2)).bin bs=64k conv=sync; \
		dd if=$(KDIR)/root.$(1) \
			of=$(call imgname,$(1),$(2)-rootfs).bin bs=128k conv=sync; \
		( \
			dd if=$(KDIR)/root.$(1); \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=1408k conv=sync; \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
	fi
endef

define Image/Build/AP121
	$(call PatchKernelLzma,$(2),$(3) $($(4)))
	$(call MkImageLzma,$(KDIR)/vmlinux-$(2).bin.lzma,$(KDIR)/vmlinux-$(2).uImage)
	if [ `stat -c%s $(KDIR)/vmlinux-$(2).uImage` -gt $(5) ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).uImage is too big"; \
	elif [ `stat -c%s $(BIN_DIR)/$(IMG_PREFIX)-root.$(1)` -gt $(6) ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		cp $(KDIR)/vmlinux-$(2).uImage $(call imgname,kernel,$(2)).bin; \
		cp $(BIN_DIR)/$(IMG_PREFIX)-root.$(1) $(call imgname,$(1),$(2)-rootfs).bin; \
		( \
			dd if=$(BIN_DIR)/$(IMG_PREFIX)-root.$(1) bs=$(6) conv=sync; \
			dd if=$(KDIR)/vmlinux-$(2).uImage; \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
	fi
endef

define Image/Build/AP121/initramfs
	$(call PatchKernelLzma,$(2),$(3) $($(4)))
	$(call MkImageLzma,$(KDIR)/vmlinux-$(2).bin.lzma,$(call imgname,$(1),$(2))-uImage.bin)
endef

define Image/Build/MyLoader
	-$(STAGING_DIR_HOST)/bin/mkmylofw -B $(2) -s $(3) \
		-p0x030000:0xe0000:al:0x80060000:kernel:$(KDIR)/vmlinux.bin.lzma \
		-p0x110000:0:::rootfs:$(KDIR)/root.$(1) \
		$(call imgname,$(1),$(2))-$(4)-factory.img
endef

ubntxm_mtdlayout=mtdparts=spi0.0:256k(u-boot)ro,64k(u-boot-env)ro,1024k(kernel),6528k(rootfs),256k(cfg)ro,64k(EEPROM)ro,7552k@0x50000(firmware)
define Image/Build/UBNTXM
	$(call PatchKernelLzma,$(2),$(3) $(ubntxm_mtdlayout))
	$(call MkImageLzma,$(KDIR)/vmlinux-$(2).bin.lzma,$(KDIR)/vmlinux-$(2).uImage.bin)
	dd if=$(KDIR)/vmlinux-$(2).uImage.bin of=$(KDIR)/vmlinux-$(2).uImage bs=1024k conv=sync
	-$(STAGING_DIR_HOST)/bin/mkfwimage \
		-B $(4) -v $(5).$(6).OpenWrt.$(REVISION) \
		-k $(KDIR)/vmlinux-$(2).uImage \
		-r $(BIN_DIR)/$(IMG_PREFIX)-root.$(1) \
		-o $(call imgname,$(1),$(2))-factory.bin
	( \
		dd if=$(KDIR)/vmlinux-$(2).uImage; \
		dd if=$(BIN_DIR)/$(IMG_PREFIX)-root.$(1); \
	) > $(call imgname,$(1),$(2))-sysupgrade.bin
endef

define Image/Build/UBNT
	$(call PatchKernelLzma,$(2),$(3))
	dd if=$(KDIR)/vmlinux-$(2).bin.lzma of=$(KDIR)/vmlinux-$(2).lzma bs=64k conv=sync
	-$(STAGING_DIR_HOST)/bin/mkfwimage \
		-B $(4) -v $(5).$(6).OpenWrt.$(REVISION) \
		-k $(KDIR)/vmlinux-$(2).lzma \
		-r $(BIN_DIR)/$(IMG_PREFIX)-root.$(1) \
		-o $(call imgname,$(1),$(2))-factory.bin
	-sh $(TOPDIR)/scripts/combined-image.sh \
		"$(KDIR)/vmlinux-$(2).lzma" \
		"$(BIN_DIR)/$(IMG_PREFIX)-root.$(1)" \
		$(call imgname,$(1),$(2))-sysupgrade.bin
endef

define Image/Build/Planex
	$(call PatchKernelGzip,$(2),$(3))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.gz"` -gt 1441792 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.gz is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 6356992 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C gzip -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.gz $(KDIR)/vmlinux-$(2).uImage; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=1408k conv=sync; \
			dd if=$(KDIR)/root.$(1); \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
		$(STAGING_DIR_HOST)/bin/mkplanexfw \
			-B $(2) \
			-v 2.00.00 \
			-i $(call imgname,$(1),$(2))-sysupgrade.bin \
			-o $(call imgname,$(1),$(2))-factory.bin; \
	fi; fi
endef

define Image/Build/TPLINK
	$(call PatchKernelGzip,$(2),$(3))
	-$(STAGING_DIR_HOST)/bin/mktplinkfw \
		-B $(4) -N OpenWrt -V $(REVISION)\
		-k $(KDIR)/vmlinux-$(2).bin.gz \
		-r $(BIN_DIR)/$(IMG_PREFIX)-root.$(1) \
		-o $(call imgname,$(1),$(2))-factory.bin
	-$(STAGING_DIR_HOST)/bin/mktplinkfw \
		-B $(4) -N OpenWrt -V $(REVISION) -s \
		-k $(KDIR)/vmlinux-$(2).bin.gz \
		-r $(BIN_DIR)/$(IMG_PREFIX)-root.$(1) \
		-o $(call imgname,$(1),$(2))-sysupgrade.bin
endef

define Image/Build/TPLINK/initramfs
	$(call PatchKernelGzip,$(2),$(3))
	-$(STAGING_DIR_HOST)/bin/mktplinkfw -c \
		-B $(4) -N OpenWrt -V $(REVISION) -s \
		-k $(KDIR)/vmlinux-$(2).bin.gz \
		-o $(call imgname,$(1),$(2))-uImage.bin
endef

define Image/Build/CyberTAN
	$(call PatchKernelGzip,$(2),$(3))
	$(call MkImageGzip,$(KDIR)/vmlinux-$(2).bin.gz,$(KDIR)/vmlinux-$(2).uImage)
	$(STAGING_DIR_HOST)/bin/trx -o $(KDIR)/image.tmp -f $(KDIR)/vmlinux-$(2).uImage \
		-x 32 -a 0x10000 -x -32 -f $(KDIR)/root.$(1)
	-$(STAGING_DIR_HOST)/bin/addpattern -B $(2) -v v$(4) \
		-i $(KDIR)/image.tmp \
		-o $(call imgname,$(1),$(2))-sysupgrade.bin
	-$(STAGING_DIR_HOST)/bin/addpattern -B $(2) -v v$(4) -g \
		-i $(KDIR)/image.tmp \
		-o $(call imgname,$(1),$(2))-factory.bin
	rm $(KDIR)/image.tmp
endef

define Image/Build/Netgear
	$(call PatchKernelLzma,$(2),$(3) $($(4)))
	$(call MkImageLzma,$(KDIR)/vmlinux-$(2).bin.lzma,$(KDIR)/vmlinux-$(2).uImage)
	mkdir -p $(KDIR)/wndr3700/image
	$(STAGING_DIR_HOST)/bin/wndr3700 \
		$(KDIR)/vmlinux-$(2).uImage \
		$(KDIR)/wndr3700/image/uImage \
		$(5)
	$(STAGING_DIR_HOST)/bin/mksquashfs-lzma \
		$(KDIR)/wndr3700 $(KDIR)/vmlinux-$(2).uImage.squashfs.tmp \
		-nopad -noappend -root-owned -be
	-rm -rf $(KDIR)/wndr3700
	mkimage -A mips -O linux -T filesystem -C none \
		-a 0xbf070000 -e 0xbf070000 \
		-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
		-d $(KDIR)/vmlinux-$(2).uImage.squashfs.tmp \
		$(KDIR)/vmlinux-$(2).uImage.squashfs.tmp2
	$(STAGING_DIR_HOST)/bin/wndr3700 \
		$(KDIR)/vmlinux-$(2).uImage.squashfs.tmp2 \
		$(KDIR)/vmlinux-$(2).uImage.squashfs \
		$(5)
	-rm -f $(KDIR)/vmlinux-$(2).uImage.squashfs.tmp*
	( \
		dd if=$(KDIR)/vmlinux-$(2).uImage.squashfs bs=1024k conv=sync; \
		dd if=$(KDIR)/root.$(1) bs=64k; \
	) > $(call imgname,$(1),$(2))-sysupgrade.bin
	for r in $(7) ; do \
		[ -n "$$$$r" ] && dashr="-$$$$r" || dashr= ; \
		$(STAGING_DIR_HOST)/bin/mkdniimg \
			-B $(6) -v OpenWrt.$(REVISION) -r "$$$$r" $(8) \
			-i $(call imgname,$(1),$(2))-sysupgrade.bin \
			-o $(call imgname,$(1),$(2))-factory$$$$dashr.img; \
	done
endef

define Image/Build/Netgear/initramfs
	$(call PatchKernelLzma,$(2),$(3) $($(4)))
	$(call MkImageLzma,$(KDIR)/vmlinux-$(2).bin.lzma,$(KDIR)/vmlinux-$(2).uImage)
	$(STAGING_DIR_HOST)/bin/wndr3700 \
		$(KDIR)/vmlinux-$(2).uImage \
		$(call imgname,$(1),$(2))-uImage.bin \
		$(5)
endef

ifdef CONFIG_PACKAGE_uboot-ar71xx-nbg460n_550n_550nh
  define Image/Build/ZyXEL
	$(call PatchKernelLzma,$(2),$(3))
	if [ `stat -c%s "$(KDIR)/vmlinux-$(2).bin.lzma"` -gt 917504 ]; then \
		echo "Warning: $(KDIR)/vmlinux-$(2).bin.lzma is too big"; \
	else if [ `stat -c%s $(KDIR)/root.$(1)` -gt 2752512 ]; then \
		echo "Warning: $(KDIR)/root.$(1) is too big"; \
	else if [ ! -f $(BIN_DIR)/$(IMG_PREFIX)-$(2)-u-boot.bin ]; then \
		echo "Warning: $(IMG_PREFIX)-$(2)-u-boot.bin not found"; \
	else \
		mkimage -A mips -O linux -T kernel -a 0x80060000 -C lzma -e \
			0x80060000 \
			-n 'MIPS OpenWrt Linux-$(LINUX_VERSION)' \
			-d $(KDIR)/vmlinux-$(2).bin.lzma $(KDIR)/vmlinux-$(2).uImage; \
		( \
			dd if=$(KDIR)/vmlinux-$(2).uImage bs=896k conv=sync; \
			dd if=$(KDIR)/root.$(1); \
		) > $(call imgname,$(1),$(2))-sysupgrade.bin; \
		$(STAGING_DIR_HOST)/bin/mkzynfw \
			-B $(4) \
			-b $(BIN_DIR)/$(IMG_PREFIX)-$(2)-u-boot.bin \
			-r $(call imgname,$(1),$(2))-sysupgrade.bin:0x10000 \
			-o $(call imgname,$(1),$(2))-factory.bin; \
	fi; fi; fi
  endef
endif

define Image/Build/Zcomax
	$(call PatchKernelLzma,$(2),$(3))
	$(call MkImageLzma,$(KDIR)/vmlinux-$(2).bin.lzma,$(KDIR)/vmlinux-$(2).uImage.bin)
	-$(STAGING_DIR_HOST)/bin/mkzcfw \
		-B $(2) \
		-k $(KDIR)/vmlinux-$(2).uImage.bin \
		-r $(BIN_DIR)/$(IMG_PREFIX)-root.$(1) \
		-o $(call imgname,$(1),$(2))-factory.img
	( \
		dd if=$(BIN_DIR)/$(IMG_PREFIX)-root.$(1) bs=6208k count=1 conv=sync; \
		dd if=$(KDIR)/vmlinux-$(2).uImage.bin bs=1472k count=1; \
	) > $(call imgname,$(1),$(2))-sysupgrade.bin
endef

define Image/Build/Template/initramfs/initramfs
	$(call Image/Build/$(1)/initramfs,initramfs,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/all/squashfs
	$(call Image/Build/$(1),squashfs,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/all/jffs2-64k
	$(call Image/Build/$(1),jffs2-64k,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/all/jffs2-128k
	$(call Image/Build/$(1),jffs2-128k,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/squashfs-only/squashfs
	$(call Image/Build/$(1),squashfs,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/4k/squashfs
	$(call Image/Build/$(1),squashfs-4k,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/64k/squashfs
	$(call Image/Build/$(1),squashfs-64k,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/64k/jffs2-64k
	$(call Image/Build/$(1),jffs2-64k,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/128k/squashfs
	$(call Image/Build/$(1),squashfs,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef

define Image/Build/Template/128k/jffs2-128k
	$(call Image/Build/$(1),jffs2-128k,$(2),$(3),$(4),$(5),$(6),$(7),$(8))
endef


ap121_mtdlayout_2M=mtdparts=spi0.0:64k(u-boot)ro,1216k(rootfs),704k(kernel),64k(art)ro,1920k@0x10000(firmware)
ap121_mtdlayout_4M=mtdparts=spi0.0:256k(u-boot)ro,64k(u-boot-env)ro,2752k(rootfs),896k(kernel),64k(nvram),64k(art)ro,3648k@0x50000(firmware)

ap121_cmdline=board=AP121 console=ttyATH0,115200
define Image/Build/Profile/AP121
	$(call Image/Build/Template/$(fs_4k)/$(1),AP121,ap121-2M,$(ap121_cmdline),ap121_mtdlayout_2M,720896,1245184)
	$(call Image/Build/Template/$(fs_64k)/$(1),AP121,ap121-4M,$(ap121_cmdline),ap121_mtdlayout_4M,917504,2818048)
endef

ap121_mini_cmdline=board=AP121-MINI console=ttyATH0,115200
define Image/Build/Profile/AP121MINI
	$(call Image/Build/Template/$(fs_64k)/$(1),AP121,ap121-mini,$(ap121_mini_cmdline),ap121_mtdlayout_4M,917504,2818048)
endef

ap81_cmdline=board=AP81 console=ttyS0,115200
define Image/Build/Profile/AP81
	$(call Image/Build/Template/$(fs_64k)/$(1),AP83,ap81,$(ap81_cmdline))
endef

ap83_cmdline=board=AP83 console=ttyS0,115200
define Image/Build/Profile/AP83
	$(call Image/Build/Template/$(fs_64k)/$(1),AP83,ap83,$(ap83_cmdline))
endef

db120_cmdline=board=DB120 console=ttyS0,115200
define Image/Build/Profile/DB120
	$(call Image/Build/Template/$(fs_64k)/$(1),DB120,db120,$(db120_cmdline))
endef

pb42_cmdline=board=PB42 console=ttyS0,115200
define Image/Build/Profile/PB42
	$(call Image/Build/Template/$(fs_64k)/$(1),PB4X,pb42,$(pb42_cmdline))
endef

pb44_cmdline=board=PB44 console=ttyS0,115200
define Image/Build/Profile/PB44
	$(call Image/Build/Template/$(fs_64k)/$(1),PB4X,pb44,$(pb44_cmdline))
endef

pb92_cmdline=board=PB92 console=ttyS0,115200
define Image/Build/Profile/PB92
	$(call Image/Build/Template/$(fs_64k)/$(1),PB92,pb92,$(pb92_cmdline))
endef

define Image/Build/Profile/WP543
	$(call Image/Build/Template/$(fs_64k)/$(1),MyLoader,wp543,0x200000,2M)
	$(call Image/Build/Template/$(fs_64k)/$(1),MyLoader,wp543,0x400000,4M)
	$(call Image/Build/Template/$(fs_64k)/$(1),MyLoader,wp543,0x800000,8M)
	$(call Image/Build/Template/$(fs_64k)/$(1),MyLoader,wp543,0x1000000,16M)
endef

dir600a1_cmdline=board=DIR-600-A1 console=ttyS0,115200
define Image/Build/Profile/DIR600A1
	$(call Image/Build/Template/$(fs_64k)/$(1),Cameo7240,dir-600-a1,$(dir600a1_cmdline),"AP91-AR7240-RT-090223-00")
endef

eap7660d_cmdline=board=EAP7660D console=ttyS0,115200
define Image/Build/Profile/EAP7660D
	$(call Image/Build/Template/$(fs_128k)/$(1),PB4X,eap7660d,$(eap7660d_cmdline))
endef

ja76pf_cmdline=board=JA76PF console=ttyS0,115200
define Image/Build/Profile/JA76PF
	$(call Image/Build/Template/$(fs_64k)/$(1),PB4X,ja76pf,$(ja76pf_cmdline))
endef

jwap003_cmdline=board=JWAP003 console=ttyS0,115200
define Image/Build/Profile/JWAP003
	$(call Image/Build/Template/$(fs_64k)/$(1),PB4X,jwap003,$(jwap003_cmdline))
endef

fr54rtr_cmdline=board=DIR-600-A1 console=ttyS0,115200
define Image/Build/Profile/FR54RTR
	$(call Image/Build/Template/$(fs_64k)/$(1),Cameo7240,fr-54rtr,$(fr54rtr_cmdline),"AP91-AR7240-RT-090223-01")
endef

dir615c1_cmdline=board=DIR-615-C1 console=ttyS0,115200
define Image/Build/Profile/DIR615C1
	$(call Image/Build/Template/$(fs_squash)/$(1),Cameo,dir-615-c1,$(dir615c1_cmdline),"AP81-AR9130-RT-070614-02")
endef

tew632brp_cmdline=board=TEW-632BRP console=ttyS0,115200
define Image/Build/Profile/TEW632BRP
	$(call Image/Build/Template/$(fs_squash)/$(1),Cameo,tew-632brp,$(tew632brp_cmdline),"AP81-AR9130-RT-070614-00")
endef

tew652brp_cmdline=board=TEW-632BRP console=ttyS0,115200
define Image/Build/Profile/TEW652BRP
	$(call Image/Build/Template/$(fs_squash)/$(1),Cameo,tew-652brp,$(tew652brp_cmdline),"AP81-AR9130-RT-080609-05")
	$(call Image/Build/Template/$(fs_squash)/$(1),Cameo,tew-652brp-recovery,$(tew652brp_cmdline),"AP81-AR9130-RT-070614-02")
endef

a05rbw300n_cmdline=board=TEW-632BRP console=ttyS0,115200
define Image/Build/Profile/A02RBW300N
	$(call Image/Build/Template/$(fs_squash)/$(1),Cameo,a02-rb-w300n,$(a05rbw300n_cmdline),"AP81-AR9130-RT-070614-03")
endef

ubntrs_cmdline=board=UBNT-RS console=ttyS0,115200
define Image/Build/Profile/UBNTRS
	$(call Image/Build/Template/$(fs_64k)/$(1),UBNT,ubnt-rs,$(ubntrs_cmdline),RS,RSx,ar7100)
endef

ubntrspro_cmdline=board=UBNT-RSPRO console=ttyS0,115200
define Image/Build/Profile/UBNTRSPRO
	$(call Image/Build/Template/$(fs_64k)/$(1),UBNT,ubnt-rspro,$(ubntrspro_cmdline),RSPRO,RSPRO,ar7100pro)
endef

ubntlssr71_cmdline=board=UBNT-LS-SR71 console=ttyS0,115200
define Image/Build/Profile/UBNTLSSR71
	$(call Image/Build/Template/$(fs_64k)/$(1),UBNT,ubnt-ls-sr71,$(ubntlssr71_cmdline),LS-SR71,LS-SR71,ar7100)
endef

ubntbulletm_cmdline=board=UBNT-BM console=ttyS0,115200
define Image/Build/Profile/UBNTBULLETM
	$(call Image/Build/Template/$(fs_64k)/$(1),UBNTXM,ubnt-bullet-m,$(ubntbulletm_cmdline),XM,UBNTXM,ar7240)
endef

ubntrocketm_cmdline=board=UBNT-RM console=ttyS0,115200
define Image/Build/Profile/UBNTROCKETM
	$(call Image/Build/Template/$(fs_64k)/$(1),UBNTXM,ubnt-rocket-m,$(ubntrocketm_cmdline),XM,UBNTXM,ar7240)
endef

ubntnanom_cmdline=board=UBNT-NM console=ttyS0,115200
define Image/Build/Profile/UBNTNANOM
	$(call Image/Build/Template/$(fs_64k)/$(1),UBNTXM,ubnt-nano-m,$(ubntnanom_cmdline),XM,UBNTXM,ar7240)
endef

ubntunifi_cmdline=board=UBNT-XM console=ttyS0,115200
define Image/Build/Profile/UBNTUNIFI
	$(call Image/Build/Template/$(fs_64k)/$(1),UBNTXM,ubnt-unifi,$(ubntunifi_cmdline),XM,UBNTXM,ar7240)
endef

define Image/Build/Profile/UBNT
	$(call Image/Build/Profile/UBNTRS,$(1))
	$(call Image/Build/Profile/UBNTRSPRO,$(1))
	$(call Image/Build/Profile/UBNTLSSR71,$(1))
	$(call Image/Build/Profile/UBNTBULLETM,$(1))
	$(call Image/Build/Profile/UBNTROCKETM,$(1))
	$(call Image/Build/Profile/UBNTNANOM,$(1))
	$(call Image/Build/Profile/UBNTUNIFI,$(1))
endef

mzkw04nu_cmdline=board=MZK-W04NU console=ttyS0,115200
define Image/Build/Profile/MZKW04NU
	$(call Image/Build/Template/$(fs_64k)/$(1),Planex,mzk-w04nu,$(mzkw04nu_cmdline))
endef

mzkw300nh_cmdline=board=MZK-W300NH console=ttyS0,115200
define Image/Build/Profile/MZKW300NH
	$(call Image/Build/Template/$(fs_64k)/$(1),Planex,mzk-w300nh,$(mzkw300nh_cmdline))
endef

nbg460n_cmdline=board=NBG460N console=ttyS0,115200
define Image/Build/Profile/NBG_460N_550N_550NH
	$(call Image/Build/Template/$(fs_64k)/$(1),ZyXEL,nbg460n_550n_550nh,$(nbg460n_cmdline),NBG-460N)
endef

tlmr3220_cmdline=board=TL-MR3220 console=ttyS0,115200
define Image/Build/Profile/TLMR3220V1
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-mr3220-v1,$(tlmr3220_cmdline),TL-MR3220v1)
endef

tlmr3420_cmdline=board=TL-MR3420 console=ttyS0,115200
define Image/Build/Profile/TLMR3420V1
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-mr3420-v1,$(tlmr3420_cmdline),TL-MR3420v1)
endef

tlwa901nd_cmdline=board=TL-WA901ND console=ttyS0,115200
define Image/Build/Profile/TLWA901NDV1
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-wa901nd-v1,$(tlwa901nd_cmdline),TL-WA901NDv1)
endef

tlwa901ndv2_cmdline=board=TL-WA901ND-v2 console=ttyS0,115200
define Image/Build/Profile/TLWA901NDV2
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-wa901nd-v2,$(tlwa901ndv2_cmdline),TL-WA901NDv2)
endef

tlwe741ndv1_cmdline=board=TL-WR741ND console=ttyS0,115200
define Image/Build/Profile/TLWR741NDV1
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-wr741nd-v1,$(tlwe741ndv1_cmdline),TL-WR741NDv1)
endef

tlwe740nv1_cmdline=board=TL-WR741ND console=ttyS0,115200
define Image/Build/Profile/TLWR740NV1
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-wr740n-v1,$(tlwe740nv1_cmdline),TL-WR740Nv1)
endef

tlwr841nv15_cmdline=board=TL-WR841N-v1.5 console=ttyS0,115200
define Image/Build/Profile/TLWR841NV15
	$(call Image/Build/Template/$(fs_squash)/$(1),TPLINK,tl-wr841n-v1.5,$(tlwr841nv15_cmdline),TL-WR841Nv1.5)
endef

tlwr841ndv3_cmdline=board=TL-WR941ND console=ttyS0,115200
define Image/Build/Profile/TLWR841NDV3
	$(call Image/Build/Template/$(fs_squash)/$(1),TPLINK,tl-wr841nd-v3,$(tlwr841ndv3_cmdline),TL-WR841NDv3)
endef

tlwr841ndv5_cmdline=board=TL-WR741ND console=ttyS0,115200
define Image/Build/Profile/TLWR841NDV5
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-wr841nd-v5,$(tlwr841ndv5_cmdline),TL-WR841NDv5)
endef

tlwr841ndv7_cmdline=board=TL-WR741ND console=ttyS0,115200
define Image/Build/Profile/TLWR841NDV7
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-wr841nd-v7,$(tlwr841ndv7_cmdline),TL-WR841NDv7)
endef

tlwr941ndv2_cmdline=board=TL-WR941ND console=ttyS0,115200
define Image/Build/Profile/TLWR941NDV2
	$(call Image/Build/Template/$(fs_squash)/$(1),TPLINK,tl-wr941nd-v2,$(tlwr941ndv2_cmdline),TL-WR941NDv2)
endef

tlwr941ndv3_cmdline=board=TL-WR941ND console=ttyS0,115200
define Image/Build/Profile/TLWR941NDV3
	$(call Image/Build/Template/$(fs_squash)/$(1),TPLINK,tl-wr941nd-v3,$(tlwr941ndv3_cmdline),TL-WR941NDv2)
endef

tlwr941ndv4_cmdline=board=TL-WR741ND console=ttyS0,115200
define Image/Build/Profile/TLWR941NDV4
	$(call Image/Build/Template/$(fs_64k)/$(1),TPLINK,tl-wr941nd-v4,$(tlwr941ndv4_cmdline),TL-WR941NDv4)
endef

tlwr1043nd_cmdline=board=TL-WR1043ND console=ttyS0,115200
define Image/Build/Profile/TLWR1043NDV1
	$(call Image/Build/Template/$(fs_squash)/$(1),TPLINK,tl-wr1043nd-v1,$(tlwr1043nd_cmdline),TL-WR1043NDv1)
endef

wndr3700_cmdline=board=WNDR3700 console=ttyS0,115200
wndr3700_mtdlayout=mtdparts=spi0.0:320k(u-boot)ro,128k(u-boot-env)ro,1024k(kernel),6656k(rootfs),64k(art)ro,7680k@0x70000(firmware)
wndr3700v2_cmdline=board=WNDR3700v2 console=ttyS0,115200
wndr3700v2_mtdlayout=mtdparts=spi0.0:320k(u-boot)ro,128k(u-boot-env)ro,1024k(kernel),14848k(rootfs),64k(art)ro,15872k@0x70000(firmware)
define Image/Build/Profile/WNDR3700
	$(call Image/Build/Template/$(fs_64k)/$(1),Netgear,wndr3700,$(wndr3700_cmdline),wndr3700_mtdlayout,3700,WNDR3700,"" NA,)
	$(call Image/Build/Template/$(fs_64k)/$(1),Netgear,wndr3700v2,$(wndr3700v2_cmdline),wndr3700v2_mtdlayout,3701,WNDR3700v2,"",-H 29763654+16+64)
endef

ap96_cmdline=board=AP96 console=ttyS0,115200
define Image/Build/Profile/AP96
	$(call Image/Build/Template/$(fs_64k)/$(1),AP83,ap96,$(ap96_cmdline))
endef

wr400n_cmdline=board=WRT400N console=ttyS0,115200
define Image/Build/Profile/WRT400N
	$(call Image/Build/Template/$(fs_64k)/$(1),WRT400N,wrt400n,$(wr400n_cmdline))
endef

dir825b1_cmdline=board=DIR-825-B1 console=ttyS0,115200
define Image/Build/Profile/DIR825B1
	$(call Image/Build/Template/$(fs_64k)/$(1),DIR825B1,dir-825-b1,$(dir825b1_cmdline))
endef

wrt160nl_cmdline=board=WRT160NL console=ttyS0,115200
define Image/Build/Profile/WRT160NL
	$(call Image/Build/Template/$(fs_64k)/$(1),CyberTAN,wrt160nl,$(wrt160nl_cmdline),1.00.01)
endef

wzrhpg300nh_cmdline=board=WZR-HP-G300NH console=ttyS0,115200
wzrhpg301nh_cmdline=board=WZR-HP-G301NH console=ttyS0,115200
define Image/Build/Profile/WZRHPG30XNH
	$(call Image/Build/Template/$(fs_128k)/$(1),WZRHPG30XNH,wzr-hp-g300nh,$(wzrhpg300nh_cmdline),WZR-HP-G300NH)
	$(call Image/Build/Template/$(fs_128k)/$(1),WZRHPG30XNH,wzr-hp-g301nh,$(wzrhpg301nh_cmdline),WZR-HP-G301NH)
endef

wzrhpag300h_cmdline=board=WZR-HP-AG300H console=ttyS0,115200
define Image/Build/Profile/WZRHPAG300H
	$(call Image/Build/Template/$(fs_128k)/$(1),WZRHPG30XNH,wzr-hp-ag300h,$(wzrhpag300h_cmdline),WZR-HP-AG300H)
endef

zcn1523h28_cmdline=board=ZCN-1523H-2 console=ttyS0,115200
define Image/Build/Profile/ZCN1523H28
	$(call Image/Build/Template/$(fs_64k)/$(1),Zcomax,zcn-1523h-2-8,$(zcn1523h28_cmdline))
endef

zcn1523h516_cmdline=board=ZCN-1523H-5 console=ttyS0,115200
define Image/Build/Profile/ZCN1523H516
	$(call Image/Build/Template/$(fs_64k)/$(1),Zcomax,zcn-1523h-5-16,$(zcn1523h516_cmdline))
endef

define Image/Build/Profile/Default
	$(call Image/Build/Profile/AP121,$(1))
	$(call Image/Build/Profile/AP121MINI,$(1))
	$(call Image/Build/Profile/AP81,$(1))
	$(call Image/Build/Profile/AP83,$(1))
	$(call Image/Build/Profile/A02RBW300N,$(1))
	$(call Image/Build/Profile/DB120,$(1))
	$(call Image/Build/Profile/DIR600A1,$(1))
	$(call Image/Build/Profile/DIR615C1,$(1))
	$(call Image/Build/Profile/DIR825B1,$(1))
	$(call Image/Build/Profile/EAP7660D,$(1))
	$(call Image/Build/Profile/FR54RTR,$(1))
	$(call Image/Build/Profile/JA76PF,$(1))
	$(call Image/Build/Profile/JWAP003,$(1))
	$(call Image/Build/Profile/MZKW04NU,$(1))
	$(call Image/Build/Profile/MZKW300NH,$(1))
	$(call Image/Build/Profile/NBG_460N_550N_550NH,$(1))
	$(call Image/Build/Profile/PB42,$(1))
	$(call Image/Build/Profile/PB44,$(1))
	$(call Image/Build/Profile/PB92,$(1))
	$(call Image/Build/Profile/TEW632BRP,$(1))
	$(call Image/Build/Profile/TEW652BRP,$(1))
	$(call Image/Build/Profile/TLMR3220V1,$(1))
	$(call Image/Build/Profile/TLMR3420V1,$(1))
	$(call Image/Build/Profile/TLWA901NDV1,$(1))
	$(call Image/Build/Profile/TLWA901NDV2,$(1))
	$(call Image/Build/Profile/TLWR741NDV1,$(1))
	$(call Image/Build/Profile/TLWR740NV1,$(1))
	$(call Image/Build/Profile/TLWR841NV15,$(1))
	$(call Image/Build/Profile/TLWR841NDV3,$(1))
	$(call Image/Build/Profile/TLWR841NDV5,$(1))
	$(call Image/Build/Profile/TLWR841NDV7,$(1))
	$(call Image/Build/Profile/TLWR941NDV2,$(1))
	$(call Image/Build/Profile/TLWR941NDV3,$(1))
	$(call Image/Build/Profile/TLWR941NDV4,$(1))
	$(call Image/Build/Profile/TLWR1043NDV1,$(1))
	$(call Image/Build/Profile/UBNT,$(1))
	$(call Image/Build/Profile/WP543,$(1))
	$(call Image/Build/Profile/WNDR3700,$(1))
	$(call Image/Build/Profile/AP96,$(1))
	$(call Image/Build/Profile/WRT400N,$(1))
	$(call Image/Build/Profile/WRT160NL,$(1))
	$(call Image/Build/Profile/WZRHPG30XNH,$(1))
	$(call Image/Build/Profile/WZRHPAG300H,$(1))
	$(call Image/Build/Profile/ZCN1523H28,$(1))
	$(call Image/Build/Profile/ZCN1523H516,$(1))
endef

define Image/Build/Profile/Minimal
	$(call Image/Build/Profile/Default,$(1))
endef

define Image/Build/Profile/Madwifi
	$(call Image/Build/Profile/EAP7660D,$(1))
	$(call Image/Build/Profile/UBNTRS,$(1))
	$(call Image/Build/Profile/UBNTRSPRO,$(1))
	$(call Image/Build/Profile/UBNTLSSR71,$(1))
	$(call Image/Build/Profile/WP543,$(1))
endef

define Image/Build/squashfs
	cp $(KDIR)/root.squashfs $(KDIR)/root.squashfs-4k
	$(STAGING_DIR_HOST)/bin/padjffs2 $(KDIR)/root.squashfs-4k 4
	cp $(KDIR)/root.squashfs-4k $(BIN_DIR)/$(IMG_PREFIX)-root.squashfs-4k
	cp $(KDIR)/root.squashfs $(KDIR)/root.squashfs-64k
	$(STAGING_DIR_HOST)/bin/padjffs2 $(KDIR)/root.squashfs-64k 4 8 64
	cp $(KDIR)/root.squashfs-64k $(BIN_DIR)/$(IMG_PREFIX)-root.squashfs-64k
	$(call prepare_generic_squashfs,$(KDIR)/root.squashfs)
endef

define Image/Build/Initramfs
	$(call Image/Build/Profile/$(PROFILE),initramfs)
endef

define Image/Build
	$(call Image/Build/$(1))
	dd if=$(KDIR)/root.$(1) of=$(BIN_DIR)/$(IMG_PREFIX)-root.$(1) bs=128k conv=sync

	$(call Image/Build/Profile/$(PROFILE),$(1))
endef

$(eval $(call BuildImage))
