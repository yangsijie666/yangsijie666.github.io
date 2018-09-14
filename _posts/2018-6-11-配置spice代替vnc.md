--- 
layout: post
title: openstack之配置spice
subtitle: 配置spice替换novnc
date: 2018-06-11
author: YangSijie
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - openstack
---

# 配置spice代替vnc

### `Ubuntu16.04版本测试可用`

## 一.控制节点：

1.安装spice相关组件：

```bash
apt install nova-spiceproxy spice-html5 spice-vdagent
```

2.修改`/etc/nova/nova.conf`配置文件：
    
- 修改`[vnc]`中的参数:

```vim
[vnc]
enabled = false
```

- 在`[spice]`中添加配置：

```vim
enabled = True
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

> 其中`$my_ip`为`[DEFAULT]`中的`my_ip`的值。

3.停止`nova-novncproxy`服务：

```bash
service nova-novncproxy stop
```

4.重新启动nova的相关服务：

```bash
service nova-api restart
service nova-consoleauth restart
service nova-scheduler restart
service nova-conductor restart
service nova-spiceproxy restart
```

## 二.计算节点

1.修改`/etc/nova/nova.conf`配置文件：

- 修改`[vnc]`中的参数:

```vim
[vnc]
enabled = false
```

- 在`[spice]`中添加配置：

```vim
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
html5proxy_base_url = http://CONTROLLER_ADDRESS:6082/spice_auto.html
```

> 其中`$my_ip`为`[DEFAULT]`中的`my_ip`的值;
> `CONTROLLER_ADDRESS`为controller节点的ip地址。

2.重新启动nova的相关服务：

```bash
service nova-compute restart
```

## 三.硬重启所有已创建的实例。
