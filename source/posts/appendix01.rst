===========================================
附录一 OpenStack概念、部署、与高级网络应用
===========================================

首先在这里我会使用RDO快速部署一个具有基本功能的OpenStack环境，如果你想要更完整的部署（比如Heat、Trove组件），可以参考 `官方文档 <http://docs.openstack.org/icehouse/install-guide/install/yum/content>`_ 。

也可以使用Mirantis来进行快速部署，参考 `Mirantis <https://software.mirantis.com/>`_ ；StackOps部署，参考 `StackOps <https://www.stackops.com>`_ 。

如果不想安装任何东西，只是在线使用的话，可以访问 http://trystack.org/ 。

要学习更多关于OpenStack的内容，可以参考 `陈沙克的日志 <http://www.chenshake.com/cloud-computing/>`_ 。

API使用请参考http://developer.openstack.org/api-ref.html 以及 http://docs.openstack.org/developer/openstack-projects.html 。

关于在ubuntu/debian上部署OpenStack请参考 `Server-World <http://www.server-world.info/en/>`_ 。

---------------
OpenStack 部署
---------------

在开始之前需要将这些关键组件关系理清。

- nova：提供compute服务，即保证虚拟机运行的必须服务之一，虚拟机运行于所有提供compute服务的主机之上。

- neutron：提供network服务，同时提供Open vSwitch、L3、DHCP代理服务。

- cinder：提供块存储服务，可配合swift使用。

- swift：提供对象存储服务，目录与文件皆视为对象，外部可以方便取用。

- glance：提供镜像管理服务。

- ceilometer：主要功能是监测、收集用户对资源的使用情况，以方便计费、报警等。

- heat：orchestration模块驱动服务，即从模板根据需求更改配置创建新虚拟机。

- keystone：身份认证服务。

- trove：数据库服务。

- sahara：Hadoop模块。

- ironic：提供从模板创建新应用程序。

Mirantis Fuel 部署
===================

RDO 快速部署
=============

使用 `RDO <http://openstack.redhat.com/Main_Page>`_ 来部署OpenStack。

.. note:: **安装说明**

    在一台安装有RedHat系列（CentOS）系统上部署，将selinux置为permissive；禁用NetworkManager，启用network服务，详细配置请参考以前章节。

    如果安装失败，请查看你的CentOS或者Fedora是否为最新发行版。笔者在写时使用的是CentOS6，目前为CentOS7。

.. code::

    # yum install -y http://rdo.fedorapeople.org/rdo-release.rpm
    # yum install -y epel-release
    # yum update -y
    # yum install -y openstack-packstack
    # packstack --allinone

请耐心等待，以上过程预计花费一到两小时。有关此次部署的详细信息，在安装完成后可以看到：

.. code::

     **** Installation completed successfully ******

     Additional information:
     * A new answerfile was created in: /root/packstack-answers-20140730-110621.txt
     * Time synchronization installation was skipped. Please note that unsynchronized time on server instances might be problem for some OpenStack components.
     * File /root/keystonerc_admin has been created on OpenStack client host 192.168.2.160. To use the command line tools you need to source the file.
     * To access the OpenStack Dashboard browse to http://192.168.2.160/dashboard .
     Please, find your login credentials stored in the keystonerc_admin in your home directory.
     * To use Nagios, browse to http://192.168.2.160/nagios username: nagiosadmin, password: ea65dc070f034776
     * Because of the kernel update the host 192.168.2.160 requires reboot.
     * The installation log file is available at: /var/tmp/packstack/20140730-110621-upxlZJ/openstack-setup.log
     * The generated manifests are available at: /var/tmp/packstack/20140730-110621-upxlZJ/manifests

可修改 */root/packstack-answers-20140730-110621.txt* 内容以 `增加计算节点 <http://openstack.redhat.com/Adding_a_compute_node>`_ ；同理可增加网络节点（待实验）。

.. note::

    假如更换了admin/demo/services的密码，不要忘记在此配置文件中将其修改为新密码。
    # packstack --answer-file=/root/packstack-answers-20140730-110621.txt

分步详细部署
=============

CentOS 7以及Ubuntu等发行版部署OpenStack的过程基本一致，在此以CentOS 7示例。

机器准备
---------

控制节点：controller0

计算节点：controller0，compute0

网络节点：neutron0

存储节点：stor0，stor1，stor2，swift0

Heat节点：heat0

每台机器首先将selinux设置为permissive或者disable、打开ssh服务、禁用防火墙、关闭NetworkManager服务、打开network服务并配置IP。

配置KeyStone
-------------

初始化数据库等
~~~~~~~~~~~~~~~

在控制节点controller0，配置源、数据库、RabbitMQ、Memcached。

