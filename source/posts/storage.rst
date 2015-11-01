===================================
虚拟化中存储配置典型场景－启动风暴
===================================

桌面私有云相较于一般公有云，有着虚拟机启动密度大、服务器资源冗余性低等特点，本文尝试模拟桌面云中的教学机房场景来说明启动风暴问题的解决方法，并在现有方法的基础上提出一些改进措施，对公有云也有一定参考价值。

1. 现有虚拟化平台的一般配置介绍
===============================

1. 磁盘分配方式

现有比较成熟的开源虚拟化/云计算平台，包括OpenStack、CloudStack和oVirt等，都有比较清晰的模板和实例概念，同时也有模板与实例镜像存储位置相关的存储域配置，为了区分虚拟机中的模板与实例，我们下文都默认：模板即原始虚拟机，所有新实例的创建都依赖于它；实例即依赖模板创建的虚拟机，是具有运行生命周期的虚拟机通称。

在虚拟化平台中，一般有两种新建实例的方式：克隆与增量。

克隆即完整克隆模板配置与磁盘，磁盘的创建方式一般为cp或者qemu-img convert。如此创建的实例，其磁盘与模板磁盘相对独立，在服务器存储上拥有各自的扇区位置，所以在读写操作时受机械盘磁头引起的小区域并发问题影响较小，缺点是创建时需要消耗一定时间，不能满足秒级创建的需求。

增量即新建虚拟机只复制模板配置信息，但其磁盘文件通过qemu-img的backing file方法创建，基于backing file创建的磁盘不能是raw格式，命令如下：

.. code::

       qemu-img create -b hda.qcow2 -f qcow2 hda-new.qcow2

如此创建的磁盘对模板磁盘的依赖较大，非增量文件的读操作（比如启动系统）绝大部分在模板磁盘上进行，所以在传统机械盘上多个实例的并发启动风暴更容易在此种格式的磁盘上发生。

2. 分布式存储管理方式

-----------------------
1.1. 模板与实例同一存储
-----------------------

当模板与实例存储在同一文件系统中时，我们创建虚拟机的常用方法有两种————使用模板的磁盘的hardlink作为实例磁盘的backing file来创建增量磁盘，或者直接克隆模板磁盘。

于oVirt中我们可以看到如下主存储域的目录结构：

.. code::

    # tree data/
    data/4440c3f9-ba1f-4803-86ac-d59c694c61c1/
    ├── dom_md
    │   ├── ids
    │   ├── inbox
    │   ├── leases
    │   ├── metadata
    │   └── outbox
    ├── images
    │   ├── d25c4d22-15c1-4f3c-8aef-2ed74deb4723
    │   │   ├── 2bf1ba4c-3fb5-4e25-9368-cc96724b1a63
    │   │   ├── 2bf1ba4c-3fb5-4e25-9368-cc96724b1a63.lease
    │   │   └── 2bf1ba4c-3fb5-4e25-9368-cc96724b1a63.meta
    │   ├── d6d7e74d-76fc-48c6-b3c6-3c506b98e73e
    │   │   ├── 2bf1ba4c-3fb5-4e25-9368-cc96724b1a63
    │   │   ├── 2bf1ba4c-3fb5-4e25-9368-cc96724b1a63.lease
    │   │   ├── 2bf1ba4c-3fb5-4e25-9368-cc96724b1a63.meta
    │   │   ├── 81031024-5942-4086-901b-f991777ff32a
    │   │   ├── 81031024-5942-4086-901b-f991777ff32a.lease
    │   │   └── 81031024-5942-4086-901b-f991777ff32a.meta
    │   └── ed834c83-82c3-4ea3-a08d-82cfbb213af4
    │       ├── 3e56f6dd-be45-48d0-aba8-830d0ee1f068
    │       ├── 3e56f6dd-be45-48d0-aba8-830d0ee1f068.lease
    │       └── 3e56f6dd-be45-48d0-aba8-830d0ee1f068.meta
    └── master
        ├── tasks
        └── vms

其中，2bf1ba4c-3fb5-4e25-9368-cc96724b1a63为模板磁盘，d6d7e74d-76fc-48c6-b3c6-3c506b98e73e即为从模板创建的增量实例磁盘。

此时，实例的所有操作都在此文件系统上进行。

.. note:: 在同一文件系统上使用hardlink的而非softlink的原因

    1. 节省inode使用量
    2. 由于省去了访问softlink的inode这一步，直接访问原文件的inode会带来性能上的稍许提升
    3. 改变源文件路径不会影响hardlink文件的访问
    4. 冗余性及安全性考虑，只有删除了inode上的所有引用，此inode才会被删除。

-----------------------
1.2. 模板与实例分离存储
-----------------------

当模板磁盘与实例磁盘分别存储在不同的文件系统中时，已经不能使用hardlink创建。oVirt跨存储域创建实例的方式为克隆创建，而不是直接使用backing file创建增量磁盘。我们从其存储域概念可以推测出它这么做的原因，即：

1. 存储域间没有模板与实例的关联，易于存储域管理（删除、导入存储域与其中的虚拟机）；

2. 对于跨文件系统的存储域，使用拷贝而不是增量创建更易于减少模板所在存储域负担。

