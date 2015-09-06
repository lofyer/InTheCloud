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

现在国内使用Mirantis部署的人已经越来越多了，而我们同样使用它来一次快速部署。精通了Mirantis就掌握了大规模集群的部署方式，不只是OpenStack。

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

添加计算节点
-------------

使用Neutron网络
---------------

分步详细部署
=============

CentOS 7以及Ubuntu等发行版部署OpenStack的过程基本一致，在此以CentOS 7示例。

机器准备
---------

控制节点：controller0(192.168.77.50)

计算节点：controller0(192.168.77.50)，compute0(192.168.77.51)

网络节点：neutron0(192.168.77.30)

存储节点：cinder0(192.168.77.60)，swift0(192.168.77.70)，swift-stor0(192.168.77.71)，swift-stor1(192.168.77.72)，swift-stor2(192.168.77.73)

Heat节点：heat0(192.168.77.80)

每台机器首先将selinux设置为permissive或者disable、打开ssh服务、禁用防火墙（可安装iptables-services服务，关闭firewalld，iptables -F后再service iptables save）、关闭NetworkManager服务、打开network服务并配置IP。

.. note:: bash_rc文件用例

    bash_rc文件比如os_bootstrap、admin_keystone，其中os_bootstrap用于创建基本keystone服务使用，admin_keystone为创建的admin用户。

.. note:: 认证服务

    OpenStack中的keystone服务负责绝大部分authentication的工作，其中属于service组的用户（nova、glance）也是基于keystone认证的，所以不要认为service中的服务仅仅是一个服务而已。

.. code::

    # cat os_bootstrap
    export SERVICE_TOKEN=admin
    export SERVICE_ENDPOINT=http://192.168.77.50:35357/v2.0/

    # cat admin_keystone
    export OS_USERNAME=admin
    export OS_PASSWORD=admin
    export OS_TENANT_NAME=admin
    export OS_AUTH_URL=http://localhost:35357/v2.0/
    export PS1='[\u@\h \W(keystone)]\$ '

初始化控制节点
---------------

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

          # 设置mysql的root密码
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

配置KeyStone
-------------

初始化Keystone
~~~~~~~~~~~~~~~

.. code::

    # 安装keystone
    [root@controller0 ~]# yum install -y openstack-keystone openstack-utils
    # 添加数据库
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

配置keystone
~~~~~~~~~~~~~

.. code::

    [root@controller0 ~]# vi /etc/keystone/keystone.conf

    # line 13:  超级管理员密码为admin，此密码仅供设置keystone，在生产环境中应该禁用
    admin_token=admin

    # line 418: database
    connection=mysql://keystone:password@localhost/keystone

    # line 1434: token格式
    # 可能不要
    token_format=PKI

    # line 1624: signing
    certfile=/etc/keystone/ssl/certs/signing_cert.pem
    keyfile=/etc/keystone/ssl/private/signing_key.pem
    ca_certs=/etc/keystone/ssl/certs/ca.pem
    ca_key=/etc/keystone/ssl/private/cakey.pem
    key_size=2048
    valid_days=3650
    cert_subject=/C=CN/ST=Di/L=Jiang/O=InTheCloud/CN=controller0.lofyer.org

    # 设置证书，同步数据库
    [root@controller0 ~]# keystone-manage pki_setup --keystone-user keystone --keystone-group keystone 
    [root@controller0 ~]# keystone-manage db_sync 
    # 删除日志文件并启动，否则可能因为log文件权限问题而报错
    [root@controller0 ~]# rm /var/log/keystone/keystone.log 
    [root@controller0 ~]# systemctl start openstack-keystone 
    [root@controller0 ~]# systemctl enable openstack-keystone 

添加用户、角色、服务与endpoint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

将超级管理员配置保存到文件，方便以后管理：

.. code::
    
    [root@controller0 ~]# cat os_bootstrap
    export SERVICE_TOKEN=admin
    export SERVICE_ENDPOINT=http://192.168.77.50:35357/v2.0/ 
    [root@controller0 ~]# source os_bootstrap

添加admin及service的tenant组：

.. code::

    [root@controller0 ~]# keystone tenant-create --name admin --description "Admin Tenant" --enabled true
      +-------------+----------------------------------+
      |   Property  |              Value               |
      +-------------+----------------------------------+
      | description |           Admin Tenant           |
      |   enabled   |               True               |
      |      id     | c0c4e7b797bb41798202b55872fba074 |
      |     name    |              admin               |
      +-------------+----------------------------------+

    [root@controller0 ~]# keystone tenant-create --name service --description "Service Tenant" --enabled true
      +-------------+----------------------------------+
      |   Property  |              Value               |
      +-------------+----------------------------------+
      | description |          Service Tenant          |
      |   enabled   |               True               |
      |      id     | 9acf83020ae34047b6f1e320c352ae44 |
      |     name    |             service              |
      +-------------+----------------------------------+

    [root@controller0 ~]# keystone tenant-list 
      +----------------------------------+---------+---------+
      |                id                |   name  | enabled |
      +----------------------------------+---------+---------+
      | c0c4e7b797bb41798202b55872fba074 |  admin  |   True  |
      | 9acf83020ae34047b6f1e320c352ae44 | service |   True  |
      +----------------------------------+---------+---------+

创建角色：

.. code::

    # 创建admin角色
    [root@controller0 ~]# keystone role-create --name admin 
      +----------+----------------------------------+
      | Property |              Value               |
      +----------+----------------------------------+
      |    id    | 95c4b8fb8d97424eb52a4e8a00a357e7 |
      |   name   |              admin               |
      +----------+----------------------------------+

    # 创建Member角色
    [root@controller0 ~]# keystone role-create --name Member 
      +----------+----------------------------------+
      | Property |              Value               |
      +----------+----------------------------------+
      |    id    | aa8c08c0ff63422881c7662472b173e6 |
      |   name   |              Member              |
      +----------+----------------------------------+
      
    [root@controller0 ~]# keystone role-list
      +----------------------------------+----------+
      |                id                |   name   |
      +----------------------------------+----------+
      | aa8c08c0ff63422881c7662472b173e6 |  Member  |
      | 9fe2ff9ee4384b1894a90878d3e92bab | _member_ |
      | 95c4b8fb8d97424eb52a4e8a00a357e7 |  admin   |
      +----------------------------------+----------+

添加用户并赋予角色：

.. code::

    # 添加admin用户至admin组，此处的密码仅仅是admin用户密码，与之前的admin_token可以不同
    [root@controller0 ~]# keystone user-create --tenant admin --name admin --pass admin --enabled true
      +----------+----------------------------------+
      | Property |              Value               |
      +----------+----------------------------------+
      |  email   |                                  |
      | enabled  |               True               |
      |    id    | cf11b4425218431991f095c2f58578a0 |
      |   name   |              admin               |
      | tenantId | c0c4e7b797bb41798202b55872fba074 |
      | username |              admin               |
      +----------+----------------------------------+
    # 赋予admin用户以admin角色
    [root@controller0 ~]# keystone user-role-add --user admin --tenant admin --role admin

    # 添加即将用到的glance、nova用户与服务
    [root@controller0 ~]# keystone user-create --tenant service --name glance --pass servicepassword --enabled true 
      +----------+----------------------------------+
      | Property |              Value               |
      +----------+----------------------------------+
      |  email   |                                  |
      | enabled  |               True               |
      |    id    | 2dcaa8929688442dbc1df30bee8921eb |
      |   name   |              glance              |
      | tenantId | 9acf83020ae34047b6f1e320c352ae44 |
      | username |              glance              |
      +----------+----------------------------------+
    [root@controller0 ~]# keystone user-role-add --user glance --tenant service --role admin

    [root@controller0 ~]# keystone user-create --tenant service --name nova --pass servicepassword --enabled true
      +----------+----------------------------------+
      | Property |              Value               |
      +----------+----------------------------------+
      |  email   |                                  |
      | enabled  |               True               |
      |    id    | 566fe34145af4390b0aadb906131a9e8 |
      |   name   |               nova               |
      | tenantId | 9acf83020ae34047b6f1e320c352ae44 |
      | username |               nova               |
      +----------+----------------------------------+
    [root@controller0 ~]# keystone user-role-add --user nova --tenant service --role admin

添加服务：

.. code::
    
    [root@controller0 ~]# keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
      +-------------+----------------------------------+
      |   Property  |              Value               |
      +-------------+----------------------------------+
      | description |    Keystone Identity Service     |
      |   enabled   |               True               |
      |      id     | b3ea5d31edce4c10b3b4c18359de0d09 |
      |     name    |             keystone             |
      |     type    |             identity             |
      +-------------+----------------------------------+

    [root@controller0 ~]# keystone service-create --name=glance --type=image --description="Glance Image Service" 
      +-------------+----------------------------------+
      |   Property  |              Value               |
      +-------------+----------------------------------+
      | description |       Glance Image Service       |
      |   enabled   |               True               |
      |      id     | 6afe8a067e2945fca023f85c7760ae53 |
      |     name    |              glance              |
      |     type    |              image               |
      +-------------+----------------------------------+

    [root@controller0 ~]# keystone service-create --name=nova --type=compute --description="Nova Compute Service"
      +-------------+----------------------------------+
      |   Property  |              Value               |
      +-------------+----------------------------------+
      | description |       Nova Compute Service       |
      |   enabled   |               True               |
      |      id     | 80edb3d3914644c4b0570fd8d8dabdaa |
      |     name    |               nova               |
      |     type    |             compute              |
      +-------------+----------------------------------+

    [root@controller0 ~]# keystone service-list
      +----------------------------------+----------+----------+---------------------------+
      |                id                |   name   |   type   |        description        |
      +----------------------------------+----------+----------+---------------------------+
      | 6afe8a067e2945fca023f85c7760ae53 |  glance  |  image   |    Glance Image Service   |
      | b3ea5d31edce4c10b3b4c18359de0d09 | keystone | identity | Keystone Identity Service |
      | 80edb3d3914644c4b0570fd8d8dabdaa |   nova   | compute  |    Nova Compute Service   |
      +----------------------------------+----------+----------+---------------------------+

添加endpoint：

.. code::

    [root@controller0 ~]# export my_host=192.168.77.50

    # 添加keystone的endpoint
    [root@controller0 ~]# keystone endpoint-create --region RegionOne \
    --service keystone \
    --publicurl "http://$my_host:\$(public_port)s/v2.0" \
    --internalurl "http://$my_host:\$(public_port)s/v2.0" \
    --adminurl "http://$my_host:\$(admin_port)s/v2.0"
      +-------------+-------------------------------------------+
      |   Property  |                   Value                   |
      +-------------+-------------------------------------------+
      |   adminurl  |  http://192.168.77.50:$(admin_port)s/v2.0 |
      |      id     |      09c263fa9b3c4a58bcead0b2f5aba1a1     |
      | internalurl | http://192.168.77.50:$(public_port)s/v2.0 |
      |  publicurl  | http://192.168.77.50:$(public_port)s/v2.0 |
      |    region   |                 RegionOne                 |
      |  service_id |      b3ea5d31edce4c10b3b4c18359de0d09     |
      +-------------+-------------------------------------------+

    # 添加glance的endpoint
    [root@controller0 ~]# keystone endpoint-create --region RegionOne \
    --service glance \
    --publicurl "http://$my_host:9292/v1" \
    --internalurl "http://$my_host:9292/v1" \
    --adminurl "http://$my_host:9292/v1" 
      +-------------+----------------------------------+
      |   Property  |              Value               |
      +-------------+----------------------------------+
      |   adminurl  |   http://192.168.77.50:9292/v1   |
      |      id     | 975ff2836b264e299c669372076666ee |
      | internalurl |   http://192.168.77.50:9292/v1   |
      |  publicurl  |   http://192.168.77.50:9292/v1   |
      |    region   |            RegionOne             |
      |  service_id | 6afe8a067e2945fca023f85c7760ae53 |
      +-------------+----------------------------------+

    # 添加nova的endpoint
    keystone endpoint-create --region RegionOne \
    --service nova \
    --publicurl "http://$my_host:\$(compute_port)s/v2/\$(tenant_id)s" \
    --internalurl "http://$my_host:\$(compute_port)s/v2/\$(tenant_id)s" \
    --adminurl "http://$my_host:\$(compute_port)s/v2/\$(tenant_id)s" 
      +-------------+--------------------------------------------------------+
      |   Property  |                         Value                          |
      +-------------+--------------------------------------------------------+
      |   adminurl  | http://192.168.77.50:$(compute_port)s/v2/$(tenant_id)s |
      |      id     |            194b7ddd24c94a0ebf79cd7275478dfc            |
      | internalurl | http://192.168.77.50:$(compute_port)s/v2/$(tenant_id)s |
      |  publicurl  | http://192.168.77.50:$(compute_port)s/v2/$(tenant_id)s |
      |    region   |                       RegionOne                        |
      |  service_id |            80edb3d3914644c4b0570fd8d8dabdaa            |
      +-------------+--------------------------------------------------------+

    [root@controller0 ~]# keystone endpoint-list 
      +----------------------------------+-----------+--------------------------------------------------------+--------------------------------------------------------+--------------------------------------------------------+----------------------------------+
      |                id                |   region  |                       publicurl                        |                      internalurl                       |                        adminurl                        |            service_id            |
      +----------------------------------+-----------+--------------------------------------------------------+--------------------------------------------------------+--------------------------------------------------------+----------------------------------+
      | 09c263fa9b3c4a58bcead0b2f5aba1a1 | RegionOne |       http://192.168.77.50:$(public_port)s/v2.0        |       http://192.168.77.50:$(public_port)s/v2.0        |        http://192.168.77.50:$(admin_port)s/v2.0        | b3ea5d31edce4c10b3b4c18359de0d09 |
      | 194b7ddd24c94a0ebf79cd7275478dfc | RegionOne | http://192.168.77.50:$(compute_port)s/v2/$(tenant_id)s | http://192.168.77.50:$(compute_port)s/v2/$(tenant_id)s | http://192.168.77.50:$(compute_port)s/v2/$(tenant_id)s | 80edb3d3914644c4b0570fd8d8dabdaa |
      | 975ff2836b264e299c669372076666ee | RegionOne |              http://192.168.77.50:9292/v1              |              http://192.168.77.50:9292/v1              |              http://192.168.77.50:9292/v1              | 6afe8a067e2945fca023f85c7760ae53 |
      +----------------------------------+-----------+--------------------------------------------------------+--------------------------------------------------------+--------------------------------------------------------+----------------------------------+

配置Glance
-----------

初始化glance
~~~~~~~~~~~~~

