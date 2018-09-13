--- 
layout: post
title: python学习之rabbitmq
subtitle: rabbitmq学习
date: 2018-06-11
author: YangSijie
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - python
---

# RPC与rabbitmq相关

- 每一个routing_key对应一个message queue，可以认为是MQ的名字；
- exchange负责将数据路由至对应的MQ；
- basic_consume()像是声明，只有等到从参数中的MQ中获取到数据后，才会调用对应的callback函数执行；
- basic_publish()只要执行，就会向MQ发送数据；
- 三种典型的exchange：direct（一对一，只发给某一个MQ）、fanout（广播，发给所有MQ）、topic（组播，只发给特定的若干MQ）；
- topic类型的exchange实现是依靠nova.scheduler、nova.host……，依靠字符匹配：'*'表示一个词语，'#'表示所有词语；如：'*.scheduler'则匹配到nova.scheduler，'#'则匹配到nova.scheduler、nova.host……这些所有的词语；
- publisher将数据发布到exchange中去，consumer将MQ绑定到exchange中，然后就可以获取数据了；
- 经常会看到consumer和publisher两端都用queue_declare()方法声明MQ的，其实只需要在consumer端声明即可（用exchange_declare()声明exchange也应当是如此）；

[点击这里下载](ftp://207.148.67.29/rpc_consumer.py)rpc_consumer.py

[点击这里下载](ftp://207.148.67.29/rpc_publisher.py)rpc_publisher.py