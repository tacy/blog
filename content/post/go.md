---
title: "golang notes"
date: 2016-03-01
lastmod: 2016-03-01
draft: false
tags: ["tech", "golang", "notes"]
categories: ["tech"]
description: "go开发笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# 配置 #
pacman -S go

export GOPATH=~/workspace/go/go-tools/

go get github.com/mailgun/godebug
go get github.com/nsf/gocode
go get github.com/rogpeppe/godef
go get github.com/tools/godep
go get golang.org/x/tools/cmd/goimports
go get golang.org/x/tools/cmd/oracle

export PATH=$PATH:~/workspace/go/go-tools/bin

不建议在bash_profile里面设置GOPATH，Go的依赖问题容易导致冲突，建议为大的项目设置独立的GOPATH，比如Kubernetes这类的

## PROXY
export http_proxy https_proxy
git config --global http.proxy https.proxy

git config --config unset http.proxy

# 依赖管理 #
## Godep ##

使用流程：先编写好代码，编译通过之后，运行`godep save`, godep会帮你把项目的依赖都存入到Godep/_workspace，同时会在Godep目录下生成Godeps.json文件，里面是项目的所有依赖类库。

后续要编译项目的时候通过命令：`godep go build`, 如果你不想要godep在前面，你也可以编辑你的GOPATH：`export GOPATH=``godep path``:$GOPATH`, 这样的话Go会先找到Godep目录下的类库，直接用`go build`就不存在问题了。

如果你希望自己的项目完全自依赖，可以通过`godep save -r`实现，这样godep会改写你的import, 例如：项目C内的`import D` 变成`import C/Godeps/_workspace/src/D`, 也就是说直接修改代码依赖了。

恢复项目依赖的命令: `godep restore`

更新依赖操作：
```
go get -u foo/bar
godep update foo/bar
```

## govendor##
govendor init
govendor add +e
govendor fetch +e
govendor fetch +m //missing package
加入包以及依赖包:
govendor get -v  k8s.io/client-go/...@=v1.5.1

## dep
新工具, 暂时不稳定

## dlv
调试工具, 编译时加上`-gcflags='-N -l'`.
dlv exec运行程序的时候, 如果程序有参数, 用`--`提供, 例如`dlv exec kubectl -- create ...`

# Build
静态连接编译: `CGO_ENABLED=0 go build -gcflags '-N -l'`
# Feature #

Cgo: go调用c代码

# Test
1. 如果你需要测试单个文件, 而且test文件的包名和被测代码的包名一样, 需要带上被测试文件:

```
go test openfalcon_sink_test.go openfalcon_sink.go
```

2. 测试覆盖率

```
go test -coverprofile=coverage.out
go tool conver -html=coverage.out -o coverage.html

or

go test -cover test.go
```

# Study

## declaration

1. 返回pointer

```
var a *T = new(T)
a := new(T)
a := &T{}
```

2. 返回value

```
a := T{}
var a T

```

3. channel & map & slice
只能用make申明, 返回value, 但是注意channel和map都是引用类型, 无需用pointer传递

```
a := make(chan string, 10)
var a chan string = make(chan string, 10)
a := make([]string, 10, 20) //分配一个array, 返回一个slice引用到这个array
var a []string = new([20]int)[:10]
```
slice需要注意的是, 如果你的slice长度固定(底下的array无需重新分配), slice也无需通过pointer传递, 否则需要


## method
### pointer receiver or value receiver
Slices are one place where it's not always obvious at first. The Slice header is small, so copying it is cheap, and the underlying array is referenced via a pointer, so you can manipulate the contents of a slice with a value receiver. You can see this in the sort package, where the methods for the sortable types are defined without pointers.

The only time you need to use a pointer with a slice, is if you're going to manipulate the slice header, which means changing the length or capacity. For an Append method, you would want:

```
func (p *ByteSlice) Append(data []byte) {
    *p = append(*p, data...)
}
```

[测试代码, 看pointer receiver和value receiver的区别](https://play.golang.org/p/43OM90PVTN)

value receiver和python不同, 举例如下:

```
type Val struct {
    val int
}

func (v Val) Set(a int) {
    v.val = v.val + a
}

func (v Val) Get() int {
    return v.val
}

func main() {
    a := new(Val)
    a.Set(1)
    fmt.Println(a.Get())
}
```
上面这段代码输出: '0', 也就是说, 如果你需要修改receiver, 必须用point receiver. 这里理解起来比较麻烦, 和很多编程语言有点区别, 一定要结合 [Method expressions](https://golang.org/ref/spec#Method_expressions)来理解:
上面这段代码中, Val.Set这个method expressions其实代表的是一个function: func(v Val, a int), 这里的v就是receiver, 作为func的第一个参数, 这样就容易理解了, 如果是value receiver, 就是copy receiver传入func, 所以Set方法对val的修改并不会影响原始的receiver, 如果你想修改receiver的变量, 必须用pointer receiver
