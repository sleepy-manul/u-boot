.. SPDX-License-Identifier: GPL-2.0+
.. Copyright (C) 2024 Andreas Buschka <kontakt@andreas-buschka.de>

Firefly Boards (based on the Rockchip RK3588)

About this
----------

In this document, additional documentation on how the (T-)Firefly's Linux SDK (based on U-Boot 2017.09) was used to get
U-Boot 2024.04 working. Before starting, reading the doc/board/rockchip/rockchip.rst is highly recommended to get a
basic idea about the specific boot process.

Structure of the FIT Image
--------------------------

The FIT Image contains the following components:

+-----------+---------------------------+---------------+--------------------+
| FIT index | type                      | load address  | size in bytes      |
|           |                           | / entry point | (same info in hex) |
+===========+===========================+===============+====================+
| 0 (new)   | U-Boot                    | 0x00a00000    | ca. 1-2 MiB        |
| 0 (2017)  | U-Boot                    | 0x00200000    | ca. 1.5 MiB        |
+-----------+---------------------------+---------------+--------------------+
|     1     | ARM Trusted Firmware (1)  | 0x00040000    | 200008 (0x30d48)   |
+-----------+---------------------------+---------------+--------------------+
|     2     | ARM Trusted Firmware (2)  | 0x000f0000    | 24576 (0x6000)     |
+-----------+---------------------------+---------------+--------------------+
|     3     | ARM Trusted Firmware (3)  | 0xff100000    | 24576 (0x6000)     |
+-----------+---------------------------+---------------+--------------------+
|     4     | OP-TEE                    | 0x08400000    | 465312 (0x719a0)   |
+-----------+---------------------------+---------------+--------------------+
|     5     | Flat Device Tree          | ??????????    | ca. 500-700 KiB    |
+-----------+---------------------------+---------------+--------------------+

External ingredients
--------------------

Currently (2024-04), the following components are not available as Open Source, so we must fetch them from the
Rockchip repo at https://github.com/rockchip-linux/rkbin/tree/master/bin/rk35 .
These files get updated occasionally, so it _should_ not hurt to get the newest versions of everything. But be careful
that the files you download always begin with "rk3588_"; NEVER mix with "rk3568_" or any other prefixes.

+-----------------------------------------------+--------------------------------------------------------------------+
| File                                          | Purpose                                                            |
+-----------------------------------------------+--------------------------------------------------------------------+
| rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.16.bin  | Tertiary Boot Loader (TPL). Initialises the DRAM (=the main RAM    |
|                                               | outside the CPU package). While the U-Boot SPL runs from SRAM, it  |
|                                               | cannot proceed to U-Boot Proper without having access to DRAM.     |
|                                               | Path and filename must be passed to the build process              |
|                                               | in the ROCKCHIP_TPL shell environment variable.                    |
+-----------------------------------------------+--------------------------------------------------------------------+
| rk3588_bl31_v1.45.elf                         | ARM Trusted Firmware (a.k.a. Bootloader step 3.1, BL31)            |
|                                               | Path and filename must be passed using the BL31 variable.          |
|                                               | If you want to rename this file, the ending MUST stay ".elf".      |
+-----------------------------------------------+--------------------------------------------------------------------+
| rk3588_bl32_v1.15.bin                         | OP-TEE, implemented a Trusted Execution Environment (TEE)          |
|                                               | Path and filename must be passed using the TEE variable.           |
|                                               | If you want to rename this file, the ending MUST stay ".bin".      |
+-----------------------------------------------+--------------------------------------------------------------------+

Layout of the sectors at the start of the eMMC
----------------------------------------------

(Each sector is 512 bytes)

+--------------+-------------+---------------+-----------------------------------------------------------------------+
| first sector | last sector | length        | meaning                                                               |
+--------------+-------------+---------------+-----------------------------------------------------------------------+
| 0x0          | 0x0         | 512 bytes     | protective master boot record (MBR)                                   |
+--------------+-------------+---------------+-----------------------------------------------------------------------+
| 0x1          | 0x39        | 28 KiB        | GUID Partition Table (GPT)                                            |
+--------------+-------------+---------------+-----------------------------------------------------------------------+
| 0x40         | 0x3fff      | 8160 KiB,     | U-Boot SPL                                                            |
|              |             | almost 8 MiB  |                                                                       |
+--------------+-------------+---------------+-----------------------------------------------------------------------+
| 0x4000       | 0x7fff      | 8 MiB         | raw U-Boot Proper, no filesystem                                      |
+--------------+-------------+---------------+-----------------------------------------------------------------------+
| ...                                                                                                                |
+--------------+-------------+---------------+-----------------------------------------------------------------------+

