---
title: cpu 命名规则 intel篇
date: 2018-11-12 20:20:03
tags:
- cpu
- 命名规则
- intel
- 杂谈
description: intel cpu 命名规则
comments: true
# mathjax: true
---
intel cpu 型号众多，大概清理了一下命名规则
## intel cpu 的产品系列
---
intel 分为 八个序列，分别为
```
    Core(酷睿)：主要面向中高端消费者，工作站和发烧级的处理器系列,推出时取代了当时的Pentium(奔腾)处理器的中高端定位，并将Pentium移至入门级系列，也将Celeron(赛扬)处理器推向低端系列。Xeon(至强)处理器用于服务器和工作站市场,但早期命名规则混乱，严格来说不算是连续性的品牌。
 
    Pentium：intel 早期推出的唯一处理器序列，后来衍生出低端的 Celeron 和 面向服务器的 Xeon 系列后，intel 又推出了 Core 系列取代 Pentium 系列后，彻底成为了低端处理器序列，市场定位在 Celeron 之上，Core 之下。

    Celeron: 早期由 Pentium 系列衍生出的低端型号~，在 Pentium 被 Core 取代后，市场定位更低了，彻底成为低端型号P

    Xeon：早期由 Pentium 系列衍生出来，主要供服务器及工作站使用，亦有超级计算机采用此处理器，历史悠久~。

    Xeon Phi(至强融核)：首款英特尔集成众核（Many Integrated Core，MIC）架构产品。用作高性能计算（HPC）的超级计算机或服务器的加速卡。

    Itanium(安腾)：是英特尔安腾架构（通常称之为IA-64）的64位处理器，市面上出现较少，近年来也没有消息。

    Atom(凌动)：Intel的一个超低电压处理器系列，和 Itanium 一样，近年比较少见。

    Quark(夸克)：Intel 最近新出的面向物联网的处理器序列，特点是小尺寸和低功耗。
```
不过这里因为 Xeon Phi,Itanium,Atom,Quark 过于少见,就以常见的Core 和 Xeon 的命名规则作为讲解。

---
### Core 命名规则
举一个例子，一个intel 完整的名称是：

<center>Intel Core i7 - 7500 U</center>

分开解析,Core 是品牌， i7 是标识符，标明其在品牌中的定位，7 代表着代数，是第七代处理器，500是 __S__ tock __K__ eeping __U__ nit,类似于图书的ISBN号，用于内部标识，但是根据实际情况，数值越大，性能越强。
U 代表着后缀，类似于修饰符的功能，一般有K(可超频)，T(功耗优化)，M(移动处理器)等。

在一般看到一款CPU型号的时候，一般来说在 标识符(i3,i5,i7)和代数(5th,6th,7th)相同的情况下，SKU越大，性能越强，对于性能比较来说，通常是 标识符>代数>SKU。

---
### Xeon 命名规则

Xeon 系列 比较复杂，下面由两个分支，这里先拿标准的至强处理器当作案例

<center> Intel Xeon E3 - 1230 V3 </center>

和之前一样，Intel Xeon 是品牌，E3 是生产线区别 ，也是标识符，1 指的是单主板下最大CPU数量，2 指的是 接口类型，30 指的是 SKU ，通常在SKU后面会有一些修饰符，例如 E3-1230L V3 这种，L表示的就是低功耗的修饰符，V3 则代表了这是这款CPU型号的第三代。

近年新出的 至强可扩展处理器，下面是一个经典的案例：

<center> Intel Xeon Pltinum 8180 m processor</center>

一样，Intel Xeon 是品牌，Platium 是其型号定位，8 是 SKU level ,表示性能和级别，可以理解型号定位的细化版本，1 是处理器的代数 80 是SKU，SKU后面有可能会有一到两个修饰符。


---
## 参考链接
https://ark.intel.com/content/www/cn/zh/ark.html#@Processors

---
## TODO
- [ ] 加入时间序列
- [ ] 撰写 amd 篇
- [ ] 加上图片
- [ ] 补充其他序列