而有时我们真实部署虚拟桌面的场景中，往往需要多个本地存储域（比如SAS盘与SSD盘）混合使用。所以在以下的测试中，我也会使用跨存储域创建增量磁盘的方式。

-----------------------------------
1.3. 无状态实例的磁盘与快照分离存储
-----------------------------------

oVirt中存在一种“无状态”实例，此种实例的创建过程如下：

已有模板磁盘A，我们根据采用克隆或者增量方式创建实例磁盘B，勾选“无状态”以后，虚拟机运行会自动创建磁盘B的增量磁盘C，实例所有的改动都在C上，当虚拟机关机后，平台删除C。这样以来，所有的改动便随之删除，我们就称这种工作方式的实例为“无状态”实例。

这种状态下虚拟机，存在1-2个增量磁盘，在以下的实验中，我会将其分别放置在不同的文件系统中测试。

2. 启动风暴相关系列试验
=======================

此次实验的目的为考察多台虚拟机同时启动对磁盘IO的负载，不考虑qcow2格式与raw格式的影响，统一适用qcow2格式。所有的虚拟机均使用VirtIO接口，qcow2磁盘，backing_file格式也为qcow2，Windows XP 32位操作系统，无任何附加软件。

模板参数：

.. code::

    file: base_xp.sh
    #!/bin/bash
    /usr/libexec/qemu-kvm -no-user-config -nodefaults \
    -m 1024M -cpu host -smp 1,sockets=1,cores=1 \
    -net tap,ifname=tap0,script=no,downscript=no -net nic,model=virtio \
    -monitor stdio -vga qxl -global qxl-vga.vram_size=67108864 \
    -spice port=7001,ipv4,disable-ticketing \
    -drive file=hda.qcow2,if=none,id=drive-virtio-disk0,format=qcow2,cache=none,werror=stop,rerror=stop,aio=threads \
    -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x7,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 \
    -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x8

实验脚本：

.. code::

    创建20个以hda.qcow2为backing file的磁盘，用于实例。

    file: create-imgs.sh
    #!/bin/bash
    for i in `seq 11 30`
    do
        qemu-img create -f qcow2 -b hda.qcow2 hda-$i.qcow2
    done

    一次性启动20台实例。

    file: start-vms.sh
    #!/bin/bash
    function startvm {
        /usr/libexec/qemu-kvm -no-user-config -nodefaults \
        -m 1024M -cpu host -smp 1,sockets=1,cores=1 \
        -net tap,ifname=tap0,script=no,downscript=no -net nic,model=virtio \
        -monitor stdio \
        -vga qxl -global qxl-vga.vram_size=67108864 \
        -spice port=$1,ipv4,disable-ticketing \
        -drive file=$2,if=none,id=drive-virtio-disk0,format=qcow2,cache=none,werror=stop,rerror=stop,aio=threads \
        -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x7,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 \
        -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x8
    }

    for i in `seq 11 30`
    do
        startvm 70$i hda-$i.qcow2
    done

测量：

.. code::

    iostat -cdmx 1|tee 20-xp.iostat-cdm.out

数据预处理，我们只需要读写速度（MB/s）、读写请求（q/s）、CPU利用（%user,%sys）。

.. code::

    awk 'BEGIN {print "cpu usage\n";i=0};$1 ~ /[0-9]/ {print i,$1+$3;i+=1;}' 20-xp.iostat-cdm.out > 20-xp.iostat-cdm-cpu.out
    awk 'BEGIN {print "sda info\nTime IOPS MBps";i=0};$1 ~ /^sda/ {iops=$4+$5;iombps=$6+$7;print i,iops,iombps;i+=1;}' 20-xp.iostat-cdm.out > 20-xp.iostat-cdm-sda.out
    awk 'BEGIN {print "sdb info\nTime IOPS MBps";i=0};$1 ~ /^sdb/ {iops=$4+$5;iombps=$6+$7;print i,iops,iombps;i+=1;}' 20-xp.iostat-cdm.out > 20-xp.iostat-cdm-sdb.out

可视化示例：

.. code::

    #!/usr/bin/env python
    import numpy as np
    import matplotlib.pyplot as plt

    f_c = file('20-xp.iostat-cdm-sda.out').readlines()
    c = np.array(map(str.split,f_c[2:]),dtype='float')

    plt.plot(c[:,0],c[:,1],label="$IO Requests/s$", color="red", linewidth=2)
    plt.plot(c[:,0],c[:,2],label="$IO MB/s$", color="blue", linewidth=2)
    plt.legend()

---------------------------
2.1. WD 15K SAS启动win7实验
---------------------------

-------------------------------
2.2. Intel 120G SSD启动win7实验
-------------------------------

3. 私有云中处理启动风暴的常用方法
=================================

-------------
3.1. 启动排队
-------------

--------------
3.2. SSD一站式
--------------

-----------------------------------------
3.3. 引入二级实例存储池改善常驻桌面IO分布
-----------------------------------------

-----------------------------------------
3.4. 引入二级快照存储池改善临时桌面IO分布
-----------------------------------------

---------------
3.5. bcache
---------------

---------------
3.6. fscache
---------------

4. 总结 
=======
