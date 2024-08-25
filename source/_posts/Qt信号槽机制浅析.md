---
title: Qt信号槽机制浅析
date: 2021-11-02 22:39:24
tags: Qt
---

Qt是一个跨平台的C++软件开发框架，而Qt的核心机制是信号槽，关于信号槽，Qt官方的表述是:
<!--more-->
> Signals and slots are used for communication between objects. The signals and slots mechanism is a central feature of Qt and probably the part that differs most from the features provided by other frameworks. Signals and slots are made possible by Qt's meta-object system.

信号和槽被用于对象间的沟通，信号槽机制是Qt的核心特点，也是Qt与其他应用框架的不同之处，信号槽的机制得益于Qt的元对象系统。
下面分两步来对Qt的信号槽机制做一个简单分析。

## 从Qt程序的编译过程讲起
一个普通C++程序的编译需要经过以下几个步骤
- 预处理(Prepressing): 主要处理源代码中以“#”开始的预编译命令，例如"#include"将被包含文件插入到该指令位置、 处理"#if"、"ifdef"等条件预编译指令、删除注释
- 编译(Compilation): 将预处理完的文件进行词法分析、语法分析、语义分析生成相应的汇编代码文件
- 汇编(Assembly): 将编译生成的汇编代码转化成机器可以执行的指令，生成目标文件
- 链接(Linking): 组织汇编生成的目标文件和编译环境下相关的运行时库，将各个模块之间的相互引用部分处理好，使得各个模块间能够正确的衔接
Qt程序的编译与C++程序的编译主要区别在于moc系统

> The Meta-Object Compiler, moc, is the program that handles Qt's C++ extensions. The moc tool reads a C++ header file. If it finds one or more class declarations that contain the Q_OBJECT macro, it produces a C++ source file containing the meta-object code for those classes. Among other things, meta-object code is required for the signals and slots mechanism, the run-time type information, and the dynamic property system

元对象编译器是处理Qt的C++拓展程序， moc工具读取C++头文件，如果在文件中找到一个或多个Q_OBJECT宏，它将会生成一个包含对应类元对象代码的C++源文件，除此之外，元对象代码是信号槽机制、运行时类型信息、动态属性系统所必须的。
一个最简单的不带界面和资源文件，只使用信号槽机制的Qt程序的编译过程就是，在普通C++程序编译的过程前对包含Q_OBJECT的类使用moc工具生成对应的元对象代码，即 元对象编译->预处理->编译->汇编->链接。
下面用一个例子来验证。

```
testSignalSlot
├── main.cpp
├── reciver.cpp
├── reciver.h
├── sender.cpp
└── sender.h
```

```
//reciver.h
#ifndef RECIVER_H
#define RECIVER_H

#include <QObject>

class Reciver : public QObject
{
    Q_OBJECT
public:
    explicit Reciver(QObject *parent = nullptr);
public slots:
    void recive();
};

#endif // RECIVER_H
```
```
//reciver.cpp
#include "reciver.h"

#include <QDebug>
Reciver::Reciver(QObject *parent) : QObject(parent)
{

}

void Reciver::recive()
{
    qDebug()<<"Hello Qt!";
}

```
```
//sender.h
#ifndef SENDER_H
#define SENDER_H

#include <QObject>

class Sender : public QObject
{
    Q_OBJECT
public:
    explicit Sender(QObject *parent = nullptr);

signals:
    void send();
};

#endif // SENDER_H
```
```
//sender.cpp
#include "sender.h"

Sender::Sender(QObject *parent) : QObject(parent)
{
}
```
```
main.cpp
#include <QCoreApplication>

#include "sender.h"
#include "reciver.h"
int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);
    Sender sender;
    Reciver reciver;
    QObject::connect(&sender, &Sender::send,
                     &reciver, &Reciver::recive);
    emit sender.send();
    return a.exec();
}
```

首先使用moc工具为sender 和 reciver 生成moc代码 moc路径

```
C:\Qt\5.15.2\mingw81_64\bin\moc.exe
```

执行moc

```
$ moc sender.h -o moc_sender.cpp
$ moc reciver.h -o moc_reciver.cpp
```

完成后目录

```
testSignalSlot
├── main.cpp
├── moc_reciver.cpp     (moc生成代码)
├── moc_sender.cpp      (moc生成代码)
├── reciver.cpp
├── reciver.h
├── sender.cpp
└── sender.h
```

使用g++生成目标文件

```
$ g++ -c  -IC:\\Qt\\5.15.2\\mingw81_64\\include -IC:\\Qt\\5.15.2\\mingw81_64\\include\\QtCore  -o reciver.o reciver.cpp
$ g++ -c  -IC:\\Qt\\5.15.2\\mingw81_64\\include -IC:\\Qt\\5.15.2\\mingw81_64\\include\\QtCore  -o sender.o sender.cpp
$ g++ -c  -IC:\\Qt\\5.15.2\\mingw81_64\\include -IC:\\Qt\\5.15.2\\mingw81_64\\include\\QtCore  -o moc_reciver.o moc_reciver.cpp
$ g++ -c  -IC:\\Qt\\5.15.2\\mingw81_64\\include -IC:\\Qt\\5.15.2\\mingw81_64\\include\\QtCore  -o moc_sender.o moc_sender.cpp
$ g++ -c  -IC:\\Qt\\5.15.2\\mingw81_64\\include -IC:\\Qt\\5.15.2\\mingw81_64\\include\\QtCore  -o main.o main.cpp
```

生成可执行文件

