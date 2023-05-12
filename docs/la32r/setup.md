# La32r Setup

## Toolchains

We recommend to use [chenyy/la32r-env](https://hub.docker.com/r/chenyy/la32r-env) for uCore or Linux build.

```console
$ docker pull chenyy/la32r-env:latest
```

Then we can launch the container based on the image.

```console
$ docker run -dit \
    --name la32r-docker \
    --net=host \
    -e LANG=en_US.UTF-8 \
    -e LANGUAGE=en_US.UTF-8 \
    -e LC_ALL=en_US.UTF-8 \
    chenyy/la32r-env:latest
```

Now you can enter the container and use la32r toolchains.

```console
$ docker exec -it la32r-docker /bin/zsh
```

## uCore boot

First, we need to build uCore & CEMU.

```console
$ git clone https://github.com/cyyself/ucore-loongarch32.git
$ pushd ucore-loongarch32
$ make
$ loongarch32r-linux-gnusf-objcopy -O binary obj/ucore-kernel-initrd obj/ucore-kernel-initrd.bin
$ popd
$ git clone https://github.com/cyyself/cemu.git
$ cd cemu
```

Save the following content to `src/main.cpp`

```cpp title="src/main.cpp"
#include <iostream>
#include <bitset>
#include <termios.h>
#include <unistd.h>
#include <thread>
#include <signal.h>

#include "device/nscscc_confreg.hpp"
#include "device/uart8250.hpp"
#include "memory/memory_bus.hpp"
#include "memory/ram.hpp"
#include "core/la32r/la32r_core.hpp"

void uart_input(uart8250 &uart) {
    termios tmp;
    tcgetattr(STDIN_FILENO, &tmp);
    tmp.c_lflag &= (~ICANON & ~ECHO);
    tcsetattr(STDIN_FILENO, TCSANOW, &tmp);
    while (true) {
        char c = getchar();
        if (c == 10) c = 13; // convert lf to cr
        uart.putc(c);
    }
}

bool send_ctrl_c;

void sigint_handler(int x) {
    static time_t last_time;
    if (time(NULL) - last_time < 1) exit(0);
    last_time = time(NULL);
    send_ctrl_c = true;
}

int main(int argc, char **argv) {
    signal(SIGINT, sigint_handler);

    memory_bus cemu_mmio;

    ram cemu_memory(128 * 1024 * 1024);
    cemu_memory.load_binary(0x000000, "../ucore-loongarch32/obj/ucore-kernel-initrd.bin");
    assert(cemu_mmio.add_dev(0, 128 * 1024 * 1024, &cemu_memory));

    uart8250 uart;
    std::thread *uart_input_thread = new std::thread(uart_input, std::ref(uart));
    assert(cemu_mmio.add_dev(0x1fe001e0, 0x10, &uart));

    la32r_core<32> core(0, cemu_mmio, false);
    core.csr_cfg(0x180, 0xa0000011u);
    core.csr_cfg(0x181, 0x80000001u);
    core.csr_cfg(0x0, 0xb0);
    core.jump(0xa0000000u);
    while (true) {
        core.step(uart.irq() << 1);
        while (uart.exist_tx()) {
            char c = uart.getc();
            if (c != '\r') {
                putchar(c);
                fflush(stdout);
            }
        }
        if (send_ctrl_c) {
            uart.putc(3);
            send_ctrl_c = false;
        }
    }
    return 0;
}
```

Then build and run cemu.

```console
$ make  # in cemu project folder
$ ./cemu
++setup timer interrupts
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0xA0000120 (phys)
  etext 0xA0022000 (phys)
  edata 0xA025CEA0 (phys)
  end   0xA0260180 (phys)
Kernel executable memory footprint: 2297KB
memory management: default_pmm_manager
memory map:
    [A0000000, A2000000]

freemem start at: A02A1000
free pages: 00001D5F
## 00000020
check_alloc_page() succeeded!
check_pgdir() succeeded!
check_boot_pgdir() succeeded!
check_slab() succeeded!
kmalloc_init() succeeded!
check_vma_struct() succeeded!
check_pgfault() succeeded!
check_vmm() succeeded.
sched class: stride_scheduler
proc_init succeeded
Initrd: 0xa00601c0 - 0xa02541bf, size: 0x001f4000, magic: 0x2f8dbe2a
ramdisk_init(): initrd found, magic: 0x2f8dbe2a, 0x00000fa0 secs
sfs: mount: 'simple file system' (352/148/500)
vfs: mount disk0.
kernel_execve: pid = 2, name = "sh".
user sh is running!!!
$
```

## Linux boot

### Download Prebuilt Busybox Ramdisk

Use the following command to download prebuilt busybox ramdisk.

```console
$ mkdir -p initrd_d
$ wget -qO- https://gitee.com/loongson-edu/la32r-Linux/releases/download/v0.2/initrd_d.tar.gz | tar -xv -C initrd_d --strip-components 1
```

### Build Linux Kernel

First, we need to clone la32r-Linux repository.

```console
$ git clone --depth 1 https://gitee.com/loongson-edu/la32r-Linux.git
$ pushd la32r-Linux
```

Save the following content to file `cemu.patch`.

```diff title="cemu.patch"
From 78c2470efcd6b285e3d0e73893d53b32c51b809b Mon Sep 17 00:00:00 2001
From: coekjan <cn_yzr@qq.com>
Date: Fri, 12 May 2023 23:44:37 +0800
Subject: [PATCH] Patch For CEMU

---
 .../boot/dts/loongson/loongson32_ls.dts       | 43 -------------------
 arch/loongarch/include/asm/time.h             |  2 +-
 la_build.sh                                   |  5 ++-
 3 files changed, 4 insertions(+), 46 deletions(-)

diff --git a/arch/loongarch/boot/dts/loongson/loongson32_ls.dts b/arch/loongarch/boot/dts/loongson/loongson32_ls.dts
index f33c0ce23..947605d40 100644
--- a/arch/loongarch/boot/dts/loongson/loongson32_ls.dts
+++ b/arch/loongarch/boot/dts/loongson/loongson32_ls.dts
@@ -56,49 +56,6 @@ cpu_uart0: serial@0x1fe001e0 {
                         no-loopback-test;
                 };
 
-        gmac0: dmfe@0x1ff00000{
-                        compatible = "dmfe";
-                        reg = <0x1ff00000 0x10000>;
-                        interrupt-parent = <&cpuic>;
-                        interrupts = <3>;
-                        interrupt-names = "macirq";
-                        mac-address = [ 64 48 48 48 48 60 ];/* [>mac 64:48:48:48:48:60 <]*/
-                        phy-mode = "rgmii";
-                        bus_id = <0x0>;
-                        phy_addr = <0xffffffff>;
-                        dma-mask = <0xffffffff 0xffffffff>;
-                };
-#if 0
-            ahci@0x1fe30000{
-                compatible = "snps,spear-ahci";
-                reg = <0x1fe30000 0x10000>;
-                interrupt-parent = <&cpuic>;
-                interrupts = <4>;
-                dma-mask = <0x0 0xffffffff>;
-            };
-#endif
-
-        nand@0x1fe78000{
-             #address-cells = <1>;
-             #size-cells = <1>;
-             compatible = "ls1a-nand";
-             reg = <0x1fe78000 0x4000
-                 0x1fd01160 0x0>;
-             interrupt-parent = <&cpuic>;
-             interrupts = <4>;
-             interrupt-names = "nand_irq";
-
-             number-of-parts = <0x2>;
-             partition@0 {
-                 label = "kernel_partition";
-                 reg = <0x0000000 0x01400000>;
-             };
-
-             partition@0x01400000 {
-                 label = "os_partition";
-                 reg = <0x01400000 0x0>;
-             };
-         };
 
     };
 };
diff --git a/arch/loongarch/include/asm/time.h b/arch/loongarch/include/asm/time.h
index d4cc3876d..1a348b561 100644
--- a/arch/loongarch/include/asm/time.h
+++ b/arch/loongarch/include/asm/time.h
@@ -41,7 +41,7 @@ static inline unsigned int calc_const_freq(void)
 #define LOONGARCH32_FREQ_REG 0x9fd0f030
 static inline unsigned int calc_const_freq(void)
 {
-    unsigned int freq = *(int *)LOONGARCH32_FREQ_REG;
+    unsigned int freq = 33333333;
 	return freq;
 }
 #endif
diff --git a/la_build.sh b/la_build.sh
index 7ce234a08..b32815540 100755
--- a/la_build.sh
+++ b/la_build.sh
@@ -1,6 +1,6 @@
 #!/bin/bash
 
-export CROSS_COMPILE=~/install-32-glibc-loongarch-novec-reduce-linux-5-14/bin/loongarch32-linux-gnu-
+export CROSS_COMPILE=loongarch32r-linux-gnusf-
 export ARCH=loongarch
 OUT=la_build
 
@@ -9,7 +9,8 @@ if [ ! -d la_build ] ;then
     make la32_defconfig O=${OUT}
 fi
 
+sed -i "/^CONFIG_INITRAMFS_SOURCE/s,=.*$,=\"$(realpath ../initrd_d)\"," ./la_build/.config
+
 echo "----------------output ${OUT}----------------"
 
-make menuconfig O=${OUT}
 make vmlinux -j  O=${OUT} 2>&1 | tee -a build_error.log
-- 
2.40.1


```

Then, apply the patch and build kernel.

```console
$ git apply cemu.patch
$ ./la_build.sh
$ loongarch32r-linux-gnusf-objcopy -O binary la_build/vmlinux la_build/vmlinux.bin
$ popd
```

### Build & RUN CEMU

Clone CEMU repository.

```console
$ git clone https://github.com/cyyself/cemu.git
$ cd cemu
```

Fetch `init_5f.txt` from chiplab.

```console
$ wget -q https://gitee.com/loongson-edu/chiplab/raw/chiplab_diff/software/linux/init_5f.txt
```

Save the following content to `src/main.cpp`

```cpp title="src/main.cpp"
#include <iostream>
#include <bitset>
#include <termios.h>
#include <unistd.h>
#include <thread>
#include <signal.h>

#include "device/nscscc_confreg.hpp"
#include "device/uart8250.hpp"
#include "memory/memory_bus.hpp"
#include "memory/ram.hpp"
#include "core/la32r/la32r_core.hpp"

void uart_input(uart8250 &uart) {
    termios tmp;
    tcgetattr(STDIN_FILENO, &tmp);
    tmp.c_lflag &= (~ICANON & ~ECHO);
    tcsetattr(STDIN_FILENO, TCSANOW, &tmp);
    while (true) {
        char c = getchar();
        if (c == 10) c = 13; // convert lf to cr
        uart.putc(c);
    }
}

bool send_ctrl_c;

void sigint_handler(int x) {
    static time_t last_time;
    if (time(NULL) - last_time < 1) exit(0);
    last_time = time(NULL);
    send_ctrl_c = true;
}

int main(int argc, char **argv) {
    signal(SIGINT, sigint_handler);

    memory_bus cemu_mmio;

    ram cemu_memory(128 * 1024 * 1024);
    cemu_memory.load_binary(0x300000, "../la32r-Linux/la_build/vmlinux.bin");
    cemu_memory.load_text(0x5f00000, "./init_5f.txt");
    assert(cemu_mmio.add_dev(0, 128 * 1024 * 1024, &cemu_memory));

    uart8250 uart;
    std::thread *uart_input_thread = new std::thread(uart_input, std::ref(uart));
    assert(cemu_mmio.add_dev(0x1fe001e0, 0x10, &uart));

    la32r_core<32> core(0, cemu_mmio, false);
    core.csr_cfg(0x180, 0xa0000001u);
    core.csr_cfg(0x181, 0x00000001u);
    core.csr_cfg(0x0, 0x10);
    core.reg_cfg(4, 2);
    core.reg_cfg(5, 0xa5f00000u);
    core.reg_cfg(6, 0xa5f00080u);
    core.jump(0xa07c5c28u);

    while (true) {
        core.step(uart.irq() << 1);
        while (uart.exist_tx()) {
            char c = uart.getc();
            if (c != '\r') {
                putchar(c);
                fflush(stdout);
            }
        }
        if (send_ctrl_c) {
            uart.putc(3);
            send_ctrl_c = false;
        }
    }
    return 0;
}
```

Finally, build and run CEMU.

```console
$ make  # in cemu project folder
$ ./cemu
[    0.000000] Linux version 5.14.0-rc2-gbb6b2ca56ade-dirty (@Inspiron-Manjaro) (loongarch32r-linux-gnusf-gcc (GCC) 8.3.0, GNU ld (GNU Binutils) 2.31.1.20190122) #3 PREEMPT Fri May 12 16:08:23 UTC 2023
[    0.000000] Standard 32-bit Loongson Processor probed
[    0.000000] the link is empty!
[    0.000000] Scan bootparam failed
[    0.000000] printk: bootconsole [early0] enabled
[    0.000000] initrd start < PAGE_OFFSET
[    0.000000] Can't find EFI system table.
[    0.000000] start_pfn=0x0, end_pfn=0x8000, num_physpages:0x8000
[    0.000000] The BIOS Version: (null)
[    0.000000] Initrd not found or empty - disabling initrd
[    0.000000] CPU0 revision is: 00004200 (Loongson-32bit)
[    0.000000] Primary instruction cache 8kB, 2-way, VIPT, linesize 16 bytes.
[    0.000000] Primary data cache 8kB, 2-way, VIPT, no aliases, linesize 16 bytes
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000000000000-0x00000000ffffffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000000000-0x0000000007ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x0000000007ffffff]
[    0.000000] eentry = 0xa0210000,tlbrentry = 0xa0201000
[    0.000000] pcpu-alloc: s0 r0 d32768 u32768 alloc=1*32768
[    0.000000] pcpu-alloc: [0] 0 
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32480
[    0.000000] Kernel command line: =/init loglevel=8
[    0.000000] Dentry cache hash table entries: 16384 (order: 4, 65536 bytes, linear)
[    0.000000] Inode-cache hash table entries: 8192 (order: 3, 32768 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 117736K/131072K available (4926K kernel code, 1114K rwdata, 944K rodata, 2480K init, 473K bss, 13336K reserved, 0K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu:     RCU event tracing is enabled.
[    0.000000]  Trampoline variant of Tasks RCU enabled.
[    0.000000]  Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] NR_IRQS: 320
[    0.000000] Constant clock event device register
[    0.000000] clocksource: Constant: mask: 0xffffffffffffffff max_cycles: 0x7b00c4bad, max_idle_ns: 440795202744 ns
[    0.000000] Constant clock source device register
[    0.004000] Console: colour dummy device 80x25
[    0.004000] Calibrating delay loop (skipped), value calculated using timer frequency.. 66.66 BogoMIPS (lpj=133333)
[    0.004000] pid_max: default: 32768 minimum: 301
[    0.008000] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.008000] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.036000] rcu: Hierarchical SRCU implementation.
[    0.040000] devtmpfs: initialized
[    0.052000] random: get_random_u32 called from bucket_table_alloc.isra.34+0x68/0x1d8 with crng_init=0
[    0.056000] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.056000] futex hash table entries: 256 (order: -1, 3072 bytes, linear)
[    0.064000] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.248000] pps_core: LinuxPPS API ver. 1 registered
[    0.248000] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.264000] clocksource: Switched to clocksource Constant
[    0.268000] FS-Cache: Loaded
[    0.384000] NET: Registered PF_INET protocol family
[    0.384000] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.400000] tcp_listen_portaddr_hash hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.400000] TCP established hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.400000] TCP bind hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.400000] TCP: Hash tables configured (established 1024 bind 1024)
[    0.400000] UDP hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.400000] UDP-Lite hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.404000] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.428000] workingset: timestamp_bits=14 max_order=15 bucket_order=1
[    0.620000] IPMI message handler: version 39.2
[    0.620000] ipmi device interface
[    0.620000] ipmi_si: IPMI System Interface driver
[    0.628000] ipmi_si: Unable to find any System Interface(s)
[    0.732000] Serial: 8250/16550 driver, 16 ports, IRQ sharing enabled
[    0.816000] printk: console [ttyS0] disabled
[    0.816000] 1fe001e0.serial: ttyS0 at MMIO 0x1fe001e0 (irq = 18, base_baud = 2062500) is a 16550A
[    0.816000] printk: console [ttyS0] enabled
[    0.816000] printk: console [ttyS0] enabled
[    0.816000] printk: bootconsole [early0] disabled
[    0.816000] printk: bootconsole [early0] disabled
[    0.840000] ls1a-nand driver initializing
[    0.840000] ITC MAC 10/100M Fast Ethernet Adapter driver 1.0 init
[    0.864000] libphy: Fixed MDIO Bus: probed
[    0.872000] mousedev: PS/2 mouse device common for all mice
[    0.888000] IR MCE Keyboard/mouse protocol handler initialized
[    0.888000] hid: raw HID events driver (C) Jiri Kosina
[    0.920000] NET: Registered PF_INET6 protocol family
[    0.948000] Segment Routing with IPv6
[    0.948000] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    1.016000] random: fast init done
[    5.424000] Warning: unable to open an initial console.
[    5.752000] Freeing unused kernel image (initmem) memory: 2480K
[    5.752000] This architecture does not have kernel memory protection.
[    5.752000] Run /init as init process
[    5.752000]   with arguments:
[    5.752000]     /init
[    5.752000]   with environment:
[    5.752000]     HOME=/
[    5.752000]     TERM=linux

Processing /etc/profile... Done

/ # 
```
