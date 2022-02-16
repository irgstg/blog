## Intro
Sep 18, ARM announce a new feature which they called Memory Tagging Extension (MTE).  
As time went by, code-support for this feature began to appear, and I found myself more and more intruiged by this feature.  
After (not so much) reading, I wanted to raise an exception caused by MTE by myself!

My end goal was simple as that: run a simple C-written executable, which raises a fault.
That's it.

I didn't find any practical information or tutorials regarding the how-to (tried it a year ago!).  
The how-to's are with me since then, now decided to write a little about that, might be useful for someone out there.

This paper does not include any theoretical information regarding how MTE works. Instead, I assume you know the theoretical thesis behind MTE.
Anyways, I briefly covered this subject in [What the fuss is all about?](#what-the-fuss-is-all-about), and included some very useful references.

## Sections:
- [What all the fuss is about?](#what-the-fuss-is-all-about)
- [How can I raise MTE fault by myself?](#how-can-i-raise-mte-fault-by-myself)
- [Compiling QEMU (in case needed)](#compile-qemu)
- [Compiling the Kernel](#compile-the-kernel)
- [Creating appropriate FS](#generate-fs-using-buildroot)
- [Compile our Demo Code](#compile-our-demo-code)
- [SEGFAULT?](#expecting-our-segmentation-fault)
- [About the next part](#whats-next)

## What the fuss is all about?
Although explaining MTE isn’t what this article is about (I might write another part for that dig into that in much more details), I do cover it in a nutshell.

MTE is a feature (ISA extension to be more precise), on arm64, that aims to improve security on a device by taking advantage of an already existing feature named Top Byte Ignore, or TBI. 
MTE takes advantage of TBI by randomizing, setting and validating a tag when referenced with a pointer (alter the ignored byte) to a certain memory.  
In case of a tag mismatch - meaning, that a pointer refers to a location in memory that tagged with a different tag, a segmentation fault will occur.

MTE (Memory Tagging Extension) basically comes to make it harder to exploit many memory related vulnerabilities, on ARM 64.

As I mentioned above, explaining the details of MTE isn’t a huge part of this article. For further reading about both TBI [[1](https://source.android.com/devices/tech/debug/tagged-pointers)] and MTE [[2](https://www.usenix.org/system/files/login/articles/login_summer19_03_serebryany.pdf)] [[3](https://community.arm.com/developer/ip-products/processors/b/processors-ip-blog/posts/enhancing-memory-safety)] [[4](https://security.googleblog.com/2019/08/adopting-arm-memory-tagging-extension.html)].

## How can I raise MTE fault by myself?
Obviously, we need to run an appropriate environment. We will compile and run a simple C code, with MTE instrumentation. We will also compile a supported kernel and run it over QEMU.
From the program perspective, when run MTE "*manualy*" (without supporting compiler instrumentation and `glibc` support), we need:
1. The program should check whether it runs on a MTE supported system.
When run with supported kernel, we can receive `HWCAP2_MTE` via `getauxval(AT_HWCAP2)`.
2. Enabling tagged addresses ABI by the current process, using `prctl` (screenshot from `man prctl`)
![from `man prctl`](https://user-images.githubusercontent.com/86537665/153384422-682f0864-0aa1-4a38-9560-9cb94b113667.png)
3. Page support for MTE, enabled via `mmap` / `mprotect` and `PROT_MTE`.
4. Using MTE instrumentation to set a random tag both on a pointer, and in it's respective memory.

Let's start...
## Compile QEMU
We need QEMU that is at least version 5.1, as we can see in it's [Changelog](https://wiki.qemu.org/ChangeLog/5.1#Arm).  
*Many [optional packages](https://wiki.qemu.org/Hosts/Linux#Recommended_additional_packages) are included as described in QEMU's documentation, didn't try to install without them.*  
How-to (see `$QEMU`):
```
apt -y update && apt -y install make gcc g++ git-email libaio-dev libbluetooth-dev libbrlapi-dev libbz2-dev libcap-dev libcap-ng-dev libcurl4-gnutls-dev libgtk-3-dev \
libibverbs-dev libjpeg8-dev libncurses5-dev libnuma-dev librbd-dev librdmacm-dev libsasl2-dev libsdl1.2-dev libseccomp-dev libsnappy-dev libssh2-1-dev libvde-dev \
libvdeplug-dev libxen-dev liblzo2-dev valgrind xfslibs-dev libnfs-dev libiscsi-dev ninja-build libglib2.0-dev  libpixman-1-dev && \
mkdir -p $QEMU && \
git clone https://github.com/qemu/qemu.git --depth=1 $QEMU && cd $QEMU && \
git submodule init && \
git submodule update --recursive && \
git submodule status --recursive && \
mkdir build && \
cd build && \
../configure && \
make -j$(nproc)
```
## Compile the kernel
As said, we need supported kernel. The kernel supports user-mode MTE via `CONFIG_ARM64_MTE`.  
Cross-compiling with the right environment variables, `CONFIG_ARM64_MTE` will be enabled with `defconfig` by default.  
Here, I used the latest stable release (at the time of writing these lines, `5.16.8`).  

How-to (see `$KERNEL`):
```
apt -y update && \
apt -y install build-essential libncurses-dev bison flex libssl-dev libelf-dev wget bc binutils-aarch64-linux-gnu gcc-aarch64-linux-gnu clang-11 && \
mkdir -p $KERNEL && cd $KERNEL && \
wget -O- https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.16.8.tar.xz | tar -Jxf- && \
cd linux-5.16.8 && \
CC=clang ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig && \
CC=clang ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc) Image
```

## Generate FS using buildroot
I wrote a small patch to [`buildroot`](https://buildroot.org/), which configures `glibc` with `--enable-memory-tagging`. This flag make sure
glibc will compile supporting MTE.

How-to (see `$BUILDROOT`):
1. Create a new directory, clone `buildroot` and apply the patch:
```
apt -y update && \
apt -y install cpio unzip rsync && \
mkdir -p $BUILDROOT && \
git clone --depth 1 --branch 2021.11.1 https://github.com/buildroot/buildroot.git $BUILDROOT && \
cd $BUILDROOT && \
wget -O- https://patchwork.ozlabs.org/project/buildroot/patch/20211030083753.192-1-irgstg@gmail.com/raw/ | patch -p1 -i -
```
2. `make menuconfig`:  
*Target Options* -> *Target Architecture* -> *AArch64 (little endian)*  
*Toolchain* -> *C library* -> select *glibc*  
*Toolchain* -> select *Install glibc utilities*  
*Toolchain* -> select *Install glibc support for MTE*  
*Filesystem images* -> select *ext2/3/4 root filesystem*  
*Filesystem images* -> *ext2/3/4 variant* -> select *ext4*  
Recommended:  
*Filesystem images* -> *Exact size* -> write `*512M*`  
Save and exit
3. `make -j$(nproc)`

## Compile our Demo Code
So we sort of got everything, except our demo code.  
Let's see this code:
```
#include <stdio.h>
#include <stdlib.h>

int main()
{
	int *i = NULL;
	printf(" [+] Allocating...\n");
	i = (int *) malloc (sizeof(int) * 1);
	printf(" [#] Tag mismatch...\n");
	i[20] = 0x00;
	printf(" [!] NO FAULT OCCURED...\n");
	return 0;
}
```
As you can see, this code is very easy to understand.  
Basically, we're allocating memory with malloc, and trying to write to memory out of our tag scope (tag alignments are by 16, didn't covered in this paper).
If we'll see `[!] NO FAULT OCCURED...`, well, you know.
You obviously can play with this code or try to write new one by yourself.  
So let's:
1. Save this code in `mte-demo.c`
2. Generate the executable, and FS with the compiled executable:
```
apt install -y g++-multilib && \
clang -target aarch64-linux-gnu -march=armv8.5a+memtag -fsanitize=memtag mte-demo.c -o $BUILDROOT/output/target/mte-demo && \
cd $BUILDROOT && \
make
```
*`-fsanitize=memtag` isn't a must here, because of our `glibc` support and heap-allocations*  

## Expecting our Segmentation Fault
Phew, that's has been a long process.
Let's run our FS with the kernel (username is `root`):  
```
qemu-system-aarch64 -machine virt,mte=on -cpu max -kernel $KERNEL/linux-5.16.8/arch/arm64/boot/Image \
-hda $BUILDROOT/output/images/rootfs.ext4 -smp 2 -m 2G -display none -serial stdio -append "root=/dev/vda"
```
*If you installed QEMU manually, call `$QEMU/build/qmeu-system-aarch64`, or add the binary to path*  

Let's try to run our lovely executable:
`/mte-demo`

...
Let me guess... no SEGFAULT, right?  
I've tuckled this for a while.  
glibc's MTE support isn't enabled by default for every executable.  
In order for `glibc` to run an executable with MTE support, [a tunable (`glibc.mem.tagging`) must to be set](https://sourceware.org/git/?p=glibc.git;a=blob;f=INSTALL;h=02dcf6b1ca3a4c43a17fdcae5e7dae8189c1c50b;hb=ae37d06c7d127817ba43850f0f898b793d42aea7#l150).  
These are the options to set `glibc.mem.tagging`:
1. 0 - no support for MTE whatsoever
2. 1 - MTE async fault is supported
3. 2 - MTE sync fault is supported

Let's try our code with async fault support:
`GLIBC_TUNABLES=glibc.mem.tagging=1 /mte-demo`

![image](https://user-images.githubusercontent.com/86537665/153727457-3e6c8eac-d3b6-426b-b4ba-3aa9edff3fb6.png)

Yea.

## Whats Next?
So MTE is a (relatively) new feature, and is not supported by any board out in the market yet.  
There are many contributions to increase support for this feature, and more to come.
There are many things that I didn't cover in here, such as:
1. cover more features in details
2. how the kernel supports in-kernel MTE
3. share in details some projects I did with MTE
  
and so on...  
This paper might be the beggining of a series MTE-related papers I'll publish, I follow this subject closely for a year now, and I find it fascinating.  

I'm far from being perfect, and this paper might not be perfect also.
So if you read this paper and enjoyed, have some insights or remarks, please, e-mail me on irgstg@gmail.com.