After sector 0x7fff, the eMMC can be partitioned in any way you like. However, two partitions are highly recommended to
have:
* a Linux /boot partition with at least 1.5 GiB of space to keep two installed Linux kernels (one for booting,
  one for backup) including modules, initrds (initial ram disks) and DTBs (device tree blobs). The recommended
  filesystem is ext4 (FAT variants and btrfs are theoretically supported, but such a setup is not tested)
  This partition *MUST* have the following properties:
  - Partition GUID code: 0FC63DAF-8483-4772-8E79-3D69D8477DE4 (in parted and gdisk, you can select
    this using the type code 8300 for "Linux filesystem")
  This partition *SHOULD* (recommendation) have the following properties:
  - Partition label: boot
* an UEFI system partition with at least 200 MiB of space.
  This partition *MUST* have the following properties:
  - File system: FAT32 (can be created with mkfs.vfat -F32 /dev/mmcblk0p<number of the partition>)
  - Partition GUID code: C12A7328-F81F-11D2-BA4B-00A0C93EC93B (EFI system partition) (in parted and gdisk, you can
    select this using the type code ef00)
  - Attribute flags: 0000000000000001 (=attribute "system partition")
  This partition *SHOULD* (recommendation) have the following properties:
  - Partition label: EFI

BL33 ("Non-trusted Firmware") = U-Boot

Build artefacts and their contents
----------------------------------

Inhalt von spl_loader auspacken: mkdir -p tmp && ./tools/boot_merger  unpack -i rk3588_spl_loader_v1.13.112.bin  -o tmp

rk3588_spl_loader_v1.13.112.bin =
    extracting entry=471,file=UsbHead.bin
    extracting entry=471,file=rk3588_ddr_lp4_2112.bin = rk3588_ddr_lp4_2112MHz_lp5_2736MHz_v1.13.bin
    extracting entry=472,file=rk3588_usbplug_v1.bin   = rk3588_usbplug_v1.10.bin
    extracting entry=loader,file=FlashHead.bin
    extracting entry=loader,file=FlashData.bin        = rk3588_spl_v1.12.bin
    extracting entry=loader,file=FlashBoot.bin

Kuriosum:
"Files u-boot.img and u-boot-dtb.img are identical"

"arch/arm/mach-rockchip/make_fit_atf.sh -t 0x08400000" erzeugt u-boot.its
u-boot.its + "./tools/mkimage -f u-boot.its -E u-boot.itb" erzeugt u-boot.itb

~/firefly-work/rkbin$ ./tools/trust_merger RKTRUST/RK3588TRUST.ini erzeugt trust.img
    trust.img = rk3588_bl31_v1.42.elf + rk3588_bl32_v1.14.bin wegen folgender Konfiguration in RK3588TRUST.ini:
    [BL31_OPTION]
    SEC=1
    PATH=bin/rk35/rk3588_bl31_v1.42.elf
    ADDR=0x00040000
    [BL32_OPTION]
    SEC=1
    PATH=bin/rk35/rk3588_bl32_v1.14.bin
    ADDR=0x08400000

Disk /dev/mmcblk0: 488570880 sectors, 233.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 73987B6B-4974-4C94-A3E8-58AB2EB7A946
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 488570846
Partitions will be aligned on 2048-sector boundaries
Total free space is 16350 sectors (8.0 MiB)

Beginn der MMC:
1 Sektor = 512 Bytes


Codegröße in 2017:
 u-boot-spl-dtb.bin                             │ 229242
*u-boot-spl-nodtb.bin                           │ 224312

Codegröße in 2024:
 u-boot-spl-dtb.bin                             │ 177712
*u-boot-spl-nodtb.bin                           │ 172512

Aber: CONFIG_SPL_MAX_SIZE legt die Obergrenze auf 0x40000 (=262.144 Bytes, 256 kB) fest! o.o
SPL startet bei 0x8000. Wird ggf. ausgefüllt bis SPL_PAD_TO [=0x7f8000]. 0x8000 + 0x7f8000 = 0x800000.

--> SPL lebt zu 100% am Beginn der EMMC im Bereich [0x8000 - 0x7fffff] !

SPL_PAYLOAD [=u-boot.bin]
 Payload for SPL boot. For backward compatibility, default to
 u-boot.bin, i.e. RAW image without any header. In case of
 TPL, tpl/u-boot-with-tpl.bin. For new boards, suggest
 use u-boot.img.

SYS_SPI_U_BOOT_OFFS [=0x60000]
  <-- 0x60000 ist merkwürdig. Ist der SPL zweigeteilt?
  0x60000 auf dem eMMC enthält nichts besonders, es scheint mittem im Binärcode zu liegen

--> idbloader.img passt vom Aufbau ("RKNS...") und Länge theoretisch ideal!
    Byte-identisch zu idbloader-spi.img

Was ist wo drin?
================

Suchstrings:
- BL31 / ARM-Trusted-Firmware (ATF):
  Initializing BL32
- DRAM / rk3588_ddr_lp4_2112MHz_lp5_2736MHz_v1.13.bin
  DRAM type unsupported
- OP-TEE:
  TEE load address
- SPL:
  U-Boot SPL
