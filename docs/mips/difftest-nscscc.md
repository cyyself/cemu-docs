# Differential Test in NSCSCC

[The CPU design track of the National Student Computer System Capability Challenge (NSCSCC)](http://www.nscscc.com/), also known as the "Loongson Cup" (“龙芯杯” in Chinese) is a competition for college students in China.

We can use CEMU with [SoC-Simulator](https://github.com/cyyself/soc-simulator/) to help the competitors in various way, such as:

- Much faster simulation speed compared to Vivado
- Differential Test when running Linux / uCore
- Add some performance counter in software to profile the performance

## Prerequisites

[Verilator](https://verilator.org/guide/latest/install.html) >= v4.2

!!! Warning

    Note for Debian/Ubuntu Users:
    If your Ubuntu version < 23.04(lunar) or Debian version < 12(bookworm), you can **NOT** install verilator directly through `apt install verilator`, the verilator in offcial repo of your distro is too old that can not build soc-simulator.

## Simple usage for function test and performance test

If you only need to speed up NSCSCC function test and performance test using [Verilator](https://github.com/verilator/verilator) instead of Vivado, you can use the [Lab Package from Chongqing University Computer Organization Course - Integrated Hardware Design](https://github.com/CQU-CS-LABs/CO-lab-material-CQU/tree/2022/verilator). 

And we also have the tutorial video on [bilibili](https://www.bilibili.com/video/BV19G4y1E7jj/).

Both the README in the package and the video are in Chinese.

!!! Note
    We have modified the function test in the lab package as we use 7a100t FPGA(Diligent Nexys4 DDR) for the Lab course instead of 7a200t in NSCSCC, and the BRAM resource is very limited in 7a100t, so the function test in the CQU lab package will be slightly different on memory layout from the function test in nscscc-group package.
    
    You can replace the files in the `vivado/func_test_v0.01/soft/` in our package if you want to test your CPU RTL with the original function test from nscscc, which will not affect the verilator framework.

### How does CEMU and SoC-Simulator dial with different counter value in differential performance test?

The performance test will read 2 performance counter:

- CP0.count
- CONFREG.timer

We have `debug_wb_is_timer` flag in CEMU, it will be reset to false at every step one instruction, then if the instruction load the CONFREG.timer in kseg1 or executed mfc0 to load CP0.counter, this flag will set to true to tell the diffential test function that the value may different between RTL and CEMU, so the differetial test function will call CEMU to update the GPR commit value from CPU RTL and thus we will have the same value.

??? code

    ```diff
    diff --git a/src/core/mips/mips_core.hpp b/src/core/mips/mips_core.hpp
    index 4321630..bea2270 100644
    --- a/src/core/mips/mips_core.hpp
    +++ b/src/core/mips/mips_core.hpp
    @@ -46,6 +46,7 @@ public:
            debug_wb_wen = 0;
            debug_wb_wnum = 0;
            debug_wb_wdata = 0;
    +        debug_wb_is_timer = 0;
            j_cnt = 0;
            forward_branch = 0;
            forward_branch_taken = 0;
    @@ -76,6 +77,7 @@ public:
        uint8_t  debug_wb_wen;
        uint8_t  debug_wb_wnum;
        uint32_t debug_wb_wdata;
    +    bool     debug_wb_is_timer;
        std::queue <uint32_t> pc_trace;
        std::set <uint32_t> cache_op;
        // TODO: trace with exceptions (add exception signal at commit stage is need)
    @@ -91,6 +93,7 @@ private:
            debug_wb_wen = 0;
            debug_wb_wnum = 0;
            debug_wb_wdata = 0;
    +        debug_wb_is_timer = false;
            mips_instr instr;
            mips32_exccode if_exc = EXC_OK;
            pc_trace.push(pc);
    @@ -727,6 +730,7 @@ private:
                    // LW
                    uint32_t vaddr = GPR[instr.i_type.rs] + instr.i_type.imm;
                    uint32_t buf;
    +                if (vaddr == 0xbfafe000u) debug_wb_is_timer = true; // for difftest
                    mips32_exccode stat = mmu.va_read(vaddr, 4, (unsigned char*)&buf, cp0.get_ksu(), cp0.get_asid(), tlb_invalid);
                    if (stat != EXC_OK) cp0.raise_trap(stat, vaddr, tlb_invalid);
                    else set_GPR(instr.i_type.rt, buf);
    @@ -870,6 +874,7 @@ private:
                        switch (instr.r_type.rs) {
                            case RS_MFC0: {
                                // MFC0
    +                            if (instr.r_type.rd == RD_COUNT && (instr.r_type.funct & 0b111) == 0) debug_wb_is_timer = true; // for difftest
                                set_GPR(instr.r_type.rt, cp0.mfc0(instr.r_type.rd, instr.r_type.funct&0b111));
                                break;
                            }
    @@ -927,6 +932,7 @@ private:
                    // LL as LW
                    uint32_t vaddr = GPR[instr.i_type.rs] + instr.i_type.imm;
                    uint32_t buf;
    +                if (vaddr == 0xbfafe000u) debug_wb_is_timer = true; // for difftest
                    mips32_exccode stat = mmu.va_read(vaddr, 4, (unsigned char*)&buf, cp0.get_ksu(), cp0.get_asid(), tlb_invalid);
                    if (stat != EXC_OK) cp0.raise_trap(stat, vaddr, tlb_invalid);
                    else set_GPR(instr.i_type.rt, buf);
    ```

## Differential Test when running operating systems

### Adding debug signals

Originally, NSCSCC provide the following debug signals:

```verilog
output [31:0]   debug_wb_pc,
output [ 3:0]   debug_wb_rf_wen,
output [ 4:0]   debug_wb_rf_wnum,
output [31:0]   debug_wb_rf_wdata,
```

??? note
    With the help of the **open_trace** flag in **NSCSCC confreg** to turn off the trace when the CPU behavior is different with golden trace but acceptable (such as CP0.status.BD in n77 software interrupt test), it will be easy to allow different behavior if the different GPR write data will only be propagated in a very limited range in control flow.

    However, the operating systems are much more complex than the function test because they will have timer interrupts. Although the emulator should know the accurate cycle to generate an interrupt, the CPU may not interrupt immediately.

    For example, [CQU Dual Issue Machine](https://github.com/Maxpicca-Li/CDIM) is a dual-issue five-stage pipeline processor which binds interrupts to a specific PC at the 2nd stage(decode). However, the interrupt will raise at the 4th stage(memory), so it will have at least 2 more instruction being executed after the interrupt signal fired, thus will have different behavior between emulator and CPU RTL.

    This behavior is called asynchronous interrupt, which is allowed in almost every ISA.

    Moreover, MIPS introduces TLBWR (TLB write random) instruction, and which entry in TLB to write depends on the value of CP0.random. However, the differential test can not directly know the value in CP0.random and when the CP0.random value is taken for TLBWR instruction though this behavior is CPU implementation dependent. Thus will have different TLB states between the emulator and CPU RTL.

    Here comes a challenge: How to differential test when interrupts are on and trace the TLB random write?

    To solve the interrupt synchonize problem, [SoC-Simulator](https://github.com/cyyself/soc-simulator) adds two additional debug signals of a specific PC in insturction commit trace, which are CP0.cause value and whether the interrupt fired. 

    ```verilog
    output [31:0]   debug_cp0_cause, // CP0.cause
    output          debug_int // whether the interrupt fired
    ```
    
    In this case, differential test can be implemented in the following way (pseudocode):

    ```c
    while (1) { // main loop
        RTL.step_one_instruction();
        CEMU.import_diff_test_info(RTL.debug_cp0_cause, RTL.debug_int);
        // CEMU will take the cp0 cause value to update its CSR.
        // And it will only raise interrupt when RTL.debug_int is 1.
        CEMU.step_one_instructino();
        compare_commit_result(RTL, CEMU); // check RTL behavior
    }
    ```

    Since it will only have unpredictable behavior in TLBWR among all TLB behavior, what emulator should do is simply taken the CP0.random value used for executing TLBWR.


    ```verilog
    output [31:0] debug_cp0_ramdom
    ```

SoC-Simulator extended the debug signals to:

```verilog
// nscscc debug interface
output [31:0]   debug_wb_pc,
output [ 3:0]   debug_wb_rf_wen,
output [ 4:0]   debug_wb_rf_wnum,
output [31:0]   debug_wb_rf_wdata,
// soc-simulator + cemu debug interface
output [31:0]   debug_cp0_ramdom,// cp0_random used in TLBWR
output [31:0]   debug_cp0_cause, // cp0_cause for rising interrupts and mfc0
output          debug_int,       
output          debug_commit
```

!!! warning
    
    It is worth noting that if the instruction is going to interrupt, the CPU RTL should also assert `debug_commit` to tell the SoC-Simulator that this instruction is retired even though it will not commit the GPR, and we should **NOT** assert `debug_wb_rf_wen` though the this instruction will not commit to GPR.

    And the CPU RTL should **NOT** assert the `debug_wb_rf_wen` and `debug_commit` during pipeline stall.

!!! warning

    Additionally, SoC-Simulator also support commit GPR on both the rising and falling edges of the clock signal, which is `both_edge_commit` defines in `sim_mycpu.cpp`, it will be convient for superscalar CPU which commit width is 2.

    If your CPU does not support it, you should manually **turn off** it by replace `bool both_edge_commit = true;` with `bool both_edge_commit = false;`.


### Get SoC-Simulator and Running


Assume you are in a folder like this:

```console
.
├── linux
├── nscscc-group # You can get from nscscc.com
├── supervisor-mips32 # 
├── u-boot # you can get from `git clone
└── ucore-thumips # compiled in setup.md
```

How to get these subfolders:

- linux: build from [setup.md](./setup.md)
- ucore-thumips: build from [setup.md](./setup.md)
- nscscc-group: From NSCSCC 2022 group package (www.nscscc.com)
- supervisor-mips32
    ```bash
    git clone https://github.com/cyyself/supervisor-mips32.git -b nscscc
    cd supervisor-mips32/kernel
    make GCCPREFIX=mipsel-unknown-linux-gnu- -j `nproc`
    ```
- u-boot
    ```bash
    git clone https://github.com/cyyself/u-boot.git -b cdim_soc
    cd u-boot
    make ARCH=mips CROSS_COMPILE=mipsel-unknown-linux-gnu- cdim_defconfig
    make ARCH=mips CROSS_COMPILE=mipsel-unknown-linux-gnu- -j `nproc`
    ```

Then, get soc-simulator:

```bash
git clone https://github.com/cyyself/soc-simulator.git
cd soc-simulator
bash ./scripts/init_cdim.sh
# If you dont want CDIM, you can use
# bash ./scripts/init_nscscc.sh
# Then, change the CPU source file path in Makefile
```

### Running Arguments

#### Running Mode

- `-func` NSCSCC Functional Test
- `-perf` NSCSCC Performance Test
- `-sys`  NSCSCC System Test

    Note: You may need `socat` or `nc` to convert stdio to TCP socket.
    Example:
    ```bash
    socat tcp-listen:6666,reuseaddr exec:'./obj_dir/Vmycpu_top -sys' &
    cd ../supervisor-mips32/term/
    GCCPREFIX=mipsel-unknown-linux-gnu- python3 term.py -t 127.0.0.1:6666
    ```
    
- `-linux` Run Linux
- `-ucore` Run uCore
- `-uboot` Run U-Boot

#### Arguments

- `-axifast` Turn off AXI Latency

    We have explored the latency to approach the performance test.
    
    Turning it off will not significantly improve performance if your CPU has a Cache.

- `-nodiff` Turn off diffential Test.

    It will be useful when your CPU behavior at the ISA level is different from CEMU but acceptable, you can check the behaviors of software in this mode.

- `-trace [time]` Turn on trace at start and set the trace time (in ticks, which is double of cycles).
- `-starttrace [time]` Set trace start time.

    By default, it will recode the waveform from this time to this time + 1000 ticks.

- `-pc` Print PC every time.

- `-diffmem` Differential Test AXI Memory Write (For Cache Debug).

!!! Note

    All the waveforms will be saved to `trace.fst`.

#### Arguments for Performance Test Only

- `-perfonce`: Run every performance test only once.
    
    Every test will run for 10 times by default.

- `-uart`: Output confreg UART

    You will see how the performance test is going.

- `-diffuart`: Differential Test confreg UART

    For MMIO Debug.

- `-stat`: Output statistic result

#### Keyboard function when running mode is not func and perf

- `Ctrl` + `i`: Print Current PC (Program Counter)

- `Ctrl` + `t`: recode waveform from current time to current time + 1000 ticks.


#### Example


- Linux run with differential test:

```console
./obj_dir/Vmycpu_top -linux
[    0.000000] Linux version 6.3.0-13467-g63f399f0f695-dirty (cyy@cyy-pc) (mipsel-unknown-linux-gnu-gcc (crosstool-NG 1.25.0.166_9433647) 12.2.0, GNU ld (crosstool-NG 1.25.0.166_9433647) 2.40) #25 Mon May 15 00:25:31 CST 2023
[    0.000000] printk: bootconsole [early0] enabled
[    0.000000] CPU0 revision is: 00018003 (MIPS 4Kc)
[    0.000000] MIPS: machine is nscscc,CQU_Dual_Issue_Machine
[    0.000000] earlycon: ns16550a0 at MMIO 0x1fe40000 (options '')
[    0.000000] printk: bootconsole [ns16550a0] enabled
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
[    0.000000] Memory: 117968K/131072K available (7347K kernel code, 620K rwdata, 1124K rodata, 2484K init, 216K bss, 13104K reserved, 0K cma-reserved)
[    0.000000] SLUB: HWalign=32, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS: 256
[    0.000000] arch_init_irq with ebase: 0x80000000
[    0.000000] clocksource: MIPS: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 57337813970 ns
[    0.000022] sched_clock: 32 bits at 33MHz, resolution 30ns, wraps every 64424509936ns
[    0.002330] Console: colour dummy device 80x25
[    0.003456] Calibrating delay loop... 65.79 BogoMIPS (lpj=131584)
[    0.045612] pid_max: default: 32768 minimum: 301
[    0.050073] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.050854] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.084261] cblist_init_generic: Setting adjustable number of callback queues.
[    0.084859] cblist_init_generic: Setting shift to 0 and lim to 1.
[    0.097443] devtmpfs: initialized
[    0.132962] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.134126] futex hash table entries: 256 (order: -1, 3072 bytes, linear)
[    0.158071] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.227071] SCSI subsystem initialized
[    0.232633] pps_core: LinuxPPS API ver. 1 registered
[    0.233502] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.234564] PTP clock support registered
[    0.251011] clocksource: Switched to clocksource MIPS
[    0.527496] NET: Registered PF_INET protocol family
[    0.532125] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.541770] tcp_listen_portaddr_hash hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.542758] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.543920] TCP established hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.544806] TCP bind hash table entries: 1024 (order: 1, 8192 bytes, linear)
[    0.545774] TCP: Hash tables configured (established 1024 bind 1024)
[    0.547945] UDP hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.548856] UDP-Lite hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.552318] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.561463] RPC: Registered named UNIX socket transport module.
[    0.562018] RPC: Registered udp transport module.
[    0.562506] RPC: Registered tcp transport module.
[    0.563332] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.597632] workingset: timestamp_bits=14 max_order=15 bucket_order=1
[    0.642610] NFS: Registering the id_resolver key type
[    0.644141] Key type id_resolver registered
[    0.644644] Key type id_legacy registered
[    0.645673] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.646263] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    0.647589] fuse: init (API version 7.38)
[    0.677156] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 252)
[    0.677799] io scheduler mq-deadline registered
[    0.678312] io scheduler kyber registered
[    3.392578] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    3.455389] printk: console [ttyS0] disabled
[    3.456242] 1fe40000.serial: ttyS0 at MMIO 0x1fe40000 (irq = 3, base_baud = 6250000) is a 16550A
[    3.457318] printk: console [ttyS0] enabled
[    3.457318] printk: console [ttyS0] enabled
[    3.458046] printk: bootconsole [early0] disabled
[    3.458046] printk: bootconsole [early0] disabled
[    3.459564] printk: bootconsole [ns16550a0] disabled
[    3.459564] printk: bootconsole [ns16550a0] disabled
[    3.526708] NET: Registered PF_INET6 protocol family
[    3.594829] Segment Routing with IPv6
[    3.596981] In-situ OAM (IOAM) with IPv6
[    3.598494] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    3.613067] NET: Registered PF_PACKET protocol family
[    3.615350] Key type dns_resolver registered
[    4.306439] Key type .fscrypt registered
[    4.306856] Key type fscrypt-provisioning registered
[    4.333894] clk: Disabling unused clocks
[    4.944821] Freeing unused kernel image (initmem) memory: 2484K
[    4.945250] This architecture does not have kernel memory protection.
[    4.945665] Run /sbin/init as init process

