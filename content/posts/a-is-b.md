---
title: a is b ?
date: 2020-03-30 23:20:00
tags:
- python3
- 小数据池
- cpython
- 驻留机制
- intern
description: Cpython 的驻留模式
comments: true
---

今天在社区 遇到一个蛮有意思的问题,分享出来给大家

## 问题

```python3
//猜测一下运行结果是否为True
a = "some_string"
b = "some" + "_" + "string"

c = "hello"
d = "hello"

e = "hello!"
f = "hello!"

g,h = "hello!","hello!"

print(a is b, c is d, e is f,g is h)
```

## 第一次解答

刚开始,在没有实际运行的情况下,因为考虑到小数据池的原因,猜测答案为 false true true true

具体解释为

1.无法确定 解释器否会把表达式求值后在进行引用小数据池中的内容，所以未false

2.很简单的小数据池 引用 所以为 true

3.同上

4.同上

---
## 实际答案

然而,在Cpython 3.6.9 实际运行后,答案为 True True False True

---
## 解惑

发现实际答案后，完全能够理解第一个的结果为true,解释器可能会在表达式求值后，进行对左值的赋值或引用，然而3,4结果确十分迷惑,初步猜想是达到小数据池的阈值,可是g,h 的结果所违反,在和其他人讨论后,指向了Cpython的内部实现机制-驻留机制

通常意义上来说,我们所说的Pyhton,也即Cpython的小数据池,就是由 __驻留机制(intern)__ 所实现的,驻留机制对于字符串的隐式驻留,有一道筛选 即 __只有包含ASCII字母，数字或下划线的字符串才会被隐式驻留__ 所以,e与f 才会被判定为不相似.在g,h中，也许是Cpython的进一步优化,在同一语句中,只创建了一个对象,然后两个引用. __当然,这并不能说明其发生了隐式驻留.__

```python3
b = "wtf!", "wtf!"
c = "wtf!"
a is b
>>> True
a is c
>>> False
b is c
>>> False
```

这里可以看到 c 与 a 或 b 并不相同,并且在进一步尝试下发现,空字符串和单个字符串(包含非法字符)也会被驻留
```python3
a = ''
b = ''
a is b
>>> True
a = '_'
b = '_'
a is b
>>> True
a = '&'
b = '&'
a is b
>>> True
```

当然,既然有隐式驻留,那么就有显示驻留,翻了一下相关文档后, 使用sys.intern 即可
```python3
a = intern("wtf!")
b = intern("wtf!")
a is b
>>> True
```

## TODO
- [ ] 优化文案
- [ ] 尝试查看Cpython相关文档

