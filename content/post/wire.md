+++
title = 'Wire 使用&源码解析'
date = 2024-07-20T19:57:37+08:00
draft = false
summary = 'wire 使用&核心原理'
tags =["wire","源码"]
+++

# 本文概要
本文介绍了 Go 语言依赖注入库 Wire 的使用方法，同时对 Wire 核心功能的源码进行了解析；
本文试图实现以下目标：
1. 介绍清楚 Wire 的使用方法；
2. 讲明白 Wire 的核心原理；

# Wire 简介及用法示例
## Wire 简介
Wire 是一个使用依赖注入自动连接组件的代码生成工具；在 Wire 中，组件之间的依赖关系通过函数的参数进行描述，鼓励显式地初始化而非全局变量；

由于 Wire 在没有运行时状态或反射的情况下运行，因此编写用于 Wire 的代码（即描述依赖关系的“构造函数”），对于手写的初始化也是有用的。

Wire 的官方介绍[博客文章](https://go.dev/blog/wire)

## 安装
```go
go install github.com/google/wire/cmd/wire@latest
```

确保 `$GOPATH/bin` 被添加到 `$PATH` 环境变量中；执行 `wire -h` 可以查看 Wire 的帮助信息

## 使用
在 Wire 中有两个核心概念：`provider` 和 `injector`；
`provider` 是指创建实体的函数，类似于构造函数，它的作用是描述实体或者类型之间的依赖关系；
`injector` 则负责连接 Wire 与项目代码，它收集应用所需要的 `provider`，然后通过生成代码的方式完成相关实体的创建和所需依赖的注入，最后向 `injector` 的调用方返回一个创建完毕的实体；

使用 Wire 工具，包含 4 个步骤：
1. 准备 `provider`：普通的创建函数，如 `func NewSky(cloud Cloud, sun Sun) *Sky` 即可；该 `provider` 表示 `Sky` 类型有两个依赖项：`Cloud` 和 `Sun`；至于这两个依赖项怎么来，交于 Wire 处理即可；
2. 向 Wire 提供 `provider`：`skySet := wire.NewSet(NewSky,NewCloud,NewSun)`；这里将 `NewSky、NewCloud、NewSun` 作为 `provider` 通过 `wire.NewSet` 方法放到了 Wire 提供的结构体 `ProviderSet` 中；`NewSet` 函数起到了收集 `provider` 的作用，而 `ProviderSet` 最终将投入 `injector` 中；
3. 编写 `injector`：一般是单独放在 wire.go 文件中：
``` go
package somewhere

//go:build wireinject
// +build wireinject
func doInject(config Config) *Sky{
	wire.Build(
		skySet,
	)
	return nil
}
```
这里 `doInject` 作为 `injector`，它的入参也被视为 `provider`，同时函数体中只能调用 `wire.Build()` 方法，否则 Wire 工具会报错；至于最后返回什么，其实并不重要，重要的是函数声明中已经表述清楚最后要返回的`*Sky`；
值得注意的是开头的两行条件编译指令，它们确保 wire.go 文件仅能够被 Wire 工具所编译加载（项目代码编译时一般不会指定 `wireinject` 的条件）

4. 生成代码：执行 `wire gen` 命令，Wire 将根据 `injector` 函数，在与 wire.go 的相同目录下生成  wire_gen.go 文件：
```go
//go:build !wireinject
// +build !wireinject
package somewhere

func doInject(config Config) *SomethingYouGet{
	cloud := NewCloud(config)
	sun := NewSun()
	sky := NewSky(cloud, sun)
	return sky
}
```
最后生成的文件大概长上面这个样子；由于该文件增加的条件编译指令，使得其不会被 Wire 工具所编译加载；即项目代码真正调用的是 Wire 工具生成的函数；

上述代码示例，仅供使用流程的理解；更多 Wire 功能示例请移步：hello-wire 仓库；

# 关键类型与函数
## 类型
**ProviderSet**

`type ProviderSet struct{}` ProvierSet 是一个标记类型，表示一组 `provider`

**Binding**
`type Binding struct{}` Binding 将一个 interface 映射到一个具体的类型；达到“接口”和“实现”的绑定；

**ProvidedValue**

`ProvidedValue` 所关联表达式将被复制到生成的 injector 函数中；

**StructProvider**

`StructProvider` 表示一个命名的结构体；

**StructFields**

`StructFields` 表示一系列来自结构体的字段；

## 注册方法
**NewSet**

`func NewSet(...interface{}) ProviderSet` NewSet 会创建一个将入参标记为`provider` 的 `ProviderSet`；入参的类型可以为：
1. 函数值
	将该函数的第一个返回值作为一个提供者注入 Wire；同时该函数的入参将来自其他注册到 Wire 的提供者，要求这些入参中不能出现有相同类型的参数；因为 Wire 是根据入参的类型提供具体的实参；
	函数可以返回一个可选的 `error` 以及一个 `cleanup` 函数；`cleanup` 函数类型必须是`type func()`。该函数将保证在函数入参对应的 `cleanup` 方法之前调用，即有依赖关系的提供者所对应的 `cleanup` 方法将以栈的顺序被调用；
	如果任何 `provider` 返回了错误，那么将调用相关的 `cleanup` 方法并且返回该错误；
2. ProviderSet
	将 ProviderSet 标记的值作为提供者注入 Wire；作用等同于将 `ProviderSet` 所标记的 `provider` 直接提供给 `NewSet` 函数
3. Wire 提供的一系列函数调用
	作用是将函数的返回值作为提供者注入，这些函数包括：	
	1. Struct：将返回值作为提供者注入，可以通过字段名指定需要注入的依赖；
	2. Bind：通过 Bind 方法将接口与具体的实现进行绑定，具体实现；
	3. InterfaceValue：创建一个类型为具体实现的变量，然后对象依赖图中所有需要接口类型的地方，均被注入该变量。需要注意的是，wire 并不注入该变量本身的依赖；该方法适合将依赖完毕的对象（如第三方库）绑定到具体的接口中；
	4. FieldsOf：将制定结构体的各个字段作为各自类型的提供者；
出于兼容性，如果传递 `S` 类型的结构体到 `NewSet`，会同时注入 `S` 和 `*S`两个类型的提供者；这个特性在未来的版本中会被移除，因此建议使用 `wire.Struct` 注入结构体类型的提供者；

**Bind**

`func Bind(iface, to interface{}) Binding` Bind 方法返回的是一个 `Binding` 类型的结构体，其作用是将 `to` 表示的具体类绑定到 `iface` 上，即对象依赖图中所有对 `iface` 的依赖都会被注入 `to`，需要将 `to` 对应的 `provider` 注入到 wire 中；其中入参 `iface` 和 `to` 必须是指针：
```go
type Fooer interface {
	Foo()
}

type MyFoo struct{}

func (MyFoo) Foo() {}

var MySet = wire.NewSet(
	wire.Struct(new(MyFoo)),
	wire.Bind(new(Fooer), new(MyFoo)),
)
```

从示例代码中可以看到，`Bind` 的返回值最后是注入通过 `NewSet` 方法成为 `ProviderSet` 的一部分；

**Value**

`func Value(interface{}) ProvidedValue` Value 方法返回的是 `ProvidedValue` 类型的结构体。`Value` 入参所关联的表达式不应该是 `interface`，要关联 `interface`，建议使用 `InterfaceValue` 方法；

**InterfaceValue**

`func InterfaceValue(typ interface{}, x interface{}) ProvidedValue` InterfaceValue 方法也返回 `ProvidedValue` 类型的结构体。其中 `typ` 是一个指向 `interface` 的指针，`x` 是具体的实现；

**Struct**

`func Struct(structType interface{}, fieldNames ...string) StructProvider` Struct 方法表示对于给定的 `structType` 类型结构体，需要 Wire 注入 `fieldNames` 指定的字段；第一个参数需要是指针；Wire 会同时注入 `structType` 和 `*structType`。特别的，`fieldNames` 可以通过 "\*" 来表示结构体的所有字段；
```go
type S struct {
	MyFoo *Foo
	MyBar *Bar
}
 
var Set = wire.NewSet(wire.Struct(new(S), "MyFoo")) // inject only S.MyFoo

var Set = wire.NewSet(wire.Struct(new(S), "*")) // inject all fields
```

**FieldsOf**

`func FieldsOf(structType interface{}, fieldNames ...string) StructFields` FieldsOf 方法返回 `StructFields` 结构体；该方法表示给定结构体(structType) 的指定字段(fieldNames) 将成为各自类型的提供者；第一个参需要是指向结构体的指针或者指向结构体指针的指针；如果第一个参数是指向指针的指针，那么针对指定的结构体字段，Wire 将额外注入其对应的指针类型；
```go
type S struct {
	MyFoo Foo
	MyBar Bar
}
func NewStruct() S { /* ... */ }
var Set = wire.NewSet(wire.FieldsOf(new(S), "MyFoo", "MyBar"))

// or
func NewStruct() *S { /* ... */ }
var Set = wire.NewSet(wire.FieldsOf(new(*S), "MyFoo", "MyBar"))
```

上述代码示例所表示的两个场景中，前者对于 `Foo` 和 `Bar`, Wire 将分别使用 `S.MyFoo`、 `S.MyBar` 作为 `Foo` 和 `Bar` 的 `provider`；后者 Wire 还会分别提供 `*Foo` 和 `*Bar`；这里有一个问题，就是 `S` 是怎么创建的呢？

## 入口方法
`Build` 方法体将在 `injector` 函数中被调用，Wire 的代码生成工具将在 injector 函数中生成调用 `provider` 的代码来组装需要实体；`Build` 函数的入参与 `NewSet` 一样，将成为其类型的 `provider`。

`Build` 方法的第一个返回值是 injector 函数的返回值，第二个可选的返回值是一个 `cleanup` 方法；最后一个可选的返回值是 `error`；如果注册到 Wire 的 `provider` 函数返回了 `error` 或者 `cleanup` 方法，那么 injector 函数的签名中也必须存在对应的元素；

一个 wire.go 文件中可以包含多个 `injector` 方法；

# 核心代码 & 原理
Wire 工具支持多个子命令，本节只会涉及代码生成部分的核心原理与源码解读，其余代码如果有兴趣可以自行探索；
正如把大象装入冰箱分为 3 步一样，Wire 的工作流程也分为 3 步：
1. 收集依赖关系；
2. 解析依赖关系得到调用次序；
3. 生成代码；

## 入口
Wire 作为命令行工具，其内部实现了一套子命令分派流程。简单来说就是根据传递给 Wire 的命令行参数决定执行哪个子逻辑。以下是 cmd\\wire\\main.go 的 `main` 方法，也是工具的入口：
```go
func main() {

    subcommands.Register(subcommands.CommandsCommand(), "")
    
    subcommands.Register(subcommands.FlagsCommand(), "")
    
    subcommands.Register(subcommands.HelpCommand(), "")

    subcommands.Register(&checkCmd{}, "")

    subcommands.Register(&diffCmd{}, "")

    subcommands.Register(&genCmd{}, "")

    subcommands.Register(&showCmd{}, "")

    flag.Parse()

  

    // Initialize the default logger to log to stderr.

    log.SetFlags(0)

    log.SetPrefix("wire: ")

    log.SetOutput(os.Stderr)

  

    // TODO(rvangent): Use subcommands's VisitCommands instead of hardcoded map,

    // once there is a release that contains it:

    // allCmds := map[string]bool{}

    // subcommands.DefaultCommander.VisitCommands(func(_ *subcommands.CommandGroup, cmd subcommands.Command) { allCmds[cmd.Name()] = true })

    allCmds := map[string]bool{

        "commands": true, // builtin

        "help":     true, // builtin

        "flags":    true, // builtin

        "check":    true,

        "diff":     true,

        "gen":      true,

        "show":     true,

    }

    // Default to running the "gen" command.

    if args := flag.Args(); len(args) == 0 || !allCmds[args[0]] {

        genCmd := &genCmd{}

        os.Exit(int(genCmd.Execute(context.Background(), flag.CommandLine)))

    }

    os.Exit(int(subcommands.Execute(context.Background())))

}
```

可以看到如果执行 wire 命令时没有传递任何参数，默认是执行 `gemCmd` 的 `Execute` 方法，当然也可以指定具体命令的名称，如 gen，`subcommands.Execute` 方法内部会根据命令名称进行分派；所以，我们关注的重点就是 `genCmd` 的 `Execute` 方法；

前文提到，在 Wire 中，我们通过函数来描述依赖关系：入参类型被返回值类型所依赖；我们将这些函数作为 `provider` 放在了 `ProviderSet` 等依赖关系收集器中， 然后将这些依赖关系收集器作为参数传递给了 `wire.Build()` 方法。

那么 Wire 是如何感知到我们注入的这些依赖关系呢？答案是抽象语法树。

## 概览
`genCmd.Execute` 方法中，核心逻辑是：
```go
outs, errs := wire.Generate(ctx, wd, os.Environ(), packages(f), opts)
```
其中`GenerateResult` 类型的 `outs` 即为所得到的源码，可以通过 `out.Commit()` 将得到的源码文件；

这是 `wire.Generate` 方法的代码：
```go
func Generate(ctx context.Context, wd string, env []string, patterns []string, opts *GenerateOptions) ([]GenerateResult, []error) {

    if opts == nil {

        opts = &GenerateOptions{}

    }

	// 关键第 1 步，获取到需要处理的 pkg；
    pkgs, errs := load(ctx, wd, env, opts.Tags, patterns)

    if len(errs) > 0 {

        return nil, errs

    }

    generated := make([]GenerateResult, len(pkgs))

	// 处理 pkg，每一个 pkg 对应一个 GenerateResult；
    for i, pkg := range pkgs {

        generated[i].PkgPath = pkg.PkgPath

		// 获取输出目录
        outDir, err := detectOutputDir(pkg.GoFiles)

        if err != nil {

            generated[i].Errs = append(generated[i].Errs, err)

            continue

        }
		// 获取具体的输出文件，这里设置了 wire_gen.go 的后缀
        generated[i].OutputPath = filepath.Join(outDir, opts.PrefixOutputFile+"wire_gen.go")
		// 每一个 pkg，得到一个 g
        g := newGen(pkg)
		// 关键第 2 步，对于给定的 pkg，生成 ast.File 文件 
        injectorFiles, errs := generateInjectors(g, pkg)

        if len(errs) > 0 {

            generated[i].Errs = errs

            continue

        }

		// 复制已有的变量声明到 g
        copyNonInjectorDecls(g, injectorFiles, pkg.TypesInfo)
		// 关键第 3 步，生成 go 代码
        goSrc := g.frame(opts.Tags)

        if len(opts.Header) > 0 {

            goSrc = append(opts.Header, goSrc...)

        }
		// 格式化一下
        fmtSrc, err := format.Source(goSrc)

        if err != nil {

            // This is likely a bug from a poorly generated source file.

            // Add an error but also the unformatted source.

            generated[i].Errs = append(generated[i].Errs, err)

        } else {

            goSrc = fmtSrc

        }
		// 保存生成的结果
        generated[i].Content = goSrc

    }

    return generated, nil

}
```
`wire.Generate` 方法中的关键第 1 步是获取到要生成代码的 `package`:
```go
pkgs, errs := load(ctx, wd, env, opts.Tags, patterns)
```
`pkg` 的类型是 golang.org/x/tools/go/packages 包下的 `Package`，该类型表示被加载的 Go 的 package；

关键第 2 步实际上是生成 `ast.File` 数组，并且在这个过程中可能对 `g` 填充了一些内容；`ast.File` 可以理解为一个 Go 文件；

最后一个关键步骤就是生成代码了，然后将其保存在 `GenerateResult` 中，最后由调用者决定是写入文件还是打印到控制台；

因此，要理解 Wire 的核心工作流程，就需要明白 `injectorFiles` 的创建过程以及这个过程中对 `g` 所进行的操作；

 `injectorFiles` 由 `generateInjectors` 方法创建；该方法的核心是处理前面我们加载的 `pkg` 中的语法树；

## 收集依赖关系
收集依赖关系包含这几个方面：
1. 获取 `injector` 方法，以便收集我们注入的各种 `provider`；
2. 解析各种 `provider`，收集类型之间的依赖关系以及类型绑定关系等；

处理的逻辑入口是 `generateInjectors` 方法：
```go
func generateInjectors(g *gen, pkg *packages.Package) (injectorFiles []*ast.File, _ []error) {

    oc := newObjectCache([]*packages.Package{pkg})

    injectorFiles = make([]*ast.File, 0, len(pkg.Syntax))

    ec := new(errorCollector)
	// 核心 for 循环
    for _, f := range pkg.Syntax {
	    ...
    }
    
	if len(ec.errors) > 0 {
        return nil, ec.errors
    }
    return injectorFiles, nil
```

因此，核心 `for` 循环中的逻辑可以分为两部分，处理 Declaration 和 Imports：
```go
for _, f := range pkg.Syntax {
	for _, decl := range f.Decls {
		...
	}

	for _, impt := range f.Imports {
		...
	}
}

```

`f.Decls` 的类型是 `[]ast.Decl` ，其中 `ast.Decl` 表示抽象语法树中的“声明”节点（接口），该节点有三种具体的实现：`BadDecl`、`GenDecl` 和 `FuncDecl`；

### 定位 injector 方法
`injector` 方法是由 `findInjectorBuild` 方法完成定位的，其中最为重要的是对 `qualifiedIdentObject` 函数的调用：
```go
func findInjectorBuild(info *types.Info, fn *ast.FuncDecl) (*ast.CallExpr, error) {
	...
	for _, stmt := range fn.Body.List {
		switch stmt := stmt.(type) {
		case *ast.ExprStmt:
			numStatements++
			if numStatements > 1 {
				invalid = true
			}
			call, ok := stmt.X.(*ast.CallExpr)
			if !ok {
				continue
			}
			// 关键第 1 步
			if qualifiedIdentObject(info, call.Fun) == types.Universe.Lookup("panic") {
				if len(call.Args) != 1 {
					continue
				}
				call, ok = call.Args[0].(*ast.CallExpr)
				if !ok {
					continue
				}
			}
			buildObj := qualifiedIdentObject(info, call.Fun)
			if buildObj == nil || buildObj.Pkg() == nil || !isWireImport(buildObj.Pkg().Pa
				continue
			}
			wireBuildCall = call
		case *ast.EmptyStmt:
			// Do nothing.
		case *ast.ReturnStmt:
			// Allow the function to end in a return.
			if numStatements == 0 {
				return nil, nil
			}
		default:
			invalid = true
		}

	}
	...
}
func qualifiedIdentObject(info *types.Info, expr ast.Expr) types.Object {
    switch expr := expr.(type) {
    case *ast.Ident:
        return info.ObjectOf(expr)
    case *ast.SelectorExpr:
        pkgName, ok := expr.X.(*ast.Ident)
        if !ok {
            return nil
        }

        if _, ok := info.ObjectOf(pkgName).(*types.PkgName); !ok {
            return nil
        }

        return info.ObjectOf(expr.Sel)
    default:
        return nil
    }
}

```

`findInjectorBuild` 里关键点1 处对应于 “panic”模式调用的 `wire.Build`，在之后的逻辑分支里，实际上修正了 call 变量；

至此，我们拿到了抽象语法树中表示 `wire.Build` 方法调用的节点，同时确认了 `injector` 方法：当且仅当一个方法在函数体内只调用了 `wire.Build` 函数时，我们称这个方法为 `injector`； 

### 解析 injector 方法
解析 `injector` 的目的很明确，就是得到入参以及返回值；关键函数为 `injectorFuncSignature`：
```go
func injectorFuncSignature(sig *types.Signature) (*types.Tuple, outputSignature, error) {
	out, err := funcOutput(sig)
	if err != nil {
		return nil, outputSignature{}, err
	}
	return sig.Params(), out, nil
}

```

可以看到，`injectorFuncSignature` 方法通过 `findOutput` 方法获取返回值：
```go
func funcOutput(sig *types.Signature) (outputSignature, error) {
	results := sig.Results()
	switch results.Len() {
	case 0:
		return outputSignature{}, errors.New("no return values")
	case 1:
		return outputSignature{out: results.At(0).Type()}, nil
	case 2:
		out := results.At(0).Type()
		switch t := results.At(1).Type(); {
		case types.Identical(t, errorType):
			return outputSignature{out: out, err: true}, nil
		case types.Identical(t, cleanupType):
			return outputSignature{out: out, cleanup: true}, nil
		default:
			return outputSignature{}, fmt.Errorf("second return type is %s; must be error 
		}
	case 3:
		if t := results.At(1).Type(); !types.Identical(t, cleanupType) {
			return outputSignature{}, fmt.Errorf("second return type is %s; must be func()
		}
		if t := results.At(2).Type(); !types.Identical(t, errorType) {
			return outputSignature{}, fmt.Errorf("third return type is %s; must be error",
		}
		return outputSignature{
			out:     results.At(0).Type(),
			cleanup: true,
			err:     true,
		}, nil
	default:
		return outputSignature{}, errors.New("too many return values")
	}
}
```

从 `funcOutput` 方法中可以很清晰地认识到 Wire 是如何支持 `injector` 拥有多个返回值，即对可选的 `cleanup` 函数以及 `error` 的支持；

`injectorFuncSignature` 的返回结果被包装到 InjectorArgs 结构体中：
```go
injectorArgs := &InjectorArgs{
	Name:  fn.Name.Name,
	Tuple: ins,
	Pos:   fn.Pos(),
}
```

至此，我们得到了 `injector` 的输入、输出以及对 `wire.Build` 的调用；接下来需要收集各种 `provider`，这一步是在 `objectCache.processNewSet` 方法中完成的；

### 收集 provider
```go
func (oc *objectCache) processNewSet(info *types.Info, pkgPath string, call *ast.CallExpr, args *InjectorArgs, varName string) (*ProviderSet, []error) {
	// Assumes that call.Fun is wire.NewSet or wire.Build.

	pset := &ProviderSet{
		Pos:          call.Pos(),
		InjectorArgs: args,
		PkgPath:      pkgPath,
		VarName:      varName,
	}
	ec := new(errorCollector)
	for _, arg := range call.Args {
		// 第一步，将 ast.Expr 转换为 Wire 的内部结构体
		item, errs := oc.processExpr(info, pkgPath, arg, "")
		if len(errs) > 0 {
			ec.add(errs...)
			continue
		}
		// 第二步，将内部结构体放入 ProviderSet 的对应字段
		switch item := item.(type) {
		case *Provider:
			pset.Providers = append(pset.Providers, item)
		case *ProviderSet:
			pset.Imports = append(pset.Imports, item)
		case *IfaceBinding:
			pset.Bindings = append(pset.Bindings, item)
		case *Value:
			pset.Values = append(pset.Values, item)
		case []*Field:
			pset.Fields = append(pset.Fields, item...)
		default:
			panic("unknown item type")
		}
	}
	if len(ec.errors) > 0 {
		return nil, ec.errors
	}
	var errs []error
	pset.providerMap, pset.srcMap, errs = buildProviderMap(oc.fset, oc.hasher, pset)
	if len(errs) > 0 {
		return nil, errs
	}
	if errs := verifyAcyclic(pset.providerMap, oc.hasher); len(errs) > 0 {
		return nil, errs
	}
	return pset, nil
}

```

可以看到，`processNewSet` 的逻辑是：
1. 遍历方法调用的参数，即 `call.Args`；
2. 构建 ProviderMapp；
3. 校验是否存在循环依赖；
其中 2 & 3 步骤属于解析依赖关系，1 属于收集依赖关系；因此本节主要关注步骤 1；而处理的结果被放入 `wire.PrviderSet` 需要注意的是，这里的 `ProviderSet` 与 Wire 提供给应用程序注册 `provider` 的 `ProviderSet` 仅是名称相同，并不是同一个结构体；它们之间实际上是存在一定的映射关系的； 
首先看步骤 1 的处理整体处理逻辑。

第一步将 `ast.Expr` 转换为 Wire 内部的结构体，这是通过 `processExpr` 方法完成的；

第二步按照不同的类别，将其放入 `ProviderSet` 的对应字段；

接下来，我们重点看看 `processExpr` 的实现：

```go
func (oc *objectCache) processExpr(info *types.Info, pkgPath string, expr ast.Expr, varName string) (interface{}, []error) {
    exprPos := oc.fset.Position(expr.Pos())
	exprPos := oc.fset.Position(expr.Pos())
	expr = astutil.Unparen(expr)
	if obj := qualifiedIdentObject(info, expr); obj != nil {
		item, errs := oc.get(obj)
		return item, mapErrors(errs, func(err error) error {
			return notePosition(exprPos, err)
		})
	}
	if call, ok := expr.(*ast.CallExpr); ok {
		fnObj := qualifiedIdentObject(info, call.Fun)
		if fnObj == nil {
			return nil, []error{notePosition(exprPos, errors.New("unknown pattern fnObj nil"))}
		}
		pkg := fnObj.Pkg()
		if pkg == nil {
			return nil, []error{notePosition(exprPos, fmt.Errorf("unknown pattern - pkg in fnObj is 
		}
		if !isWireImport(pkg.Path()) {
			return nil, []error{notePosition(exprPos, errors.New("unknown pattern"))}
		}
		switch fnObj.Name() {
		case "NewSet":
			pset, errs := oc.processNewSet(info, pkgPath, call, nil, varName)
			return pset, notePositionAll(exprPos, errs)
		case "Bind":
			b, err := processBind(oc.fset, info, call)
			if err != nil {
				return nil, []error{notePosition(exprPos, err)}
			}
			return b, nil
		case "Value":
			v, err := processValue(oc.fset, info, call)
			if err != nil {
				return nil, []error{notePosition(exprPos, err)}
			}
			return v, nil
		case "InterfaceValue":
			v, err := processInterfaceValue(oc.fset, info, call)
			if err != nil {
				return nil, []error{notePosition(exprPos, err)}
			}
			return v, nil
		case "Struct":
			s, err := processStructProvider(oc.fset, info, call)
			if err != nil {
				return nil, []error{notePosition(exprPos, err)}
			}
			return s, nil
		case "FieldsOf":
			v, err := processFieldsOf(oc.fset, info, call)
			if err != nil {
				return nil, []error{notePosition(exprPos, err)}
			}
			return v, nil
		default:
			return nil, []error{notePosition(exprPos, errors.New("unknown pattern"))}
		}
	}
	if tn := structArgType(info, expr); tn != nil {
		p, errs := processStructLiteralProvider(oc.fset, tn)
		if len(errs) > 0 {
			return nil, notePositionAll(exprPos, errs)
		}
		return p, nil
	}
	return nil, []error{notePosition(exprPos, errors.New("unknown pattern"))}
}
```
`processExpr` 的结构也非常清晰：将 `expr` 分为了三种情况：
1. 具体的对象，如前文所提到的“关键结构体”；
2. wire 相关函数调用，如 `Bind`、`Value`、`NewSet` 等；
3. 某一具体的结构体，如 xxx.YYY{} 等；
以上三种情况即为 `wire.Build` 参数的可能情况；

通过表达式获取关联的对象，这一步骤仍然是通过 `qualifiedIdentObject` 方法实现的；

对于第一种情况，`processExpr` 直接返回 `objectCache.get()` 返回的对象即可；

对于第二种情况，首先通过 `qualifiedIdentObject` 获取到表示方法调用的 `fnObj`；然后不同的方法经过不同的处理函数，获取到对应的 Wire 内部结构体；支持的方法有：`NewSet`、`Bind`、`Value`、`InterfaceValue`、`Struct`，这些和前文提到的“关键函数”是一致的；

这里由于篇幅所限，对具体的处理函数不做深入的分析和学习；但是对这些函数的学习有助于加深理解 go 语言所提供的 ast 包的能力；

    从学习 Wire 的实现原理角度来看，这部分内容相对来说不太重要，因为我们需要关注的是 Wire 的处理逻辑，而非处理细节；从学习 go 语言 ast 包能力的角度来看，这部分实际上很重要。因为这部分属于语言能力的具体落地应用，学习价值很高；
	考虑到，本文的主题是对 Wire 实现原理的学习，因此这部分内容留到后面有机会再进行深入的学习；

对于第三种情况，`structArgType` 方法获取到表达式所对应的类型名称，然后将其返回：
```go
...

tn, ok := qualifiedIdentObject(info, lit.Type).(*types.TypeName)

...

return tn
```

至此，依赖关系已经收集完毕，这些关系被存放到 Wire 内部的结构体 `ProviderSet` 中；接下来要学习的内容是 Wire 如何处理收集到的依赖关系：`buildProviderMap` 方法的实现；

## 解析依赖关系
首先看 `buildProviderMap` 的签名：
```go
func buildProviderMap(fset *token.FileSet, hasher typeutil.Hasher, set *ProviderSet) (*typeutil.Map, *typeutil.Map, []error)
```
入参方面，`fset` 来自我们加载的 `package` 中的 `FSet` 字段，该字段主要为包内包含的 Types、TypesInfo 以及 Syntax 提供位置信息；`hasher` 字段用来保存类型的哈希值，该哈希值用以判断两个类型是否相同；`set` 则是之前“收集依赖关系”阶段的产物，包含了依赖关系；

返回值方面，返回 `providerMap` 以及 `srcMap`，这两个值被用来覆盖 `set` 中的同名字段；即可以理解为 `buildProviderMap` 就是在构建 `ProviderSet` 的这两个字段：`providerMap` 将一个提供的类型映射到 `*ProvidedType`，`*ProvidedType`用来表示一个`provider` 具体提供的类型；`srcMap` 将一个提供的类型映射到 `*providerSetSrc`，`*providerSetSrc`用以表示某一类型的来源；

`buildProviderMap` 方法处理的内容包括：
1. set.InjectorArgs：处理 `injector` 的入参；
2. set.Imports：处理 `imports`，检查是否有绑定冲突；
3. set.Providers：处理非绑定的 `provider`；
4. set.Values：处理 `wire.Value` 方式提供的 `provider`；
5. set.Fields：处理 `wire.Fields` 表示的 `provider`
6. set.Bindings：最后处理 `wire.Bind` 表示的绑定关系，只能在最后一步进行，因为需要确保具体的 `provider` 已经被处理收集；
这里可以看看处理 `set.Bindings` 的逻辑，可以更好地理解 `providerMap` 和 `srcMap` 的差异：
```go
for _, b := range set.Bindings {
	src := &providerSetSrc{Binding: b}
	if prevSrc := srcMap.At(b.Iface); prevSrc != nil {
		ec.add(bindingConflictError(fset, b.Iface, set, src, prevSrc.(*providerSetSrc)
		continue
	}
	concrete := providerMap.At(b.Provided)
	if concrete == nil {
		setName := set.VarName
		if setName == "" {
			setName = "provider set"
		}
		ec.add(notePosition(fset.Position(b.Pos), fmt.Errorf("wire.Bind of concrete ty
		continue
	}
	providerMap.Set(b.Iface, concrete)
	srcMap.Set(b.Iface, src)
}
```
首先从 `providerMap` 里获取到某一类型的具体 `provider`，然后将待绑定 `type` 的 `provider` 记录为取出的 `provider`；

在得到 `providerSet` 的 `providerMap`  以及 `srcMap` 后，需要对收集到的依赖关系进行“循环依赖检测”；这一步是在 `verifyAcyclic` 方法里完成的；

`verifyAcyclic` 方法会检查 `providerMap` 中的每一个 `provider` 是否存在依赖冲突（循环依赖）的情况；该方法采用深度优先的方式遍历依赖图，同时采用单独的变量标记某个 `provider` 是否被检查过；`verifyAcyclic` 采用 “for 循环 + 栈”的方式实现深度优先遍历；

将某个 `provider` 的依赖链以数组的形式存储起来，然后再以数组的方式存储各个 `provider`, 即以二维数组的方式实现深度优先的处理顺序；

关于 `verifyAcyclic` 方法，可以从三个方面进行学习：如何入栈？如何出栈？如何判断存在循环依赖？
首先，入栈的元素从类型上说是一个数组，该数组的结构是依赖链+类型 A；该数组本身是首位元素的依赖链，即 S\[0\] 依赖 S\[1\]...S\[len-1\]，而类型 A 则表示下一个要处理的 `provider`；
出栈则意味着取出类型 A，然后处理该元素的依赖；

判断循环依赖则比较简单，因为我们已经存储了整条依赖链，只需要检查这条依赖链上是否存在当前处理元素的依赖即可：如果存在则表示有循环依赖，如果没有那么继续处理即可；
对于上面三个方面，这里贴出对应的代码：

入栈：
```go
if !hasCycle {
	next := append(append([]types.Type(nil), curr...), a)
	stk = append(stk, next)
}
```

出栈：
```go
curr := stk[len(stk)-1]
stk = stk[:len(stk)-1]
head := curr[len(curr)-1]
```

检查循环依赖：
```go
for _, a := range args { // args 即为 head 的依赖
	hasCycle := false
	for i, b := range curr {
		if types.Identical(a, b) {
			// 打印错误日志
		}
	}	
	// 执行入栈
}
```

至此，解析依赖关系的步骤就算是完成了！接下来，我们将开始下一段旅程——生成代码；

## 生成代码
生成代码阶段的第一个处理步骤是在 gen 的 `inject` 方法中完成的：
```go
func (g *gen) inject(pos token.Pos, name string, sig *types.Signature, set *ProviderSet, doc *ast.CommentGroup) []error {}
```
其中 `sig` 即为 `injector` 函数的签名，`set` 为补充了 `providerMap` 和 `srcMap` 的 `ProviderSet`，`doc` 是来自 `injector` 的 `CommentGroup`；
首先看 `inject` 的整体处理逻辑，然后关注重点函数的细节；

`inject` 函数首先获取到 `injector` 的函数签名——参数以及返回值，然后获取 `injector` 函数内部的调用，保存在 `calls` 中：
```go
injectSig, err := funcOutput(sig)
if err != nil {
	return []error{notePosition(g.pkg.Fset.Position(pos),
		fmt.Errorf("inject %s: %v", name, err))}
}
params := sig.Params()
calls, errs := solve(g.pkg.Fset, injectSig.out, params, set)
```

接下来就是遍历 `calls`，检查 `call` 的签名是否和 `injector` 的签名一致，这里包括是否有 `error` 以及 `cleanup` 函数；如果是 `value` 类型，则加入 `pendingVars` 数组：
```go
for i := range calls {
	c := &calls[i]
	if c.hasCleanup && !injectSig.cleanup {
		ts := types.TypeString(c.out, nil)
		ec.add(notePosition(
			g.pkg.Fset.Position(pos),
			fmt.Errorf("inject %s: provider for %s returns cleanup but injection does not return cleanup function", name, ts)))
	}
	if c.hasErr && !injectSig.err {
		ts := types.TypeString(c.out, nil)
		ec.add(notePosition(
			g.pkg.Fset.Position(pos),
			fmt.Errorf("inject %s: provider for %s returns error but injection not allowed to fail", name, ts)))
	}
	if c.kind == valueExpr {
		if err := accessibleFrom(c.valueTypeInfo, c.valueExpr, g.pkg.PkgPath); err != nil {
			// TODO(light): Display line number of value expression.
			ts := types.TypeString(c.out, nil)
			ec.add(notePosition(
				g.pkg.Fset.Position(pos),
				fmt.Errorf("inject %s: value %s can't be used: %v", name, ts, err)))
		}
		if g.values[c.valueExpr] == "" {
			t := c.valueTypeInfo.TypeOf(c.valueExpr)
			name := typeVariableName(t, "", func(name string) string { return "_wire" + export(name) + "Value" }, g.nameInFileScope)
			g.values[c.valueExpr] = name
			pendingVars = append(pendingVars, pendingVar{
				name:     name,
				expr:     c.valueExpr,
				typeInfo: c.valueTypeInfo,
			})
		}
	}
}
if len(ec.errors) > 0 {
	return ec.errors
}
```

最后便是生成代码的内容啦：
```go
// Perform one pass to collect all imports, followed by the real pass.
injectPass(name, sig, calls, set, doc, &injectorGen{
	g:       g,
	errVar:  disambiguate("err", g.nameInFileScope),
	discard: true,
})
injectPass(name, sig, calls, set, doc, &injectorGen{
	g:       g,
	errVar:  disambiguate("err", g.nameInFileScope),
	discard: false,
})
if len(pendingVars) > 0 {
	g.p("var (\n")
	for _, pv := range pendingVars {
		g.p("\t%s = ", pv.name)
		g.writeAST(pv.typeInfo, pv.expr)
		g.p("\n")
	}
	g.p(")\n\n")
}
```
这里调用了两遍 `injectPass`，其中第一遍的 `discard` 参数为 true，第二遍的 `discard` 参数为 false；

这里根据注释，第一遍调用的目的是收集 `imports` 信息，第二遍才是真正地生成代码内容，这是通过 `discard` 控制的，如果该值为 `true`，那么任何写的动作都不会被执行；

具体来说就是 `g` 的 `qualifyPkg` 方法，如：
```go
outTypeString := types.TypeString(injectSig.out, ig.g.qualifyPkg)
```
`qualifyPkg` 方法：
```go
func (g *gen) qualifyImport(name, path string) string {
	if path == g.pkg.PkgPath {
		return ""
	}
	// TODO(light): This is depending on details of the current loader.
	const vendorPart = "vendor/"
	unvendored := path
	if i := strings.LastIndex(path, vendorPart); i != -1 && (i == 0 || path[i-1] == '/') {
		unvendored = path[i+len(vendorPart):]
	}
	if info, ok := g.imports[unvendored]; ok {
		return info.name
	}
	// TODO(light): Use parts of import path to disambiguate.
	newName := disambiguate(name, func(n string) bool {
		// Don't let an import take the "err" name. That's annoying.
		return n == "err" || g.nameInFileScope(n)
	})
	g.imports[unvendored] = importInfo{
		name:    newName,
		differs: newName != name,
	}
	return newName
}
```
可以看到该方法内部是写入了 `g.imports` 字段；后续调用因为信息已经存在，会直接返回；

那么问题来了，为什么要调用两遍？只调用一遍可以吗？

个人感觉第一遍能收集的信息，即 `g.imports` 字段，第二遍生成代码的时候也可以收集；这里还没有找到很好的理解；

回到源码，`injectPass` 负责向 `g` 中写入生成的代码， 我们看看 `injectPass` 方法内部的实现逻辑；
```go
func injectPass(name string, sig *types.Signature, calls []call, set *ProviderSet, doc *ast.CommentGroup,ig *injectorGen) {
	params := sig.Params()
	injectSig, err := funcOutput(sig)
	if err != nil {
		// This should be checked by the caller already.
		panic(err)
	}
	if doc != nil {
		for _, c := range doc.List {
			ig.p("%s\n", c.Text)
		}
	}
	ig.p("func %s(", name)
	for i := 0; i < params.Len(); i++ {
		if i > 0 {
			ig.p(", ")
		}
		pi := params.At(i)
		a := pi.Name()
		if a == "" || a == "_" {
			a = typeVariableName(pi.Type(), "arg", unexport, ig.nameInInjector)
		} else {
			a = disambiguate(a, ig.nameInInjector)
		}
		ig.paramNames = append(ig.paramNames, a)
		if sig.Variadic() && i == params.Len()-1 {
			// Keep the varargs signature instead of a slice for the last argument if the
			// injector is variadic.
			ig.p("%s ...%s", ig.paramNames[i], types.TypeString(pi.Type().(*types.Slice).Elem(), ig.g.qualifyPkg))
		} else {
			ig.p("%s %s", ig.paramNames[i], types.TypeString(pi.Type(), ig.g.qualifyPkg))
		}
	}
	outTypeString := types.TypeString(injectSig.out, ig.g.qualifyPkg)
	switch {
	case injectSig.cleanup && injectSig.err:
		ig.p(") (%s, func(), error) {\n", outTypeString)
	case injectSig.cleanup:
		ig.p(") (%s, func()) {\n", outTypeString)
	case injectSig.err:
		ig.p(") (%s, error) {\n", outTypeString)
	default:
		ig.p(") %s {\n", outTypeString)
	}
	for i := range calls {
		c := &calls[i]
		lname := typeVariableName(c.out, "v", unexport, ig.nameInInjector)
		ig.localNames = append(ig.localNames, lname)
		switch c.kind {
		case structProvider:
			ig.structProviderCall(lname, c)
		case funcProviderCall:
			ig.funcProviderCall(lname, c, injectSig)
		case valueExpr:
			ig.valueExpr(lname, c)
		case selectorExpr:
			ig.fieldExpr(lname, c)
		default:
			panic("unknown kind")
		}
	}
	if len(calls) == 0 {
		ig.p("\treturn %s", ig.paramNames[set.For(injectSig.out).Arg().Index])
	} else {
		ig.p("\treturn %s", ig.localNames[len(calls)-1])
	}
	if injectSig.cleanup {
		ig.p(", func() {\n")
		for i := len(ig.cleanupNames) - 1; i >= 0; i-- {
			ig.p("\t\t%s()\n", ig.cleanupNames[i])
		}
		ig.p("\t}")
	}
	if injectSig.err {
		ig.p(", nil")
	}
	ig.p("\n}\n\n")
}
```

首先处理 `injector`，写入函数名、参数，然后根据有无 `cleanup` 、`error` 写入返回值。“写入”这个动作是通过 `ig.p` 函数完成的；

接下来处理传入参数 `calls`，首先获取变量的名称，然后根据不同的种类：结构体、函数、值和字段，分别写入生成的代码；最后处理生成代码的 `cleanup` 和 `error`；

接下来需要回到 `Generate` 方法；前面提到过，Wire 代码生成最重要的是获取到 `injectorFiles` 以及 `g`；通过前面的学习，我们已经知道要生成的代码内容已经写入到了 `g` 中；接下来需要复制一些非 `injector` 的声明到 `g` 中，这是通过 `copyNonInjectorDecls` 方法完成的，来看看 `copyNonInjectorDecls` 的代码：

```go
func copyNonInjectorDecls(g *gen, files []*ast.File, info *types.Info) {
	for _, f := range files {
		name := filepath.Base(g.pkg.Fset.File(f.Pos()).Name())
		first := true
		for _, decl := range f.Decls {
			switch decl := decl.(type) {
			case *ast.FuncDecl:
				// OK to ignore error, as any error cases should already have
				// been filtered out.
				if buildCall, _ := findInjectorBuild(info, decl); buildCall != nil {
					continue
				}
			case *ast.GenDecl:
				if decl.Tok == token.IMPORT {
					continue
				}
			default:
				continue
			}
			if first {
				g.p("// %s:\n\n", name)
				first = false
			}
			// TODO(light): Add line number at top of each declaration.
			g.writeAST(info, decl)
			g.p("\n\n")
		}
	}
```
核心是 `g.writeAST` 方法：
```go
func (g *gen) writeAST(info *types.Info, node ast.Node) {
	node = g.rewritePkgRefs(info, node)
	if err := printer.Fprint(&g.buf, g.pkg.Fset, node); err != nil {
		panic(err)
	}
}
```
首先通过 `rewritePkgRefs` 重写相关引用的包名，然后通过 `Fprint` 方法将 `node` 写入 `g.buf` 中，从而完成“复制”；

`rewritePkgRefs` 函数比较长，功能实现依赖于 go 的 ast 包，本文所关注的是 Wire 代码生成的逻辑原理，所以这部分内容并不是本文所关注的重点，故不再展开。但这并不是说这部分内容不重要，相反这部分内容正是 ast 相关内容实践的绝佳例子。有机会详细聊聊 Wire 与 go/ast 包。

到目前为止，该有的信息都已经处理完毕，该输出最后完整形态的生成代码啦：

```go
// func Generate
copyNonInjectorDecls(g, injectorFiles, pkg.TypesInfo)
goSrc := g.frame(opts.Tags)
if len(opts.Header) > 0 {
	goSrc = append(opts.Header, goSrc...)
}
fmtSrc, err := format.Source(goSrc)
if err != nil {
	// This is likely a bug from a poorly generated source file.
	// Add an error but also the unformatted source.
	generated[i].Errs = append(generated[i].Errs, err)
} else {
	goSrc = fmtSrc
}
generated[i].Content = goSrc
```
首先通过 `g.frame` 写入文件头、编译 tag、包名、import，然后写入 `g.buf` 中的内容；接下来通过 `format.Source` 格式化内容，最后将其记录在 `GenerateResult.Content` 字段中；

至此，代码生成部分的内容就介绍完毕啦；

# 最佳实践
**使用有区别的类型**
如希望提供字符串类型的 MySQL 连接字符串，则定义一种新的类型，以此避免冲突：
```go
type MySQLConnectionString string
```

**使用 `Options` 结构体**
如果一个提供者有许多外部依赖，那么将这些依赖组合在 `Options` 结构体中：
```go
type Options struct {
    // Messages is the set of recommended greetings.
    Messages []Message
    // Writer is the location to send greetings. nil goes to stdout.
    Writer io.Writer
}

func NewGreeter(ctx context.Context, opts *Options) (*Greeter, error) {
// ...
}

var GreeterSet = wire.NewSet(wire.Struct(new(Options), "*"), NewGreeter)
```

**关于库中的 `provider` 集合**
在类库中使用 `provider` 集合时，只有以下变更不会破坏兼容性：
- 在没有引入新输入类型的情况下，替换 `provider` 集合中的某个 `provider`；新的 `provider`可以有更少的输入入参。需要注意的是，在没有重新生成代码之前，注册者函数使用的仍旧是旧的 `provider`
- 仅当某个类型从未出现在 `provider` 集合中时，可以引入新的输出类型；否则，有可能该类型已经在其他 `provider` 集合中包含了，此时就会产生冲突；
其他的改动是不安全的，如：
- `provider` 集合中引入新的输入；
- 移除 `provider` 集合的输出类型；
- 添加已经存在的输出类型到 `provider` 集合；
对于破坏性的改动，考虑添加一个新的 `provider` 集合；如对于如下示例：
```go
var GreeterSet = wire.NewSet(NewStdoutGreeter)

func DefaultGreeter(ctx context.Context) *Greeter {
    // ...
}

func NewStdoutGreeter(ctx context.Context, msgs []Message) *Greeter {
    // ...
}

func NewGreeter(ctx context.Context, w io.Writer, msgs []Message) (*Greeter, error) {
    // ...
}

```

对于 `GreeterSet` 而言，可以使用 `DefaultGreeter` 替换 `NewStdoutGreeter`，因为 `DefaultGreeter` 相比 `NewStdoutGreeter` 所需要的输入依赖更少；但是不能用 `NewGreeter` 替换 `NewStdoutGreeter`，因为 `NewGreeter` 引入了新的输入依赖 `io.Writer`，同时也不能额外新增一个 `io.Writer` 的 `provider`，因为有可能其他类库可能已经提供了 `io.Writer`；

所以，在为类库代码选择 `provider` 集合的输出类型时需要更加小心；一般来说，`provider` 集合越小越好，这样和其他类库冲突的可能就越小；类库 `provider` 集合的输出类型一般仅包括类库自己所定义的类型，而类库需要的输入类型一般作为依赖项，类库自己不提供 `provider`，而是由使用方注入；

**Mock 依赖**
一般来说，对依赖的 Mock 有两种方式：
1. 将 Mock 作为参数传入注册者函数，从而使得某个接口实现被 Mock 的效果；
2. 通过 `provider` 集合描述 Mock 绑定关系，并且通过新的返回值来访问应用和依赖项；
上面两种方法不同点是注入 Mock 的方法，共同点时都需要额外的注册者函数来创建具有 Mock 功能的应用；
Mock 的核心目的有两个：
1. 得到接口 A 的 Mock 实现和待测试应用；
2. 将  Mock 实现注入应用；
1 是为了实现 Mock 逻辑和触发待测试的逻辑；2 是为了让 Mock 的逻辑在待测试应用中生效；
方法 1 和 方法 2 的不同实际上就是这两个步骤的不同：
1. 方法 1，手动创建接口 A 的 Mock 实现，然后将其通过参数的形式传递给 Wire 以此来实现注入；
2. 方法 2，让 Wire 创建带有 Mock 实现和应用的新类型 B，从而获得了 Mock 实现和应用；


# 总结整理
本文介绍了 Go 依赖注入工具 Wire 的安装、使用以及核心代码和原理等内容。不仅详细介绍了 Wire 的用法，还在源码级别介绍了 Wire 的工作流程和原理；

出于聚焦 “Wire 工作流程和原理”这一目的的考虑，本文忽略了对 ast 包所提供的能力在 Wire 中的应用的介绍，这部分内容以后有机会再详细学习交流；
