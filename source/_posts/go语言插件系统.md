---
title: go语言插件系统
tags:
  - 技术
  - Go语言
categories: 技术
comments: true
cover: 'https://cdn.jsdelivr.net/gh/shimengfei/cdn/guidao/1.png'
abbrlink: cd337a10
date: 2021-05-10 22:27:49
---
**本文章转载自**  {% btn 'https://draveness.me/golang/docs/part4-advanced/ch08-metaprogramming/golang-plugin/',draveness,far fa-hand-point-right,orange larger %}
---

# 8.1 插件系统 
熟悉 Go 语言的开发者一般都非常了解 Goroutine 和 Channel 的原理，包括如何设计基于 CSP 模型的应用程序，但是 Go 语言的插件系统是很少有人了解的模块，通过插件系统，我们可以在运行时加载动态库实现一些比较有趣的功能。

## 8.1.1 设计原理
Go 语言的插件系统基于 C 语言动态库实现的，所以它也继承了 C 语言动态库的优点和缺点，我们在本节中会对比 Linux 中的静态库和动态库，分析它们各自的特点和优势。

静态库或者静态链接库是由编译期决定的程序、外部函数和变量构成的，编译器或者链接器会将程序和变量等内容拷贝到目标的应用并生成一个独立的可执行对象文件1；
动态库或者共享对象可以在多个可执行文件之间共享，程序使用的模块会在运行时从共享对象中加载，而不是在编译程序时打包成独立的可执行文件2；
由于特性不同，静态库和动态库的优缺点也比较明显；只依赖静态库并且通过静态链接生成的二进制文件因为包含了全部的依赖，所以能够独立执行，但是编译的结果也比较大；而动态库可以在多个可执行文件之间共享，可以减少内存的占用，其链接的过程往往也都是在装载或者运行期间触发的，所以可以包含一些可以热插拔的模块并降低内存的占用。

![static-library-dynamic-library.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/static-library-dynamic-library.png)

图 8-1 静态库与动态库

使用静态链接编译二进制文件在部署上有非常明显的优势，最终的编译产物也可以直接运行在大多数的机器上，静态链接带来的部署优势远比更低的内存占用显得重要，所以很多编程语言包括 Go 都将静态链接作为默认的链接方式。

### 插件系统
在今天，动态链接带来的低内存占用优势虽然已经没有太多作用，但是动态链接的机制却可以为我们提供更多的灵活性，主程序可以在编译后动态加载共享库实现热插拔的插件系统。

![plugin-system.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/plugin-system.png)

图 8-2 插件系统

通过在主程序和共享库直接定义一系列的约定或者接口，我们可以通过以下的代码动态加载其他人编译的 Go 语言共享对象，这样做的好处是主程序和共享库的开发者不需要共享代码，只要双方的约定不变，修改共享库后也不需要重新编译主程序。

``` go
type Driver interface {
    Name() string
}

func main() {
    p, err := plugin.Open("driver.so")
    if err != nil {
	   panic(err)
    }

    newDriverSymbol, err := p.Lookup("NewDriver")
    if err != nil {
        panic(err)
    }

    newDriverFunc := newDriverSymbol.(func() Driver)
    newDriver := newDriverFunc()
    fmt.Println(newDriver.Name())
}
```
上述代码定义了 Driver 接口并认为共享库中一定包含 func NewDriver() Driver 函数，当我们通过 plugin.Open 读取包含 Go 语言插件的共享库后，获取文件中的 NewDriver 符号并转换成正确的函数类型，可以通过该函数初始化新的 Driver 并获取它的名字了。

### 操作系统
不同的操作系统会实现不同的动态链接机制和共享库格式，Linux 中的共享对象会使用 ELF 格式3并提供了一组操作动态链接器的接口，在本节的实现中我们会看到以下的几个接口4：
```c
void *dlopen(const char *filename, int flag);
char *dlerror(void);
void *dlsym(void *handle, const char *symbol);
int dlclose(void *handle);
```
dlopen 会根据传入的文件名加载对应的动态库并返回一个句柄（Handle）；我们可以直接使用 dlsym 函数在该句柄中搜索特定的符号，也就是函数或者变量，它会返回该符号被加载到内存中的地址。因为待查找的符号可能不存在于目标动态库中，所以在每次查找后我们都应该调用 dlerror 查看当前查找的结果。

## 8.1.2 动态库 
Go 语言插件系统的全部实现都包含在 plugin 中，这个包实现了符号系统的加载和决议。插件是一个带有公开函数和变量的包，我们需要使用下面的命令编译插件：

```bash
go build -buildmode=plugin ...
```
该命令会生成一个共享对象 .so 文件，当该文件被加载到 Go 语言程序时会使用下面的结构体 plugin.Plugin 表示，该结构体中包含文件的路径以及包含的符号等信息：
```go
type Plugin struct {
	pluginpath string
	syms       map[string]interface{}
	...
}
```
与插件系统相关的两个核心方法分别是用于加载共享文件的 plugin.Open 和在插件中查找符号的 plugin.Plugin.Lookup，本节将详细介绍它们的实现原理。

### CGO 
在具体分析 plugin 包中几个公有方法之前，我们需要先了解一下包中使用的两个 C 语言函数 plugin.pluginOpen 和 plugin.pluginLookup；plugin.pluginOpen 只是简单包装了一下标准库中的 dlopen 和 dlerror 函数并在加载成功后返回指向动态库的句柄：
```C
static uintptr_t pluginOpen(const char* path, char** err) {
	void* h = dlopen(path, RTLD_NOW|RTLD_GLOBAL);
	if (h == NULL) {
		*err = (char*)dlerror();
	}
	return (uintptr_t)h;
}
C
plugin.pluginLookup 使用了标准库中的 dlsym 和 dlerror 获取动态库句柄中的特定符号：

static void* pluginLookup(uintptr_t h, const char* name, char** err) {
	void* r = dlsym((void*)h, name);
	if (r == NULL) {
		*err = (char*)dlerror();
	}
	return r;
}
```
这两个函数的实现原理都比较简单，它们的作用也只是简单封装标准库中的 C 语言函数，让它们的签名看起来更像是 Go 语言中的函数签名，方便在 Go 语言中调用。