.. code::

    # 安装glance
    [root@controller0 ~]# yum install -y openstack-glance
    
    # 初始化数据库
    [root@controller0 ~]# mysql -u root -p 
    Enter password:
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 16
    Server version: 5.5.40-MariaDB-wsrep MariaDB Server, wsrep_25.11.r4026

    Copyright (c) 2000, 2014, Oracle, Monty Program Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> create database glance;
    Query OK, 1 row affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on glance.* to glance@'localhost' identified by 'password';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on glance.* to glance@'%' identified by 'password';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> flush privileges;
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> exit
    Bye

配置glance
~~~~~~~~~~~

.. code::

    [root@controller0 ~]# vi /etc/glance/glance-registry.conf

    # line 165: database
    connection=mysql://glance:password@localhost/glance

    # line 245: 添加keystone认证信息
    [keystone_authtoken]
    identity_uri=http://192.168.77.50:35357
    admin_tenant_name=service
    admin_user=glance
    admin_password=servicepassword

    # line 259: paste_deploy
    flavor=keystone

    [root@controller0 ~]# vi /etc/glance/glance-api.conf

    # line 240: 修改rabbit用户密码
    rabbit_userid=guest
    rabbit_password=password
    # line 339: database
    connection=mysql://glance:password@localhost/glance
    # line 433: 添加keystone认证信息
    [keystone_authtoken]
    auth_uri = http://192.168.77.50:35357/v2.0
    identity_uri=http://192.168.77.50:5000
    admin_tenant_name=service
    admin_user=glance
    admin_password=servicepassword
    revocation_cache_time=10
    # line 448: paste_deploy
    flavor=keystone

    [root@controller0 ~]# glance-manage db_sync 

    # 删除日志文件并启动，否则可能因为log文件权限问题而报错
    [root@controller0 ~]# rm /var/log/glance/api.log
    [root@controller0 ~]# for service in api registry; do
    systemctl start openstack-glance-$service
    systemctl enable openstack-glance-$service
    done

配置Nova
---------

初始化nova
~~~~~~~~~~~

.. code::
    
    [root@controller0 ~]# yum install -y openstack-nova
    [root@controller0 ~]# mysql -u root -p 
    Enter password:
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 18
    Server version: 5.5.40-MariaDB-wsrep MariaDB Server, wsrep_25.11.r4026

    Copyright (c) 2000, 2014, Oracle, Monty Program Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> create database nova;
    Query OK, 1 row affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on nova.* to nova@'localhost' identified by 'password';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on nova.* to nova@'%' identified by 'password';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> flush privileges;
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> exit
    Bye

配置nova
~~~~~~~~~

基本配置：

.. code::

    [root@controller0 ~]# mv /etc/nova/nova.conf /etc/nova/nova.conf.org 
    [root@controller0 ~]# vi /etc/nova/nova.conf
    # 新建以下内容
    [DEFAULT]
    # RabbitMQ服务信息
    rabbit_host=192.168.77.50
    rabbit_port=5672
    rabbit_userid=guest
    rabbit_password=password
    notification_driver=nova.openstack.common.notifier.rpc_notifier
    rpc_backend=rabbit
    # 本计算节点IP
    my_ip=192.168.77.50
    # 是否支持ipv6
    use_ipv6=false
    state_path=/var/lib/nova
    enabled_apis=ec2,osapi_compute,metadata
    osapi_compute_listen=0.0.0.0
    osapi_compute_listen_port=8774
    rootwrap_config=/etc/nova/rootwrap.conf
    api_paste_config=api-paste.ini
    auth_strategy=keystone
    lock_path=/var/lib/nova/tmp
    log_dir=/var/log/nova
    # Memcached服务信息
    memcached_servers=192.168.77.50:11211
    scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
    [glance]
    # Glance服务信息
    host=192.168.77.50
    port=9292
    protocol=http
    [database]
    # connection info for MariaDB
    connection=mysql://nova:password@localhost/nova
    [keystone_authtoken]
    # Keystone server's hostname or IP
    auth_uri = http://192.168.77.50:35357/v2.0
    identity_uri=http://192.168.77.50:5000
    admin_user=nova
    # Nova user's password added in Keystone
    admin_password=servicepassword
    admin_tenant_name=service
    signing_dir=/var/lib/nova/keystone-signing
    [root@controller0 ~]# chmod 640 /etc/nova/nova.conf 
    [root@controller0 ~]# chgrp nova /etc/nova/nova.conf 

接下来配置network服务，虽然nova-network并不是官方推荐的配置，但是它配置较为简单，所以在此仍然写出，可待后来 :ref:`neutron` 时再修改或则直接略过（注意服务以及配置文件）：

.. code::

    [root@controller0 ~]# vi /etc/nova/nova.conf
    # 在DEFAULT段中添加如下内容
    # nova-network
    network_driver=nova.network.linux_net
    libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver
    linuxnet_interface_driver=nova.network.linux_net.LinuxBridgeInterfaceDriver
    firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    network_api_class=nova.network.api.API
    security_group_api=nova
    network_manager=nova.network.manager.FlatDHCPManager
    network_size=254
    allow_same_net_traffic=False
    multi_host=True
    send_arp_for_ha=True
    share_dhcp_address=True
    force_dhcp_release=True
    # 指定public网络接口
    public_interface=eno16777736
    # 任意桥接接口
    flat_network_bridge=br100
    # 创建dummy接口
    flat_interface=dummy0

    # 添加用于flat-DHCP的虚拟接口
    [root@controller0 ~]# cat > /etc/sysconfig/network-scripts/ifcfg-dummy0 <<EOF
    DEVICE=dummy0
    BOOTPROTO=none
    ONBOOT=yes
    TYPE=Ethernet
    NM_CONTROLLED=no
    EOF

    # 加载dummy模块，用于虚拟机内网流量路由
    [root@controller0 ~]# echo "alias dummy0 dummy" > /etc/modprobe.d/dummy.conf 
    [root@controller0 ~]# ifconfig dummy0 up

启用服务，如果没用使用nova-network，请忽略数组中的network

.. code::

    [root@controller0 ~]# nova-manage db sync 
    [root@controller0 ~]# for service in api objectstore conductor scheduler cert consoleauth compute network; do
    systemctl start openstack-nova-$service
    systemctl enable openstack-nova-$service
    done

    [root@controller0 ~]# nova service-list
    +----+------------------+-------------+----------+---------+-------+----------------------------+-----------------+
    | Id | Binary           | Host        | Zone     | Status  | State | Updated_at                 | Disabled Reason |
    +----+------------------+-------------+----------+---------+-------+----------------------------+-----------------+
    | 1  | nova-conductor   | controller0 | internal | enabled | up    | 2015-06-19T01:40:07.000000 | -               |
    | 2  | nova-scheduler   | controller0 | internal | enabled | up    | 2015-06-19T01:40:08.000000 | -               |
    | 3  | nova-cert        | controller0 | internal | enabled | up    | 2015-06-19T01:40:10.000000 | -               |
    | 4  | nova-consoleauth | controller0 | internal | enabled | up    | 2015-06-19T01:40:11.000000 | -               |
    | 5  | nova-compute     | controller0 | nova     | enabled | up    | 2015-06-19T01:40:14.000000 | -               |
    | 6  | nova-network     | controller0 | internal | enabled | up    | 2015-06-19T01:40:15.000000 | -               |
    +----+------------------+-------------+----------+---------+-------+----------------------------+-----------------+

添加镜像
~~~~~~~~~

.. code::

    # 以admin用户认证
    [root@controller0 ~]# cat ~/admin_keystone
    export OS_USERNAME=admin
    export OS_PASSWORD=admin
    export OS_TENANT_NAME=admin
    export OS_AUTH_URL=http://192.168.77.50:35357/v2.0/
    export PS1='[\u@\h \W(keystone)]\$ '

    [root@controller0 ~]# source ~/admin_keystone

    # 如果可以执行下面的命令，说明认证成功，否则请检查其配置文件
    [root@controller0 ~(keystone)]# glance image-list
    +----+------+-------------+------------------+------+--------+
    | ID | Name | Disk Format | Container Format | Size | Status |
    +----+------+-------------+------------------+------+--------+
    +----+------+-------------+------------------+------+--------+

导入之前已经创建好的镜像：

.. code::

    [root@controller0 ~(keystone)]# glance image-create --name="centos7" --is-public=true --disk-format=qcow2 --container-format=bare < rhel7.0.qcow2
    +------------------+--------------------------------------+
    | Property         | Value                                |
    +------------------+--------------------------------------+
    | checksum         | 0ffb6f101c28af38804f79287f15e7e9     |
    | container_format | bare                                 |
    | created_at       | 2015-06-18T09:34:50.000000           |
    | deleted          | False                                |
    | deleted_at       | None                                 |
    | disk_format      | qcow2                                |
    | id               | 7f1f376c-0dff-44a3-87e8-d13883f795fc |
    | is_public        | True                                 |
    | min_disk         | 0                                    |
    | min_ram          | 0                                    |
    | name             | centos7                              |
    | owner            | c0c4e7b797bb41798202b55872fba074     |
    | protected        | False                                |
    | size             | 21478375424                          |
    | status           | active                               |
    | updated_at       | 2015-06-18T09:41:28.000000           |
    | virtual_size     | None                                 |
    +------------------+--------------------------------------+

    [root@controller0 ~(keystone)]# glance image-list
    +--------------------------------------+---------+-------------+------------------+-------------+--------+
    | ID                                   | Name    | Disk Format | Container Format | Size        | Status |
    +--------------------------------------+---------+-------------+------------------+-------------+--------+
    | 7f1f376c-0dff-44a3-87e8-d13883f795fc | centos7 | qcow2       | bare             | 21478375424 | active |
    +--------------------------------------+---------+-------------+------------------+-------------+--------+

配置Nova Network（可选）
~~~~~~~~~~~~~~~~~~~~~~~~

如果使用nova-network请参考此处，否则请忽略：

.. code::

    # 以admin用户认证
    [root@controller0 ~]# cat ~/admin_keystone
    export OS_USERNAME=admin
    export OS_PASSWORD=admin
    export OS_TENANT_NAME=admin
    export OS_AUTH_URL=http://192.168.77.50:35357/v2.0/
    export PS1='[\u@\h \W(keystone)]\$ '

    [root@controller0 ~]# source ~/admin_keystone

    # 创建实例的内网
    [root@controller0 ~(keystone)]# nova-manage network create --label neutron01 --dns1 10.0.0.1 --fixed_range_v4=10.1.0.0/24 
    [root@controller0 ~(keystone)]# nova-manage network list
    id      IPv4                IPv6            start address   DNS1            DNS2            VlanID          project         uuid           
    1       10.1.0.0/24         None            10.1.0.2        10.0.0.1        None            None            None            d5bac5d4-7d1f-49ea-98d7-ea9039e75740

    # 建立安全规则
    # 允许ssh访问
    [root@controller0 ~(keystone)]# nova secgroup-add-rule default tcp 22 22 0.0.0.0/0 
    +-------------+-----------+---------+-----------+--------------+
    | IP Protocol | From Port | To Port | IP Range  | Source Group |
    +-------------+-----------+---------+-----------+--------------+
    | tcp         | 22        | 22      | 0.0.0.0/0 |              |
    +-------------+-----------+---------+-----------+--------------+

    # 允许ping
    [root@controller0 ~(keystone)]# nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
    +-------------+-----------+---------+-----------+--------------+
    | IP Protocol | From Port | To Port | IP Range  | Source Group |
    +-------------+-----------+---------+-----------+--------------+
    | icmp        | -1        | -1      | 0.0.0.0/0 |              |
    +-------------+-----------+---------+-----------+--------------+

    [root@controller0 ~(keystone)]#  nova secgroup-list-rules default 
    +-------------+-----------+---------+-----------+--------------+
    | IP Protocol | From Port | To Port | IP Range  | Source Group |
    +-------------+-----------+---------+-----------+--------------+
    | tcp         | 22        | 22      | 0.0.0.0/0 |              |
    | icmp        | -1        | -1      | 0.0.0.0/0 |              |
    +-------------+-----------+---------+-----------+--------------+

    # 允许虚拟机启动时使用floating-ip
    [root@controller0 ~(keystone)]# vi /etc/nova/nova.conf
    # 在DEFAULT段中添加
    auto_assign_floating_ip=true

    # 重启nova-network
    [root@controller0 ~(keystone)]# systemctl restart openstack-nova-network

    # 指定10.0.0.0/24中的5个IP用于floating-ip给虚拟机使用
    [root@controller0 ~(keystone)]# nova-manage floating create --ip_range=10.0.0.248/29
    [root@controller0 ~(keystone)]# nova-manage floating list
    None    10.0.0.249  None    nova    eno16777736
    None    10.0.0.250  None    nova    eno16777736
    None    10.0.0.251  None    nova    eno16777736
    None    10.0.0.252  None    nova    eno16777736
    None    10.0.0.253  None    nova    eno16777736
    None    10.0.0.254  None    nova    eno16777736

    # 测试启动
    [root@controller0 ~(keystone)]# nova boot --flavor 2 --image centos7iso --security_group default centos7iso
    +--------------------------------------+---------------------------------------------------+
    | Property                             | Value                                             |
    +--------------------------------------+---------------------------------------------------+
    | OS-DCF:diskConfig                    | MANUAL                                            |
    | OS-EXT-AZ:availability_zone          | nova                                              |
    | OS-EXT-SRV-ATTR:host                 | -                                                 |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                 |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000003                                 |
    | OS-EXT-STS:power_state               | 0                                                 |
    | OS-EXT-STS:task_state                | scheduling                                        |
    | OS-EXT-STS:vm_state                  | building                                          |
    | OS-SRV-USG:launched_at               | -                                                 |
    | OS-SRV-USG:terminated_at             | -                                                 |
    | accessIPv4                           |                                                   |
    | accessIPv6                           |                                                   |
    | adminPass                            | 3qF4JPhERims                                      |
    | config_drive                         |                                                   |
    | created                              | 2015-06-19T03:47:15Z                              |
    | flavor                               | m1.small (2)                                      |
    | hostId                               |                                                   |
    | id                                   | a0cae25e-4629-48da-a054-99aed02baff9              |
    | image                                | centos7iso (d8d93d5f-56cf-4ce6-a2d1-f856fca529e2) |
    | key_name                             | -                                                 |
    | metadata                             | {}                                                |
    | name                                 | centos7iso                                        |
    | os-extended-volumes:volumes_attached | []                                                |
    | progress                             | 0                                                 |
    | security_groups                      | default                                           |
    | status                               | BUILD                                             |
    | tenant_id                            | c0c4e7b797bb41798202b55872fba074                  |
    | updated                              | 2015-06-19T03:47:15Z                              |
    | user_id                              | cf11b4425218431991f095c2f58578a0                  |
    +--------------------------------------+---------------------------------------------------+
    [root@controller0 ~(keystone)]# nova list
    +--------------------------------------+------------+--------+------------+-------------+-------------------------------+
    | ID                                   | Name       | Status | Task State | Power State | Networks                      |
    +--------------------------------------+------------+--------+------------+-------------+-------------------------------+
    | a0cae25e-4629-48da-a054-99aed02baff9 | centos7iso | BUILD  | spawning   | NOSTATE     | neutron01=10.1.0.2, 10.0.0.249|
    +--------------------------------------+------------+--------+------------+-------------+-------------------------------+
    
    # 添加另一个floating-ip
    [root@controller0 ~(keystone)]# nova add-floating-ip centos7iso 10.0.0.250
    [root@controller0 ~(keystone)]# nova list
    +--------------------------------------+------------+--------+------------+-------------+--------------------------------------------+
    | ID                                   | Name       | Status | Task State | Power State | Networks                                   |
    +--------------------------------------+------------+--------+------------+-------------+--------------------------------------------+
    | a0cae25e-4629-48da-a054-99aed02baff9 | centos7iso | BUILD  | spawning   | NOSTATE     | neutron01=10.1.0.2, 10.0.0.249, 10.0.0.250 |
    +--------------------------------------+------------+--------+------------+-------------+--------------------------------------------+

