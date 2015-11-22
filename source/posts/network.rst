==================================
复杂的网络——私有云桌面中的网络规划
==================================

本文尝试从笔者遇到的诸多客户网络环境找出几个典型，使用oVirt/OpenStack提供基本虚拟化功能，来介绍如何规划网络环境以达到用户需求与性能的平衡效果。

1. 桌面云中常用网络 
====================

笔者所接触的虚拟化平台中，普遍使用libvirt组网，其实现技术可以是tagged或者untagged vlan，NAT，桥接，openvswitch的gre等。接下来我将就使用较多的NAT网络、桥接网络和openvswitch作些详细介绍。

------------
1.1. NAT网络
------------

对于NAT（network address translation）技术，我们或许并不陌生。我们家庭路由器接入ISP提供商时，路由器会获得一个外网IP（公网IP），局域网电脑接入路由器后获得一个内网IP，我们即是从NAT后端来访问外网的。这也是NAT最广的用途之一，随着现在IPv6的流行，有人甚至认为NAT技术不如往日重要了。我们在此不讨论这个话题，只介绍一下NAT技术的在虚拟化中的常见应用场景。

.. figure:: ../images/nat1.png
    :align: center

    图1-1 NAT通用实现

内部地址（iAddr:port1）映射到外部地址（eAddr:port2），所有从iAddr:port1的包都经由eAddr:port2向外发送。所有外部主机都能通过eAddr:port2向内部主机的iAddr:port1发送数据，有些实现中，要求必须内部主机先向外部主机发送数据才能允许外部发送至内部。

.. figure:: ../images/libvirt-nat.png
    :align: center

    图1-2 libvirt NAT示例

.. code::

    <network>
      <name>default</name>
      <bridge name="virbr0" />
      <forward mode="nat"/>
      <domain name="example.com"/>
      <dns>
        <txt name="example" value="example value" />
        <forwarder addr="8.8.8.8"/>
        <forwarder addr="8.8.4.4"/>
        <srv service='name' protocol='tcp' domain='test-domain-name' target='.' port='1024' priority='10' weight='10'/>
        <host ip='192.168.122.2'>
        <hostname>myhost</hostname>
        <hostname>myhostalias</hostname>
        </host>
     </dns>
      <ip address="192.168.122.1" netmask="255.255.255.0">
        <dhcp>
          <range start="192.168.122.100" end="192.168.122.254" />
        </dhcp>
      </ip>
      <ip family="ipv6" address="2001:db8:ca2:2::1" prefix="64" />
    </network>

所有虚拟机获得的IP范围为192.168.122.100-192.168.122.254，网关为192.168.122.1，DNS为192.168.122.2，请求可转发至8.8.8.8或8.8.4.4。

-------------
1.2. 桥接网络
-------------

----------------
1.3. Openvswitch
----------------

2. oVirt/OpenStack的桌面网络应用
================================

--------------------------
2.1. 虚拟机使用主机NAT网络
--------------------------

--------------------------
2.2. 虚拟机使用主机OVS网络
--------------------------

-----------------------------
2.3. 管理、业务、存储隔离网络
-----------------------------

------------------------
2.4. 使用已有Neutron组网  
------------------------

3. 总结
========
