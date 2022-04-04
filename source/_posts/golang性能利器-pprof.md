---
title: golang性能利器-pprof
date: 2022-04-04 22:23:54
tags: 
 - golang
 - 原创

categories: golang

---
# 前言

作为一名gopher,不知道你是否遇到过以下问题？

1. CPU突然飙高(甚至死循环,CPU跑满)？
2. 某个功能接口QPS在压测中一直压不上去？
3. 应用出现goroutine泄露？
4. 内存居高不下？
5. 在某处加锁的逻辑中，迟迟得不到锁？

你是否能得心应手的找到问题的症结所在，你又是使用什么方法或工具解决的呢？今天我将介绍下我经常使用的工具pprof.它是golang自带的性能分析大杀器，基本上能快速解决上述问题。

# 如何使用pprof 

使用pprof有三种姿势，一种是使用`runtime/pprof/pprof.go`另一种是使用`net/http/pprof/pprof.go`（底层也是使用`runtime/pprof`）,还有一种是在单元测试中生成profile 数据。详细的方法在对应的`pprof.go`文件开头有说明。

## 1. runtime/pprof 方式pprof

```go
package main

import (
	"flag"
	"log"
	"os"
	"runtime"
	"runtime/pprof"
)

var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to `file`")
var memprofile = flag.String("memprofile", "", "write memory profile to `file`")

func main() {
	flag.Parse()
	if *cpuprofile != "" {
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Fatal("could not create CPU profile: ", err)
		}
		defer f.Close() // error handling omitted for example
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
		defer f.Close() // error handling omitted for example
		runtime.GC()    // get up-to-date statistics
		if err := pprof.WriteHeapProfile(f); err != nil {
			log.Fatal("could not write memory profile: ", err)
		}
	}
}

```

这种方式需要你手动启动需要pprof的类型，比如，开启CPU profile 还是heap profile 等等，pprof 会在应用启动到结束的整个生命周期生成profile 文件。其实生成profile 数据是会损耗性能的，生产环境不建议一直开启，可以在需要分析的时候临时采集那个时刻的数据，如通过监听系统信号的方式开启/关闭pprof,示例代码如下：

```go
package main

import (
	"flag"
	"log"
	"os"
	"runtime"
	"runtime/pprof"
)
var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to `file`")
var memprofile = flag.String("memprofile", "", "write memory profile to `file`")

func signalStopPprof(f *os.File) {
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGUSR2)
	for _ = range c {
		pprof.StopCPUProfile()
		log.Println("stop  profile")
		_ = f.Close()
	}
}
func signalStartPprof(f *os.File) {
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGUSR1)
	for _ = range c {
		_ = f.Truncate(0)
		if err := pprof.StartCPUProfile(f); err != nil {
			log.Println("could not start CPU profile: ", err)
		}
	}
}
func main() {
	flag.Parse()
	if *cpuprofile != "" {
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Fatal("could not create CPU profile: ", err)
		}
		go func() {
			signalStartPprof(f)
		}()
		go func() {
			signalStopPprof(f)
		}()
	}
	// ... rest of the program ...
}
```

不过本人还是更加喜欢第二种方式`net/http/pprof`

## 2. net/http/pprof 方式pprof

```go
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof"
)

func main() {
	go func() {
		log.Println(http.ListenAndServe("localhost:6060", nil))
	}()
  // ... rest of the program ...
}

```

是不是很简单，这种方式只需要开启http服务，并且`import _ "net/http/pprof"`,他会自动把相应的http的handleFunc 注册上去，若你使用的不是`DefaultServeMux`,需要自己手动注册下。

```go
func init() {
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
}
```

## 3. 单元测试中进行profile

```go
go test -cpuprofile cpu.prof -memprofile mem.prof -bench .
```

# 分析pprof

在上文[如何使用pprof](#如何使用pprof)中介绍的三种开启pprof的方式，他们都会生成profile二进制文件，有三种方式可以分析这个二进制文件

1. 姿势一：通过交互命令行,方法：`go tool pprof {profile文件}`

   若是通过http方式开启pprof，可以

   `go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30`

   若是通过方式1和3生成文件,可以

   `go tool pprof cpu.prof`

   命令交互行如下![pprof.jpg](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7k74ec7gj20ve064aap.jpg)

2. 姿势二：通过web方式查看，方法 `go tool pprof -http=:6060 {profile文件}`

   然后就可以在浏览器中访问 http://localhost:6060/ 

   当然也有一种方式在姿势一中介绍的命令行交互中输入`web`会生成 .svg文件，不过需要安装graphviz来打开这个文件，不推荐，还是建议使用命令行或本地启动web服务来查看profile 数据.

   

# 实战演练

## 1.定位CPU问题

自己写了一个模拟程序，模拟CPU问题，

按照上述姿势一打开命令行交互，执行`top10`,输出采样期间CPU使用最高的10个方法

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7mugw0vjj20x40hotdq.jpg)

