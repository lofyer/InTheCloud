===================================
虚拟化中存储配置典型场景－启动风暴
===================================

桌面私有云相较于一般公有云，有着虚拟机启动密度大、服务器资源冗余性低等特点，本文尝试模拟桌面云中的常见场景来说明启动风暴问题的解决方法，并在现有方法的基础上提出一些改进措施，对公有云也有一定参考价值。

1. 常见虚拟化平台的配置介绍
===========================

磁盘分配方式

现有比较成熟的开源虚拟化/云计算平台，包括OpenStack、CloudStack和oVirt等，都有比较清晰的模板和实例概念，同时也有模板与实例镜像存储位置相关的存储域配置，为了区分虚拟机中的模板与实例，我们下文都默认：模板即原始虚拟机，所有新实例的创建都依赖于它；实例即依赖模板创建的虚拟机，是具有运行生命周期的虚拟机通称。

在虚拟化平台中，一般有两种新建实例的方式：克隆与增量。

克隆即完整克隆模板配置与磁盘，磁盘的创建方式一般为cp或者qemu-img convert。如此创建的实例，其磁盘与模板磁盘相对独立，在服务器存储上拥有各自的扇区位置，所以在读写操作时受机械盘磁头引起的小区域并发问题影响较小，缺点是创建时需要消耗一定时间，不能满足秒级创建的需求。

增量即新建虚拟机只复制模板配置信息，但其磁盘文件通过qemu-img的backing file方法创建，基于backing file创建的磁盘不能是raw格式，命令如下：

.. code::

    qemu-img create -b hda.qcow2 -f qcow2 hda-new.qcow2

如此创建的磁盘对模板磁盘的依赖较大，非增量文件的读操作（比如启动系统）绝大部分在模板磁盘上进行，所以在传统机械盘上多个实例的并发启动风暴更容易在此种格式的磁盘上发生。

集群调度策略

实例的启动在虚拟化平台中的一般调度如下：

.. figure:: ../images/schedule.png
    :align: center

    图1-1

从中我们可以看到，一个实例在真正启动之前会根据一定策略被安排到某集群的某一主机上，而且根据集群状态可以动态迁移至其他主机，在本文中我们不展开迁移的情况。

我们在实现调度策略的时候主要考量以下几点：

- 集群的CPU类型（Family）、集群负载状态；

- 主机运行虚拟机数量、主机历史健康度；

- 主机CPU用度、网络用度、内存用度、存储I/O用度，以及它们的超过一定阈值的时间；

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

上图中的模板磁盘和实例磁盘在同一文件系统上，oVirt使用使用了hardlink链接了模板d25c4d22-15c1-4f3c-8aef-2ed74deb4723下的磁盘2bf1ba4c-3fb5-4e25-9368-cc96724b1a63和实例6d7e74d-76fc-48c6-b3c6-3c506b98e73e下的2bf1ba4c-3fb5-4e25-9368-cc96724b1a63。

.. note:: 在同一文件系统上使用hardlink的而非softlink的原因

    1. 节省inode使用量
    2. 由于省去了访问softlink的inode这一步，直接访问原文件的inode会带来性能上的稍许提升
    3. 改变源文件路径不会影响hardlink文件的访问
    4. 冗余性及安全性考虑，只有删除了inode上的所有引用，此inode才会被删除。

.. note:: 

    ids - used by sanlock - each host maintain a lockspace on each domain, allowing it to acquire resources on the leases volume.
    leases - used for acquiring resources. For example, the spm node acquire a resource on this volume
    inbox - every host can write message to the spm node using this volume
    outbox - the spm write messages to other hosts on this volume
    master - this volume is mounted on the spm node for storing stuff related to the master storage domain
    metadata - used to store storage domain metadata

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

此次实验的目的为考察多台虚拟机同时启动对磁盘I/O的负载，不考虑qcow2格式与raw格式的影响，统一适用qcow2格式。所有的虚拟机均使用VirtIO接口，qcow2磁盘，backing_file格式也为qcow2，Windows XP 32位操作系统，无任何附加软件。为减少XP系统启动后对快照磁盘的额外操作，我们所有的XP实例都运行了一个小时后再关机进行测试。

服务器配置为双路X5670 @ 2.93GHz，64GB内存，一块Intel 480G企业级SSD，一块WD 1T企业级机械硬盘，操作系统为CentOS 7.1。

模板配置：

*file: base_xp.sh*

.. code::

    #!/bin/bash
    /usr/libexec/qemu-kvm -no-user-config -nodefaults \
    -m 1024M -cpu host -smp 1,sockets=1,cores=1 \
    -net user \
    -monitor stdio -vga qxl -global qxl-vga.vram_size=67108864 \
    -spice port=7001,ipv4,disable-ticketing \
    -drive file=hda.qcow2,if=none,id=drive-virtio-disk0,format=qcow2,cache=none,werror=stop,rerror=stop,aio=threads \
    -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x7,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 \
    -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x8