配置Horizon
------------

添加web界面。

.. code::

    # 安装必需包
    [root@controller0 ~(keystone)]# yum --enablerepo=openstack-juno,epel -y install openstack-dashboard openstack-nova-novncproxy

    # 配置vnc
    [root@controller0 ~(keystone)]# vi /etc/nova/nova.conf
    # 于DEFAULT段中添加
    vnc_enabled=true
    novncproxy_host=0.0.0.0
    novncproxy_port=6080
    # replace the IP address to your own IP
    novncproxy_base_url=http://192.168.77.50:6080/vnc_auto.html
    vncserver_listen=192.168.77.50
    vncserver_proxyclient_address=192.168.77.50

    # 使能dashboard
    [root@controller0 ~(keystone)]# vi /etc/openstack-dashboard/local_settings
    # line 15: 允许所有人访问
    ALLOWED_HOSTS = ['*']
    # line 134:
    OPENSTACK_HOST = "192.168.77.50"

    # 启用服务
    [root@controller0 ~(keystone)]# systemctl start openstack-nova-novncproxy 
    [root@controller0 ~(keystone)]# systemctl restart openstack-nova-compute 
    [root@controller0 ~(keystone)]# systemctl restart httpd 
    [root@controller0 ~(keystone)]# systemctl enable openstack-nova-novncproxy 
    [root@controller0 ~(keystone)]# systemctl enable httpd


添加计算节点
------------

现在开始加入第二个计算节点compute0：

.. code::

    # 安装必需包
    [root@compute0 ~]# yum install -y openstack-nova-compute openstack-nova-api openstack-nova-network
    
    # 配置nova
    [root@compute0 ~]# mv /etc/nova/nova.conf /etc/nova/nova.conf.org 
    [root@compute0 ~]# vi /etc/nova/nova.conf
    [DEFAULT]
    rabbit_host=192.168.77.50
    rabbit_port=5672
    rabbit_userid=guest
    rabbit_password=password
    notification_driver=nova.openstack.common.notifier.rpc_notifier
    rpc_backend=rabbit
    my_ip=192.168.77.51
    use_ipv6=false
    state_path=/var/lib/nova
    enabled_apis=ec2,osapi_compute,metadata
    osapi_compute_listen=0.0.0.0
    osapi_compute_listen_port=8774
    rootwrap_config=/etc/nova/rootwrap.conf
    api_paste_config=api-paste.ini
    auth_strategy=keystone
    lock_path=/var/lib/nova/tmp
    log_dir=/var/log/nova
    memcached_servers=192.168.77.50:11211
    scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
    [glance]
    host=192.168.77.50
    port=9292
    protocol=http
    [database]
    connection=mysql://nova:password@192.168.77.50/nova
    [keystone_authtoken]
    auth_uri = http://192.168.77.50:35357/v2.0
    identity_uri=http://192.168.77.50:5000
    admin_user=nova
    # Nova user's password added in Keystone
    admin_password=servicepassword
    admin_tenant_name=service
    signing_dir=/var/lib/nova/keystone-signing

    [root@compute0 ~]# chmod 640 /etc/nova/nova.conf 
    [root@compute0 ~]# chgrp nova /etc/nova/nova.conf 

配置nova-network：

.. code::

    [root@controller0 ~]# vi /etc/nova/nova.conf
    # 在DEFAULT段中添加如下内容
    # nova-network
    network_driver=nova.network.linux_net
    libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver
    linuxnet_interface_driver=nova.network.linux_net.LinuxBridgeInterfaceDriver
    firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    network_api_class=nova.network.api.API
    security_group_api=nova
    network_manager=nova.network.manager.FlatDHCPManager
    network_size=254
    allow_same_net_traffic=False
    multi_host=True
    send_arp_for_ha=True
    share_dhcp_address=True
    force_dhcp_release=True
    # 指定public网络接口
    public_interface=eno16777736
    # 任意桥接接口
    flat_network_bridge=br100
    # 创建dummy接口
    flat_interface=dummy0
    # 如果需要自动floating-ip
    auto_assign_floating_ip=true

启动服务，如果不需要nova-network请同样省略数组中的network：

.. code::

    [root@compute0 ~]# for service in metadata-api compute network; do systemctl start openstack-nova-$service; systemctl enable openstack-nova-$service; done
    [root@compute0 ~]# nova-manage service list 
    Binary           Host                                 Zone             Status     State Updated_At
    nova-conductor   controller0                          internal         enabled    :-)   2015-06-19 05:31:48
    nova-scheduler   controller0                          internal         enabled    :-)   2015-06-19 05:31:48
    nova-cert        controller0                          internal         enabled    :-)   2015-06-19 05:31:51
    nova-consoleauth controller0                          internal         enabled    :-)   2015-06-19 05:31:52
    nova-compute     controller0                          nova             enabled    :-)   2015-06-19 05:31:45
    nova-network     controller0                          internal         enabled    :-)   2015-06-19 05:31:44
    nova-compute     compute0                             nova             enabled    :-)   2015-06-19 05:31:50
    nova-network     compute0                             internal         enabled    :-)   2015-06-19 05:31:51

.. _neutron:

配置Neutron（推荐）
-------------------

如果已经安装上面的顺序（排除nova-networking）配置下来，现在应该有两个计算节点了。

那么我们的配置如下：

.. code::

                                    
    +------------------+               |               +------------------------+
    | [ contoller0   ] |               |               |     [  neutron0  ]     |
    |     Keystone     |192.168.77.50  |  192.168.77.30|        DHCP Agent      |
    |      Glance      |---------------+---------------|        L3 Agent        |
    |     Nova API     |eth0           |           eth0|        L2 Agent        |
    |  Neutron Server  |               |               |     Metadata Agent     |
    |   Nova Compute   |               |               +------------------------+
    +------------------+               |
                                   eth0|192.168.77.51
                            +--------------------+
                            |   [  compute0  ]   |
                            |    Nova Compute    |
                            |      L2 Agent      |
                            +--------------------+

控制节点controller0配置
~~~~~~~~~~~~~~~~~~~~~~~~

安装neutron

neutron依赖于各种插件（openvswitch、linuxbridge等），我们在此使用openvswitch。

.. code::

    # 安装neutron
    [root@controller0 ~(keystone)]# yum install -y openstack-neutron openstack-neutron-ml2

    # 初始化数据库
    [root@controller0 ~(keystone)]# mysql -u root -p 
    Enter password:
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 14
    Server version: 5.5.40-MariaDB-wsrep MariaDB Server, wsrep_25.11.r4026

    Copyright (c) 2000, 2014, Oracle, Monty Program Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> create database neutron_ml2; 
    Query OK, 1 row affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on neutron_ml2.* to neutron@'localhost' identified by 'password'; 
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on neutron_ml2.* to neutron@'%' identified by 'password'; 
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> flush privileges; 
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> exit 
    Bye

    # 创建neutron服务
    [root@controller0 ~(keystone)]# keystone user-create --tenant service --name neutron --pass servicepassword --enabled true 
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |  email   |                                  |
    | enabled  |               True               |
    |    id    | 6dafe1f763de44778fa9c4848da7d20f |
    |   name   |             neutron              |
    | tenantId | 9acf83020ae34047b6f1e320c352ae44 |
    | username |             neutron              |
    +----------+----------------------------------+
    [root@controller0 ~(keystone)]# keystone user-role-add --user neutron --tenant service --role admin
    [root@controller0 ~(keystone)]# keystone service-create --name=neutron --type=network --description="Neutron Network Service" 
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |     Neutron Network Service      |
    |   enabled   |               True               |
    |      id     | 534492ae3d48407bb3b2a90607f43461 |
    |     name    |             neutron              |
    |     type    |             network              |
    +-------------+----------------------------------+

    [root@controller0 ~(keystone)]# export neutron_server=192.168.77.50
    [root@controller0 ~(keystone)]# keystone endpoint-create --region RegionOne --service neutron --publicurl "http://$neutron_server:9696/" --internalurl "http://$neutron_server:9696/" --adminurl "http://$neutron_server:9696/"
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    |   adminurl  |    http://192.168.77.50:9696/    |
    |      id     | 74fd6b095c16452d97ffcb2b1fd0dad3 |
    | internalurl |    http://192.168.77.50:9696/    |
    |  publicurl  |    http://192.168.77.50:9696/    |
    |    region   |            RegionOne             |
    |  service_id | 534492ae3d48407bb3b2a90607f43461 |
    +-------------+----------------------------------+

    # 配置neutron
    [root@controller0 ~(keystone)]# vi /etc/neutron/neutron.conf
    # [DEFAULT]
    # line 62: 后端插件
    core_plugin=ml2
    # line 69: 服务插件
    service_plugins=router
    # line 84: 认证方式
    auth_strategy=keystone
    # line 110: 取消注释
    dhcp_agent_notification=True
    # line 339: 控制节点的nova端
    nova_url=http://192.168.77.50:8774/v2
    # line 345: nova用户名
    nova_admin_username=nova
    # line 348: service用户的tenant id(可使用keystone tenant-list查看)
    nova_admin_tenant_id=9acf83020ae34047b6f1e320c352ae44
    # line 357: nova用户的service密码
    nova_admin_password=servicepassword
    # line 360: keystone认证端
    nova_admin_auth_url=http://192.168.77.50:35357/v2.0
    # [oslo_messaging_rabbit]
    # line 445: rabbitMQ服务器
    rabbit_host=192.168.77.50
    # line 449: rabbitMQ端口
    rabbit_port=5672
    # line 458: rabbitMQ用户信息
    rabbit_userid=guest
    rabbit_password=password
    # line 464: rpc后端，可从AMQ或者RABBITMQ中选择
    rpc_backend=rabbit
    # line 551: 控制信息交换格式
    control_exchange=neutron
    # line 688: keystone认证信息，由于auth_uri以后会被identity_uri代替，并且auth_host等信息也不必要了，但为兼容性起见，此处我给予保留
    [keystone_authtoken]
    auth_uri = http://192.168.77.50:35357/v2.0
    identity_uri=http://192.168.77.50:5000
    admin_tenant_name = service
    admin_user = neutron
    admin_password = servicepassword
    # line 708: 数据库
    connection = mysql://neutron:password@192.168.77.50/neutron_ml2

    # 配置ml2插件
    [root@controller0 ~(keystone)]# vi /etc/neutron/plugins/ml2/ml2_conf.ini
    # line 7: 网络支持
    type_drivers = flat,vlan,gre
    tenant_network_types = vlan,gre
    mechanism_drivers = openvswitch
    # line 93: 启用安全组
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    # 配置nova节点以支持neutron
    [root@controller0 ~(keystone)]# vi /etc/nova/nova.conf
    # add in the [DEFAULT] section
    # nova-network
    #network_driver=nova.network.linux_net
    #libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver
    #linuxnet_interface_driver=nova.network.linux_net.LinuxBridgeInterfaceDriver
    #firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    #network_api_class=nova.network.api.API
    #security_group_api=nova
    #network_manager=nova.network.manager.FlatDHCPManager
    #network_size=254
    #allow_same_net_traffic=False
    #multi_host=True
    #send_arp_for_ha=True
    #share_dhcp_address=True
    #force_dhcp_release=True
    ## specify nic for public
    #public_interface=eno16777736
    ## specify any name you like for bridge
    #flat_network_bridge=br100
    ## specify nic for flat DHCP bridge
    #flat_interface=dummy0
    #auto_assign_floating_ip=true
    # neutron-network
    network_api_class=nova.network.neutronv2.api.API
    security_group_api=neutron

    # 在末尾添加neutron用户认证信息
    [neutron]
    url = http://192.168.77.50:9696
    auth_strategy = keystone
    admin_auth_url = http://192.168.77.50:35357/v2.0
    admin_tenant_name = service
    admin_username = neutron
    admin_password = servicepassword

    [root@controller0 ~(keystone)]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini 
    [root@controller0 ~(keystone)]# neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head 
    # 启用neutron-server服务，如果之前有配置nova-network，在此需禁用
    [root@controller0 ~(keystone)]# systemctl stop openstack-nova-network
    [root@controller0 ~(keystone)]# systemctl disable openstack-nova-network
    [root@controller0 ~(keystone)]# systemctl start neutron-server 
    [root@controller0 ~(keystone)]# systemctl enable neutron-server 
    [root@controller0 ~(keystone)]# systemctl restart openstack-nova-api 


网络节点neutron0配置
~~~~~~~~~~~~~~~~~~~~~

在节点neutron0上，我们进行如下配置。

