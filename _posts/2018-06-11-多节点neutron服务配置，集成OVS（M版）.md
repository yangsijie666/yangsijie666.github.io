--- 
layout: post
title: openstack学习之多网络节点配置
subtitle: 多网络节点(ovs)
date: 2018-06-11
author: YangSijie
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - openstack
---

# 多节点neutron服务配置，集成OVS（M版）

## **一.控制节点**

**数据库相关操作**：

```bash
mysql -u root -p
```

```mysql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
```

> 替换`NEUTRON_DBPASS`为需要创建的neutron数据库的密码

**创建neutron用户、分配admin权限、创建neutron服务实体**：

```bash
source admin-openrc

openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | e0353a670a9e496da891347c589539e9 |
| enabled   | True                             |
| id        | b20a6692f77b4258926881bf831eb683 |
| name      | neutron                          |
+-----------+----------------------------------+

openstack role add --project service --user neutron admin

openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | f71529314dab4a4d8eca427e701d209e |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

> 输入的密码即为`NEUTRON_PASS`，即需要创建的neutron用户的密码

**创建neutron服务的API endpoints**：

```bash
openstack endpoint create --region RegionOne network public http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 85d80a6d02fc4b7683f611d7fc1493a3 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f71529314dab4a4d8eca427e701d209e |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne network internal http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 09753b537ac74422a68d2d791cf3714f |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f71529314dab4a4d8eca427e701d209e |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne network admin http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1ee14289c9374dffb5db92a5c112fc4e |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f71529314dab4a4d8eca427e701d209e |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```


**安装相关组件**：

```bash
yum install openstack-neutron openstack-neutron-ml2 python-neutronclient which
```

**配置服务组件：**

编辑`/etc/neutron/neutron.conf`文件：

```vim
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
rpc_backend = rabbit
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True

[database]
...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

[oslo_messaging_rabbit]
...
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[nova]
...
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```

> 替换`NEUTRON_DBPASS`为创建的neutron数据库的密码
> 替换`RABBIT_PASS`RABBITMQ的密码
> 替换`NEUTRON_PASS`为创建的用户neutron的密码
> 替换`NOVA_PASS`为创建的用户nova的密码
> 注意：将`[keystone_authtoken]`中的其余选项都注释掉！！！

**配置ML2组件：**

编辑`/etc/neutron/plugins/ml2/ml2_conf.ini`文件：

```vim
[ml2]
...
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security

[ml2_type_flat]
...
flat_networks = provider

[ml2_type_gre]
...
tunnel_id_ranges = 1:1000

[securitygroup]
...
enable_ipset = True
```

**配置计算服务，使其可以使用网络服务：**

编辑 `/etc/nova/nova.conf`文件：

```vim
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS

service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```

> 替换**NEUTRON_PASS**为创建的用户neutron的密码
> 替换**METADATA_SECRET**为设置的metadata代理的密码

**创建链接**：

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```
**同步数据库**：

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
**重启nova服务**：

```bash
systemctl restart openstack-nova-api.service
```
**启动neutron-server服务**：

```bash
systemctl enable neutron-server.service
systemctl start neutron-server.service
```




    
## 二.网络节点

**安装相关组件**：

```bash
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch
```
**配置服务组件**：

编辑`/etc/neutron/neutron.conf`文件：

删除
**`[DEFAULT]`**
中对数据库的访问！！

```vim
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
rpc_backend = rabbit
auth_strategy = keystone

[oslo_messaging_rabbit]
...
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

> 替换`RABBIT_PASS`为RABBITMQ的密码
> 替换`NEUTRON_PASS`为neutron用户的密码
> 注意：将`[keystone_authtoken]`中的其余选项都注释掉！！！

**配置ML2组件：**

编辑`/etc/neutron/plugins/ml2/ml2_conf.ini`文件：

```vim
[ml2]
...
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security

[ml2_type_flat]
...
flat_networks = provider

[ml2_type_gre]
...
tunnel_id_ranges = 1:1000

[securitygroup]
...
enable_ipset = true
```

**配置openvswitch agent：**

编辑`/etc/neutron/plugins/ml2/openvswitch_agent.ini`文件：

```vim
[ovs]
local_ip = TUNNEL_INTERFACE_IP_ADDRESS
bridge_mappings = provider:br-ex

[agent]
tunnel_types = gre
l2_population = true
prevent_arp_spoofing = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

> `TUNNEL_INTERFACE_IP_ADDRESS`为用于隧道网的网卡地址

**配置layer-3 agent：**

编辑`/etc/neutron/l3_agent.ini`文件：

```vim
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
external_network_bridge = br-ex
```
> 官网上external_network_bridge为空???

**配置DHCP agent：**

编辑`/etc/neutron/dhcp_agent.ini`文件：