Please press Enter to activate this console. 
~ # 
```

- Linux run **without** differential test **without** AXI latency:

```console
./obj_dir/Vmycpu_top -linux -nodiff -axifast
```

You will get the same output as above but different in times because of AXI latency, you will get about 40% performance improvement compared to above.

- Performance Test

```console
./obj_dir/Vmycpu_top -perf          
1b654
befbc
1e1b06
15160f
44dc7
ccd6a
d570f
bc871
d7f9
9a10e
total ticks = 30948883
```

Direct copy these lines to `IPC score calculation sheet` from nscscc final packge, you will get IPC score.

- Performance Test with statistics

```console
./obj_dir/Vmycpu_top -perf -stat    
1b654
total_clk = 231658, stall_clk = 12545, has_commit = 198308, dual_commit = 131665, dual_issue_rate = 0.66394, IPC = 1.42440
confreg_read: 28, confreg_write: 502
befbc
total_clk = 1571488, stall_clk = 88514, has_commit = 1115394, dual_commit = 593678, dual_issue_rate = 0.53226, IPC = 1.08755
confreg_read: 8, confreg_write: 156
1e1b06
total_clk = 3952382, stall_clk = 608646, has_commit = 2581866, dual_commit = 1511853, dual_issue_rate = 0.58557, IPC = 1.03576
confreg_read: 28, confreg_write: 7915
15160f
total_clk = 2769825, stall_clk = 232059, has_commit = 1615358, dual_commit = 947307, dual_issue_rate = 0.58644, IPC = 0.92521
confreg_read: 8, confreg_write: 9592
44dc7
total_clk = 570600, stall_clk = 109159, has_commit = 341275, dual_commit = 147268, dual_issue_rate = 0.43152, IPC = 0.85619
confreg_read: 28, confreg_write: 17902
ccd6a
total_clk = 1685304, stall_clk = 164488, has_commit = 1177940, dual_commit = 535953, dual_issue_rate = 0.45499, IPC = 1.01696
confreg_read: 8, confreg_write: 152
d570f
total_clk = 1755581, stall_clk = 44135, has_commit = 1195724, dual_commit = 585725, dual_issue_rate = 0.48985, IPC = 1.01473
confreg_read: 8, confreg_write: 156
bc871
total_clk = 1550455, stall_clk = 83471, has_commit = 1094799, dual_commit = 657087, dual_issue_rate = 0.60019, IPC = 1.12992
confreg_read: 8, confreg_write: 2214
d7f9
total_clk = 117860, stall_clk = 10504, has_commit = 74013, dual_commit = 43755, dual_issue_rate = 0.59118, IPC = 0.99922
confreg_read: 8, confreg_write: 154
9a10e
total_clk = 1269288, stall_clk = 147152, has_commit = 857688, dual_commit = 331955, dual_issue_rate = 0.38703, IPC = 0.93725
confreg_read: 8, confreg_write: 32134
total ticks = 30948883
```

- Function Test

```console
./obj_dir/Vmycpu_top -func                  
Number 1 Functional Test Point PASS!
Number 2 Functional Test Point PASS!
Number 3 Functional Test Point PASS!
Number 4 Functional Test Point PASS!
Number 5 Functional Test Point PASS!
Number 6 Functional Test Point PASS!
Number 7 Functional Test Point PASS!
Number 8 Functional Test Point PASS!
Number 9 Functional Test Point PASS!
Number 10 Functional Test Point PASS!
Number 11 Functional Test Point PASS!
Number 12 Functional Test Point PASS!
Number 13 Functional Test Point PASS!
Number 14 Functional Test Point PASS!
Number 15 Functional Test Point PASS!
Number 16 Functional Test Point PASS!
Number 17 Functional Test Point PASS!
Number 18 Functional Test Point PASS!
Number 19 Functional Test Point PASS!
Number 20 Functional Test Point PASS!
Number 21 Functional Test Point PASS!
Number 22 Functional Test Point PASS!
Number 23 Functional Test Point PASS!
Number 24 Functional Test Point PASS!
Number 25 Functional Test Point PASS!
Number 26 Functional Test Point PASS!
Number 27 Functional Test Point PASS!
Number 28 Functional Test Point PASS!
Number 29 Functional Test Point PASS!
Number 30 Functional Test Point PASS!
Number 31 Functional Test Point PASS!
Number 32 Functional Test Point PASS!
Number 33 Functional Test Point PASS!
Number 34 Functional Test Point PASS!
Number 35 Functional Test Point PASS!
Number 36 Functional Test Point PASS!
Number 37 Functional Test Point PASS!
Number 38 Functional Test Point PASS!
Number 39 Functional Test Point PASS!
Number 40 Functional Test Point PASS!
Number 41 Functional Test Point PASS!
Number 42 Functional Test Point PASS!
Number 43 Functional Test Point PASS!
Number 44 Functional Test Point PASS!
Number 45 Functional Test Point PASS!
Number 46 Functional Test Point PASS!
Number 47 Functional Test Point PASS!
Number 48 Functional Test Point PASS!
Number 49 Functional Test Point PASS!
Number 50 Functional Test Point PASS!
Number 51 Functional Test Point PASS!
Number 52 Functional Test Point PASS!
Number 53 Functional Test Point PASS!
Number 54 Functional Test Point PASS!
Number 55 Functional Test Point PASS!
Number 56 Functional Test Point PASS!
Number 57 Functional Test Point PASS!
Number 58 Functional Test Point PASS!
Number 59 Functional Test Point PASS!
Number 60 Functional Test Point PASS!
Number 61 Functional Test Point PASS!
Number 62 Functional Test Point PASS!
Number 63 Functional Test Point PASS!
Number 64 Functional Test Point PASS!
Number 65 Functional Test Point PASS!
Number 66 Functional Test Point PASS!
Number 67 Functional Test Point PASS!
Number 68 Functional Test Point PASS!
Number 69 Functional Test Point PASS!
Number 70 Functional Test Point PASS!
Number 71 Functional Test Point PASS!
Number 72 Functional Test Point PASS!
Number 73 Functional Test Point PASS!
Number 74 Functional Test Point PASS!
Number 75 Functional Test Point PASS!
Number 76 Functional Test Point PASS!
Number 77 Functional Test Point PASS!
Number 78 Functional Test Point PASS!
Number 79 Functional Test Point PASS!
Number 80 Functional Test Point PASS!
Number 81 Functional Test Point PASS!
Number 82 Functional Test Point PASS!
Number 83 Functional Test Point PASS!
Number 84 Functional Test Point PASS!
Number 85 Functional Test Point PASS!
Number 86 Functional Test Point PASS!
Number 87 Functional Test Point PASS!
Number 88 Functional Test Point PASS!
Number 89 Functional Test Point PASS!
```
