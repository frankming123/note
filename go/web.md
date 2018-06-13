# go web

<!-- TOC -->

- [go web](#go-web)
    - [web入门](#web入门)
    - [beego](#beego)
        - [beego简单示例](#beego简单示例)
        - [bee工具使用](#bee工具使用)
        - [beego中的模板](#beego中的模板)
            - [模板目录](#模板目录)
            - [模板标签](#模板标签)
            - [模板数据](#模板数据)
            - [模板条件](#模板条件)
            - [模板嵌套](#模板嵌套)
            - [模板函数](#模板函数)
            - [模板名称](#模板名称)

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

### beego中的模板

beego的模板处理引擎采用的是Go内置的html/template包进行处理

#### 模板目录

beego中,模板目录是views,用户可以将目录放到该目录下

#### 模板标签

Go语言的默认模板采用了{{和}}作为左右标签

可以修改beego.TemplateLeft和beego.TemplateRight字段来改变标签

    beego.TemplateLeft = "<<<"
    beego.TemplateRight = ">>>"

#### 模板数据

模板中的数据是通过在Controller中this.Data获取的

例如获取模板的内容{{.Content}}:

    this.Data["Content"] = "value"

使用各种类型的数据渲染:

- 结构体

    结构体结构:

        type A struct{
            Name string
            Age  int
        }

    控制器数据赋值:

        this.Data["a"]=&A{Name:"astaxie",Age:25}

    模板渲染数据:

        the username is {{.a.Name}}
        the age is {{.a.Age}}

- map

    控制器数据赋值:

        mp["name"]="astaxie"
        mp["nickname"] = "haha"
        this.Data["m"]=mp

    模板渲染数据:

        the username is {{.m.name}}
        the username is {{.m.nickname}}

- slice

    控制器数据赋值:

        ss :=[]string{"a","b","c"}
        this.Data["s"]=ss

    模板渲染数据:

        {{range $key, $val := .s}}
        {{$key}}
        {{$val}}
        {{end}}

#### 模板条件

有如下模板条件,用于为模板数据设置条件

- if_else

    模板渲染:

        {{if .TrueCond}}
        true condition.
        {{else}}
        false condition
        {{end}}

    其中else块可以省略

- with

    如果多个数据都是同一个map或结构体的字段,那么可以将它们归类于with语句中

    模板渲染:

        {{with .User}}
        {{.Name}};{{.Age}};{{.Sex}}
        {{end}}

- range

    range可以迭代数组或切片的每个元素

    模板渲染:

        {{range $i := .Nums}}
        {{$i}}
        {{end}}

#### 模板嵌套

define语句可以将一段html代码嵌套进来,并同时在多处使用

示例:

    {{define "test"}}
    <div>
        this is test template
    </div>
    {{end}}

使用嵌套模板:

    {{template "test"}}

#### 模板函数

模板函数即可在模板中使用的函数

常用的模板函数:

- str2html

    可以将html文档格式的模板数据解析为html

    示例:

        c.Data["html"]="<p>it's a html document</p>"

    当在模板中使用{{str2html .html}}时,即可将其解析为it's a html document

#### 模板名称

用户通过在Controller的对应方法中设置相应的模板名称,beego会自动的在viewpath目录下查询该文件并渲染

    this.TplName = "admin/add.tpl"