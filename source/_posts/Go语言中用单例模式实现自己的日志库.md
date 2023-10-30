---
title: Go语言中用单例模式实现自己的日志库
tags:
  - Go
categories: Code
date: 2022-12-06 15:07:19
---




Go语言自带的日志库不能调整日志输出级别，其他日志库需要通过实例的地址传递实现配置的一致性。对于一个日志组件来说，我们最好全局维护同一个实例。这样刚好满足单例模式。单例模式是常用的模式之一，在它的核心结构中只包含一个被称为单例的特殊类，能够保证系统运行中一个类只创建一个实例。我们就用单例模式来封装一个日志库。

<!-- more -->

代码地址：https://github.com/ShadowFl0w/logger



## 代码解释

定义日志级别

```go
const (
	LFATAL = 1 << iota
	LSilent
	LPrintln
	LERROR
	LINFO
	LWARNING
	LDEBUG
	LVerbose
)
```



初始化方法

```Go
func NewLogger(level int32) *Logger {
	w := os.Stderr
	return &Logger{
		w:             w,
		level:         level,
		fatalLogger:   log.New(w, "[FATAL]", log.Ldate|log.Ltime|log.Llongfile),
		silentLogger:  log.New(w, "", 0),
		printlnLogger: log.New(w, "", 0),
		errorLogger:   log.New(w, "[ERROR]", log.Ldate|log.Ltime|log.Llongfile),
		infoLogger:    log.New(w, "[INFO]", log.Ldate|log.Ltime|log.Lshortfile),
		warningLogger: log.New(w, "[WARNING]", log.Ldate|log.Ltime|log.Llongfile),
		debugLogger:   log.New(w, "[DEBUG]", log.Ldate|log.Ltime|log.Llongfile),
		verboseLogger: log.New(w, "[VERBOSE]", log.Ldate|log.Ltime|log.Llongfile),
	}
}
```



```go
var stdLogger = NewLogger(LVerbose)
```



定义一个私有变量，将实例给这个私有变量，实现单例。



通过SetLevel方法修改日志级别

```go
func (logger *Logger) SetLevel(level int32) {
	logger.mu.Lock()
	defer logger.mu.Unlock()
	logger.level = level
}

func SetLevel(level int32) {
	if level < LFATAL || level > LVerbose {
		return
	}

	stdLogger.SetLevel(level)
}

```

我们可以看到两个SetLevel方法，当我们外部调用SetLevel方法时，实际调用的是`stdLogger.SetLevel(level)`，stdLogger是我们的单例，通过`SetLevel`方法修改单例里的level属性。

当我们调用方法的时候就会根据单例里面的级别，输出对应的日志。

```go
func (logger *Logger) SetLevel(level int32) {
	logger.mu.Lock()
	defer logger.mu.Unlock()
	logger.level = level
}

func SetLevel(level int32) {
	if level < LFATAL || level > LVerbose {
		return
	}

	stdLogger.SetLevel(level)
}

func Info(arg ...interface{}) {
	if atomic.LoadInt32(&stdLogger.level) < LINFO {
		return
	}
	stdLogger.infoLogger.Output(2, fmt.Sprintln(arg...))
}

func Error(err error) {
	if atomic.LoadInt32(&stdLogger.level) < LERROR {
		return
	}
	stdLogger.errorLogger.Output(2, fmt.Sprintln(err))
}

func Println(arg ...interface{}) {
	if atomic.LoadInt32(&stdLogger.level) < LPrintln {
		return
	}
	stdLogger.printlnLogger.Output(2, fmt.Sprintln(arg...))
}

func Silent(arg ...interface{}) {
	if atomic.LoadInt32(&stdLogger.level) < LSilent {
		return
	}
	stdLogger.silentLogger.Output(2, fmt.Sprintln(arg...))
}
```



## 使用方法

### 输出不同格式的日志

```go
package test

import (
	"testing"

	"github.com/ShadowFl0w/logger"
)

func TestLogger(t *testing.T) {
	logger.Info("Info here!")
	logger.Println("Println here!")
}

```

输出

```
[INFO]2022/12/06 14:46:46 logger_test.go:10: Info here!
Println here!
```

可以看见，`logger.Info`输出了日期、输出位置。`logger.Println`则直接输出结果



### 调整日志输出级别

添加`	logger.SetLevel(logger.LPrintln)`

```
package test

import (
	"testing"

	"github.com/ShadowFl0w/logger"
)

func TestLogger(t *testing.T) {
	logger.Info("Info here!")
	logger.Println("Println here!")

	logger.SetLevel(logger.LPrintln)

	logger.Info("Info Level here!")
	logger.Println("Println Level here!")
}

```

我们查看结果

```
[shadowflow@ShadowOS test]$ go test
[INFO]2022/12/06 15:01:30 logger_test.go:10: Info here!
Println here!
Println Level here!
```

发现`logger.Info("Info Level here!")` 没有输出。成功调整了日志级别为`LPrintln`







