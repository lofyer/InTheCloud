================================
附录二 
================================

----------------------
虚拟机占用主机资源隔离
----------------------

网络
-----

**VLAN**

**openvswitch**

CPU
-----

使用CPU Pin可以将特定虚拟核固定在指定物理核上。

线程当作核来使用的话，每一个虚拟机使用一个核，之间互不干涉。

磁盘IO
-------

目前没有比较好的方式，建议更改内核磁盘IO调度方式：noop，cfq，deadline直接选择，参考 `Linux Doc. <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/block>`_ 。

设备
-----

使用Linux的 `Control Group <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/cgroups>`_ 来限制虚拟机对宿主机设备的访问。

--------
系统报告
--------

系统报告主要是在scaling（测量）、charging（计费）时使用，oVirt可以参考ovirt-reports，OpenStack参考其Ceilometer。