这里简单介绍下flat flat% sum% cum cum%这五个参数的含义,详细可以查看[Pprof and golang - how to interpret a results?](https://stackoverflow.com/questions/32571396/pprof-and-golang-how-to-interpret-a-results)

flat：指的是该方法所占用的CPU时间（不包含这个方法中调用其他方法所占用的时间）

flat%: 指的是该方法flat时间占全部采样时间的比例

cum：指的是该方法以及方法中调用其他方法所占用的CPU时间总和，这里注意区别于flat

cum%:指的是该方法cum时间占全部采样时间的比例

sum%: 指的是执行到当前方法累积占用的CPU时间总和，也即是前面flat%总和

上图可以看出worker()占用CPU时间较久，我们可以`list main.worker`查看具体代码
![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7nf6zxjaj21ja0qqdln.jpg)

当然也可以通过上述姿势二，启动web服务查看火焰图`go tool pprof -http=:6061 cpu.profile `

打开http://localhost:6061/ui/ 火焰图如下，其中颜色越红代表占用的CPU时间越多

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7nl73p3xj20yg18i11r.jpg)

找到了耗时比较久的地方，就能看看正常的业务代码还是可以优化的逻辑，就可以优化后在pprof

## 2. 定位内存问题

排查内存再用过高问题

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7te6d0fij21je0f6qbr.jpg)

其中`inuse_space`表示查看常驻内存的使用情况

list 查看相应的函数

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7t6d59b3j21bg0iytcm.jpg)

原来是一致在append内存，并持有到1GB不释放

当然还可以使用`alloc_objects`：分析应用程序的内存临时分配情况

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7tlk3i02j21iq0kcn7w.jpg)

可以看到应用稳定后，除了上面初始分配1GB外，应用临时内存分配主要在worker中

## 3.定位goroutine问题

例如goroutine泄露等问题，方法如下

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7u104p52j21eg0gq7d0.jpg)

可以打出各个goroutine的调用栈

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7u4p0of3j20wa104n3o.jpg)

可以看到有900+ 的goroutine 阻塞在`runtime/gopark`并且是由`main.goroutine.fun1`方法引起的 list 查看方法内容`list main.goroutine.fun1`

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy7u7qv0hfj21cm0i4tc1.jpg)

可以看出主要是阻塞在channel c 上.当然也可以通过`traces runtime.gopark `查看那些方法最终阻塞在`gopark`上，另外也可以在web页面http://localhost:6060/debug/pprof/goroutine?debug=1直接查看。

## 4.定位锁问题

锁的问题可能导致程序运行缓慢，pprof mutex 相关的需要设置采样率

`runtime.SetMutexProfileFraction(1)`,若采样率<0 将不进行采样

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy8ynn7v28j21ce0e6wmj.jpg)

可以看出，主要在`main.mutex.func1`上，可以查看调用链

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy8yomftwzj20x00jg0w1.jpg)

`list main.mutex.func1`

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy8ypmn2axj21dq0fmtbh.jpg)

可以看出主要阻塞在锁`m`上

## 5. 定位goroutine阻塞等待同步问题

区别于4 ，这里主要是记录goroutine阻塞等待同步的位置,而4中主要是互斥锁分析，报告互斥锁的竞争情况。同样需要设置采样率`runtime.SetBlockProfileRate(1)`

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy8z65fl7ej21cw0ka12i.jpg)

主要主色在chanrecv1上，查看源码

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gy8z7cdre6j21560scjwy.jpg)

可以发现主要阻塞在channel c 上

# 写在最后

上面已经一步一步演示了一遍常见应用性能问题，以及如何分析定位，后面将写下这些分析数据背后是如何采集，原理是什么。

最后附上本次测试源码https://github.com/John520/pprof-project.git

# 参考文献

1. [pprof 官方README](https://github.com/google/pprof/blob/master/doc/README.md)
2. [Profiling Go Programs](https://go.dev/blog/pprof)
3. [Golang 大杀器之性能剖析 PProf](https://segmentfault.com/a/1190000016412013)
4. [golang pprof 实战](https://blog.wolfogre.com/posts/go-ppof-practice/)
5. https://golang2.eddycjy.com/posts/ch6/01-pprof-1/

