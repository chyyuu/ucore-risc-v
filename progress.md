# progress on porting ucore on risc-v

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
