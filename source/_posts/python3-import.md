---
title: python3_import
date: 2019-03-31 14:45:42
tags:
- python3
- import
- 
---

# Python3 Import 之 finder
&ensp;&ensp;&ensp;&ensp;在Python 中,导入一个模块最常见的方式就是使用 __import__ 语句, import 语句本质是对 __\_\_import\_\___ 语句的重写,不过这里并不多说, import 实际上分为两个过程,第一部分是搜索指定名称的模块,然后返回一个说明对象,另一个部分是根据前者返回的说明对象进行加载,这里我们首先说第一部分.

## sys.modules
&ensp;&ensp;&ensp;&ensp;首先,Python 会对sys.modules 进行搜索, modules 这里起到了缓存之前导入的所有模块的作用. 如果在搜索 sys.modules 中找模块名称关联的值,那么说明其不是第一次导入,其关联的值就是需要导入的模块,导入过程完成.

&ensp;&ensp;&ensp;&ensp;这里有需要注意的几点
- sys.modules是可写的 删除key 并不会破坏关联的模块(因为其他模块可能会保留对它的引用),但会使模块名的缓存目录失效,导致下次导入时,重新搜索.
- 如果 sys.modules 中存在指定的名称，但是关联值却为 __none__ 那么会引发 __ModuleNotFoundError。__

&ensp;&ensp;&ensp;&ensp;如果指定名称的模块在 sys.modules 找不到，则将发起调用 Python 的导入协议以查找和加载该模块. 此协议由两个概念性模块构成，即 __查找器__ 和 __加载器__ .查找器的任务是确定是否能使用其所知的策略找到该名称的模块。 同时实现这两种接口的对象称为 __导入器__ 它们在确定能加载所需的模块时会返回其自身。

## sys.meta_path
&ensp;&ensp;&ensp;&ensp; Python默认包含三个导入器, 第一个知道如何定位内置模块，第二个知道如何定位冻结模块。 第三个默认查找器会在 导入路径中(一般为sys.path) 中搜索模块。 导入路径 是一个由文件系统路径或 zip 文件组成的位置列表。 它还可以扩展为搜索任意可定位资源，例如由 URL 指定的资源。

PS: 可以在sys.meta_path 看到默认包含的三个导入器

PS:导入机制是可扩展的，因此可以加入新的查找器以扩展模块搜索的范围和作用域

## find_spec()
&ensp;&ensp;&ensp;&ensp; Python 中,如果你在sys.modules中无法找到模块,那么会搜索sys.meta_path ,其中包含 __元路径查找器__ 对象列表,查找器必须实现 find_spec() 这一方法，该方法接受三个参数：名称、导入路径和目标模块（可选，元路径查找器可使用任何策略来确定它是否能处理指定名称的模块。

如果元路径查找器知道如何处理指定名称的模块，返回一个说明对象，不行则返回None,如果 sys.meta_path 处理过程到达结尾，仍未返回说明对象，引发ModuleNotFoundError 并且向上传播，并放弃导入过程。

find_spec() 方法调用带有两到三个参数。 第一个是被导入模块的完整限定名称，例如 foo.bar.baz。 第二个参数是供模块搜索使用的路径条目。 对于最高层级模块，第二个参数为 None，但对于子模块或子包，第二个参数为父包__path__属性的值。 如果相应的__path__属性无法访问，将引发 ModuleNotFoundError。 第三个参数是一个将被作为稍后加载目标的现有模块对象。 导入系统仅会在重加载期间传入一个目标模块。

## PathFinder
Python 的默认 sys.meta_path 具有三种元路径查找器
- 内置模块导入(BuiltinImporter)
- 冻结模块导入(FrozenImporter)
- 路径查找导入(PathFinder)

这里主要说第三种,也即PathFinder,它会搜索包含一个 路径条目 的列表，列表中每个路径代表着一个用于搜索模块的位置

PathFinder 会有 __sys.path__ , __sys.path_hooks__ 和 __sys.path_importer_cache__ 三个变量供它使用,与此同时，在导入次级包的时候，顶级包的__path__也会被使用,不过经常被使用的是 __sys.path__ 和顶级包的__path__

__sys.path__ 包含一个提供模块和包搜索位置的字符串列表。 它初始化自 PYTHONPATH 环境变量以及多种其他特定安装和实现的默认设置。 sys.path 条目可指定的名称有文件系统中的目录、zip 文件和其他可用于搜索模块的潜在“位置”（参见 site 模块），例如 URL 或数据库查询等。 在 sys.path 中只能出现字符串和字节串；所有其他数据类型都会被忽略。

在Python 编译运行中,sys.path 会把启动文件的目录放在这个列表中的第一位，
与此同时，如果我们想要方便快捷的管理自己的模块，也可以在其中插入自己的路径，以供搜索。

### TODO
- [ ] 重写
- [ ] 添加子模块导入的内容
- [ ] 添加图表

### 参考链接
https://docs.python.org/3.7/reference/import.html