实验脚本：

创建20个以hda.qcow2为backing file的磁盘，用于实例。

*file: create-imgs.sh*

.. code::

    #!/bin/bash
    for i in `seq 11 30`
    do
        qemu-img create -f qcow2 -b hda.qcow2 hda-$i.qcow2
    done

一次性启动20台实例。

*file: start-vms.sh*

.. code::

    #!/bin/bash
    function startvm {
        /usr/libexec/qemu-kvm -no-user-config -nodefaults \
        -m 1024M -cpu host -smp 1,sockets=1,cores=1 \
        -net user \
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

测量，采样频率为 1 Hz：

.. code::

    iostat -cdmx 1|tee 20-xp.iostat-cdm.out

数据预处理，我们只需要总读写速度（MB/s）、总读写请求（Requests/s）、CPU利用（%user,%sys）。

.. code::

    awk 'BEGIN {print "cpu usage\n";i=0};$1 ~ /[0-9]/ {print i,$1+$3;i+=1;}' 20-xp.iostat-cdm.out > 20-xp.iostat-cdm-cpu.out
    awk 'BEGIN {print "sdb info\nTime IOPS RMBps WMBps";i=0};$1 ~ /^sda/ {iops=$4+$5;print i,iops,$6,$7;i+=1;}' 20-xp.iostat-cdm.out > 20-xp.iostat-cdm-sdb.out
    awk 'BEGIN {print "sdb info\nTime IOPS RMBps WMBps";i=0};$1 ~ /^sdb/ {iops=$4+$5;print i,iops,$6,$7;i+=1;}' 20-xp.iostat-cdm.out > 20-xp.iostat-cdm-sdb.out

可视化示例，由于请求数比速度高很多倍，为方便数据显示我们将请求数除以10：

.. code::

    #!/usr/bin/env python
    import numpy as np
    import matplotlib.pyplot as plt

    f_c = file('20-xp.iostat-cdm-sda.out').readlines()
    c = np.array(map(str.split,f_c[2:]),dtype='float')

    plt.plot(c[:,0],c[:,1]/10,label="IO Requests/s", color="red", linewidth=2)
    plt.plot(c[:,0],c[:,2],label="Read MB/s", color="blue", linewidth=2)
    plt.plot(c[:,0],c[:,3],label="Write MB/s", color="blue", linewidth=2)
    plt.legend()
    plt.show()

---------------------------
2.1. WD 15K 启动xp实验
---------------------------

所有实例于WD 1T硬盘上启动的结果如下：

启动单台XP的CPU及I/O用度，系统在第8秒左右进入桌面：

.. figure:: ../images/1-xp-sata-cpu.png
    :align: center

    图2-1

.. figure:: ../images/1-xp-sata-io.png
    :align: center

    图2-2

启动20台XP的CPU及I/O用度，全部系统在第300秒左右进入桌面：

.. figure:: ../images/20-xp-sata-cpu.png
    :align: center

    图2-3

.. figure:: ../images/20-xp-sata-io.png
    :align: center

    图2-4

-------------------------------
2.2. Intel 120G SSD启动xp实验
-------------------------------

所有实例于Intel 480G SSD硬盘上启动的结果如下：

启动单台XP的CPU及I/O用度，系统在第6秒左右进入桌面：

.. figure:: ../images/1-xp-ssd-cpu.png
    :align: center

    图2-5

.. figure:: ../images/1-xp-ssd-io.png
    :align: center

    图2-6

启动20台XP的CPU及I/O用度，全部系统在第35秒左右进入桌面：

.. figure:: ../images/20-xp-ssd-cpu.png
    :align: center

    图2-7

.. figure:: ../images/20-xp-ssd-io.png
    :align: center

    图2-8

---------
2.3. 小结
---------

从上图中我们可以总结出XP自启动时会有大量的读请求以及大量数据读出，写请求相对少很多，同时CPU用度随着I/O请求量上升；而在进入桌面时刻左右会有少量读请求和少量数据读出，有较多的写I/O请求和少量数据写入，同时CPU用度相对前一阶段较为平缓。

一般我们称启动时的前段时间造成的I/O风暴为“启动风暴”，进入桌面时的称之为“登录风暴”。

3. 私有云中处理启动风暴的常用方法
=================================

这里介绍一下我们在私有云中处理启动风暴的方法，其中一些可能并不适用于所有场景，但相信仍有些参考及折腾价值。

-------------
3.1. 启动排队
-------------

启动排队是一种改善启动风暴比较常见的做法，它在虚拟化平台分配实例到某台服务器后执行，即我们不考虑实例在多台服务器上的调度，其基本原理如下。

.. image:: ../images/storage-01.png
    :align: center

实例在启动时都会被加进固定长度为m+n的队列末端，其中m为实例运行时磁盘所在存储的最优实例启动并发数，我们称之为启动队列。n为等待启动的实例数目，我们称之为为待启动队列。我们将实例自启动到其iops降至其空载运行水平视为完全启动，此段时间为t。当队列m中有实例完全启动后，将其从队列中移除，队列n中的首端的实例加入至m队列尾端并开始启动。

使用启动排队时，有以下几点设计原则需要注意：

1. t和m的值可以根据实例运行的所处环境自动动态调整（后台自动测量服务器、存储服务水平），管理员也可手动调整。

2. 实例会根据其优先级插队至n的首端，但不可插队至m队列中。

3. 当队列n满时，可以选择拒绝启动或者等待资源以启动实例；某些设计中也可将m与n合并为一个启动队列。

此种方式能够比较高效地利用服务器和存储资源，而且对已启动实例的附带影响又较少。

-----------------
3.2. 存储分层选择
-----------------

启动排队的方式是从策略上解决问题，接下来我们从实例磁盘存储位置的选择上进一步降低启动风暴带来的影响。

模板磁盘、实例磁盘、增量磁盘以及“无状态磁盘”的所在存储位置的性能能够直接影响实例的启动过程。

3.2.1. 统一高速存储改善整体I/O分布
----------------------------------

从第二节的实验中我们可以看出，单台XP实例启动的前10秒是IOPS消耗最大的时期。在启动风暴来临之前，我们只要准备好IOPS充足且适当的存储设备，再适量调整我们的I/O调度算法即可一定程度上缓冲第一波风暴，而后的登录风暴便也便不是问题了。

假设我们现在有IOPS充足且适当的存储设备，我们将模板、实例、快照磁盘全部放置到这台设备上，从而比较“不负责任”地缓解这个问题。

为什么说“不负责任”呢？

因为虽然一股脑的全部上SSD会改善I/O情况而减缓启动风暴带来的影响，但成本上对比根据实例实际读写I/O分配合理规划存储的方案，就显得些许浪费了。

当然，不论是后端存储还是前端服务器OS，使用响应时间短、读写带宽大的磁盘总是有益的。现在制造SSD的技术不断提高，成本、寿命、速度、容量等与机械硬盘相比优势也越来越突出，因此我相信在虚拟化桌面领域中SSD会代替机械硬盘会成为主流。

3.2.2. 增量磁盘高速存储池改善实例I/O分布
----------------------------------------

从第二节的实验中我们可以看出，实例启动时会有大量的读请求和相对较少的写请求，而我们的实例即是从模板母盘读取数据，并将CED（创建、修改、删除）数据写入增量盘。那么，我们不妨将模板磁盘放置在一般的SSD（MLC、TLC）存储上满足读请求要求，增量磁盘放置在成本相对低廉的SATA盘RAID阵列存储上来满足平稳且分散的写请求。这样以来，我们就让它们各取所需，从而妥当地改善启动风暴带来的影响。

.. figure:: ../images/layer-1.png
    :align: center

    图3-1 增量磁盘读写请求

.. figure:: ../images/20-xp-ssd-sata.png
    :align: center

    图3-2 模板位于SSD，实例位于SATA的读写请求

当然，随着桌面中安装的自启动软件和服务越来越多，而这些后来者都写入了我们的增量盘中，所以增量盘的读请求数在后期会有一定量的上升，从而拖慢整体的读请求水平。根据安装软件的“重度”，我们可以选择提前在模板中安装比较“重”的软件，尽量使增量磁盘中的静态数据（比如文档、媒体文件等）占比扩大，这样就可以使后期的读请求时间较为分散，保证系统的启动速度不受太多影响。

3.2.3. 无状态实例磁盘高速存储池改善实例I/O分布
----------------------------------------------

由于无状态桌面在桌面云中的重要位置，我们在这里要特别说明一下，看看如何放置在存储中放置它们的磁盘，从而达到既省钱又办了实事儿的效果。

首先，模板会承受启动时的大量读请求，那么我们肯定是要把模板放置到SSD这样的高IOPS存储了。

然后，我们再看一下“无状态”磁盘，即将实例磁盘作为backing_file的磁盘。在桌面云中它的生命周期短，生成频率高，并且用户使用的软件和文档都已经被“模板化”，后来很少对系统有大量写的情况产生。所以，它承受的读写请求量较低，但是它的每一次生成和删除对所在物理存储又有频繁的重复写入操作。考虑到SSD的擦除次数有限，我们最好将它放置在机械磁盘的存储中。

最后，从模板创建的实例磁盘也会承受大量的读请求，但是接下来我们要考虑两种状况：当实例磁盘中有较多的拖慢启动时间的软件和服务时，为保证启动时IOPS不会成为瓶颈，我们最好把它也放置到SSD中；当实例磁盘中静态数据占比较多时，我们可以优先考虑将其放置在机械盘的阵列存储中，并且可以与“无状态”磁盘同一文件系统中。

.. figure:: ../images/layer-2.png
    :align: center

    图3-3 无状态实例的读写请求

---------------------------------
3.3. 其他提升桌面云存储性能的方式
---------------------------------

3.3.1. fscache
--------------

FS-Cache 是一种将通过网络获取的数据缓存到本地常驻存储以加速本地应用访问，从而减少网络网络流量的技术。

.. figure:: ../images/fs-cache.png
    :align: center

    图3-4 FS-Cache原理图

FS-Cache在设计之初，就尽量保持对管理员和用户透明的特性。不同于Solaris系统的cachefs，FS-Cache允许服务器端的文件系统直接与客户端的本地cache进行交互，而不需要额外挂载文件系统。比如在NFS上使用FS-Cache时，我们只需要在挂载选项中加一个参数就可以启用它。

FS-Cache在不改变网络文件系统基本操作的前提下，提供了文件系统中一个常驻cache用于数据的暂存。比如，一个NFS客户端即使之前配置了FS-Cache并且有一部分数据被cache，关闭后它后仍然可以挂载NFS并且使用被cache的数据部分。FS-Cache也可隐藏掉客户端文件系统驱动的所有I/O错误。在配置FS-Cache的时候，我们需要一个cache的后端，这个后端的文件系统需要支持bmap和扩展属性。

但FS-Cache不能cache任意网络文件系统，它需要的共享文件系统驱动需要能够与FS-Cache交互、存储数据、建立和验证元数据。FS-Cache通过cache后端文件系统中数据的键索引和一致性检查来保证数据的持久特性，即数据的有效性校验。

FS-Cache在私有云中一般适用于教学、办公等模板相同的批量桌面，不适用于存储模板大多相异的虚拟服务器。

3.3.2. bcache
-------------

bcache是Linux内核块设备层的cache模块，它可以让一块或多块SSD用作为普通硬盘的cache，有点类似“混合硬盘”。

bcache有点类似ZFS中的L2Arc，但是除去当作write through的cache外，它也可当做write back的cache，并且文件系统对它来说是透明的。在使用上bcache可以很方便地启用，并且不需要额外设置就可以满足我们大部分需求。但是，它不会去cache顺序I/O，只会cache SSD擅长的随机I/O，这点特性让它的应用范围一定程度上有所限制了。

还有一点比较有用的特性，即掉电后它的数据不会丢失，这样它就有点像一个带电池的raid控制器了。关于这点，社区和商业公司已经有做了很多工作，所以我们放心用就好了。

但由于它是块设备级别的cache，它对单个服务器的提升比较大，但是对于跑在外接存储设备上的虚拟机就没有什么效果了。所以除去商业存储设备的选择外，bcache和fscache让我们多了一个自建存储的理由。

3.3.3. SSD PCI-E 卡
-------------------

PCI Express SSD产品通常采用特殊的驱动器通过PCI总线进行直接存储器访问(Direct Memory Access，简称DMA)，而非只是将闪存或DRAM内存封装成SCSI连接的硬盘驱动器。这是一种比较革命性的改变，它使得随机读写性能相比其他存储设备都有质的提升。

存储设备连接服务器有以下几种常见方式：

- PCI-E总线连接RAID控制器，再连接SAS或SATA硬盘。

- 通过PCI-E总线连接HBA卡，再连接到磁盘阵列。

- SSD挂载到PCI-E SSD卡，再挂到PCIe插槽。

我们可以看到CPU通过PCI-E SSD卡提供的短路径来访问SSD，结合flash的高速读写性能，极大的提升存储性能，突破存储I/O瓶颈。

而目前在桌面云中，这是一种奢侈的解决方案。

4. 总结 
=======

从模拟试验中我们可以看到服务器运行一定数量（20-50）的虚拟桌面下的存储I/O负载，了解到桌面在启动时的读写请求分布状况，而后我们再根据实验结果提出桌面功能的简单优化策略。

总之，目前要完全避免启动风暴、登录风暴需要较大的成本投入，而在私有桌面云中，我们最好合理规划存储以减少风暴带来的影响。

对于读者来说，找到合适的就好，毕竟架构没有完美的，都是在经验中进化的。