.. code::

    # 安装必需包
    [root@neutron0 ~]# yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch

    # 打开ip_forward
    [root@neutron0 ~]# echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf 
    [root@neutron0 ~]# echo 'net.ipv4.conf.default.rp_filter=0' >> /etc/sysctl.conf 
    [root@neutron0 ~]# echo 'net.ipv4.conf.all.rp_filter=0' >> /etc/sysctl.conf 
    [root@neutron0 ~]# sysctl -p 

    # 配置neutron
    [root@network ~]# vi /etc/neutron/neutron.conf
    # line 60
    core_plugin=ml2
    # line 69
    service_plugins=router
    # line 84
    auth_strategy=keystone
    # line 110
    dhcp_agent_notification=True
    # line 444: rabbitMQ信息
    rabbit_host=192.168.77.50
    # line 448
    rabbit_port=5672
    # line 457
    rabbit_userid=guest
    # line 460
    rabbit_password=password
    # line 545
    rpc_backend=rabbit
    # line 550
    control_exchange=neutron
    # line 687: keystone认证信息
    [keystone_authtoken]
    auth_uri = http://192.168.77.50:35357/v2.0
    identity_uri=http://192.168.77.50:5000
    admin_tenant_name = service
    admin_user = neutron
    admin_password = servicepassword

    # 配置三层交换代理
    [root@neutron0 ~]# vi /etc/neutron/l3_agent.ini
    # line 19: uncomment
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    # line 25: uncomment
    use_namespaces = True
    # line 63: add (it's OK to keep value empty (set it if needed))
    external_network_bridge =

    # 配置dhcp代理
    [root@neutron0 ~]# vi /etc/neutron/dhcp_agent.ini
    # line 27: uncomment
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    # line 31: uncomment
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    # line 37: uncomment
    use_namespaces = True

    # 配置元数据代理
    [root@neutron0 ~]# vi /etc/neutron/metadata_agent.ini
    # line 6: change (specify endpoint of keystone)
    auth_url = http://192.168.77.50:35357/v2.0
    # line 12: change (auth info ofr keystone)
    admin_tenant_name = service
    admin_user = neutron
    admin_password = servicepassword
    # line 20: uncomment and specify Nova API server
    nova_metadata_ip = 10.0.0.30
    # line 23: uncomment
    nova_metadata_port = 8775
    # line 43: uncomment and specify any secret key you like
    metadata_proxy_shared_secret = metadata_secret

    # 配置ml2
    [root@neutron0 ~]# vi /etc/neutron/plugins/ml2/ml2_conf.ini
    # line 7: add
    type_drivers = flat,vlan,gre
    tenant_network_types = vlan,gre
    mechanism_drivers = openvswitch
    # line 92: uncomment and add
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [root@neutron0 ~]# mv /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini.org 
    [root@neutron0 ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini 
    [root@neutron0 ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini 
    [root@neutron0 ~]# systemctl start openvswitch 
    [root@neutron0 ~]# systemctl enable openvswitch 
    [root@neutron0 ~]# ovs-vsctl add-br br-int 
    [root@neutron0 ~]# for service in dhcp-agent l3-agent metadata-agent openvswitch-agent; do
    systemctl start neutron-$service
    systemctl enable neutron-$service
    done 

计算节点compute0配置
~~~~~~~~~~~~~~~~~~~~~

在除controller0的另一个计算节点compute0上，我们进行如下配置。

.. code::

    # 安装必需包
    [root@compute0 ~]# yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch

    # 调节rp_filter
    [root@compute0 ~]# echo 'net.ipv4.conf.default.rp_filter=0' >> /etc/sysctl.conf 
    [root@compute0 ~]# echo 'net.ipv4.conf.all.rp_filter=0' >> /etc/sysctl.conf 
    [root@compute0 ~]# sysctl -p 

    # 配置neutron
    [root@compute0 ~]# vi /etc/neutron/neutron.conf
    # line 60
    core_plugin=ml2
    # line 69
    service_plugins=router
    # line 84
    auth_strategy=keystone
    # line 110
    dhcp_agent_notification=True
    # line 444: rabbitMQ信息
    rabbit_host=192.168.77.50
    # line 448
    rabbit_port=5672
    # line 457
    rabbit_userid=guest
    # line 460
    rabbit_password=password
    # line 545
    rpc_backend=rabbit
    # line 550
    control_exchange=neutron
    # line 687: keystone认证信息
    [keystone_authtoken]
    auth_uri = http://192.168.77.50:35357/v2.0
    identity_uri=http://192.168.77.50:5000
    admin_tenant_name = service
    admin_user = neutron
    admin_password = servicepassword

    [root@compute0 ~]# vi /etc/neutron/plugins/ml2/ml2_conf.ini
    # line 7: add
    type_drivers = flat,vlan,gre
    tenant_network_types = vlan,gre
    mechanism_drivers = openvswitch
    # line 69: uncomment and add
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [root@compute0 ~]# mv /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini.org 
    [root@compute0 ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini 
    [root@compute0 ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini 

    # 配置nova
    [root@compute0 ~]# vi /etc/nova/nova.conf
    # add in the [DEFAULT] section
    linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver=nova.virt.firewall.NoopFirewallDriver
    network_api_class=nova.network.neutronv2.api.API
    security_group_api=neutron
    # specify the Neutron endpoint
    neutron_url=http://192.168.77.50:9696
    # specify the auth info for keystone
    neutron_auth_strategy=keystone
    neutron_admin_tenant_name=service
    neutron_admin_username=neutron
    neutron_admin_password=servicepassword
    neutron_admin_auth_url=http://192.168.77.50:35357/v2.0
    metadata_listen=0.0.0.0
    # specify the Control node
    metadata_host=192.168.77.50
    service_neutron_metadata_proxy=True
    # specify the metadata secret key (it is just the value you set in the Network node)
    neutron_metadata_proxy_shared_secret=metadata_secret
    vif_plugging_is_fatal=false
    vif_plugging_timeout=0

    # 启用服务，如果之前有配置nova-network，在此需禁用
    [root@compute0 ~]# systemctl stop openstack-nova-network
    [root@compute0 ~]# systemctl disable openstack-nova-network

    [root@compute0 ~]# systemctl start openvswitch 
    [root@compute0 ~]# systemctl enable openvswitch 
    [root@compute0 ~]# ovs-vsctl add-br br-int 
    [root@compute0 ~]# systemctl restart openstack-nova-compute 
    [root@compute0 ~]# systemctl restart openstack-nova-metadata-api 
    [root@compute0 ~]# systemctl start neutron-openvswitch-agent 
    [root@compute0 ~]# systemctl enable neutron-openvswitch-agent 

使用Neutron
~~~~~~~~~~~~

.. code::

    +-------------+                    +----+----+
    | Name Server |                    | Gateway |
    +------+------+                    +----+----+
           |192.168.77.2                    |192.168.77.2
           |                                |
           +--------------+-----------------+------------------------+
           |              |                 |                        |
           |              |                 |                        |192.168.77.200-192.168.77.254
       eth0|192.168.77.50 |    192.168.77.30| eth0          +--------+-------+
  +--------+---------+    |     +-----------+----------+    | Virtual Router |
  | [ controller0 ]  |    |     |   [   neutron0   ]   |    +--------+-------+
  |     Keystone     |    |     |       DHCP Agent     |       192.168.100.1
  |      Glance      |    | eth2|       L3 Agent       |eth1         |            192.168.100.0/24
  |     Nova API     |    |     |       L2 Agent       |             |           +-----------------+
  |  Neutron Server  |    |     |    Metadata Agent    |             |       +---| Virtual Machine |
  +------------------+    |     +----------------------+             |       |   +-----------------+
                          |                                          |       |   +-----------------+
                          |     +----------------------+             +-------+---| Virtual Machine |
                          | eth0|     [ compute0  ]    |eth1                 |   +-----------------+
                          +-----|     Nova Compute     |                     |   +-----------------+
                   192.168.77.51|       L2 Agent       |                     |---| Virtual Machine |
                                +----------------------+                         +-----------------+

    其中，controller0、compute0都有两个物理网口，neutron0有三个物理网口。

修改控制节点配置：

.. code::

    [root@controller0 ~(keystone)]# vi /etc/neutron/plugins/ml2/ml2_conf.ini
    # line 64
    [ml2_type_vlan]
    network_vlan_ranges = physnet1:1000:2999
    # 末尾添加
    [ovs]
    tenant_network_type = vlan
    bridge_mappings = physnet1:br-eth1
    [root@controller0 ~(keystone)]# systemctl restart neutron-server 

在网络节点和计算节点同时添加eth1作内网：

.. code::

    # 添加一个桥
    [root@neutron0 ~]# ovs-vsctl add-br br-eth1
    # 将eno33554984网口附加到桥，即对应eth1
    [root@neutron0 ~]# ovs-vsctl add-port br-eth1 eno33554984
    [root@neutron0 ~]# vi /etc/neutron/plugins/ml2/ml2_conf.ini
    # line 64
    [ml2_type_vlan]
    network_vlan_ranges = physnet1:1000:2999
    # 末尾添加
    [ovs]
    tenant_network_type = vlan
    bridge_mappings = physnet1:br-eth1
    [root@neutron0 ~]# systemctl restart neutron-openvswitch-agent 

在网络节点添加eth2作外网：

.. code::

    [root@neutron0 ~]# ovs-vsctl add-br br-ext 
    # eno50332208对应eth2
    [root@neutron0 ~]# ovs-vsctl add-port br-ext eno50332208
    [root@neutron0 ~]# vi /etc/neutron/l3_agent.ini
    # line 63
    external_network_bridge = br-ext
    [root@neutron0 ~]# systemctl restart neutron-l3-agent 

在任意节点修改（neutron的配置属于集群全局配置，此处在控制节点修改，其他节点也可）：

.. code::

    # create a virtual router
    [root@controller0 ~(keystone)]# neutron router-create router01
    Created a new router:
    +-----------------------+--------------------------------------+
    | Field                 | Value                                |
    +-----------------------+--------------------------------------+
    | admin_state_up        | True                                 |
    | distributed           | False                                |
    | external_gateway_info |                                      |
    | ha                    | False                                |
    | id                    | 8bf0184c-1cd8-4993-b3e0-7be94aaf2757 |
    | name                  | router01                             |
    | routes                |                                      |
    | status                | ACTIVE                               |
    | tenant_id             | c0c4e7b797bb41798202b55872fba074     |
    +-----------------------+--------------------------------------+

    [root@controller0 ~(keystone)]# Router_ID=`neutron router-list | grep router01 | awk '{ print $2 }'` 

    # 创建内网
    [root@controller0 ~(keystone)]# neutron net-create int_net 
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | id                        | 532e391d-562d-4499-8dee-48ca31345466 |
    | mtu                       | 0                                    |
    | name                      | int_net                              |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  | 1000                                 |
    | router:external           | False                                |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tenant_id                 | c0c4e7b797bb41798202b55872fba074     |
    +---------------------------+--------------------------------------+

    # 创建内网子网
    [root@controller0 ~(keystone)]# neutron subnet-create --gateway 192.168.100.1 --dns-nameserver 192.168.77.2 int_net 192.168.100.0/24
    Created a new subnet:
    +-------------------+------------------------------------------------------+
    | Field             | Value                                                |
    +-------------------+------------------------------------------------------+
    | allocation_pools  | {"start": "192.168.100.2", "end": "192.168.100.254"} |
    | cidr              | 192.168.100.0/24                                     |
    | dns_nameservers   | 192.168.77.2                                         |
    | enable_dhcp       | True                                                 |
    | gateway_ip        | 192.168.100.1                                        |
    | host_routes       |                                                      |
    | id                | c08dcadf-f632-44b7-9a10-8a3a89c86853                 |
    | ip_version        | 4                                                    |
    | ipv6_address_mode |                                                      |
    | ipv6_ra_mode      |                                                      |
    | name              |                                                      |
    | network_id        | 532e391d-562d-4499-8dee-48ca31345466                 |
    | subnetpool_id     |                                                      |
    | tenant_id         | c0c4e7b797bb41798202b55872fba074                     |
    +-------------------+------------------------------------------------------+
    [root@controller0 ~(keystone)]# Int_Subnet_ID=`neutron net-list | grep int_net | awk '{ print $6 }'`

    # 将内网实例附加到路由
    [root@controller0 ~(keystone)]# neutron router-interface-add $Router_ID $Int_Subnet_ID 
    Added interface a2e9bedc-0505-45da-8f87-4a82928a6206 to router 8bf0184c-1cd8-4993-b3e0-7be94aaf2757.

    # 创建外网
    [root@controller0 ~(keystone)]# neutron net-create ext_net --router:external
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | id                        | e041481d-f8b8-42a7-b87b-3d346167ef21 |
    | mtu                       | 0                                    |
    | name                      | ext_net                              |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  | 1001                                 |
    | router:external           | True                                 |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tenant_id                 | c0c4e7b797bb41798202b55872fba074     |
    +---------------------------+--------------------------------------+

    # 创建外网子网
    [root@controller0 ~(keystone)]# neutron subnet-create ext_net --allocation-pool start=192.168.77.200,end=192.168.77.254 --gateway 192.168.77.2 --dns-nameserver 192.168.77.2 192.168.77.0/24 --disable-dhcp 
    Created a new subnet:
    +-------------------+------------------------------------------------------+
    | Field             | Value                                                |
    +-------------------+------------------------------------------------------+
    | allocation_pools  | {"start": "192.168.77.200", "end": "192.168.77.254"} |
    | cidr              | 192.168.77.0/24                                      |
    | dns_nameservers   | 192.168.77.2                                         |
    | enable_dhcp       | False                                                |
    | gateway_ip        | 192.168.77.2                                         |
    | host_routes       |                                                      |
    | id                | 98f97e64-94d8-4743-b8a1-a715f2c07e08                 |
    | ip_version        | 4                                                    |
    | ipv6_address_mode |                                                      |
    | ipv6_ra_mode      |                                                      |
    | name              |                                                      |
    | network_id        | e041481d-f8b8-42a7-b87b-3d346167ef21                 |
    | subnetpool_id     |                                                      |
    | tenant_id         | c0c4e7b797bb41798202b55872fba074                     |
    +-------------------+------------------------------------------------------+

    # 将外网实例附加到路由
    [root@controller0 ~(keystone)]# Ext_Net_ID=`neutron net-list | grep ext_net | awk '{ print $2 }'` 
    [root@controller0 ~(keystone)]# neutron router-gateway-set $Router_ID $Ext_Net_ID 
    Set gateway for router 8bf0184c-1cd8-4993-b3e0-7be94aaf2757

    # 创建并启动虚拟机
    [root@controller0 ~(keystone)]# Int_Net_ID=`neutron net-list | grep int_net | awk '{ print $2 }'` 
    [root@controller0 ~(keystone)]# nova image-list 
    +--------------------------------------+---------+--------+--------+
    | ID                                   | Name    | Status | Server |
    +--------------------------------------+---------+--------+--------+
    | 4a663fce-97eb-42d3-93d6-415e477bc0a4 | CentOS7 | ACTIVE |        |
    +--------------------------------------+---------+--------+--------+

    [root@controller0 ~(keystone)]# nova boot --flavor 2 --image CentOS7 --security_group default --nic net-id=$Int_Net_ID CentOS_70 
    [root@controller0 ~(keystone)]# nova list 
    +-----------+-----------+--------+------------+-------------+-----------------------+
    | ID        | Name      | Status | Task State | Power State | Networks              |
    +-----------+-----------+--------+------------+-------------+-----------------------+
    | 33bb9427- | CentOS_70 | ACTIVE | -          | Running     | int_net=192.168.100.2 |
    +-----------+-----------+--------+------------+-------------+-----------------------+

    # 添加浮动IP
    [root@controller0 ~(keystone)]# neutron floatingip-create ext_net 
    Created a new floatingip:
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | fixed_ip_address    |                                      |
    | floating_ip_address | 10.0.0.201                           |
    | floating_network_id | bd216cab-c07b-4475-90ef-e9ad402bd57b |
    | id                  | da8eef0d-5bc8-488e-8fd4-0c6df1f5922a |
    | port_id             |                                      |
    | router_id           |                                      |
    | status              | DOWN                                 |
    | tenant_id           | e8f6ac69de5f46afa189fcefd99c8a1a     |
    +---------------------+--------------------------------------+

    [root@controller0 ~(keystone)]# Device_ID=`nova list | grep CentOS_70 | awk '{ print $2 }'` 
    [root@controller0 ~(keystone)]# Port_ID=`neutron port-list -- --device_id $Device_ID | grep 192.168.100.2 | awk '{ print $2 }'` 
    [root@controller0 ~(keystone)]# Floating_ID=`neutron floatingip-list | grep 10.0.0.201 | awk '{ print $2 }'` 
    [root@controller0 ~(keystone)]# neutron floatingip-associate $Floating_ID $Port_ID 
    Associated floating IP da8eef0d-5bc8-488e-8fd4-0c6df1f5922a
    # confirm settings
    [root@controller0 ~(keystone)]# neutron floatingip-show $Floating_ID 
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | fixed_ip_address    | 192.168.100.2                        |
    | floating_ip_address | 10.0.0.201                           |
    | floating_network_id | bd216cab-c07b-4475-90ef-e9ad402bd57b |
    | id                  | da8eef0d-5bc8-488e-8fd4-0c6df1f5922a |
    | port_id             | d4f17f91-c4e9-45ec-af2d-223907e891ea |
    | router_id           | a0d08cb3-bf96-4872-ab95-b24a697b080a |
    | status              | ACTIVE                               |
    | tenant_id           | e8f6ac69de5f46afa189fcefd99c8a1a     |
    +---------------------+--------------------------------------+

    # 添加安全组
    # permit SSH
    [root@controller0 ~(keystone)]# nova secgroup-add-rule default tcp 22 22 0.0.0.0/0 
    +-------------+-----------+---------+-----------+--------------+
    | IP Protocol | From Port | To Port | IP Range  | Source Group |
    +-------------+-----------+---------+-----------+--------------+
    | tcp         | 22        | 22      | 0.0.0.0/0 |              |
    +-------------+-----------+---------+-----------+--------------+

    # permit ICMP
    [root@controller0 ~(keystone)]# nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0 
    +-------------+-----------+---------+-----------+--------------+
    | IP Protocol | From Port | To Port | IP Range  | Source Group |
    +-------------+-----------+---------+-----------+--------------+
    | icmp        | -1        | -1      | 0.0.0.0/0 |              |
    +-------------+-----------+---------+-----------+--------------+

    [root@controller0 ~(keystone)]# nova secgroup-list-rules default 
    +-------------+-----------+---------+-----------+--------------+
    | IP Protocol | From Port | To Port | IP Range  | Source Group |
    +-------------+-----------+---------+-----------+--------------+
    | tcp         | 22        | 22      | 0.0.0.0/0 |              |
    | icmp        | -1        | -1      | 0.0.0.0/0 |              |
    +-------------+-----------+---------+-----------+--------------+


配置Cinder
-----------

结构图如下：

.. code::

                                             +------------------+
                                192.168.77.60|   [ cinder0 ]    |
    +------------------+               +-----+   Cinder-Volume  |
    | [ controller0 ]  |               | eth0|                  |
    |     Keystone     |192.168.77.50  |     +------------------+
    |      Glance      |---------------+
    |     Nova API     |eth0           |     +------------------+
    |    Cinder API    |               | eth0|   [ compute0 ]   |
    +------------------+               +-----+   Nova Compute   |
                                192.168.77.51|                  |
                                             +------------------+

控制节点初始化Cinder信息：

.. code::

    # 安装Cinder服务
    [root@controller0 ~(keystone)]# yum install -y openstack-cinder

    # 配置keystone，添加endpoint
    [root@controller0 ~(keystone)]# keystone user-create --tenant service --name cinder --pass servicepassword --enabled true 
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |  email   |                                  |
    | enabled  |               True               |
    |    id    | 6c6438aac109473d92ba22ed64ef7f4a |
    |   name   |              cinder              |
    | tenantId | 9acf83020ae34047b6f1e320c352ae44 |
    | username |              cinder              |
    +----------+----------------------------------+
    [root@controller0 ~(keystone)]# keystone user-role-add --user cinder --tenant service --role admin
    [root@controller0 ~(keystone)]# keystone service-create --name=cinder --type=volume --description="Cinder Service" 
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |          Cinder Service          |
    |   enabled   |               True               |
    |      id     | f9745ca8657f40d188a464c706d1d923 |
    |     name    |              cinder              |
    |     type    |              volume              |
    +-------------+----------------------------------+
    [root@controller0 ~(keystone)]# keystone service-create --name=cinderv2 --type=volumev2 --description="Cinder Service" 
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |          Cinder Service          |
    |   enabled   |               True               |
    |      id     | b11416c99c274ed9872ed5eaffad83b7 |
    |     name    |             cinderv2             |
    |     type    |             volumev2             |
    +-------------+----------------------------------+
    [root@controller0 ~(keystone)]# export cinder_api=192.168.77.50
    [root@controller0 ~(keystone)]#  keystone endpoint-create --region RegionOne \
    --service cinder \
    --publicurl "http://$cinder_api:8776/v1/\$(tenant_id)s" \
    --internalurl "http://$cinder_api:8776/v1/\$(tenant_id)s" \
    --adminurl "http://$cinder_api:8776/v1/\$(tenant_id)s" 
    +-------------+--------------------------------------------+
    |   Property  |                   Value                    |
    +-------------+--------------------------------------------+
    |   adminurl  | http://192.168.77.50:8776/v1/$(tenant_id)s |
    |      id     |      073dafcb7ee049cb8bfd3ebbe149dbc0      |
    | internalurl | http://192.168.77.50:8776/v1/$(tenant_id)s |
    |  publicurl  | http://192.168.77.50:8776/v1/$(tenant_id)s |
    |    region   |                 RegionOne                  |
    |  service_id |      f9745ca8657f40d188a464c706d1d923      |
    +-------------+--------------------------------------------+
    [root@controller0 ~(keystone)]# keystone endpoint-create --region RegionOne \
    --service cinderv2 \
    --publicurl "http://$cinder_api:8776/v2/\$(tenant_id)s" \
    --internalurl "http://$cinder_api:8776/v2/\$(tenant_id)s" \
    --adminurl "http://$cinder_api:8776/v2/\$(tenant_id)s" 
    +-------------+--------------------------------------------+
    |   Property  |                   Value                    |
    +-------------+--------------------------------------------+
    |   adminurl  | http://192.168.77.50:8776/v2/$(tenant_id)s |
    |      id     |      3f00de1ec9474183971ba3c1c0d35c7d      |
    | internalurl | http://192.168.77.50:8776/v2/$(tenant_id)s |
    |  publicurl  | http://192.168.77.50:8776/v2/$(tenant_id)s |
    |    region   |                 RegionOne                  |
    |  service_id |      b11416c99c274ed9872ed5eaffad83b7      |
    +-------------+--------------------------------------------+

    # 添加数据库
    [root@controller0 ~(keystone)]# mysql -u root -p 
    Enter password:
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 16
    Server version: 5.5.40-MariaDB-wsrep MariaDB Server, wsrep_25.11.r4026

    Copyright (c) 2000, 2014, Oracle, Monty Program Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> create database cinder; 
    Query OK, 1 row affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on cinder.* to cinder@'localhost' identified by 'password'; 
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on cinder.* to cinder@'%' identified by 'password'; 
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> flush privileges; 
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> exit 
    Bye

    # 配置cinder参数
    [root@controller0 ~(keystone)]# mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.org
    [root@controller0 ~(keystone)]# vi /etc/cinder/cinder.conf
    [DEFAULT]
    state_path=/var/lib/cinder
    api_paste_config=api-paste.ini
    enable_v1_api=true
    rootwrap_config=/etc/cinder/rootwrap.conf
    auth_strategy=keystone
    # specify RabbitMQ server
    rabbit_host=192.168.77.50
    rabbit_port=5672
    # specify RabbitMQ user for auth
    rabbit_userid=guest
    # specify RabbitMQ user's password above
    rabbit_password=password
    rpc_backend=rabbit
    scheduler_driver=cinder.scheduler.filter_scheduler.FilterScheduler
    volume_manager=cinder.volume.manager.VolumeManager
    volume_api_class=cinder.volume.api.API
    volumes_dir=$state_path/volumes
    # auth info for MariaDB

    [database]
    connection=mysql://cinder:password@192.168.77.50/cinder
    # auth info for Keystone
    [keystone_authtoken]
    auth_host=192.168.77.50
    auth_port=35357
    auth_protocol=http
    admin_user=cinder
    admin_password=servicepassword
    admin_tenant_name=service

    # 启用服务
    [root@controller0 ~(keystone)]# chmod 640 /etc/cinder/cinder.conf 
    [root@controller0 ~(keystone)]# chgrp cinder /etc/cinder/cinder.conf 
    [root@controller0 ~(keystone)]# cinder-manage db sync 
    [root@controller0 ~(keystone)]# for service in api scheduler; do
    systemctl start openstack-cinder-$service
    systemctl enable openstack-cinder-$service
    done 
    [root@controller0 ~(keystone)]# cinder-manage service list
    Binary           Host                                 Zone             Status     State Updated At
    cinder-scheduler controller0                          nova             enabled    :-)   None

配置Cinder节点：

.. code::

    # 安装Cinder服务
    [root@cinder0 ~(keystone)]# yum install -y openstack-cinder
    [root@cinder0 ~]# mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.org 
    [root@cinder0 ~]# vi /etc/cinder/cinder.conf
    [DEFAULT]
    state_path=/var/lib/cinder
    api_paste_config=api-paste.ini
    enable_v1_api=true
    osapi_volume_listen=0.0.0.0
    osapi_volume_listen_port=8776
    rootwrap_config=/etc/cinder/rootwrap.conf
    auth_strategy=keystone
    # specify Glance server
    glance_host=192.168.77.50
    glance_port=9292
    # specify RabbitMQ server
    rabbit_host=192.168.77.50
    rabbit_port=5672
    # RabbitMQ user for auth
    rabbit_userid=guest
    # RabbitMQ user's password for auth
    rabbit_password=password
    rpc_backend=rabbit
    # specify iSCSI target (it's just the own IP)
    iscsi_ip_address=192.168.77.60
    iscsi_port=3260
    iscsi_helper=tgtadm
    scheduler_driver=cinder.scheduler.filter_scheduler.FilterScheduler
    volume_manager=cinder.volume.manager.VolumeManager
    volume_api_class=cinder.volume.api.API
    volumes_dir=$state_path/volumes
    # auth info for MariaDB

    [database]
    connection=mysql://cinder:password@192.168.77.50/cinder
    # auth info for Keystone
    [keystone_authtoken]
    auth_host=192.168.77.50
    auth_port=35357
    auth_protocol=http
    admin_user=cinder
    admin_password=servicepassword
    admin_tenant_name=service

    # 启用服务
    [root@cinder0 ~]# chmod 640 /etc/cinder/cinder.conf
    [root@cinder0 ~]# chgrp cinder /etc/cinder/cinder.conf 
    [root@cinder0 ~]# systemctl start openstack-cinder-volume 
    [root@cinder0 ~]# systemctl enable openstack-cinder-volume
    ln -s '/usr/lib/systemd/system/openstack-cinder-volume.service' '/etc/systemd/system/multi-user.target.wants/openstack-cinder-volume.service'
    [root@cinder0 ~]# cinder-manage service list 
    Binary           Host                                 Zone             Status     State Updated At
    cinder-scheduler controller0                          nova             enabled    :-)   2015-07-21 03:29:39
    cinder-volume    cinder0                              nova             enabled    :-)   None

配置LVM后端
~~~~~~~~~~~~

存储节点配置

.. code::

    # 创建PV
    [root@cinder0 ~]# pvcreate /dev/sdb
      Physical volume "/dev/sdb" successfully created
    [root@cinder0 ~]# pvcreate /dev/sdb1
      Device /dev/sdb1 not found (or ignored by filtering).
    [root@cinder0 ~]# pvdisplay 
      "/dev/sdb" is a new physical volume of "20.00 GiB"
      --- NEW Physical volume ---
      PV Name               /dev/sdb
      VG Name               
      PV Size               20.00 GiB
      Allocatable           NO
      PE Size               0   
      Total PE              0
      Free PE               0
      Allocated PE          0
      PV UUID               lDpf6L-zPJT-6Uth-lcPA-KtAS-TYNS-B5LH4c
      
    [root@cinder0 ~]# vgcreate -s 32M vg_volume01 /dev/sdb
      Volume group "vg_volume01" successfully created
    [root@cinder0 ~]# vgdisplay 
      --- Volume group ---
      VG Name               vg_volume01
      System ID             
      Format                lvm2
      Metadata Areas        1
      Metadata Sequence No  1
      VG Access             read/write
      VG Status             resizable
      MAX LV                0
      Cur LV                0
      Open LV               0
      Max PV                0
      Cur PV                1
      Act PV                1
      VG Size               19.97 GiB
      PE Size               32.00 MiB
      Total PE              639
      Alloc PE / Size       0 / 0   
      Free  PE / Size       639 / 19.97 GiB
      VG UUID               IYI8rR-d0u4-p58f-h1Bp-afAW-EPRK-21qSdv

    # 修改cinder配置
    [root@cinder0 ~]# vi /etc/cinder/cinder.conf
    # 在DEFAULT段中添加
    enabled_backends = lvm
    # 在末尾添加
    [lvm]
    iscsi_helper = lioadm
    # volume group name just created
    volume_group = vg_volume01
    # IP address of Storage Node
    iscsi_ip_address = 192.168.77.60
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volumes_dir = $state_path/volumes
    iscsi_protocol = iscsi

    # 重启服务
    [root@cinder0 ~]# systemctl restart openstack-cinder-volume

计算节点配置，在所有计算节点中都要配置

.. code::

    [root@controller0 ~]# vi /etc/nova/nova.conf
    # add follows into [DEFAULT] section
    osapi_volume_listen=0.0.0.0
    volume_api_class=nova.volume.cinder.API
    [root@controller0 ~]# systemctl restart openstack-nova-compute 

创建测试磁盘

.. code::

    # 在任意计算节点中都可执行cinder命令创建磁盘
    [root@controller0 ~]# echo "export OS_VOLUME_API_VERSION=2" >> ~/keystonerc 
    [root@controller0 ~]# source admin_keystone
    [root@controller0 ~(keystone)]# cinder create --display_name disk01 10 
    +---------------------------------------+--------------------------------------+
    |                Property               |                Value                 |
    +---------------------------------------+--------------------------------------+
    |              attachments              |                  []                  |
    |           availability_zone           |                 nova                 |
    |                bootable               |                false                 |
    |          consistencygroup_id          |                 None                 |
    |               created_at              |      2015-07-27T08:24:32.000000      |
    |              description              |                 None                 |
    |               encrypted               |                False                 |
    |                   id                  | 7a974afe-a71a-479f-b63d-b208daae1707 |
    |                metadata               |                  {}                  |
    |              multiattach              |                False                 |
    |                  name                 |                disk01                |
    |         os-vol-host-attr:host         |                 None                 |
    |     os-vol-mig-status-attr:migstat    |                 None                 |
    |     os-vol-mig-status-attr:name_id    |                 None                 |
    |      os-vol-tenant-attr:tenant_id     |   c0c4e7b797bb41798202b55872fba074   |
    |   os-volume-replication:driver_data   |                 None                 |
    | os-volume-replication:extended_status |                 None                 |
    |           replication_status          |               disabled               |
    |                  size                 |                  10                  |
    |              snapshot_id              |                 None                 |
    |              source_volid             |                 None                 |
    |                 status                |               creating               |
    |                user_id                |   cf11b4425218431991f095c2f58578a0   |
    |              volume_type              |                 None                 |
    +---------------------------------------+--------------------------------------+
    [root@controller0 ~(keystone)]# cinder list
    +--------------------------------------+-----------+--------+------+-------------+----------+-------------+
    |                  ID                  |   Status  |  Name  | Size | Volume Type | Bootable | Attached to |
    +--------------------------------------+-----------+--------+------+-------------+----------+-------------+
    | 7a974afe-a71a-479f-b63d-b208daae1707 | available | disk01 |  10  |     None    |  false   |             |
    +--------------------------------------+-----------+--------+------+-------------+----------+-------------+

存储节点上查看

.. code::

    [root@cinder0 ~]# lvdisplay 
      --- Logical volume ---
      LV Path                /dev/vg_volume01/volume-7a974afe-a71a-479f-b63d-b208daae1707
      LV Name                volume-7a974afe-a71a-479f-b63d-b208daae1707
      VG Name                vg_volume01
      LV UUID                Pp91xd-Kj0M-J5eI-tUXY-0iMH-MdJ6-PryIq7
      LV Write Access        read/write
      LV Creation host, time cinder0, 2015-07-27 16:24:33 +0800
      LV Status              available
      # open                 0
      LV Size                10.00 GiB
      Current LE             320
      Segments               1
      Allocation             inherit
      Read ahead sectors     auto
      - currently set to     8192
      Block device           253:0

计算节点上附加磁盘到虚拟机

.. code::

    [root@controller0 ~(keystone)]# nova list 
    +----------------+----------+---------+------------+-------------+-----------------------+
    | ID             | Name     | Status  | Task State | Power State | Networks              |
    +----------------+----------+---------+------------+-------------+-----------------------+
    | 16971b4c-c901- | CentOS_7 | SHUTOFF | -          | Shutdown    | int_net=192.168.100.4 |
    +----------------+----------+---------+------------+-------------+-----------------------+

    [root@controller0 ~(keystone)]# nova volume-attach CentOS_7 7a974afe-a71a-479f-b63d-b208daae1707 auto 
    +----------+--------------------------------------+
    | Property | Value                                |
    +----------+--------------------------------------+
    | device   | /dev/vdb                             |
    | id       | 7a974afe-a71a-479f-b63d-b208daae1707 |
    | serverId | 16971b4c-c901-4e95-8334-b2ff36b99633 |
    | volumeId | 7a974afe-a71a-479f-b63d-b208daae1707 |
    +----------+--------------------------------------+

    # the status of attached disk turns "in-use" like follows
    [root@controller0 ~(keystone)]# cinder list 

配置NFS、Glusterfs混合后端
~~~~~~~~~~~~~~~~~~~~~~~~~~~

此处使用NFS、Glusterfs混合后端，也可根据实际需求添加LVM后端或者使用三者之一。

.. code::

                                              +------------------+                +------------------+
                                 192.168.77.60|   [ cinder0 ]    |  192.168.77.100|                  |
    +------------------+                +-----+   Cinder-Volume  |     +----------+   GlusterFS #1   |
    |  [ controller0 ] |                | eth0|                  |     |      eth0| gfs01.lofyer.org |
    |     Keystone     |192.168.77.50   |     +------------------+     |          +------------------+
    |      Glance      |----------------+------------------------------+
    |     Nova API     |eth0            |     +------------------+     |                +------------------+
    |    Cinder API    |                | eth0|   [ compute0 ]   |     |            eth0|                  |
    +------------------+                +-----+   Nova Compute   |     +----------------+   GlusterFS #2   |
                                 192.168.77.51|                  |     |  192.168.77.101| gfs02.lofyer.org |
                                              +------------------+     |                +------------------+
                                                                       |
                                                                       |          +------------------+
                                                                       |      eth0|                  |
                                                                       +----------+        NFS       |
                                                                    192.168.77.110|  nfs.lofyer.org  |
                                                                                  +------------------+

存储节点配置

.. code::

    # 安装软件包
    [root@cinder0 ~]# yum -y install nfs-utils glusterfs glusterfs

    # 修改cinder配置
    [root@cinder0 ~]# vi /etc/cinder/cinder.conf
    # 于DEFAULT段中添加
    enabled_backends=nfs,glusterfs
    # 于末尾添加
    [nfs]
    volume_driver = cinder.volume.drivers.nfs.NfsDriver
    volume_backend_name = NFS
    nfs_shares_config = /etc/cinder/nfs_shares
    nfs_mount_point_base = $state_path/nfs
    [glusterfs]
    volume_driver = cinder.volume.drivers.glusterfs.GlusterfsDriver
    volume_backend_name = GlusterFS
    glusterfs_shares_config = /etc/cinder/glusterfs_shares
    glusterfs_mount_point_base = $state_path/glusterfs

    # 修改存储信息
    [root@cinder0 ~]# vi /etc/cinder/nfs_shares
    # 指定NFS存储路径
    nfs.lofyer.org:/storage
    [root@cinder0 ~]# vi /etc/cinder/glusterfs_shares
    # 指定Glusterfs存储路径
    gfs01.lofyer.org:/vol_replica
    [root@cinder0 ~]# chmod 640 /etc/cinder/nfs_shares 
    [root@cinder0 ~]# chgrp cinder /etc/cinder/nfs_shares 
    [root@cinder0 ~]# chmod 640 /etc/cinder/glusterfs_shares 
    [root@cinder0 ~]# chgrp cinder /etc/cinder/glusterfs_shares 
    [root@cinder0 ~]# systemctl restart openstack-cinder-volume 

计算节点配置，所有计算节点都要配置

.. code::

    # 安装软件包
    [root@controller0 ~]# yum --enablerepo=epel -y install nfs-utils glusterfs glusterfs-fuse
    [root@controller0 ~]# vi /etc/nova/nova.conf
    # 于DEFAULT段中添加
    osapi_volume_listen=0.0.0.0
    volume_api_class=nova.volume.cinder.API
    # 重启计算服务
    [root@controller0 ~]# systemctl restart openstack-nova-compute 

添加卷种类

.. code::

    # 在任意计算节点中都可执行cinder命令创建磁盘
    [root@controller0 ~]# echo "export OS_VOLUME_API_VERSION=2" >> ~/keystonerc 
    [root@controller0 ~]# source admin_keystone
    [root@controller0 ~(keystone)]# cinder type-create nfs 
    +--------------------------------------+------+
    |                  ID                  | Name |
    +--------------------------------------+------+
    | 7ac3a255-cf70-498d-97d8-2a7fcdd84d2c | nfs  |
    +--------------------------------------+------+

    [root@controller0 ~(keystone)]# cinder type-create glusterfs 
    +--------------------------------------+-----------+
    |                  ID                  |    Name   |
    +--------------------------------------+-----------+
    | e2608bee-cc52-48e8-ba72-b94124f36a57 | glusterfs |
    +--------------------------------------+-----------+

    [root@controller0 ~(keystone)]# cinder type-list 
    +--------------------------------------+-----------+
    |                  ID                  |    Name   |
    +--------------------------------------+-----------+
    | 7ac3a255-cf70-498d-97d8-2a7fcdd84d2c |    nfs    |
    | e2608bee-cc52-48e8-ba72-b94124f36a57 | glusterfs |
    +--------------------------------------+-----------+

添加存储种类别名

.. code::

    [root@controller0 ~(keystone)]# cinder type-key nfs set volume_backend_name=NFS 
    [root@controller0 ~(keystone)]# cinder type-key glusterfs set volume_backend_name=GlusterFS 
    [root@controller0 ~(keystone)]# cinder extra-specs-list 
    +--------------------------------------+-----------+----------------------------------------+
    |                  ID                  |    Name   |              extra_specs               |
    +--------------------------------------+-----------+----------------------------------------+
    | 7ac3a255-cf70-498d-97d8-2a7fcdd84d2c |    nfs    |    {u'volume_backend_name': u'NFS'}    |
    | e2608bee-cc52-48e8-ba72-b94124f36a57 | glusterfs | {u'volume_backend_name': u'GlusterFS'} |
    +--------------------------------------+-----------+----------------------------------------+

添加磁盘

.. code::

    [root@controller0 ~(keystone)]# cinder create --display_name disk_nfs --volume-type nfs 10 
    +---------------------------------------+--------------------------------------+
    |                Property               |                Value                 |
    +---------------------------------------+--------------------------------------+
    |              attachments              |                  []                  |
    |           availability_zone           |                 nova                 |
    |                bootable               |                false                 |
    |          consistencygroup_id          |                 None                 |
    |               created_at              |      2015-06-20T18:23:23.000000      |
    |              description              |                 None                 |
    |               encrypted               |                False                 |
    |                   id                  | 1e92b9ca-20d7-4f63-881e-dea3f8b6b523 |
    |                metadata               |                  {}                  |
    |              multiattach              |                False                 |
    |                  name                 |               disk_nfs               |
    |         os-vol-host-attr:host         |                 None                 |
    |     os-vol-mig-status-attr:migstat    |                 None                 |
    |     os-vol-mig-status-attr:name_id    |                 None                 |
    |      os-vol-tenant-attr:tenant_id     |   98ea1b896d3a48438922c0dfa9f6bc52   |
    |   os-volume-replication:driver_data   |                 None                 |
    | os-volume-replication:extended_status |                 None                 |
    |           replication_status          |               disabled               |
    |                  size                 |                  10                  |
    |              snapshot_id              |                 None                 |
    |              source_volid             |                 None                 |
    |                 status                |               creating               |
    |                user_id                |   704a7f5cf84a479796e10f47c30bb629   |
    |              volume_type              |                 nfs                  |
    +---------------------------------------+--------------------------------------+

    [root@controller0 ~(keystone)]# cinder create --display_name disk_glusterfs --volume-type glusterfs 10 
    +---------------------------------------+--------------------------------------+
    |                Property               |                Value                 |
    +---------------------------------------+--------------------------------------+
    |              attachments              |                  []                  |
    |           availability_zone           |                 nova                 |
    |                bootable               |                false                 |
    |          consistencygroup_id          |                 None                 |
    |               created_at              |      2015-06-20T18:23:49.000000      |
    |              description              |                 None                 |
    |               encrypted               |                False                 |
    |                   id                  | d8dbaed2-e857-4162-baab-0178fbef4593 |
    |                metadata               |                  {}                  |
    |              multiattach              |                False                 |
    |                  name                 |            disk_glusterfs            |
    |         os-vol-host-attr:host         |                 None                 |
    |     os-vol-mig-status-attr:migstat    |                 None                 |
    |     os-vol-mig-status-attr:name_id    |                 None                 |
    |      os-vol-tenant-attr:tenant_id     |   98ea1b896d3a48438922c0dfa9f6bc52   |
    |   os-volume-replication:driver_data   |                 None                 |
    | os-volume-replication:extended_status |                 None                 |
    |           replication_status          |               disabled               |
    |                  size                 |                  10                  |
    |              snapshot_id              |                 None                 |
    |              source_volid             |                 None                 |
    |                 status                |               creating               |
    |                user_id                |   704a7f5cf84a479796e10f47c30bb629   |
    |              volume_type              |              glusterfs               |
    +---------------------------------------+--------------------------------------+

    [root@controller0 ~(keystone)]# cinder list 
    +--------------------------+-----------+----------------+------+-------------+----------+-------------+
    |                  ID      |   Status  |      Name      | Size | Volume Type | Bootable | Attached to |
    +--------------------------+-----------+----------------+------+-------------+----------+-------------+
    | 1e92b9ca-20d7-4f63-881e- | available |    disk_nfs    |  10  |     nfs     |  false   |             |
    | d8dbaed2-e857-4162-baab- | available | disk_glusterfs |  10  |  glusterfs  |  false   |             |
    +--------------------------+-----------+----------------+------+-------------+----------+-------------+

附加磁盘到虚拟机

.. code::

    [root@controller0 ~(keystone)]# nova list 
    +----------------+----------+---------+------------+-------------+-----------------------------------+
    | ID             | Name     | Status  | Task State | Power State | Networks                          |
    +----------------+----------+---------+------------+-------------+-----------------------------------+
    | 2c7a1025-30d6- | CentOS_7 | SHUTOFF | -          | Shutdown    | int_net=192.168.100.3, 10.0.0.201 |
    +----------------+----------+---------+------------+-------------+-----------------------------------+

    [root@controller0 ~(keystone)]# nova volume-attach CentOS_7 1e92b9ca-20d7-4f63-881e-dea3f8b6b523 auto 
    +----------+--------------------------------------+
    | Property | Value                                |
    +----------+--------------------------------------+
    | device   | /dev/vdc                             |
    | id       | 1e92b9ca-20d7-4f63-881e-dea3f8b6b523 |
    | serverId | 2c7a1025-30d6-446a-a4ff-309347b64eca |
    | volumeId | 1e92b9ca-20d7-4f63-881e-dea3f8b6b523 |
    +----------+--------------------------------------+

    [root@controller0 ~(keystone)]# nova volume-attach CentOS_7 d8dbaed2-e857-4162-baab-0178fbef4593 auto 
    +----------+--------------------------------------+
    | Property | Value                                |
    +----------+--------------------------------------+
    | device   | /dev/vdd                             |
    | id       | d8dbaed2-e857-4162-baab-0178fbef4593 |
    | serverId | 2c7a1025-30d6-446a-a4ff-309347b64eca |
    | volumeId | d8dbaed2-e857-4162-baab-0178fbef4593 |
    +----------+--------------------------------------+

    # the status of attached disk turns "in-use" like follows
    [root@controller0 ~(keystone)]# cinder list 
    +--------------------------+--------+----------------+------+-------------+----------+-----------------------------+
    |                  ID      | Status |      Name      | Size | Volume Type | Bootable |             Attached to     |
    +--------------------------+--------+----------------+------+-------------+----------+-----------------------------+
    | 1e92b9ca-20d7-4f63-881e- | in-use |    disk_nfs    |  10  |     nfs     |  false   | 2c7a1025-30d6-446a-a4ff-309 |
    | d8dbaed2-e857-4162-baab- | in-use | disk_glusterfs |  10  |  glusterfs  |  false   | 2c7a1025-30d6-446a-a4ff-309 |
    +--------------------------+--------+----------------+------+-------------+----------+-----------------------------+


配置Swift
----------

在此我们有三个存储节点作为swift backend，swift0作为swift访问入口。

.. code::

                                        |
     +------------------+               |               +-----------------+
     | [ controller0 ]  |192.168.77.50  |  192.168.77.70|    [ swift0 ]   |
     |     Keystone     |---------------+---------------|      Proxy      |
     +------------------+               |               +-----------------+
                                        |
            +---------------------------+--------------------------+
            |                           |                          |
            |192.168.77.71              |192.168.77.72             |192.168.77.73
    +-------+----------+       +--------+---------+       +--------+---------+
    | [ swift-stor0 ]  |       | [ swift-stor1 ]  |       | [ swift-stor2 ]  |
    |                  |-------|                  |-------|                  |
    +------------------+       +------------------+       +------------------+

配置控制节点

.. code::

    # 添加swift用户
    [root@controller0 ~(keystone)]# openstack user create --project service --password servicepassword swift 
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | email      | None                             |
    | enabled    | True                             |
    | id         | 9e19bc053b0f44bdbabf751b279c9afd |
    | name       | swift                            |
    | project_id | 9acf83020ae34047b6f1e320c352ae44 |
    | username   | swift                            |
    +------------+----------------------------------+
    # 赋予swift用户admin角色
    [root@controller0 ~(keystone)]# openstack role add --project service --user swift admin
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | 95c4b8fb8d97424eb52a4e8a00a357e7 |
    | name  | admin                            |
    +-------+----------------------------------+
    # 创建swift服务
    [root@controller0 ~(keystone)]# openstack service create --name swift --description "OpenStack Object Storage" object-store
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | OpenStack Object Storage         |
    | enabled     | True                             |
    | id          | c8d07b32376a4e2780d8ec9b2b836e41 |
    | name        | swift                            |
    | type        | object-store                     |
    +-------------+----------------------------------+
    [root@controller0 ~(keystone)]# export swift_proxy=192.168.77.70
    # 添加endpoint
    root@controller0 ~(keystone)]# openstack endpoint create \
     --publicurl http://$swift_proxy:8080/v1/AUTH_%\(tenant_id\)s \
     --internalurl http://$swift_proxy:8080/v1/AUTH_%\(tenant_id\)s \
     --adminurl http://$swift_proxy:8080 \
     --region RegionOne \
     object-store
    +--------------+-------------------------------------------------+
    | Field        | Value                                           |
    +--------------+-------------------------------------------------+
    | adminurl     | http://192.168.77.70:8080                       |
    | id           | d7e1f6a5f7064f3690181f7bd8922ac4                |
    | internalurl  | http://192.168.77.70:8080/v1/AUTH_%(tenant_id)s |
    | publicurl    | http://192.168.77.70:8080/v1/AUTH_%(tenant_id)s |
    | region       | RegionOne                                       |
    | service_id   | c8d07b32376a4e2780d8ec9b2b836e41                |
    | service_name | swift                                           |
    | service_type | object-store                                    |
    +--------------+-------------------------------------------------+

