---
layout: single
title:  "如何使用go tool pprof"
categories:
  - Written in Chinese
tags:
  - tools
toc: false
excerpt_separator: <!--more-->
---

pprof是golang内置的性能调优工具，可以分析：

- 内存使用情况
- CPU使用情况
- go routine阻塞情况
- 互斥锁使用情况

等。同时，1.11版本以后，它集成了go-torch的功能，可以将这些profile可视化，让结果更直观。

<!--more-->

## 如何生成profile

有三种方式可以生成profile，具体可以参考runtime/pprof的[官方文档](<https://golang.org/pkg/runtime/pprof/>)。

### 运行测试时生成

go test集成了profile功能，它提供了一些选项，可以方便的生成profile文件。

```bash
go test -cpuprofile cpu.prof -memprofile mem.prof -bench .
```

### 使用runtime/pprof库来生成

对于独立的可执行程序，如果我们想生成profile文件，可以添加一些代码调用runtime/pprof库来生成他们，官方文档的例子如下。

```go
var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to `file`")
var memprofile = flag.String("memprofile", "", "write memory profile to `file`")

func main() {
    flag.Parse()
    if *cpuprofile != "" {
        f, err := os.Create(*cpuprofile)
        if err != nil {
            log.Fatal("could not create CPU profile: ", err)
        }
        defer f.Close()
        if err := pprof.StartCPUProfile(f); err != nil {
            log.Fatal("could not start CPU profile: ", err)
        }
        defer pprof.StopCPUProfile()
    }

    // ... rest of the program ...

    if *memprofile != "" {
        f, err := os.Create(*memprofile)
        if err != nil {
            log.Fatal("could not create memory profile: ", err)
        }
        defer f.Close()
        runtime.GC() // get up-to-date statistics
        if err := pprof.WriteHeapProfile(f); err != nil {
            log.Fatal("could not write memory profile: ", err)
        }
    }
}
```

### 导入net/http/pprof包

还有一种可以动态获取profile数据的方式是在代码中导入net/http/pprof这个包。它会注册一个HTTP handler，然后我们就可以通过HTTP调用的方式来动态的获取profile。

```go
import _ "net/http/pprof"
```

## 如何分析profile

有了profile文件，我们就可以使用go提供的命令行工具来分析他们。

```bash
go tool pprof cpu.prof
```

这种方式需要我们很熟悉pprof这个工具，需要熟练使用它的各个选项，并且很多数据呈现在命令行，不是很直观。当然，它有一个子命令web，它会打开浏览器，以图像的方式来呈现profile数据。

pprof还可以直接启动一个HTTP server来对外服务profile数据，它有很多丰富的展示选项，除了默认的图形化的展示以外，还可以展示火焰图。使用它可以让我们快速、直观的浏览profile数据。

```bash
go tool pprof -http=":8081" [binary] [profile]
```

