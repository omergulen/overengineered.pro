---
title:  How to add system call (syscall) to the kernel, compile and test it?
date: '2019-12-25'
spoiler: Not that hard.
cta: 'syscall'
---

_One of my example `syscall` methods will print `Hello world` to the kernel buffer._

The second one gets PID of a process and prints the `elapsed time` of it to the kernel buffer in my other blog post:

[How to add system call (syscall) that prints elapsed time of a process with given PID to the kernel and test it?](/how-to-add-system-call-syscall-that-prints-elapsed-time-of-a-process-with-given-pid-to-the-kernel-and-test-it/)


## 1. Setup environment

I prefer to get `root` and apply the steps as `root` to prevent getting permission failures on the way. If you don't want to apply the steps as a root, you can just use `sudo` for root required commands.

```bash
su -
```

will switch to `root` user after entering the password of the `root`. Then, first you can check the active kernel version of your OS with:

```bash
uname -r
```

This prints out `4.19.0-6-amd64` in my case. Then, we need to get the source of a kernel. I will use slightly newer version (4.20.1) from my version with following `wget` command.

```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.20.1.tar.xz
```

This will download the source of `4.20.1`. It may take a while, like 2-3 minutes depending on your internet speed.

Extract the compressed kernel code with

```bash
tar -xvf linux-4.20.1.tar.xz
```

It will create a folder named `linux-4.20.1.tar.xz` and extract the compressed code into that folder.

Now we will change our directory to new kernel code.

```bash
cd linux-4.20.1
```

## 2. Add "Hello world" syscall to kernel

I prefer creating new folder for my own stuff while adding a new syscall.

```bash
mkdir hello && cd hello
```

After that, I will create a C file for my syscall implementation. I prefer `vim hello.c` to create and edit the file and insert following C code.

```c
#include <linux/kernel.h>

asmlinkage long sys_hello(void) 
{
        //printk prints to the kernel’s log file.
        printk("Hello world\n");
        return 0;
}
```

We need to create a `Makefile` in the `hello` directory.

```bash
vim Makefile
```

and then insert this:

```obj-y := hello.o```

Then, go to the parent directory (kernel source main directory):

```bash
cd ..
```

We need to add our new syscall directory to `Makefile`, in this way it will compile our syscall, too. To achieve this, search for `core-y` in the `Makefile` then, find the

_In vim you can do search with `/core-y` after pressing ESC._

```core-y += kernel/ mm/ fs/ ipc/ security/ crypto/ block/``` line and add `hello/` to the end of this line.

As a result, the line should be looking like this:

```core-y += kernel/ mm/ fs/ ipc/ security/ crypto/ block/ hello/```

Next step is adding the new system call to the system call table.
If you are on a 32-bit system you’ll need to change `syscall_32.tbl` file.

```bash
vim arch/x86/entry/syscalls/syscall_32.tbl
```

For 64-bit, change `syscall_64.tbl`.

```bash
vim arch/x86/entry/syscalls/syscall_64.tbl
```

_I will continue with my 64-bit OS and my steps will be accordingly._

We need to keep table's structure while adding our syscall. So that we will add our line to the end of the line. My last syscall has number of `547`, so that I will use `548`, you also should use `N+1`.

My syscall data:

```548       64        hello          sys_hello```

Example file:

```
546     ...     ...     ...
547     ...     ...     ...
548     64      hello   sys_hello
```

Now, we need to add our syscall method signature to `syscalls` header file which is `syscalls.h`.

```bash
vim include/linux/syscalls.h
```

Then, add the following line to the end of the document before the #endif statement:

```c
asmlinkage long sys_hello(void);
```

## 3. Compiling our kernel

Before starting to compile you need to install a few packages. Type the following commands in your terminal to install required packages:

```bash
apt-get install gcc &&
apt-get install libncurses5-dev &&
apt-get install bison &&
apt-get install flex &&
apt-get install libssl-dev &&
apt-get install libelf-dev &&
apt-get update &&
apt-get upgrade &&
apt-get make
```

Now, you can configure your kernel by using the config menu by executing:

```bash
make menuconfig
```

**IMPORTANT NOTE**: While entering this command, you might want to maximize your terminal screen. Otherwise, you might get and error and a pop-up screen will not appear.

Once the above command is used to configure the Linux kernel, you will get a pop up window with the list of 
menus and you can select the items for the new configuration. If your unfamiliar with the configuration just check for the file systems menu and check whether “ext4” is chosen or not, if not select it and save the configuration.