配置swift代理节点

.. code::

    [root@swift0 ~]# vi /etc/swift/proxy-server.conf
    # 第53行
    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    # 注释如下部分
    #admin_tenant_name = %SERVICE_TENANT_NAME%
    #admin_user = %SERVICE_USER%
    #admin_password = %SERVICE_PASSWORD%
    #
    #identity_uri = http://localhost:35357/
    #auth_uri = http://localhost:5000/
    #
    signing_dir = /tmp/keystone-signing-swift
    # 添加认证
    auth_uri = http://192.168.77.50:5000
    auth_url = http://192.168.77.50:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = swift
    password = servicepassword
    delay_auth_decision = true

    [root@swift0 ~]# vi /etc/swift/swift.conf
    # 更改为任意
    [swift-hash]
    swift_hash_path_suffix = swift_shared_path

    # 创建swift哈希环
    # swift-ring-builder builder-file create <part_power> <replicas> <min_part_hours>
    # Creates <builder_file> with 2^<part_power> partitions and <replicas>.
    # <min_part_hours> is number of hours to restrict moving a partition more
    # than once
    [root@swift0 ~]# swift-ring-builder /etc/swift/account.builder create 12 3 1 
    [root@swift0 ~]# swift-ring-builder /etc/swift/container.builder create 12 3 1 
    [root@swift0 ~]# swift-ring-builder /etc/swift/object.builder create 12 3 1
    [root@swift0 ~]# swift-ring-builder /etc/swift/account.builder add r0z0-192.168.77.71:6002/device0 100 
    Device d0r0z0-192.168.77.71:6002R192.168.77.71:6002/device0_"" with 100.0 weight got id 0
    [root@swift0 ~]# swift-ring-builder /etc/swift/account.builder add r0z0-192.168.77.72:6002/device0 100 
    Device d1r0z0-192.168.77.72:6002R192.168.77.72:6002/device0_"" with 100.0 weight got id 1
    [root@swift0 ~]# swift-ring-builder /etc/swift/account.builder add r0z0-192.168.77.73:6002/device0 100 
    Device d2r0z0-192.168.77.73:6002R192.168.77.73:6002/device0_"" with 100.0 weight got id 2
    [root@swift0 ~]# swift-ring-builder /etc/swift/container.builder add r0z0-192.168.77.71:6001/device0 100 
    Device d0r0z0-192.168.77.71:6001R192.168.77.71:6001/device0_"" with 100.0 weight got id 0
    [root@swift0 ~]# swift-ring-builder /etc/swift/container.builder add r0z0-192.168.77.72:6001/device0 100 
    Device d1r0z0-192.168.77.72:6001R192.168.77.72:6001/device0_"" with 100.0 weight got id 1
    [root@swift0 ~]# swift-ring-builder /etc/swift/container.builder add r0z0-192.168.77.73:6001/device0 100 
    Device d2r0z0-192.168.77.73:6001R192.168.77.73:6001/device0_"" with 100.0 weight got id 2
    [root@swift0 ~]# swift-ring-builder /etc/swift/object.builder add r0z0-192.168.77.71:6000/device0 100
    Device d0r0z0-192.168.77.71:6000R192.168.77.71:6000/device0_"" with 100.0 weight got id 0
    [root@swift0 ~]# swift-ring-builder /etc/swift/object.builder add r0z0-192.168.77.72:6000/device0 100
    Device d1r0z0-192.168.77.72:6000R192.168.77.72:6000/device0_"" with 100.0 weight got id 1
    [root@swift0 ~]# swift-ring-builder /etc/swift/object.builder add r0z0-192.168.77.73:6000/device0 100
    Device d2r0z0-192.168.77.73:6000R192.168.77.73:6000/device0_"" with 100.0 weight got id 2
    [root@swift0 ~]# swift-ring-builder /etc/swift/account.builder rebalance 
    Reassigned 4096 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
    [root@swift0 ~]# swift-ring-builder /etc/swift/container.builder rebalance 
    Reassigned 4096 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
    [root@swift0 ~]# swift-ring-builder /etc/swift/object.builder rebalance
    Reassigned 4096 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
    [root@swift0 ~]# chown swift. /etc/swift/*.gz 
    [root@swift0 ~]# systemctl start memcached openstack-swift-proxy
    [root@swift0 ~]# systemctl enable memcached openstack-swift-proxy 
    ln -s '/usr/lib/systemd/system/memcached.service' '/etc/systemd/system/multi-user.target.wants/memcached.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-proxy.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-proxy.service'

