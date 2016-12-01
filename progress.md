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

### test kernel
```
$ spike +disk=path/to/root.img bbl vmlinux
```
