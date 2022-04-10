---
title: 编写定制的golangci-lint的linter
date: 2022-04-10 11:29:43
tags: 
- golang
- golangci-lint
- 原创
---

# 前言

在前文[go/analysis使用及linter入门](https://john520.github.io/2022/04/04/go-analysis%E4%BD%BF%E7%94%A8%E5%8F%8Alinter%E5%85%A5%E9%97%A8/) 我们已经介绍了如何使用`go/analysis`编写linter, 本文将继承上文的思路，编写自定义的golangci-lint 的linter.

我们经常会在golang 项目中使用goroutine, 然而在golang规范中，若某个goroutine抛出的panic 没有在defer 中 `recover`住，将导致整个程序crash. 本文将自定义linter用于检测项目中没有recover 的 goroutine.

# 开始编写linter

由于我们要编写的linter是用来校验goroutine是否使用在recover来捕获panic ,故这里我直接给项目取名`goroutine-with-recover`

我们先来看看怎样的goroutine是捕获了panic的goroutine,而哪些可能导致整个程序panic

```go
func main(){
 //若下面goroutine panic ,整个程序将crash
  go func() {
		fmt.Println("panic not recover")
	}()
//若下面goroutine panic ,不影响其他goroutine运行,整个程序不会crash  
  go func() {
		defer func() {
			recover()
		}()
	}()
//若下面goroutine panic ,不影响其他goroutine运行,整个程序不会crash  
	go func() {
		defer func() {
			r := recover()
			fmt.Println(r)
		}()
	}()
  //若下面goroutine panic ,不影响其他goroutine运行,整个程序不会crash  
	go func() {
		defer func() {
			if r := recover(); r != nil {

			}
		}()
	}()
  
}


```

除了第一种没有在defer func 中进行`recover` 可能导致goroutine painc, 剩下三个goroutine 都已经 `recover `了。你可能会好奇，为什么下面三种recover 效果都一样，为什么要区分对待呢。这是因为我们需要通过golang 的AST（注：这里可以查看上篇 [go/analysis使用及linter入门](https://john520.github.io/2022/04/04/go-analysis%E4%BD%BF%E7%94%A8%E5%8F%8Alinter%E5%85%A5%E9%97%A8/)） 语法树来进行识别，就需要整理出全部的可能情况,才能在linter 中识别这几种情况。

我们先来看下我们的项目结构：

```bash
├── README.md
├── analyzer
│   ├── analyzer.go
│   └── analyzer_test.go
├── cmd
│   └── goroutine-with-recover
│       └── main.go
├── go.mod
├── go.sum
├── goroutinewithrecover.so
└── testdata
    └── src
        └── p
            └── p.go

```

其中`analyzer.go`主要是使用`go/analysis`编写语法树检测逻辑，`testdata`主要是使用golangci-lint自带的单元测试框架进行测试

`main.go`是将我们编写的linter导出成plugin,以便在golangci-lint 中进行导入使用。具体的项目源码可以[查看](https://github.com/John520/goroutine-with-recover)

## 编写analyzer

```go
var Analyzer = &analysis.Analyzer{
   Name:     "goroutinewithrecover",
   Doc:      "Checks that goroutine has recover in defer function",
   Run:      run,
   Requires: []*analysis.Analyzer{inspect.Analyzer},
}

func run(pass *analysis.Pass) (interface{}, error) {
   inspector := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)
   nodeFilter := []ast.Node{
      (*ast.GoStmt)(nil),
   }
   inspector.Preorder(nodeFilter, func(node ast.Node) {
      gostat := node.(*ast.GoStmt)
      var r bool
      switch gostat.Call.Fun.(type) {
      case *ast.FuncLit: //形如 go func(){}
         funcLit := gostat.Call.Fun.(*ast.FuncLit)
         r = hasRecover(funcLit.Body)
      case *ast.Ident: //形如 go goFuncWithoutRecover()
         id := gostat.Call.Fun.(*ast.Ident)
         fd, ok := id.Obj.Decl.(*ast.FuncDecl) //fd 是 goFuncWithoutRecover 定义
         if !ok {
            return
         }
         r = hasRecover(fd.Body)
      default:

      }
      if !r {
         pass.Reportf(node.Pos(), "goroutine should have recover in defer func")
      }
   })

   return nil, nil
}
```

我们首先自定义了一个`Analyzer`, 它的`Run`方法主要定义了语法树的分析方法，我们使用 

```go
nodeFilter := []ast.Node{
   (*ast.GoStmt)(nil),
}
```

定义过滤条件（即只需要分析go func 节点），然后通过`inspector.Preorder`进行前序递归语法树，过滤出全部的go func 节点。然后通过一下函数进行分析检测

```go
func (node ast.Node){
  gostat := node.(*ast.GoStmt)
  
  ... //按照规则检测出没有recover 的goroutine
  
  //将不符合规范的go func 识别上报，通过这里上报，golangci-lint就会在执行 golangci-lint run 的时候输出
  if !r {
			pass.Reportf(node.Pos(), "goroutine should have recover in defer func") 
	}
}
```

 `gostat.Call.Fun`一般是`*ast.FuncLit`或`*ast.Ident`两种形式，其中`*ast.FuncLit`对应`go func(){}()` 这种匿名函数，而`*ast.Ident`对应具名函数。我们可以判断具名函数和匿名函数的函数体中是否存在defer 链，并判断defer 链是否包含recover  逻辑，具体如下：

```go
func hasRecover(bs *ast.BlockStmt) bool {
	for _, blockStmt := range bs.List {
		deferStmt, ok := blockStmt.(*ast.DeferStmt) //是否包含defer 语句
		if !ok {
			return false
		}
		switch deferStmt.Call.Fun.(type) {
		case *ast.SelectorExpr:
			//判断是否defer中包含  helper.Recover()
			selectorExpr := deferStmt.Call.Fun.(*ast.SelectorExpr)
			if "Recover" == selectorExpr.Sel.Name {
				return true
			}
		case *ast.FuncLit:
			//判断是否有 defer func(){ }()
			fl := deferStmt.Call.Fun.(*ast.FuncLit)
			for i := range fl.Body.List {

				stmt := fl.Body.List[i]
				switch stmt.(type) {
				case *ast.ExprStmt:
					exprStmt := stmt.(*ast.ExprStmt)
					if isRecoverExpr(exprStmt.X) { //recover()
						return true
					}
				case *ast.IfStmt:
					is := stmt.(*ast.IfStmt) // if r:=recover();r!=nil{}
					as, ok := is.Init.(*ast.AssignStmt)
					if !ok {
						continue
					}
					if isRecoverExpr(as.Rhs[0]) {
						return true
					}
				case *ast.AssignStmt:
					as := stmt.(*ast.AssignStmt) //r=:recover
					if isRecoverExpr(as.Rhs[0]) {
						return true
					}

				}
			}
		}
	}
	return false
}
func isRecoverExpr(expr ast.Expr) bool {
	ac, ok := expr.(*ast.CallExpr) // r:=recover()
	if !ok {
		return false
	}
	id, ok := ac.Fun.(*ast.Ident)
	if !ok {
		return false
	}
	if "recover" == id.Name {
		return true
	}
	return false
}

```

上述`hasRecover`方法中的`deferStmt.Call.Fun`有两种可能的类型，分别是 `*ast.SelectorExpr`,`*ast.FuncLit`,其中 `*ast.SelectorExpr`主要针对自己封装了`Recover`方法，如

```go
package util

func Recover() {
	if r := recover(); r != nil {
    fmt.Println(r)
	}
}
```

那么 `*ast.SelectorExpr`代表的是`util.Recover`这个节点

另一种`*ast.FuncLit`对应匿名的方法中包含`recover`，如

```go
go func() {
		defer func() {
			recover()
		}()
	}()

	go func() {
		defer func() {
			r := recover()
			fmt.Println(r)
		}()
	}()
	go func() {
		defer func() {
			if r := recover(); r != nil {

			}
		}()
	}()
```

至此，我们已经编写好了analyzer。接下来我们需要按照golangci-lint 规范，定义如何获取我们的anlyzer，并将其编译成plugin

```go
import (
	"github.com/John520/goroutine-with-recover/analyzer"
	a "golang.org/x/tools/go/analysis"
)

type analyzerPlugin struct{}

// This must be implemented
func (*analyzerPlugin) GetAnalyzers() []*a.Analyzer {
	return []*a.Analyzer{
		analyzer.Analyzer,
	}
}

// This must be defined and named 'AnalyzerPlugin'
var AnalyzerPlugin analyzerPlugin

```

通过下面命令行，我们可以将analyzer 编译成plugin 供golangci-lint 使用

```bash
go build  -buildmode=plugin  -o goroutinewithrecover.so cmd/goroutine-with-recover/main.go
```

这样我们就生成了`goroutinewithrecover.so`插件。

这里多说一句，这个plugin加载有点类似 Java 中通过类加载器，动态的将远程或者外部的Class 载入JVM 进行使用，不过这里golang 的plugin 还是感觉不是很好用，如果编译plugin 和编译golangci-lint 的编译环境和工具不一样，将导致加载plugin时报错

> ERRO Unable to load custom analyzer goroutinewithrecover:./goroutinewithrecover.so, plugin.Open("/Users/jiangjiang.xu/go/src/git.garena.com/shopee/game/pet-svr/goroutinewithrecover"): plugin was built with a different version of package internal/unsafeheader 
> ERRO Running error: unknown linters: 'goroutinewithrecover', run 'golangci-lint help linters' to see the list of supported linters 

解决的思路就是将golangci-lint `git clone` 下来,重新`go intall` ，保证编译的环境/工具一致



# golangci-lint 添加新的linter

正如golangci-lint官网[介绍](https://golangci-lint.run/contributing/new-linters/)中提到的两种方式：公共的linter和私有的linter.

- 公共的linter ：

  需要合入官方的golangci-lint代码库中，任何使用golangci-lint 都可以通过配置使用该linter 功能，如上文 [go/analysis使用及linter入门](https://john520.github.io/2022/04/04/go-analysis%E4%BD%BF%E7%94%A8%E5%8F%8Alinter%E5%85%A5%E9%97%A8/#%E9%9B%86%E6%88%90)中提到的，需要拉下golangci-lint的源码，在pkg/golinters 下添加自己的 linter,并添加相应大testdata 单元测试等，具体可以看官网的介绍。

- 私有的lingter :

   需要将自己的linter 编译成plugin，然后在`.golangci.yml`文件中添加 `linters-settings:custom`

本文的`gorouting-with-recover`linter 将通过私有的方式添加到golangci-lint中

如配置文中的`gorouting-with-recover`需要添加

```yaml
linters-settings:
  custom:
    example:
      path: ./goroutingwithrecover.so  //填写上述编译的plugin路径
      description: The description of the linter
      original-url: github.com/John520/gorouting-with-recover

```

默认情况下 自定义的linter 是打开的，但是若配置文件中将linters 默认关闭`linters:disable-all: true`，则需要在`linters:enable:`列表中添加该linter,如

```yaml

linters:
  disable-all: true
  enable:
    - goroutingwithrecover
    ...
```

修改好`.golangci.yml`,就可以通过`golangci-lint run`检查代码中的goroutine 了 ：）

 # 参考文献

1. https://golangci-lint.run/contributing/new-linters/
2. https://github.com/golangci/example-plugin-linter
3. https://tonybai.com/2021/07/19/understand-go-plugin/

