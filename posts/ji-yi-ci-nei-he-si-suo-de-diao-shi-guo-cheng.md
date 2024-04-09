---
title: '记一次内核死锁的调试过程'
date: 2024-04-09 22:39:02
tags: []
published: true
hideInList: false
feature: 
isTop: false
---
# Kernel关机死锁问

# 背景

## 内核版本

Linux 4.4.194 ohci_hcd

## 问题描述

**执行shutdown -h now**
```
[  OK  ] Stopped Load Kernel Modules.

[  OK  ] Unmounted /share.

[  OK  ] Stopped target Local File Systems (Pre).

[  OK  ] Reached target Unmount All Filesystems.

[  OK  ] Stopped Create Static Device Nodes in /dev.

[  OK  ] Stopped Create System Users.

[  OK  ] Stopped Remount Root and Kernel File Systems.

[  OK  ] Reached target Shutdown.

[  OK  ] Reached target Final Step.

[  OK  ] Finished Power-Off.

[  OK  ] Reached target Power-Off.

[   69.279680]

[   69.281172] ======================================================

[   69.287345] [ INFO: possible circular locking dependency detected ]

[   69.293605] 4.4.194 #1 Not tainted

[   69.297001] -------------------------------------------------------

[   69.303265] systemd-shutdow/1 is trying to acquire lock:

[   69.303286]  (&policy->rwsem){+++++.}, at: [<ffffff800884335c>] cpufreq_update_policy+0x40/0x158

[   69.303287]

[   69.303287] but task is already holding lock:

[   69.303300]  (mdev_list_sem){++++..}, at: [<ffffff80085577d0>] rockchip_system_status_notifier+0x34/0x118

[   69.303302]

[   69.303302] which lock already depends on the new lock.

[   69.303302]

[   69.303303]

[   69.303303] the existing dependency chain (in reverse order) is:

[   69.303307]

[   69.303307] -> #2 (mdev_list_sem){++++..}:

[   69.303313]        [<ffffff800811f264>] __lock_acquire+0x1348/0x191c

[   69.303317]        [<ffffff800811febc>] lock_acquire+0x1e0/0x23c

[   69.303323]        [<ffffff8008c4ea64>] down_read+0x5c/0xdc

[   69.303327DDR Version 1.24 20191016

In

---

执行reboot，内核deadlock，会自动重启

Stopping WPA supplicant...

[  OK  ] Stopped WPA supplicant.

Stopping D-Bus System Message Bus...

[  OK  ] Stopped D-Bus System Message Bus.

[  OK  ] Stopped target Basic System.

[  OK  ] Stopped target Paths.

[  OK  ] Stopped target Slices.

[  OK  ] Removed slice User and Session Slice.

[  OK  ] Stopped target Sockets.

[  OK  ] Closed D-Bus System Message Bus Socket.

[  OK  ] Closed Docker Socket for the API.

[  OK  ] Stopped target System Initialization.

[  OK  ] Stopped target Local Encrypted Volumes.

[  OK  ] Stopped Dispatch Password …ts to Console Directory Watch.

[  OK  ] Stopped Forward Password R…uests to Wall Directory Watch.

[  OK  ] Stopped target Swap.

[  OK  ] Closed Syslog Socket.

Stopping Network Time Synchronization...

Stopping Update UTMP about System Boot/Shutdown...

[  OK  ] Stopped Network Name Resolution.

[  OK  ] Stopped Update UTMP about System Boot/Shutdown.

[  OK  ] Stopped ifup for eth0.

[  OK  ] Stopped Network Time Synchronization.

[  OK  ] Stopped Create Volatile Files and Directories.

[  OK  ] Stopped ifup for eth1.13.

[  OK  ] Stopped ifup for eth1.

[  OK  ] Stopped Raise network interfaces.

[  OK  ] Stopped target Local File Systems.

Unmounting /share...

[  OK  ] Stopped Apply Kernel Variables.

[  OK  ] Stopped Load Kernel Modules.

[  OK  ] Unmounted /share.

[  OK  ] Stopped target Local File Systems (Pre).

[  OK  ] Reached target Unmount All Filesystems.

[  OK  ] Stopped Create Static Device Nodes in /dev.

[  OK  ] Stopped Create System Users.

[  OK  ] Stopped Remount Root and Kernel File Systems.

[  OK  ] Reached target Shutdown.

[  OK  ] Reached target Final Step.

[  OK  ] Finished Reboot.

[  OK  ] Reached target Reboot.

[   21.169263] dw_wdt: unexpected close, system will reboot soon

[   21.550470]

[   21.551963] ======================================================

[   21.558139] [ INFO: possible circular locking dependency detected ]

[   21.564401] 4.4.194 #1 Not tainted

[   21.567799] -------------------------------------------------------

[   21.574059] systemd-shutdow/1 is trying to acquire lock:

[   21.579364]  (&policy->rwsem){+++++.}, at: [<ffffff800884335c>] cpufreq_update_policy+0x40/0x158

[   21.588223]

[   21.588223] but task is already holding lock:

[   21.594049]  (mdev_list_sem){++++..}, at: [<ffffff80085577d0>] rockchip_system_status_notifier+0x34/0x118

[   21.603681]

[   21.603681] which lock already depends on the new lock.

[   21.603681]

[   21.611852]

[   21.611852] the existing dependency chain (in reverse order) is:

[   21.619321]

- > #2 (mdev_list_sem){++++..}:

[   21.623668]        [<ffffff800811f264>] __lock_acquire+0x1348/0x191c

[   21.630031]        [<ffffff800811febc>] lock_acquire+0x1e0/0x23c

[   21.636052]        [<ffffff8008c4ea64>] down_read+0x5c/0xdc

[   21.641641]        [<ffffff8008557310>] rockchip_monitor_cpufreq_policy_notifier+0x48/0x18c

[   21.649995]        [<ffffff80080e0c94>] notifier_call_chain+0x78/0x98

[   21.656451]        [<ffffff80080e10b0>] __blocking_notifier_call_chain+0x58/0x84

[   21.663850]        [<ffffff80080e1118>] blocking_notifier_call_chain+0x3c/0x4c

[   21.671079]        [<ffffff8008842f48>] cpufreq_set_policy+0xac/0x480

[   21.677535]        [<ffffff8008843adc>] cpufreq_init_policy+0x90/0xc0

[   21.683990]        [<ffffff80088443a4>] cpufreq_online+0x694/0x70c

[   21.690181]        [<ffffff8008844514>] cpufreq_add_dev+0x7c/0xd0

[   21.696286]        [<ffffff800865d148>] subsys_interface_register+0xb8/0xcc

[   21.703264]        [<ffffff80088437b8>] cpufreq_register_driver+0x134/0x204

[   21.710239]        [<ffffff800884dff4>] dt_cpufreq_probe+0x158/0x174

[   21.716600]        [<ffffff80086611fc>] platform_drv_probe+0x5c/0xb0

[   21.722960]        [<ffffff800865ed5c>] driver_probe_device+0x264/0x3d4

[   21.729585]        [<ffffff800865f034>] __device_attach_driver+0x68/0xa4

[   21.736295]        [<ffffff800865ce08>] bus_for_each_drv+0x90/0xa0

[   21.742486]        [<ffffff800865e9e8>] __device_attach+0xb4/0x130

[   21.748676]        [<ffffff800865f1d0>] device_initial_probe+0x24/0x30

[   21.755217]        [<ffffff800865de5c>] bus_probe_device+0x38/0x9c

[   21.761407]        [<ffffff800865bb84>] device_add+0x480/0x528

[   21.767247]        [<ffffff8008660ee4>] platform_device_add+0xe8/0x22c

[   21.773786]        [<ffffff8008661ae8>] platform_device_register_full+0xac/0xec

[   21.781100]        [<ffffff80091fd464>] rockchip_cpufreq_driver_init+0xb8/0x2f8

[   21.788417]        [<ffffff80080830f8>] do_one_initcall+0x78/0x198

[   21.794608]        [<ffffff80091c0e94>] kernel_init_freeable+0x27c/0x280

[   21.801319]        [<ffffff8008c4a0e8>] kernel_init+0x18/0x100

[   21.807159]        [<ffffff8008082f10>] ret_from_fork+0x10/0x40

[   21.813084]

- > #1 ((cpufreq_policy_notifier_list).rwsem){.+.+.+}:

[   21.819422]        [<ffffff800811f264>] __lock_acquire+0x1348/0x191c

[   21.825782]        [<ffffff800811febc>] lock_acquire+0x1e0/0x23c

[   21.831803]        [<ffffff8008c4ea64>] down_read+0x5c/0xdc

[   21.837389]        [<ffffff80080e1098>] __blocking_notifier_call_chain+0x40/0x84

[   21.844787]        [<ffffff80080e1118>] blocking_notifier_call_chain+0x3c/0x4c

[   21.852015]        [<ffffff8008844350>] cpufreq_online+0x640/0x70c

[   21.858206]        [<ffffff8008844514>] cpufreq_add_dev+0x7c/0xd0

[   21.864312]        [<ffffff800865d148>] subsys_interface_register+0xb8/0xcc

[   21.871287]        [<ffffff80088437b8>] cpufreq_register_driver+0x134/0x204

[   21.878262]        [<ffffff800884dff4>] dt_cpufreq_probe+0x158/0x174

[   21.884623]        [<ffffff80086611fc>] platform_drv_probe+0x5c/0xb0

[   21.890982]        [<ffffff800865ed5c>] driver_probe_device+0x264/0x3d4

[   21.897607]        [<ffffff800865f034>] __device_attach_driver+0x68/0xa4

[   21.904317]        [<ffffff800865ce08>] bus_for_each_drv+0x90/0xa0

[   21.910507]        [<ffffff800865e9e8>] __device_attach+0xb4/0x130

[   21.916698]        [<ffffff800865f1d0>] device_initial_probe+0x24/0x30

[   21.923238]        [<ffffff800865de5c>] bus_probe_device+0x38/0x9c

[   21.929429]        [<ffffff800865bb84>] device_add+0x480/0x528

[   21.935269]        [<ffffff8008660ee4>] platform_device_add+0xe8/0x22c

[   21.941809]        [<ffffff8008661ae8>] platform_device_register_full+0xac/0xec

[   21.949122]        [<ffffff80091fd464>] rockchip_cpufreq_driver_init+0xb8/0x2f8

[   21.956436]        [<ffffff80080830f8>] do_one_initcall+0x78/0x198

[   21.962626]        [<ffffff80091c0e94>] kernel_init_freeable+0x27c/0x280

[   21.969335]        [<ffffff8008c4a0e8>] kernel_init+0x18/0x100

[   21.975176]        [<ffffff8008082f10>] ret_from_fork+0x10/0x40

[   21.981101]

- > #0 (&policy->rwsem){+++++.}:

[   21.985531]        [<ffffff800811ae44>] print_circular_bug+0x60/0x2bc

[   21.991986]        [<ffffff800811ee1c>] __lock_acquire+0xf00/0x191c

[   21.998261]        [<ffffff800811febc>] lock_acquire+0x1e0/0x23c

[   22.004281]        [<ffffff8008c4eb44>] down_write+0x60/0xd8

[   22.009953]        [<ffffff800884335c>] cpufreq_update_policy+0x40/0x158

[   22.016663]        [<ffffff8008557878>] rockchip_system_status_notifier+0xdc/0x118

[   22.024243]        [<ffffff80080e0c94>] notifier_call_chain+0x78/0x98

[   22.030697]        [<ffffff80080e10b0>] __blocking_notifier_call_chain+0x58/0x84

[   22.038096]        [<ffffff80080e1118>] blocking_notifier_call_chain+0x3c/0x4c

[   22.045324]        [<ffffff8008556b58>] rockchip_system_status_notifier_call_chain+0x2c/0x54

[   22.053773]        [<ffffff8008556be8>] rockchip_set_system_status+0x68/0xc8

[   22.060833]        [<ffffff8008557628>] rockchip_monitor_reboot_notifier+0x18/0x3c

[   22.068412]        [<ffffff80080e0c94>] notifier_call_chain+0x78/0x98

[   22.074867]        [<ffffff80080e10b0>] __blocking_notifier_call_chain+0x58/0x84

[   22.082265]        [<ffffff80080e1118>] blocking_notifier_call_chain+0x3c/0x4c

[   22.089493]        [<ffffff80080e2f54>] kernel_restart_prepare+0x2c/0x50

[   22.096203]        [<ffffff80080e30a0>] kernel_restart+0x20/0x68

[   22.102223]        [<ffffff80080e33f8>] SyS_reboot+0x180/0x1dc

[   22.108064]        [<ffffff8008082f70>] el0_svc_naked+0x24/0x28

[   22.113989]

[   22.113989] other info that might help us debug this:

[   22.113989]

[   22.121980] Chain exists of:

&policy->rwsem --> (cpufreq_policy_notifier_list).rwsem --> mdev_list_sem

[   22.131813]  Possible unsafe locking scenario:

[   22.131813]

[   22.137724]        CPU0                    CPU1

[   22.142255]        ----                    ----

[   22.146787]   lock(mdev_list_sem);

[   22.150223]                                lock((cpufreq_policy_notifier_list).rwsem);

[   22.158164]                                lock(mdev_list_sem);

[   22.164111]   lock(&policy->rwsem);

[   22.167632]

[   22.167632]  *** DEADLOCK ***

[   22.167632]

[   22.173545] 5 locks held by systemd-shutdow/1:

[   22.177980]  #0:  (reboot_mutex){+.+...}, at: [<ffffff80080e3378>] SyS_reboot+0x100/0x1dc

[   22.186233]  #1:  ((reboot_notifier_list).rwsem){.+.+..}, at: [<ffffff80080e1098>] __blocking_notifier_call_chain+0x40/0x84

[   22.197441]  #2:  (system_status_mutex){+.+...}, at: [<ffffff8008556bac>] rockchip_set_system_status+0x2c/0xc8

[   22.207527]  #3:  ((system_status_notifier_list).rwsem){.+.+..}, at: [<ffffff80080e1098>] __blocking_notifier_call_chain+0x40/0x84

[   22.219339]  #4:  (mdev_list_sem){++++..}, at: [<ffffff80085577d0>] rockchip_system_status_notifier+0x34/0x118

[   22.229425]

[   22.229425] stack backtrace:

[   22.233781] CPU: 4 PID: 1 Comm: systemd-shutdow Not tainted 4.4.194 #1

[   22.240306] Hardware name: Rockchip RK3399 Board Vclusters (DT)

[   22.246218] Call trace:

[   22.248664] [<ffffff8008088a40>] dump_backtrace+0x0/0x234

[   22.254057] [<ffffff8008088c98>] show_stack+0x24/0x30

[   22.259111] [<ffffff80084b0ba4>] dump_stack+0xb4/0xf4

[   22.264163] [<ffffff800811afcc>] print_circular_bug+0x1e8/0x2bc

[   22.270075] [<ffffff800811ee1c>] __lock_acquire+0xf00/0x191c

[   22.275731] [<ffffff800811febc>] lock_acquire+0x1e0/0x23c

[   22.281124] [<ffffff8008c4eb44>] down_write+0x60/0xd8

[   22.286177] [<ffffff800884335c>] cpufreq_update_policy+0x40/0x158

[   22.292270] [<ffffff8008557878>] rockchip_system_status_notifier+0xdc/0x118

[   22.299221] [<ffffff80080e0c94>] notifier_call_chain+0x78/0x98

[   22.305047] [<ffffff80080e10b0>] __blocking_notifier_call_chain+0x58/0x84

[   22.311828] [<ffffff80080e1118>] blocking_notifier_call_chain+0x3c/0x4c

[   22.318440] [<ffffff8008556b58>] rockchip_system_status_notifier_call_chain+0x2c/0x54

[   22.326262] [<ffffff8008556be8>] rockchip_set_system_status+0x68/0xc8

[   22.332693] [<ffffff8008557628>] rockchip_monitor_reboot_notifier+0x18/0x3c

[   22.339644] [<ffffff80080e0c94>] notifier_call_chain+0x78/0x98

[   22.345470] [<ffffff80080e10b0>] __blocking_notifier_call_chain+0x58/0x84

[   22.352251] [<ffffff80080e1118>] blocking_notifier_call_chain+0x3c/0x4c

[   22.358863] [<ffffff80080e2f54>] kernel_restart_prepare+0x2c/0x50

[   22.364954] [<ffffff80080e30a0>] kernel_restart+0x20/0x68

[   22.370346] [<ffffff80080e33f8>] SyS_reboot+0x180/0x1dc

[   22.375568] [<ffffff8008082f70>] el0_svc_naked+0x24/0x28

[   22.381157] cpu cpu4: min=816000, max=816000

[   22.386760] cpu cpu0: min=816000, max=816000

[   22.401700] rk-vcodec ff660000.rkvdec: shutdown

[   22.406278] rk-vcodec ff650000.vpu_service: shutdown

[   22.415938] reboot: Restarting system

```

## 原因

Rockchip在初始化时，会注册一个cpu状态更新和cpu策略更新的通知回调。两个回调都会对mdev_list_sem上锁

关机时会更新cpu状态（怀疑关机就是将CPU频率设置为0，这样CPU就不工作了）

更新cpu频率时cpufreq驱动会通知rockchip_system_status_notifier更新cpu状态

Rockchip在更新cpu状态时，反过来再去设置cpu频率并更新CPU策略

而上一次更新CPU状态时，已经对mdev_list_sem上锁并未释放，现在更新CPU策略又要再次对它上锁，因而死锁

## 解决办法

在更新CPU设备状态时，先不着急更新CPU策略。

先把所有CPU 设备状态更新了，并记录哪些CPU设备状态已经更新，然后释放锁。

再更新记录的CPU设备策略。死锁解决

# 结束