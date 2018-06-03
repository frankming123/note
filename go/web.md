# go web

<!-- TOC -->

- [go web](#go-web)
    - [web入门](#web入门)
    - [beego](#beego)
        - [beego简单示例](#beego简单示例)

<!-- /TOC -->

## web入门

go中的web开发基于net/http框架

简单的web程序:

    package main

    import (
        "io"
        "log"
        "net/http"
    )

    func main() {
        //设置路由
        http.HandleFunc("/", sayHello)

        err := http.ListenAndServe(":80", nil)
        if err != nil {
            log.Fatal(err)
        }
    }

    func sayHello(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "hello world, this is version 1")
    }


## beego

beego是一个快速开发Go应用的HTTP框架,他可以用来快速开发API,Web及后端服务等各种应用,是一个RESTful的框架,主要设计灵感来源于tornado,sinatra和flask这三个框架,但是结合了Go本身的一些特性(interface,struct嵌入等)而设计的一个框架

beego官网:<https://beego.me/>

beego安装:

    ]# go get github.com/astaxie/beego

### beego简单示例

示例程序:

    package main

    import (
        "github.com/astaxie/beego"
    )

    type HomeController struct {
        beego.Controller
    }

    func (this *HomeController) Get() {
        this.Ctx.WriteString("hello world")
    }

    func main() {
        beego.Router("/", &HomeController{})
        beego.Run()
    }

打开浏览器访问localhost:8080,可以得到"hello world"