配置存储节点，每个节点都要配置，注意selinux策略，否则会操作失败。

.. code::

    # 禁用selinux
    [root@swift-stor0 ~]# setenforce permissive
    [root@swift-stor0 ~]# vi /etc/selinux/config
    SELINUX=enforcing


    # 安装所需包
    [root@swift-stor0 ~]# yum install -y openstack-swift-account openstack-swift-container openstack-swift-object xfsprogs rsync openssh-clients

    # 格式化存储磁盘为xfs
    [root@swift-stor0 ~]# mkfs.xfs -i size=1024 -s size=4096 /dev/sdb -f
    meta-data=/dev/sdb               isize=1024   agcount=4, agsize=1310720 blks
             =                       sectsz=4096  attr=2, projid32bit=1
             =                       crc=0        finobt=0
    data     =                       bsize=4096   blocks=5242880, imaxpct=25
             =                       sunit=0      swidth=0 blks
    naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
    log      =internal log           bsize=4096   blocks=2560, version=2
             =                       sectsz=4096  sunit=1 blks, lazy-count=1
    realtime =none                   extsz=4096   blocks=0, rtextents=0

    # 创建目录并添加fstab
    [root@swift-stor0 ~]# mkdir -p /srv/node/device0 
    [root@swift-stor0 ~]# mount -o noatime,nodiratime,nobarrier /dev/sdb /srv/node/device0
    [root@swift-stor0 ~]# chown -R swift. /srv/node
    [root@swift-stor0 ~]# vi /etc/fstab 

    # 拷贝hash环文件
    [root@swift-stor0 ~]# scp root@192.168.77.70:/etc/swift/*.gz /etc/swift/
    root@192.168.77.70's password: 
    account.ring.gz                            100% 3862     3.8KB/s   00:00    
    container.ring.gz                          100% 3861     3.8KB/s   00:00    
    object.ring.gz                             100% 3887     3.8KB/s   00:00
    [root@swift-stor0 ~]# chown swift. /etc/swift/*.gz 

    # 修改后缀为proxy设置值
    [root@swift-stor0 ~]# vi /etc/swift/swift.conf
    [swift-hash] swift_hash_path_suffix = swift_shared_path
    # 修改account、container、object服务端口
    [root@swift-stor0 ~]# vi /etc/swift/account-server.conf
    bind_ip = 192.168.77.71
    bind_port = 6002
    [root@swift-stor0 ~]# vi /etc/swift/container-server.conf
    bind_ip = 192.168.77.71
    bind_port = 6002
    [root@swift-stor0 ~]# vi /etc/swift/object-server.conf
    bind_ip = 192.168.77.71
    bind_port = 6000
    # 配置rsync
    [root@swift-stor0 ~]# vi /etc/rsyncd.conf
    # 在末尾添加
    # add to the end
     pid file = /var/run/rsyncd.pid
     log file = /var/log/rsyncd.log
     uid = swift
     gid = swift
     address = 192.168.77.71

     [account]
     path            = /srv/node
     read only       = false
     write only      = no
     list            = yes
     incoming chmod  = 0644
     outgoing chmod  = 0644
     max connections = 25
     lock file =     /var/lock/account.lock

     [container]
     path            = /srv/node
     read only       = false
     write only      = no
     list            = yes
     incoming chmod  = 0644
     outgoing chmod  = 0644
     max connections = 25
     lock file =     /var/lock/container.lock

     [object]
     path            = /srv/node
     read only       = false
     write only      = no
     list            = yes
     incoming chmod  = 0644
     outgoing chmod  = 0644
     max connections = 25
     lock file =     /var/lock/object.lock

     [swift_server]
     path            = /etc/swift
     read only       = true
     write only      = no
     list            = yes
     incoming chmod  = 0644
     outgoing chmod  = 0644
     max connections = 5
     lock file =     /var/lock/swift_server.lock

     # 开启服务
    [root@swift-stor0 ~]# systemctl start rsyncd 
    [root@swift-stor0 ~]# systemctl enable rsyncd 
    ln -s '/usr/lib/systemd/system/rsyncd.service' '/etc/systemd/system/multi-user.target.wants/rsyncd.service'
    [root@swift-stor0 ~]# for ringtype in account container object; do 
         systemctl start openstack-swift-$ringtype
         systemctl enable openstack-swift-$ringtype
         for service in replicator updater auditor; do
             if [ $ringtype != 'account' ] || [ $service != 'updater' ]; then
                 systemctl start openstack-swift-$ringtype-$service
                 systemctl enable openstack-swift-$ringtype-$service
             fi
         done
     done 
    ln -s '/usr/lib/systemd/system/openstack-swift-account.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-account.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-account-replicator.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-account-replicator.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-account-auditor.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-account-auditor.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-container.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-container.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-container-replicator.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-container-replicator.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-container-updater.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-container-updater.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-container-auditor.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-container-auditor.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-object.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-object.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-object-replicator.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-object-replicator.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-object-updater.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-object-updater.service'
    ln -s '/usr/lib/systemd/system/openstack-swift-object-auditor.service' '/etc/systemd/system/multi-user.target.wants/openstack-swift-object-auditor.service'