- U-Boot main
  Executing script
- Device Tree Blob (complete)
  hdmi0-sound
- Device Tree Blob (SPL) - only if it does NOT contain "es8388-sound"
  rockchip,rk3588-firefly-itx-3588j

build process in a nutshell:
- Compile C-Code
  --> u-boot
      ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
      eu-readelf u-boot --all shows:
      ELF entry point: 0xa00000
      Program Headers:
      Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
      LOAD           0x010000 0x0000000000a00000 0x0000000000a00000 0x0ebcc0 0x0ebcc0 RWE 0x10000
      GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10

- objcopy of the core ELF sections of "u-boot" into a new binary, then apply ELF relocations statically
  --> u-boot-nodtb.bin
- Print the dynamic symbol table entries of "u-boot" using objdump
  --> u-boot.sym (symbols for debugging)
- Compile DTS (data tree sources and includes)
  --> *.dtb (device tree blobs), especially:
  --> dts/dt.dtb (device tree for the "U-Boot proper", the main program)
  --> u-boot.dtb (copy with a new name)
- cat u-boot-nodtb.bin dts/dt.dtb
  --> u-boot-dtb.bin (statically relocated "u-boot" + full device tree)
- ./tools/mkimage ... -b <board name>.dtb -d u-boot-nodtb.bin u-boot.img
  --> u-boot.img (FIT image containing u-boot and the full DTB)
- cp <board name>.dtb to...
- ./tools/mkimage ... -b <board name>.dtb -d u-boot-nodtb.bin u-boot.img
  --> u-boot.img
- (same mkimage, different output file name, same result):
  --> u-boot-dtb.img
  --> u-boot.bin (copy with a new name)
- fdtgrep only the parts relevant for SPL from dts/dt.dtb
  --> spl/dts/dt-spl.dtb
  --> spl/u-boot-spl.dtb (copy with a new name)
- Compile C-Code for the SPL
  --> spl/u-boot-spl (ELF binary)
      ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
      eu-readelf spl/u-boot-spl --all shows:
      ELF entry point: 0x0
      no program headers
- Print the symbol table (NOT dynamic this time) entries of "spl/u-boot-spl" using objdump
  --> spl/u-boot-spl.sym (symbols for debugging)
- objcopy of the core ELF sections of "spl/u-boot-spl" into a new binary, then apply ELF relocations statically
  --> spl/u-boot-spl-nodtb.bin
- cat spl/u-boot-spl-nodtb.bin spl/u-boot-spl.dtb
  --> spl/u-boot-spl-dtb.bin
  --> spl/u-boot-spl.bin (copy with a new nmae)
-  ./tools/binman/binman ... -d u-boot.dtb -a atf-bl31-path=b/rk3588/atf-bl31 -a tee-os-path=b/rk3588/rk3588_bl32_v1.15.bin
                             -a rockchip-tpl-path=b/rk3588/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.16.bin
  --> idbloader-spi.img
  --> idbloader.img (copy with a new name)
  --> u-boot.itb
  --> u-boot-rockchip.bin
  --> u-boot-rockchip-spi.bin

Binary                   | DRAM |BL31 | OP-TEE | SPL | DTB (SPL) | U-Boot main | DTB-full |  Size              | BLOB type
=========================+======+=====+========+=====+===========+=============+===============================+===========================================================================================================
u-boot.dtb               |      |     |        |     |           |             | yes      |  ~ 175KiB          | device tree blob (DTB)
u-boot                   |      |     |        |     |           | yes         |          |  ~ 8-9 MiB         | ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
u-boot-nodtb.bin         |      |     |        |     |           | yes         |          |  ~ 900 KiB         | custom binary format of "u-boot"
u-boot.img           [1] |      |     |        |     |           | yes         | yes      |  ~ 1 MiB           | FIT-Image with   u-boot-nodtb.bin + u-boot.dtb
u-boot-dtb.img       [1] |      |     |        |     |           | yes         | yes      |  ~ 1 MiB           | copy of u-boot.img
u-boot-dtb.bin       [1] |      |     |        |     |           | yes         | yes      |  ~ 1 MiB           | Concatenation of u-boot-nodtb.bin + u-boot.dtb
u-boot.bin           [1] |      |     |        |     |           | yes         | yes      |  ~ 1 MiB           | copy of u-boot-dtb.bin
-------------------------+------+-----+--------+-----+-----------+-------------+----------+--------------------+-------------------------------------------------------
spl/u-boot-spl.dtb       |      |     |        |     | yes       |             |          |  ~ 6-10 KiB        | device tree blob (DTB), severely cut down for SPL
spl/u-boot-spl           |      |     |        | yes |           |             |          |  ~ 3 MiB           | ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
spl/u-boot-spl-nodtb.bin |      |     |        | yes |           |             |          |  ~ 180 KiB         | custom binary format of "spl/u-boot-spl"
spl/u-boot-sql-dtb.bin   |      |     |        | yes | yes       |             |          |  ~ 190 KiB         | Concatenation of spl/u-boot-spl-nodtb.bin + u-boot-spl.dtb
-------------------------+------+-----+--------+-----+-----------+-------------+----------+--------------------+-------------------------------------------------------
idbloader(-spi).img [1]  | yes  |     |        | yes |           |             |          |  266240 (0x41000)  | RKNS/rksd, contains ROCKCHIP_TPL + spl/u-boot-spl-nodtb.bin
-------------------------+------+-----+--------+-----+-----------+-------------+----------+--------------------+-------------------------------------------------------
u-boot.itb          [1]  |      | yes | yes    |     |           | yes         | yes      | ~ 1.9 MiB          | FIT image with BL31 (atf-1, -2 and -3), OP-TEE, U-Boot main and full DTB
-------------------------+------+-----+--------+-----+-----------+-------------+----------+--------------------+-------------------------------------------------------
u-boot-rockchip.bin      | yes  | yes | yes    | yes | yes       | yes         | yes      |
u-boot-rockchip-spi.bin  | yes  | yes | yes    | yes | yes       | yes         | yes      |
-------------------------+------+-----+--------+-----+-----------+-------------+----------+--------------------+-------------------------------------------------------
uboot.img                |      | yes | yes    |     |  yes         | yes      |
(the following binaries are created by the Rockchip build pipeline code):

