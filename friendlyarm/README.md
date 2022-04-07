# 本目录下文件全为开发板附带资料，此处只做简要记录：

## 工作环境：

- Ubuntu20.04

- arm-linux-gcc version: 4.5.1

- linux3.5 (friendlyarm提供)

- busybox1.17.2(friendlyarm提供)

- uboot2010.12(friendlyarm提供)

- 一张4G以上的SD卡，分区如下：

  ```shell
  chao@ubuntu:~$ sudo lsblk -f /dev/sdb
  NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
  sdb                                                                     
  ├─sdb1 vfat         3568-C113                                           
  ├─sdb2 ext3         b0e1f3f5-eb2a-4796-ba6c-f80c509ae2d7                
  └─sdb3
  ```

  

- 

## 从sd卡加载uboot、kernel、rootfs
编译
设置临时环境变量(也可以直接修改Makefile,或者在执行make时进行传参)

```shell
chao@ubuntu:~$ export ARCH=arm
chao@ubuntu:~$ export CROSS_COMPILE=arm-linu-

#设置工作根目录
chao@ubuntu:~$ export WORKDIR=/home/chao/tiny412
```

### 准备uboot

```shell
chao@ubuntu:~$ cd ${WORKDIR}/friendlyarm/uboot_tiny4412
chao@ubuntu:~/tiny4412/friendlyarm/uboot_tiny4412$ make tiny4412_config
chao@ubuntu:~/tiny4412/friendlyarm/uboot_tiny4412$ make -j`nproc`
chao@ubuntu:~/tiny4412/friendlyarm/uboot_tiny4412$ cd sd_fuse
chao@ubuntu:~/tiny4412/friendlyarm/uboot_tiny4412/sd_fuse$ make
```

#### 烧录uboot

在ubuntu上插入sd卡，查看`/dev`，找到sd卡的节点

```shell
chao@ubuntu:~$ cd ${WORKDIR}/friendlyarm/uboot_tiny4412/sd_fuse/tiny4412
chao@ubuntu:~/tiny4412/friendlyarm/uboot_tiny4412/sd_fuse/tiny4412$ sudo ./sd_fusing.sh /dev/sdb
```

PS: `/dev/sdb`不唯一，根据ubuntu识别到的sd卡决定。

### 准备kernel

```shell
chao@ubuntu:~$ cd ${WORKDIR}/friendlyarm/linux3.5
chao@ubuntu:~/tiny4412/friendlyarm/linux-3.5$ mv tiny4412* arch/arm/configs/
chao@ubuntu:~/tiny4412/friendlyarm/linux-3.5$ make tiny4412_linux_defconfig
chao@ubuntu:~/tiny4412/friendlyarm/linux-3.5$ make zImage -j`nproc`

#将镜像拷贝到sd卡
chao@ubuntu:~/tiny4412/friendlyarm/linux-3.5$ sudo mount /dev/sdb1 /mnt
chao@ubuntu:~/tiny4412/friendlyarm/linux-3.5$ sudo cp arch/arm/boot/zImage /mnt
chao@ubuntu:~/tiny4412/friendlyarm/linux-3.5$ sync
chao@ubuntu:~/tiny4412/friendlyarm/linux-3.5$ sudo umount /mnt
```

### 准备rootfs

```shell
chao@ubuntu:~$ cd ${WORKDIR}/friendlyarm/busybox-1.17.2
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2$ make menuconfig
	Busybox Settings  --->
		Build Options  --->
			(arm-linux-)Cross Compiler prefix
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2$ make -j`nproc`
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2$ make install

# 丰富跟文件系统
# 将交叉编译器中的sys-root中的so拷贝到_install
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2$ cd _install
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ mkdir lib usr/lib
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ cp -rfd /opt/arm-none-linux-gnueabi-4.5.1/arm-none-linux-gnueabi/sys-root/lib/*so* lib
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ cp -rfd /opt/arm-none-linux-gnueabi-4.5.1/arm-none-linux-gnueabi/sys-root/usr/lib/*so* usr/lib
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ mdkir dev
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ sudo mknod console c 5 1
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ sudo mknod null c 1 3
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ mkdir etc etc/init.d etc/sysconfig
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ cd etc
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install/etc$ vi fstab
proc        /proc   proc    defaults    0   0
tmpfs       /tmp    tmpfs   defaults    0   0
sysfs       /sys    sysfs   defaults    0   0
tmpfs       /dev    tmpfs   defaults    0   0
var         /dev    tmpfs   defaults    0   0
ramfs       /dev    ramfs   defaults    0   0
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install/etc$ vi inittab
#/etc/inittab
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::restart:/sbin/init
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install/etc$ vi profile
# /etc/profile: system-wide .profile file for the Bourne shells
echo
echo -n "Processing /etc/profile... "
# no-op
echo "Done"
echo
USER="root"
LOGNAME=$USER
export HOSTNAME=`/bin/hostname`
export USER=root
export HOME=/root
export PS1="$USER@$HOSTNAME:\w\# "
PATH=/bin:/sbin:/usr/bin:/usr/sbin
LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH
ifconfig eth0 192.168.3.25
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install/etc$ vi etc/rcS
#!/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
runlevel=S
prevlevel=N
umask 022
export PATH runlevel prevlevel

mount -a
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
#mount -n -t usbfs none /proc/bus/usb
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
mkdir -p /var/lock

ifconfig lo 127.0.0.1

/bin/hostname -F /etc/sysconfig/HOSTNAME
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install/etc$ chmod +x init.d/rcS
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install/etc$ vi sysconfig/HOSTNAME
tiny4412
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install/etc$ cd ..
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ mkdir mnt opt proc sys tmp
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ sudo mount /dev/sdb2 /mnt
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ sudo cp -rfd * /mnt
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ sync
chao@ubuntu:~/tiny4412/friendlyarm/busybox-1.17.2/_install$ sudo umount /mnt
```

到此，准备工作已经完成，接下来进入uboot；