添加用户

.. code::

    # 添加工程
    [root@controller0 ~(keystone)]# openstack project create --description "Swift Service Project" swiftservice 
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | Swift Service Project            |
    | enabled     | True                             |
    | id          | 65e95f038a654db3a2ee0ae93daaf2b3 |
    | name        | swiftservice                     |
    +-------------+----------------------------------+
    # 创建客户端角色
    [root@controller0 ~(keystone)]# openstack role create swiftoperator 
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | dbcb21e865ae435da39507bea32fe312 |
    | name  | swiftoperator                    |
    +-------+----------------------------------+
    # 添加工程用户
    [root@controller0 ~(keystone)]# openstack user create --project swiftservice --password userpassword user01 
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | email      | None                             |
    | enabled    | True                             |
    | id         | c1d2e56e9d72447096221a4542e17e58 |
    | name       | user01                           |
    | project_id | 65e95f038a654db3a2ee0ae93daaf2b3 |
    | username   | user01                           |
    +------------+----------------------------------+
    # 赋予用户客户端角色
    [root@controller0 ~(keystone)]# openstack role add --project swiftservice --user user01 swiftoperator 
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | dbcb21e865ae435da39507bea32fe312 |
    | name  | swiftoperator                    |
    +-------+----------------------------------+

客户端操作

