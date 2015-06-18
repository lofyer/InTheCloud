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

存储节点：stor0，stor1，swift0

Heat节点：heat0

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
    > --service keystone \
    > --publicurl "http://$my_host:\$(public_port)s/v2.0" \
    > --internalurl "http://$my_host:\$(public_port)s/v2.0" \
    > --adminurl "http://$my_host:\$(admin_port)s/v2.0"
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
    > --service glance \
    > --publicurl "http://$my_host:9292/v1" \
    > --internalurl "http://$my_host:9292/v1" \
    > --adminurl "http://$my_host:9292/v1" 
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
    > --service nova \
    > --publicurl "http://$my_host:\$(compute_port)s/v2/\$(tenant_id)s" \
    > --internalurl "http://$my_host:\$(compute_port)s/v2/\$(tenant_id)s" \
    > --adminurl "http://$my_host:\$(compute_port)s/v2/\$(tenant_id)s" 
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
    identity_uri=http://192.168.77.50:35357
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
    # 新建
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
    # Glance服务信息
    glance_host=192.168.77.50
    glance_port=9292
    glance_protocol=http
    lock_path=/var/lib/nova/tmp
    log_dir=/var/log/nova
    # Memcached服务信息
    memcached_servers=192.168.77.50:11211
    scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
    [database]
    # connection info for MariaDB
    connection=mysql://nova:password@localhost/nova
    [keystone_authtoken]
    # Keystone server's hostname or IP
    auth_host=192.168.77.50
    auth_port=35357
    auth_protocol=http
    auth_version=v2.0
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

    # 启用服务，如果没用使用nova-network，请忽略数组中的network
    [root@controller0 ~]# nova-manage db sync 
    [root@controller0 ~]# for service in api objectstore conductor scheduler cert consoleauth compute network; do
    systemctl start openstack-nova-$service
    systemctl enable openstack-nova-$service
    done

添加镜像
~~~~~~~~~

至此我们可以使用keystone来进行正常的认证了。

.. code::

    [root@controller0 ~]# vi ~/admin_keystone
    export OS_USERNAME=admin
    export OS_PASSWORD=admin
    export OS_TENANT_NAME=admin
    export OS_AUTH_URL=http://192.168.77.50:35357/v2.0/
    export PS1='[\u@\h \W(keystone)]\$ '

    [root@controller0 ~]# source ~/admin_keystone
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

配置Nova Networking（可选）
~~~~~~~~~~~~~~~~~~~~~~~~~~~

启动实例
~~~~~~~~~~

配置Horizon
------------

添加计算节点
------------

.. _neutron:

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