.. code::

    [root@controller0 ~]# yum -y install http://repos.fedorapeople.org/repos/openstack/openstack-kilo/rdo-release-kilo.rpm epel-release
    [root@controller0 ~]# yum install -y galera mariadb-galera-server rabbitmq-server memcached
    [root@controller0 ~]# systemctl start mariadb
    [root@controller0 ~]# systemctl enable mariadb
    [root@controller0 ~]# systemctl start rabbitmq-server
    [root@controller0 ~]# systemctl enable rabbitmq-server
    [root@controller0 ~]# systemctl start memcached
    [root@controller0 ~]# systemctl enable memcached

    # 初始化mysql
    [root@controller0 ~]# mysql_secure_installation 
    /usr/bin/mysql_secure_installation: line 379: find_mysql_client: command not found

    NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
          SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

          In order to log into MariaDB to secure it, we'll need the current
          password for the root user.  If you've just installed MariaDB, and
          you haven't set the root password yet, the password will be blank,
          so you should just press enter here.

          Enter current password for root (enter for none):
          OK, successfully used password, moving on...

          Setting the root password ensures that nobody can log into the MariaDB
          root user without the proper authorisation.

          # set root password
          Set root password? [Y/n] y
          New password:
          Re-enter new password:
          Password updated successfully!
          Reloading privilege tables..
           ... Success!

        By default, a MariaDB installation has an anonymous user, allowing anyone
        to log into MariaDB without having to have a user account created for
        them.  This is intended only for testing, and to make the installation
        go a bit smoother.  You should remove them before moving into a
        production environment.
        # remove anonymous users
        Remove anonymous users? [Y/n] y
         ... Success!

      Normally, root should only be allowed to connect from 'localhost'.  This
      ensures that someone cannot guess at the root password from the network.

      # disallow root login remotely
      Disallow root login remotely? [Y/n] y
       ... Success!

    By default, MariaDB comes with a database named 'test' that anyone can
    access.  This is also intended only for testing, and should be removed
    before moving into a production environment.

    # remove test database
    Remove test database and access to it? [Y/n] y
     - Dropping test database...
        ... Success!
         - Removing privileges on test database...
            ... Success!

         Reloading the privilege tables will ensure that all changes made so far
         will take effect immediately.

         # reload privilege tables
         Reload privilege tables now? [Y/n] y
          ... Success!

       Cleaning up...

       All done!  If you've completed all of the above steps, your MariaDB
       installation should now be secure.

       Thanks for using MariaDB!

    # 重置rabbitmq密码
    [root@controller0 ~]# rabbitmqctl change_password guest password 

初始化Keystone
~~~~~~~~~~~~~~~

.. code::

    [root@controller0 ~]# yum install -y openstack-keystone openstack-utils
    [root@controller0 ~]# mysql -u root -p 
    Enter password:
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 10
    Server version: 5.5.40-MariaDB-wsrep MariaDB Server, wsrep_25.11.r4026

    Copyright (c) 2000, 2014, Oracle, Monty Program Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> create database keystone;
    Query OK, 1 row affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on keystone.* to keystone@'localhost' identified by 'password';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on keystone.* to keystone@'%' identified by 'password';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> flush privileges;
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> exit
    Bye

    [root@controller0 ~]# vi /etc/keystone/keystone.conf

    # line 13: 修改admin密码为admin
    admin_token=admin

    # line 633: 数据库连接信息
    connection=mysql://keystone:password@10.0.0.30/keystone

    # line 1434: token格式
    token_format=PKI

    # line 1440: 证书信息
    certfile=/etc/keystone/ssl/certs/signing_cert.pem
    keyfile=/etc/keystone/ssl/private/signing_key.pem
    ca_certs=/etc/keystone/ssl/certs/ca.pem
    ca_key=/etc/keystone/ssl/private/cakey.pem
    key_size=2048
    valid_days=3650
    cert_subject=/C=CN/ST=Di/L=Jiang/O=InTheCloud/CN=controller0.inthecloud

    [root@controller0 ~]# keystone-manage pki_setup --keystone-user keystone --keystone-group keystone 
    [root@controller0 ~]# keystone-manage db_sync 
    [root@controller0 ~]# systemctl start openstack-keystone 
    [root@controller0 ~]# systemctl enable openstack-keystone 
    * remove the log file if keystone will not start.
    [root@controller0 ~]# rm /var/log/keystone/keystone.log 

配置Glance
-----------

配置Nova
---------

添加镜像
~~~~~~~~~

配置Nova Networking（可选）
~~~~~~~~~~~~~~~~~~~~~~~~~~~

启动实例
~~~~~~~~~~

配置Horizon
------------

添加计算节点
------------

配置Neutron（推荐）
-------------------

配置Cinder存储
---------------

配置Swift
----------

