# MIPS Setup

## Build cross-compile Toolchains

We recommand use [crosstool-NG](https://github.com/crosstool-ng/crosstool-ng).

```bash
git clone https://github.com/crosstool-ng/crosstool-ng
pushd crosstool-ng
    ./bootstrap
    sudo mkdir /opt/ct-ng
    sudo chown -R $USER /opt/ct-ng
    ./configure --prefix=/opt/ct-ng
    make -j `nproc`
    make install
    export PATH=/opt/ct-ng/bin:$PATH
popd
sudo mkdir /opt/mips32
sudo chown -R $USER /opt/mips32
pushd /opt/mips32
    ct-ng mipsel-unknown-linux-gnu
    sed -i 's/CT_ARCH_ARCH="mips1"/CT_ARCH_ARCH="mips32"/g' .config
    ct-ng build
    # Now, toolchains available at `/opt/mips32/.build/mipsel-unknown-linux-gnu/buildtools/bin`
    echo "export PATH=$HOME/x-tools/mipsel-unknown-linux-gnu/bin:\$PATH" >> ~/.bashrc
    source ~/.bashrc
    # Replace .bashrc if you are using some shells other than bash, such as .zshrc for zsh.
popd
```

## uCore boot

```bash
git clone https://github.com/cyyself/ucore-thumips.git
pushd ucore-thumips
    make GCCPREFIX=mipsel-unknown-linux-gnu-
popd
git clone https://github.com/cyyself/cemu.git
pushd cemu
```

Save the following file to `cemu/src/main.cpp`

```cpp title="src/main.cpp"
#include <iostream>
#include <bitset>
#include <cassert>
#include <thread>
#include <termios.h>
#include <cassert>
#include <unistd.h>
#include <csignal>
#include <fcntl.h>

#include "mips_core.hpp"
#include "nscscc_confreg.hpp"
#include "memory_bus.hpp"
#include "ram.hpp"
#include "uart8250.hpp"

void uart_input(uart8250 &uart) {
    termios tmp;
    tcgetattr(STDIN_FILENO,&tmp);
    tmp.c_lflag &=(~ICANON & ~ECHO);
    tcsetattr(STDIN_FILENO,TCSANOW,&tmp);
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

void ucore_run(int argc, const char* argv[]) {
    signal(SIGINT, sigint_handler);

    memory_bus cemu_mmio;

    ram cemu_memory(128*1024*1024, "../ucore-thumips/obj/ucore-kernel-initrd.bin");
    assert(cemu_mmio.add_dev(0,128*1024*1024,&cemu_memory));

    // uart8250 at 0x1fe40000 (APB)
    uart8250 uart;
    std::thread *uart_input_thread = new std::thread(uart_input,std::ref(uart));
    assert(cemu_mmio.add_dev(0x1fe40000,0x10000,&uart));

    mips_core mips(cemu_mmio);
    mips.jump(0x80000000u);
    uint32_t lastpc = 0;
    bool delay_cr = false;
    while (true) {
        mips.step(uart.irq() << 1);
        while (uart.exist_tx()) {
            char c = uart.getc();
            if (c == '\r') delay_cr = true;
            else {
                if (delay_cr && c != '\n') std::cout << "\r" << c;
                else std::cout << c;
                std::cout.flush();
                delay_cr = false;
            }
        }
        if (send_ctrl_c) {
            uart.putc(3);
            send_ctrl_c = false;
        }
    }
}

int main(int argc, const char* argv[]) {
    ucore_run(argc, argv);
    return 0;
}
```

```bash
# at cemu project folder
make
./cemu
++setup timer interrupts
Initrd: 0x8006aa50 - 0x800c224f, size: 0x00057800, magic: 0x2f8dbe2a
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x80000108 (phys)
  etext 0x8002B400 (phys)
  edata 0x800C2250 (phys)
  end   0x800C5570 (phys)
Kernel executable memory footprint: 617KB
memory management: buddy_pmm_manager
memory map:
    [80000000, 82000000]

freemem start at: 80106000
free pages: 00001EFA
## 00000020
check_alloc_page() succeeded!
check_pgdir() succeeded!
check_boot_pgdir() succeeded!
-------------------- BEGIN --------------------
--------------------- END ---------------------
check_slab() succeeded!
kmalloc_init() succeeded!
check_vma_struct() succeeded!
check_pgfault() succeeded!
check_vmm() succeeded.
sched class: RR_scheduler
ramdisk_init(): initrd found, magic: 0x2f8dbe2a, 0x000002bc secs
sfs: mount: 'simple file system' (81/6/87)
vfs: mount disk0.
kernel_execve: pid = 2, name = "sh".
user sh is running!!!
$ 
```

## Linux boot

### Build Busybox and prepare rootfs

```bash
git clone https://github.com/mirror/busybox.git
pushd busybox
    make ARCH=mips CROSS_COMPILE=mipsel-unknown-linux-gnu- defconfig
    sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/g' .config
    # Note: If you don't want static build, you need to copy the entire ~/x-tools/mipsel-unknown-linux-gnu/mipsel-unknown-linux-gnu/sysroot/ to your rootfs, it will be too huge for initrd by default
    make -j `nproc`
    make install
popd busybox
mkdir rootfs
pushd rootfs
    sudo rsync -ar ../busybox/_install/* .
    sudo chown root:root * -R
    sudo mkdir -p proc sys dev etc/init.d
    sudo mknod -m 622 dev/console c 5 1
    sudo mknod -m 666 dev/null c 1 3
    sudo mknod -m 666 dev/zero c 1 5
    sudo mknod -m 666 dev/ptmx c 5 2
    sudo mknod -m 666 dev/tty c 5 0
    sudo mknod -m 444 dev/random c 1 8
    sudo mknod -m 444 dev/urandom c 1 9
    sudo chown root:tty dev/{console,ptmx,tty}
    sudo sh -c 'echo "#!/bin/sh\nmount -t proc none /proc\nmount -t sysfs none /sys\nln -s /dev/null /dev/tty2\nln -s /dev/null /dev/tty3\nln -s /dev/null /dev/tty4\nexit 0" > etc/init.d/rcS'
    sudo chmod a+x etc/init.d/rcS
popd
```

### Build Linux Kernel

```bash
git clone https://github.com/cyyself/linux.git -b nscscc_cdim
pushd linux
    make ARCH=mips CROSS_COMPILE=mipsel-unknown-linux-gnu- cdim_defconfig
    # Disable axi intc and axi emaclite in device-tree
    sed -i 's/status = "okay"/status = "disabled"/g' arch/mips/boot/dts/nscscc/cdim.dts
    make ARCH=mips CROSS_COMPILE=mipsel-unknown-linux-gnu- -j `nproc`
    mipsel-unknown-linux-gnu-objcopy -O binary vmlinux vmlinux.bin
popd
git clone https://github.com/cyyself/cemu.git
pushd cemu
```

Save the following file to `cemu/src/main.cpp`

```cpp title="src/main.cpp"
#include <iostream>
#include <bitset>
#include <cassert>
#include <thread>
#include <termios.h>
#include <cassert>
#include <unistd.h>
#include <csignal>
#include <fcntl.h>

#include "mips_core.hpp"
#include "nscscc_confreg.hpp"
#include "memory_bus.hpp"
#include "ram.hpp"
#include "uart8250.hpp"

void uart_input(uart8250 &uart) {
    termios tmp;
    tcgetattr(STDIN_FILENO,&tmp);
    tmp.c_lflag &=(~ICANON & ~ECHO);
    tcsetattr(STDIN_FILENO,TCSANOW,&tmp);
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

void linux_run(int argc, const char* argv[]) {
    signal(SIGINT, sigint_handler);

    memory_bus cemu_mmio;

    ram cemu_memory(128*1024*1024);
    cemu_memory.load_binary(0x100000, "../linux/vmlinux.bin");
    assert(cemu_mmio.add_dev(0,128*1024*1024,&cemu_memory));

    // uart8250 at 0x1fe40000 (APB)
    uart8250 uart;
    std::thread *uart_input_thread = new std::thread(uart_input,std::ref(uart));
    assert(cemu_mmio.add_dev(0x1fe40000,0x10000,&uart));

    mips_core mips(cemu_mmio);
    mips.jump(0x80100000u);
    uint32_t lastpc = 0;
    bool delay_cr = false;
    while (true) {
        mips.step(uart.irq() << 1);
        while (uart.exist_tx()) {
            char c = uart.getc();
            if (c == '\r') delay_cr = true;
            else {
                if (delay_cr && c != '\n') std::cout << "\r" << c;
                else std::cout << c;
                std::cout.flush();
                delay_cr = false;
            }
        }
        if (send_ctrl_c) {
            uart.putc(3);
            send_ctrl_c = false;
        }
    }
}

int main(int argc, const char* argv[]) {
    linux_run(argc, argv);
    return 0;
}
```

```bash
# at cemu project folder
make
./cemu                                                                             
[    0.000000] Linux version 6.3.0-13467-g75fb980ad88e-dirty (cyy@cyy-pc) (mipsel-unknown-linux-gnu-gcc (crosstool-NG 1.25.0.166_9433647) 12.2.0, GNU ld (crosstool-NG 1.25.0.166_9433647) 2.40) #16 Mon May  8 00:56:24 CST 2023
[    0.000000] printk: bootconsole [early0] enabled
[    0.000000] CPU0 revision is: 00018003 (MIPS 4Kc)
[    0.000000] MIPS: machine is nscscc,CQU_Dual_Issue_Machine
[    0.000000] Malformed early option 'earlycon'
[    0.000000] Initrd not found or empty - disabling initrd
[    0.000000] Primary instruction cache 8kB, VIPT, 2-way, linesize 64 bytes.
[    0.000000] Primary data cache 8kB, 2-way, VIPT, no aliases, linesize 64 bytes
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000000000000-0x0000000007ffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000000000-0x0000000007ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x0000000007ffffff]
[    0.000000] Kernel command line: console=ttyS0,115200n8 rdinit=/sbin/init earlycon
[    0.000000] Dentry cache hash table entries: 16384 (order: 4, 65536 bytes, linear)
[    0.000000] Inode-cache hash table entries: 8192 (order: 3, 32768 bytes, linear)
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32480
[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off
[    0.000000] Memory: 118096K/131072K available (7257K kernel code, 610K rwdata, 1096K rodata, 2484K init, 216K bss, 12976K reserved, 0K cma-reserved)
[    0.000000] SLUB: HWalign=32, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS: 256
[    0.000000] arch_init_irq with ebase: 0x80000000
[    0.000000] clocksource: MIPS: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 57337813970 ns
[    0.000009] sched_clock: 32 bits at 33MHz, resolution 30ns, wraps every 64424509936ns
[    0.002404] Console: colour dummy device 80x25
[    0.002804] Calibrating delay loop... 33.15 BogoMIPS (lpj=66304)
[    0.052502] pid_max: default: 32768 minimum: 301
[    0.054988] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.055391] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.073476] cblist_init_generic: Setting adjustable number of callback queues.
[    0.073746] cblist_init_generic: Setting shift to 0 and lim to 1.
[    0.080127] devtmpfs: initialized
[    0.097662] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.098119] futex hash table entries: 256 (order: -1, 3072 bytes, linear)
[    0.117142] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.155162] SCSI subsystem initialized
[    0.167799] clocksource: Switched to clocksource MIPS
[    0.306714] NET: Registered PF_INET protocol family
[    0.310480] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.323866] tcp_listen_portaddr_hash hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.324632] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.325076] TCP established hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.325755] TCP bind hash table entries: 1024 (order: 1, 8192 bytes, linear)
[    0.326501] TCP: Hash tables configured (established 1024 bind 1024)
[    0.327416] UDP hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.328059] UDP-Lite hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.329948] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.334508] RPC: Registered named UNIX socket transport module.
[    0.334752] RPC: Registered udp transport module.
[    0.334971] RPC: Registered tcp transport module.
[    0.335190] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.364317] workingset: timestamp_bits=14 max_order=15 bucket_order=1
[    0.374149] NFS: Registering the id_resolver key type
[    0.374893] Key type id_resolver registered
[    0.375109] Key type id_legacy registered
[    0.375614] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.375983] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    0.376492] fuse: init (API version 7.38)
[    0.383223] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 254)
[    0.383516] io scheduler mq-deadline registered
[    0.383750] io scheduler kyber registered
[    1.693393] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    1.734751] printk: console [ttyS0] disabled
[    1.735135] 1fe40000.serial: ttyS0 at MMIO 0x1fe40000 (irq = 3, base_baud = 6250000) is a 16550A
[    1.735707] printk: console [ttyS0] enabled
[    1.735707] printk: console [ttyS0] enabled
[    1.736247] printk: bootconsole [early0] disabled
[    1.736247] printk: bootconsole [early0] disabled
[    1.753251] NET: Registered PF_INET6 protocol family
[    1.789533] Segment Routing with IPv6
[    1.791205] In-situ OAM (IOAM) with IPv6
[    1.792675] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    1.801848] NET: Registered PF_PACKET protocol family
[    1.803474] Key type dns_resolver registered
[    2.182276] Key type .fscrypt registered
[    2.182572] Key type fscrypt-provisioning registered
[    2.193653] clk: Disabling unused clocks
[    3.596393] Freeing unused kernel image (initmem) memory: 2484K
[    3.596670] This architecture does not have kernel memory protection.
[    3.596976] Run /sbin/init as init process

Please press Enter to activate this console. 
~ # 
```