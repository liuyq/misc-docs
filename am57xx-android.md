# Bringing up Android Oreo and OPTEE on X15/AM57xx High Security
Based on Igor instructions with patches / steps added for optee.

### TI linux kernel v4.4
Alternatively you can clone from
$git clone --depth 1 -b ti-ion-sdp git@github.com:petegriffin/ti-kernel-android.git
For WIP ION patches for SDP support.

```bash
$ sudo apt-get install liblz4-dev liblz4-tool
$ git clone --depth 1 -b p-ti-android-linux-4.4.y git://git.ti.com/android/kernel.git ti-kernel.git
$ export PATH=/media/xdev/work/reps/android.repo/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin:$PATH
$ export CROSS_COMPILE=arm-linux-androideabi-
$ export ARCH=arm
$ cd ti-kernel.git
$ ./ti_config_fragments/defconfig_builder.sh -t ti_sdk_am57x_android_release
$ make ti_sdk_am57x_android_release_defconfig
$ make -jn zImage am57xx-evm-reva3.dtb am57xx-beagle-x15.dtb
```

### Android
Ensure following patch is included in your android/device/ti/am57xevm
https://github.com/petegriffin/device-ti-am57xevm/commit/3a863ec172bde7fa5392ae3a6af11a9889d7c499
or http://review.omapzoom.org/38821 optee: Integrate optee components.

```bash
$ repo init -u http://git.ti.com/git/android/manifest.git -b oreo-dev
$ repo sync -j20
$ export KERNELDIR=${KERNEL_PATH}
$ source build/envsetup.sh
$ lunch full_am57xevm-userdebug
$ make -j12 TARGET_AM57XEVM_HS_BOARD=true
```

### U-Boot (HS image)

It's mandatory to use Linux arm-eabi GCC 6 toolchain:
For HS board secdev tools need to be installed from TI
and are used as part of the UBoot build.

```bash
$ tar zxvf secdev-dra7xx_lite_4.6.4.tgz
$ export TI_SECURE_DEV_PKG=./secdev-dra7xx_lite_4.6.4
$ git clone git://git.denx.de/u-boot.git u-boot.git
$ git checkout 2017.11
$ export ARCH=arm
$ make am57xx_hs_evm_defconfig
$ make
```

### Create signed images

Gather all the various bootloader, kernel, optee  images in a directory.

```bash
$ mkdir work
$ cd work
$ mkdir -p signed/dtb
cp ${UBOOT}/u-boot-spl_HS_X-LOADER .
cp ${UBOOT}/u-boot.img .
cp ${MYDROID}/out/target/product/am57xevm/ramdisk.img .
cp ${KERNEL}/arch/arm/boot/zImage .
cp ${KERNEL}/arch/arm/boot/dts/am57xx*.dtb .
cp ${MYDROID}/out/target/product/am57xevm/optee/arm-plat-ti-am57xx/core/tee.bin .
```

### Sign the ramdisk, kernel and dtb files

```bash
${TI_SECURE_DEV_PKG}/scripts/ift-image-sign.sh dra7xx zImage signed/zImage
${TI_SECURE_DEV_PKG}/scripts/ift-image-sign.sh dra7xx ramdisk.img signed/ramdisk.img
${TI_SECURE_DEV_PKG}/scripts/ift-image-sign.sh dra7xx tee.bin signed/tee.bin
${TI_SECURE_DEV_PKG}/scripts/ift-image-sign.sh dra7xx am57xx-evm-reva3.dtb signed/dtb/am57xx-evm-reva3.dtb
```

### Example FIT image config file

```bash
/dts-v1/;

/ {
	description = "U-Boot fitImage for Android";
	#address-cells = <1>;

	images {
		kernel@1 {
			description = "Linux kernel";
			data = /incbin/("signed/zImage");
			type = "kernel";
			arch = "arm";
			os = "linux";
			compression = "none";
			load = <0x82000000>;
			entry = <0x82000000>;
		};

		am57xx-evm-reva3.dtb {
			description = "Flattened Device Tree blob";
			data = /incbin/("signed/dtb/am57xx-evm-reva3.dtb");
			type = "flat_dt";
			arch = "arm";
			compression = "none";
		};

		optee {
			description = "OPTEE OS Image";
			data = /incbin/("signed/tee.bin");
			type = "tee";
			arch = "arm";
			compression = "none";
		};

                ramdisk@1 {
                        description = "Android Ramdisk";
                        data = /incbin/("signed/ramdisk.img");
                        type = "ramdisk";
                        arch = "arm";
                        os = "linux";
                        compression = "gzip";
                        load = <0>;
                        entry = <0>;
		};
	};

	configurations {
		default = "am57xx-evm-reva3.dtb";

		am57xx-evm-reva3.dtb {
			description = "Linux kernel, FDT blob, OPTEE OS, RAMDISK";
			kernel = "kernel@1";
			fdt = "am57xx-evm-reva3.dtb";
			ramdisk = "ramdisk@1";
			loadables = "optee";
		};
	};
};
```

### Create the boot FIT image
```bash
mkimage -f fitImage-am57xx.its boot.itb
```

# Flashing

1. Replace old U-Boot with newly compiled
Firstly, flash new U-Boot and reboot the board.

Run in U-boot shell:
```
=> fas 1
```

Run on the host PC:
```bash
$ fastboot flash xloader MLO
$ fastboot flash bootloader u-boot.img
$ fastboot erase environment
$ fastboot flash boot boot.itb
$ fastboot reboot
```

2. After booting new U-boot, create new partition table for Android
Run in U-boot shell:
```
=> env default -f -a
=> setenv partitions $partitions_android
=> env save
=> fas 1 // fas=fastboot, 1 - USB controller number (OTG)
```

Run on the host PC:
```bash
$ fastboot oem format # create new partition table, based on description in partitions variable.
```

Run in U-boot shell:
```
=> part list mmc 1 // mmc 1 = eMMC
```

