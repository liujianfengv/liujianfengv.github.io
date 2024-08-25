---
title: 'MQTT基础文档翻译(五):话题和相关的实践'
date: 2022-03-02 00:06:43
tags: MQTT
---
## 原文链接
[MQTT Topics & Best Practices - MQTT Essentials: Part 5](https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-Topics-best-practices/)<!--more-->
## 正文
欢迎来到关于MQTT核心特性和概念系列文章的第五篇。在这篇文章中，我们将专注于MQTT的Topic和相关的最佳实践。正如我们之前提到的，MQTT的broker通过消息中的Topic信息来决定哪一个客户端将收到消息。我们还将看到一些特殊的系统Topic，这些Topic代表了一些关于broker状态的Topic。

### Topic
在MQTT中，Topic代表一个被broker用来为每个已连接的客户端筛选消息的UTF-8字符串。Topic由一级或多级组成。每一级之间通过正斜杠来划分。
![](topic_basics.png)

和消息队列相比，MQTTTopic是非常轻量的。客户端在发布或订阅一条消息之前，不需要提前创建一个Topic。broker会在没有任何初始化的情况下接受所有正确的Topic。<br/>
这里有一些Topic的例子:
```
myhome/groundfloor/livingroom/temperature
USA/California/San Francisco/Silicon Valley
5ff4a2ce-e485-40f4-826c-b1a5d81be9b6/status
Germany/Bavaria/car/2382340923453/latitude
```
注意，每个Topic必须含有至少一个字符，并且Topic字符串允许空字符串。 Topic 是大小写敏感的。例如 myhome/temperature 和 MyHome/Temperature 是两个不同的Topic。另外，单一的正斜杠是一个合法的Topic。

### 通配符
当一个客户端订阅一个Topic，它可以精确的订阅一个Topic，也可以通过通配符同时订阅多个Topic。通配符只可以被用于订阅Topic，不可以用于发布消息。这里有两种不同的通配符:单层和多层。

#### 单层通配符: +
正如名字所表示的含义，一个单层通配符可以替换一级Topic。在Topic的字符串中，加号代表着一个单层通配符。
![](topic_wildcard_plus.png)

当Topic的某一级为通配符时，在这一级有任何字符串都可以被匹配。例如 一条形如 myhome/groundfloor/+/temperature 的订阅消息对一下的Topic会产生不同的结果:
![](topic_wildcard_plus_example.png)

#### 多层通配符: #
多层通配符可以覆盖Topic中多层字符，'#'符号在Topic字符串中代表一个多层通配符。为了让broker可以确定哪些Topic可以被匹配，多层通配符必须是Topic字符串的字符，并且倒数第二个字符为正斜杠。
![](topic_wildcard_hash.png)

当一个客户端使用多层通配符订阅Topic，它将会收到以通配符之前字符串开头的所有消息，不管这条消息有多长。如果你只把 '#' 符号作为一条Topic。你会收到所有发往MQTT broker的消息。如果你想要高吞吐量，就不应该只使用一个'#'作为Topic。

### 以 '$' 符号开头的话题
一般来说，你可以给MQTT Topic任意的名字。但是，这有一个例外， 以'\$'符号开头的Topic被borker保留用来做内部的计数。客户端不可以给这些话题推送消息。同时，这些Topic没有被官方标准化。通常 \$SYS/ 被用于代表以下消息，但不同的实现可以能有不同的意义。 [MQTT GitHub Wiki](https://github.com/mqtt/mqtt.org/wiki/SYS-Topics)中有\$SYS 相关Topic的建议。这里有一些例子:

```
$SYS/broker/clients/connected
$SYS/broker/clients/disconnected
$SYS/broker/clients/total
$SYS/broker/messages/sent
$SYS/broker/uptime
```
### 总结
上述是一些关于MQTT消息的基础知识。如你所见，MQTT Topic是动态并且具有灵活性的。当你在真实的程序中使用通配符时，有一些挑战需要你留心。下面我们会讲一些从很多MQTT项目的实践中总结出的一些MQTT Topic相关的最佳实践，你有任何想法都可以和我们讨论。

## 最佳实践
- 不要使用正斜杠作为Topic的首字符 
正斜杠作为Topic首字符在MQTT中是被允许的，例如, /myhome/groundfloor/livingroom. 然而，前导正斜杠带来了一个没有任何字符的多余层级，这没有带来任何好处并且容易产生误解。
- 不要在Topic中使用空格
空格是所有程序员的敌人。当事情没有按预期发生时，Topic中的空格会加大查找问题的代价。和前导正斜杠一样，一些行为被允许不代表它是对的。UTF-8有许多中不同类型的空格，实践中应该避免这样不统一的字符。
- 保持Topic的精简
每个Topic将被所有关于它的消息携带，所以应该尽可能的让Topics变得精简。在一些小型设备上，每一个字节都有很大的影响。
- 只使用ASCII字符，避免使用不可打印的字符
因为非 ASCII的 UTF-8字符经常不可以正常显示。所以如果不是迫不得已，我们建议避免在Topic中使用任何的非ASCII字符。
- 将唯一ID或客户端ID加入到Topic中
将唯一ID或客户端ID加入到Topic中是非常有帮助的，这可以帮助你明确是哪个客户端发出的消息。Topic内置ID可以用来控制权限。只有当一个客户端和Topic拥有相同ID时才被允许向Topic发布消息。例如，一个客户端的ID为 'clent1',它可以向client1/status发布消息。但不可以向 client2/status 发布消息。
- 不要订阅 '#'
在有些场景中，我们必须订阅通过broker的所有消息。例如，将所有消息持久化到数据库。不要通过使用一个客户端订阅 '#' Topic的方式来获取所有消息。一般来说，一个客户端很难承担所有消息的负载(特别是消息吞吐量大时)。
我们推荐给MQTT broker实现一个插件。例如，使用plugin system of HiveMQ 你可以给broker添加一个钩子，异步的将所有消息存入数据库。
- 不要忘记可扩展性
Topics 是一个灵活的概念，并且在任何情况下都不需要提前配置它。然而，发布者和订阅者都必须知道用来通信的Topic。思考如何设计Topic可以有助于为项目添加新特性是非常重要的。例如，为智能家居添加新的传感器时，尽量将他们添加到Topic树的一部分，而不是修改整个Topic的结构。
- 使用特定的话题
当你给一个Topic命名时，不要使用给队列命名时相同的习惯。尽量使不同的Topic之间便于区分。例如，如果你有三个传感器在卧室中，创建这样的三个Topic:
```
myhome/livingroom/temperature
myhome/livingroom/brightness
myhome/livingroom/humidity. 
```
不要通过 myhome/livingroom发送所有消息。使用特定的消息有助于使用其他MQTT的特性，例如 retained messages;

## 结语
MQTT系列文章的第五篇文章结束了，下一章我们将主要讲解服务质量(Quality of Service)相关的内容。