**OR:** You can simply execute the following command and use the default configuration.

```bash
make defconfig
```

Now, finally we can compile the kernel with the following command:

```bash
sudo make
```

The compilation took 40 mins to 1 hour on my uni-core VM. As an alternative solution you can increase the core count of your VM in the VM settings and use them to compilation with the `-jn` parameter. `n` is the number of cores dedicated to your VM.

In my case, I've used the following ocmmand for 8-core VM which reduced the compilation time to 20 seconds to 1 minute:

```bash
make -j8
```

## 4. Installing our kernel

After the successful compilation, to install/update kernel use the following command:

```bash
make modules_install install
```

The command will create some files under `/boot/` directory and it will `automatically` make a entry in your grub.cfg. 
To check whether it made correct entry, check the files under `/boot/` directory . If you have followed the steps without any error you will find the following files in it in addition to others.

## 5. Testing our syscall in new kernel

Now all we need to do is restart the system:

```bash
shutdown -r now
```

After computer restarts, in `grub`'s advanced options, you can see there are 2 options (without counting the recovery mode options).

Once your computer is up again, you can run the following command the check your kernel version:

```bash
uname -r
```

which prints `4.20.1` in my OS after installing the kernel.

After checking the version of the kernel, we will test our `syscall` with tiny C program.

```bash
vim hello_test.c
```

Insert the following C code into the `hello_test.c` file:

```c
#include <stdio.h>
#include <linux/kernel.h>
#include <sys/syscall.h>
#include <unistd.h>

int main()
{
        long int helloCheck = syscall(548);
        printf("System call sys_hello returned %ld\n", helloCheck);
        return 0;
}
```

Then compile and run it:

```bash
gcc hello_test.c -o hello.o && ./hello.o
```

it prints `System call sys_hello returned 0` if there is no problem on the execution and now we can check the kernel log buffer to see if our `Hello world` is there with the command below:

```bash
dmesg
```

It will print tons of line and in the end we should be seeing our `Hello world`.

```bash
[    ........] ...
[    ........] ...
[    ........] ...
[    2.858022] IPv6: ADDRCONF(NETDEV_UP): enp0s3: link is not ready
[    2.858144] ip (1443) used greatest stack depth: 12424 bytes left
[    2.860434] Adding 10483708k swap on /dev/sda5.  Priority:-2 extents:1 across:10483708k 
[    3.391290] hrtimer: interrupt took 4499049 ns
[    4.887969] e1000: enp0s3 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
[    4.888216] IPv6: ADDRCONF(NETDEV_CHANGE): enp0s3: link becomes ready
[  306.497742] Hello world
```

We can observe that our `Hello world` is printed to kernel log buffer. So it worked!

## Error - Fix

#### Debian Certificate Problem

On my first trial I have faced with an error such below:

```bash
make[1]: *** No rule to make target 'debian/certs/debian-uefi-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
make: *** [Makefile:1055: certs] Error 2
```


#### - SOLUTION -

I've found the solution of this problem in the [debian.org's bug report section](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=897163). The issue was reported by *Heinrich Schuchardt* as the following:

> The Debian kernels are currently built with CONFIG_SYSTEM_TRUSTED_KEYS="debian/certs benh@debian.org.cert.pem"
> This was introduced with this https://alioth-lists-archive.debian.net/pipermail/kernel-svn-changes/2016-April/022904.html.
> 
> We are now two years after Ben's contribution and we are still not using kernel module signing (CONFIG_MODULE_SIG is not set in config).
> 
> As there is no need for the kernel trusting Ben's certificate, please, remove the setting.
> 
> Best regards
> 
> Heinrich Schuchardt

As a result, I've cleared the `CONFIG_SYSTEM_TRUSTED_KEYS` config in the configuration file.

In `linux-4.20.1` (which is my kernel source directory), there is a file named `.config` which stores the configuration of the kernel. With the following command I opened it:

```bash
vim .config
```

Then searched for `CONFIG_SYSTEM_TRUSTED_KEYS` and changed

```
CONFIG_SYSTEM_TRUSTED_KEYS="debian/certs benh@debian.org.cert.pem"
```

into this:

```
CONFIG_SYSTEM_TRUSTED_KEYS=""
```

then, saved and closed the file, and tried to compile again and *there was no issue.*

