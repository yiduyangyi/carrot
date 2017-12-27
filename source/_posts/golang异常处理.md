---
title: golang异常处理
tags: ["golang", "异常处理"]
categories: ["golang", "异常"]
icon: fa-handshake-o
---

# error与panic

error：可预见的错误

panic：不可预见的异常

# panic处理
通过panic，defer，recover来处理异常

如下示例代码，当一个http链接到来时，golang会调用serve函数，serve函数会解析http协议，然后交给上层的handler处理，如果上层的handler内抛出异常的话，会被defer里面的recover函数捕获，打印出当前的堆栈信息，这样就可以防止一个http请求发生异常导致整个程序崩溃了。

    func (c *conn)serve() {
        defer func() {
            if err := recover(); err != nil {
                const size = 64 << 10
                buf := make([]byte, size)
                buf = buf[:runtime.Stack(buf, false)]
                fmt.Printlf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
            }
        }()
        w := c.readRequest()
        ServeHTTP(w, w.req)
    }


# error处理
## 在defer中集中处理错误


    func demo() {
        var err error
        defer func() {
            if err != nil {
                //  处理发生的错误，打印日志等信息
                return
            }
        }
        err = func1()
        if err != nil {
            return
        }
        err = func2()
        if err != nil {
            return
        }
        .....
    }

## all-or-nothing check
如果一个完整的任务内有多个小的任务，我们没有必要执行完每个小的任务都检查下是否有错，我们可以在所有的小任务执行完成后再去判断整个任务的执行过程中有没有错误。

    //冗长的代码，满篇重复的 if err != nil
    _, err = fd.Write(p0[a:b])
    if err != nil {
        return err
    }
    _, err = fd.Write(p1[c:d])
    if err != nil {
        return err
    }
    _, err = fd.Write(p2[e:f])
    if err != nil {
        return err
    }
    // and so on

我们可以这样简化代码

    type errWriter struct {
        w   io.Writer
        err error
    }
    func (ew *errWriter) write(buf []byte) {
        if ew.err != nil {
            return
        }
        _, ew.err = ew.w.Write(buf)
    }
    ew := &errWriter{w: fd}
    ew.write(p0[a:b])
    ew.write(p1[c:d])
    ew.write(p2[e:f])
    // 只需要在最后检查一次
    if ew.err != nil {
        return ew.err
    }

这种处理方式有一个明显的问题是不知道整个任务是在哪一个小的任务执行的时候发生错误，如果我们需要关注每个小任务的进度则这种方式就不适合了。

## 错误上下文
error是一个接口，任何实现了Error()函数的类型都满足该接口，errors.New()实际上返回的是一个errorString类型，该类型内仅包含一个string字符串，没有错误发生的上下文信息，当我们把error不断向上抛，在上层做统一处理时，只能输出一个字符串信息而不知道错误是在什么地方什么情况下发生的。

    type error interface {
        Error() string
    }
    type errorString struct {
        s string
    }
    func (e *errorString) Error() string {
        return e.s
    }
    func New(text string) error {
        return &errorString{text}
    }

既然error是一个接口，那我们也可以自己实现一个满足该接口的类型，并记录下发生错误的文件，函数，堆栈等信息，更方便错误的查找。

    package stackerr
    import (
        "fmt"
        "runtime"
        "strings"
    )
    type StackErr struct {
        Filename      string
        CallingMethod string
        Line          int
        ErrorMessage  string
        StackTrace    string
    }
    func New(err interface{}) *StackErr {
        var errMessage string
        switch t := err.(type) {
        case *StackErr:
            return t
        case string:
            errMessage = t
        case error:
            errMessage = t.Error()
        default:
            errMessage = fmt.Sprintf("%v", t)
        }
        stackErr := &StackErr{}
        stackErr.ErrorMessage = errMessage
        _, file, line, ok := runtime.Caller(1)
        if ok {
            stackErr.Line = line
            components := strings.Split(file, "/")
            stackErr.Filename = components[(len(components) - 1)]
        }
        const size = 1 << 12
        buf := make([]byte, size)
        n := runtime.Stack(buf, false)
        stackErr.StackTrace = string(buf[:n])
        return stackErr
    }
    func (this *StackErr) Error() string {
        return this.ErrorMessage
    }
    func (this *StackErr) Stack() string {
        return fmt.Sprintf("{%s:%d} %s\nStack Info:\n %s", this.Filename, this.Line, this.ErrorMessage, this.StackTrace)
    }
    func (this *StackErr) Detail() string {
        return fmt.Sprintf("{%s:%d} %s", this.Filename, this.Line, this.ErrorMessage)
    }


## 第三方错误库推荐
[github.com/pkg/errors](https://github.com/pkg/errors)是一款提供了日志上下文封装的error库，可以非常方便的给error加上额外的信息、记录stack信息等等。详细使用说明可以参考github或者源码。
