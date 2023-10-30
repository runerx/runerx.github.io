---
title: Go语言中用工厂模式自动注册实现扫描器插件化
tags:
  - Go
categories: Code
date: 2022-12-06 10:43:13
---




Go要实现插件化有几种方法，官方方法是编译成so文件，这太麻烦了；还有的是用一个Go解释器，这对于我们太重了；还有的方法是嵌入`go run`语句，这太不优雅了。最终我们选择用工厂模式动态创建对象，用map容器来保存对象，实现插件化。

<!-- more -->

工厂方法模式（英语：Factory method pattern）是一种实现了“工厂”概念的面向对象设计模式。就像其他创建型模式一样，它也是处理在不指定对象具体类型的情况下创建对象的问题。工厂方法模式的实质是“定义一个创建对象的接口，但让实现这个接口的类来决定实例化哪个类。工厂方法让类的实例化推迟到子类中进行。





先定义一个叫Plugins的map，用来存储类。

```go
var Plugins map[string]Plugin
```



Info用来保存插件信息

```go
// Plugin Info
type Info struct {
	Name string
	Desc string
}
```



定义一个叫接Plugins口的

```go
// Plugin 插件接口
type Plugin interface {
	New() Info
	Scan(sender Sender)
}
```

定义一个叫Regist用来注册插件的方法

```go
// Regist 注册插件
func Regist(target string, plugin Plugin) {
	Plugins[target] = plugin
}
```







根据Plugin接口实现两个插件

- log4j

```go
package passive

import (
	"ShadowScan/plugin"
	"log"
)

type log4j struct {
	info plugin.Info
}

func init() {
	plugin.Regist("log4j", &log4j{})
}
func (p *log4j) New() plugin.Info {
	p.info = plugin.Info{
		Name: "log4j",
		Desc: "log4j scan",
	}
	return p.info
}

func (p *log4j) Scan(sender plugin.Sender) {
	log.Println("log4jscan...")

}

```

- sqli

```go
package passive

import (
	"ShadowScan/plugin"
	"log"
)

type sqli struct {
	info plugin.Info
}

func init() {
	plugin.Regist("sqli", &sqli{})
}
func (p *sqli) New() plugin.Info {
	p.info = plugin.Info{
		Name: "sqli",
		Desc: "sqli scan",
	}
	return p.info
}

func (p *sqli) Scan(sender plugin.Sender) {
	log.Println("sqli scan...")

}

```


然后我们用匿名包的方式，在main函数执行前执行这两个init方法，我们只需要在main文件中引入匿名包即可

```go
	_ "ShadowScan/plugin/passive"
```

由于我们在注册插件之前需要先初始化Plugins用来存储插件对象，所以我们在引入匿名包之前在main文件中提前引入Plugins所在包

```go
o	"ShadowScan/plugin"
	_ "ShadowScan/plugin/passive"
```



Plugins初始化方法

```go
func init() {
	Plugins = make(map[string]Plugin)
}
```



最后调用插件

```go
func PluginScan(req *http.Request) {
	// Plugins = make(map[string]Plugin)
	sender, _ := NewSender(req)

	for _, plugin := range Plugins {
		plugin.Scan(sender)
	}
}
```

