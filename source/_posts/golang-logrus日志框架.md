---
title: golang logrus日志库使用小记
tags: ["golang", "logrus"]
categories: ["golang", "日志"]
icon: fa-handshake-o
---

golang自带日志框架功能比较简单，仅提供print、panic、fatal三个方法，对于常规的日志切割等功能并未提供支持。
这里推荐使用[sirupsen/logrus](https://github.com/sirupsen/logrus)日志库。

# logrus feature
* 结构化、可插拔的日志模块
* 完全兼容官方log库接口
* Field机制
* 可扩展的HOOK机制
* TEXT与JSON两种可选格式

# 简单使用示例

## std Logger
与官方log类似，logrus也提供了一个名为std的标准logger，对外导出的各类方法直接使用std记录日志，一般可按如下方式使用

    package main

    import log "github.com/sirupsen/logrus"

    func main() {

        log.Info("hello, world.")
    }


## 自建Logger实例

    package main

    import (
      "os"
      "github.com/sirupsen/logrus"
    )

    // Create a new instance of the logger. You can have any number of instances.
    var log = logrus.New()

    func main() {
      // The API for setting attributes is a little different than the package level
      // exported logger. See Godoc.
      log.Out = os.Stdout

      // You could set this to any `io.Writer` such as a file
      // file, err := os.OpenFile("logrus.log", os.O_CREATE|os.O_WRONLY, 0666)
      // if err == nil {
      //  log.Out = file
      // } else {
      //  log.Info("Failed to log to file, using default stderr")
      // }

      log.WithFields(logrus.Fields{
        "animal": "walrus",
        "size":   10,
      }).Info("A group of walrus emerges from the ocean")
    }

# 输出到本地文件系统
下面示例代码，通过一个hook将日志输出到本地文件系统中，并提供日志切割功能

    import (
        "github.com/lestrrat/go-file-rotatelogs"
        "github.com/rifflock/lfshook"
        log "github.com/sirupsen/logrus"
        "time"
        "os"
        "github.com/pkg/errors"
        "path"
    )
    // config logrus log to local filesystem, with file rotation
    func ConfigLocalFilesystemLogger(logPath string, logFileName string, maxAge time.Duration, rotationTime time.Duration) {
        baseLogPaht := path.Join(logPath, logFileName)
        writer, err := rotatelogs.New(
            baseLogPaht+".%Y%m%d%H%M",
            rotatelogs.WithLinkName(baseLogPaht), // 生成软链，指向最新日志文件
            rotatelogs.WithMaxAge(maxAge), // 文件最大保存时间
            rotatelogs.WithRotationTime(rotationTime), // 日志切割时间间隔
        )
        if err != nil {
            log.Errorf("config local file system logger error. %+v", errors.WithStack(err))
        }
        lfHook := lfshook.NewHook(lfshook.WriterMap{
            log.DebugLevel: writer, // 为不同级别设置不同的输出目的
            log.InfoLevel:  writer,
            log.WarnLevel:  writer,
            log.ErrorLevel: writer,
            log.FatalLevel: writer,
            log.PanicLevel: writer,
        })
        log.AddHook(lfHook)
    }

# 输出到MQ或ES
如下示例代码通过hook将日志输出到amqp消息队列，或者es中。

    import (
        "github.com/vladoatanasov/logrus_amqp"
        "gopkg.in/olivere/elastic.v5"
        "gopkg.in/sohlich/elogrus.v2"
        log "github.com/sirupsen/logrus"
        "github.com/pkg/errors"
    )

    // config logrus log to amqp
    func ConfigAmqpLogger(server, username, password, exchange, exchangeType, virtualHost, routingKey string) {
        hook := logrus_amqp.NewAMQPHookWithType(server, username, password, exchange, exchangeType, virtualHost, routingKey)
        log.AddHook(hook)
    }

    // config logrus log to es
    func ConfigESLogger(esUrl string, esHOst string, index string) {
        client, err := elastic.NewClient(elastic.SetURL(esUrl))
        if err != nil {
            log.Errorf("config es logger error. %+v", errors.WithStack(err))
        }
        esHook, err := elogrus.NewElasticHook(client, esHOst, log.DebugLevel, index)
        if err != nil {
            log.Errorf("config es logger error. %+v", errors.WithStack(err))
        }
        log.AddHook(esHook)
    }