[1] The following files are identical, in both length and sha256sum after build:
    - idbloader.img
      idbloader-spi.img
    - u-boot.itb
      simple-bin.fit.fit
    - mkimage-in-simple-bin.mkimage-u-boot-spl
      mkimage-in-simple-bin-spi.mkimage-u-boot-spl
    - mkimage-in-simple-bin.mkimage-rockchip-tpl
      mkimage-in-simple-bin-spi.mkimage-rockchip-tpl
    - u-boot.bin
      u-boot-dtb.bin
    - u-boot.img
      u-boot-dtb.img

Contents of final FIT image (u-boot.itb):
    FIT description: FIT image for U-Boot with bl31 (TF-A)
    Created:         Wed Mar 27 00:22:40 2024
     Image 0 (u-boot)
      Description:  U-Boot
      Created:      Wed Mar 27 00:22:40 2024
      Type:         Standalone Program
      Compression:  gzip compressed
      Data Size:    496940 Bytes = 485.29 KiB = 0.47 MiB
      Architecture: AArch64
      Load Address: 0x00a00000
      Entry Point:  0x00a00000
      Hash algo:    sha256
      Hash value:   c021b36859464b4212e780946864f32a7a1f7a25c8b4af1e16b1815e91400650
     Image 1 (atf-1)
      Description:  ARM Trusted Firmware
      Created:      Wed Mar 27 00:22:40 2024
      Type:         Firmware
      Compression:  uncompressed
      Data Size:    200008 Bytes = 195.32 KiB = 0.19 MiB
      Architecture: AArch64
      OS:           ARM Trusted Firmware
      Load Address: 0x00040000
      Hash algo:    sha256
      Hash value:   c902200be1343fe569e54778c286005b1c6163606664c463a24d787be4376966
     Image 2 (atf-2)
      Description:  ARM Trusted Firmware
      Created:      Wed Mar 27 00:22:40 2024
      Type:         Firmware
      Compression:  uncompressed
      Data Size:    24576 Bytes = 24.00 KiB = 0.02 MiB
      Architecture: AArch64
      OS:           ARM Trusted Firmware
      Load Address: 0x000f0000
      Hash algo:    sha256
      Hash value:   aa71013e72d7ab4be264c1093b155ef06e65d0a263d552be25b13c8ddf285586
     Image 3 (atf-3)
      Description:  ARM Trusted Firmware
      Created:      Wed Mar 27 00:22:40 2024
      Type:         Firmware
      Compression:  uncompressed
      Data Size:    24576 Bytes = 24.00 KiB = 0.02 MiB
      Architecture: AArch64
      OS:           ARM Trusted Firmware
      Load Address: 0xff100000
      Hash algo:    sha256
      Hash value:   225d6bf0712f850648223365ba06a73ba5f6315fb8a9580f23ab48ece795f91e
     Image 4 (optee)
      Description:  OP-TEE
      Created:      Wed Mar 27 00:22:40 2024
      Type:         Firmware
      Compression:  uncompressed
      Data Size:    465312 Bytes = 454.41 KiB = 0.44 MiB
      Architecture: AArch64
      OS:           Trusted Execution Environment
      Load Address: 0x08400000
      Hash algo:    sha256
      Hash value:   4b2d406bfcf3aaed41877fd2356ac694a808f6788e9ec2545742fd2911ef5d5d
     Image 5 (fdt-1)
      Description:  fdt-rk3588-firefly-itx-3588j
      Created:      Wed Mar 27 00:22:40 2024
      Type:         Flat Device Tree
      Compression:  uncompressed
      Data Size:    573160 Bytes = 559.73 KiB = 0.55 MiB
      Architecture: AArch64
      Hash algo:    sha256
      Hash value:   e4580482f92b7b2def313172d9d49b09d1fd10de297c1a86ef1dd3cf9e7a7dba
     Default Configuration: 'config-1'
     Configuration 0 (config-1)
      Description:  rk3588-firefly-itx-3588j.dtb
      Kernel:       unavailable
      Firmware:     atf-1
      FDT:          fdt-1
      Loadables:    u-boot
                    atf-2
                    atf-3
                    optee
      Sign algo:    sha256,rsa2048:dev
      Sign value:   unavailable
      Timestamp:    unavailable


