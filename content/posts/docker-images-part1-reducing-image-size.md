---
title: '[翻译]Docker镜像:第一部分-减少镜像大小'
date: 2020-04-03 17:38:13
tags:
- docker
- 翻译

description: 如何合理的缩减Docker镜像大小
---
该文为翻译文章,因文章写的太好,见猎心喜,未拿到原作者授权,侵删.

[原文链接](https://www.ardanlabs.com/blog/2020/02/docker-images-part1-reducing-image-size.html)

以下为正文

----

### 介绍
在刚开始使用容器的时候,很容易被我们所生成的镜像大小震惊.我们将回顾许多在不缩减开发人员和维护人员的便捷性的情况下,同时缩减镜像尺寸的技术.在第一部分中,我们将讨论多阶段构建,因为任何想要缩减镜像大小的人都需要从这开始.我们还将说明静态链接和动态链接之间的区别,以及为什么我们要关心它.同时,这也是介绍Alpine[^1]的机会.

在第二部分,我们将看到与各种流行语言有关的一些特殊性.我们将讨论 GO,以及Java,Node,Python,Ruby,Rust.我们还将讨论有关Alpine的更多信息,以及如何全面利用Alpine.

在第三部分中,我们将介绍一些与大多数语言和框架相关的 Pattern ( and anti-patterns! ),例如使用通用基础镜像,分离二进制文件并减小大小.我们将总结一些更奇特的或高级的方法,例如 Bazel,Distroless,DockerSlim或UPX . 我们将看到其中的一些方法在某些情况下会适得其反,但在某些特定情况下可能会有用.

请注意,示例代码以及此处提到的所有Dockerfile,都可以在公共GitHub存储库中方便地获得,并带有一个Compose文件来构建所有镜像并轻松比较它们的大小.

### 我们正在尝试解决的问题

我敢打赌,每个构建了第一个Docker镜像并编译了一些代码的人都对该镜像的大小感到惊讶.

看看用C编写的这个繁琐的 "hello world" 程序：
```c
/* hello.c */
int main () {
  puts("Hello, world!");
  return 0;
}
```

我们可以使用下面这个Dockerfile构建它：

```Dockerfile
FROM gcc
COPY hello.c .
RUN gcc -o hello hello.c
CMD ["./hello"]
```
但是生成的镜像将超过1 GB,因为它包含着整个gcc镜像！

如果我们使用Ubuntu镜像,安装一个C编译器,然后编译程序,我们将得到300 MB镜像；看起来更棒,但对于本身小于20 kB的二进制文件而言,仍然太多了：
```shell
$ ls -l hello
-rwxr-xr-x   1 root root 16384 Nov 18 14:36 hello
```
一个同等效果的Go程序：
```go
package main

import "fmt"

func main () {
  fmt.Println("Hello, world!")
}
```
如果使用该golang镜像构建此代码,即使hello程序只有2 MB ,生成的镜像仍为800 MB：

```shell
$ ls -l hello
-rwxr-xr-x 1 root root 2008801 Jan 15 16:41 hello
```
一定有更好的方法！

让我们看看如何大幅减少这些镜像的大小.在某些情况下,我们将实现99.8％的尺寸减小(但是我们会发现,跨度这么大并不总是一个好主意).

专业提示：为了轻松比较镜像的大小,我们将使用相同的镜像名称,但使用不同的标签.举例来说,我们的形象会hello:gcc,hello:ubuntu,hello:thisweirdtrick等这样的话,这样我们可以运行docker images hello,它会列出所有的标签为hello镜像,与它们的大小,而不会受到Docker引擎上成千上万的其他镜像的困扰.

### 多阶段构建

这是减小镜像尺寸的第一步(也是最激烈的一步).这里我们需要小心,因为如果处理不正确,可能会导致镜像出现问题(甚至可能完全损坏).

多阶段构建来自一个简单的想法：“我不需要在最终的应用程序镜像中包括C或Go编译器以及整个构建工具链.我只想运送二进制文件！”

我们通过FROM在Dockerfile中添加另一行来获得多阶段构建.看下面的例子：
```dockerfile
FROM gcc AS mybuildstage
COPY hello.c .
RUN gcc -o hello hello.c
FROM ubuntu
COPY --from=mybuildstage hello .
CMD ["./hello"]
```
我们使用gcc镜像来构建hello.c程序.然后,我们使用该ubuntu镜像开始一个新阶段(我们称为“运行阶段”).我们的hello从上一阶段复制二进制文件.最终镜像为64 MB,而不是1.1 GB,因此大小减少了约95％：
```text
$ docker images minimage
REPOSITORY          TAG                    ...         SIZE
minimage            hello-c.gcc            ...         1.14GB
minimage            hello-c.gcc.ubuntu     ...         64.2MB
```
还不错吧？我们可以做得更好.但是首先,需要注意一些技巧和警告.

在声明构建阶段时,您不必使用AS关键字.从上一个阶段复制文件时,您只需指明该构建阶段的编号(从零开始).

换句话说,以下两行是相同的：

```dockerfile
COPY --from=mybuildstage hello .
COPY --from=0 hello .
```
就个人而言,我认为在较短的Dockerfile(例如,不超过10行)的构建阶段中使用数字是很好的,但是,只要您的Dockerfile变长(并且可能更复杂,具有多个构建阶段),命名是一个好主意.明确的阶段.这将有助于您的团队成员的维护(以及将来您将在几个月后复习该Dockerfile的将来).
#### 警告：使用经典镜像
我强烈建议您在“运行”阶段坚持使用经典镜像.我所说的“经典”是指CentOS,Debian,Fedora,Ubuntu;以及其他熟悉的东西.您可能听说过Alpine,并很想使用它.暂时不要这么做！至少还没有到使用它们的时候.稍后我们将讨论Alpine,并解释为什么我们需要谨慎使用Alpine.

#### 警告：COPY --from使用绝对路径
从上一阶段复制文件时,路径被解释为相对于上一阶段的根目录.

在我们使用带有WORKDIR的生成器镜像(例如golang镜像)后,问题就会出现.

例如我们尝试构建此Dockerfile：
```dockerfile
FROM golang
COPY hello.go .
RUN go build hello.go
FROM ubuntu
COPY --from=0 hello .
CMD ["./hello"]
```
我们收到与以下错误类似的错误：

```text
COPY failed: stat /var/lib/docker/overlay2/1be...868/merged/hello: no such file or directory
```
这是因为COPY命令尝试复制 /hello,但是由于WORKDIR 在 golang是/go,所以程序路径实际上是/go/hello.

如果我们在构建中使用的是正式(或非常稳定)的镜像,通过指定完整的绝对路径,使得我们不必理会它.

但是,如果将来我们的构建或运行可能会改变这个镜像,那么我建议WORKDIR在构建镜像中指定一个.这将确保文件出现在期望的位置,即使我们用于构建阶段的基本镜像以后也会更改.

遵循这一原则,用于构建Go程序的Dockerfile如下所示：
```dockerfile
FROM golang
WORKDIR /src
COPY hello.go .
RUN go build hello.go
FROM ubuntu
COPY --from=0 /src/hello .
CMD ["./hello"]
```
如果您想知道Golang多阶段构建的效果,OK,它们让我们从800 MB的镜像下降到66 MB的镜像：
```text
$ docker images minimage
REPOSITORY     TAG                              ...    SIZE
minimage       hello-go.golang                  ...    805MB
minimage       hello-go.golang.ubuntu-workdir   ...    66.2MB
```
### 使用 FROM scratch
回到我们的“ Hello World”程序.C版本为16 kB,Go版本为2 MB.我们可以得到这么大的镜像吗？

我们可以仅使用二进制文件而不用其他文件来构建镜像吗？

是的! 我们要做的就是使用多阶段构建,然后选择scratch运行镜像.scratch是虚拟镜像.您不能拉或运行它,因为它完全是空的.这就是为什么Dockerfile 如果以 FROM scratch 开始,则意味着我们是从零开始构建的,因为没有使用任何预先存在的东西.

这为我们提供了以下Dockerfile：
```dockerfile
FROM golang
COPY hello.go .
RUN go build hello.go
FROM scratch
COPY --from=0 /go/hello .
CMD ["./hello"]
```
如果我们构建该镜像,则其大小恰好是二进制文件的大小(2 MB),并且可以正常工作！

但是,当scratch用作基础时,需要记住一些注意事项.

#### 没有 shell
该scratch镜像没有 shell.这意味着我们不能将字符串语法与CMD(或RUN)一起使用.考虑以下Dockerfile：
```dockerfile
...
FROM scratch
COPY --from=0 /go/hello .
CMD ./hello
```
如果尝试docker run生成结果镜像,则会收到以下错误消息：
```
docker: Error response from daemon: OCI runtime create failed: container_linux.go:345: starting container process caused "exec: \"/bin/sh\": stat /bin/sh: no such file or directory": unknown.
```
它的显示结果不是很清楚,但是核心信息在这里：/bin/sh is missing from the image.

发生这种情况是因为当我们将字符串语法与CMD或RUN一起使用时,参数将传递给/bin/sh.这意味着我们CMD ./hello上面会执行/bin/sh -c "./hello",因为我们没有/bin/sh 在scratch镜像,这将导致失败.

解决方法很简单：在Dockerfile中使用JSON语法.CMD ./hello 换成 CMD ["./hello"].当Docker检测到JSON语法时,它将直接运行参数,而无需使用shell.

#### 没有调试工具
根据scratch定义,该镜像为空；因此它没有任何帮助我们解决容器问题的方法.没有 shell (正如我们在上一段说的)也没什么ls,ps,ping,等等等等.这意味着我们将无法输入容器(使用docker exec或kubectl exec进行查看).

(请注意,严格来说,有一些方法可以对我们的容器进行故障排除.我们可以用来docker cp将文件从容器中取出；我们可以docker run --net container:用来与网络堆栈进行交互；像这样的低级工具nsenter可能非常强大. Kubernetes的版本具有短暂容器的概念,但是它仍然处于alpha状态.请记住,所有这些技术肯定会使我们的生活变得更加复杂,尤其是当我们有很多事情要做的时候！)

这里的一个解决办法是使用 像busybox或alpine 这种镜像代替scratch.当然,它们更大(分别为1.2 MB和5.5 MB),但是在庞大的方案中,如果将其与原始镜像的数百兆字节或千兆字节进行比较起来,付出的代价很小.

#### 没有libc
这是一个很难解决的问题.我们在Go中使用简单的“ hello world”可以很好地工作,但是,如果我们尝试在scratch镜像中放置C程序,或者在更复杂的Go程序中(例如,使用网络封装的任何东西),则会收到以下错误消息：
```
standard_init_linux.go:211: exec user process caused "no such file or directory"
```
某些文件似乎丢失了.但这并不能告诉我们确切缺少哪个文件.

丢失的文件是运行我们的程序所必需的动态库.

什么是动态库,为什么我们需要它？

程序编译后,将与所使用的库链接.(很简单,我们的“ hello world”程序仍在使用库；这就是puts函数的来源.)很久以前(90年代之前),我们主要使用静态链接,这意味着a所使用的所有库程序将包含在二进制文件中.当从软盘或盒式磁带执行软件时,或者根本没有标准库时,这是完美的选择.但是,在Linux等分时系统上,我们运行许多并发程序,这些程序存储在硬盘上.这些程序几乎总是使用标准的C库.

在这种情况下,使用动态链接会更加有利.使用动态链接,最终的二进制文件不包含它使用的所有库的代码.相反,它包含这些库的引用,如“这个程序需要的功能 cos 和 sin 和 tan 在 libtrigonometry.so.执行程序时,系统会查找libtrigonometry.so该程序并将其加载到程序旁边,以便程序可以调用这些函数.

动态链接具有多个优点.

它节省了磁盘空间,因为不再需要复制通用库.
因为这些库可以从磁盘加载一次,然后使用它们在多个程序之间共享,所以可以节省内存.
这使维护更加容易,因为在更新库时,我们不需要使用该库重新编译所有程序.
(如果我们想更透彻一点,内存节省不是动态库的结果,而是共享库的结果.也就是说,两者通常是并存的.您知道吗,在Linux上,动态库文件通常具有扩展名.so,代表共享库吗？在Windows上是.DLL,它代表动态链接库.)

回到我们的故事：默认情况下,C程序是动态链接的.使用某些软件包的Go程序也是如此.我们的特定程序使用标准的C库,该库在最新的Linux系统上在文件中libc.so.6中.因此,要运行,我们的程序需要将该文件显示在容器镜像中.而且,如果我们使用scratch,则显然没有该文件.如果我们使用busybox或alpine,则是相同的,因为busybox它不包含标准库,并且alpine正在使用另一个不兼容的库.稍后我们将详细介绍.

我们该如何解决？至少有3个选项.
#### 构建静态库
我们可以告诉我们的工具链制作一个静态二进制文件.有多种方法可以实现该目标(首先要取决于我们如何构建程序),但是如果使用gcc,我们要做的就是添加-static到命令行中：
```shell
gcc -o hello hello.c -static
```
现在生成的二进制文件是760 kB(在我的系统上),而不是16 kB.当然,我们将库嵌入二进制文件中,因此更大.但是该二进制文件现在将在scratch镜像中正确运行.

如果使用Alpine构建静态二进制文件,则可以得到更小的镜像.结果小于100 kB！
#### 将库添加到我们的镜像
我们可以使用该ldd工具找出程序需要的库：
```shell
$ ldd hello
	linux-vdso.so.1 (0x00007ffdf8acb000)
	libc.so.6 => /usr/lib/libc.so.6 (0x00007ff897ef6000)
	/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007ff8980f7000)
```
我们可以看到程序所需的库,以及系统找到它们的确切路径.

在上面的示例中,唯一的“真实”库是libc.so.6.linux-vdso.so.1与一种称为VDSO(虚拟动态共享对象)的机制有关,该机制可以加速某些系统调用.让我们假装它不在那里.至于ld-linux-x86-64.so.2,它实际上是动态链接器本身.(从技术上讲,我们的hello二进制文件包含以下信息：“嘿,这是一个动态程序,并且知道如何将其所有部分放在一起的东西是ld-linux-x86-64.so.2”.)

如果我们愿意,可以将上面列出的所有文件手动添加ldd到镜像中.这将是相当繁琐且难以维护的,尤其是对于程序将有很多依赖性的情况.对于我们的 hello word 程序,这将正常的工作.但是对于更复杂的程序,例如使用DNS的程序,我们会遇到另一个问题.GNU C库(在大多数Linux系统上使用)通过称为名称服务开关(简称为NSS )的相当复杂的机制实现DNS(以及其他一些功能).该机制需要一个配置文件/etc/nsswitch.conf和其他库.但是这些库没有显示ldd,因为它们会在程序运行时稍后加载.如果我们希望DNS解析正常工作,我们仍然需要包括它们！(这些库通常位于/lib64/libnss_*.)

我个人不建议这样做,因为它非常神秘,难以维护,并且将来很可能会中断.
#### 使用 busybox:glibc
有专门为解决所有这些问题而设计的镜像：busybox:glibc.这是一个小镜像(5 MB),使用busybox(因此提供了许多用于故障排除和操作的有用工具)并提供了GNU C库(或glibc).该镜像恰好包含我们前面提到的所有这些讨厌的文件.如果要在小的镜像中运行动态二进制文件,则应使用此方法.

但是请记住,如果我们的程序使用其他库,则也需要复制这些库.
### 总结和(部分)结论
让我们看看我们如何在C. Spoiler警报中为“ hello world”程序做些事情：此列表包括通过使用Alpine所获得的结果,这将在本系列的下一部分中进行描述.

原始镜像内置gcc：1.14 GB
多级构建与gcc和ubuntu：64.2 MB
静态glibc二进制文件alpine：6.5 MB
动态二进制文件alpine：5.6 MB
静态二进制输入scratch：940 kB
静态musl二进制in scratch：94 kB

大小减少了12000倍,或磁盘空间减少了99.99％.

不错.

就个人而言,我不会选择 使用scratch这些镜像(因为对它们进行故障排除可能会很麻烦),但是,如果您要这样做,他们就会在这里为您服务！

在下一部分中,我们将介绍Go语言特定的一些方面,包括cgo和标签.我们还将介绍其他流行语言,并且我们将讨论更多有关Alpine的信息,非常欢迎您与我一起讨论




[^1]: [这里是校注](https://wiki.alpinelinux.org/wiki/Docker)