配置Heat（可选）
----------------

配置Ceilometer
---------------

配置Sahara（可选）
------------------

配置Ironic（可选）
------------------

----------
使用示例
----------

基本操作
==========

一些常用操作。

添加镜像
----------

以admin或者demo用户身份登录dashboard后，选择“镜像”，上传ISO。

.. image:: ../images/apx01-01.png
    :align: center
    

从ISO安装新实例
----------------

在“实例”选项卡中，选择“添加实例”，并从现有镜像启动。

.. image:: ../images/apx01-02.png
    :align: center

与owncloud集成
===============

1. 创建一个指定region的endpoint于swift服务中

    .. code::

        # source ./keystone_admin
        # keystone endpoint-create --service swift --region swift_region \
          --publicurl "http://192.168.2.160:8080/v1/AUTH_7d11dd5a3f3544149e8b6a9799a2aa48/oc"

    其中的publicurl可以从container的详细信息中查看。

2. 使用owncloud的第三方app——external storage，如下进行填写

    - 目录名称：显示在owncloud中的目录名称。

    - user：project用户名。

    - bucket：容器名。

    - region：上一步指定的region。

    - key：用户密码。

    - tenant：project名。

    - password：用户密码。

    - service_name：服务名，即swift。

    - url：使用keystone认证的url，即http://192.168.2.160:5000/v2.0 。

    - timeout：超时时长，可不填。

    .. image:: ../images/apx01-12.jpg
        :align: center

oVirt使用Glance与Neutron服务
=============================

oVirt自3.3版本起，便可以添加外部组件，比如Foreman、OpenStack的网络或镜像服务。

在添加OpenStack相关组件之前，oVirt管理端需要配置OpenStack的KeyStone URL：

.. code::

    # engine-config --set KeystoneAuthUrl=http://192.168.2.160:35357/v2.0
    # service ovirt-engine restart

添加OpenStack镜像服务Glance至oVirt
-----------------------------------

1. 在OpenStack的控制台中，添加一个新镜像，比如my_test_image，格式为raw。

.. image:: ../images/apx01-03.png
    :align: center

2. 在oVirt左边栏，选择External Provider添加OpenStack Image服务。

.. image:: ../images/apx01-04.png
    :align: center

.. note:: 认证选项

    用户名：glance

    密码：存于RDO配置文件中，形如 CONFIG_GLANCE_KS_PW=bf83b75a635843b4

    Tenant：services

3. 然后可以在oVirt的存储域中看到刚刚添加的Glance服务。

.. image:: ../images/apx01-05.png
    :align: center

Neutron
--------

.. image:: ../images/apx01-06.jpeg
    :align: center

可参考 `NeutronVirtualAppliance <http://www.ovirt.org/Features/NeutronVirtualAppliance>`_ 以及 `Overlay_Networks_with_Neutron_Integration <http://www.ovirt.org/Overlay_Networks_with_Neutron_Integration>`_ ，另外提供 `操作视频 <http://pan.baidu.com/s/1o6G61vG>`_ 。

1. 配置oVirt。
   
.. code::

    # engine-config --set OnlyRequiredNetworksMandatoryForVdsSelection=true
    # yum install vdsm-hook-openstacknet
    # service ovirt-engine restart

2. 如图添加Neutron组件。

.. image:: ../images/apx01-07.png
    :align: center

.. image:: ../images/apx01-08.png
    :align: center

.. note:: 认证选项

    用户名：neutron

    密码：存于RDO配置文件中，形如 CONFIG_NEUTRON_KS_PW=a16c52e3ea634324

    Tenant：services

    agent 配置相同

------------------
OpenStack常见问题
------------------

Q：管理界面Swift不能删除目录。

A：使用命令 swift delete public_container aaa/ 进行删除。

Q： Neutron 网络快速开始？

A：参考https://www.ustack.com/blog/neutron_intro/

Q：OpenStack组件间的通信是靠什么？

A：AMQP，比如RabbitMQ、Apache的ActiveMQ，部署时候可以选择，如果对这种消息传输工具有兴趣可以参考 `rabbitmq tutorial <http://www.rabbitmq.com/getstarted.html>`_ 以及 `各种有用的插件（web监视等） <http://www.rabbitmq.com/plugins.html>`_ 。

Q：Swift有什么好用的客户端么？

A：`python-swiftclient <https://github.com/openstack/python-swiftclient>`_ 、 `Gladient Cloud Desktop <http://www.gladient.com/>`_ 、 `Cloudberry <http://www.cloudberrylab.com/>`_ 、 `Cyberduck <http://cyberduck.ch/>`_ 、 `WebDrive <http://www.webdrive.com/>`_ 、 `S3 Browser <http://s3browser.com/>`_ 等。
