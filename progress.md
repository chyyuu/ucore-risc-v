# progress on porting ucore on risc-v

## 20161203
- prepare [related info](https://github.com/chyyuu/ucore-risc-v/blob/master/related-info.md)
- read spec

### user-level spec
#### brief intro
An integer base plus these four standard extensions (“IMAFD”) is given the abbreviation “G” and provides a
general-purpose scalar instruction set. RV32G and RV64G are currently the default target of our
compiler toolchains. 

- registers: 32 general-purpose x0~x31. x0=0. pc is program counter
- instruction: In the base ISA, there are four core instruction formats (R/I/S/U)

The RISC-V ISA keeps the source (rs1 and rs2) and destination (rd) registers at the same position
in all formats to simplify decoding.

#### Memory Model
The base RISC-V ISA supports multiple concurrent threads of execution within a single user address
space. Each RISC-V thread has its own user register state and program counter, and executes an
independent sequential instruction stream. 

RISC-V has a relaxed memory model between threads,
requiring an explicit FENCE instruction to guarantee any specific ordering between memory oper-
ations from different RISC-V threads. 

#### Control and Status Register Instructions
SYSTEM instructions are used to access system functionality that might require privileged access
and are encoded using the I-type instruction format. These can be divided into two main classes:
those that atomically read-modify-write control and status registers (CSRs), and all other poten-
tially privileged instructions. 

#### Environment Call and Breakpoints
The ECALL instruction (like sysenter in x86) is used to make a request to the supporting execution environment, which is
usually an operating system. 

The EBREAK instruction is used by debuggers to cause control to be transferred back to a debug-
ging environment.

####  standard RISC-V pseudoinstructions（assembly lang）

| Pseudoinstruction            | Base Instruction(s)                      | meaning                         |
| ---------------------------- | ---------------------------------------- | ------------------------------- |
| la rd, symbol                | auipc rd, symbol[31:12]  ;  addi rd, rd, symbol[11:0] | Load address                    |
| l{b\|h\|w\|d} rd, symbol     | auipc rd, symbol[31:12] ;  l{b\|h\|w\|d} rd, symbol[11:0] (rd) | Load global                     |
| s{b\|h\|w\|d} rd, symbol, rt | auipc rt, symbol[31:12] ; s{b\|h\|w\|d} rd, symbol[11:0] (rt) | Store global                    |
| fl{w\|d} rd, symbol, rt      | auipc rt, symbol[31:12] ;  fl{w\|d} rd, symbol[11:0] (rt) | Floating-point load global      |
| fs{w\|d} rd, symbol, rt      | auipc rt, symbol[31:12] ;  fs{w\|d} rd, symbol[11:0] (rt) | Floating-point store global     |
| nop                          | addi x0, x0, 0                           | No operation                    |
| li rd, immediate             | Myriad sequences                         | Load immediate                  |
| mv rd, rs                    | addi rd, rs, 0                           | Copy register                   |
| not rd, rs                   | xori rd, rs, -1                          | One’s complement                |
| neg rd, rs                   | sub rd, x0, rs                           | Two’s complement                |
| negw rd, rs                  | subw rd, x0, rs                          | Two’s complement word           |
| sext.w rd, rs                | addiw rd, rs, x0                         | Sign extend word                |
| seqz rd, rs                  | sltiu rd, rs, 1                          | Set if = zero                   |
| snez rd, rs                  | sltu rd, x0, rs                          | Set if <> zero                  |
| sltz rd, rs                  | slt rd, rs, x0                           | Set if < zero                   |
| sgtz rd, rs                  | slt rd, x0, rs                           | Set if > zero                   |
| fmv.s rd, rs                 | fsgnj.s rd, rs, rs                       | Copy single-precision register  |
| fabs.s rd, rs                | fsgnjx.s rd, rs, rs                      | Single-precision absolute value |
| fneg.s rd, rs                | fsgnjn.s rd, rs, rs                      | Single-precision negate         |
| fmv.d rd, rs                 | fsgnj.d rd, rs, rs                       | Copy double-precision register  |
| fabs.d rd, rs                | fsgnjx.d rd, rs, rs                      | Double-precision absolute value |
| fneg.d rd, rs                | fsgnjn.d rd, rs, rs                      | Double-precision negate         |
| beqz rs, offset              | beq rs, x0, offset                       | Branch if <> zero               |
| bnez rs, offset              | bne rs, x0, offset                       | Branch if = zero                |
| blez rs, offset              | bge x0, rs, offset                       | Branch if ≤ zero                |
| bgez rs, offset              | bge rs, x0, offset                       | Branch if ≥ zero                |
| bltz rs, offset              | blt rs, x0, offset                       | blt rs, x0, offset              |
| bgtz rs, offset              | blt x0, rs, offset                       | Branch if > zero                |
| j offset                     | jal x0, offset                           | Jump                            |
| jal offset                   | jal x1, offset                           | Jump and link                   |
| jr rs                        | jalr x0, rs, 0                           | Jump register                   |
| jalr rs                      | jalr x1, rs, 0                           | Jump and link register          |
| ret                          | jalr x0, x1, 0                           | Return from subroutine          |
| call offset                  | auipc x6, offset[31:12] ;  jalr x1, x6, offset[11:0] | Call far-away subroutine        |
| tail offset                  | auipc x6, offset[31:12] ; jalr x0, x6, offset[11:0] | Tail call far-away subroutine   |

#### 

#### C Datatypes and Alignment

| C type      | Description              | Bytes in RV32 | Bytes in RV64 |
| ----------- | ------------------------ | ------------- | ------------- |
| char        | Character value/byte     | 1             | 1             |
| short       | short integer            | 2             | 2             |
| int         | integer                  | 4             | 4             |
| long        | long integer             | 4             | 8             |
| long long   | long long integer        | 8             | 8             |
| void *      | pointer                  | 4             | 8             |
| float       | single-precision float   | 4             | 4             |
| double      | double-precision float   | 8             | 8             |
| long double | extended-precision float | 16            | 16            |

#### Call Convention

The RISC-V calling convention passes arguments in registers when possible. Up to eight integer registers, a0–a7, and up to eight floating-point registers, fa0–fa7, are used for this purpose.

If the arguments to a function are conceptualized as fields of a C struct, each with pointer alignment, the argument registers are a shadow of the first eight pointer-words of that struct. If argument i < 8 is a floating-point type, it is passed in floating-point register fai; otherwise, it is passed in integer register ai. However, floating-point arguments that are part of unions or array fields of structures are passed in integer registers. Additionally, floating-point arguments to variadic functions (except those that are explicitly named in the parameter list) are passed in integer
registers.


Arguments smaller than a pointer-word are passed in the least-significant bits of argument registers. Correspondingly, sub-pointer-word arguments passed on the stack appear in the lower addresses of a pointer-word, since RISC-V has a little-endian memory system. When primitive arguments twice the size of a pointer-word are passed on the stack, they are naturally aligned. When they are passed in the integer registers, they reside in an aligned even-odd register pair, with the even register holding the least-significant bits. In RV32, for example, the function void foo(int, long long) is passed its first argument in a0 and its second in a2 and
a3. Nothing is passed in a1.

Arguments more than twice the size of a pointer-word are passed by reference.

The portion of the conceptual struct that is not passed in argument registers is passed on the stack. The stack pointer sp points to the first argument not passed in a register.

Values are returned from functions in integer registers a0 and a1 and floating-point registers fa0 and fa1. Floating-point values are returned in floating-point registers only if they are primitives or members of a struct consisting of only one or two floating-point values. Other return values that fit into two pointer-words are returned in a0 and a1. Larger return values are passed entirely in memory; the caller allocates this memory region and passes a pointer to it as an implicit first
parameter to the callee.

In the standard RISC-V calling convention, the stack grows downward and the stack pointer is always kept 16-byte aligned.

In addition to the argument and return value registers, seven integer registers t0–t6 and twelve floating-point registers ft0–ft11 are temporary registers that are volatile across calls and must be saved by the caller if later used. Twelve integer registers s0–s11 and twelve floating-point registers fs0–fs11 are preserved across calls and must be saved by the callee if used. 



| Register | ABI name | Description                      | Saver  |
| -------- | -------- | -------------------------------- | ------ |
| x0       | zero     | Hard-wired zero                  | --     |
| x1       | ra       | return address                   | caller |
| x2       | sp       | stack pointer                    | callee |
| x3       | gp       | global pointer                   | --     |
| x4       | tp       | thread pointer                   | --     |
| x5-7     | t0-2     | temporaries                      | caller |
| x8       | s0/fp    | saved register/frame pointer     | callee |
| x9       | s1       | saved register                   | callee |
| x10-11   | a0-1     | function arguments/return values | caller |
| x12-17   | a2-7     | function arguments               | caller |
| x18-27   | s2-11    | saved register                   | callee |
| x28-31   | t3-6     | temporaries                      | caller |
| f0-7     | ft0-7    | FP temporaries                   | caller |
| f8-9     | fs0-1    | FP saved registers               | callee |
| f10-11   | fa0-1    | FP arguments/return values       | caller |
| f12-17   | fa2-7    | FP arguments                     | caller |
| f18-27   | fs2-11   | FP saved registers               | callee |
| f28-31   | ft8-11   | FP temporaries                   | callee |



### privileged spec

#### brief intro

The machine level has the highest privileges and is the only mandatory privilege level for a RISC-V hardware platform. Code run in machine-mode (M-mode) is inherently trusted, as it has low-level access to the machine implementation. M-mode is used to manage secure execution environments on RISC-V. User-mode (U-mode) and supervisor-mode (S-mode) are intended for conventional application and operating system usage respectively, while hypervisor-mode (H-mode) is intended to support virtual machine monitors.

####　Control and Status Registers (CSRs)

The SYSTEM major opcode is used to encode all privileged instructions in the RISC-V ISA. These　can be divided into two main classes: those that atomically read-modify-write control and status　registers (CSRs), and all other privileged instructions. 

The standard RISC-V ISA sets aside a 12-bit encoding space (csr[11:0]) for up to 4,096 CSRs.　By convention, the upper 4 bits of the CSR address (csr[11:8]) are used to encode the read and　write accessibility of the CSRs according to privilege level.   The top two bits (csr[11:10]) indicate whether the register is read/write (00, 01, or 10) or read-only. The next two bits (csr[9:8]) encode the lowest privilege level that can access the CSR.

#### supervisor-level(OS) CSR addr

- supervisor trap setup

| 0x100 | srw  | sstatus | sv                               |
| ----- | ---- | ------- | -------------------------------- |
| 0x102 | srw  | sedeleg | sv status register               |
| 0x103 | srw  | sideleg | sv exception delegation register |
| 0x104 | srw  | sie     | sv interrupt delegation register |
| 0x105 | srw  | stvec   | sv trap handler base address     |

- supervisor trap handling

  ​


| 0x140 | srw  | sscratch | scratch register for sv trap handlers |
| ----- | ---- | -------- | ------------------------------------- |
| 0x141 | srw  | sepc     | sv exception program counter          |
| 0x142 | srw  | scause   | sv trap cause                         |
| 0x143 | srw  | sbadaddr | sv bad address                        |
| 0x144 | srw  | sip      | sv interrupt pending                  |

- supervisor protection and translation

| 0x180 | srw  | sptbr | page-table base register |
| ----- | ---- | ----- | ------------------------ |
|       |      |       |                          |


- supervisor counter/timers

  | 0xd00 | sro  | scycle    | sv cycle counter                      |
  | ----- | ---- | --------- | ------------------------------------- |
  | 0xd01 | sro  | stime     | sv wall-clock time                    |
  | 0xd02 | sro  | sinstret  | sv instr-retired counter              |
  | 0xd80 | sro  | scycleh   | upper 32-bits of scycle, RV32I only   |
  | 0xd81 | sro  | stimeh    | upper 32-bits of stime, RV32I only    |
  | 0xd82 | sro  | sinstreth | upper 32-bits of sinstret, RV32I only |



- Machine Information Registers

| oxf10 | mro  | misa      | isa and extensions supported |
| ----- | ---- | --------- | ---------------------------- |
| 0xf11 | mro  | mvendorid | vendor id                    |
| 0xf12 | mro  | marchid   | architecture id              |
| 0xf13 | mro  | mimpid    | implementation id            |
| 0xf14 | mro  | mhartid   | hardware thread id           |

#### RISC-V Privileged Instructions

- Trap-Return Instructions: URET, SRET, HRET, MRET
- Interrupt-Management Instructions: WFI
- Memory-Management Instructions: SFENCE.VM

####   Sv32: Page-Based 32-bit Virtual-Memory Systems

When Sv32 is written to the VM field in the mstatus register, the supervisor operates in a 32-bit paged virtual-memory system. Sv32 is supported on RV32 systems and is designed to include mechanisms sufficient for supporting modern Unix-based operating systems.

Sv32 implementations support a 32-bit virtual address space, divided into 4 KiB pages. An Sv32 virtual address is partitioned into a virtual page number (VPN) and page offset, When Sv32 virtual memory mode is selected in the VM field of the mstatus register, supervisor virtual addresses are translated into supervisor physical addresses via a two-level page table. The 20-bit VPN is translated into a 22-bit physical page number (PPN), while the 12-bit page offset is untranslated. The resulting supervisor-level physical addresses are then checked using any physical memory protection structures, before being directly converted to
machine-level physical addresses.

Sv32 page tables consist of 2^10 page-table entries (PTEs), each of four bytes. A page table is exactly the size of a page and must always be aligned to a page boundary. The physical page number of the root page table is stored in the sptbr register.

Sv39 implementations support a 39-bit virtual address space, divided into 4 KiB pages. An Sv39 address is partitioned as shown below. Load and store effective addresses, which are 64 bits, must have bits 63–39 all equal to bit 38, or else an access fault will occur. The 27-bit VPN is translated into a 38-bit PPN via a three-level page table, while the 12-bit page offset is untranslated. Sv39 page tables contain 2^9 page table entries (PTEs), eight bytes each. A page table is exactly
the size of a page and must always be aligned to a page boundary. The physical address of the root page table is stored in the sptbr register. Any level of PTE may be a leaf PTE, so in addition to 4 KiB pages, Sv39 supports 2 MiB megapages
and 1 GiB gigapages, each of which must be virtually and physically aligned to a boundary equal to its size. The algorithm for virtual-to-physical address translation is the same as sv32, except LEVELS equals 3 and PTESIZE equals 8.

Sv48 implementations support a 48-bit virtual address space, divided into 4 KiB pages. An Sv48 address is partitioned as shown below. Load and store effective addresses, which are 64 bits, must have bits 63–48 all equal to bit 47, or else an access fault will occur. The 36-bit VPN is translated into a 38-bit PPN via a four-level page table, while the 12-bit page offset is untranslated. The PTE format for Sv48 is shown below. Bits 9–0 have the same meaning as for Sv32. Any
level of PTE may be a leaf PTE, so in addition to 4 KiB pages, Sv48 supports 2 MiB megapages, 1 GiB gigapages, and 512 GiB terapages, each of which must be virtually and physically aligned to a boundary equal to its size.The algorithm for virtual-to-physical address translation is the same as sv32, except LEVELS equals 4 and PTESIZE equals 8.



- sv32 phy/virt addr: P/VPN[1]:12 bit, P/VPN[0]:10 bit, page offset:12 bit = 32 bit phy addr. support 4KB/4MB page
- sv32 page table entry:  PPN[1]:12 bit, PPN[0]10 bit, reserved for SW:2 bit, D/A/G/U/X/W/R/V: 1 bit, all 8bits.
- sv39 virt addr: VPN[2]:9 bits, VPN[1]:9 bits, VPN[0]:9 bits, page offset:12 its
- sv39 phy addr: PPN[2]:20 bits, PPN[1]:9 bits, PPN[0]:9 bits, page offset:12 its
- sv39 page table entry: Reserved:16 bits,  PPN[2]:20 bits, PPN[1]:9 bits, PPN[0]:9 bits, reserved for SW:2 bit, D/A/G/U/X/W/R/V: 1 bit, all 8bits.
- sv48 virt addr: VPN[3]:9 bits, VPN[2]:9 bits, VPN[1]:9 bits, VPN[0]:9 bits, page offset:12 its
- sv48 phy addr: PPN[3]:11 bits,PPN[2]:9 bits, PPN[1]:9 bits, PPN[0]:9 bits, page offset:12 its
- sv48 page table entry: Reserved:16 bits,   PPN[3]:11 bits,PPN[2]:9 bits, PPN[1]:9 bits, PPN[0]:9 bits, reserved for SW:2 bit, D/A/G/U/X/W/R/V: 1 bit, all 8bits.

The V bit indicates whether the PTE is valid; if it is 0, bits 31–1 of the PTE are don’t-cares and may be used freely by software. The permission bits, R, W, and X, indicate whether the page is readable, writable, and executable, respectively. When all three are zero, the PTE is a pointer to the next level of the page table; otherwise, it is a leaf PTE. Writable pages must also be marked readable; the contrary combinations are reserved for future use.The U bit indicates whether the page is accessible to user mode. U-mode software may only access the page when U=1. If the PUM bit in the sstatus register is clear, supervisor mode software may also access pages with U=1. However, supervisor code normally operates with the PUM bit set, in which case, supervisor code will fault on accesses to user-mode pages. 

The G bit designates a global mapping. Global mappings are those that exist in all address spaces. For non-leaf PTEs, the global setting implies that all mappings in the subsequent levels of the page table are global. Note that failing to mark a global mapping as global merely reduces performance, whereas marking a non-global mapping as global is an error.

Each leaf PTE maintains an accessed (A) and dirty (D) bit. When a virtual page is read, written, or fetched from, the implementation sets the A bit in the corresponding PTE. When a virtual page is written, the implementation additionally sets the D bit in the corresponding PTE. The PTE updates are exact and are observed in program order by the local hart. The ordering on loads and stores provided by FENCE instructions and the acquire/release bits on atomic instructions also orders the PTE updates associated with those loads and stores as observed by remote harts.

Any level of PTE may be a leaf PTE, so in addition to 4 KiB pages, Sv32 supports 4 MiB megapages. A megapage must be virtually and physically aligned to a 4 MiB boundary.

#### Virtual Address Translation Process

A virtual address va is translated into a physical address pa as follows:

1.  Let a be sptbr.ppn × PAGESIZE, and let i = LEVELS − 1. (For Sv32, PAGESIZE=2^12 and LEVELS=2.)
2. Let pte be the value of the PTE at address a+va.vpn[i]*PTESIZE. (For Sv32, PTESIZE=4.)
3.  If pte.v = 0, or if pte.r = 0 and pte.w = 1, stop and raise an access exception.
4. Otherwise, the PTE is valid. If pte.r = 1 or pte.x = 1, go to step 5. Otherwise, this PTE is a pointer to the next level of the page table. Let i = i − 1. If i < 0, stop and raise an access exception. Otherwise, let a = pte.ppn × PAGESIZE and go to step 2.
5.  A leaf PTE has been found. Determine if the requested memory access is allowed by the pte.r, pte.w, and pte.x bits. If not, stop and raise an access exception. Otherwise, the translation is successful. Set pte.a to 1, and, if the memory access is a store, set pte.d to 1. The translated physical address is given as follows:
   - pa.pgoff = va.pgoff.
   -  If i > 0, then this is a superpage translation and pa.ppn[i − 1 : 0] = va.vpn[i − 1 : 0].
   - pa.ppn[LEVELS − 1 : i] = pte.ppn[LEVELS − 1 : i].

#### Platform-Level Interrupt Controller (PLIC)

RISC-V harts can have both local and global interrupt sources. Only global interrupt sources are handled by the PLIC. The PLIC connects global interrupt sources, which are usually I/O devices, to interrupt targets, which are usually hart contexts. The PLIC contains multiple interrupt gateways, one per interrupt source, together with a PLIC core that performs interrupt prioritization and routing. Global interrupts are sent from their source to an interrupt gateway that processes the interrupt signal from each source and sends a single interrupt request to the PLIC core, which latches these in the core interrupt pending bits (IP). Each interrupt source is assigned a separate priority. The PLIC core contains a matrix of interrupt enable (IE) bits to select the interrupts that are enabled for each target. The PLIC core forwards an interrupt notification to one or more targets if the targets have any pending interrupts enabled, and the priority of the pending interrupts exceeds a per-target threshold. When the target takes the external interrupt, it sends an interrupt claim request to retrieve the identifier of the highest-priority global interrupt source pending for that target from the PLIC core, which then clears the corresponding interrupt source pending bit. After the target has serviced the interrupt, it sends
the associated interrupt gateway an interrupt completion message and the interrupt gateway can now forward another interrupt request for the same source to the PLIC. 

#### Local Interrupt Sources

Each hart has a number of local interrupt sources that do not pass through the PLIC, including the standard software interrupts and timer interrupts for each privilege level.

All local interrupts follow a level-based model, where an interrupt is pending if the corresponding bit in mip is set. The interrupt handler must clear the hardware condition that is causing the mip bit to be set to avoid retaking the interrupt after re-enabling interrupts on exit from the interrupt handler.

Additional non-standard local interrupt sources can be made visible to machine-mode by adding them to the high bits of the mip/mie registers, with corresponding additional cause values returned in the mcause register. These additional non-standard local interrupts may also be made visible to lower privilege levels, using the corresponding bits in the mideleg register.  The priority of non-standard local interrupt sources relative to external, timer, and software interrupts is platform-
specific.

#### Global Interrupt Sources

Global interrupt sources are those that are prioritized and distributed by the PLIC. Depending on the platform-specific PLIC implementation, any global interrupt source could be routed to any hart context. Global interrupt sources can take many forms, including level-triggered, edge-triggered, and message-signalled. Some sources might queue up a number of interrupt requests. All global interrupt sources are converted to a common interrupt request format for the PLIC.



#### Interrupt Targets and Hart Contexts

Interrupt targets are usually hart contexts, where a hart context is a given privilege mode on a given hart. A multithreaded processor core could run multiple
independent interrupt handlers on different hart contexts at the same time. A processor core could also provide hart contexts that are only used for interrupt handling to reduce interrupt service latency, and these might preempt interrupt handlers for other harts on the same core.

#### interrupt Identifiers/Priorities/Enables/Priority Thresholds/Notifications/Claims/Completion

Global interrupt sources are assigned small unsigned integer identifiers, beginning at the value 1. An interrupt ID of 0 is reserved to mean “no interrupt”. Each target has a vector of interrupt enable (IE) bits, one per interrupt source. Each interrupt target has an associated priority threshold, held in a platform-specific memory-mapped register. Only active interrupts that have a priority strictly greater than the threshold will cause a interrupt notification to be sent to the target. Each interrupt target has an external interrupt pending (EIP) bit in the PLIC core that indicates that the corresponding target has a pending interrupt waiting for service. Sometime after a target receives an interrupt notification, it might decide to service the interrupt. The target sends an interrupt claim message to the PLIC core, which will usually be implemented as a non-idempotent memory-mapped I/O control register read. After a handler has completed service of an interrupt, the associated gateway must be sent an interrupt completion message, usually as a write to a non-idempotent memory-mapped I/O control register. The gateway will only forward additional interrupts to the PLIC core after receiving the completion message.

## 20161201

### install tools

#### some reference
- https://riscv.org/software-tools/
- https://riscv.org/software-tools/linux/
- https://www.ocf.berkeley.edu/~qmn/linux/install-newlib.html
- https://www.ocf.berkeley.edu/~qmn/linux/install.html
- https://github.com/riscv/riscv-linux
- http://stackoverflow.com/questions/37849574/how-to-rebuild-bbl-with-payload-option

#### riscv64-unknown-elf-gcc...
```
$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc

$ git clone https://github.com/riscv/riscv-tools.git
$ git submodule update --init --recursive
$ export RISCV=/path/to/install/riscv/toolchain
$ ./build.sh
```

#### riscv64-unknown-linux-gnu-gcc

```
$cd riscv-gnu-toolchain
make linux
```

### test 
```
$ cd $TOP //set spke riscv-gcc path
$ echo -e '#include <stdio.h>\n int main(void) { printf("Hello world!\\n"); return 0; }' > hello.c
$ riscv-gcc -o hello hello.c
$ spike $RISCV/target/bin/pk hello
```
### build kernel
#### get kernel code
```
    $ curl -L https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.6.2.tar.xz | tar -xJ
    $ cd linux-4.6.2
    $ git init
    $ git remote add -t master origin https://github.com/riscv/riscv-linux.git
    $ git fetch
    $ git checkout -f -t origin/master
```

#### build kernel
```
$ make ARCH=riscv defconfig
make -j4 ARCH=riscv vmlinux
```
#### Exporting kernel headers //I didn't try
The riscv-gnu-toolchain repository includes a copy of the kernel header files. If the userspace API has changed, export the updated headers to the riscv-gnu-toolchain source directory:

```
$ make ARCH=riscv headers_check
$ make ARCH=riscv INSTALL_HDR_PATH=path/to/riscv-gnu-toolchain/linux-headers headers_install
```
Rebuild riscv64-unknown-linux-gnu-gcc with the linux target:

```
$ cd path/to/riscv-gnu-toolchain
$ make linux
```

#### build bbl with vmlinux
```
export RISCV=/path/to/your/already/built/toolchain
cd riscv-tools/riscv-pk
mkdir build.with.payload
cd build.with.payload
../configure --prefix=$RISCV/riscv64-unknown-elf \
--host=riscv64-unknown-elf \
--with-payload=/path/to/vmlinux

```
#### /etc/inittab
```
::sysinit:/bin/busybox mount -t proc proc /proc
::sysinit:/bin/busybox mount -t tmpfs tmpfs /tmp
::sysinit:/bin/busybox mount -o remount,rw /dev/htifbd0 /
::sysinit:/bin/busybox --install -s
/dev/ttyHTIF0::sysinit:-/bin/ash 
```
### build boot.img
```
$ cd $TOP/linux-3.4.53
$ dd if=/dev/zero of=root-gcc.img bs=1M count=256
$ mkfs.ext2 -F root-gcc.img
$ mkdir mnt-gcc
$ sudo mount -o loop root-gcc.img mnt-gcc
$ cd mnt-gcc
$ mkdir -p mkdir -p bin etc dev lib proc sbin tmp usr usr/bin usr/lib usr/sbin
$ curl http://www.ocf.berkeley.edu/~qmn/linux/linux-inittab > etc/inittab 
$ cp $TOP/busybox-1.21.1/busybox bin/
$ ln -s ../bin/busybox sbin/init
```
### test kernel
I build root.img ,but is useless now
```
$ spike +disk=path/to/root.img bbl vmlinux
```

### results
```
chyyuu@box-151:/chyhome/chyyuu/develop/riscv-related$ spike bbl
              vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
                  vvvvvvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrr       vvvvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrr      vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrrrr    vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrrrr    vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrrrr    vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrr      vvvvvvvvvvvvvvvvvvvvvv  
rrrrrrrrrrrrr       vvvvvvvvvvvvvvvvvvvvvv    
rr                vvvvvvvvvvvvvvvvvvvvvv      
rr            vvvvvvvvvvvvvvvvvvvvvvvv      rr
rrrr      vvvvvvvvvvvvvvvvvvvvvvvvvv      rrrr
rrrrrr      vvvvvvvvvvvvvvvvvvvvvv      rrrrrr
rrrrrrrr      vvvvvvvvvvvvvvvvvv      rrrrrrrr
rrrrrrrrrr      vvvvvvvvvvvvvv      rrrrrrrrrr
rrrrrrrrrrrr      vvvvvvvvvv      rrrrrrrrrrrr
rrrrrrrrrrrrrr      vvvvvv      rrrrrrrrrrrrrr
rrrrrrrrrrrrrrrr      vv      rrrrrrrrrrrrrrrr
rrrrrrrrrrrrrrrrrr          rrrrrrrrrrrrrrrrrr
rrrrrrrrrrrrrrrrrrrr      rrrrrrrrrrrrrrrrrrrr
rrrrrrrrrrrrrrrrrrrrrr  rrrrrrrrrrrrrrrrrrrrrr

       INSTRUCTION SETS WANT TO BE FREE
[    0.000000] Linux version 4.6.2-00033-g2e978be (chyyuu@box-151) (gcc version 6.1.0 (GCC) ) #1 Thu Dec 1 17:49:20 CST 2016
[    0.000000] Available physical memory: 2040MB
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000080400000-0x00000000ffbfffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080400000-0x00000000ffbfffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080400000-0x00000000ffbfffff]
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 515100
[    0.000000] Kernel command line: 
[    0.000000] PID hash table entries: 4096 (order: 3, 32768 bytes)
[    0.000000] Dentry cache hash table entries: 262144 (order: 9, 2097152 bytes)
[    0.000000] Inode-cache hash table entries: 131072 (order: 8, 1048576 bytes)
[    0.000000] Sorting __ex_table...
[    0.000000] Memory: 2054456K/2088960K available (1960K kernel code, 104K rwdata, 396K rodata, 64K init, 221K bss, 34504K reserved, 0K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS:0 nr_irqs:0 0
[    0.000000] clocksource: riscv_clocksource: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 191126044627 ns
[    0.000000] Calibrating delay loop (skipped), value calculated using timer frequency.. 20.00 BogoMIPS (lpj=100000)
[    0.000000] pid_max: default: 32768 minimum: 301
[    0.000000] Mount-cache hash table entries: 4096 (order: 3, 32768 bytes)
[    0.000000] Mountpoint-cache hash table entries: 4096 (order: 3, 32768 bytes)
[    0.000000] devtmpfs: initialized
[    0.000000] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.000000] NET: Registered protocol family 16
[    0.000000] clocksource: Switched to clocksource riscv_clocksource
[    0.000000] NET: Registered protocol family 2
[    0.000000] TCP established hash table entries: 16384 (order: 5, 131072 bytes)
[    0.000000] TCP bind hash table entries: 16384 (order: 5, 131072 bytes)
[    0.000000] TCP: Hash tables configured (established 16384 bind 16384)
[    0.000000] UDP hash table entries: 1024 (order: 3, 32768 bytes)
[    0.000000] UDP-Lite hash table entries: 1024 (order: 3, 32768 bytes)
[    0.000000] NET: Registered protocol family 1
[    0.020000] console [sbi_console0] enabled
[    0.020000] futex hash table entries: 256 (order: 0, 6144 bytes)
[    0.020000] workingset: timestamp_bits=61 max_order=19 bucket_order=0
[    0.030000] jitterentropy: Initialization failed with host not compliant with requirements: 2
[    0.030000] io scheduler noop registered
[    0.030000] io scheduler cfq registered (default)
[    0.040000] VFS: Cannot open root device "(null)" or unknown-block(0,0): error -6
[    0.040000] Please append a correct "root=" boot option; here are the available partitions:
[    0.040000] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    0.040000] CPU: 0 PID: 1 Comm: swapper Not tainted 4.6.2-00033-g2e978be #1
[    0.040000] Call Trace:
[    0.040000] [<ffffffff80011fac>] walk_stackframe+0x0/0xc8
[    0.040000] [<ffffffff8005441c>] panic+0xec/0x20c
[    0.040000] [<ffffffff800011d8>] mount_block_root+0x248/0x328
[    0.040000] [<ffffffff8000292c>] ksysfs_init+0x10/0x40
[    0.040000] [<ffffffff80001488>] prepare_namespace+0x148/0x198
[    0.040000] [<ffffffff80000da0>] kernel_init_freeable+0x1c8/0x200
[    0.040000] [<ffffffff801f62b0>] rest_init+0x80/0x84
[    0.040000] [<ffffffff801f62c4>] kernel_init+0x10/0x11c
[    0.040000] [<ffffffff801f62b0>] rest_init+0x80/0x84
[    0.040000] [<ffffffff80010bfc>] ret_from_syscall+0x10/0x14
[    0.040000] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```
