---
title: go/analysis使用及linter入门
date: 2022-04-04 21:19:07
tags: 
 - 原创
 - golangci-lint

categories: golang

---
本文至译文，原文件下面的参考文章，主要介绍从0-1 开发自定义的linter，并集成至go vet 和golangci-lint，废话不多说，请看下文

# 前言

我们今天将在go 语言中使用`go/analysis`来编写一个有用的`linter`，然后可以集成至`go vet` 和`golangci-lint`

# 什么是 lint

让我们找到所有没有以`f `结尾的`printf`风格的方法，在go语言中，惯例使用`f`结尾的方法名，如`fmt.Sprintf`, `fmt.Errorf`, `fmt.Printf`, `fmt.Fprintf`, `fmt.Scanf`, `fmt.Sscanf`, `log.Printf`, `log.Fatalf`, `log.Panicf`.

# 编写一个简单的linter

## 有问题的样例代码

让我们用一下的示例来解释和测试，下面的`myLog`应该使用`myLogf`方法名：

```go
//example.go
package main

import "log"

func myLog(format string, args ...interface{}) {
	const prefix = "[my] "
	log.Printf(prefix + format, args...)
}
```

## 什么是AST

AST (abstract syntax tree 抽象语法树)，通常用在代码的检测分析中

我们可以在这个网站[GoAST Viewer](http://goast.yuroyoro.net/)可视化我们程序的抽象语法树

我们可以将上面的`example.go`中的代码放到该网址中查看语法树，我们看下`myLog`方法部分，简化如下

```go
&ast.FuncDecl{
    Name: &ast.Ident{Name: "myLog"},
    Type: &ast.FuncType{
        Params: &ast.FieldList{
            List: []*ast.Field{
                &ast.Field{
                    Names: []*ast.Ident{&ast.Ident{Name: "format"}},
                    Type:  &ast.Ident{Name: "string"},
                },
                &ast.Field{
                    Names: []*ast.Ident{&ast.Ident{Name: "args"}},
                    Type: &ast.Ellipsis{
                        Elt: &ast.InterfaceType{
                            Methods: &ast.FieldList{
                                List: nil, // no methods in the interface{}
                            }, // end of Methods
                        }, // end of &ast.InterfaceType
                    }, // end of  &ast.Ellipsis
                }, // end of &ast.Field
            }, // end of []*ast.Field
        }, // end of &ast.FieldList
    }, // end of &ast.FuncType
} // end of &ast.FuncDecl

```

## 开始编写linter

```go
// main.go
package main

import (
	"bytes"
	"fmt"
	"go/ast"
	"go/parser"
	"go/printer"
	"go/token"
	"log"
	"os"
)

func main() {
	v := visitor{fset: token.NewFileSet()}
	for _, filePath := range os.Args[1:] {
		if filePath == "--" { // to be able to run this like "go run main.go -- input.go"
			continue
		}

		f, err := parser.ParseFile(v.fset, filePath, nil, 0)
		if err != nil {
			log.Fatalf("Failed to parse file %s: %s", filePath, err)
		}

		ast.Walk(&v, f)
	}
}

type visitor struct {
	fset *token.FileSet
}

func (v *visitor) Visit(node ast.Node) ast.Visitor {
	if node == nil {
		return nil
	}

	var buf bytes.Buffer
	printer.Fprint(&buf, v.fset, node)
	fmt.Printf("%s | %#v\n", buf.String(), node)

 	return v
}
```

`parser.ParseFile`用于解析和构建AST 语法树.`ast.Walk`使用`visitor`来遍历访问AST `node`节点：它会一直遍历`node`直至`Visit`方法返回`nil`或所有的`node `都遍历完毕。

我们可以使用它来解析 `myLog`方法

`go run ./main.go -- ./example.go`

查看输出（这里省略，大家可以自己操作下），可以看到打印出全部的`node`,让我们来在`visitor.Visit`中编写lint规则

 ```go
func (v *visitor) Visit(node ast.Node) ast.Visitor {
	funcDecl, ok := node.(*ast.FuncDecl)
	if !ok {
		return v
	}

	params := funcDecl.Type.Params.List
	if len(params) != 2 { // [0] must be format (string), [1] must be args (...interface{})
		return v
	}

	firstParamType, ok := params[0].Type.(*ast.Ident)
	if !ok { // first param type isn't identificator so it can't be of type "string"
		return v
	}

	if firstParamType.Name != "string" { // first param (format) type is not string
		return v
	}

	secondParamType, ok := params[1].Type.(*ast.Ellipsis)
	if !ok { // args are not ellipsis (...args)
		return v
	}

	elementType, ok := secondParamType.Elt.(*ast.InterfaceType)
	if !ok { // args are not of interface type, but we need interface{}
		return v
	}

	if elementType.Methods != nil && len(elementType.Methods.List) != 0 {
		return v // has >= 1 method in interface, but we need an empty interface "interface{}"
	}

	if strings.HasSuffix(funcDecl.Name.Name, "f") {
		return v
	}

	fmt.Printf("%s: printf-like formatting function '%s' should be named '%sf'\n",
		v.fset.Position(node.Pos()), funcDecl.Name.Name, funcDecl.Name.Name)
	return v
}
 ```

让我们来试一下看它能否正常工作：

````go
go run ./main.go -- ./example.go
//输出
./example.go:5:1: printf-like formatting function 'myLog' should be named 'myLogf'

````

上面已经可以通过AST 分析出那些不符合`printf`风格的方法命名，接下来我看看如何集成到`go/analysis`中

## 使用`go/analysis`

`go/analysis`是用来模块化分析的API,也是对所有linter的通用接口，这个API 简化了开发一个新的linter,它强制在静态代码分析中使用最佳实践，如，一次使用一个包

在这个API 中的主要类型是`analysis.Analyzer`. 一个 `Analyzer` 静态描述了一个分析方法：它的名字，文档，flags,以及和其他analyzers的关系，还有它的逻辑

定义一个分析器，需要首先申明`*analysis.Analyzer`

```go
type Analyzer struct {
	Name			string
	Doc			string
	Flags			flag.FlagSet
	Run			func(*Pass) (interface{}, error)
	RunDespiteErrors	bool
	ResultType		reflect.Type
	Requires		[]*Analyzer
	FactTypes		[]Fact
}
```

我们最关心的是`Run`方法:它是linter 的逻辑区域，他的传参是`*Pass`

```go
type Pass struct {
	Fset   		*token.FileSet
	Files		[]*ast.File
	OtherFiles	[]string
	Pkg		*types.Package
	TypesInfo	*types.Info
	ResultOf	map[*Analyzer]interface{}
	Report		func(Diagnostic)
	...
}

```

`*Pass`描述了单个工作单元：特定的`Analyzer`(分析器)分析特定报下的go代码，`*Pass`提供给`Analyzer`的`Run`方法中被分析的代码信息，它为Run方法提供操作，以将诊断信息和其他的信息报告给驱动程序。

`Fset`, `Files`, `Pkg`, 和 `TypesInfo`提供了源文件位置信息，语法树，类型信息等

## 为什么我们需要`go/analysis`?

这里主要有两天原因：统一的接口和代码复用

`go/analysis`为linter 提供了统一的接口，它简化了与IDE,metalinters，代码Review等工具的集成。如，任何`go/analysis`linter都可以高效的被`go vet`执行。稍后我们会介绍如何集成它

在我们简单的linter里面。我们需要用户提供一份需要分析的文件列表。但是还有一种更加方便的方法，就是传递`./...`参数，这个参数会递归的遍历全部的包路径和全部文件。

## 开始使用`go/analysis`

让我们移植我们上述的简单linter,使用`go/analysis`来开发，我们先建立一个工程 `go-prinf-func-name`

项目结构如下，

```bash
.
├── README.md
├── cmd
│   └── go-printf-func-name
│       └── main.go
├── go.mod
├── go.sum
└── pkg
    └── analyzer
        └── analyzer.go
```

代码如下

```go
//cmd/go-printf-func-name/main.go
package main

import (
	"github.com/jirfag/go-printf-func-name/pkg/analyzer"
	"golang.org/x/tools/go/analysis/singlechecker"
)

func main() {
	singlechecker.Main(analyzer.Analyzer)
}
```

```go
//pkg/analyzer/analyzer.go
package analyzer

import (
	"go/ast"
	"strings"

	"golang.org/x/tools/go/analysis"
)

var Analyzer = &analysis.Analyzer{
	Name: "goprintffuncname",
	Doc:  "Checks that printf-like functions are named with `f` at the end.",
	Run:  run,
}

func run(pass *analysis.Pass) (interface{}, error) {
	inspect := func(node ast.Node) bool {
		funcDecl, ok := node.(*ast.FuncDecl)
		if !ok {
			return true
		}

		params := funcDecl.Type.Params.List
		if len(params) != 2 { // [0] must be format (string), [1] must be args (...interface{})
			return true
		}

		firstParamType, ok := params[0].Type.(*ast.Ident)
		if !ok { // first param type isn't identificator so it can't be of type "string"
			return true
		}

		if firstParamType.Name != "string" { // first param (format) type is not string
			return true
		}

		secondParamType, ok := params[1].Type.(*ast.Ellipsis)
		if !ok { // args are not ellipsis (...args)
			return true
		}

		elementType, ok := secondParamType.Elt.(*ast.InterfaceType)
		if !ok { // args are not of interface type, but we need interface{}
			return true
		}

		if elementType.Methods != nil && len(elementType.Methods.List) != 0 {
			return true // has >= 1 method in interface, but we need an empty interface "interface{}"
		}

		if strings.HasSuffix(funcDecl.Name.Name, "f") {
			return true
		}

		pass.Reportf(node.Pos(), "printf-like formatting function '%s' should be named '%sf'",
			funcDecl.Name.Name, funcDecl.Name.Name)
		return true
	}

	for _, f := range pass.Files {
		ast.Inspect(f, inspect)
	}
	return nil, nil
}
```

注意，我们的主要包括：

1. 我们使用`ast.Inspect`代替了`ast.Walk`:它减低了代码。我们只需要移除visitor以及返回`true`来代替visitor
2. 我们遍历`pass.File`来获得当前包的语法树，不需要去手动`parse.ParseFile`来解析
3. 当我们发现问题，通过`pass.Reportf`代替`fmt.Printf`.这个方法是`go/analysis`提供的抽象上报逻辑

## 运行它

```bash
go run ./cmd/go-printf-func-name/main.go -- ./example.go
/Users/denis/go-printf-func-name/example.go:5:1: printf-like formatting function 'myLog' should be named 'myLogf'

exit status 3
```

它正常工作，并上报了又问题的代码

##  `ast.Inspect` vs `Inspector`

你是否注意到`ast.Insepct`和`ast.Walk`无效，我们访问了每个AST node :二进制表达式，变量，常量，等，但是我们仅需要的是方法的声明，我们是否可以只遍历方法来提高性能呢

是的，`go/analysis`里面包含了`insepct.Analyzer`,它可以通过AST node的类型来进行过滤，我们先来看下`insepct.Analyzer`定义

```go
var Analyzer = &analysis.Analyzer{
	Name:     "goprintffuncname",
	Doc:      "Checks that printf-like functions are named with `f` at the end.",
	Run:      run,
	Requires: []*analysis.Analyzer{inspect.Analyzer},}
```

我们可以修改`fun`方法

```go
func run(pass *analysis.Pass) (interface{}, error) {
  // pass.ResultOf[inspect.Analyzer] will be set if we've added inspect.Analyzer to Requires.
  inspector := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)

	nodeFilter := []ast.Node{ // filter needed nodes: visit only them
		(*ast.FuncDecl)(nil),
	}

	inspector.Preorder(nodeFilter, func(node ast.Node) {
		funcDecl := node.(*ast.FuncDecl)

		params := funcDecl.Type.Params.List
		if len(params) != 2 { // [0] must be format (string), [1] must be args (...interface{})
			return
		}

		firstParamType, ok := params[0].Type.(*ast.Ident)
		if !ok { // first param type isn't identificator so it can't be of type "string"
			return
		}

		if firstParamType.Name != "string" { // first param (format) type is not string
			return
		}

		secondParamType, ok := params[1].Type.(*ast.Ellipsis)
		if !ok { // args are not ellipsis (...args)
			return
		}

		elementType, ok := secondParamType.Elt.(*ast.InterfaceType)
		if !ok { // args are not of interface type, but we need interface{}
			return
		}

		if elementType.Methods != nil && len(elementType.Methods.List) != 0 {
			return // has >= 1 method in interface, but we need an empty interface "interface{}"
		}

		if strings.HasSuffix(funcDecl.Name.Name, "f") {
			return
		}

		pass.Reportf(node.Pos(), "printf-like formatting function '%s' should be named '%sf'",
			funcDecl.Name.Name, funcDecl.Name.Name)
	})

	return nil, nil
}
```

我们修改了AST 的遍历方式，使用`inspector.Preorder`进行遍历，这个方法不需要返回值，根据文档介绍，这种方式比ast walk 快了近2.5倍

```go
// During construction, the inspector does a complete traversal and
// builds a list of push/pop events and their node type. Subsequent
// method calls that request a traversal scan this list, rather than walk
// the AST, and perform type filtering using efficient bit sets.
//
// Experiments suggest the inspector's traversals are about 2.5x faster
// than ast.Inspect, but it may take around 5 traversals for this
// benefit to amortize the inspector's construction cost.
// If efficiency is the primary concern, do not use Inspector for
// one-off traversals.
```

我们希望我们的linter 被运行在linter runner 上的，如`golangci-lint`,这种runner包含了数十个`go/analysis`的analyzers

## 自动测试

我们需要测试我们的linter.`go/analysis`提供了测试框架，我们先创建 `analyzer_test.go` 和 `testdata/src/p/p.go`:目录结构如下

```bash
.
├── README.md
├── cmd
│   └── go-printf-func-name
│       └── main.go
├── example.go
├── go.mod
├── go.sum
├── pkg
│   └── analyzer
│       ├── analyzer.go
│       └── analyzer_test.go
└── testdata
    └── src
        └── p
            └── p.go
```

在`analyzer_test.go`编写测试代码

```go
func TestAll(t *testing.T) {
	wd, err := os.Getwd()
	if err != nil {
		t.Fatalf("Failed to get wd: %s", err)
	}

	testdata := filepath.Join(filepath.Dir(filepath.Dir(wd)), "testdata")
	analysistest.Run(t, testdata, Analyzer, "p")
}
```

在`testdata/src/p/p.go`中编写测试代码

如果linter 会上报错误信息，我们需要在方法后加上注释`// want "printf-like formatting function"`,`go/analysis`会自动找到这些注释并匹配testcase中上报的issue

```go
package p

func notPrintfFuncAtAll() {}

func funcWithEllipsis(args ...interface{}) {}

func printfLikeButWithStrings(format string, args ...string) {}

func printfLikeButWithBadFormat(format int, args ...interface{}) {}

func prinfLikeFunc(format string, args ...interface{}) {} // want "printf-like formatting function"

func prinfLikeFuncWithReturnValue(format string, args ...interface{}) string { // want "printf-like formatting function"
	return ""
}
```

运行测试案例

```bash
go test -v ./...
?       github.com/jirfag/go-printf-func-name   [no test files]
?       github.com/jirfag/go-printf-func-name/cmd/go-printf-func-name   [no test files]
=== RUN   TestAll
--- PASS: TestAll (0.29s)
PASS
ok      github.com/jirfag/go-printf-func-name/pkg/analyzer      0.387s
```

还能通过代码覆盖率查看我们的测试覆盖率情况

```go
go test ./... -coverprofile=coverage.out 
go tool cover -html=coverage.out 
```

译者注：这里覆盖了没有达到100%，还有需要优化的，同时这里可能存在误报的情况，如：

```go
func (tx *Tx) Query(query string, args ...interface{}) (*Rows, error) {
        return tx.QueryContext(context.Background(), query, args...)
}
```

因此需要还需要在方法中底层跟踪判断是否底层调用了类似`fmt.Sprinf`的方法等

# 集成

## 集成至go vet

```bash
go install ./cmd/...
go vet -vettool=$(which go-printf-func-name) ./...
```

## 集成至 golangci-lint

`golangci-lint`是一个linter runner,我们可以拉下官方源码，然后在pkg/golinters 下添加`goprinffuncname.go`

```go
func NewGoPrintfFuncName() *goanalysis.Linter {
	return goanalysis.NewLinter(
		"goprintffuncname",
		"Checks that printf-like functions are named with `f` at the end",
		[]*analysis.Analyzer{analyzer.Analyzer},
		nil,
	).WithLoadMode(goanalysis.LoadModeSyntax)
}
```

然后提交MR 到官网,本文MR可以[查看](https://github.com/golangci/golangci-lint/pull/850)



# 参考文献



* https://disaev.me/p/writing-useful-go-analysis-linter/
* https://golangci-lint.run/contributing/new-linters/