### 加载过程
用于加载共享对象的函数 plugin.Open 会将共享对象文件的路径作为参数并返回 plugin.Plugin 结构：
```go
func Open(path string) (*Plugin, error) {
	return open(path)
}
```
上述函数会调用私有的函数 plugin.open 加载插件，它是插件加载过程的核心函数，我们可以将该函数拆分成以下几个步骤：

* 准备 C 语言函数 plugin.pluginOpen 的参数；
* 通过 cgo 调用 plugin.pluginOpen 并初始化加载的模块；
* 查找加载模块中的 init 函数并调用该函数；
* 通过插件的文件名和符号列表构建 plugin.Plugin 结构；
首先是使用 cgo 提供的一些结构准备调用 plugin.pluginOpen 所需要的参数，下面的代码会将文件名转换成 *C.char 类型的变量，该类型的变量可以作为参数传入 C 函数：
``` go
func open(name string) (*Plugin, error) {
	cPath := make([]byte, C.PATH_MAX+1)
	cRelName := make([]byte, len(name)+1)
	copy(cRelName, name)
	if C.realpath(
		(*C.char)(unsafe.Pointer(&cRelName[0])),
		(*C.char)(unsafe.Pointer(&cPath[0]))) == nil {
		return nil, errors.New(`plugin.Open("` + name + `"): realpath failed`)
	}

	filepath := C.GoString((*C.char)(unsafe.Pointer(&cPath[0])))

	...
	var cErr *C.char
	h := C.pluginOpen((*C.char)(unsafe.Pointer(&cPath[0])), &cErr)
	if h == 0 {
		return nil, errors.New(`plugin.Open("` + name + `"): ` + C.GoString(cErr))
	}
	...
}
```
当我们拿到了指向动态库的句柄之后会调用 plugin.lastmoduleinit，链接器会将它会链接到运行时的 runtime.plugin_lastmoduleinit 函数上，它会解析文件中的符号并返回共享文件的目录和其中包含的全部符号：
``` go
func open(name string) (*Plugin, error) {
	...
	pluginpath, syms, errstr := lastmoduleinit()
	if errstr != "" {
		plugins[filepath] = &Plugin{
			pluginpath: pluginpath,
			err:        errstr,
		}
		pluginsMu.Unlock()
		return nil, errors.New(`plugin.Open("` + name + `"): ` + errstr)
	}
	...
}
```
在该函数的最后，我们会构建一个新的 plugin.Plugin 结构体并遍历 plugin.lastmoduleinit 返回的全部符号，为每一个符号调用 plugin.pluginLookup：
``` go
func open(name string) (*Plugin, error) {
	...
	p := &Plugin{
		pluginpath: pluginpath,
	}
	plugins[filepath] = p
	...
	updatedSyms := map[string]interface{}{}
	for symName, sym := range syms {
		isFunc := symName[0] == '.'
		if isFunc {
			delete(syms, symName)
			symName = symName[1:]
		}

		fullName := pluginpath + "." + symName
		cname := make([]byte, len(fullName)+1)
		copy(cname, fullName)

		p := C.pluginLookup(h, (*C.char)(unsafe.Pointer(&cname[0])), &cErr)
		valp := (*[2]unsafe.Pointer)(unsafe.Pointer(&sym))
		if isFunc {
			(*valp)[1] = unsafe.Pointer(&p)
		} else {
			(*valp)[1] = p
		}
		updatedSyms[symName] = sym
	}
	p.syms = updatedSyms
	return p, nil
}
```
上述函数在最后会返回一个包含符号名到函数或者变量映射的 plugin.Plugin 结构体，调用方可以将该结构体作为句柄查找其中的符号，需要注意的是，我们在这段代码中省略了查找 init 并初始化插件的过程。

### 符号查找
plugin.Plugin.Lookup 可以在 plugin.Open 返回的结构体中查找符号 plugin.Symbol，该符号是 interface{} 类型的一个别名，我们可以将它转换成变量或者函数真实的类型：

``` go
func (p *Plugin) Lookup(symName string) (Symbol, error) {
	return lookup(p, symName)
}

func lookup(p *Plugin, symName string) (Symbol, error) {
	if s := p.syms[symName]; s != nil {
		return s, nil
	}
	return nil, errors.New("plugin: symbol " + symName + " not found in plugin " + p.pluginpath)
}
```
上述方法调用的私有函数 plugin.lookup 实现比较简单，它直接利用了结构体中的符号表，如果没有找到对应的符号会直接返回错误。

## 8.1.3 小结
Go 语言的插件系统利用了操作系统的动态库实现模块化的设计，它提供功能虽然比较有趣，但是在实际使用中会遇到比较多的限制，目前的插件系统也仅支持 Linux、Darwin 和 FreeBSD，在 Windows 上是没有办法使用的。因为插件系统的实现基于一些黑魔法，所以跨平台的编译也会遇到一些比较奇葩的问题，作者在使用插件系统时也踩过很多坑，如果对 Go 语言不是特别了解，还是不建议使用该模块的。

## 8.1.4 推荐阅读
Static Libraries vs. Dynamic Libraries https://medium.com/@StueyGK/static-libraries-vs-dynamic-libraries-af78f0b5f1e4