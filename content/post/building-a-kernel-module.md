---
title: "Building a Linux Kernel Module"
date: 2020-04-23T00:21:24+03:00
draft: false
---

Last night a friend of mine asked for help for her homework on operating
systems. It's about building a simple Linux kernel module and linked list
operations.  I hadn't worked on a kernel module before but somehow knew the
basics are simple to grasp. This is the transcript of my experience.

## The basics

Kernel modules have two entrypoints: `init` and `exit`. The `init` function runs
when you run `insmod <module>` and `exit` function runs when you run `rmmod
<module>`.

```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("egegunes");
MODULE_DESCRIPTION("Hello");

static int __init hello_init(void) {
    printk(KERN_INFO "Hello World");

    // If your init function returns a non-zero value,
    // kernel won't load your module.
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye, cruel world, I'm leaving you today");
}

module_init(hello_init);
module_exit(hello_exit);
```

That's all the code you need to create a kernel module. To build it, you also
need a `Makefile`:

```makefile
obj-m := hello.o
KDIR := /lib/modules/$(shell uname -r)/build

build:
    $(MAKE) -C $(KDIR) M=$(PWD)
clean:
    $(MAKE) -C $(KDIR) M=$(PWD) clean
```

This `Makefile` is for using `kbuild` build system used by the Linux kernel.
For more information see [kernel
documentation](https://github.com/torvalds/linux/blob/master/Documentation/kbuild/modules.rst).

I built and loaded the module in `fedora/31-cloud-base v31.20191023.0` Vagrant
box without installing any additional packages.

```
$ make
make -C /lib/modules/5.3.7-301.fc31.x86_64/build M=/home/vagrant/hello
make[1]: Entering directory '/usr/src/kernels/5.3.7-301.fc31.x86_64'
  Building modules, stage 2.
  MODPOST 1 modules
make[1]: Leaving directory '/usr/src/kernels/5.3.7-301.fc31.x86_64'
$ sudo insmod ./hello.ko
$ sudo dmesg
...
[  519.920518] Hello World
$ sudo rmmod ./hello.ko
...
[  519.920518] Hello World
[  523.976396] Goodbye, cruel world, I'm leaving you today
```

## Oops

Actual kernel module we've worked on was not that simple though. It was
processing items of a linked list on `init` and `exit`. When I tried to remove
the module, the kernel replied with a cold message:

```shell
$ sudo rmmod module
Killed
```

Seeing `Killed` automatically triggers the "out of memory?" question for me but
no, this was a different case. I looked at kernel logs and encountered
first-ever kernel panic that directly caused from my code:

```shell
$ sudo dmesg
[  100.472826] BUG: kernel NULL pointer dereference, address: 0000000000000000
[  100.472992] #PF: supervisor write access in kernel mode
[  100.473118] #PF: error_code(0x0002) - not-present page
[  100.473241] PGD 0 P4D 0
[  100.473306] Oops: 0002 [#1] SMP NOPTI
[  100.473397] CPU: 0 PID: 2394 Comm: rmmod Tainted: G           OE     5.3.7-301.fc31.x86_64 #1
[  100.473623] Hardware name: innotek GmbH VirtualBox/VirtualBox, BIOS VirtualBox 12/01/2006
[  100.473818] RIP: 0010:hello_exit+0xc/0x1000 [hello]
[  100.473938] Code: Bad RIP value.
[  100.474018] RSP: 0018:ffffbd14406cbed8 EFLAGS: 00010246
[  100.474143] RAX: 000000000000002b RBX: 00000000000000b0 RCX: 0000000000000000
[  100.474312] RDX: 0000000000000000 RSI: ffff94b8bda17908 RDI: ffff94b8bda17908
[  100.474496] RBP: ffffffffc0345000 R08: ffff94b8bda17908 R09: 00000000000001b9
[  100.474664] R10: 0000000000000001 R11: ffffffff92ee37c0 R12: 0000000000000000
[  100.474833] R13: 0000000000000000 R14: 0000000000000000 R15: 0000000000000000
[  100.475002] FS:  00007f37d424a740(0000) GS:ffff94b8bda00000(0000) knlGS:0000000000000000
[  100.475192] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  100.475329] CR2: ffffffffc0342fe2 CR3: 000000007b780000 CR4: 00000000000406f0
[  100.475503] Call Trace:
[  100.475575]  __x64_sys_delete_module+0x13d/0x280
[  100.475698]  ? exit_to_usermode_loop+0xa7/0x100
[  100.475811]  do_syscall_64+0x5f/0x1a0
[  100.475912]  entry_SYSCALL_64_after_hwframe+0x44/0xa9
[  100.476037] RIP: 0033:0x7f37d4379fab
[  100.476127] Code: 73 01 c3 48 8b 0d dd fe 0b 00 f7 d8 64 89 01 48 83 c8 ff c3 66 2e 0f 1f 84 00 00 00 00 00 90 f3 0f 1e fa b8 b0 00 00 00 0f 05 <48> 3d 01 f0 ff ff 73 01 c3 48 8b 0d ad fe 0b 00 f7 d8 64 89 01 48
[  100.476569] RSP: 002b:00007fff8489a3f8 EFLAGS: 00000206 ORIG_RAX: 00000000000000b0
[  100.476748] RAX: ffffffffffffffda RBX: 000055e696d84780 RCX: 00007f37d4379fab
[  100.477316] RDX: 000000000000000a RSI: 0000000000000800 RDI: 000055e696d847e8
[  100.477860] RBP: 00007fff8489a458 R08: 0000000000000000 R09: 0000000000000000
[  100.478397] R10: 00007f37d43edac0 R11: 0000000000000206 R12: 00007fff8489a620
[  100.478948] R13: 00007fff8489c7a8 R14: 000055e696d842a0 R15: 000055e696d84780
[  100.479486] Modules linked in: hello(OE-) e1000 joydev i2c_piix4 video ip_tables vboxvideo(OE) drm_kms_helper ttm drm crct10dif_pclmul crc32_pclmul crc32c_intel ghash_clmulni_intel serio_raw vboxguest(OE) ata_generic pata_acpi
[  100.481065] CR2: 0000000000000000
[  100.481567] ---[ end trace ad23453eb9e8484f ]---
[  100.482050] RIP: 0010:hello_exit+0xc/0x1000 [hello]
[  100.482562] Code: Bad RIP value.
[  100.483004] RSP: 0018:ffffbd14406cbed8 EFLAGS: 00010246
[  100.483509] RAX: 000000000000002b RBX: 00000000000000b0 RCX: 0000000000000000
[  100.484062] RDX: 0000000000000000 RSI: ffff94b8bda17908 RDI: ffff94b8bda17908
[  100.484602] RBP: ffffffffc0345000 R08: ffff94b8bda17908 R09: 00000000000001b9
[  100.485122] R10: 0000000000000001 R11: ffffffff92ee37c0 R12: 0000000000000000
[  100.485661] R13: 0000000000000000 R14: 0000000000000000 R15: 0000000000000000
[  100.486178] FS:  00007f37d424a740(0000) GS:ffff94b8bda00000(0000) knlGS:0000000000000000
[  100.486734] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  100.487224] CR2: ffffffffc0342fe2 CR3: 000000007b780000 CR4: 00000000000406f0
```

Look how messy it is! I was so freaked out when I saw this that I didn't even
try to understand what it means at first. I tried debugging by trial and error
until I gave up.

To be clear, it is not the actual panic. I injected an error to our example to
demonstrate:

```c
static void __exit hello_exit(void) {
    int *p;

    printk(KERN_INFO "Goodbye, cruel world, I'm leaving you today");

    p = NULL;
    *p = 0;
}
```

After some research, I understood some of the messages enough to track down the
bug. `Oops` means the severity of the issue is high. There is the `BUG: kernel
NULL pointer dereference, address: 0000000000000000` that tells the actual
error and `RIP: 0010:hello_exit+0xc/0x1000 [hello]` is the CPU register of the
instruction that error happened.

```shell
$ gdb hello.ko
(gdb) list *(hello_exit+0xc)
0x70 is in hello_exit (/home/vagrant/hello/hello.c:22).
17      static void __exit hello_exit(void) {
18          int *p;
19          printk(KERN_INFO "Goodbye, cruel world, I'm leaving you today\n");
20
21          p = NULL;
22          *p = 0;
23      }
24
25      module_init(hello_init);
26      module_exit(hello_exit);
```

It shows the line number where the bug occured. With this information I
was able to track down the problem and eventually fixed it.

## Resources

1. https://www.tldp.org/LDP/lkmpg/2.6/html/c38.html
2. https://jvns.ca/blog/2014/09/18/you-can-be-a-kernel-hacker/
3. https://opensourceforu.com/2011/01/understanding-a-kernel-oops/