```vim
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

**配置metadata agent：**

编辑`/etc/neutron/metadata_agent.ini`文件：

```vim
[DEFAULT]
...
nova_metadata_ip = controller
metadata_proxy_shared_secret = METADATA_SECRET
```

> 替换`METADATA_SECRET`为metada的密码

**创建链接：**

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

**创建网桥br-ex，并将网卡加入其中：**

```bash
systemctl enable openvswitch.service
systemctl start openvswitch.service

ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex INTERFACE_NAME
```

> 替换`INTERFACE_NAME`为用于外部网络的网卡名

**启用各项服务：**

```bash
systemctl start neutron-openvswitch-agent.service neutron-l3-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
systemctl enable neutron-openvswitch-agent.service neutron-l3-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```






## 三.计算节点

**安装相关组件：**

```bash
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch
```

**配置各个组件：**

编辑`/etc/neutron/neutron.conf`文件：

> 将`[DEFAULT]`中的关于连接数据库部分都注释掉！！！

```vim
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```

> 替换`RABBIT_PASS`为RABBITMQ的密码
> 替换`NEUTRON_PASS`为用户neutron的密码
> 将`[keystone_authtoken]`中的其他选项全部注释掉！！！

**配置其ovs agent：**

编辑`/etc/neutron/plugins/ml2/openvswitch_agent.ini`文件：

```vim
[ovs]
...
local_ip = TUNNEL_INTERFACE_IP_ADDRESS

[agent]
...
tunnel_types = gre
l2_population = true

[securitygroup]
...
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = true
```

> 替换`TUNNEL_INTERFACE_IP_ADDRESS`为用于隧道网的网卡地址

**配置nova,使其能使用网络服务：**

编辑`/etc/nova/nova.conf`文件：

```vim
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```

> 替换`NEUTRON_PASS`为用户neutron的密码

**启用各项服务：**

```bash
systemctl restart openstack-nova-compute.service
systemctl enable neutron-openvswitch-agent.service
systemctl start neutron-openvswitch-agent.service
```

验证是否安装成功

```bash
neutron ext-list
+---------------------------+-----------------------------------------------+
| alias                     | name                                          |
+---------------------------+-----------------------------------------------+
| default-subnetpools       | Default Subnetpools                           |
| network-ip-availability   | Network IP Availability                       |
| network_availability_zone | Network Availability Zone                     |
| auto-allocated-topology   | Auto Allocated Topology Services              |
| ext-gw-mode               | Neutron L3 Configurable external gateway mode |
| binding                   | Port Binding                                  |
| agent                     | agent                                         |
| subnet_allocation         | Subnet Allocation                             |
| l3_agent_scheduler        | L3 Agent Scheduler                            |
| tag                       | Tag support                                   |
| external-net              | Neutron external network                      |
| net-mtu                   | Network MTU                                   |
| availability_zone         | Availability Zone                             |
| quotas                    | Quota management support                      |
| l3-ha                     | HA Router extension                           |
| provider                  | Provider Network                              |
| multi-provider            | Multi Provider Network                        |
| address-scope             | Address scope                                 |
| extraroute                | Neutron Extra Route                           |
| timestamp_core            | Time Stamp Fields addition for core resources |
| router                    | Neutron L3 Router                             |
| extra_dhcp_opt            | Neutron Extra DHCP opts                       |
| dns-integration           | DNS Integration                               |
| security-group            | security-group                                |
| dhcp_agent_scheduler      | DHCP Agent Scheduler                          |
| router_availability_zone  | Router Availability Zone                      |
| rbac-policies             | RBAC Policies                                 |
| standard-attr-description | standard-attr-description                     |
| port-security             | Port Security                                 |
| allowed-address-pairs     | Allowed Address Pairs                         |
| dvr                       | Distributed Virtual Router                    |
+---------------------------+-----------------------------------------------+

neutron agent-list
+--------------------------------------+--------------------+----------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host     | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+----------+-------------------+-------+----------------+---------------------------+
| 250ffcfd-afb1-43ed-b23b-77297cdf842b | L3 agent           | network  | nova              | :-)   | True           | neutron-l3-agent          |
| 2cc4f859-cf2f-4238-9053-210583ed96d5 | DHCP agent         | network  | nova              | :-)   | True           | neutron-dhcp-agent        |
| 7382d15a-8a75-405b-b829-748d5a93dd94 | Metadata agent     | network  |                   | :-)   | True           | neutron-metadata-agent    |
| 8d504da9-5d70-4fd9-b8f6-5520fa7c7a5f | Open vSwitch agent | network  |                   | :-)   | True           | neutron-openvswitch-agent |
| a09e5522-a4fc-4d21-be9a-968826386f3c | Open vSwitch agent | compute1 |                   | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+----------+-------------------+-------+----------------+---------------------------+

```