```
g++ -o testSignalSlot.exe main.o reciver.o sender.o moc_reciver.o moc_sender.o  C:\\Qt\\5.15.2\\mingw81_64\\lib\\libQt5Core.a

```
到此编译完成，完成后目录:
```
testSignalSlot
├── testSignalSlot.exe
├── main.cpp
├── main.o
├── moc_reciver.cpp
├── moc_reciver.o
├── moc_sender.cpp
├── moc_sender.o
├── reciver.cpp
├── reciver.h
├── reciver.o
├── sender.cpp
├── sender.h
└── sender.o
```

运行验证一下:

```
$ ./testSignalSlot.exe
Hello Qt!
```

## 用标准C++完成一个简易的信号槽
经过第一部分的了解后，我们知道元对象系统会使用moc工具对带有Q_OBJECT宏的头文件生成一份带有元对象信息的moc_xxx.cpp文件，那么我们应该可以手动编写moc_xxx.cpp代码,来完成moc所做的事情。
接下来我们用标准C++来实现一个简易的信号槽机制，用以简单说明信号槽的实现原理。将手动编写以下几个文件，样例代码上传在github上。
- [iso-cpp-reimp-signal-slot](https://github.com/liujianfengv/iso-cpp-reimp-signal-slot)

```
├── main.cpp
├── moc_object.cpp
├── object.cpp
├── object.h
```

信号槽的实现大致需要以下几点

- 类中应该保存有信号和槽的字符串信息
- 字符串和信号槽函数要关联，有对应关系
- 类实例中应该存有信号和监听该信号的reciver和槽函数的信息，通过connect借助字符串完成对象间信号和槽的关联
- 信号触发时通过上述信息完成槽函数的调用

所以，首先我们定义一个结构体，里面存放信号和槽的名称信息

```
//object.h
struct MetaObject
{
    const char * sig_names;     //存放信号函数字符串
    const char * slts_names;    //存放槽函数字符串
};
...
```

并且将其作为一个静态成员变量放入Object类中,这样所有Object的实例就都拥有了信号和槽的字符串信息

```
//object.h
class Object
{
    static MetaObject meta;
    ...
```

然后，在自己生成的moc_object.cpp文件中填充这个结构体信息， 当有多个信号或槽时，我们约定将所有信号的字符串连接起来并用'\n'分割，这样每个信号或槽就有了一个对应的索引信息。 示例中使用信号 click 和 槽函数 onClick 

```
//moc_object.cpp
static const char sig_names[] = "click\n";
static const char slts_names[] = "onClick\n";
MetaObject Object::meta = {sig_names, slts_names};
```

接下来为每个对象保存连接信息，首先定义一个结构体。并定义一个mutimap作为Object的私有成员变量

```
//object.h
struct Connection
{
    Object *receiver;
    int method;         //槽函数字符串索引
};
typedef std::multimap<int, Connection> ConnectionMap;
...
class Object{
    ...
private:
    // key:信号函数索引 value:连接信息(接收者指针，接收者槽函数索引)
     ConnectionMap connections;
};
```

然后定义一个connect方法:

```
void Object::connect(Object* sender, const char* sig, Object* receiver, const char* slt)
{
    int sig_idx = find_substr_index(sender->meta.sig_names, sig);
    int slt_idx = find_substr_index(receiver->meta.slts_names, slt);
    if (sig_idx == -1 || slt_idx == -1) {
        perror("signal or slot not found!");
    } else {
        Connection c = {receiver, slt_idx};
        sender->connections.insert(std::pair<int, Connection>(sig_idx, c));
    }
}
```

到此，类中的信号和槽相关信息都已经完成，接下来就是信号的触发了。

```
#define slots
#define signals protected
#define emit
...
signals:
    void click();
public slots:
    void onClick();
```

信号和槽函数其实都是普通的函数，类似signals、slots关键字都只是一个标识，信号和槽的区别是，槽函数需要我们自己去实现，而信号函数会由moc替我们实现在moc_xxx.cpp文件中，而这里，我们需要自己去实现这个信号对应的函数。
这里又用到之前的元对象,在其中添加active方法

```
//object.h
struct MetaObject
{
    const char * sig_names;     //存放信号函数字符串
    const char * slts_names;    //存放槽函数字符串
    static void active(Object * sender, int idx);
};
//object.cpp
void MetaObject::active(Object* sender, int idx)
{
    auto ret = sender->connections.equal_range(idx);
    for (auto it = ret.first; it != ret.second; ++it) {
        Connection c = (*it).second;
        //按索引调用receiver对应的槽函数
        c.receiver->metacall(c.method);
    }
}
//moc_object.cpp
void Object::click()
{
    // 0 即为当前信号名字符串在meta.sig_names中的索引位置
    MetaObject::active(this, 0);
}

//通过索引调用槽函数 
void Object::metacall(int idx)
{
    switch (idx) {
        case 0:
        // 0 即为此槽函数名字符串在meta.slts_names中的索引位置
            onClick();
            break;
        default:
            break;
    };
}

```

至此，一个用C++实现的简单的信号槽就完成了，简单编译测试下

```
g++ main.cpp object.cpp moc_object.cpp -o test.exe
$ ./test.exe
Hello Qt!
```

我现在使用的Qt版本是5.15.2，信号槽相关的源代码已经特别的多，充斥着各种宏和模板的奇计淫巧，还有多线程等其他特性相关的代码，看起来比较吃力，但也可以看到信号槽最基本的概念大体如此。这篇文章就先到这里，后续可能会写一篇对Qt5.15.2源代码中信号槽机制具体实现的分析。

## 参考链接
- [Using the Meta-Object Compiler (moc)](https://doc.qt.io/archives/qt-4.8/moc.html)
- [用ISO C++实现自己的信号槽(Qt另类学习)](https://blog.csdn.net/dbzhang800/article/details/6376422)