```shell
OK

U-Boot 2010.12 (Apr 05 2022 - 09:53:57) for TINY4412


CPU:    S5PC220 [Samsung SOC on SMP Platform Base on ARM CortexA9]
        APLL = 1400MHz, MPLL = 800MHz

Board:  TINY4412
DRAM:   1023 MiB

vdd_arm: 1.2
vdd_int: 1.0
vdd_mif: 1.1

BL1 version:  N/A (TrustZone Enabled BSP)


Checking Boot Mode ... SDMMC
REVISION: 1.1
MMC Device 0: 7580 MB
MMC Device 1: 3728 MB
MMC Device 2: N/A
Net:    No ethernet found.
Hit any key to stop autoboot:  0 
TINY4412 # print
baudrate=115200
bootargs=root=/dev/mmcblk0p2 rw rootfstype=ext4 console=ttySAC0,115200 init=/linuxrc earlyprintk
bootcmd=fatload mmc 0:1 40008000 zimage; bootm 40008000
bootdelay=3
ethaddr=00:40:5c:26:0a:5b
gatewayip=192.168.0.1
ipaddr=192.168.0.20
netmask=255.255.255.0
serverip=192.168.0.10

Environment size: 312/16380 bytes
TINY4412 # boot
```

修改uboot中的bootargs和bootcmd，即可启动。