3. Flash other partitions
Run on the host PC:
```bash
$ fastboot flash xloader MLO
$ fastboot flash bootloader u-boot.img
$ fastboot erase environment
$ fastboot flash boot boot.itb
$ sudo fastboot reboot
$ fastboot flash cache cache.img
$ fastboot flash system system.img
$ fastboot flash userdata userdata.img
$ fastboot flash recovery recovery.img
$ fastboot flash vendor vendor.img
$ fastboot reboot
```
4. Check of SGX driver is loaded correctly (it's vital for bringing up GUI on TI boards):
Verify Android SGX boot by checking line in dmesg on the target board:
```
[    6.451230] PVR_K: UM DDK-(4001660) and KM DDK-(4001660) match. [ OK ]
```

5. If it doesn't load, rebuild SGX in Android dir and push it to `/vendor/lib/modules/pvrsrvkm.ko`

Run on the host PC:
```bash
$ adb root # restarting adbd as root
$ adb remount # remount vendor paritition as rw
$ adb push device/ti/proprietary-open/jacinto6/sgx_src/pvrsrvkm.ko /vendor/lib/modules
device/ti/proprietary-open/jacinto6/sgx_src/pvrsrvkm.ko: 1 file pushed. 5.1 MB/s (288468 bytes in 0.054s)
```

# Contributing to http://review.omapzoom.org/device/ti/am57xevm
Need to use -oHostKeyAlgorithms=+ssh-dss to SSH.
scp -oHostKeyAlgorithms=+ssh-dss -p -P 29418 peter.griffin@review.omapzoom.org:hooks/commit-msg am57xevm/.git/hooks/

# Example bootlog
If it doesn't work, maybe this working bootlog will help you!

```bash
U-Boot SPL 2017.01-00455-gdf499e7 (Dec 04 2017 - 17:32:23)
DRA752-HS ES2.0
Trying to boot from MMC2_2
Authentication passed: CERT_U-BOOT-NODT
Authentication passed: CERT_AM57XX-EVM-


U-Boot 2017.01-00455-gdf499e7 (Dec 04 2017 - 17:32:23 +0000)

CPU  : DRA752-HS ES2.0
Model: TI AM572x EVM Rev A3
Board: AM572x EVM REV A.30
DRAM:  2 GiB
MMC:   OMAP SD/MMC: 0, OMAP SD/MMC: 1
SCSI:  SATA link 0 timeout.
AHCI 0001.0300 32 slots 1 ports 3 Gbps 0x1 impl SATA mode
flags: 64bit ncq stag pm led clo only pmp pio slum part ccc apst 
scanning bus for devices...
Found 0 device(s).
Net:   
Warning: ethernet@48484000 using MAC address from ROM
eth0: ethernet@48484000
Hit any key to stop autoboot:  0 
MMC: no card present
MMC: no card present
MMC: no card present
MMC: no card present
Trying to boot Linux from eMMC ...
switch to partitions #0, OK
mmc1(part 0) is current device
SD/MMC found on device 1
Failed to mount ext2 filesystem...
** Unrecognized filesystem type **
Trying to boot Android from eMMC ...
switch to partitions #0, OK
mmc1(part 0) is current device

MMC read: dev # 1, block # 5376, count 512 ... 512 blocks read: OK

MMC read: dev # 1, block # 121376, count 20480 ... 20480 blocks read: OK
## Loading kernel from FIT Image at 87000000 ...
   Using 'am57xx-evm-reva3.dtb' configuration
   Trying 'kernel@1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x870000cc
     Data Size:    7938568 Bytes = 7.6 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x82000000
     Entry Point:  0x82000000
   Verifying Hash Integrity ... OK
Authentication passed: CERT_ZIMAGE
## Loading ramdisk from FIT Image at 87000000 ...
   Using 'am57xx-evm-reva3.dtb' configuration
   Trying 'ramdisk@1' ramdisk subimage
     Description:  Android Ramdisk
     Type:         RAMDisk Image
     Compression:  gzip compressed
     Data Start:   0x877db170
     Data Size:    1814996 Bytes = 1.7 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x00000000
     Entry Point:  0x00000000
   Verifying Hash Integrity ... OK
Authentication passed: CERT_RAMDISK
## Flattened Device Tree blob at 88000000
   Booting using the fdt blob at 0x88000000
## Loading loadables from FIT Image at 87000000 ...
   Trying 'optee' loadables subimage
     Description:  OPTEE OS Image
     Type:         Trusted Execution Environment Image
     Compression:  uncompressed
     Data Start:   0x877ab47c
     Data Size:    195716 Bytes = 191.1 KiB
   Verifying Hash Integrity ... OK
Authentication passed: CERT_TEE
TEE_LOAD_MASTER Done
TEE_LOAD_SLAVE Done
   Loading Kernel Image ... OK
   Loading Ramdisk to 8fe44000, end 8ffff0bc ... OK
   Loading Device Tree to 8fe28000, end 8fe43f5d ... OK
Using machid 0xfe6 from environment

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Initializing cgroup subsys cpuacct
[    0.000000] Linux version 4.4.91-11645-gc39f2de (griffinp@r710) (gcc version 4.9.x 20150123 (prerelease) (GCC) ) #3 SMP PREEMPT Thu J8
[    0.000000] CPU: ARMv7 Processor [412fc0f2] revision 2 (ARMv7), cr=30c5387d
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, PIPT instruction cache
[    0.000000] Machine model: TI AM572x EVM Rev A3
[    0.000000] Reserved memory: created CMA memory pool at 0x0000000095800000, size 56 MiB
[    0.000000] Reserved memory: initialized node ipu2_cma@95800000, compatible id shared-dma-pool
[    0.000000] Reserved memory: created CMA memory pool at 0x0000000099000000, size 64 MiB
[    0.000000] Reserved memory: initialized node dsp1_cma@99000000, compatible id shared-dma-pool
[    0.000000] Reserved memory: created CMA memory pool at 0x000000009d000000, size 32 MiB
[    0.000000] Reserved memory: initialized node ipu1_cma@9d000000, compatible id shared-dma-pool
[    0.000000] Reserved memory: created CMA memory pool at 0x000000009f000000, size 8 MiB
[    0.000000] Reserved memory: initialized node dsp2_cma@9f000000, compatible id shared-dma-pool
[    0.000000] cma: Reserved 24 MiB at 0x00000000fe400000
[    0.000000] Forcing write-allocate cache policy for SMP
[    0.000000] Memory policy: Data cache writealloc
[    0.000000] OMAP4: Map 0x00000000ffd00000 to fe600000 for dram barrier
[    0.000000] DRA752 ES2.0
[    0.000000] PERCPU: Embedded 13 pages/cpu @eed2c000 s24256 r8192 d20800 u53248
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 513600
[    0.000000] Kernel command line: root=/dev/mmcblk0p2 rw init=/init rootfstype=ext4 rootwait drm.rnodes=1 androidboot.selinux=permissid
[    0.000000] PID hash table entries: 4096 (order: 2, 16384 bytes)
[    0.000000] Dentry cache hash table entries: 131072 (order: 7, 524288 bytes)
[    0.000000] Inode-cache hash table entries: 65536 (order: 6, 262144 bytes)
[    0.000000] Memory: 1833676K/2061312K available (10240K kernel code, 650K rwdata, 3148K rodata, 2048K init, 476K bss, 39220K reserved)
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vector  : 0xffff0000 - 0xffff1000   (   4 kB)
[    0.000000]     fixmap  : 0xffc00000 - 0xfff00000   (3072 kB)
[    0.000000]     vmalloc : 0xf0800000 - 0xff800000   ( 240 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xf0000000   ( 768 MB)
[    0.000000]     pkmap   : 0xbfe00000 - 0xc0000000   (   2 MB)
[    0.000000]     modules : 0xbf000000 - 0xbfe00000   (  14 MB)
[    0.000000]       .text : 0xc0008000 - 0xc0c00000   (12256 kB)
[    0.000000]       .init : 0xc1000000 - 0xc1200000   (2048 kB)
[    0.000000]       .data : 0xc1200000 - 0xc12a2ba4   ( 651 kB)
[    0.000000]        .bss : 0xc12a2ba4 - 0xc1319d68   ( 477 kB)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] Preemptible hierarchical RCU implementation.
[    0.000000]  Build-time adjustment of leaf fanout to 32.
[    0.000000] NR_IRQS:16 nr_irqs:16 16
[    0.000000] ti_dt_clocks_register: failed to lookup clock node gmac_gmii_ref_clk_div
[    0.000000] OMAP clockevent source: timer1 at 32786 Hz
[    0.000000] Architected cp15 timer(s) running at 6.14MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x16af5adb9, max_idle_ns: 440795202250 ns
[    0.000004] sched_clock: 56 bits at 6MHz, resolution 162ns, wraps every 4398046511023ns
[    0.000015] Switching to timer-based delay loop, resolution 162ns
[    0.000305] clocksource: 32k_counter: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 58327039986419 ns
[    0.000312] OMAP clocksource: 32k_counter at 32768 Hz
[    0.000679] Calibrating delay loop (skipped), value calculated using timer frequency.. 12.29 BogoMIPS (lpj=61475)
[    0.000694] pid_max: default: 32768 minimum: 301
[    0.000767] Security Framework initialized
[    0.000779] SELinux:  Initializing.
[    0.000855] Mount-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.000865] Mountpoint-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.001438] Initializing cgroup subsys io
[    0.001455] Initializing cgroup subsys memory
[    0.001479] Initializing cgroup subsys devices
[    0.001492] Initializing cgroup subsys freezer
[    0.001503] Initializing cgroup subsys perf_event
[    0.001515] Initializing cgroup subsys pids
[    0.001553] CPU: Testing write buffer coherency: ok
[    0.001792] /cpus/cpu@0 missing clock-frequency property
[    0.001809] /cpus/cpu@1 missing clock-frequency property
[    0.001819] CPU0: update cpu_capacity 1024
[    0.001827] CPU0: thread -1, cpu 0, socket 0, mpidr 80000000
[    0.001873] Setting up static identity map for 0x80200000 - 0x80200060
[    0.080205] CPU1: update cpu_capacity 1024
[    0.080210] CPU1: thread -1, cpu 1, socket 0, mpidr 80000001
[    0.080283] Brought up 2 CPUs
[    0.080298] SMP: Total of 2 processors activated (24.59 BogoMIPS).
[    0.080305] CPU: All CPU(s) started in SVC mode.
[    0.080337] CPU1: update max cpu_capacity 1024
[    0.090215] CPU1: update max cpu_capacity 1024
[    0.108512] VFP support v0.3: implementor 41 architecture 4 part 30 variant f rev 0
[    0.109440] omap_hwmod: l3_main_2 using broken dt data from ocp
[    0.217977] omap_hwmod: dcan1: _wait_target_disable failed
[    0.316459] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.316480] futex hash table entries: 512 (order: 3, 32768 bytes)
[    0.320095] pinctrl core: initialized pinctrl subsystem
[    0.321072] NET: Registered protocol family 16
[    0.322278] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.350302] cpuidle: using governor ladder
[    0.380333] cpuidle: using governor menu
[    0.389370] OMAP GPIO hardware version 0.1
[    0.396020] irq: no irq domain found for /ocp/l4@4a000000/scm@2000/pinmux@1400 !
[    0.420771] hw-breakpoint: found 5 (+1 reserved) breakpoint and 4 watchpoint registers.
[    0.420781] hw-breakpoint: maximum watchpoint size is 8 bytes.
[    0.421232] omap4_sram_init:Unable to allocate sram needed to handle errata I688
[    0.421243] omap4_sram_init:Unable to get sram pool needed to handle errata I688
[    0.421892] OMAP DMA hardware revision 0.0
[    0.481203] omap-dma-engine 4a056000.dma-controller: OMAP DMA engine driver (LinkedList1/2/3 supported)
[    0.482320] edma 43300000.edma: memcpy is disabled
[    0.487175] edma 43300000.edma: TI EDMA DMA engine driver
[    0.491256] omap-iommu 40d01000.mmu: 40d01000.mmu registered
[    0.491442] omap-iommu 40d02000.mmu: 40d02000.mmu registered
[    0.491604] omap-iommu 58882000.mmu: 58882000.mmu registered
[    0.491767] omap-iommu 55082000.mmu: 55082000.mmu registered
[    0.492047] omap-iommu 41501000.mmu: 41501000.mmu registered
[    0.492238] omap-iommu 41502000.mmu: 41502000.mmu registered
[    0.494253] SCSI subsystem initialized
[    0.494486] usbcore: registered new interface driver usbfs
[    0.494550] usbcore: registered new interface driver hub
[    0.494643] usbcore: registered new device driver usb
[    0.495729] palmas 0-0058: Irq flag is 0x00000008
[    0.512135] palmas 0-0058: Muxing GPIO 2f, PWM 0, LED 0
[    0.594217] omap_i2c 48070000.i2c: bus 0 rev0.12 at 400 kHz
[    0.594751] omap_i2c 48060000.i2c: bus 2 rev0.12 at 400 kHz
[    0.595336] omap_i2c 4807c000.i2c: bus 4 rev0.12 at 400 kHz
[    0.595538] media: Linux media interface: v0.10
[    0.595592] Linux video capture interface: v2.00
[    0.595638] pps_core: LinuxPPS API ver. 1 registered
[    0.595646] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.595671] PTP clock support registered
[    0.595723] EDAC MC: Ver: 3.0.0
[    0.596585] omap-mailbox 4883c000.mailbox: omap mailbox rev 0x400
[    0.596869] omap-mailbox 4883e000.mailbox: omap mailbox rev 0x400
[    0.597157] omap-mailbox 48840000.mailbox: omap mailbox rev 0x400
[    0.597441] omap-mailbox 48842000.mailbox: omap mailbox rev 0x400
[    0.597860] Advanced Linux Sound Architecture Driver Initialized.
[    0.599327] clocksource: Switched to clocksource arch_sys_counter
[    0.641493] NET: Registered protocol family 2
[    0.642084] TCP established hash table entries: 8192 (order: 3, 32768 bytes)
[    0.642148] TCP bind hash table entries: 8192 (order: 4, 65536 bytes)
[    0.642273] TCP: Hash tables configured (established 8192 bind 8192)
[    0.642322] UDP hash table entries: 512 (order: 2, 16384 bytes)
[    0.642353] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes)
[    0.642509] NET: Registered protocol family 1
[    0.643062] Trying to unpack rootfs image as initramfs...
[    0.716152] Freeing initrd memory: 1776K
[    0.717098] hw perfevents: enabled with armv7_cortex_a15 PMU driver, 7 counters available
[    0.719675] audit: initializing netlink subsys (disabled)
[    0.719733] audit: type=2000 audit(0.709:1): initialized
[    0.726976] VFS: Disk quotas dquot_6.6.0
[    0.727140] VFS: Dquot-cache hash table entries: 1024 (order 0, 4096 bytes)
[    0.728207] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.728459] ntfs: driver 2.1.32 [Flags: R/O].
[    0.728784] fuse init (API version 7.23)
[    0.732934] bounce: pool size: 64 pages
[    0.733090] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 247)
[    0.733104] io scheduler noop registered
[    0.733116] io scheduler deadline registered
[    0.733148] io scheduler cfq registered (default)
[    0.738128] pinctrl-single 4a003400.pinmux: 282 pins at pa fc003400 size 1128
[    0.741960] PCI host bridge /ocp/axi@0/pcie_rc@51000000 ranges:
[    0.741975]   No bus range found for /ocp/axi@0/pcie_rc@51000000, using [bus 00-ff]
[    0.742007]    IO 0x20003000..0x20012fff -> 0x00000000
[    0.742028]   MEM 0x20013000..0x2fffffff -> 0x20013000
[    0.772917] dra7-pcie 51000000.pcie_rc: link is not up
[    0.773086] dra7-pcie 51000000.pcie_rc: PCI host bridge to bus 0000:00
[    0.773100] pci_bus 0000:00: root bus resource [bus 00-ff]
[    0.773111] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    0.773121] pci_bus 0000:00: root bus resource [mem 0x20013000-0x2fffffff]
[    0.773526] PCI: bus0: Fast back to back transfers disabled
[    0.773649] PCI: bus1: Fast back to back transfers enabled
[    0.773733] pci 0000:00:00.0: BAR 0: assigned [mem 0x20100000-0x201fffff]
[    0.773749] pci 0000:00:00.0: BAR 1: assigned [mem 0x20020000-0x2002ffff]
[    0.773761] pci 0000:00:00.0: PCI bridge to [bus 01]
[    0.773977] pcieport 0000:00:00.0: Signaling PME through PCIe PME interrupt
[    0.775226] backlight supply power not found, using dummy regulator
[    0.779427] Serial: 8250/16550 driver, 10 ports, IRQ sharing disabled
[    0.781797] console [ttyS2] disabled
[    0.781848] 48020000.serial: ttyS2 at MMIO 0x48020000 (irq = 301, base_baud = 3000000) is a 8250
[    1.793013] console [ttyS2] enabled
[    1.796971] 48422000.serial: ttyS7 at MMIO 0x48422000 (irq = 302, base_baud = 3000000) is a 8250
[    1.806900] [drm] Initialized drm 1.1.0 20060810
[    1.813484] OMAP DSS rev 6.1
[    1.817270] omapdss_dss 58000000.dss: bound 58001000.dispc (ops dispc_component_ops)
[    1.825733] omapdss_dss 58000000.dss: bound 58040000.encoder (ops hdmi5_component_ops)
[    1.925835] brd: module loaded
[    1.994045] loop: module loaded
[    2.000467] libphy: Fixed MDIO Bus: probed
[    2.005141] tun: Universal TUN/TAP device driver, 1.6
[    2.010231] tun: (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
[    2.016518] CAN device driver interface
[    2.069356] davinci_mdio 48485000.mdio: davinci mdio revision 1.6
[    2.075482] davinci_mdio 48485000.mdio: detected phy mask fffffff9
[    2.085844] libphy: 48485000.mdio: probed
[    2.089925] davinci_mdio 48485000.mdio: phy[1]: device 48485000.mdio:01, driver Micrel KSZ9031 Gigabit PHY
[    2.099664] davinci_mdio 48485000.mdio: phy[2]: device 48485000.mdio:02, driver Micrel KSZ9031 Gigabit PHY
[    2.110070] cpsw 48484000.ethernet: Detected MACID = a0:f6:fd:bc:d9:c6
[    2.116724] cpsw 48484000.ethernet: cpts: overflow check period 800
[    2.123731] cpsw 48484000.ethernet: cpsw: Detected MACID = a0:f6:fd:bc:d9:c7
[    2.131415] PPP generic driver version 2.4.2
[    2.135815] PPP BSD Compression module registered
[    2.140566] PPP Deflate Compression module registered
[    2.145652] PPP MPPE Compression module registered
[    2.150485] NET: Registered protocol family 24
[    2.358836] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    2.365427] ehci-pci: EHCI PCI platform driver
[    2.369957] ehci-platform: EHCI generic platform driver
[    2.375511] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    2.381755] ohci-pci: OHCI PCI platform driver
[    2.386270] ohci-platform: OHCI generic platform driver
[    2.392243] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
[    2.397771] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 1
[    2.405714] xhci-hcd xhci-hcd.1.auto: hcc params 0x0220f04c hci version 0x100 quirks 0x00210010
[    2.414509] xhci-hcd xhci-hcd.1.auto: irq 469, io mem 0x48890000
[    2.420740] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
[    2.427562] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.434835] usb usb1: Product: xHCI Host Controller
[    2.439750] usb usb1: Manufacturer: Linux 4.4.91-11645-gc39f2de xhci-hcd
[    2.446481] usb usb1: SerialNumber: xhci-hcd.1.auto
[    2.451843] hub 1-0:1.0: USB hub found
[    2.455641] hub 1-0:1.0: 1 port detected
[    2.459966] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
[    2.465484] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 2
[    2.473272] usb usb2: We don't know the algorithms for LPM for this host, disabling LPM.
[    2.481546] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003
[    2.488368] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.495655] usb usb2: Product: xHCI Host Controller
[    2.500572] usb usb2: Manufacturer: Linux 4.4.91-11645-gc39f2de xhci-hcd
[    2.507302] usb usb2: SerialNumber: xhci-hcd.1.auto
[    2.512674] hub 2-0:1.0: USB hub found
[    2.516472] hub 2-0:1.0: 1 port detected
[    2.520879] usbcore: registered new interface driver usb-storage
[    2.527207] mousedev: PS/2 mouse device common for all mice
[    2.532977] usbcore: registered new interface driver xpad
[    2.538445] usbcore: registered new interface driver usb_acecad
[    2.544470] usbcore: registered new interface driver aiptek
[    2.550132] usbcore: registered new interface driver gtco
[    2.555602] usbcore: registered new interface driver hanwang
[    2.561437] usbcore: registered new interface driver kbtab
[    2.690117] input: pixcir_tangoc as /devices/platform/44000000.ocp/4807c000.i2c/i2c-4/4-005c/input/input0
[    2.700526] i2c /dev entries driver
[    2.704958] vpe 489d0000.vpe: loading firmware vpdma-1b8.bin
[    2.712119] vip 48990000.vip: loading firmware vpdma-1b8.bin
[    2.718185] gspca_main: v2.14.0 registered
[    2.725058] gpio-fan gpio_fan: GPIO fan initialized
[    2.730110] vpe 489d0000.vpe: Device registered as /dev/video0
[    2.736886] tmp102 0-0048: initialized
[    2.740693] vip 48990000.vip: VPDMA firmware loaded
[    2.747886] device-mapper: uevent: version 1.0.3
[    2.752779] device-mapper: ioctl: 4.34.0-ioctl (2015-10-28) initialised: dm-devel@redhat.com
[    2.762480] omap_hsmmc 4809c000.mmc: Got CD GPIO
[    2.767460] vdd_3v3: supplied by regen1
[    2.779354] usb 1-1: new high-speed USB device number 2 using xhci-hcd
[    2.819758] omap_hsmmc 480b4000.mmc: no pinctrl state for sdr25 mode
[    2.826144] omap_hsmmc 480b4000.mmc: no pinctrl state for sdr12 mode
[    2.919757] usb 1-1: New USB device found, idVendor=0451, idProduct=8142
[    2.926494] usb 1-1: New USB device strings: Mfr=0, Product=0, SerialNumber=1
[    2.934966] ledtrig-cpu: registered to indicate activity on CPUs
[    2.941030] usb 1-1: SerialNumber: 940B1869DF41
[    2.946005] hidraw: raw HID events driver (C) Jiri Kosina
[    2.953449] hub 1-1:1.0: USB hub found
[    2.958235] usbcore: registered new interface driver usbhid
[    2.963865] hub 1-1:1.0: 4 ports detected
[    2.967969] usbhid: USB HID core driver
[    2.972062] ashmem: initialized
[    2.976458] omap-rproc 58820000.ipu: assigned reserved memory node ipu1_cma@9d000000
[    2.984483]  remoteproc0: 58820000.ipu is available
[    2.989591]  remoteproc0: Note: remoteproc is still under development and considered experimental.
[    2.999285]  remoteproc0: THE BINARY FORMAT IS NOT YET FINALIZED, and backward compatibility isn't yet guaranteed.
[    3.009827]  remoteproc0: Direct firmware load for dra7-ipu1-fw.xem4 failed with error -2
[    3.018042]  remoteproc0: Falling back to user helper
[    3.023272] omap-rproc 55020000.ipu: assigned reserved memory node ipu2_cma@95800000
[    3.031249]  remoteproc1: 55020000.ipu is available
[    3.036149]  remoteproc1: Note: remoteproc is still under development and considered experimental.
[    3.045250] usb 2-1: new SuperSpeed USB device number 2 using xhci-hcd
[    3.051858]  remoteproc1: THE BINARY FORMAT IS NOT YET FINALIZED, and backward compatibility isn't yet guaranteed.
[    3.062420]  remoteproc1: Direct firmware load for dra7-ipu2-fw.xem4 failed with error -2
[    3.070809] omap-rproc 40800000.dsp: assigned reserved memory node dsp1_cma@99000000
[    3.078626]  remoteproc2: 40800000.dsp is available
[    3.083589]  remoteproc1: Falling back to user helper
[    3.088920] usb 2-1: New USB device found, idVendor=0451, idProduct=8140
[    3.095781] usb 2-1: New USB device strings: Mfr=0, Product=0, SerialNumber=0
[    3.102991]  remoteproc2: Note: remoteproc is still under development and considered experimental.
[    3.112357]  remoteproc2: THE BINARY FORMAT IS NOT YET FINALIZED, and backward compatibility isn't yet guaranteed.
[    3.123112] hub 2-1:1.0: USB hub found
[    3.126939] hub 2-1:1.0: 4 ports detected
[    3.131319] omap-rproc 41000000.dsp: assigned reserved memory node dsp2_cma@9f000000
[    3.139135]  remoteproc3: 41000000.dsp is available
[    3.144546]  remoteproc2: Direct firmware load for dra7-dsp1-fw.xe66 failed with error -2
[    3.152818]  remoteproc2: Falling back to user helper
[    3.158476]  remoteproc3: Note: remoteproc is still under development and considered experimental.
[    3.167543]  remoteproc3: THE BINARY FORMAT IS NOT YET FINALIZED, and backward compatibility isn't yet guaranteed.
[    3.178170]  remoteproc3: Direct firmware load for dra7-dsp2-fw.xe66 failed with error -2
[    3.187029]  remoteproc3: Falling back to user helper
[    3.199708] palmas-usb 48070000.i2c:tps659038@58:tps659038_usb: USB cable is attached
[    3.218368] optee: probing for conduit method from DT.
[    3.223830] optee: initialized driver
[    3.227745] usbcore: registered new interface driver snd-usb-audio
[    3.236530] omap-hdmi-audio omap-hdmi-audio.0.auto: snd-soc-dummy-dai <-> 58040000.encoder mapping ok
[    3.247543] u32 classifier
[    3.250289]     input device check on
[    3.253967]     Actions configured
[    3.257393] Netfilter messages via NETLINK v0.30.
[    3.262911] nf_conntrack version 0.5.0 (16384 buckets, 65536 max)
[    3.269546] ctnetlink v0.93: registering with nfnetlink.
[    3.275459] xt_time: kernel timezone is -0000
[    3.280272] ip_tables: (C) 2000-2006 Netfilter Core Team
[    3.285799] arp_tables: (C) 2002 David S. Miller
[    3.290631] Initializing XFRM netlink socket
[    3.295596] NET: Registered protocol family 10
[    3.311165] mip6: Mobile IPv6
[    3.314174] ip6_tables: (C) 2000-2006 Netfilter Core Team
[    3.320005] sit: IPv6 over IPv4 tunneling driver
[    3.325260] NET: Registered protocol family 17
[    3.329779] NET: Registered protocol family 15
[    3.334287] can: controller area network core (rev 20120528 abi 9)
[    3.340598] NET: Registered protocol family 29
[    3.345107] can: raw protocol (rev 20120528)
[    3.349454] can: broadcast manager protocol (rev 20120528 t)
[    3.355189] can: netlink gateway (rev 20130117) max_hops=1
[    3.361042] NET: Registered protocol family 41
[    3.366013] omap_voltage_late_init: Voltage driver support not added
[    3.373037] Adding alias for supply vdd,cpu0 -> vdd,4a003b20.oppdm
[    3.379283] Adding alias for supply vbb,cpu0 -> vbb,4a003b20.oppdm
[    3.386055] Adding alias for supply vdd,cpu0 -> vdd,4a003b20.oppdm
[    3.392326] Adding alias for supply vbb,cpu0 -> vbb,4a003b20.oppdm
[    3.400083] Power Management for TI OMAP4+ devices.
[    3.405246] Registering SWP/SWPB emulation handler
[    3.410649] registered taskstats version 1
[    3.415590] dmm 4e000000.dmm: workaround for errata i878 in use
[    3.423378] dmm 4e000000.dmm: initialized all PAT entries
[    3.442589] [drm] Supports vblank timestamp caching Rev 2 (21.10.2013).
[    3.449232] [drm] No driver support for vblank timestamp query.
[    3.455211] mmc1: MAN_BKOPS_EN bit is not set
[    3.457397] [drm] Enabling DMM ywrap scrolling
[    3.464095] omapdrm omapdrm.0: fb0: omapdrm frame buffer device
[    3.477695] mmc1: new DDR MMC card at address 0001
[    3.492936] mmcblk0: mmc1:0001 S10004 3.56 GiB 
[    3.497652] mmcblk0boot0: mmc1:0001 S10004 partition 1 4.00 MiB
[    3.499765] [drm] Initialized omapdrm 1.0.0 20110917 on minor 0
[    3.509760] mmcblk0boot1: mmc1:0001 S10004 partition 2 4.00 MiB
[    3.518247]  mmcblk0: p1 p2 p3 p4 p5 p6 p7 p8 p9 p10 p11 p12 p13 p14 p15
[    3.715727] input: gpio_keys as /devices/platform/gpio_keys/input/input1
[    3.722621] hctosys: unable to open rtc device (rtc0)
[    3.737784] aic_dvdd_fixed: disabling
[    3.742268] vmmcwl_fixed: disabling
[    3.745927] ALSA device list:
[    3.748902]   #1: HDMI 58040000.encoder
[    3.754160] Freeing unused kernel memory: 2048K
[    3.760881] init: init first stage started!
[    3.765196] init: First stage mount skipped (missing/incompatible fstab in device tree)
[    3.773290] init: Skipped setting INIT_AVB_VERSION (not in recovery mode)
[    3.780132] init: Loading SELinux policy
[    3.875682] audit: type=1403 audit(3.859:2): policy loaded auid=4294967295 ses=4294967295
[    3.884138] selinux: SELinux: Loaded policy from /sepolicy
[    3.884138] 
[    3.893872] selinux: SELinux: Loaded file_contexts
[    3.893872] 
[    3.901434] random: init: uninitialized urandom read (40 bytes read, 10 bits of entropy available)
[    3.911643] init: init second stage started!
[    3.920133] init: property_set("ro.boot.console", "ttyS2") failed: property already set
[    3.928189] init: property_set("ro.boot.hardware", "am57xevmboard") failed: property already set
[    3.938966] selinux: SELinux: Loaded file_contexts
[    3.938966] 
[    3.945677] selinux: SELinux: Loaded property_contexts from /plat_property_contexts & /nonplat_property_contexts.
[    3.945677] 
[    3.957503] init: Running restorecon...
[    3.988327] selinux: SELinux:  Could not stat /dev/block: No such file or directory.
[    3.988327] 
[    3.997742] init: waitpid failed: No child processes
[    4.002785] init: Couldn't load properties from /system/etc/prop.default: No such file or directory
[    4.011921] init: Couldn't load properties from /prop.default: No such file or directory
[    4.020277] init: Couldn't load properties from /odm/default.prop: No such file or directory
[    4.028773] init: Couldn't load properties from /vendor/default.prop: No such file or directory
[    4.037924] init: Created socket '/dev/socket/property_service', mode 666, user 0, group 0
[    4.046385] init: Parsing file /init.rc...
[    4.050869] init: Added '/init.environ.rc' to import list
[    4.056306] init: Added '/init.usb.rc' to import list
[    4.061410] init: Added '/init.am57xevmboard.rc' to import list
[    4.067368] init: Added '/vendor/etc/init/hw/init.am57xevmboard.rc' to import list
[    4.075003] init: Added '/init.usb.configfs.rc' to import list
[    4.080887] init: Added '/init.zygote32.rc' to import list
[    4.087595] init: Parsing file /init.environ.rc...
[    4.092538] init: Parsing file /init.usb.rc...
[    4.097405] init: Parsing file /init.am57xevmboard.rc...
[    4.102832] init: Added '/init.am57xevmboard.usb.rc' to import list
[    4.109139] init: Added '/init.am57xhsevmboard.rc' to import list
[    4.115496] init: Parsing file /init.am57xevmboard.usb.rc...
[    4.121427] init: Parsing file /init.am57xhsevmboard.rc...
[    4.126958] init: could not import file '/init.am57xhsevmboard.rc' from '/init.am57xevmboard.rc': No such file or directory
[    4.138185] init: Parsing file /vendor/etc/init/hw/init.am57xevmboard.rc...
[    4.145211] init: could not import file '/vendor/etc/init/hw/init.am57xevmboard.rc' from '/init.rc': No such file or directory
[    4.156688] init: Parsing file /init.usb.configfs.rc...
[    4.162562] init: Parsing file /init.zygote32.rc...
[    4.167567] init: Parsing file /system/etc/init...
[    4.172435] init: Parsing file /vendor/etc/init...
[    4.177270] init: Parsing file /odm/etc/init...
[    4.181899] init: processing action (early-init)
[    4.186920] init: starting service 'ueventd'...
[    4.192048] init: failed to open /acct/uid_0/pid_115/cgroup.procs: No such file or directory
[    4.193007] random: ueventd: uninitialized urandom read (40 bytes read, 11 bits of entropy available)
[    4.194552] ueventd: ueventd started!
[    4.195072] ueventd: /sys/ rule /sys/devices/platform/trusty.* trusty_version
[    4.195091] ueventd: /sys/ rule /sys/devices/virtual/input/input* enable
[    4.195109] ueventd: /sys/ rule /sys/devices/virtual/input/input* poll_delay
[    4.195126] ueventd: /sys/ rule /sys/devices/virtual/usb_composite/* enable
[    4.195151] ueventd: /sys/ rule /sys/devices/system/cpu/cpu* cpufreq/scaling_max_freq
[    4.195167] ueventd: /sys/ rule /sys/devices/system/cpu/cpu* cpufreq/scaling_min_freq
[    4.197418] selinux: SELinux: Loaded file_contexts
[    4.197418] 
[    4.202135] selinux: SELinux: Loaded file_contexts
[    4.202135] 
[    4.204689] ueventd: fixup /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq 1000 1000 664
[    4.204846] ueventd: fixup /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq 1000 1000 664
[    4.205275] ueventd: fixup /sys/devices/system/cpu/cpu1/cpufreq/scaling_max_freq 1000 1000 664
[    4.205362] ueventd: fixup /sys/devices/system/cpu/cpu1/cpufreq/scaling_min_freq 1000 1000 664
[    4.305344] init: createProcessGroup(0, 115) failed for service 'ueventd': No such file or directory
[    4.314631] init: Command 'start ueventd' action=early-init (/init.rc:30) returned 0 took 127.725ms.
[    4.323977] init: processing action (wait_for_coldboot_done)
[    4.355327] ueventd: firmware: loading 'dra7-dsp2-fw.xe66' for '/devices/platform/44000000.ocp/41000000.dsp/remoteproc3/firmware/dra7'
[    4.378168] ueventd: firmware: loading 'dra7-ipu2-fw.xem4' for '/devices/platform/44000000.ocp/55020000.ipu/remoteproc1/firmware/dra7'
[    4.409159] ueventd: firmware: loading 'dra7-dsp1-fw.xe66' for '/devices/platform/44000000.ocp/40800000.dsp/remoteproc2/firmware/dra7'
[    4.422858] ueventd: firmware: loading 'dra7-ipu1-fw.xem4' for '/devices/platform/44000000.ocp/58820000.ipu/remoteproc0/firmware/dra7'
[    4.451816] ueventd: Coldboot took 0.254238 seconds
[    4.456425] init: Command 'wait_for_coldboot_done' action=wait_for_coldboot_done returned 0 took 126.707ms.
[    4.456467] init: processing action (mix_hwrng_into_linux_rng)
[    4.456497] init: /dev/hw_random not found
[    4.456524] init: processing action (set_mmap_rnd_bits)
[    4.456748] init: processing action (set_kptr_restrict)
[    4.456965] init: processing action (keychord_init)
[    4.456992] init: processing action (console_init)
[    4.457032] init: processing action (init)
[    4.501630] init: write_file: Unable to open '/proc/sys/kernel/hung_task_timeout_secs': No such file or directory
[    4.517619] init: write_file: Unable to open '/proc/sys/abi/swp': No such file or directory
[    4.526255] init: write_file: Unable to open '/sys/class/leds/vibrator/trigger': No such file or directory
[    4.536150] init: processing action (mix_hwrng_into_linux_rng)
[    4.542346] init: /dev/hw_random not found
[    4.546477] init: processing action (late-init)
[    4.551112] init: processing action (queue_property_triggers)
[    4.556903] init: processing action (fs)
[    4.561918] init: [libfs_mgr]Not running /system/bin/tune2fs on /dev/block/platform/44000000.ocp/480b4000.mmc/by-name/system (executa)
[    4.580371] EXT4-fs (mmcblk0p10): mounted filesystem with ordered data mode. Opts: (null)
[    4.588674] init: [libfs_mgr]__mount(source=/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/system,target=/system,type=ext4)=0
[    4.604880] init: [libfs_mgr]superblock s_max_mnt_count:10,/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/vendor
[    4.615736] init: [libfs_mgr]Requested quota status is match on /dev/block/platform/44000000.ocp/480b4000.mmc/by-name/vendor
[    4.631134] EXT4-fs (mmcblk0p11): mounted filesystem with ordered data mode. Opts: (null)
[    4.639411] init: [libfs_mgr]__mount(source=/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/vendor,target=/vendor,type=ext4)=0
[    4.652434] init: [libfs_mgr]superblock s_max_mnt_count:10,/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/cache
[    4.663221] init: [libfs_mgr]do_quota_with_shutdown_check(): was not clealy shutdown, state flag:1incompat flag:46
[    4.673635] init: [libfs_mgr]Requested quota status is match on /dev/block/platform/44000000.ocp/480b4000.mmc/by-name/cache
[    4.685220] EXT4-fs (mmcblk0p12): Ignoring removed nomblk_io_submit option
[    4.700198] EXT4-fs (mmcblk0p12): recovery complete
[    4.706027] EXT4-fs (mmcblk0p12): mounted filesystem with ordered data mode. Opts: errors=remount-ro,nomblk_io_submit
[    4.716793] init: [libfs_mgr]check_fs(): mount(/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/cache,/cache,ext4)=0: Success
[    4.752944] init: [libfs_mgr]check_fs(): unmount(/cache) succeeded
[    4.759253] init: [libfs_mgr]Running /system/bin/e2fsck on /dev/block/platform/44000000.ocp/480b4000.mmc/by-name/cache
[    4.770528] ueventd: loading /devices/platform/44000000.ocp/55020000.ipu/remoteproc1/firmware/dra7-ipu2-fw.xem4 took 0.392564 seconds
[    4.783250]  remoteproc1: powering up 55020000.ipu
[    4.788135]  remoteproc1: Booting fw image dra7-ipu2-fw.xem4, size 3730752
[    4.795160] omap-iommu 55082000.mmu: 55082000.mmu: version 2.1
[    4.812754] random: e2fsck: uninitialized urandom read (40 bytes read, 15 bits of entropy available)
[    4.854325]  remoteproc1: remote processor 55020000.ipu is now up
[    4.861147] virtio_rpmsg_bus virtio0: rpmsg host is online
[    4.866720]  remoteproc1: registered virtio0 (type 7)
[    4.872269] virtio_rpmsg_bus virtio0: creating channel rpmsg-rpc addr 0x65
[    4.879793] rpmsg_rpc rpmsg0: probing service dce-callback with src 1024 dst 101
[    4.887998] virtio_rpmsg_bus virtio0: creating channel rpmsg-rpc addr 0x66
[    4.895469] rpmsg_rpc rpmsg1: probing service rpmsg-dce with src 1025 dst 102
[    4.903279] rpmsg_rpc rpmsg0: published functions = 4
[    4.908571] rpmsg_rpc rpmsg1: published functions = 9
[    4.921476] random: e2fsck: uninitialized urandom read (40 bytes read, 18 bits of entropy available)
[    4.965359] e2fsck: e2fsck 1.43.3 (04-Sep-2016)
[    4.965359] 
[    4.971509] e2fsck: Pass 1: Checking inodes, blocks, and sizes
[    4.971509] 
[    4.978853] e2fsck: Pass 2: Checking directory structure
[    4.978853] 
[    4.985703] e2fsck: Pass 3: Checking directory connectivity
[    4.985703] 
[    4.992867] e2fsck: Pass 4: Checking reference counts
[    4.992867] 
[    4.999454] e2fsck: Pass 5: Checking group summary information
[    4.999454] 
[    5.006797] e2fsck: cache: 14/16384 files (0.0% non-contiguous), 2093/65536 blocks
[    5.006797] 
[    5.018180] EXT4-fs (mmcblk0p12): mounted filesystem with ordered data mode. Opts: (null)
[    5.026509] init: [libfs_mgr]__mount(source=/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/cache,target=/cache,type=ext4)=0
[    5.039296] init: [libfs_mgr]superblock s_max_mnt_count:10,/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/userdata
[    5.050355] init: [libfs_mgr]do_quota_with_shutdown_check(): was not clealy shutdown, state flag:1incompat flag:46
[    5.060787] init: [libfs_mgr]Requested quota status is match on /dev/block/platform/44000000.ocp/480b4000.mmc/by-name/userdata
[    5.072671] EXT4-fs (mmcblk0p15): Ignoring removed nomblk_io_submit option
[    5.226512] EXT4-fs (mmcblk0p15): recovery complete
[    5.232160] EXT4-fs (mmcblk0p15): mounted filesystem with ordered data mode. Opts: errors=remount-ro,nomblk_io_submit
[    5.242943] init: [libfs_mgr]check_fs(): mount(/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/userdata,/data,ext4)=0: Success
[    5.293141] init: [libfs_mgr]check_fs(): unmount(/data) succeeded
[    5.299292] init: [libfs_mgr]Running /system/bin/e2fsck on /dev/block/platform/44000000.ocp/480b4000.mmc/by-name/userdata
[    5.311925] random: e2fsck: uninitialized urandom read (40 bytes read, 31 bits of entropy available)
[    5.331182] random: e2fsck: uninitialized urandom read (40 bytes read, 31 bits of entropy available)
[    5.577857] e2fsck: e2fsck 1.43.3 (04-Sep-2016)
[    5.577857] 
[    5.583950] e2fsck: Pass 1: Checking inodes, blocks, and sizes
[    5.583950] 
[    5.591321] e2fsck: Pass 2: Checking directory structure
[    5.591321] 
[    5.598139] e2fsck: Pass 3: Checking directory connectivity
[    5.598139] 
[    5.605298] e2fsck: Pass 4: Checking reference counts
[    5.605298] 
[    5.611883] e2fsck: Pass 5: Checking group summary information
[    5.611883] 
[    5.619226] e2fsck: data: 1881/131072 files (1.6% non-contiguous), 40759/524288 blocks
[    5.619226] 
[    5.633707] EXT4-fs (mmcblk0p15): mounted filesystem with ordered data mode. Opts: (null)
[    5.642044] init: [libfs_mgr]__mount(source=/dev/block/platform/44000000.ocp/480b4000.mmc/by-name/userdata,target=/data,type=ext4)=0
[    5.654565] init: Parsing directory /system/etc/init...
[    5.660402] init: Parsing file /system/etc/init/android.hidl.allocator@1.0-service.rc...
[    5.668966] init: Parsing file /system/etc/init/atrace.rc...
[    5.676022] init: Parsing file /system/etc/init/atrace_userdebug.rc...
[    5.683184] init: Parsing file /system/etc/init/audioserver.rc...
[    5.689811] init: Parsing file /system/etc/init/bootanim.rc...
[    5.696085] init: Parsing file /system/etc/init/bootstat.rc...
[    5.702437] init: Parsing file /system/etc/init/cameraserver.rc...
[    5.709061] init: Parsing file /system/etc/init/drmserver.rc...
[    5.715521] init: Parsing file /system/etc/init/dumpstate.rc...
[    5.721918] init: Parsing file /system/etc/init/gatekeeperd.rc...
[    5.728416] init: Parsing file /system/etc/init/hwservicemanager.rc...
[    5.735445] init: Parsing file /system/etc/init/init-debug.rc...
[    5.741905] init: Parsing file /system/etc/init/installd.rc...
[    5.748783] init: Parsing file /system/etc/init/keystore.rc...
[    5.755120] init: Parsing file /system/etc/init/lmkd.rc...
[    5.761051] init: Parsing file /system/etc/init/logcatd.rc...
[    5.767376] init: Parsing file /system/etc/init/logd.rc...
[    5.773366] init: Parsing file /system/etc/init/logtagd.rc...
[    5.779605] init: Parsing file /system/etc/init/mdnsd.rc...
[    5.785611] init: Parsing file /system/etc/init/mediadrmserver.rc...
[    5.792431] init: Parsing file /system/etc/init/mediaextractor.rc...
[    5.799216] init: Parsing file /system/etc/init/mediametrics.rc...
[    5.805836] init: Parsing file /system/etc/init/mediaserver.rc...
[    5.812382] init: Parsing file /system/etc/init/mtpd.rc...
[    5.818370] init: Parsing file /system/etc/init/netd.rc...
[    5.824889] init: Parsing file /system/etc/init/perfprofd.rc...
[    5.831264] init: Parsing file /system/etc/init/racoon.rc...
[    5.837386] init: Parsing file /system/etc/init/servicemanager.rc...
[    5.844225] init: Parsing file /system/etc/init/storaged.rc...
[    5.850601] init: Parsing file /system/etc/init/surfaceflinger.rc...
[    5.857410] init: Parsing file /system/etc/init/tombstoned.rc...
[    5.863857] init: Parsing file /system/etc/init/uncrypt.rc...
[    5.870084] init: Parsing file /system/etc/init/vdc.rc...
[    5.875868] init: Parsing file /system/etc/init/vold.rc...
[    5.881829] init: Parsing file /system/etc/init/webview_zygote32.rc...
[    5.888796] init: Parsing file /system/etc/init/wifi-events.rc...
[    5.895847] init: Parsing directory /vendor/etc/init...
[    5.901498] init: Parsing file /vendor/etc/init/android.hardware.audio@2.0-service.rc...
[    5.910087] init: Parsing file /vendor/etc/init/android.hardware.camera.provider@2.4-service.rc...
[    5.919482] init: Parsing file /vendor/etc/init/android.hardware.configstore@1.0-service.rc...
[    5.928521] init: Parsing file /vendor/etc/init/android.hardware.graphics.allocator@2.0-service.rc...
[    5.938202] init: Parsing file /vendor/etc/init/android.hardware.keymaster@3.0-service.rc...
[    5.947072] init: Parsing file /vendor/etc/init/android.hardware.media.omx@1.0-service.rc...
[    5.956029] init: Parsing file /vendor/etc/init/android.hardware.usb@1.0-service.rc...
[    5.964365] init: Parsing file /vendor/etc/init/android.hardware.wifi@1.0-service.rc...
[    5.972787] init: Parsing file /vendor/etc/init/vndservicemanager.rc...
[    5.979823] init: Parsing file /odm/etc/init...
[    5.984441] init: Command 'mount_all /fstab.am57xevmboard' action=fs (/init.am57xevmboard.rc:34) returned 0 took 1423.5ms.
[    5.995950] init: processing action (post-fs)
[    6.001204] init: Couldn't load properties from /odm/build.prop: No such file or directory
[    6.009991] init: Couldn't load properties from /factory/factory.prop: No such file or directory
[    6.018915] init: [libfs_mgr]fs_mgr_read_fstab_dt(): failed to read fstab from dt
[    6.027494] init: computing context for service 'logd'
[    6.033696] init: starting service 'logd'...
[    6.039285] init: Created socket '/dev/socket/logd', mode 666, user 1036, group 1036
[    6.047562] init: Created socket '/dev/socket/logdr', mode 666, user 1036, group 1036
[    6.055578] init: computing context for service 'servicemanager'
[    6.055974] init: Created socket '/dev/socket/logdw', mode 222, user 1036, group 1036
[    6.056073] init: Opened file '/proc/kmsg', flags 0
[    6.056113] init: Opened file '/dev/kmsg', flags 1
[    6.059575] random: logd: uninitialized urandom read (40 bytes read, 50 bits of entropy available)
[    6.077390] random: logd: uninitialized urandom read (40 bytes read, 51 bits of entropy available)
[    6.088383] logd.auditd: start
[    6.088397] logd.klogd: 6079078640
[    6.103970] init: starting service 'servicemanager'...
[    6.109882] init: Command 'start servicemanager' action=post-fs (/init.rc:301) returned 0 took 71.0837ms.
[    6.114461] random: servicemanager: uninitialized urandom read (40 bytes read, 52 bits of entropy available)
[    6.129545] init: computing context for service 'hwservicemanager'
[    6.134374] random: servicemanager: uninitialized urandom read (40 bytes read, 53 bits of entropy available)
[    6.145814] init: starting service 'hwservicemanager'...
[    6.152397] init: computing context for service 'vndservicemanager'
[    6.158863] init: starting service 'vndservicemanager'...
[    6.216523] selinux: SELinux: Skipping restorecon_recursive(/cache)
[    6.216523] 
[    6.220562] binder: 146:146 ioctl 620a a32064f7 returned -22
[    6.236090] init: write_file: Unable to open '/sys/kernel/tracing/tracing_on': No such file or directory
[    6.246904] init: processing action (late-fs)
[    6.251804] init: computing context for service 'keymaster-3-0'
[    6.257876] init: starting service 'keymaster-3-0'...
[    6.263762] init: processing action (post-fs-data)
[    6.268934] init: computing context for service 'vold'
[    6.274282] init: starting service 'vold'...
[    6.280557] init: Created socket '/dev/socket/vold', mode 660, user 0, group 1009
[    6.288628] init: Created socket '/dev/socket/cryptd', mode 660, user 0, group 1009
[    6.401297] vdc: 200 150 Command succeeded
[    6.405667] init: init_user0 result: 0
[    6.409655] init: Command 'init_user0' action=post-fs-data (/init.rc:489) returned 0 took 64.225ms.
[    6.429170] selinux: SELinux: Skipping restorecon_recursive(/data)
[    6.429170] 
[    6.437230] init: computing context for service 'exec 1 (/system/bin/tzdatacheck)'
[    6.445018] init: starting service 'exec 1 (/system/bin/tzdatacheck)'...
[    6.452884] init: SVC_EXEC pid 155 (uid 1000 gid 1000+0 context default) started; waiting...
[    6.468025] init: Service 'exec 1 (/system/bin/tzdatacheck)' (pid 155) exited with status 0 waiting took 0.030858 seconds
[    6.482296] selinux: SELinux: Skipping restorecon_recursive(/data/tee)
[    6.482296] 
[    6.491198] init: processing action (ro.crypto.state=unsupported && zygote-start)
[    6.498726] init: ExecStart(update_verifier_nonencrypted): Service not found
[    6.505921] init: computing context for service 'netd'
[    6.511249] init: starting service 'netd'...
[    6.516373] init: computing context for service 'zygote'
[    6.516824] init: Created socket '/dev/socket/netd', mode 660, user 0, group 1000
[    6.517301] init: Created socket '/dev/socket/dnsproxyd', mode 660, user 0, group 3003
[    6.517637] init: Created socket '/dev/socket/mdns', mode 660, user 0, group 1000
[    6.517975] init: Created socket '/dev/socket/fwmarkd', mode 660, user 0, group 3003
[    6.552867] init: starting service 'zygote'...
[    6.558068] init: do_start: Service zygote_secondary not found
[    6.561599] init: Created socket '/dev/socket/zygote', mode 660, user 0, group 1000
[    6.571812] init: processing action (load_persist_props_action)
[    6.577819] init: Couldn't load properties from /data/local.prop: No such file or directory
[    6.643092] init: Command 'load_persist_props' action=load_persist_props_action (/init.rc:250) returned 0 took 65.2653ms.
[    6.654255] init: computing context for service 'logd-reinit'
[    6.660265] init: starting service 'logd-reinit'...
[    6.665951] init: processing action (firmware_mounts_complete)
[    6.672011] init: processing action (early-boot)
[    6.682109] init: processing action (boot)
[    6.686842] init: write_file: Unable to open '/proc/sys/vm/min_free_order_shift': No such file or directory
[    6.700751] init: computing context for service 'hidl_memory'
[    6.701078] logd.daemon: reinit
[    6.710656] init: starting service 'hidl_memory'...
[    6.716334] init: computing context for service 'audio-hal-2-0'
[    6.719387] ueventd: firmware: could not find firmware for dra7-dsp2-fw.xe66
[    6.719458] ueventd: loading /devices/platform/44000000.ocp/41000000.dsp/remoteproc3/firmware/dra7-dsp2-fw.xe66 took 2.36435 seconds
[    6.719865]  remoteproc3: failed to load dra7-dsp2-fw.xe66
[    6.747251] init: starting service 'audio-hal-2-0'...
[    6.753149] init: computing context for service 'camera-provider-2-4'
[    6.756690] ueventd: firmware: could not find firmware for dra7-ipu1-fw.xem4
[    6.756765] ueventd: loading /devices/platform/44000000.ocp/58820000.ipu/remoteproc0/firmware/dra7-ipu1-fw.xem4 took 2.33407 seconds
[    6.757146]  remoteproc0: failed to load dra7-ipu1-fw.xem4
[    6.763472] ueventd: firmware: could not find firmware for dra7-dsp1-fw.xe66
[    6.763543] ueventd: loading /devices/platform/44000000.ocp/40800000.dsp/remoteproc2/firmware/dra7-dsp1-fw.xe66 took 2.35459 seconds
[    6.763906]  remoteproc2: failed to load dra7-dsp1-fw.xe66
[    6.809052] init: starting service 'camera-provider-2-4'...
[    6.815539] init: computing context for service 'configstore-hal-1-0'
[    6.822304] init: starting service 'configstore-hal-1-0'...
[    6.822575] init: couldn't write 172 to /dev/cpuset/camera-daemon/tasks: No such file or directory
[    6.837756] init: computing context for service 'gralloc-2-0'
[    6.843814] init: starting service 'gralloc-2-0'...
[    6.849546] init: computing context for service 'usb-hal-1-0'
[    6.855537] init: starting service 'usb-hal-1-0'...
[    6.861277] init: computing context for service 'wifi_hal_legacy'
[    6.867604] init: starting service 'wifi_hal_legacy'...
[    6.873646] init: Command 'class_start hal' action=boot (/init.rc:618) returned 0 took 174.081ms.
[    6.882842] init: Service 'logd-reinit' (pid 164) exited with status 0
[    6.889702] init: computing context for service 'healthd'
[    6.895308] init: starting service 'healthd'...
[    6.900695] init: computing context for service 'pvrsrvinit'
[    6.906625] init: starting service 'pvrsrvinit'...
[    6.912305] init: computing context for service 'lmkd'
[    6.917699] init: starting service 'lmkd'...
[    6.922925] init: computing context for service 'surfaceflinger'
[    6.929196] init: starting service 'surfaceflinger'...
[    6.935383] init: computing context for service 'exec 2 (/system/bin/init.am57xevmboard.cpuset.sh)'
[    6.944731] init: starting service 'exec 2 (/system/bin/init.am57xevmboard.cpuset.sh)'...
[    6.953619] init: SVC_EXEC pid 183 (uid 0 gid 0+1 context default) started; waiting...
[    6.975756] init: Created socket '/dev/socket/lmkd', mode 660, user 1000, group 1000
[    6.991524] init: Failed to bind socket 'pdx/system/vr/display/client': No such file or directory
[    7.030259] init: Failed to bind socket 'pdx/system/vr/display/manager': No such file or directory
[    7.090044] init: Failed to bind socket 'pdx/system/vr/display/vsync': No such file or directory
[    7.305609] healthd: unable to get HAL interface, using defaults
[    7.335494] PVR_K: UM DDK-(4081762) and KM DDK-(4081762) match. [ OK ]
[    7.360228] healthd: No battery devices found
[    7.364653] healthd: battery l=100 v=0 t=42.4 h=2 st=2 chg=a
[    7.371955] init: Service 'exec 2 (/system/bin/init.am57xevmboard.cpuset.sh)' (pid 183) exited with status 0 waiting took 0.436671 ses
[    7.388223] init: Service 'pvrsrvinit' (pid 180) exited with status 0
[    7.395244] init: insmod: open("/system/lib/modules/galcore.ko") failed: No such file or directory
[    7.409154] file system registered
[    7.426799] init: processing action (persist.sys.usb.config=* && boot)
[    7.434171] init: processing action (enable_property_trigger)
[    7.440722] init: processing action (security.perf_harden=1)
[    7.447082] init: processing action (ro.debuggable=1)
[    7.454391] init: starting service 'console'...
[    7461118] init: processing action (sys.usb.config=adb && sys.usb.configfs=1)
[    7.468735] init: starting service 'adbd'...
[    7.474173] init: processing action (nonencrypted)
[    7.480246] init: Created socket '/dev/socket/adbd', mode 660, user 1000, group 1000
[    7.488184] init: setpgid failed for console: Operation not permitted
[    7.488221] init: cannot find '/system/bin/install-recovery.sh', disabling 'flash_recovery': No such file or directory
[    7.488320] init: computing context for service 'tee-supplicant'
[    7.488578] init: starting service 'tee-supplicant'...
[    7.489458] init: computing context for service 'audioserver'
[    7.489648] init: starting service 'audioserver'...
[    7.490506] init: computing context for service 'cameraserver'
[    7.490698] init: starting service 'cameraserver'...
[    7.491507] init: computing context for service 'drm'
[    7.491695] init: starting service 'drm'...
[    7.492500] init: computing context for service 'installd'
[    7.492663] init: starting service 'installd'...
[    7.493382] init: computing context for service 'keystore'
[    7.493561] init: starting service 'keystore'...
[    7.494343] init: computing context for service 'mediadrm'
[    7.494547] init: starting service 'mediadrm'...
[    7.495357] init: computing context for service 'mediaextractor'
[    7.495566] init: starting service 'mediaextractor'...
[    7.496350] init: computing context for service 'mediametrics'
[    7.496537] init: starting service 'mediametrics'...
[    7.497293] init: computing context for service 'media'
[    7.497494] init: starting service 'media'...
[    7.498249] init: computing context for service 'storaged'
[    7.498425] init: starting service 'storaged'...
[    7.499180] init: computing context for service 'mediacodec'
[    7.539997] init: Failed to open file '/d/mmc0/mmc0:0001/ext_csd': No such file or directory
[    7.549457] init: starting service 'mediacodec'...
[    7.550362] init: Command 'class_start main' action=nonencrypted (/init.rc:623) returned 0 took 62.0923ms.
[    7.550537] init: computing context for service 'gatekeeperd'
[    7.550725] init: starting service 'gatekeeperd'...
[    7.555591] type=1400 audit(7.539:3): avc: denied { read write } for pid=221 comm="tee-supplicant" name="teepriv0" dev="tmpfs" ino=611
[    7.555954] type=1400 audit(7.539:3): avc: denied { read write } for pid=221 comm="tee-supplicant" name="teepriv0" dev="tmpfs" ino=611
[    7.555962] type=1400 audit(7.539:4): avc: denied { open } for pid=221 comm="tee-supplicant" path="/dev/teepriv0" dev="tmpfs" ino=6161
[    7.556025] type=1400 audit(7.539:4): avc: denied { open } for pid=221 comm="tee-supplicant" path="/dev/teepriv0" dev="tmpfs" ino=6161
[    7.556031] type=1400 audit(7.539:5): avc: denied { ioctl } for pid=221 comm="tee-supplicant" path="/dev/teepriv0" dev="tmpfs" ino=611
[    7.562091] read descriptors
[    7.562099] read descriptors
[    7.562103] read strings
[    7.564705] init: cannot find '/system/xbin/perfprofd', disabling 'perfprofd': No such file or directory
[    7.564829] init: computing context for service 'tombstoned'
[    7.565043] init: starting service 'tombstoned'...
[    7.579719] init: Created socket '/dev/socket/tombstoned_crash', mode 666, user 1000, group 1000
[    7.580133] init: Created socket '/dev/socket/tombstoned_intercept', mode 666, user 1000, group 1000
[    7.601292] init: processing action (sys.usb.config=adb && sys.usb.configfs=1 && sys.usb.ffs.ready=1)
[    7.812270] init: couldn't write 223 to /dev/cpuset/camera-daemon/tasks: No such file or directory
[    7.974260] init: Command 'write /config/usb_gadget/g1/UDC ${sys.usb.controller}' action=sys.usb.config=adb && sys.usb.configfs=1 && .
[    8.036108] android_work: sent uevent USB_STATE=CONNECTED
[    8.058784] android_work: sent uevent USB_STATE=DISCONNECTED
am57xevm:/ $ [    8.124257] android_work: sent uevent USB_STATE=CONNECTED
[    8.130839] configfs-gadget gadget: high-speed config #1: b
[    8.139543] android_work: sent uevent USB_STATE=CONFIGURED
[    8.250621] init: computing context for service 'bootanim'
[    8.288861] init: starting service 'bootanim'...
[    8.630704] healthd: battery l=100 v=0 t=42.4 h=2 st=2 chg=a
[    8.948215] random: nonblocking pool is initialized
[   10.047787] capability: warning: `main' uses 32-bit capabilities (legacy support in use)
[   12.980500] healthd: battery l=100 v=0 t=42.4 h=2 st=2 chg=a
[   14.055460] init: processing action (sys.sysctl.extra_free_kbytes=*)
[   14.681158] net eth0: initializing cpsw version 1.15 (0)
[   14.687184] net eth0: initialized cpsw ale version 1.4
[   14.693523] net eth0: ALE Table size 1024
[   14.801491] net eth0: phy found : id is : 0x221622
[   14.821641] IPv6: ADDRCONF(NETDEV_UP): eth0: link is not ready
[   14.894509] IPv6: ADDRCONF(NETDEV_UP): eth0: link is not ready
[   14.926285] init: computing context for service 'webview_zygote32'
[   14.938935] init: starting service 'webview_zygote32'...
[   14.951520] init: Created socket '/dev/socket/webview_zygote', mode 660, user 1053, group 1000
[   15.345426] omap-iommu 55082000.mmu: 55082000.mmu: version 2.1
[   17.443373] init: Service 'bootanim' (pid 253) exited with status 0
[   17.496829] init: processing action (sys.boot_completed=1)
[   17.504042] init: processing action (sys.boot_completed=1 && sys.logbootcomplete=1)
[   17.521551] init: computing context for service 'exec 3 (/system/bin/bootstat)'
[   17.530480] init: starting service 'exec 3 (/system/bin/bootstat)'...
[   17.548692] init: SVC_EXEC pid 686 (uid 0 gid 0+0 context default) started; waiting...
[   17.578009] init: Service 'exec 3 (/system/bin/bootstat)' (pid 686) exited with status 0 waiting took 0.056579 seconds
[   17.591002] init: computing context for service 'exec 4 (/system/bin/bootstat)'
[   17.600092] init: starting service 'exec 4 (/system/bin/bootstat)'...
[   17.609219] init: SVC_EXEC pid 687 (uid 0 gid 0+0 context default) started; waiting...
[   17.674505] init: Service 'exec 4 (/system/bin/bootstat)' (pid 687) exited with status 0 waiting took 0.083494 seconds
[   17.708915] init: computing context for service 'exec 5 (/system/bin/bootstat)'
[   17.725083] init: starting service 'exec 5 (/system/bin/bootstat)'...
[   17.743715] init: SVC_EXEC pid 708 (uid 0 gid 0+0 context default) started; waiting...
[   17.760580] init: Command 'exec - root root -- /system/bin/bootstat --record_time_since_factory_reset' action=sys.boot_completed=1 &&.
[   17.783756] init: Service 'exec 5 (/system/bin/bootstat)' (pid 708) exited with status 0 waiting took 0.074849 seconds
[   17.797262] init: computing context for service 'exec 6 (/system/bin/bootstat)'
[   17.809507] init: starting service 'exec 6 (/system/bin/bootstat)'...
[   17.817659] init: SVC_EXEC pid 713 (uid 0 gid 0+0 context default) started; waiting...
[   17.859688] init: Command 'exec - root root -- /system/bin/bootstat -l' action=sys.boot_completed=1 && sys.logbootcomplete=1 (/system.
[   17.906852] init: Service 'exec 6 (/system/bin/bootstat)' (pid 713) exited with status 0 waiting took 0.109582 seconds
[   17.929587] init: processing action (sys.boot_completed=1 && sys.wifitracing.started=0)
[   18.008480] init: Command 'mkdir /sys/kernel/debug/tracing/instances/wifi 711' action=sys.boot_completed=1 && sys.wifitracing.started.
[   20.308862] init: Command 'restorecon_recursive /sys/kernel/debug/tracing/instances/wifi' action=sys.boot_completed=1 && sys.wifitrac.
[   20.821756] type=1400 audit(7.539:5): avc: denied { ioctl } for pid=221 comm="tee-supplicant" path="/dev/teepriv0" dev="tmpfs" ino=611
[   20.821760] selinux: avc:  denied  { set } for property=sys.usb.ffs.mtp.ready pid=744 uid=10006 gid=10006 scontext=u:r:priv_app:s0:c5e
[   20.821760] 
[   20.869302] type=1400 audit(20.809:6): avc: denied { write } for pid=744 comm="d.process.media" name="property_service" dev="tmpfs" i1
[   20.891560] type=1400 audit(20.809:6): avc: denied { write } for pid=744 comm="d.process.media" name="property_service" dev="tmpfs" i1
[   20.913893] type=1400 audit(20.809:7): avc: denied { connectto } for pid=744 comm="d.process.media" path="/dev/socket/property_servic1

am57xevm:/ $ 
am57xevm:/ $ xtest
Run test suite with level=0

TEE test application started with device [(null)]
[  142.779179] type=1400 audit(20.809:7): avc: denied { connectto } for pid=744 comm="d.process.media" path="/dev/socket/property_servic1
######################################################
#
# regrssion
#
######################################################
 
[  142.806826] type=1400 audit(142.759:8): avc: denied { read write } for pid=1060 comm="xtest" name="tee0" dev="tmpfs" ino=6168 scontex1
 *regression_1001 Core self tests
 r egression_1001 OK
 
 *regrssion_1002 PTA parameters
[  142.837806] type=1400 audit(142.759:8): avc: denied { read write } for pid=1060 comm="xtest" name="tee0" dev="tmpfs" ino=6168 scontex1
  regression_1002 OK
 
* regression_1004 Test User Crypt TA
[  142.871813] type=1400 audit(142.759:9): avc: denied { open } for pid=1060 comm="xtest" path="/dev/tee0" dev="tmpfs" ino=6168 scontext1
 roegression_1004.1 AES encrypt[  142.890139] type=1400 audit(142.759:9): avc: denied { open } for pid=1060 comm="xtest" path="/dev/tee01

  regression_1004.1 OK
o regression_1004.2 AES decrypt[  142.910993] type=1400 audit(142.759:10): avc: denied { ioctl } for pid=1060 comm="xtest" path="/dev/te1

  regression_1004.2 OK
o regression_1004.3 SHA-256 test, 3 byts input
  regression_1004.3 OK
o regression_1004.4 AES-256 ECB encrypt test, 32 bytes input, with fixed key
  regression_1004.4 OK
o regression_1004.5 AES-256 ECB decrypt test, 32 bytes input, with fixed key
  regression_1004.5 OK
  regression_1004 OK
 
* regression_1005 Many sessions
  regression_1005 OK
 
* regression_1006 Test Basic OS features
ta_entry_basic: enter
Getting properties for current TA
Getting properties for current client
Getting properties for implementation
system time 148.318
REE time 142.989
ERROR:   USER-TA:test_time:520: TA time not stored
TA time 0.000
TA time 1.002
INFO:    USER-TA: Testing floating point operations
INFO:    USER-TA: Returned via longjmp

*********** TESTBENCH ***********
***         RUNNING: <<< Variables >>>
*********************************

*** INFO : Testing BigIntInit 

*********** TESTBENCH ***********
***         PASSED:  <<< Variables >>>
*********************************


*********** TESTBENCH ***********
***         RUNNING: <<< Conversion functions >>>
*********************************

*** INFO : Testing GetShort and SetShort 
*** INFO : Testing Convert to and from OctetString 

*********** TESTBENCH ***********
***         PASSED:  <<< Conversion functions >>>
*********************************


*********** TESTBENCH ***********
***         RUNNING: <<< Comparison functions >>>
*********************************

*** INFO : Testing TEE_BigIntCompare 
*** INFO :    Testing various cases 
*** INFO :    Testing equality 
*** INFO :    Testing equal magnitude, but different signs 
*** INFO : Testing TEE_BigIntCmpS32 
*** INFO :    Testing various cases 
*** INFO :    Testing large BigInt 

*********** TESTBENCH ***********
***         PASSED:  <<< Comparison functions >>>
*********************************


*********** TESTBENCH ***********
***         RUNNING: <<< Addition and Subtraction >>>
*********************************

*** INFO : Testing basic cases 
*** INFO : Both ops positive 
*** INFO : Both ops negative 
*** INFO : Op1 positive, op2 negative, |op1| > |op2| 
*** INFO : Op1 positive, op2 negative, |op1| < |op2| 
*** INFO : Op1 negative, op2 positive, |op1| > |op2| 
*** INFO : Op1 negative, op2 positive, |op1| < |op2| 
*** INFO : Testing AddWord and SubWord  
*** INFO : Testing Neg 

*********** TESTBENCH ***********
***         PASSED:  <<< Addition and Subtraction >>>
*********************************


*********** TESTBENCH ***********
***         RUNNING: <<< Multiplication >>>
*********************************

*** INFO : Testing basic cases 

*********** TESTBENCH ***********
***         PASSED:  <<< Multiplication >>>
*********************************


*********** TESTBENCH ***********
***         RUNNING: <<< Division >>>
*********************************

*** INFO :    Testing basic cases 
*** INFO :    Testing random divisions 
*** INFO :    Testing signs of q and r 

*********** TESTBENCH ***********
***         PASSED:  <<< Division >>>
*********************************


*********** TESTBENCH ***********
***         RUNNING: <<< Modular arithmetic >>>
*********************************

*** INFO :    Testing modular reduction 
*** INFO :    Testing modular addition and subtraction 
*** INFO :    Testing modular multiplication 
*** INFO :    Testing modular inversion 

*********** TESTBENCH ***********
***         PASSED:  <<< Modular arithmetic >>>
*********************************


*********** TESTBENCH ***********
***         RUNNING: <<< Primality Algorithms >>>
*********************************

*** INFO : Simple cases 
*** INFO : Large Composites 
*** INFO : Large Primes 

*********** TESTBENCH ***********
***         PASSED:  <<< Primality Algorithms >>>
*********************************


*********** TESTBENCH ***********
***     ALL TESTS PASSED      ***
*********************************

  regression_1006 OK
 
* regression_1007 Test Panic
ta_entry_panic: enter
<snip>
```





