## 资料来源

> https://go.dev/blog/using-go-modules

## 基本步骤

### 初始化和测试
创建源文件和测试文件

```go
// hello.go
package hello

func Hello() string {
    return "Hello, 世界"
}
```

```go
// hello_test.go
package hello

import "testing"

func TestHello(t *testing.T) {
    want := "Hello, 世界"
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
}
```

创建并初始化模块
```bash
$ go mod init example.com/hello
go: creating new go.mod: module example.com/hello
go: to add module requirements and sums:
        go mod tidy
```

测试源文件
```bash
$ go test
PASS
ok      example.com/hello       0.197s
```

### 导入依赖

导入依赖
```go
// hello.go
package hello

import "rsc.io/quote"

func Hello() string {
    return quote.Hello()
}
```

安装依赖

```bash
$ go get rsc.io/quote
go: added golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
go: added rsc.io/quote v1.5.2
go: added rsc.io/sampler v1.3.0
```

再次测试

```bash
$ go test
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "你好，世界。", want "Hello, 世界"
FAIL
exit status 1
FAIL    example.com/hello       0.213s
```

修改测试数据

```go
// hello_test.go
/* 第 6 行 */want := "你好，世界。"    
```

再次测试

```bash
$ go test
PASS
ok      example.com/hello       0.197s
```

查看此时的项目依赖

```bash
$ go list -m all  
example.com/hello
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
```

### 升级依赖（patch）

升级 golang.org/x/text 依赖。版本只变更 patch，无伤大雅。

```bash
$ go get golang.org/x/text
go: upgraded golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c => v0.3.7
```

再次测试，成功，项目成功更新依赖

```bash
$ go test
PASS
ok      example.com/hello       0.224s
```

查看当前版本依赖( `go mod tidy` 的时候 go.mod 文件有变化 )

```bash
$ go list -m all
example.com/hello
golang.org/x/text v0.3.7
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
$ go mod tidy
$ go list -m all
example.com/hello
golang.org/x/text v0.3.7
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
```

### 升级依赖（mirror）

现在依赖新版本。（未指定版本时，默认安装最新版本号）

```bash
$ go get rsc.io/sampler
go: upgraded rsc.io/sampler v1.3.0 => v1.99.99
```

再次测试。

```bash
$ go test
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of beer, ...", want "你好，世界 
。"
FAIL
exit status 1
FAIL    example.com/hello       0.212s
```

出错了，说明新版本依赖对源代码产生了影响（类比项目本来跑得动，结果已更新，就出 bug 了）。

按理来说，mirror 版本的迭代，需要兼容旧版本，但生活中充满意外 和 bug，何况这个版本是从 3 ————> 99 呢

查看有问题的依赖，有哪些版本

```bash
$ go list -m -versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99
```

“容易看出”，我们应该使用 v1.3.1 版本

版本换成 v1.3.1

```bash
$ go get rsc.io/sampler@v1.3.1
go: downgraded rsc.io/sampler v1.99.99 => v1.3.1
```

再次测试

```bash
$ go test
PASS
ok      example.com/hello       0.226s
```

版本升级成功了（1.3.0 --> 1.3.1）

### 升级主版本（major）

主版本即大改动，需要修改源码，但是不可能一次性迭代（大项目中）

（模拟大项目慢慢迭代）修改项目代码

```go
// hello.go
package hello

import (
    "rsc.io/quote"
    quoteV3 "rsc.io/quote/v3"
)

func Hello() string {
    return quote.Hello()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

测试代码需要新增一个函数来测试项目新的代码部分

```go
// hello_test.go 中新增这个函数
func TestProverb(t *testing.T) {
    want := "Concurrency is not parallelism."
    if got := Proverb(); got != want {
        t.Errorf("Proverb() = %q, want %q", got, want)
    }
}

```

安装新版本依赖（此时新旧共存）

```bash
$ go get rsc.io/quote/v3
go: added rsc.io/quote/v3 v3.1.0
```

测试一下

```bash
$ go test
PASS
ok      example.com/hello       0.242s
```

测试成功（项目可以慢慢迭代新版本）

查看一下当前 q 开头的依赖

```bash
$ go list -m rsc.io/q...
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
```

可以看到此时两个版本共存中

### 项目全面换新的主版本（major）

查看新版本依赖文档

```bash
$ go doc rsc.io/quote/v3
package quote // import "rsc.io/quote/v3"

Package quote collects pithy sayings.

func Concurrency() string
func GlassV3() string
func GoV3() string
func HelloV3() string
func OptV3() string
```

易得 Hello 函数迭代成了 Hello3 函数

（模拟项目代码全面更新）修改项目代码

```go
// hello.go
package hello

import "rsc.io/quote/v3"

func Hello() string {
    return quote.HelloV3()
}

func Proverb() string {
    return quote.Concurrency()
}
```

测试一下是否成功

```bash
$ go test
PASS
ok      example.com/hello       0.068s
```

测试成功，版本大迭代，更新一下依赖模块 go.mod （执行此命令前也可以先输出 `go list -m all` 方便与后面进行对比）

```bash
$ go mod tidy
```

再测试一下

```bash
$ go test 
PASS
ok      example.com/hello       0.196s
```

没问题，可以查看一下当前版本依赖情况

```bash
$ go list -m all
example.com/hello
golang.org/x/text v0.3.7
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1
```

## 总结

相关命令说明

`go mod init <NAME>` 创建新模块，初始化 go.mod 模块描述

`go list -m all` 打印模块依赖项

`go get <URL>` 修改依赖版本或者增加依赖

`go mod tidy` 移除无用的依赖

前面演示了一个简单的通过 go module 来更新项目版本的案例。

模拟了 patch 更新、mirror 更新、major 更新。

> 版本格式一般型如 v0.1.2，其中 0 代表主版本（major），1 代表次版本（mirror），2 代表补丁（patch）
> mirror 和 patch 一般都是会兼容低版本的。而 major 更新一般会有较大变动。