--> u-boot-rockchip.bin enthält sowohl SPL als auch U-Boot Main.
    Wird es an die vorgesehene Position (0x8000 absolut) geflashed,
    überdeckt es auch Partition #1, so dass U-Boot (Main) automatisch mit geflashed wird.
    In DD: seek=64 -> Überspringt 64 Sektoren -> Position 0x8000
    Ergo: dd if=u-boot-rockchip.bin of=/dev/mmcblk0 seek=64


Number  Start (sector)    End (sector)  Size       Code  Name
   1           16384           32767   8.0 MiB     FFFF  uboot    <--- uboot.img raw, no file system
   2           32768           40959   4.0 MiB     FFFF  misc     <--- Only relevant for Android
   3           40960          565247   256.0 MiB   FFFF  boot     <--- Normal Linux /boot partition with kernel, initrd, rk-kernel.dtb and extlinux/extlinux.conf
   4          565248          827391   128.0 MiB   FFFF  recovery <--- Only relevant for Android
   5          827392          892927   32.0 MiB    FFFF  backup   <--- No idea, but I never needed it
   6          892928        13475839   6.0 GiB     FFFF  rootfs   <--- rootfs.img
   7        13475840       488570846   226.5 GiB   FFFF  userdata

------ PACKAGE ------
Add file: ./package-file
package-file,Add file: ./package-file done,offset=0x800,size=0xf4,userspace=0x1
Add file: ./Image/MiniLoaderAll.bin
bootloader,Add file: ./Image/MiniLoaderAll.bin done,offset=0x1000,size=0x731c0,userspace=0xe7
Add file: ./Image/parameter.txt
parameter,Add file: ./Image/parameter.txt done,offset=0x74800,size=0x1e2,userspace=0x1,flash_address=0x00000000
Add file: ./Image/uboot.img
uboot,Add file: ./Image/uboot.img done,offset=0x75000,size=0x800000,userspace=0x1000,flash_address=0x00004000
Add file: ./Image/misc.img
misc,Add file: ./Image/misc.img done,offset=0x875000,size=0xc000,userspace=0x18,flash_address=0x00008000
Add file: ./Image/boot.img
boot,Add file: ./Image/boot.img done,offset=0x881000,size=0x10000000,userspace=0x20000,flash_address=0x0000a000
Add file: ./Image/recovery.img
recovery,Add file: ./Image/recovery.img done,offset=0x10881000,size=0x4bee800,userspace=0x97dd,flash_address=0x0008a000
Add file: ./Image/rootfs.img
rootfs,Add file: ./Image/rootfs.img done,offset=0x1546f800,size=0x2df00000,userspace=0x5be00,flash_address=0x000da000
Add CRC...
Make firmware OK!
------ OK ------

Boot output (old U-Boot, old Kernel)
------------------------------------

(from boot0 / ROM):
    DDR V1.13 25cee80c4f cym 23/08/11-09:31:58
    LPDDR4, 2112MHz
    channel[0] BW=16 Col=10 Bk=8 CS0 Row=18 CS1 Row=18 CS=2 Die BW=8 Size=8192MB
    channel[1] BW=16 Col=10 Bk=8 CS0 Row=18 CS1 Row=18 CS=2 Die BW=8 Size=8192MB
    channel[2] BW=16 Col=10 Bk=8 CS0 Row=18 CS1 Row=18 CS=2 Die BW=8 Size=8192MB
    channel[3] BW=16 Col=10 Bk=8 CS0 Row=18 CS1 Row=18 CS=2 Die BW=8 Size=8192MB
    Manufacturer ID:0xff
    CH0 RX Vref:36.1%, TX Vref:13.2%,13.2%
    CH1 RX Vref:35.2%, TX Vref:13.2%,13.2%
    CH2 RX Vref:38.1%, TX Vref:13.2%,13.2%
    CH3 RX Vref:39.0%, TX Vref:13.2%,13.2%
    change to F1: 528MHz
    change to F2: 1068MHz
    change to F3: 1560MHz
    change to F0: 2112MHz
    out

