# go web

<!-- TOC -->

- [go web](#go-web)
    - [web入门](#web入门)
    - [beego](#beego)
        - [beego简单示例](#beego简单示例)
        - [bee工具使用](#bee工具使用)

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
        //监听地址,并传递处理函数,nil则为默认
        err := http.ListenAndServe(":80", nil)
        if err != nil {
            log.Fatal(err)
        }
    }

    func sayHello(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "hello world, this is version 1")
    }

可以自定义handler处理函数,而不用默认的httpServeMux

版本2:

    func main() {
        mux:=http.NewServeMux()
        mux.Handle("/",&myHandler{})
        mux.HandleFunc("/hello",sayHello)
        err:=http.ListenAndServe(":80", mux)
        if err != nil {
            log.Fatal(err)
        }
    }

    type myHandler struct{}

    func (*myHandler) ServeHTTP(w http.ResponseWriter,r *http.Request){
        io.WriteString(w,"URL:"+r.URL.String())
    }

    func sayHello(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "hello world, this is version 2")
    }

http.ListenAndServe本质上是在向http.Server结构体传值

版本3:使用map对路由进行统一管理

    package main

    import (
        "io"
        "log"
        "net/http"
        "time"
    )

    var mux map[string]func(http.ResponseWriter, *http.Request)

    func main() {
        server := http.Server{
            Addr:        ":80",
            Handler:     &myHandler{},
            ReadTimeout: 5 * time.Second,
        }

        mux = make(map[string]func(http.ResponseWriter, *http.Request))
        mux["/hello"] = sayHello
        mux["/bye"] = sayBye

        err := server.ListenAndServe()
        if err != nil {
            log.Fatal(err)
        }
    }

    type myHandler struct{}

    func (*myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        if h, ok := mux[r.URL.String()]; ok {
            h(w, r)
            return
        }
        io.WriteString(w, "URL: "+r.URL.String())
    }

    func sayHello(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "hello world, this is version 3")
    }
    func sayBye(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "Bye bye, this is version 3.")
    }

示例:一个简易的文件服务器

    package main

    import (
        "log"
        "net/http"
        "os"
    )

    func main() {
        mux := http.NewServeMux()

        wd, err := os.Getwd()
        if err != nil {
            log.Fatal(err)
        }
        mux.Handle("/",
            http.StripPrefix("/",
                http.FileServer(http.Dir(wd))))

        err = http.ListenAndServe(":80", mux)
        if err != nil {
            log.Fatal(err)
        }
    }


## beego

beego是一个快速开发Go应用的HTTP框架,他可以用来快速开发API,Web及后端服务等各种应用,是一个RESTful的框架,主要设计灵感来源于tornado,sinatra和flask这三个框架,但是结合了Go本身的一些特性(interface,struct嵌入等)而设计的一个框架

beego官网:<https://beego.me/>

beego安装:

    ]# go get github.com/astaxie/beego

bee安装(beego的工具):

    ]# go get github.com/beego/bee

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

### bee工具使用

准备web环境

    ]# bee new my_web_app

启动web程序

    ]# bee run my_web_app