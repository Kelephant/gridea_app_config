---
title: '内核Call Trace调试'
date: 2024-04-09 15:43:29
tags: [kernel,debug,gdb]
published: true
hideInList: false
feature: /post-images/nei-he-call-trace-diao-shi.webp
isTop: false
---
# 参考文章

1. 该链接讲述了Call Trace 的调试方法
[decode-kernel-call-trace.md](https://gist.github.com/doughgle/735229c34c52f9006ca92a2cf24da990)

2. 带symbol的内核镜像安装，正常情况下/usr/lib/debug/boot/会保存带symbol的内核镜像，如果找不到，可以按照以下方式自行
[https://wiki.ubuntu.com/Debug Symbol Packages](https://wiki.ubuntu.com/Debug%20Symbol%20Packages)

# 调试步骤

## 创建调试目录

```jsx
mkdir -p ~/workspace/call_trace
```

## 设计异常模块

1. 创建kernel_call_trace.c文件

```jsx
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/mm.h>

char *ptr = NULL;

int __init init_module(void) {
  //ptr = kmalloc(sizeof(int), GFP_KERNEL);

  *ptr = 0;

  printk(KERN_ALERT "Call trace:\n");
  //backtrace();

  // BUG();

  return 0;
}

void __exit cleanup_module(void) {
  //kfree(ptr);
}
```

2. 创建Makefile文件
```jsx
obj-m := kernel_call_trace.o

all:
        make -C /usr/src/linux-headers-$(shell uname -r) M=$(PWD) modules

clean:
        make -C /usr/src/linux-headers-$(shell uname -r) M=$(PWD) clean
```

3. make，生成 kernel_call_trace.ko 模块
```jsx
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ make 
make -C /usr/src/linux-headers-6.2.0-37-generic M=/home/wenqikai/workspace/call_trace modules
make[1]: Entering directory '/usr/src/linux-headers-6.2.0-37-generic'
warning: the compiler differs from the one used to build the kernel
  The kernel was built by: x86_64-linux-gnu-gcc-11 (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
  You are using:           gcc-11 (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
  CC [M]  /home/wenqikai/workspace/call_trace/kernel_call_trace.o
  MODPOST /home/wenqikai/workspace/call_trace/Module.symvers
  CC [M]  /home/wenqikai/workspace/call_trace/kernel_call_trace.mod.o
  LD [M]  /home/wenqikai/workspace/call_trace/kernel_call_trace.ko
  BTF [M] /home/wenqikai/workspace/call_trace/kernel_call_trace.ko
Skipping BTF generation for /home/wenqikai/workspace/call_trace/kernel_call_trace.ko due to unavailability of vmlinux
make[1]: Leaving directory '/usr/src/linux-headers-6.2.0-37-generic'
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ ls -l
total 264
-rw-rw-r-- 1 wenqikai wenqikai   3611 12月 19 13:37 call_trace.log
-rwxr-xr-x 1 wenqikai wenqikai   7383 12月 19 11:26 decode_stacktrace.sh
-rw-rw-r-- 1 wenqikai wenqikai    343 12月 19 11:33 kernel_call_trace.c
-rw-rw-r-- 1 wenqikai wenqikai 115376 12月 19 14:13 kernel_call_trace.ko
-rw-rw-r-- 1 wenqikai wenqikai     56 12月 19 14:13 kernel_call_trace.mod
-rw-rw-r-- 1 wenqikai wenqikai    891 12月 19 14:13 kernel_call_trace.mod.c
-rw-rw-r-- 1 wenqikai wenqikai  93400 12月 19 14:13 kernel_call_trace.mod.o
-rw-rw-r-- 1 wenqikai wenqikai  23344 12月 19 14:13 kernel_call_trace.o
-rw-rw-r-- 1 wenqikai wenqikai    176 12月 19 14:12 Makefile
-rw-rw-r-- 1 wenqikai wenqikai     56 12月 19 14:13 modules.order
-rw-rw-r-- 1 wenqikai wenqikai      0 12月 19 14:13 Module.symvers
```

4. 加载kernel_call_trace.ko 模块，由于该内核模块是我们故意制造异常的，所以在一加载时便会报错，通过dmesg可以查到call trace信息。下面标红部分是call trace的完整信息，其中其中标蓝部分表示异常的模块名
```jsx
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ sudo insmod  kernel_call_trace.ko 
[sudo] password for wenqikai: 
Killed
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ sudo dmesg | tail -100
...
[ 2023-12-19 11:34:15 ] [70466.575935] Call Trace:
[ 2023-12-19 11:34:15 ] [70466.578246]  <TASK>
[ 2023-12-19 11:34:15 ] [70466.585635]  ? show_regs+0x72/0x90
[ 2023-12-19 11:34:15 ] [70466.614356]  ? __die+0x25/0x80
[ 2023-12-19 11:34:15 ] [70466.614381]  ? page_fault_oops+0x79/0x190
[ 2023-12-19 11:34:15 ] [70466.615773]  ? do_user_addr_fault+0x30c/0x640
[ 2023-12-19 11:34:15 ] [70466.615775]  ? down_write+0x13/0x90
[ 2023-12-19 11:34:15 ] [70466.634448]  ? exc_page_fault+0x81/0x1b0
[ 2023-12-19 11:34:15 ] [70466.635485]  ? asm_exc_page_fault+0x27/0x30
[ 2023-12-19 11:34:15 ] [70466.635793]  ? __pfx_init_module+0x10/0x10 [kernel_call_trace]
[ 2023-12-19 11:34:15 ] [70466.635802]  ? init_module+0x14/0xff0 [kernel_call_trace]
[ 2023-12-19 11:34:15 ] [70466.635806]  ? do_one_initcall+0x46/0x240
[ 2023-12-19 11:34:15 ] [70466.635870]  ? kmalloc_trace+0x2a/0xb0
[ 2023-12-19 11:34:15 ] [70466.636483]  do_init_module+0x52/0x240
[ 2023-12-19 11:34:15 ] [70466.636737]  load_module+0xb96/0xd60
[ 2023-12-19 11:34:15 ] [70466.636740]  ? kernel_read_file+0x25c/0x2b0
[ 2023-12-19 11:34:15 ] [70466.636965]  __do_sys_finit_module+0xcc/0x150
[ 2023-12-19 11:34:15 ] [70466.636967]  ? __do_sys_finit_module+0xcc/0x150
[ 2023-12-19 11:34:15 ] [70466.636972]  __x64_sys_finit_module+0x18/0x30
[ 2023-12-19 11:34:15 ] [70466.636974]  do_syscall_64+0x59/0x90
[ 2023-12-19 11:34:15 ] [70466.636994]  ? irqentry_exit+0x43/0x50
[ 2023-12-19 11:34:15 ] [70466.636997]  ? exc_page_fault+0x92/0x1b0
[ 2023-12-19 11:34:15 ] [70466.637000]  entry_SYSCALL_64_after_hwframe+0x73/0xdd
[ 2023-12-19 11:34:15 ] [70466.637003] RIP: 0033:0x7ff0b911e69d
[ 2023-12-19 11:34:15 ] [70466.637007] Code: 5b 41 5c c3 66 0f 1f 84 00 00 00 00 00 f3 0f 1e fa 48 89 f8 48 89 f7 48 89 d6 48 89 ca 4d 89 c2 4d 89 c8 4c 8b 4c 24 08 0f 05 <48> 3d 01 f0 ff ff 73 01 c3 48 8b 0d 63 a7 0f 00 f7 d8 64 89 01 48
[ 2023-12-19 11:34:15 ] [70466.637009] RSP: 002b:00007fff662d0fc8 EFLAGS: 00000246 ORIG_RAX: 0000000000000139
[ 2023-12-19 11:34:15 ] [70466.637012] RAX: ffffffffffffffda RBX: 0000562af52237b0 RCX: 00007ff0b911e69d
[ 2023-12-19 11:34:15 ] [70466.637013] RDX: 0000000000000000 RSI: 0000562af4eb5cd2 RDI: 0000000000000003
[ 2023-12-19 11:34:15 ] [70466.637015] RBP: 0000000000000000 R08: 0000000000000000 R09: 0000000000000000
[ 2023-12-19 11:34:15 ] [70466.637016] R10: 0000000000000003 R11: 0000000000000246 R12: 0000562af4eb5cd2
[ 2023-12-19 11:34:15 ] [70466.637018] R13: 0000562af5223760 R14: 0000562af4eb4888 R15: 0000562af52238d0
```

5. 将Call Trace信息存储到call_trace.log文件中
6. 拷贝内核符号表解析工具到当前目录下
```jsx
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ cp /usr/src/linux-headers-$(uname -r)/scripts/decode_stacktrace.sh .
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ ls -l
total 264
-rw-rw-r-- 1 wenqikai wenqikai   3611 12月 19 13:37 call_trace.log
-rwxr-xr-x 1 wenqikai wenqikai   7383 12月 19 14:22 decode_stacktrace.sh
-rw-rw-r-- 1 wenqikai wenqikai    343 12月 19 11:33 kernel_call_trace.c
-rw-rw-r-- 1 wenqikai wenqikai 115376 12月 19 14:13 kernel_call_trace.ko
-rw-rw-r-- 1 wenqikai wenqikai     56 12月 19 14:13 kernel_call_trace.mod
-rw-rw-r-- 1 wenqikai wenqikai    891 12月 19 14:13 kernel_call_trace.mod.c
-rw-rw-r-- 1 wenqikai wenqikai  93400 12月 19 14:13 kernel_call_trace.mod.o
-rw-rw-r-- 1 wenqikai wenqikai  23344 12月 19 14:13 kernel_call_trace.o
-rw-rw-r-- 1 wenqikai wenqikai    176 12月 19 14:12 Makefile
-rw-rw-r-- 1 wenqikai wenqikai     56 12月 19 14:13 modules.order
-rw-rw-r-- 1 wenqikai wenqikai      0 12月 19 14:13 Module.symvers
```

7. 使用以下目录解析
./decode_stacktrace.sh /usr/lib/debug/boot/vmlinux-$(uname -r)  auto /home/wenqikai/workspace/call_trace/ < call_trace.log 

参数解析：
- ./decode_stacktrace.sh：解析程序
- /usr/lib/debug/boot/vmlinux-$(uname -r)：带符号表的内核文件，如果找不到该文件，则参考该链接操作安装[https://wiki.ubuntu.com/Debug Symbol Packages](https://wiki.ubuntu.com/Debug%20Symbol%20Packages)
- auto： 这里应该是自动找到系统模块目录
- /home/wenqikai/workspace/call_trace/：这里是我们自己的模块存放目录
- call_trace.log：call trace文件
```jsx
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ ./decode_stacktrace.sh 
ERROR! vmlinux image must be specified
Usage:
        ./decode_stacktrace.sh -r <release> | <vmlinux> [<base path>|auto] [<modules path>]
wenqikai@wenqikai-virtual-machine:~/workspace/call_trace$ ./decode_stacktrace.sh /usr/lib/debug/boot/vmlinux-$(uname -r)  auto /home/wenqikai/workspace/call_trace/ < call_trace.log 
[70466.575935] Call Trace:
[70466.578246]  <TASK>
[70466.585635] ? show_regs (arch/x86/kernel/dumpstack.c:479) 
[70466.614356] ? __die (arch/x86/kernel/dumpstack.c:421 arch/x86/kernel/dumpstack.c:434) 
[70466.614381] ? page_fault_oops (arch/x86/mm/fault.c:727) 
[70466.615773] ? do_user_addr_fault (arch/x86/mm/fault.c:1270) 
[70466.615775] ? down_write (arch/x86/include/asm/preempt.h:80 kernel/locking/rwsem.c:261 kernel/locking/rwsem.c:1313 kernel/locking/rwsem.c:1323 kernel/locking/rwsem.c:1574) 
[70466.634448] ? exc_page_fault (arch/x86/include/asm/paravirt.h:700 arch/x86/mm/fault.c:1479 arch/x86/mm/fault.c:1527) 
[70466.635485] ? asm_exc_page_fault (arch/x86/include/asm/idtentry.h:570) 
[70466.635793] ? __pfx_init_module (/home/wenqikai/workspace/call_trace/kernel_call_trace.c:7) kernel_call_trace
[70466.635802] ? init_module (/home/wenqikai/workspace/call_trace/kernel_call_trace.c:10) kernel_call_trace
[70466.635806] ? do_one_initcall (init/main.c:1295) 
[70466.635870] ? kmalloc_trace (mm/slab_common.c:1065) 
[70466.636483] do_init_module (kernel/module/main.c:2463) 
[70466.636737] load_module (kernel/module/main.c:2872) 
[70466.636740] ? kernel_read_file (fs/kernel_read_file.c:110) 
[70466.636965] __do_sys_finit_module (kernel/module/main.c:2972) 
[70466.636967] ? __do_sys_finit_module (kernel/module/main.c:2972) 
[70466.636972] __x64_sys_finit_module (kernel/module/main.c:2939) 
[70466.636974] do_syscall_64 (arch/x86/entry/common.c:50 arch/x86/entry/common.c:80) 
[70466.636994] ? irqentry_exit (kernel/entry/common.c:446) 
[70466.636997] ? exc_page_fault (arch/x86/mm/fault.c:1531) 
[70466.637000] entry_SYSCALL_64_after_hwframe (arch/x86/entry/entry_64.S:120) 
[70466.637003] RIP: 0033:0x7ff0b911e69d
./decode_stacktrace.sh: line 225: ./decodecode: No such file or directory
[70466.637009] RSP: 002b:00007fff662d0fc8 EFLAGS: 00000246 ORIG_RAX: 0000000000000139
[70466.637012] RAX: ffffffffffffffda RBX: 0000562af52237b0 RCX: 00007ff0b911e69d
[70466.637013] RDX: 0000000000000000 RSI: 0000562af4eb5cd2 RDI: 0000000000000003
[70466.637015] RBP: 0000000000000000 R08: 0000000000000000 R09: 0000000000000000
[70466.637016] R10: 0000000000000003 R11: 0000000000000246 R12: 0000562af4eb5cd2
[70466.637018] R13: 0000562af5223760 R14: 0000562af4eb4888 R15: 0000562af52238d0
[70466.637038]  </TASK>
```

# 其他

包查找方法
方法1. sudo aptitude search linux-image-$(uname -r)-dbgsym

方法2. 	$ apt search linux-image-$(uname -r)

Sorting... Done
Full Text Search... Done
linux-image-5.4.0-80-generic/focal-updates,focal-security,now 5.4.0-80.90 amd64 [installed,automatic]
Signed kernel image generic

```
	linux-image-5.4.0-80-generic-dbgsym/focal-updates,now 5.4.0-80.90 amd64 [installed]
	  Signed kernel image generic

	$ sudo apt install linux-image-`uname -r`-dbgsym

```

# 结束