(from U-Boot SPL):
    U-Boot SPL board init
    U-Boot SPL 2017.09-g5f53abfa1e-221223 #zzz (Dec 26 2022 - 09:10:09)
    unknown raw ID 0 0 0
    unrecognized JEDEC id bytes: 00, 00, 00
    Trying to boot from MMC2
    MMC: no card present
    mmc_init: -123, time 1
    spl: mmc init failed with error: -123
    Trying to boot from MMC1
    SPL: A/B-slot: _a, successful: 0, tries-remain: 7
    Trying fit image at 0x4000 sector
    ## Verified-boot: 0
    ## Checking atf-1 0x00040000 ... sha256(c902200be1...) + OK
    ## Checking uboot 0x00200000 ... sha256(38f5a5cdb7...) + OK
    ## Checking fdt 0x00365900 ... sha256(8ecccd16d2...) + OK
    ## Checking atf-2 0x000f0000 ... sha256(aa71013e72...) + OK
    ## Checking atf-3 0xff100000 ... sha256(225d6bf071...) + OK
    ## Checking optee 0x08400000 ... sha256(66e30bf9e8...) + OK
    Jumping to U-Boot(0x00200000) via ARM Trusted Firmware(0x00040000)
    Total: 120.340 ms

(from ARM Trusted Firmware and OP-TEE):
    INFO:    Preloader serial: 2
    NOTICE:  BL31: v2.3():v2.3-639-g87bcc5dfe:derrick.huang
    NOTICE:  BL31: Built : 18:06:16, Sep  9 2023
    INFO:    spec: 0x1
    INFO:    ext 32k is not valid
    INFO:    ddr: stride-en 4CH
    INFO:    GICv3 without legacy support detected.
    INFO:    ARM GICv3 driver initialized in EL3
    INFO:    valid_cpu_msk=0xff bcore0_rst = 0x0, bcore1_rst = 0x0
    INFO:    l3 cache partition cfg-0
    INFO:    system boots from cpu-hwid-0
    INFO:    idle_st=0x21fff, pd_st=0x11fff9, repair_st=0xfff70001
    INFO:    dfs DDR fsp_params[0].freq_mhz= 2112MHz
    INFO:    dfs DDR fsp_params[1].freq_mhz= 528MHz
    INFO:    dfs DDR fsp_params[2].freq_mhz= 1068MHz
    INFO:    dfs DDR fsp_params[3].freq_mhz= 1560MHz
    INFO:    BL31: Initialising Exception Handling Framework
    INFO:    BL31: Initializing runtime services
    INFO:    BL31: Initializing BL32
    I/TC:
    I/TC: OP-TEE version: 3.13.0-743-gb5340fd65 #hisping.lin (gcc version 10.2.1 20201103 (GNU Toolchain for the A-profile Architecture 10.2-2020.11 (arm-10.16))) #6 Mon Aug 28 18:01:38 CST 2023 aarch64
    I/TC: Primary CPU initializing
    I/TC: Primary CPU switching to normal world boot
    INFO:    BL31: Preparing for EL3 exit to normal world
    INFO:    Entry point address = 0x200000
    INFO:    SPSR = 0x3c9

