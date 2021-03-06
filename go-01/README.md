Golang源码探索(一) 编译和调试源码

GO可以说是近几年最热门的新兴语言之一了, 一般人看到**分布式**和**大数据**就会想到GO,
这个系列的文章会通过研究golang的源代码来分析内部的实现原理,
和CoreCLR不同的是, golang的源代码已经被很多人研究过了, 我将会着重研究他们未提到过的部分.

另一点和CoreCLR不同的是, golang的源代码**非常易懂**, 注释也**非常的丰富**,
很明显Google的工程师在写代码的时候有考虑其他人会去看这份代码.

尽管代码非常易懂, 研究它们还是需要实际运行和调试才能得到更好的理解,
这个系列分析的golang源代码是Google官方的实现的1.9.2版本, 不适用于其他版本和gccgo等其他实现,
运行环境是Ubuntu 16.04 LTS 64bit.

# 编译golang源代码

go的源代码是用go写的, 编译也需要一个可运行的go.
首先我们从官网下载源代码和二进制文件.

[go1.9.2.src.tar.gz](https://redirector.gvt1.com/edgedl/go/go1.9.2.src.tar.gz)
[go1.9.2.linux-amd64.tar.gz](https://redirector.gvt1.com/edgedl/go/go1.9.2.linux-amd64.tar.gz)

注意两个压缩包解压出来文件夹名称都是go, 我们解压到以下目录:

``` text
源代码: ~/git_go/go_src
二进制: ~/git_go/go_bin
```

![](881857-20171103115003716-874672127.jpg)

编译go之前需要设置环境变量,
`GOROOT_BOOTSTRAP`是go二进制文件夹的所在目录,
`GO_GCFLAGS`是编译go时使用的参数.

```
export GOROOT_BOOTSTRAP=~/git_go/go_bin
export GO_GCFLAGS="-N -l"
```

这里的`-N`参数代表禁止优化, `-l`参数代表禁止内联, go在编译目标程序的时候会嵌入运行时(runtime)的二进制,
禁止优化和内联可以让运行时(runtime)中的函数变得更容易调试.

都准备好以后就可以进入go的源代码文件夹执行`all.bash`编译了:

![](881857-20171103115009576-92975619.jpg)

编译的结果在`~/git_go/go_src/bin`下:

![](881857-20171103115014045-1759540924.jpg)

# 调试golang源代码

之前CoreCLR的系列中我使用了lldb, 在这个系列中我继续沿用这个调试器.
这个系列中使用的是lldb 4.0.

以以下源代码(hello.go)为例:

``` go
package main

import (
	"fmt"
	"time"
)

func printNumber(from, to int, c chan int) {
	for x := from; x <= to; x++ {
		fmt.Printf("%d\n", x)
		time.Sleep(1 * time.Millisecond)
	}
	c <- 0
}

func main() {
	c := make(chan int, 3)
	go printNumber(1, 3, c)
	go printNumber(4, 6, c)
	_, _ = <- c, <- c
}
```

编译源代码使用以下命令, 这里的`-l`参数的意思和上面一样, 如果有需要还可以加`-N`参数:

``` text
~/git_go/go_src/bin/go build -gcflags "-l" hello.go
```

编译后使用lldb运行:

``` text
lldb ./hello
```

![](881857-20171103115022748-1405848210.jpg)

go里面的函数符号名称的命名规则是`包名称.函数名称`, 例如主函数的符号名称是`main.main`, 运行时中的`newobject`的符号名称是`runtime.newobject`.
首先给主函数下一个断点然后运行:

![](881857-20171103115028076-1600078782.jpg)

可以看到成功的进入了主函数, 并且有源代码提示.
接下来给按文件名和行数来下断点:

![](881857-20171103115036029-2126126710.jpg)

然后查看函数的汇编代码:

![](881857-20171103115040638-1929857151.jpg)

关于lldb的命令可以查看[这篇文档](http://lldb.llvm.org/tutorial.html).
在我使用的环境中lldb可以正常的下断点, 步进和步过go代码或者汇编指令,
但**打印变量输出的值有可能是错的**, 即使不开启优化.

虽然打印变量这个功能不好用, 我们仍然可以直接让go输出我们想要的值,
例如修改`runtime/malloc.go`输出当前环境下arena|spans|bitmap区的大小:

![](881857-20171103115047029-1330671788.jpg)

修改后进入`src`并执行`./make.bash`, 然后重新编译目标程序, 运行:

![](881857-20171103115053826-1500300209.jpg)

可以看到当前环境下arena是512G, spans是512M, bitmap是16G.
这个方法虽然比较笨, 但是可以在任何情况下输出我们想要的值.

此外, go运行时(runtime)的源代码会包括在目标文件中,
例如你对`runtime.newobject`下断点可以对go自身的源代码进行调试.

# 参考链接

https://golang.org
https://golang.org/doc/install/source
https://golang.org/doc/gdb
http://lldb.llvm.org/tutorial.html
http://legendtkl.com/archives

接下来我将分析golang的任务调度机制和三色GC的具体实现, 敬请期待.