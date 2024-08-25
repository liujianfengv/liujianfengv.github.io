---
title: 'MQTT基础文档翻译(三):MQTT客户端和Broker和MQTT服务器和连接建立的解释'
date: 2021-11-14 23:41:40
tags: MQTT
---
## 原文链接
[MQTT Client and Broker and MQTT Server and Connection Establishment Explained - MQTT Essentials: Part 3](https://www.hivemq.com/blog/mqtt-essentials-part-3-client-broker-connection-establishment/)
<!--more-->
## 正文
欢迎来到关于MQTT核心特性和概念系列文章的第三篇。在这篇文章中，我们将讨论MQTT客户端和broker的角色，连接MQTT服务器时的参数和选项，并解释MQTT服务器连接的建立。<br/>
在文章的末尾，我们有一个视频来补充这篇文章，我们建议你阅读完文章之后观看视频获取补充内容。<br/>
上周，我们解释了发布/订阅模式是如何工作的以及在MQTT中是如何应用这一模式的，下面是核心内容的回顾。

- 发布/订阅模式解耦了消息发送方客户端即发布者和消息接收方客户端即订阅者。
- MQTT使用消息中的话题(topic)信息来决定这条消息被发送给哪个订阅者。话题是一个有层次结构的字符串，可以用来过滤和路由消息。

上篇文章我们从高级别的抽象介绍了发布/订阅模式还有发布/订阅模式与传统消息队列的区别。这篇文章我们将讲述一些更加贴近实际的内容: MQTT客户端和broker的定义，MQTT连接的基础，MQTT连接时的参数和通过broker的确认(Ack)来确立连接。

### 简介
因为MQTT解耦了发布者和订阅者，客户端连接通常被broker所处理，在我们了解连接的细节之前，我们需要先清楚客户端和broker的意义。

- 客户端(Client)

当我们谈论客户端，通常指的是MQTT客户端，包括发布者和订阅者。发布者和订阅者的区别在于客户端正在发布消息还是正在订阅一个主题，并接收消息。(发布和订阅可以同时实现在一个客户端中)。一个MQTT客户端可以是运行着MQTT库并且通过网络和一个MQTT broker连接的任意设备(从微型控制器到大型服务器)。例如，MQTT客户端可以是一个非常小的、资源受限的设备拥有一个最小的实现库，通过无线网连接broker。MQTT客户端也可以是一台有MQTT客户端图形界面用来测试的计算机。总之，通过在TCP/IP之上的MQTT进行数据交互的设备都可以称为MQTT客户端。客户端实现MQTT协议是非常容易的。由于易于实现的特性MQTT特别适合体积小的设备。MQTT客户端开发库有许多语言的版本，例如Android,Arduino,C,C++,C#,Go,iOS,Java,JavaScript,和.NET。你可以在MQTT维基上看到完整的列表。

- Broker

MQTT客户端对应的是MQTT broker。 broker是发布/订阅模式的核心。根据实现，broker可以同时处理百万以上的MQTT客户端。<br/>
broker的责任是接收全部消息，过滤消息，确定由谁订阅了某条消息，并将消息发送给订阅该话题的客户端。broker持有所有持续会话客户端的会话信息，包括订阅和错过的消息。broker的另一个责任是授权和认证客户端。
通常，broker是可扩展的，这有助于定制授权、认证和集成到后端系统。因为broker通常是直接暴露到网络的模块、处理许多客户端、并且需要传递消息到下游的分析和处理系统所以集成是非常重要的。像上篇文章讨论的那样，订阅消息并不是唯一的选择。简而言之，broker是一个每个消息必须通过的消息枢纽。所以，它必须高度可扩展，易于集成到后端系统，易于监控，具有稳定性。HiveMQ通过使用先进的事件驱动网络、开放的扩展系统和提供标准监控来实现这些需求。
### MQTT Connection
MQTT是基于TCP/IP的，客户端和broker都需要拥有TCP/IP协议栈。
![](mqtt-tcp-ip-stack.png)
MQTT连接总是在客户端和broker之间，客户端之间不会直接连接。为了初始化一个连接，客户端发送CONNECT消息给broker。broker回复一个CONNACK消息和状态码。一旦连接被建立。broker会保持连接打开，直到客户端发送断开命令或者连接中断。
![](connect-flow.gif)

- MQTT通过NAT(network address translation)连接

在许多实际情况中，MQTT客户端处于一个使用NAT的路由器之后。路由器会完成一个公有地址到私有地址(像192.168.x.x,10.0.x.x)的转换。正如我们之前提到的，MQTT客户端通过发送CONNECT消息来建立连接。broker有一个公网IP地址，并且在收到CONNECT消息之后，就有了与客户端双向通信的能力，所以即便客户端在一个NAT网络之后不会有什么问题。

- 客户端通过CONNECT消息初始化连接

现在我们来看一看MQTT的CONNECT消息。为了初始化连接，客户端发送一个命令消息给broker。如果这条消息格式是错误的，或者打开网络socket和发送连接消息之间花费了太多时间，broker会关闭这个连接。这样可以防止恶意的客户端降低broker的速度。一个正确的MQTT客户端会发送一个以下内容的消息:
![](connect.png)
CONNECT消息中包含的一些内容可能是面向MQTT库实现者而不是使用者的，更多细节，查看MQTT3.1.1规范。我们将关注以下几个选项。

- ClientId

客户端标识符用来区分连接到MQTT broker的每一个MQTT客户端。broker通过使用ClientId来验证客户端和客户端当前的状态。所以，这个id需要是唯一的。在MQTT 3.1.1中，你可以发送一个空的ClientId，如果你不需要broker记录状态。空的ClientId会使一个连接没有任何状态信息，在这种情况下，清空会话的标志必须被设置成true，否则broker会拒绝这个连接。

- CleanSession

CleanSession标志告诉broker客户端是否想要建立一个持久会话。当标志为假时建立持久会话，broker会存储客户端所有的订阅和所有客户端错过的服务质量为1或2的消息。当标志为真时建立非持久会话，broker不会为客户端存储任何信息，并且会删除之前持久会话的所有信息。

- Username/Password

MQTT可以通过发送用户名和密码来进行认证和授权。然而，如果这些信息没有被加密或者hash处理，密码会被明文传输，我们强烈建议用户名和密码通过安全的方式传输。HiveMQ可以通过SSL认证来鉴定客户端，所以不需要用户名和密码。

- Will Message

last will 消息是MQTT Last Will 和 Testament 特性的一部分。这条消息会在客户端非正常关闭时通知其他客户端。当客户端连接上之后，它可以向broker的CONNECT消息中提供last will消息的话题和消息内容。如果客户端非正常关闭，broker会代替客户端发送LWT消息，你可以在系列文章的第九部分了解更多关于LWT的内容。

- KeepAlive

keep alive 是一个以秒为单位的时间间隔，由客户端在连接建立时指定。这个时间间隔定义了broker和客户端可以忍受没有消息发送的最长时间间隔。 通常，客户端发送常规的PING消息，broker回复PING回复消息。这种方法允许两端都可以判断对方是否仍然在线。系列文章的第十章详细讲述了MQTT的keep alive功能。<br/>
基本上，这是为了连接MQTT3.1.1客户端连接MQTT broker所需要的全部信息。不同的实现通常会有一些可以配置的附加选项。比如，在某些实现中可以设置存储消息的方法。

- Broker response with a CONNACK message

当broker收到一条CONNECT消息，他有义务回复一条CONNACK消息。<br />
CONNACK消息包含两条内容:

- Session Present flag

会话状态标志告诉客户端broker是否已经有一个持久会话在与此客户端之前的沟通中建立。当客户端连接时设置Clean Session为真时，session persent flag总是被设置成false，因为这样不会有可用的会话。当客户端连接时设置Clean Session为假时，这里有两种可能，如果此clientId有可以用的会话信息，并且broker存储着会话信息，则session present flag被设为真。否则，如果broker没有任何这个clientId相关的会话信息，session present flag被设置为假。这个标志在MQTT3.1.1中被加入用来帮助客户端判断他们是需要订阅话题或者订阅信息已经被存储在持久会话相关信息中。

- Connect return code
CONNACK消息中第二个标志是连接确认标志，这个标志会包含一个结果码，会告诉客户端这次连接请求是否成功。
![](connack1.png)

返回码定义
![](retcode.png)
更多细节的定义请查阅MQTT文档。

### 结语
你可能想直到MQTT在没有消息发送的时候是如何保持连接的，或者如何知道连接已经失效。不要担心，我们会在后续的文章中讲述相关话题。<br\>
MQTT系列文章的第三篇文章结束了，希望文章中有可以帮到你的内容，下周我们将会发布关于发布、订阅、取消订阅相关内容的文章。
