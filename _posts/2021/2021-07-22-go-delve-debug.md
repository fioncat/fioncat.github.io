---
layout:       post
title:        "使用Delve调试go web请求"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Go
    - Debug
---

[Delve](https://github.com/go-delve/delve)是Golang的一个debugger，比起gdb，它有更多对于go的原生支持，例如gdb只能查看运行时线程，而delve可以查看goroutine。因此对于gopher来说使用delve似乎是个更加不错的选择。

网上已经有了很多delve调试简单程序的例子，但是在实际项目中，我们一般会开发大型的web工程，有时候需要在本地通过断点单独调试http接口。我发现网上关于这一块的教程较少，因此特此写了这篇文章供参考。

## Install/Update delve

对于`go1.16`以及以后的版本，其实可以通过一个命令快速完成delve的安装或升级，在go module打开的情况下，执行：

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

对于以前的版本，还是需要手动clone下来然后进行install：

```bash
git clone https://github.com/go-delve/delve
cd delve
go install github.com/go-delve/delve/cmd/dlv
```

## 编写http server

为了示例，我编写一个非常简单的http服务器，它仅包含一个接口：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func simpleHandler(w http.ResponseWriter, r *http.Request) {
    sum := 0
    for i := 1; i <= 5; i++ {
        sum += i
    }
    _, err := fmt.Fprintf(w, "sum = %d\n", sum)
    if err != nil {
        log.Printf("write response: %v", err)
    }
}

func main() {
    http.HandleFunc("/sum", simpleHandler)
    err := http.ListenAndServe(":1234", nil)
    if err != nil {
        log.Fatalf("listen failed: %v", err)
    }
}
```

这段程序有一个`/sum`接口，它计算1到5的和然后返回。我们使用`go run main.go`运行之后，可以直接使用curl调用接口：

```bash
curl 'http://127.0.0.1:1234/sum'
# sum = 15
```

## attach

delve有多种方式调试一个go程序，可以直接使用`dlv debug main.go`命令构建执行程序并进入调试模式，也可以使用attach来连接到一个已经运行起来的go程序中。

对于`debug`的用法，网上已经有很多教程了，这里不再阐述，例如：Delve调试器。而在服务器场景中，我们可能更希望服务器能够一直运行而不是不断临时启动调试，所以我这里重点介绍一下attach的形式：

在进行调试前，需要先启动服务器：

```bash
go build -gcflags "-N -l" main.go
./main
```

注意加上`-gcflags "-N -l"`，表示关闭一些优化，以让delve能够查看到所有的变量信息。

然后我们需要获取到这个进程的`pid`，这可以通过查看端口监听来实现：

```bash
lsof -i tcp:1234
# COMMAND   PID    USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
# main    33050    ...    ..   ...              ...             ...
```

在这个例子中`pid`是`33050`，有了pid，就可以让delve连接并调试这个进程了：

```bash
dlv attach 33050
# Type 'help' for list of commands.
# (dlv)
```

到这里，就进入了调试模式，可以输入各种命令打断点，并进行调试了。

## debug

首先，我们需要为需要的接口打断点。可以通过`filename:line_num`或`module.function`的方式来打断点。

在上面的例子中，我们想给`simpleHandler`打断点，可以通过下面两种形式：

```text
(dlv) break main.go:9  # 通过文件名的方式
Breakpoint 1 (enabled) set at 0x1222f1b for main.simpleHandler() ./Desktop/golang/projects/test/main.go:9
(dlv) break main.simpleHandler  # 通过函数名的方式
Breakpoint 1 (enabled) set at 0x1222f1b for main.simpleHandler() ./Desktop/golang/projects/test/main.go:9
```

打完断点之后，不要急着执行`continue`，这时候程序是阻塞的状态，我们需要至少执行一个http请求才能让程序有机会执行`simpleHandler`函数以触发断点，因此先调用接口：

```bash
curl 'http://127.0.0.1:1234/sum'
... # 这里会阻塞住，不要退出
```

这时候你会发现请求会被阻塞住。我们需要回到delve，然后执行continue命令让程序继续执行，因为调用了接口，所以接下来就会触发断点从而开始调试了：

```text
(dlv) continue
> main.simpleHandler() ./Desktop/golang/projects/test/main.go:9 (hits goroutine(6):1 total:1) (PC: 0x1222f1b)
     4:         "fmt"
     5:         "log"
     6:         "net/http"
     7: )
     8:
=>   9: func simpleHandler(w http.ResponseWriter, r *http.Request) {
    10:         sum := 0
    11:         for i := 1; i <= 5; i++ {
    12:                 sum += i
    13:         }
    14:         _, err := fmt.Fprintf(w, "sum = %d\n", sum)
```

这时候程序成功阻塞在了断点处，我们可以输入一系列命令进行调试，可以输入`help`查看，用的比较多的是：

- `c`：继续执行直到遇到下一个断点
- `l`：显示当前调试代码
- `locals [pattern]`：显示所有局部变量，可以传入pattern根据名称过滤
- `vars [pattern]`：显示所有全局变量，可以传入pattern根据名称过滤
- `args [pattern]`：显示当前函数的参数，可以传入pattern根据名称过滤
- `p [var]`：显示某个变量的具体值，如果变量是结构体，这会详细显示结构体的所有字段
- `n`：继续执行，如果有函数不会进入
- `s`：继续执行，如果有函数会进入
- `so`：跳出函数

例如，在上面的例子，当我们执行到12行时，可以通过`locals`查看所有局部变量：

```text
(dlv) n
> main.simpleHandler() ./Desktop/golang/projects/test/main.go:12 (PC: 0x1222f50)
     7: )
     8:
     9: func simpleHandler(w http.ResponseWriter, r *http.Request) {
    10:         sum := 0
    11:         for i := 1; i <= 5; i++ {
=>  12:                 sum += i
    13:         }
    14:         _, err := fmt.Fprintf(w, "sum = %d\n", sum)
    15:         if err != nil {
    16:                 log.Printf("write response: %v", err)
    17:         }

(dlv) locals
sum = 0
i = 1
```

不断执行`n`，你会发现`sum`和`i`都在递增。

假如我们想查看当前请求的Header信息，我们知道请求数据都储存在参数`r *http.Request`中，可以通过p命令来查看某个变量的详细信息：

```text
(dlv) p r
*net/http.Request {
    ....
```

这会输出很多信息，而我们只关心Header这个字段，因此可以进一步做过滤：

```text
(dlv) p r.Header
net/http.Header [
        "User-Agent": [
                "curl/7.64.1",
        ],
        "Accept": ["*/*"],
]
```

delve还可以做很多事情，例如查看当前运行的goroutine等，这里不再演示，使用者可以在调试中自行探索。

我们调试完毕之后，输入`exit`命令可以退出调试，这时delve会问你要不要关闭go进程，如果在生产环境调试这里千万不要选择`y`，否则会关闭生产服务器：

```text
(dlv) exit
Would you like to kill the process? [Y/n] n
```