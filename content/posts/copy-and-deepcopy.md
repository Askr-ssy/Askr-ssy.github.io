---
title: copy_and_deepcopy
date: 2018-12-22 13:43:36
tags: [python3, copy]
description: python3 三种复制的区别
comments: true
categories: ["python3"]

---
## copy and deepcopy
在python3 中 复制分为三种,即 "赋值" "浅拷贝" "深拷贝".python 官方文档解释如下
```text
Assignment statements in Python do not copy objects, they create bindings between a target and an object.

For collections that are mutable or contain mutable items, a copy is sometimes needed so one can change one copy without changing the other.

This module provides generic shallow and deep copy operations (explained below).
```

---
### 赋值
赋值即利用 "=" 号运算符进行赋值, 但是这种赋值其本质是 “引用”, 即 重新创建一个变量名,然后绑定到 "=" 号右值的地址,这种情况下,**任何对新创建的变量进行的修改都会同步到旧的变量上!**

案例如下:
```python3
>>> old = 2
>>> new = old
>>> id(old)
140721449431328
>>> id(new)
140721449431328

>>> old = [1,2,3]
>>> new = old
>>> old[0]="1"
>>> new
['1', 2, 3]
>>> old is new
True
>>> id(old)==id(new)
True
```
PS:总结来说,则是**创建了一个新的变量名.然后指向旧变量的地址**

---
### 浅拷贝
第二种即是我们常说的 **浅拷贝** 这种拷贝是在于自定义的数据类型 被 "=" 运算符使用时所发生的.这种情况下 任何简易的修改(指 第一层引用的修改) 并不会改动另一个变量,但是在修改深处的值时,则依旧会影响到
```python3
>>> import copy
>>> old = [1,2,[3,4]]
>>> new = copy.copy(old)
>>> old[0]="1"
>>> old
['1', 2, [3, 4]]
>>> new
[1, 2, [3, 4]]
>>> old[2][0] = "3"
>>> old
['1', 2, ['3', 4]]
>>> new
[1, 2, ['3', 4]]
```
PS:总结来说,**浅拷贝在复制的时候,对于变量的第一层地址.创建了新的空间.但是对于第二层以下的地址.则是直接偷懒.把地址复制了赋予左边,并没有开创同等的空间**

---
### 深拷贝
第三种 深拷贝 才是真正意义上的完全拷贝.即重新创建与右值同样大小的空间.然后循环复制.这种情况下.新创建的变量无论如何修改都不会影响到旧的变量

```python3
>>> import copy
>>> old = [1,2,[3,4]]
>>> new = copy.deepcopy(old)
>>> new
[1, 2, [3, 4]]
>>> new[0]="1"
>>> new[2][0]="3"
>>> new
['1', 2, ['3', 4]]
>>> old
[1, 2, [3, 4]]
```

#### 引用
https://docs.python.org/3/library/copy.html
##### TODO
    [ ] 加上图片