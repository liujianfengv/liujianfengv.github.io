---
title: 'MQTT基础文档翻译(四):发布、订阅和取消订阅'
date: 2022-02-17 22:49:11
tags: MQTT
---
## 原文链接
[MQTT Publish, Subscribe & Unsubscribe - MQTT Essentials: Part 4](https://www.hivemq.com/blog/mqtt-essentials-part-4-mqtt-publish-subscribe-unsubscribe/)
<!--more-->
## 正文
欢迎来到MQTT系列文章的第四部分。在这篇文章中，我们将主要关注MQTT中的发布、订阅和取消订阅。在之前的文章中讲述了发布/订阅模型的基础概念。在这篇文章中我们专注于MQTT协议中发布/订阅相关特性。如果你没有阅读过之前关于发布订阅模型的基础知识介绍，我们强烈建议你先阅读相关的文章。 <br/>
上周，我们主要关注在MQTT客户端和broker之间简历连接。这一周，我们以这些知识为基础来讨论发送和接收消息。在文章的末尾有一个关于这个话题的视频来补充文章的内容。

### 发布(Publish)

一个MQTT客户端只要建立起和broker的连接之后，就可以发布消息。 MQTT利用broker基于话题(Topic)的消息过滤方式。每一条消息必须含有一个Topic，这样broker就可以通过Topic信息来将消息转发到对这个Topic感兴趣的客户端。通常，每个消息有一块由字节格式数据组成的负载(payload)。MQTT是数据不可知的，客户端的使用方法决定了负载中数据的结构。发送消息的客户端(publisher)自由决定发送二进制数据或文本数据甚至是XML或者是JSON格式的数据。<br/>
在MQTT中一个PUBLISH消息有许多我们想要细致讨论的属性。

![](publish_packet.png)

- Packet Identifier

数据包ID是客户端和broker之间消息的唯一识别码，数据包ID只有在Qos级别大于0时生效。客户端程序库或者broker负责设置这个MQTT内部的识别ID。

- Topic Name

话题名称是一个使用斜杠作为分隔符的层次化简单字符串。 例如 "myhome/livingroom/temperatur" 或者 "Germary/Munich/Octoberfest/people". 更多关于Topics的细节请看第五章。

- Qos

这个数字表明了服务质量(Quality of Service)级别。总共有0，1，2三个级别。服务级别决定了对消息到达目的地(客户端或broker)提供了什么样的保证。更多关于Qos的细节请看第六章。

- Retain Flag

这个标志决定这条消息是否被broker保存来作为某个Topic的(last known good value)。当新的客户端订阅这个Topic，他们会收到这个Topic保存着的最后一条消息。 更多关于retained messages的细节请看第八章。

- Payload

这是消息的实际内容。内容可能包含图像、任何编码方式的文字、加密数据或任何二进制数据。

- DUP flag

这个标志表示这条消息是一个因为目标接受者没有确认原始消息而重传的复制数据。这个标志只有在 Qos 大于0时生效。通常，复制和重传的机制被MQTT客户端库或者broker实现。更多相关消息请看第六章。 <br/>
当一个客户端向MQTT broker发布一条消息。broker读取并确认消息(取决于Qos级别)，然后处理消息。处理的过程包括确认哪些客户端订阅了当前主题，并将消息发送给他们。
![](publish_flow.png)

客户端发布消息时只关心将消息发送给broker的过程。当broker接收到被发布的消息，由broker负责将消息传递给所有的订阅者。发布消息的客户端不会知道是否有其他客户端对这条消息感兴趣，也不会知道有多少客户端从broker收到了这条消息。

### 订阅(Subscribe)
在没有任何人订阅(即没有任何客户端订阅相关话题)的情况下发布消息是无意义的。客户端通过发送一条订阅消息给broker来接收感兴趣的消息。订阅消息的格式非常简单，包含了一个唯一数据包ID和一个订阅话题列表。
![](subscribe_packet.png)

- Packet Identifier

数据包ID是客户端和broker之间消息的唯一识别码。客户端程序库或者broker负责设置这个MQTT内部的识别ID。

- List of Subscriptions

一条订阅消息可以包含多条订阅。每条订阅由Topic和Qos级别组成。订阅消息中的Topic可以包含通配符，这样可以匹配一类Topic而不是某个特定的Topic。如果某个客户端重复订阅了一个Topic，broker会按照其中Qos级别最高的一条来处理对应的消息。

### Suback

为了确认每一条订阅消息，broker会回复一个SUBACK消息给客户端。这条消息包含着对应的订阅消息的数据包ID，和一系列的返回码。
![](suback_packet.png)

- Packet Identifier

数据包ID是客户端和broker之间消息的唯一识别码。和订阅消息中的ID一致。

- Return Code

broker为在订阅消息中收到的每一个 Topic/Qos 对返回一个结果码。 例如，如果订阅消息包含5个订阅，则SUBACK消息包含5个结果码。结果码确认每一个Topic并且返回被broker使用的Qos级别。如果broker拒绝了某个订阅，SUBACK消息会包含一个代表失败的结果码给对应的订阅。例如客户端没有足够的权限去订阅某个Topic。

![](return_code.png)

![](subscribe_flow.png)

在客户端成功的发送一条SUBSCRIBE消息，并且收到SUBACK消息之后。客户端会收到被订阅的话题下的每一条消息。

### 取消订阅(Unsubscribe)

和订阅(Subscribe)消息对应的是取消订阅(Unsubscribe)消息。这条消息为客户端删除一个订阅。取消订阅消息和订阅消息类似，包含一个数据包ID和一系列Topic。

![](unsubscribe_packet.png)

- Packet Identifier

数据包ID是客户端和broker之间消息的唯一识别码。客户端程序库或者broker负责设置这个MQTT内部的识别ID。

- List of Topic

话题列表可以包含多个客户端想要取消订阅的Topic。只需要发送Topic，不需要Qos。broker不管当前Topic属于什么Qos级别，都会为客户端取消对某个Topic的订阅。

### Unsunack

为了确认取消订阅消息，broker会发送一个UNSUBACK的确认消息给客户端。这条消息只包含着取消订阅消息的数据包ID(为了明确当前消息对应的取消订阅消息)。

![](unsuback_packet.png)

- Packet Identifier 

数据包ID是客户端和broker之间消息的唯一识别码。之前提到过，这个ID和UNSUBSCRIBE消息中的ID相等。

![](unsubscribe_flow.png)

当客户端从broker收到UNSUBACK消息后，客户端可以认为在UNSUBSCRIBE消息中包含的订阅已经被删除了。

### 结语

第四章的内容已经结束了，希望你有所收获。在下一章中我们将深入了解MQTT Topic的使用方法。 包括Topic的基础知识、如何使用通配符，并且会提供大量的例子。