.. code::

    # 安装所需包
    [root@controller0 ~(keystone)]# yum install -y python-keystoneclient python-swiftclient

    # 添加keystone_rc文件
    [root@controller0 ~(swift)]# cat swift_keystone 
    export OS_PROJECT_DOMAIN_ID=default
    export OS_USER_DOMAIN_ID=default
    export OS_PROJECT_NAME=swiftservice
    export OS_TENANT_NAME=swiftservice
    export OS_USERNAME=user01
    export OS_PASSWORD=userpassword
    export OS_AUTH_URL=http://192.168.77.50:35357/v2.0
    export PS1='[\u@\h \W(swift)]\$ '

    [root@controller0 ~(keystone)]# source swift_keystone
    [root@controller0 ~(swift)]# swift stat
            Account: AUTH_65e95f038a654db3a2ee0ae93daaf2b3
         Containers: 0
            Objects: 0
              Bytes: 0
    X-Put-Timestamp: 1438064191.81736
         Connection: keep-alive
        X-Timestamp: 1438064191.81736
         X-Trans-Id: txe9426af85d424b7185086-0055b71e3e
       Content-Type: text/plain; charset=utf-8

    # 创建测试容器
    [root@controller0 ~(swift)]# swift post test_container
    [root@controller0 ~(swift)]# swift list
    test_container
    # 上传文件
    [root@controller0 ~(swift)]# swift upload test_container admin_keystone
    [root@controller0 ~(swift)]# swift list test_container
    admin_keystone
    # 下载文件
    [root@controller0 ~(swift)]# swift download test_container admin_keystone > admin_keystone.txt


配置Heat（可选）
----------------

Heat服务即用于提供orchestration，在此我们使用neutron0当作heat节点。

.. code::

                                    |
  +------------------+              |              +-----------------------+
  | [ controller0 ]  |              |              |      [ neutron0 ]     |
  |     Keystone     |192.168.77.50 | 192.168.77.30|    DHCP,L3,L2 Agent   |
  |      Glance      |--------------+--------------|     Metadata Agent    |
  |     Nova API     |eth0          |           th0|    Heat API,API-CFN   |
  |  Neutron Server  |              |              |       Heat Engine     |
  +------------------+              |              +-----------------------+
                                eth0|192.168.77.51
                         +--------------------+
                         |    [ compute0 ]    |
                         |    Nova Compute    |
                         |      L2 Agent      |
                         +--------------------+

控制节点配置

.. code::

    # 安装所需包
    [root@controller0 ~]# yum install -y openstack-heat-common

    # 添加用户，创建endpoint
    [root@controller0 ~(keystone)]# openstack user create --project service --password servicepassword heat
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | email      | None                             |
    | enabled    | True                             |
    | id         | d18e4d820a5d4384a676bb0064448c09 |
    | name       | heat                             |
    | project_id | 9acf83020ae34047b6f1e320c352ae44 |
    | username   | heat                             |
    +------------+----------------------------------+
    [root@controller0 ~(keystone)]# openstack role add --project service --user heat admin 
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | 95c4b8fb8d97424eb52a4e8a00a357e7 |
    | name  | admin                            |
    +-------+----------------------------------+
    [root@controller0 ~(keystone)]# openstack role create heat_stack_owner 
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | 91ef76ae6d6c4fc0b908cf7416055da0 |
    | name  | heat_stack_owner                 |
    +-------+----------------------------------+
    [root@controller0 ~(keystone)]# openstack role create heat_stack_user 
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | 0bf5c075fcb7448d844699131c8008fb |
    | name  | heat_stack_user                  |
    +-------+----------------------------------+
    [root@controller0 ~(keystone)]# openstack role add --project admin --user admin heat_stack_owner
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | 91ef76ae6d6c4fc0b908cf7416055da0 |
    | name  | heat_stack_owner                 |
    +-------+----------------------------------+
    [root@controller0 ~(keystone)]# openstack service create --name heat --description "Openstack Orchestration" orchestration 
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | Openstack Orchestration          |
    | enabled     | True                             |
    | id          | fc32f278bf3243d9b6b6c7a5cbf13135 |
    | name        | heat                             |
    | type        | orchestration                    |
    +-------------+----------------------------------+
    [root@controller0 ~(keystone)]# openstack service create --name heat-cfn --description "Openstack Orchestration" cloudformation
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | Openstack Orchestration          |
    | enabled     | True                             |
    | id          | f3334830bc254e6491b5f7de8f7d9a58 |
    | name        | heat-cfn                         |
    | type        | cloudformation                   |
    +-------------+----------------------------------+
    [root@controller0 ~(keystone)]# heat_api=192.168.77.30
    [root@controller0 ~(keystone)]# openstack endpoint create \
    --publicurl http://$heat_api:8004/v1/%\(tenant_id\)s \
    --internalurl http://$heat_api:8004/v1/%\(tenant_id\)s \
    --adminurl http://$heat_api:8004/v1/%\(tenant_id\)s \
    --region RegionOne \
    orchestration
    +--------------+--------------------------------------------+
    | Field        | Value                                      |
    +--------------+--------------------------------------------+
    | adminurl     | http://192.168.77.30:8004/v1/%(tenant_id)s |
    | id           | 1a91dbfe0913466db38595c1b4f0bd59           |
    | internalurl  | http://192.168.77.30:8004/v1/%(tenant_id)s |
    | publicurl    | http://192.168.77.30:8004/v1/%(tenant_id)s |
    | region       | RegionOne                                  |
    | service_id   | fc32f278bf3243d9b6b6c7a5cbf13135           |
    | service_name | heat                                       |
    | service_type | orchestration                              |
    +--------------+--------------------------------------------+
    [root@controller0 ~(keystone)]# openstack endpoint create \
    --publicurl http://$heat_api:8000/v1 \
    --internalurl http://$heat_api:8000/v1 \
    --adminurl http://$heat_api:8000/v1 \
    --region RegionOne \
    cloudformation 
    +--------------+----------------------------------+
    | Field        | Value                            |
    +--------------+----------------------------------+
    | adminurl     | http://192.168.77.30:8000/v1     |
    | id           | e40570fd8201420fbea42abf3231d8eb |
    | internalurl  | http://192.168.77.30:8000/v1     |
    | publicurl    | http://192.168.77.30:8000/v1     |
    | region       | RegionOne                        |
    | service_id   | f3334830bc254e6491b5f7de8f7d9a58 |
    | service_name | heat-cfn                         |
    | service_type | cloudformation                   |
    +--------------+----------------------------------+

    # 注意末尾的信息，同时手动更新/etc/heat/heat.conf文件
    [root@controller0 ~(keystone)]# heat-keystone-setup-domain \
    --stack-user-domain-name heat_user_domain \
    --stack-domain-admin heat_domain_admin \
    --stack-domain-admin-password domainpassword 

    Please update your heat.conf with the following in [DEFAULT]

    stack_user_domain_id=55654da575f048869a9128db12d26f27
    stack_domain_admin=heat_domain_admin
    stack_domain_admin_password=domainpassword
    [root@controller0 ~(keystone)]# vim /etc/heat/heat.conf 

    # 创建数据库
    [root@controller0 ~]# mysql -uroot -p
    Enter password: 
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 11
    Server version: 5.5.40-MariaDB-wsrep MariaDB Server, wsrep_25.11.r4026

    Copyright (c) 2000, 2014, Oracle, MariaDB Corporation Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> create database heat; 
    Query OK, 1 row affected (0.00 sec)

    MariaDB [(none)]> grant all privileges on heat.* to heat@'localhost' identified by 'password'; 
    Query OK, 0 rows affected (0.00 sec)

    MariaDB [(none)]> grant all privileges on heat.* to heat@'%' identified by 'password'; 
    Query OK, 0 rows affected (0.00 sec)

    MariaDB [(none)]> flush privileges; 
    Query OK, 0 rows affected (0.00 sec)

    MariaDB [(none)]> exit
    Bye

heat节点配置

.. code::

    [root@neutron0 ~]# mv /etc/heat/heat.conf /etc/heat/heat.conf.org 
    [root@neutron0 ~]# vi /etc/heat/heat.conf
    [DEFAULT]
    deferred_auth_method = trusts
    trusts_delegated_roles = heat_stack_owner
    # Heat installed server
    heat_metadata_server_url = http://192.168.77.30:8000
    heat_waitcondition_server_url = http://192.168.77.30:8000/v1/waitcondition
    heat_watch_server_url = http://192.168.77.30:8003
    heat_stack_user_role = heat_stack_user
    stack_user_domain_id=55654da575f048869a9128db12d26f27
    stack_domain_admin=heat_domain_admin
    stack_domain_admin_password=domainpassword
    rpc_backend = rabbit
    [database]
    # MariaDB connection info
    connection = mysql://heat:password@192.168.77.50/heat
    # RabbitMQ connection info
    [oslo_messaging_rabbit]
    rabbit_host = 192.168.77.50
    rabbit_port = 5672
    rabbit_userid = guest
    rabbit_password = password
    [ec2authtoken]
    # Keystone server
    auth_uri = http://192.168.77.50:35357/v2.0
    [heat_api]
    bind_host = 0.0.0.0
    bind_port = 8004
    [heat_api_cfn]
    bind_host = 0.0.0.0
    bind_port = 8000
    [keystone_authtoken]
    # Keystone auth info
    auth_host = 192.168.77.50
    auth_port = 35357
    auth_protocol = http
    auth_uri = http://192.168.77.50:35357/v2.0
    # Heat admin user
    admin_user = heat
    # Heat admin user's password
    admin_password = servicepassword
    # Heat admin user's tenant
    admin_tenant_name = service

    [root@neutron0 ~]# chgrp heat /etc/heat/heat.conf 
    [root@neutron0 ~]# chmod 640 /etc/heat/heat.conf 
    [root@neutron0 ~]# chmod 640 /etc/heat/heat.conf 
    [root@neutron0 ~]# heat-manage db_sync
    [root@neutron0 ~]# systemctl start openstack-heat-api openstack-heat-api-cfn openstack-heat-engine 
    [root@neutron0 ~]# systemctl enable openstack-heat-api openstack-heat-api-cfn openstack-heat-engine
    ln -s '/usr/lib/systemd/system/openstack-heat-api.service' '/etc/systemd/system/multi-user.target.wants/openstack-heat-api.service'
    ln -s '/usr/lib/systemd/system/openstack-heat-api-cfn.service' '/etc/systemd/system/multi-user.target.wants/openstack-heat-api-cfn.service'
    ln -s '/usr/lib/systemd/system/openstack-heat-engine.service' '/etc/systemd/system/multi-user.target.wants/openstack-heat-engine.service'

使用heat创建虚拟机

.. code::

    [root@controller0 ~]# vi sample-stack.yml
    heat_template_version: 2014-10-16

    description: Heat Sample Template

    parameters:
      ImageID:
       type: string
       description: Image used to boot a server
      NetID:
       type: string
       description: Network ID for the server

    resources:
     server1:
      type: OS::Nova::Server
      properties:
       name: "Heat_Deployed_Server"
       image: { get_param: ImageID }
       flavor: "m1.small"
       networks:
       - network: { get_param: NetID }

    outputs:
     server1_private_ip:
      description: IP address of the server in the private network
      value: { get_attr: [ server1, first_address ] }

    [root@controller0 ~]# source admin_keystone
    [root@controller0 ~(keystone)]# glance image-list
    +--------------------------------------+---------+-------------+------------------+-------------+--------+
    | ID                                   | Name    | Disk Format | Container Format | Size        | Status |
    +--------------------------------------+---------+-------------+------------------+-------------+--------+
    | fc9da1a8-0418-4ae8-8e67-22a7975f7a0b | centos6 | qcow2       | bare             | 10739318784 | active |
    +--------------------------------------+---------+-------------+------------------+-------------+--------+
    [root@controller0 ~(keystone)]# Int_Net_ID=`neutron net-list | grep int_net | awk '{ print $2 }'`
    [root@controller0 ~(keystone)]# heat stack-create -f sample-stack.yml -P "ImageID=centos6;NetID=$Int_Net_ID" Sample-Stack 
    +--------------------------------------+--------------+--------------------+----------------------+
    | id                                   | stack_name   | stack_status       | creation_time        |
    +--------------------------------------+--------------+--------------------+----------------------+
    | 35b5c1e6-ec84-4bf8-9e77-1946bcf8d09b | Sample-Stack | CREATE_IN_PROGRESS | 2015-07-31T03:06:27Z |
    +--------------------------------------+--------------+--------------------+----------------------+

配置Ceilometer（可选）
----------------------

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

Q：如何校验密码是否正确？

A：keystone --os-username=neutron --os-password=servicepassword --os-auth-url=http://localhost:35357/v2.0 token-get

Q：管理界面Swift不能删除目录？

A：使用命令 swift delete public_container aaa/ 进行删除。

Q： Neutron 网络快速开始？

A：参考https://www.ustack.com/blog/neutron_intro/

Q：OpenStack组件间的通信是靠什么？

A：AMQP，比如RabbitMQ、Apache的ActiveMQ，部署时候可以选择，如果对这种消息传输工具有兴趣可以参考 `rabbitmq tutorial <http://www.rabbitmq.com/getstarted.html>`_ 以及 `各种有用的插件（web监视等） <http://www.rabbitmq.com/plugins.html>`_ 。

Q：Swift有什么好用的客户端么？

A：`python-swiftclient <https://github.com/openstack/python-swiftclient>`_ 、 `Gladient Cloud Desktop <http://www.gladient.com/>`_ 、 `Cloudberry <http://www.cloudberrylab.com/>`_ 、 `Cyberduck <http://cyberduck.ch/>`_ 、 `WebDrive <http://www.webdrive.com/>`_ 、 `S3 Browser <http://s3browser.com/>`_ 等。
