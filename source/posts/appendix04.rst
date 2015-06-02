================================
附录四  Docker 使用及自建repo
================================

Docker已经越来越流行了（IaaS平台开始支持它，PaaS平台也开始支持它），不介绍它总感觉过不去。

它是基于LXC的容器类型虚拟化技术，从实现上说更类似于chroot，用户空间的信息被很好隔离的同时，又实现了网络相关的分离。它取代LXC的原因，我想是因为其REPO非常丰富，操作上类似git。

另外，它有提供Windows/MacOSX的客户端 boot2docker。

中文入门手册请参考 `Docker中文指南 <http://www.widuu.com/chinese_docker/>`_ ，另外它有一个WebUI `shipyard <https://github.com/shipyard/shipyard>`_ 。

官方repo `https://registry.hub.docker.com/ <https://registry.hub.docker.com/>`_ 。

---------
镜像操作
---------

运行简单命令

.. code::

    docker run ubuntu /bin/echo "Hello world!"

运行交互shell

.. code::
    
    docker run -t -i ubuntu /bin/bash

运行Django程序

.. code::
    
    docker run -d -P training/webapp python app.py

获取container信息

.. code::
    
    docker ps

获取container内部信息

.. code::
    
    docker inspect -f '{{ .NetworkSettings.IPAddress }}' my_container

获取container历史

.. code::
    
    docker log my_container

commit/save/load

.. note:: 保存

    只有commit，对docker做的修改才会保存，形如docker run centos yum install -y nmap不会保存。

.. code::

    docker images
    docker commit $image_id$ myimage
    docker save myimage > myimage.tar
    docker load < myimage.tar

-------------
Registry操作
-------------

登录，默认为DockerHub

.. code::

    docker login 

创建Registry

参考 https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04 以及 http://blog.docker.com/2013/07/how-to-use-your-own-registry/ 。

.. code::

    # 获取docker-registry，从github或者直接 pip install docker-registry
    # git clone https://github.com/dotcloud/docker-registry.git
    # cd docker-registry
    # cp config_sample.yml config.yml
    # pip install -r requirements.txt
    # gunicorn --access-logfile - --log-level debug --debug 
    -b 0.0.0.0:5000 -w 1 wsgi:application
    
push/pull

.. code::

    # docker pull ubuntu
    # docker tag ubuntu localhost:5000/ubuntu
    # docker push localhost:5000/ubuntu
