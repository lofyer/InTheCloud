=================
附录二 公有云参考
=================

-----------------------
虚拟机占用主机资源隔离
-----------------------

网络
-----

1. 使用 **VLAN** 、 **openvswitch** 进行隔离或限制。

2. 使用tc（Traffic Controller）命令进行速率的限制，许多虚拟化平台使用的都是它。

****

CPU
-----

1. 使用CPU Pin可以将特定虚拟核固定在指定物理核上。线程当作核来使用的话，每一个虚拟机使用一个核，之间互不干涉。

2. 使用Linux的 `Control Group <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/cgroups>`_ 来限制虚拟机对宿主机CPU的用度。

磁盘IO
-------

1. 更改内核磁盘IO调度方式：noop，cfq，deadline直接选择，参考 `Linux Doc. <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/block>`_ 。

2. 使用Linux的 `Control Group <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/cgroups>`_ 来限制虚拟机对宿主机设备的访问。

    使能cgroup，假如没有启动的话。

    .. code::
 
        # mount tmpfs cgroup_root /sys/fs/cgroup
        # mkdir /sys/fs/cgroup/blkio
        # mount -t cgroup -o blkio none /sys/fs/cgroup/blkio

    创建一个1 Mbps的IO限制组，X:Y 为MAJOR:MINOR。

    .. code::
 
        # lsblk
        # mkdir -p /sys/fs/cgroup/blkio/limit1M/
        # echo "8:0  1048576" > /sys/fs/cgroup/blkio/limit1M/blkio.throttle.write_bps_device

    将虚拟机进程附加到限制组。

    .. code::
 
        # echo $VM_PID > /sys/fs/cgroup/blkio/limit1M/tasks

    目前没有删除task功能，只能将它移到根组，或者是删除此组。

    .. code::

        # echo $VM_PID > /sys/fs/cgroup/blkio/tasks

3. 更改qemu drive cache。

设备
-----

使用Linux的 `Control Group <https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/cgroups>`_ 来限制虚拟机对宿主机设备的访问。

---------------
资源用度、计费
---------------

系统报告主要是在scaling（测量）、charging（计费）时使用。

在测量时可以使用平台提供的API、测量组件，综合利用nagios/Icinga，使用Django快速开发。

oVirt可以参考ovirt-reports，OpenStack参考其Ceilometer。

计费模块OpenStack可参考新浪云的 `dough项目 <https://github.com/sinacloud/dough>`_ 。

--------------------------
DeltaCloud/Libcloud混合云
--------------------------

**DeltaCloud支持：**

arubacloud

azure

ec2

rackspace

terremark

openstack

fgcp

eucalyptus

digitalocean

sbc

mock

condor

rhevm

google

opennebula

vsphere

gogrid

rimuhosting

**Libcloud支持：**

biquo

PCextreme

Azure Virtual machines

Bluebox Blocks

Brightbox

CloudFrames

CloudSigma (API v2.0)

CloudStack

DigitalOcean    

Dreamhost

Amazon EC2  

Enomaly Elastic Computing Platform

ElasticHosts

Eucalyptus   

Exoscale    

Gandi

Google Compute Engine   

GoGrid

HostVirtual

HP Public Cloud (Helion)    

IBM SmartCloud Enterprise   

Ikoula  

Joyent

Kili Public Cloud   

KTUCloud

Libvirt 

Linode

NephoScale

Nimbus  

Ninefold

OpenNebula (v3.8)

OpenStack   

Opsource   

Outscale INC    

Outscale SAS    

ProfitBricks

Rackspace Cloud

RimuHosting  

ServerLove  

skalicloud 

SoftLayer 

vCloud   

VCL     

vCloud 

Voxel VoxCLOUD      

vps.net   

VMware vSphere  

Vultr

DeltaCloud示例
--------------

Libcloud示例
--------------

----------------
SDN学习/mininet
----------------

SDN广泛用在内容加速、虚拟网络、监控等领域。

关于SDN有许多学习工具： `mininet <http://mininet.org>`_ 、 `POX <https://github.com/noxrepo/pox>`_  、 `Netwrok Heresy Blog <http://networkheresy.com/>`_ 。

学习视频： `Coursera SDN <https://www.coursera.org/course/sdn1>`_ 。