```shell
Starting kernel ...

Uncompressing Linux... done, booting the kernel.
[    0.000000] Booting Linux on physical CPU 0
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Linux version 3.5.0-FriendlyARM (chao@ubuntu) (gcc version 4.8.3 20140320 (prerelease) (Sourcery CodeBench Lite 2014.05-29) ) #22
[    0.000000] CPU: ARMv7 Processor [413fc090] revision 0 (ARMv7), cr=10c5387d
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
[    0.000000] Machine: TINY4412
[    0.000000] cma: CMA: reserved 32 MiB at 6d800000
[    0.000000] Memory policy: ECC disabled, Data cache writealloc
[    0.000000] CPU EXYNOS4412 (id 0xe4412011)
[    0.000000] S3C24XX Clocks, Copyright 2004 Simtec Electronics
[    0.000000] s3c_register_clksrc: clock armclk has no registers set
[    0.000000] s3c_register_clksrc: clock audiocdclk has no registers set
[    0.000000] audiocdclk: no parent clock specified
[    0.000000] EXYNOS4: PLL settings, A=1400000000, M=800000000, E=96000000 V=108000000
[    0.000000] EXYNOS4: ARMCLK=1400000000, DMC=400000000, ACLK200=160000000
[    0.000000] ACLK100=100000000, ACLK160=160000000, ACLK133=133333333
[    0.000000] sclk_pwm: source is ext_xtal (0), rate is 24000000
[    0.000000] sclk_csis: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_csis: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_cam0: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_cam1: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_fimc: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_fimc: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_fimc: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_fimc: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_fimd: source is xusbxti (1), rate is 1500000
[    0.000000] sclk_jpeg: source is mout_jpeg0 (0), rate is 50000000
[    0.000000] sclk_fimg2d: source is mout_g2d0 (0), rate is 200000000
[    0.000000] sclk_g3d: source is mout_g3d0 (0), rate is 50000000
[    0.000000] sclk_mfc: source is mout_mfc0 (0), rate is 50000000
[    0.000000] PERCPU: Embedded 8 pages/cpu @c12fc000 s11712 r8192 d12864 u32768
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 251394
[    0.000000] Kernel command line: root=/dev/mmcblk0p2 rw rootfstype=ext4 console=ttySAC0,115200 init=/linuxrc earlyprintk
[    0.000000] PID hash table entries: 4096 (order: 2, 16384 bytes)
[    0.000000] Dentry cache hash table entries: 131072 (order: 7, 524288 bytes)
[    0.000000] Inode-cache hash table entries: 65536 (order: 6, 262144 bytes)
[    0.000000] Memory: 384MB 607MB = 991MB total
[    0.000000] Memory: 961552k/961552k available, 86000k reserved, 269312K highmem
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vector  : 0xffff0000 - 0xffff1000   (   4 kB)
[    0.000000]     fixmap  : 0xfff00000 - 0xfffe0000   ( 896 kB)
[    0.000000]     vmalloc : 0xf0000000 - 0xff000000   ( 240 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xef800000   ( 760 MB)
[    0.000000]     pkmap   : 0xbfe00000 - 0xc0000000   (   2 MB)
[    0.000000]     modules : 0xbf000000 - 0xbfe00000   (  14 MB)
[    0.000000]       .text : 0xc0008000 - 0xc089489c   (8755 kB)
[    0.000000]       .init : 0xc0895000 - 0xc08cbdc0   ( 220 kB)
[    0.000000]       .data : 0xc08cc000 - 0xc0981c00   ( 727 kB)
[    0.000000]        .bss : 0xc0981c24 - 0xc09f2cf0   ( 453 kB)
[    0.000000] SLUB: Genslabs=11, HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
[    0.000000] Preemptible hierarchical RCU implementation.
[    0.000000] NR_IRQS:549
[    0.000000] sched_clock: 32 bits at 200 Hz, resolution 5000000ns, wraps every 4294967291ms
[    0.000000] Console: colour dummy device 80x30
[    0.045000] Calibrating delay loop... 2795.11 BogoMIPS (lpj=6987776)
[    0.045000] pid_max: default: 32768 minimum: 301
[    0.045000] Mount-cache hash table entries: 512
[    0.045000] Initializing cgroup subsys debug
[    0.045000] Initializing cgroup subsys cpuacct
[    0.045000] Initializing cgroup subsys freezer
[    0.045000] CPU: Testing write buffer coherency: ok
[    0.045000] CPU0: thread -1, cpu 0, socket 10, mpidr 80000a00
[    0.045000] hw perfevents: enabled with ARMv7 Cortex-A9 PMU driver, 7 counters available
[    0.045000] Setting up static identity map for 0x4060f9f0 - 0x4060fa48
[    0.045000] L310 cache controller enabled
[    0.045000] l2x0: 16 ways, CACHE_ID 0x4100c4c8, AUX_CTRL 0x7e470001, Cache size: 1048576 B
[    0.100000] CPU1: thread -1, cpu 1, socket 10, mpidr 80000a01
[    0.140000] CPU2: thread -1, cpu 2, socket 10, mpidr 80000a02
[    0.180000] CPU3: thread -1, cpu 3, socket 10, mpidr 80000a03
[    0.180000] Brought up 4 CPUs
[    0.180000] SMP: Total of 4 processors activated (11180.44 BogoMIPS).
[    0.185000] NET: Registered protocol family 16
[    0.195000] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.195000] HW revision: 5
[    0.195000] samsung-adc-v3: no platform data supplied
[    0.200000] hw-breakpoint: found 5 (+1 reserved) breakpoint and 1 watchpoint registers.
[    0.200000] hw-breakpoint: maximum watchpoint size is 4 bytes.
[    0.200000] S3C Power Management, Copyright 2004 Simtec Electronics
[    0.200000] EXYNOS4x12 PMU Initialize
[    0.200000] EXYNOS: Initializing architecture
[    0.200000] s3c24xx-pwm s3c24xx-pwm.1: tin at 100000000, tdiv at 100000000, tin=divclk, base 8
[    0.200000] s3c24xx-pwm s3c24xx-pwm.0: tin at 100000000, tdiv at 100000000, tin=divclk, base 0
[    0.205000] bio: create slab <bio-0> at 0
[    0.205000] exynos_ion_heap_create: 34
[    0.205000] exynos_ion_heap_create: 34
[    0.205000] exynos_ion_heap_create: 34
[    0.205000] SCSI subsystem initialized
[    0.205000] usbcore: registered new interface driver usbfs
[    0.205000] usbcore: registered new interface driver hub
[    0.205000] usbcore: registered new device driver usb
[    0.205000] s3c-i2c s3c2440-i2c.0: slave address 0x10
[    0.205000] s3c-i2c s3c2440-i2c.0: bus frequency set to 195 KHz
[    0.205000] s3c-i2c s3c2440-i2c.0: i2c-0: S3C I2C adapter
[    0.205000] s3c-i2c s3c2440-i2c.1: slave address 0x10
[    0.205000] s3c-i2c s3c2440-i2c.1: bus frequency set to 195 KHz
[    0.205000] s3c-i2c s3c2440-i2c.1: i2c-1: S3C I2C adapter
[    0.205000] s3c-i2c s3c2440-i2c.2: slave address 0x10
[    0.205000] s3c-i2c s3c2440-i2c.2: bus frequency set to 97 KHz
[    0.205000] s3c-i2c s3c2440-i2c.2: i2c-2: S3C I2C adapter
[    0.205000] s3c-i2c s3c2440-i2c.3: slave address 0x10
[    0.205000] s3c-i2c s3c2440-i2c.3: bus frequency set to 195 KHz
[    0.205000] s3c-i2c s3c2440-i2c.3: i2c-3: S3C I2C adapter
[    0.205000] s3c-i2c s3c2440-i2c.7: slave address 0x10
[    0.205000] s3c-i2c s3c2440-i2c.7: bus frequency set to 195 KHz
[    0.205000] s3c-i2c s3c2440-i2c.7: i2c-7: S3C I2C adapter
[    0.205000] s3c-i2c s3c2440-hdmiphy-i2c: slave address 0x10
[    0.205000] s3c-i2c s3c2440-hdmiphy-i2c: bus frequency set to 390 KHz
[    0.205000] s3c-i2c s3c2440-hdmiphy-i2c: i2c-8: S3C I2C adapter
[    0.205000] Linux media interface: v0.10
[    0.205000] Linux video capture interface: v2.00
[    0.205000] Advanced Linux Sound Architecture Driver Version 1.0.25.
[    0.205000] Bluetooth: Core ver 2.16
[    0.205000] NET: Registered protocol family 31
[    0.205000] Bluetooth: HCI device and connection manager initialized
[    0.205000] Bluetooth: HCI socket layer initialized
[    0.205000] Bluetooth: L2CAP socket layer initialized
[    0.205000] Bluetooth: SCO socket layer initialized
[    0.205000] Switching to clocksource mct-frc
[    0.210000] NET: Registered protocol family 2
[    0.210000] IP route cache hash table entries: 32768 (order: 5, 131072 bytes)
[    0.210000] TCP established hash table entries: 131072 (order: 8, 1048576 bytes)
[    0.210000] TCP bind hash table entries: 65536 (order: 7, 786432 bytes)
[    0.215000] TCP: Hash tables configured (established 131072 bind 65536)
[    0.215000] TCP: reno registered
[    0.215000] UDP hash table entries: 512 (order: 2, 16384 bytes)
[    0.215000] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes)
[    0.215000] NET: Registered protocol family 1
[    0.215000] RPC: Registered named UNIX socket transport module.
[    0.215000] RPC: Registered udp transport module.
[    0.215000] RPC: Registered tcp transport module.
[    0.215000] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.215000] NetWinder Floating Point Emulator V0.97 (double precision)
[    0.215000] Attached IOMMU controller to exynos4-fimc.0 device.
[    0.215000] Attached IOMMU controller to exynos4-fimc.1 device.
[    0.215000] Attached IOMMU controller to exynos4-fimc.2 device.
[    0.215000] Attached IOMMU controller to exynos4-fimc.3 device.
[    0.215000] Attached IOMMU controller to s5p-mfc-l device.
[    0.215000] Attached IOMMU controller to s5p-mfc-r device.
[    0.215000] Attached IOMMU controller to s5p-mixer device.
[    0.215000] Attached IOMMU controller to s5p-fimg2d device.
[    0.215000] Attached IOMMU controller to s5p-jpeg.0 device.
[    0.215000] platform exynos-fimc-lite.0: No SYSMMU found
[    0.215000] Attached IOMMU controller to exynos-fimc-lite.0 device.
[    0.215000] platform exynos-fimc-lite.1: No SYSMMU found
[    0.215000] Attached IOMMU controller to exynos-fimc-lite.1 device.
[    0.215000] s3c-adc samsung-adc-v4: attached adc driver
[    0.215000] Failed to declare coherent memory for MFC device (0 bytes at 0x43000000)
[    0.215000] highmem bounce pool size: 64 pages
[    0.220000] NFS: Registering the id_resolver key type
[    0.220000] Key type id_resolver registered
[    0.220000] NTFS driver 2.1.30 [Flags: R/W].
[    0.220000] fuse init (API version 7.19)
[    0.220000] msgmni has been set to 1416
[    0.220000] io scheduler noop registered
[    0.220000] io scheduler deadline registered
[    0.220000] io scheduler cfq registered (default)
[    0.260000] s3c-fb exynos4-fb.0: window 0: fb 
[    0.295000] s3c-fb exynos4-fb.0: window 1: fb 
[    0.330000] s3c-fb exynos4-fb.0: window 2: fb 
[    0.370000] s3c-fb exynos4-fb.0: window 3: fb 
[    0.410000] s3c-fb exynos4-fb.0: window 4: fb 
[    0.410000] Start display and show logo
[    0.415000] dma-pl330 dma-pl330.0: Loaded driver for PL330 DMAC-267056
[    0.415000] dma-pl330 dma-pl330.0:   DBUFF-32x4bytes Num_Chans-8 Num_Peri-32 Num_Events-32
[    0.415000] dma-pl330 dma-pl330.1: Loaded driver for PL330 DMAC-267056
[    0.415000] dma-pl330 dma-pl330.1:   DBUFF-32x4bytes Num_Chans-8 Num_Peri-32 Num_Events-32
[    0.420000] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    0.420000] exynos4210-uart.0: ttySAC0 at MMIO 0x13800000 (irq = 84) is a S3C6400/10
[    1.340000] console [ttySAC0] enabled
[    1.345000] exynos4210-uart.1: ttySAC1 at MMIO 0x13810000 (irq = 85) is a S3C6400/10
[    1.350000] exynos4210-uart.2: ttySAC2 at MMIO 0x13820000 (irq = 86) is a S3C6400/10
[    1.360000] exynos4210-uart.3: ttySAC3 at MMIO 0x13830000 (irq = 87) is a S3C6400/10
[    1.365000] leds     initialized
[    1.370000] buttons  initialized
[    1.370000] pwm      initialized
[    1.375000] backlight        initialized
[    1.380000] tiny4412-adc     initialized
[    1.385000] Mali: init_mali_clock mali_clock c08f8024 at 440 MHz
[    1.390000] Mali: Mali device driver loaded
[    1.395000] brd: module loaded
[    1.395000] loop: module loaded
[    1.400000] tun: Universal TUN/TAP device driver, 1.6
[    1.405000] tun: (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
[    1.410000] PPP generic driver version 2.4.2
[    1.415000] PPP BSD Compression module registered
[    1.420000] PPP Deflate Compression module registered
[    1.425000] PPP MPPE Compression module registered
[    1.430000] NET: Registered protocol family 24
[    1.430000] pegasus: v0.6.14 (2006/09/27), Pegasus/Pegasus II USB Ethernet driver
[    1.440000] usbcore: registered new interface driver pegasus
[    1.445000] usbcore: registered new interface driver asix
[    1.450000] usbcore: registered new interface driver cdc_ether
[    1.455000] usbcore: registered new interface driver dm9601
[    1.460000] usbcore: registered new interface driver dm9620
[    1.470000] usbcore: registered new interface driver net1080
[    1.475000] usbcore: registered new interface driver cdc_subset
[    1.480000] usbcore: registered new interface driver zaurus
[    1.485000] usbcore: registered new interface driver cdc_ncm
[    1.490000] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    1.500000] s5p-ehci s5p-ehci: S5P EHCI Host Controller
[    1.500000] s5p-ehci s5p-ehci: new USB bus registered, assigned bus number 1
[    1.510000] s5p-ehci s5p-ehci: irq 102, io mem 0x12580000
[    1.525000] s5p-ehci s5p-ehci: USB 0.0 started, EHCI 1.00
[    1.525000] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
[    1.525000] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.535000] usb usb1: Product: S5P EHCI Host Controller
[    1.540000] usb usb1: Manufacturer: Linux 3.5.0-FriendlyARM ehci_hcd
[    1.545000] usb usb1: SerialNumber: s5p-ehci
[    1.550000] hub 1-0:1.0: USB hub found
[    1.555000] hub 1-0:1.0: 3 ports detected
[    1.555000] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    1.565000] exynos-ohci exynos-ohci: PHY already ON
[    1.570000] exynos-ohci exynos-ohci: EXYNOS OHCI Host Controller
[    1.575000] exynos-ohci exynos-ohci: new USB bus registered, assigned bus number 2
[    1.580000] exynos-ohci exynos-ohci: irq 102, io mem 0x12590000
[    1.645000] usb usb2: New USB device found, idVendor=1d6b, idProduct=0001
[    1.645000] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.645000] usb usb2: Product: EXYNOS OHCI Host Controller
[    1.645000] usb usb2: Manufacturer: Linux 3.5.0-FriendlyARM ohci_hcd
[    1.650000] usb usb2: SerialNumber: exynos-ohci
[    1.655000] hub 2-0:1.0: USB hub found
[    1.660000] hub 2-0:1.0: 3 ports detected
[    1.660000] Initializing USB Mass Storage driver...
[    1.665000] usbcore: registered new interface driver usb-storage
[    1.675000] USB Mass Storage support registered.
[    1.680000] usbcore: registered new interface driver usbserial
[    1.685000] usbcore: registered new interface driver usbserial_generic
[    1.690000] USB Serial support registered for generic
[    1.695000] usbserial: USB Serial Driver core
[    1.700000] usbcore: registered new interface driver aircable
[    1.705000] USB Serial support registered for aircable
[    1.710000] usbcore: registered new interface driver ark3116
[    1.715000] USB Serial support registered for ark3116
[    1.720000] usbcore: registered new interface driver belkin_sa
[    1.725000] USB Serial support registered for Belkin / Peracom / GoHubs USB Serial Adapter
[    1.735000] usbcore: registered new interface driver ch341
[    1.740000] USB Serial support registered for ch341-uart
[    1.745000] usbcore: registered new interface driver cp210x
[    1.750000] USB Serial support registered for cp210x
[    1.755000] usbcore: registered new interface driver cyberjack
[    1.760000] USB Serial support registered for Reiner SCT Cyberjack USB card reader
[    1.770000] usbcore: registered new interface driver cypress_m8
[    1.775000] USB Serial support registered for DeLorme Earthmate USB
[    1.780000] USB Serial support registered for HID->COM RS232 Adapter
[    1.790000] USB Serial support registered for Nokia CA-42 V2 Adapter
[    1.795000] usbcore: registered new interface driver digi_acceleport
[    1.800000] USB Serial support registered for Digi 2 port USB adapter
[    1.805000] USB Serial support registered for Digi 4 port USB adapter
[    1.815000] usbcore: registered new interface driver io_edgeport
[    1.820000] USB Serial support registered for Edgeport 2 port adapter
[    1.825000] USB Serial support registered for Edgeport 4 port adapter
[    1.835000] USB Serial support registered for Edgeport 8 port adapter
[    1.840000] USB Serial support registered for EPiC device
[    1.845000] usbcore: registered new interface driver io_ti
[    1.850000] USB Serial support registered for Edgeport TI 1 port adapter
[    1.855000] USB Serial support registered for Edgeport TI 2 port adapter
[    1.865000] usb 1-2: new high-speed USB device number 2 using s5p-ehci
[    1.870000] usbcore: registered new interface driver empeg
[    1.875000] USB Serial support registered for empeg
[    1.880000] usbcore: registered new interface driver f81232
[    1.885000] USB Serial support registered for f81232
[    1.890000] usbcore: registered new interface driver ftdi_sio
[    1.895000] USB Serial support registered for FTDI USB Serial Device
[    1.905000] ftdi_sio: v1.6.0:USB FTDI Serial Converters Driver
[    1.910000] usbcore: registered new interface driver funsoft
[    1.915000] USB Serial support registered for funsoft
[    1.920000] usbcore: registered new interface driver garmin_gps
[    1.925000] USB Serial support registered for Garmin GPS usb/tty
[    1.930000] usbcore: registered new interface driver hp4x
[    1.935000] USB Serial support registered for hp4X
[    1.940000] usbcore: registered new interface driver ipaq
[    1.945000] USB Serial support registered for PocketPC PDA
[    1.950000] usbcore: registered new interface driver ipw
[    1.955000] USB Serial support registered for IPWireless converter
[    1.965000] usbcore: registered new interface driver ir_usb
[    1.970000] USB Serial support registered for IR Dongle
[    1.975000] ir_usb: v0.5:USB IR Dongle driver
[    1.980000] usbcore: registered new interface driver iuu_phoenix
[    1.985000] USB Serial support registered for iuu_phoenix
[    1.990000] usbcore: registered new interface driver keyspan
[    1.995000] USB Serial support registered for Keyspan - (without firmware)
[    2.000000] USB Serial support registered for Keyspan 1 port adapter
[    2.010000] USB Serial support registered for Keyspan 2 port adapter
[    2.015000] USB Serial support registered for Keyspan 4 port adapter
[    2.020000] usbcore: registered new interface driver keyspan_pda
[    2.025000] usb 1-2: New USB device found, idVendor=0424, idProduct=4604
[    2.025000] USB Serial support registered for Keyspan PDA
[    2.025000] USB Serial support registered for Keyspan PDA - (prerenumeration)
[    2.025000] USB Serial support registered for Xircom / Entregra PGS - (prerenumeration)
[    2.025000] usbcore: registered new interface driver kl5kusb105
[    2.025000] USB Serial support registered for KL5KUSB105D / PalmConnect
[    2.030000] usbcore: registered new interface driver kobil_sct
[    2.030000] USB Serial support registered for KOBIL USB smart card terminal
[    2.030000] usbcore: registered new interface driver mct_u232
[    2.030000] USB Serial support registered for MCT U232
[    2.030000] usbcore: registered new interface driver metro_usb
[    2.030000] USB Serial support registered for Metrologic USB to Serial
[    2.030000] usbcore: registered new interface driver mos7720
[    2.030000] USB Serial support registered for Moschip 2 port adapter
[    2.030000] usbcore: registered new interface driver mos7840
[    2.030000] USB Serial support registered for Moschip 7840/7820 USB Serial Driver
[    2.030000] usbcore: registered new interface driver moto_modem
[    2.030000] USB Serial support registered for moto-modem
[    2.030000] usbcore: registered new interface driver navman
[    2.030000] USB Serial support registered for navman
[    2.030000] usbcore: registered new interface driver omninet
[    2.030000] USB Serial support registered for ZyXEL - omni.net lcd plus usb
[    2.030000] usbcore: registered new interface driver opticon
[    2.030000] USB Serial support registered for opticon
[    2.030000] usbcore: registered new interface driver option
[    2.030000] USB Serial support registered for GSM modem (1-port)
[    2.030000] usbcore: registered new interface driver oti6858
[    2.030000] USB Serial support registered for oti6858
[    2.030000] usbcore: registered new interface driver pl2303
[    2.030000] USB Serial support registered for pl2303
[    2.030000] usbcore: registered new interface driver qcaux
[    2.030000] USB Serial support registered for qcaux
[    2.030000] usbcore: registered new interface driver qcserial
[    2.030000] USB Serial support registered for Qualcomm USB modem
[    2.030000] usbcore: registered new interface driver quatech2
[    2.030000] USB Serial support registered for Quatech 2nd gen USB to Serial Driver
[    2.030000] safe_serial: v0.1:USB Safe Encapsulated Serial
[    2.030000] usbcore: registered new interface driver safe_serial
[    2.030000] USB Serial support registered for safe_serial
[    2.030000] usbcore: registered new interface driver siemens_mpi
[    2.030000] USB Serial support registered for siemens_mpi
[    2.030000] usbcore: registered new interface driver sierra
[    2.030000] USB Serial support registered for Sierra USB modem
[    2.030000] usbcore: registered new interface driver spcp8x5
[    2.030000] USB Serial support registered for SPCP8x5
[    2.030000] usbcore: registered new interface driver ssu100
[    2.030000] USB Serial support registered for Quatech SSU-100 USB to Serial Driver
[    2.030000] usbcore: registered new interface driver symbolserial
[    2.030000] USB Serial support registered for symbol
[    2.030000] usbcore: registered new interface driver ti_usb_3410_5052
[    2.030000] USB Serial support registered for TI USB 3410 1 port adapter
[    2.030000] USB Serial support registered for TI USB 5052 2 port adapter
[    2.030000] ti_usb_3410_5052: v0.10:TI USB 3410/5052 Serial Driver
[    2.030000] usbcore: registered new interface driver visor
[    2.030000] USB Serial support registered for Handspring Visor / Palm OS
[    2.030000] USB Serial support registered for Sony Clie 5.0
[    2.030000] USB Serial support registered for Sony Clie 3.5
[    2.030000] usbcore: registered new interface driver whiteheat
[    2.030000] USB Serial support registered for Connect Tech - WhiteHEAT - (prerenumeration)
[    2.030000] USB Serial support registered for Connect Tech - WhiteHEAT
[    2.030000] usbcore: registered new interface driver vivopay_serial
[    2.030000] USB Serial support registered for vivopay-serial
[    2.030000] usbcore: registered new interface driver zio
[    2.030000] USB Serial support registered for zio
[    2.030000] s3c-hsotg s3c-hsotg: regs f0c40000, irq 103
[    2.030000] s3c-hsotg s3c-hsotg: PHY already ON
[    2.030000] s3c-hsotg s3c-hsotg: EPs:15
[    2.030000] s3c-hsotg s3c-hsotg: dedicated fifos
[    2.030000] s3c-hsotg s3c-hsotg: still being used
[    2.030000] file system registered
[    2.030000]  gadget: Mass Storage Function, version: 2009/09/11
[    2.030000]  gadget: Number of LUNs=1
[    2.030000]  lun0: LUN: removable file: (no medium)
[    2.030000]  gadget: android_usb ready
[    2.030000] s3c-hsotg s3c-hsotg: bound driver android_usb
[    2.030000] s3c-hsotg s3c-hsotg: PHY already ON
[    2.035000] s3c-hsotg s3c-hsotg: s3c_hsotg_irq: USBRst
[    2.035000] s3c-hsotg s3c-hsotg: still being used
[    2.035000] mousedev: PS/2 mouse device common for all mice
[    2.035000] usbcore: registered new interface driver xpad
[    2.035000] input: ft5x0x_ts as /devices/virtual/input/input0
[    2.045000] read reg (0xa6) error, -6
[    2.045000] ft5x0x_ts 1-0038: chip not found
[    2.495000] usb 1-2: New USB device strings: Mfr=0, Product=0, SerialNumber=0
[    2.505000] hub 1-2:1.0: USB hub found
[    2.505000] hub 1-2:1.0: 5 ports detected
[    2.510000] ft5x0x_ts 1-0038: probe ft5x0x TouchScreen failed, -6
[    2.515000] touchscreen-1wire        initialized
[    2.520000] backlight-1wire  initialized
[    2.525000] setup_irq: ret = 0
[    2.530000] PWM clock = 100000000
[    2.530000] TCNT_FOR_SAMPLE_BIT = 650, TCFG1 = 00010004
[    2.535000] input: fa_ts_input as /devices/virtual/input/input1
[    2.540000] ts-if    initialized
[    2.545000] s3c-rtc s3c64xx-rtc: rtc disabled, re-enabling
[    2.550000] s3c-rtc s3c64xx-rtc: rtc core: registered s3c as rtc0
[    2.555000] i2c /dev entries driver
[    2.560000] gspca_main: v2.14.0 registered
[    2.565000] usbcore: registered new interface driver benq
[    2.570000] usbcore: registered new interface driver conex
[    2.575000] usbcore: registered new interface driver cpia1
[    2.580000] usbcore: registered new interface driver etoms
[    2.585000] 4: TCNTB=0000028a, TCNTO=0000019f, TINT_CSTAT=00000008
[    2.590000] usbcore: registered new interface driver finepix
[    2.600000] usbcore: registered new interface driver jeilinj
[    2.605000] usbcore: registered new interface driver jl2005bcd
[    2.610000] usbcore: registered new interface driver kinect
[    2.615000] usbcore: registered new interface driver konica
[    2.620000] usbcore: registered new interface driver mars
[    2.625000] usbcore: registered new interface driver mr97310a
[    2.630000] usbcore: registered new interface driver nw80x
[    2.635000] 4: TCNTB=0000028a, TCNTO=0000013c, TINT_CSTAT=00000008
[    2.645000] usbcore: registered new interface driver ov519
[    2.650000] usbcore: registered new interface driver ov534
[    2.655000] usbcore: registered new interface driver ov534_9
[    2.660000] usbcore: registered new interface driver pac207
[    2.665000] usbcore: registered new interface driver gspca_pac7302
[    2.670000] usbcore: registered new interface driver pac7311
[    2.675000] usbcore: registered new interface driver se401
[    2.680000] usbcore: registered new interface driver sn9c2028
[    2.690000] usbcore: registered new interface driver gspca_sn9c20x
[    2.695000] usbcore: registered new interface driver sonixb
[    2.700000] usbcore: registered new interface driver sonixj
[    2.705000] usbcore: registered new interface driver spca500
[    2.710000] usbcore: registered new interface driver spca501
[    2.715000] usbcore: registered new interface driver spca505
[    2.720000] usbcore: registered new interface driver spca506
[    2.730000] usbcore: registered new interface driver spca508
[    2.735000] usbcore: registered new interface driver spca561
[    2.740000] usbcore: registered new interface driver spca1528
[    2.745000] usbcore: registered new interface driver sq905
[    2.750000] usbcore: registered new interface driver sq905c
[    2.755000] usbcore: registered new interface driver sq930x
[    2.760000] usbcore: registered new interface driver sunplus
[    2.765000] usbcore: registered new interface driver stk014
[    2.775000] usbcore: registered new interface driver stv0680
[    2.780000] usbcore: registered new interface driver t613
[    2.785000] usbcore: registered new interface driver gspca_topro
[    2.790000] usbcore: registered new interface driver tv8532
[    2.795000] usbcore: registered new interface driver vc032x
[    2.800000] usbcore: registered new interface driver vicam
[    2.805000] usbcore: registered new interface driver xirlink-cit
[    2.810000] usbcore: registered new interface driver gspca_zc3xx
[    2.820000] usbcore: registered new interface driver ALi m5602
[    2.825000] usbcore: registered new interface driver STV06xx
[    2.830000] usbcore: registered new interface driver gspca_gl860
[    2.855000] usb 1-2.4: new high-speed USB device number 3 using s5p-ehci
[    2.865000] FIMC-IS probe completed
[    2.865000] [INFO]flite_probe:759: fimc-lite0 probe success
[    2.865000] [INFO]flite_probe:759: fimc-lite1 probe success
[    2.865000] s5p-fimc-md: Registered fimc.0.m2m as /dev/video0
[    2.865000] s5p-fimc-md: Registered fimc.0.capture as /dev/video1
[    2.870000] s5p-fimc-md: Registered fimc.1.m2m as /dev/video2
[    2.875000] s5p-fimc-md: Registered fimc.1.capture as /dev/video3
[    2.880000] s5p-fimc-md: Registered fimc.2.m2m as /dev/video4
[    2.890000] s5p-fimc-md: Registered fimc.2.capture as /dev/video5
[    2.895000] s5p-fimc-md: Registered fimc.3.m2m as /dev/video6
[    2.900000] s5p-fimc-md: Registered fimc.3.capture as /dev/video7
[    2.905000] s5p-mfc s5p-mfc: decoder registered as /dev/video8
[    2.910000] s5p-mfc s5p-mfc: encoder registered as /dev/video9
[    2.915000] HDMI unplugged
[    2.920000] s5p-hdmiphy 8-0038: probe successful
[    2.925000] s5p-hdmi exynos4-hdmi: probe successful
[    2.930000] i2c i2c-7: attached s5p_ddc into i2c adapter successfully
[    2.935000] i2c-core: driver [s5p_ddc] using legacy suspend method
[    2.940000] i2c-core: driver [s5p_ddc] using legacy resume method
[    2.950000] Samsung TV Mixer driver, (c) 2010-2011 Samsung Electronics Co., Ltd.
[    2.950000] usb 1-2.4: config 1 interface 0 altsetting 0 endpoint 0x83 has an invalid bInterval 0, changing to 9
[    2.965000] s5p-mixer s5p-mixer: probe start
[    2.970000] s5p-mixer s5p-mixer: resources acquired
[    2.975000] s5p-mixer s5p-mixer: added output 'S5P HDMI connector' from module 's5p-hdmi'
[    2.985000] s5p-mixer s5p-mixer: module s5p-sdo is missing
[    2.990000] s5p-mixer s5p-mixer: registered layer graph0 as /dev/video10
[    2.995000] s5p-mixer s5p-mixer: registered layer graph1 as /dev/video11
[    3.000000] s5p-mixer s5p-mixer: registered layer video0 as /dev/video12
[    3.010000] s5p-mixer s5p-mixer: probe successful
[    3.015000] Initialize JPEG driver
[    3.015000] s5p-jpeg s5p-jpeg.0: JPEG driver is registered to /dev/video13
[    3.020000] usb 1-2.4: New USB device found, idVendor=0000, idProduct=9621
[    3.020000] usb 1-2.4: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[    3.035000] Exynos Graphics 2D driver, (c) 2011 Samsung Electronics
[    3.045000] [fimg2d_probe] base address: 0x10800000
[    3.050000] [fimg2d_probe] irq: 121
[    3.050000] [fimg2d_clk_setup] parent clk: mout_g2d0
[    3.055000] [fimg2d_clk_setup] sclk: sclk_fimg2d
[    3.060000] [fimg2d_clk_setup] clkrate: 200000000 parent clkrate: 800000000
[    3.070000] [fimg2d_clk_setup] gate clk: fimg2d
[    3.075000] [fimg2d_probe] enable runtime pm
[    3.075000] [fimg2d_probe] sysmmu disabled for fimg2d
[    3.080000] usbcore: registered new interface driver uvcvideo
[    3.090000] USB Video Class driver (1.1.1)
[    3.090000] samsung-fake-battery samsung-fake-battery: samsung_fake_bat_probe
[    3.105000] usb 1-2.5: new high-speed USB device number 4 using s5p-ehci
[    3.145000] input: mma7660 as /devices/platform/s3c2440-i2c.3/i2c-3/3-004c/input/input2
[    3.145000] mma7660 3-004c: MMA7660 device is probed successfully.
[    3.145000] i2c-core: driver [mma7660] using legacy suspend method
[    3.145000] i2c-core: driver [mma7660] using legacy resume method
[    3.150000] MMA7660 sensor driver registered.
[    3.155000] Exynos: Kernel Thermal management registered
[    3.160000] s3c2410_wdt: S3C2410 Watchdog Timer, (c) 2004 Simtec Electronics
[    3.165000] s3c2410-wdt s3c2410-wdt: watchdog inactive, reset disabled, irq disabled
[    3.175000] device-mapper: uevent: version 1.0.3
[    3.180000] device-mapper: ioctl: 4.22.0-ioctl (2011-10-19) initialised: dm-devel@redhat.com
[    3.185000] Bluetooth: Virtual HCI driver ver 1.3
[    3.190000] Front
[    3.195000] Bluetooth: HCI UART driver ver 2.2
[    3.200000] Bluetooth: HCI H4 protocol initialized
[    3.205000] Bluetooth: HCI BCSP protocol initialized
[    3.210000] Bluetooth: HCILL protocol initialized
[    3.215000] usbcore: registered new interface driver bcm203x
[    3.220000] usbcore: registered new interface driver bpa10x
[    3.225000] usbcore: registered new interface driver bfusb
[    3.230000] usbcore: registered new interface driver btusb
[    3.235000] cpuidle: using governor ladder
[    3.240000] usb 1-2.5: New USB device found, idVendor=0424, idProduct=2530
[    3.245000] usb 1-2.5: New USB device strings: Mfr=0, Product=2, SerialNumber=0
[    3.245000] cpuidle: using governor menu
[    3.245000] sdhci: Secure Digital Host Controller Interface driver
[    3.245000] sdhci: Copyright(c) Pierre Ossman
[    3.245000] Synopsys Designware Multimedia Card Interface Driver[    3.265000] usb 1-2.5: Product: Bridge device

[    3.280000] dw_mmc dw_mmc: Using internal DMA controller.
[    3.315000] mmc_host mmc0: Bus speed (slot 0) = 50000000Hz (slot req 400000Hz, actual 396825HZ div = 63)
[    3.315000] dw_mmc dw_mmc: Version ID is 240a
[    3.315000] dw_mmc dw_mmc: DW MMC controller at irq 109, 32 bit host data width, 128 deep fifo
[    3.315000] s3c-sdhci exynos4-sdhci.2: clock source 2: mmc_busclk.2 (100000000 Hz)
[    3.355000] mmc1: SDHCI controller on samsung-hsmmc [exynos4-sdhci.2] using ADMA
[    3.355000] s3c-sdhci exynos4-sdhci.3: clock source 2: mmc_busclk.2 (100000000 Hz)
[    3.360000] mmc_host mmc0: Bus speed (slot 0) = 50000000Hz (slot req 52000000Hz, actual 50000000HZ div = 0)
[    3.360000] mmc_host mmc0: Bus speed (slot 0) = 100000000Hz (slot req 52000000Hz, actual 50000000HZ div = 1)
[    3.365000] mmc0: new high speed DDR MMC card at address 0001
[    3.370000] mmcblk1: mmc0:0001 4YMD3R 3.64 GiB 
[    3.375000] mmcblk1boot0: mmc0:0001 4YMD3R partition 1 4.00 MiB
[    3.380000] mmcblk1boot1: mmc0:0001 4YMD3R partition 2 4.00 MiB
[    3.385000] mmc2: SDHCI controller on samsung-hsmmc [exynos4-sdhci.3] using ADMA
[    3.385000] usbcore: registered new interface driver usbhid
[    3.385000] usbhid: USB HID core driver
[    3.385000] Samsung Audio Subsystem Driver, (c) 2011 Samsung Electronics
[    3.385000] audss_init: RCLK SRC[busclk]
[    3.385000] Samsung SRP driver, (c)2011 Samsung Electronics
[    3.420000]  mmcblk1: p1 p2 p3 p4
[    3.425000] GACT probability NOT on
[    3.425000]  mmcblk1boot1: unknown partition table
[    3.430000] Mirror/redirect action on
[    3.430000]  mmcblk1boot0: unknown partition table
[    3.440000] u32 classifier
[    3.440000]     Actions configured
[    3.445000] Netfilter messages via NETLINK v0.30.
[    3.450000] nf_conntrack version 0.5.0 (15536 buckets, 62144 max)
[    3.455000] ctnetlink v0.93: registering with nfnetlink.
[    3.460000] NF_TPROXY: Transparent proxy support initialized, version 4.1.0
[    3.470000] NF_TPROXY: Copyright (c) 2006-2007 BalaBit IT Ltd.
[    3.475000] xt_time: kernel timezone is -0000
[    3.480000] ip_tables: (C) 2000-2006 Netfilter Core Team
[    3.485000] arp_tables: (C) 2002 David S. Miller
[    3.490000] TCP: cubic registered
[    3.490000] Initializing XFRM netlink socket
[    3.495000] NET: Registered protocol family 10
[    3.500000] mip6: Mobile IPv6
[    3.505000] ip6_tables: (C) 2000-2006 Netfilter Core Team
[    3.505000] mmc1: new high speed SDHC card at address e624
[    3.505000] mmcblk0: mmc1:e624 SU08G 7.40 GiB 
[    3.505000]  mmcblk0: p1 p2 p3
[    3.520000] sit: IPv6 over IPv4 tunneling driver
[    3.525000] NET: Registered protocol family 17
[    3.530000] NET: Registered protocol family 15
[    3.535000] Bluetooth: RFCOMM TTY layer initialized
[    3.540000] Bluetooth: RFCOMM socket layer initialized
[    3.545000] Bluetooth: RFCOMM ver 1.11
[    3.550000] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[    3.555000] Bluetooth: BNEP filters: protocol multicast
[    3.560000] Bluetooth: HIDP (Human Interface Emulation) ver 1.2
[    3.565000] NET: Registered protocol family 35
[    3.570000] Key type dns_resolver registered
[    3.575000] EXYNOS4X12: Adaptive Support Voltage init
[    3.580000] EXYNOS4X12(SG): ORIG : 3 MOD : 0 RESULT : 3
[    3.585000] EXYNOS: Fail to get HPM Value
[    3.590000] VFP support v0.3: implementor 41 architecture 3 part 30 variant 9 rev 4
[    3.595000] ThumbEE CPU extension supported.
[    3.600000] Registering SWP/SWPB emulation handler
[    3.605000] [ 6 ] Memory Type Undertermined.
[    3.610000] DVFS : VDD_INT Voltage table set with 0 Group
[    3.640000] Failed to init busfreq.
[    3.640000] s3c-rtc s3c64xx-rtc: setting system clock to 2013-01-01 12:01:01 UTC (1357041661)
[    3.640000] hotplug_policy_init: intialised with policy : DVFS_NR_BASED_HOTPLUG
[    3.640000] ALSA device list:
[    3.640000]   No soundcards found.
[    3.685000] EXT4-fs (mmcblk0p2): recovery complete
[    3.685000] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Opts: (null)
[    3.685000] VFS: Mounted root (ext4 filesystem) on device 179:26.
[    3.685000] Freeing init memory: 216K

Please press Enter to activate this console. 

Processing /etc/profile... Done

ifconfig: SIOCSIFADDR: No such device
root@tiny4412:/#
```

