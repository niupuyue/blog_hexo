---
title: 重拾android路(三十五) MQTT
date: 2019-12-24 21:35:51
tags:
  - android
  - mqtt
---

# 关于MQTT
<!--more-->
## 介绍
MQTT，消息队列遥测传输，是IBM开发的一个即时通信协议。它是一种发布/订阅，极其简单和轻量级的消息传递协议，专门为受限设备和低带宽，高延迟或不可靠的网络而设计。它的设计思想是轻巧，开放，简单，规范，易于实现。这些特点使得他对很多场景来说都是很好的选择。特别是一些受限环境如机器与机器的通信(M2M)以及物联网环境，相对于XMPP，MQTT更加轻量级，并且占用的宽带低。

## 特点
1. 使用发布/订阅消息模式。提供一对多的消息发布，解除应用程序耦合
2. 对敷在内容屏蔽的消息传输模式
3. 使用TCP/IP提供网络连接
4. 有三种消息发布服务质量
    - qos为0:"至多1次"，消息发布完全依赖底层TCP/IP网络。会发生消息丢失或重复，这一级别可用于如下情况：环境传感器数据，丢失一次几路无所谓，因为不久之后还有第二次发送
    - qos为1:"至少1次"，确保消息到达，但消息重复可能会发生，这一级别可用于如下情况：你需要获取每条消息，并且消息重复发送对我们来说无影响
    - qos为2::"只有1次"，确保消息到达一次，这一级别可用于如下情况，在计费系统中，消息重复或丢失导致不正确的结果
5. 小型传输，开销很小(固定长度的头部是2字节)。协议交换最小化，以降低网络流量。使用Last Will和Testament特性通知有关各方客户端异常中断的机制

## MQTT体系结构
如下图所示
![MQTT体系结构](/assets/android/mqtt01.png)

我们可以这样理解，我们去一个景区玩，进入景区配置一台闸机设备作为发布者(publisher),当闸机设备监控到有游客进入的时候，会发布一个带有主体(topic)的消息(例如主题为：welcome)给服务器(MQTT-Broker),当服务器收到发不过来的消息后，进行基于主题的过滤，将消息转发给鼎娱乐该主题的订阅者。而景区的大屏幕作为定于这(Subscriber),订阅的主题也是welcome,这样就能接收到服务器转发过来的消息，收到消息后在大屏幕上显示我们需要的内容

在该结构图中，闸机设备和大屏幕都是客户端，都可以进行发布和订阅。

# 基础知识

## 发布/订阅模式
发布/订阅模式解耦了发布消息的客户端(发布者)和订阅消息的客户端(订阅者)之间的关系，这意味着发布者和订阅者之间不需要直接建立联系。就比如，打电话给朋友，我们需要等到朋友接通电话才能交流，这就是一种典型的同步请求/回答的场景；而给一个好友发送邮件就不一样了，发送的邮件之后，我们可以去做别的事，朋友有时间去看邮件就好了。这就是一个典型的异步发布/订阅的场景。

> 发布者与订阅者不需要彼此了解，只需要认识同一个消息代理即可；发布者和订阅者不需要交互，发布者无需等待订阅者确认而导致锁定；发布者和订阅者不需要同时在线，可以自由选择时间消费消息

## 主题
MQTT是通过主题对消息进行分类的，本质上就是一个UTF-8的字符串，不过可以通过反斜杠表示多个层级关系，主题不需要创建，直接使用即可
主题还可以通过通配符进行过滤，其中+可以过滤一个层级，*只能出现在主题最后表示过滤任意级别的层级

> building-b/floor-5:表示B楼5层的设置；+/floor-5:表示任何一个楼里的5层的设置；building-b/*:表示B楼中的所有设配

## 服务质量
为了满足不同的场景，MQTT支持三种不同级别的服务质量QoS，为不同的场景提供消息的可靠性

1. 级别0：尽力而为，消息发送者会想尽办法将消息发送出去，但如遇意外，并不会重新发送
2. 级别1：至少一次，消息接受者如果没有知会或者知会本身丢失，消息发送者再次发送该消息，以确保消息能够被接收到，这种情况，可能会造成消息的重复
3. 级别2：恰好一次，保证这种语义肯定会减少并发或增加延迟，不过丢失或者重复消息是不可被接受的。

服务质量是一个老话题，级别2所提供的的不重复不丢失很多情况下是最理想的，不过往返多次的确认，一定会对并发和延迟带来影响；级别1提供至少一次语义在日志处理这种场景下是没问题的，所以像Kafka这类的系统利用这个特点减少确认从而提高高并发；级别0适合几个数据场景，但是并不是特别多，姑且不表；

## 消息类型
1. CONNECT：客户端连接到MQTT代理
2. CONNACK：连接确认
3. PUBLISH：新发布消息
4. PUBACK：新发布消息确认，是QoS 1给PUBLISH消息的回复
5. PUBREC：QoS 2消息流的第一部分，表示消息发布已记录
6. PUBREL：QoS 2消息流的第二部分，表示消息发布已释放
7. PUBCOMP：QoS 2消息流的第三部分，表示消息发布完成
8. SUBSCRIBE：客户端订阅某个主题
9. SUBACK：对于SUBSCRIBE消息的确认
10. UNSUBSCRIBE：客户端终止订阅的消息
11. UNSUBACK：对于UNSUBSCRIBE消息的确认
12. PINGREQ：心跳
13. PINGRESP：确认心跳
14. DISCONNECT：客户端终止连接前优雅地通知MQTT代理



# 参考博客
[android App必备高级功能，消息推送之MQTT](https://blog.csdn.net/qq_17250009/article/details/52774472)
[Android消息推送MQTT实战](https://www.jianshu.com/p/73436a5cf855)
[Getting started with MQTT](https://www.hivemq.com/blog/how-to-get-started-with-mqtt/)
[MQTT入门知识](https://blog.csdn.net/github_33304260/article/details/73555475)