(from U-Boot Proper):
    U-Boot 2017.09(u-boot commit id: 16305c9502)(sdk version: rk3588_linux_release_20240105_v1.3.0b)-g16305c9502-230909-dirty #ab (Mar 15 2024 - 00:14:07 +0100)

    Model: Rockchip RK3588 Evaluation Board
    MPIDR: 0x81000000
    PreSerial: 2, raw, 0xfeb50000
    DRAM:  31.7 GiB
    Sysmem: init
    Relocation Offset: ed9f8000
    Relocation fdt: eb7fa5e0 - eb7fece0
    CR: M/C/I
    Using default environment

    optee api revision: 2.0
    mmc@fe2c0000: 1, mmc@fe2e0000: 0
    Bootdev(atags): mmc 0
    MMC0: HS400 Enhanced Strobe, 200Mhz
    PartType: EFI
    TEEC: Waring: Could not find security partition
    DM: v2
    extboot part -> 3
    =================begin===================
    282968 bytes read in 7 ms (38.6 MiB/s)
    DTB(Distro): rk-kernel.dtb
    boot mode: normal
    I2c0 speed: 100000Hz
    vsel-gpios- not found!
    en-gpios- not found!
    vdd_cpu_big0_s0 800000 uV
    vsel-gpios- not found!
    en-gpios- not found!
    vdd_cpu_big1_s0 800000 uV
    I2c1 speed: 100000Hz
    vsel-gpios- not found!
    en-gpios- not found!
    vdd_npu_s0 800000 uV
    spi2: RK806: 2
    ON=0x80, OFF=0x80
    vdd_gpu_s0 750000 uV
    vdd_cpu_lit_s0 750000 uV
    vdd_log_s0 750000 uV
    vdd_vdenc_s0 init 750000 uV
    vdd_ddr_s0 850000 uV
    I2c6 speed: 400000Hz
    Firefly fixed_regulator_set_enable: dev='vcc-hub-reset-regulator', enable=1, delay=0, has_gpio=1
    Firefly fixed_regulator_set_enable: dev='vcc-hub3-reset-regulator', enable=1, delay=0, has_gpio=1
    Firefly fixed_regulator_set_enable: dev='vcc-sata-pwr-en-regulator', enable=1, delay=0, has_gpio=1
    Firefly fixed_regulator_set_enable: dev='vcc-fan-pwr-en-regulator', enable=1, delay=0, has_gpio=1
    Firefly fixed_regulator_set_enable: dev='vcc-sdcard-pwr-en-regulator', enable=1, delay=0, has_gpio=1
    get vp0 plane mask:0x5, primary id:2, cursor_plane:-1, from dts
    get vp1 plane mask:0xa, primary id:3, cursor_plane:-1, from dts
    get vp2 plane mask:0x140, primary id:8, cursor_plane:-1, from dts
    get vp3 plane mask:0x280, primary id:9, cursor_plane:-1, from dts
    Could not find baseparameter partition
    In:    serial
    Out:   serial
    Err:   serial
    Model: Firefly ITX-3588J MIPI(Linux)
    MPIDR: 0x81000000
    No resource file: logo.bmp
    ** Bad partition specification mmc 0:3 4 **
    Rockchip UBOOT DRM driver version: v1.0.1
    vp0 have layer nr:2[0 2 ], primary plane: 2
    vp1 have layer nr:2[1 3 ], primary plane: 3
    vp2 have layer nr:2[6 8 ], primary plane: 8
    vp3 have layer nr:2[7 9 ], primary plane: 9
    Using display timing dts
    dsi@fde20000:  detailed mode clock 72600 kHz, flags[a]
        H: 0800 0832 0846 0872
        V: 1280 1360 1368 1388
    bus_format: 100e
    VOP update mode to: 800x1280p60, type: MIPI0 for VP3
    [list]p_rate=1188000000, best_rate=69882352, div=17, sel=0
    [list]p_rate=1500000000, best_rate=71428571, div=21, sel=1
    [list]p_rate=0, best_rate=71428571, div=21, sel=1
    [list]p_rate=786431991, best_rate=71493817, div=11, sel=3
    [result]p_rate=786431991, best_rate=71493817, div=11, sel=3
    VP3 set crtc_clock to 71494KHz
    VOP VP3 enable Esmart3[500x501->500x501@150x389] fmt[2] addr[0xedf20000]
    final DSI-Link bandwidth: 476626 Kbps x 4
    command interface is busy: 0x10001
    [Firefly]-[rockchip_panel_send_dsi_cmds]-[291]:  read 4 = 0
    [Firefly]-[rockchip_panel_send_dsi_cmds]-[296]: Not Found ID = 83 MIPI!
    failed to write/read cmd0: -999
    failed to send on cmds: -999
    hdmi@fde80000 disconnected
    Monitor has basic audio support
    Could not find baseparameter partition
    color_format:1
    hdmi_select_link_config use tmds mode
    mode:3840x2160 bus_format:0x2025
    hdmi@fdea0000:  detailed mode clock 594000 kHz, flags[5]
        H: 3840 4016 4104 4400
        V: 2160 2168 2178 2250
    bus_format: 2025
    VOP update mode to: 3840x2160p60, type: HDMI1 for VP1
    dclk:594000,if_pixclk_div;2,if_dclk_div:4
    hdptx_ropll_cmn_config bus_width:5aa320 rate:5940000
    hdptx phy pll locked!
    VP1 set crtc_clock to 5940KHz
    VOP VP1 enable Esmart1[500x501->500x501@1670x829] fmt[2] addr[0xedf20000]
    CEA mode used vic=97
    mtmdsclock:594000000
    hdptx_ropll_cmn_config bus_width:5aa320 rate:5940000
    hdptx phy pll locked!
    dw_hdmi_setup HDMI mode
    don't use dsc mode
    dw hdmi qp use tmds mode
    bus_width:0x5aa320,bit_rate:5940000
    hdptx phy lane locked!
    CLK: (sync kernel. arm: enter 1008000 KHz, init 1008000 KHz, kernel 0N/A)
      b0pll 24000 KHz
      b1pll 24000 KHz
      lpll 24000 KHz
      v0pll 24000 KHz
      aupll 786431 KHz
      cpll 1500000 KHz
      gpll 1188000 KHz
      npll 850000 KHz
      ppll 1100000 KHz
      aclk_center_root 702000 KHz
      pclk_center_root 100000 KHz
      hclk_center_root 396000 KHz
      aclk_center_low_root 500000 KHz
      aclk_top_root 750000 KHz
      pclk_top_root 100000 KHz
      aclk_low_top_root 396000 KHz
    Net:   eth1: ethernet@fe1c0000, eth0: ethernet@fe1b0000
    Autoboot in 5 seconds
    ANDROID: reboot reason: "(none)"
    Not AVB images, AVB skip
    No valid android hdr
    Android image load failed
    Android boot failed, error -1.
    ## Booting FIT Image FIT: No fit blob
    FIT: No FIT image
    Unknown command 'bootrkp' - try 'help'
    MMC: no card present
    mmc_init: -123, time 0
    switch to partitions #0, OK
    mmc0(part 0) is current device
    Scanning mmc 0:3...
    Found /extlinux/extlinux.conf
    Retrieving file: /extlinux/extlinux.conf
    =================begin===================
    191 bytes read in 2 ms (92.8 KiB/s)
    FIREFLY: use /rk-kernel.dtb
    FIREFLY: use /rk-kernel.dtb
    1:      rk-kernel.dtb linux-5.10.160-opensuse
    Retrieving file: /initrd
    =================begin===================
    18832171 bytes read in 75 ms (239.5 MiB/s)
    Retrieving file: /Image
    =================begin===================
    39848824 bytes read in 141 ms (269.5 MiB/s)
    Retrieving file: /rk-kernel.dtb
    =================begin===================
    282968 bytes read in 7 ms (38.6 MiB/s)
    Fdt Ramdisk skip relocation
    ## Flattened Device Tree blob at 0x08300000
       Booting using the fdt blob at 0x08300000
       Using Device Tree in place at 0000000008300000, end 0000000008348157
    No resource file: logo_kernel.bmp
    ** Bad partition specification mmc 0:3 4 **
    WARNING: could not set reg FDT_ERR_BADOFFSET.
    ## reserved-memory:
      cma: addr=10000000 size=10000000
      drm-logo@00000000: addr=edf00000 size=136000
      ramoops@110000: addr=110000 size=f0000
    Adding bank: 0x00200000 - 0x08400000 (size: 0x08200000)
    Adding bank: 0x09400000 - 0xf0000000 (size: 0xe6c00000)
    Adding bank: 0x100000000 - 0x3fc000000 (size: 0x2fc000000)
    Adding bank: 0x3fc500000 - 0x3fff00000 (size: 0x03a00000)
    Adding bank: 0x400000000 - 0x800000000 (size: 0x400000000)
    Total: 6414.581/6838.743 ms

    Starting kernel ...

