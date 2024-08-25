---
title: Learn shared_ptr by Error
date: 2022-03-03 21:46:10
tags: C++、shared_ptr
---

> Shared_ptr is a smart pointer that retains shared ownership of an object through a pointer.
<!--more-->
shared_ptr 是一个通过指针来保有对象共享所有权的智能指针。 <br/>
智能指针是现代C++的重要特性之一，简单来说智能指针的出现就是为了解决原生指针使用时非常容易出现内存泄漏(例如忘记delete、函数异常提前退出、数据的所有权不明确)的问题。而智能指针有两个必须要实现的基本功能：
- 形如原生指针。智能指针的使用方法、对应操作的结果，以及在多态情况下的行为都应该与原生指针一致。
- 自动内存管理。使用智能指针时，程序员应该不需要负责内存的回收、并且在出现异常时应保证不会发生内存泄漏。

本文以shared_ptr为例子，通过不断的 **遇到问题(内存泄漏、编译错误、与原生指针行为不一致等)->找寻对应解决方案** 的模式，来阐述一份shared_ptr是如何从0到1实现、并完成以上两个功能的。

#### 工具准备
- 源代码: [shared_ptr](https://github.com/protocolbuffers/protobuf/blob/3.5.x/src/google/protobuf/stubs/shared_ptr.h) protocolbuffers库早期版本中使用的shared_ptr实现。简洁，易于学习。是本文主要参考的代码。
- 编译环境: Windows、MSVC2019
- 内存检测工具: [Visual Leak Detector](https://kinddragon.github.io/vld/)

## 正文
