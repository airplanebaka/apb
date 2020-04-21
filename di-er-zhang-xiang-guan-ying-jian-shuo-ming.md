---
description: 简介
---

# 第二章 相关介绍说明

## Chisel简介

        在硬件设计上大家更加了解Verilog这门HDL语言。我们都知道Verilog语言太啰嗦，一个简单模块可能要写满屏幕的代码，导致可读性和维护性都很差。另一个大问题就是参数化很差，参数传递不清楚，封装不干净，导致代码量过大和复用性很低。

        其他方面将其他语言转换为Verilog，例如C到Verilog。但是这种转换效率低下，设计者必须学习电路知识，学习工具规范后才能开发，而且写硬件的语法还和写软件不一样，导致许多奇奇怪怪的错误，甚至转换后的代码也存在诸多BUG。这些问题将高级语言的高效性除去，增加了繁琐的操作，大大降低了效率。

        而一种语言打破了这种限制，那就是Chisel。 Chisel（Constructing Hardware In a Scala Embedded Language）是UC Berkeley开发的一种开源硬件构造语言。它是建构在Scala语言之上的领域专用语言（DSL），支持高度参数化的硬件生成器。Chisel是基于Scala的语言，所以它继承了便于封装和扩展的特性。chisel设计了bundle的概念，于是成千上百的端口互联只用一句程序就可以实现，干净简单还不容易出错。在编译器方面，chisel的编译器可以实现功能相当强大的错误检查。基本上，如果一个reg或者wire是中间变量，chisel不建议我们声明它的位宽，编译器会在编译过程中自动推断它的位宽。在一个连接链上，编译器会检查所有的东西位宽一致。这就杜绝了很多低级人为错误的出现。

        由于Chisel的学习具有较高的门槛，要求大家熟练掌握对面向对象编程，导致最近几年Chisel的热度下降，只有部分高校和公司在使用。

## 硬件说明

### CPU和SOC介绍

> 本小节参考别人的文档
>
> [https://moon548834.github.io/cyc10-rcore-tutorial/Chap2/2-0-hardware.html](https://moon548834.github.io/cyc10-rcore-tutorial/Chap2/2-0-hardware.html)

该cpu是经典的5级流水线架构，支持m态和s态特权级，并且具有mmu单元与tlb，无cache设计。

soc则基于《自己动手写CPU》\(雷思磊\)结构，采用的是wishbone总线，其中sdram使用的是开源代码，uart为参考文档作者编写\(为了节省资源，只支持发送功能，且只有一个端口，波特率写死，无ready位\)，片外flash目前尚不清楚如何工作，故采用片内rom，使用intel的.hex文件进行例化。总线交互也采用的是开源代码。





