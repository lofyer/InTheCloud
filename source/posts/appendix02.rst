================================
附录二 
================================

-----------------------------------
虚拟机占用主机资源隔离（公有云参考）
-----------------------------------

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

1. 更改内核磁盘IO调度方式：noop，cfq，deadline直接选择，参考 `Linux Doc. <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/block>`_ 。

2. 使用Linux的 `Control Group <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/cgroups>`_ 来限制虚拟机对宿主机设备的访问。

    使能cgroup，假如没有启动的话。

    .. code::
 
        # ount tmpfs cgroup_root /sys/fs/cgroup
        # mkdir /sys/fs/cgroup/blkio
        # mount -t cgroup -o blkio none /sys/fs/cgroup/blkio

    创建一个1 Mbps的IO限制组。

    .. code::
 
        # mkdir -p /sys/fs/cgroup/blkio/limit1M/
        # echo "X:Y  1048576" > /sys/fs/cgroup/blkio/limit1M/blkio.throttle.write_bps_device

    将虚拟机进程附加到限制组。

    .. code::
 
        # echo $VM_PID > /sys/fs/cgroup/blkio/limit1M/tasks

设备
-----

使用Linux的 `Control Group <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/cgroups>`_ 来限制虚拟机对宿主机设备的访问。

---------------------------
资源用度、计费（公有云参考）
---------------------------

系统报告主要是在scaling（测量）、charging（计费）时使用。

在测量时可以使用平台提供的API、测量组件，综合利用nagios/Icinga，使用Django快速开发。

oVirt可以参考ovirt-reports，OpenStack参考其Ceilometer。

计费模块OpenStack可参考新浪云的 `dough项目 <https://github.com/sinacloud/dough>`_ 。
