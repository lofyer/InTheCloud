======
附录一
======

--------------------------
OpenStack/Neutron快速搭建
--------------------------

RDO
----

使用 `RDO <http://openstack.redhat.com/Main_Page>`_ 来部署OpenStack。

.. note:: **安装说明**

    在RedHat系列（CentOS）系统上部署，将selinux置为permissive；禁用NetworkManager，启用network服务，详细配置请参考以前章节。

.. code::

    sudo yum update -y
    sudo yum install -y http://rdo.fedorapeople.org/rdo-release.rpm
    sudo yum install -y openstack-packstack
    packstack --allinone

请耐心等待，以上过程预计花费一到两小时。

有关此次部署的详细信息，在以下位置可以看到：

----------------
SDN学习/mininet
----------------

-----------------
常见性能测量工具
-----------------

------------
常用运维工具
------------

nagios
-------

foreman
--------

chef 自动化部署
----------------