(from the Linux kernel):
    [    6.843468] Booting Linux on physical CPU 0x0000000000 [0x412fd050]
    [    6.843488] Linux version 5.10.160-150606-fireflyitx3588j (geeko@buildhost) (kernel commit id: ) (sdk version: .xml) (gcc-10 (SUSE Linux) 10.5.0, GNU ld (GNU Binutils; openSUSE Tumbleweed) 2.42.0.20240130-2) #1 SMP Fri Jan 5 13:08:45 UTC 2024 (97d7414)
    [    6.853868] Machine model: Firefly ITX-3588J MIPI(Linux)
    [    6.853941] printk: debug: ignoring loglevel setting.
    [    6.853966] earlycon: uart8250 at MMIO32 0x00000000feb50000 (options '')
    [    6.858142] printk: bootconsole [uart8250] enabled
    [    6.860810] efi: UEFI not found.
    [    6.866485] OF: fdt: Reserved memory: failed to reserve memory for node 'drm-cubic-lut@00000000': base 0x0000000000000000, size 0 MiB
    [    6.867631] Reserved memory: created CMA memory pool at 0x0000000010000000, size 256 MiB
    [    6.868373] OF: reserved mem: initialized node cma, compatible id shared-dma-pool
    [    7.270861] Zone ranges:
    [    7.271104]   DMA      [mem 0x0000000000200000-0x00000000ffffffff]
    [    7.271676]   DMA32    empty
    [    7.271944]   Normal   [mem 0x0000000100000000-0x00000007ffffffff]
    [    7.272513] Movable zone start for each node
    [    7.272906] Early memory node ranges
    [    7.273236]   node   0: [mem 0x0000000000200000-0x00000000083fffff]
    [    7.273811]   node   0: [mem 0x0000000009400000-0x00000000efffffff]
    [    7.274389]   node   0: [mem 0x0000000100000000-0x00000003fbffffff]
    [    7.274968]   node   0: [mem 0x00000003fc500000-0x00000003ffefffff]
    [    7.275544]   node   0: [mem 0x0000000400000000-0x00000007ffffffff]
    [    7.276124] Initmem setup node 0 [mem 0x0000000000200000-0x00000007ffffffff]
    [    7.276771] On node 0 totalpages: 8316928
    [    7.277142]   DMA zone: 15288 pages used for memmap
    [    7.277590]   DMA zone: 0 pages reserved
    [    7.277953]   DMA zone: 978432 pages, LIFO batch:63
    [    7.278402]   Normal zone: 114688 pages used for memmap
    [    7.278882]   Normal zone: 7338496 pages, LIFO batch:63
    [    7.453941] On node 0, zone Normal: 256 pages in unavailable ranges
