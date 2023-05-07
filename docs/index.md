# CEMU Documentation

## Introduction

[CEMU](https://github.com/cyyself/cemu) is a full system emulator **framework** designed to easily change all aspects in a simple CPU and SoC for education and research purposes.

You can use CEMU for:

- Differential Test with CPU RTL with [SoC-simulator](https://github.com/cyyself/soc-simulator)
- Develop new ISA-level extension
- Define your own performance counter and tracer for software workload exploration
- Develop a simple CPU and bus performance model

## ISA Support

- RISC-V
    - rv64imac (rv64gc without FPU)
    - Three privilege levels of  Machine, Supervisor, and User
    - SV39 Virtual Memory with TLB emulation

- MIPS32
    - MIPS32 Release 1 without Branch-Likely
    - TLB based MMU Support, 4KB Page Only

- LoongArch32 (Reduced)
    - Support LoongArch32(Reduced) instruction set, except FP instructions

## Device Support

- UART8250
- Xilinx UARTLite
- Xilinx AXI EthernetLite (Without ping-pong buffer and MDIO)
- RISC-V CLINT
- RISC-V PLIC

## Some talks about CEMU

- [重庆大学陈泱宇：SoC-Simulator——一个简单易用的软件定义AXI Slave设备框架](https://www.bilibili.com/video/BV1bd4y1R72j/?t=299)