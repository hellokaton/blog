---
layout: post
title: 用 Golang 来编写 cli 程序吧，Happy~
cover: /public/images/cover/use-golang-write-cli-programs.jpg
tags: ['golang', '命令行']
---

## 背景

我是个 Java 开发者，做过非常多开源软件，经常会有在终端下提供命令行帮助程序的这种小需求，一般大家实现这个需求也就这么几种办法。

1. 编写批处理或者 Shell（Windows 和 Linux需要写两次）
2. 使用编程语言解决（golang、python都是不错的跨平台选择）

程序员都是懒人，我才不要写两次呢~ 很早之前也用Python写过类似的程序，但是打包出来的结果比较大，另一方面 Go 语言越来越火，也是我比较喜欢的一门编程语言，而且支持跨平台，所以就选用它了。

在这篇小文中我将教你编写一个可以查看本地天气的小程序，比较简单，你可以通过学习这篇文章做出自己中意的小工具。

<!-- more -->

完整的代码可以在我的 [Github](https://github.com/biezhi/weather-cli){:target="_blank"} 查看。

## 安装环境

我们在开始之前先要准备 Go 语言的环境，如果你已经安装过了这步可以略过。你可以在 [这里](https://golang.org/dl/){:target="_blank"} 下载到最新版的 Go 语言版本，如果你的网络环境被迫是下面这个样子。

![timeout]({{ "/public/images/2017/11/p3f4ktao8kjmaro78unjugogs5.png" | prepend: site.cdnurl }} "timeout")

你可以在 [Golang中国](https://golangtc.com/download){:target="_blank"} 下载最新发布包。Go 语言环境的安装方式也有好几种，我们选择最简单的方式：**标准包安装**。

Mac 环境下下载以 `.pkg` 结尾的包文件，在 Windows 下下载 `.msi` 结尾的文件，下载好后傻瓜式安装即可。

> 注意你的操作系统架构不要选错，Linux源码安装这里不讲啦。

### 配置环境变量

学习很多编程语言都需要配置环境变量，安装软件的时候其实也有部分程序静默的帮我们做了这件事，在前面我们安装了 Go 语言，下面我们了解下 Go 语言中的环境变量以及如何配置。

- `GOROOT`：Go 的安装路径
- `GOPATH`：告诉 Go 命令和其他相关工具，在那里去找到安装在你系统上的 Go 包。

那么我们创建一个工作目录来存储自己编写的源码包吧~

在 Windows 下假设是 `D:/go` 这个目录，Linux/MacOSX 下假设是 `~/workspace/golang` 这个目录。我目前是 Mac 系统，就按照这个设置环境变量了。

1. 加入环境变量 `export GOROOT=/usr/local/go`
2. 加入环境变量 `export GOPATH=/Users/biezhi/workspace/golang`
3. 修改系统环境变量 `export PATH=$PATH:$GOPATH/bin`

测试一下

```bash
# biezhi in ~
» go version
go version go1.8.3 darwin/amd64
```

大功告成，接下来的内容需要你具备一种编程语言的基础，否则无法食用。

## 和Java语言的一些区别

这里我们说几个不同之处，无法涵盖到所有，满足本文的需求。

**声明变量、常量**

Java 中

```java
private String name = "biezhi";
public static final String VERSION = "0.2.1";
```

Golang 中

```golang
name := "biezhi"
const version = "0.2.1"
```

矮油，很简洁哦~

**类和对象**

Java 中

```java
public class Config {
  private String key;
  private String value;
  // getter and setter
}

// 使用
Config config = new Config();
config.setKey("name");
config.setValue("biezhi");
```

Golang 中

```shell
type Config struct {
    key string
    value string
}

// 使用
conf := Config{key: "name", value: "biezhi"}
```

矮油，又 tm 简洁了。。。

> golang中没有class关键字，却引入了type，golang中更强调类型。

**函数返回值**

Java 中

```java
public String getFileName(){
  return "不可描述.jpg";
}
```

这里如果需要返回多个值，需要用类或者 Map 类型替换。

Golang 中

```shell
func getFileNmae() (string, error)  {
  return "可以描述.jpg", nil
}
```

Go 语言天生支持多返回值（毕竟后起之秀，社会社会）

Java 中关闭流的操作一般会写这样的代码

```java
try {
  in.balabala~
} catch(Exception e) {
  // 处理异常
} finally {
  in.close();
}
```

在 Go 中没有 `finally`，试试 `defer`

```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}
```

`CopyFile` 方法简单的实现了文件内容的拷贝功能，将源文件的内容拷贝到目标文件。乍一看还没什么问题，不过Golang 中的资源也是需要释放的，假如 `os.Create` 方法的调用出了错误，下面的语句会直接 `return`，导致这两个打开的文件没有机会被释放。这个时候，`defer` 就可以派上用场了。

前面 `BB` 了这么多只是简单的让大家在脑海中过一下Go的代码都长什么样。下面开始我们的正式话题了，如果你对基础还是不了解可以去看基础部分的书籍、资料（找不到私信我）。

## 原生写法实现

我不会一上来就教你如何使用某个库，这很不负责，你应该清楚在没有库的时代人们是如何做的，当有了更方便的工具为我减轻了什么，这个过程中你可能会了解到自己没见过的API如何使用，长此以往才会在编程中做到灵活运用。

这里需要了解几个基础包：

- `fmt`：格式化包，实现了类似C语言printf和scanf的格式化I/O
- `encoding/json`：原生JSON解析包
- `net/http`：发送http请求
- `flag`：供了一系列解析命令行参数的功能接口
- `io/ioutil`：IO处理

有这几个包后我们编写一个 `Hello World` 瞧瞧，我的项目名是 `weather-cli`

通过 `flag.XxxVar()` 方法将flag绑定到一个变量，该种方式返回值类型，如

```shell
package main

import (
    "fmt"
    "os"
)

func main() {
    var city string

    flag.StringVar(&city, "c", "上海", "城市中文名")
    flag.Parse()

    fmt.Println("城市是:", city)
}
```

运行一下试试

```bash
# biezhi in ~/workspace/golang/src/github.com/biezhi/weather-cli
» go build && ./weather-cli
城市是: 上海

» go build && ./weather-cli -c 北京
城市是: 北京
```

解析参数是比较简单的，在这个演示中我们加入两个参数，第一个是城市，第二个是显示哪天，具体代码如下：

```shell
func main() {
    var city string
    var day string

    flag.StringVar(&city, "c", "上海", "城市中文名")
    flag.StringVar(&day, "d", "今天", "可选: 今天, 昨天, 预测")
    flag.Parse()
}
```

此时已经可以获取到终端输入的参数了，那么接下来该找个接口调用天气 API 了，我找了 [这个](http://www.sojson.com/blog/234.html){:target="_blank"} 免费的API接口进行调用。我们需要编写一个方法用于HTTP请求。

```shell
func Request(url string) (string, error) {
    response, err := http.Get(url)
    if err != nil {
        return "", err
    }
    defer response.Body.Close()
    body, _ := ioutil.ReadAll(response.Body)
    return string(body), nil
}
```

果然比Java方便啊，这个函数很简单，输入一个 URL，返回响应的 Body 为字符串。我们将输入的城市传递进去即可，默认是 `上海`。

```shell
var city string
var day string

flag.StringVar(&city, "c", "上海", "城市中文名")
flag.StringVar(&day, "d", "今天", "可选: 今天, 昨天, 预测")
flag.Parse()

var body, err = Request(apiUrl + city)
if err != nil {
    fmt.Printf("err was %v", err)
    return
}
```

然后我们需要定义一些 `类型` 来存储JSON，在Java语言中是Class。这个类型结构是怎样的根据API的返回结果来定义，我将它们单独写在 `types.go` 文件中。

```shell
// 响应
type Response struct {
    Status   int    `json:"status"`
    CityName string `json:"city"`
    Data     Data   `json:"data"`
    Date     string `json:"date"`
    Message  string `json:"message"`
    Count    int    `json:"count"`
}

// 响应数据
type Data struct {
    ShiDu     string `json:"shidu"`
    Quality   string `json:"quality"`
    Ganmao    string `json:"ganmao"`
    Yesterday Day    `json:"yesterday"`
    Forecast  []Day  `json:"forecast"`
}

// 某一天的数据
type Day struct {
    Date    string  `json:"date"`
    Sunrise string  `json:"sunrise"`
    High    string  `json:"high"`
    Low     string  `json:"low"`
    Sunset  string  `json:"sunset"`
    Aqi     float32 `json:"aqi"`
    Fx      string  `json:"fx"`
    Fl      string  `json:"fl"`
    Type    string  `json:"type"`
    Notice  string  `json:"notice"`
}
```

类型定义好后就可以把HTTP请求得到的 JSON 解析为定义好的类型了。

```shell
var r Response
err = json.Unmarshal([]byte(body), &r)
if err != nil {
    fmt.Printf("\nError message: %v", err)
}
if r.Status != 200 {
    fmt.Printf("获取天气API出现错误, %s", r.Message)
    return
}
```

这里使用了 Go 自带的JSON解析（哎，我大 Java 咋没有呢。。），最后我们将得到的数据输出出来就 Ok 了。

```shell
func Print(day string, r Response) {
    fmt.Println("城市:", r.CityName)
    if day == "今天" {
        fmt.Println("湿度:", r.Data.ShiDu)
        fmt.Println("空气质量:", r.Data.Quality)
        fmt.Println("温馨提示:", r.Data.Ganmao)
    } else if day == "昨天" {
        fmt.Println("日期:", r.Data.Yesterday.Date)
        fmt.Println("温度:", r.Data.Yesterday.Low, r.Data.Yesterday.High)
        fmt.Println("风量:", r.Data.Yesterday.Fx, r.Data.Yesterday.Fl)
        fmt.Println("天气:", r.Data.Yesterday.Type)
        fmt.Println("温馨提示:", r.Data.Yesterday.Notice)
    } else if day == "预测" {
        fmt.Println("====================================")
        for _, item := range r.Data.Forecast {
            fmt.Println("日期:", item.Date)
            fmt.Println("温度:", item.Low, item.High)
            fmt.Println("风量:", item.Fx, item.Fl)
            fmt.Println("天气:", item.Type)
            fmt.Println("温馨提示:", item.Notice)
            fmt.Println("====================================")
        }
    } else {
        fmt.Println("大熊你是想刁难我胖虎吗?_?")
    }

}
```

此时这个小玩意已经可以运行了，我们来试试吧

```bash
# biezhi in ~/workspace/golang/src/github.com/biezhi/weather-cli
» go build && ./weather-cli
城市: 上海
湿度: 72%
空气质量: 良
温馨提示: 极少数敏感人群应减少户外活动

» ./weather-cli -c 北京
城市: 北京
湿度: 78%
空气质量: 轻度污染
温馨提示: 儿童、老年人及心脏、呼吸系统疾病患者人群应减少长时间或高强度户外锻炼
```

## 使用第三方库实现

这里我们用一款业界流行的库 [cli](https://github.com/urfave/cli){:target="_blank"}，这个家伙怎么使用呢？创建一个 `cli_main.go` 文件

```shell
package main

import (
  "fmt"
  "os"

  "github.com/urfave/cli"
)

func main() {
  app := cli.NewApp()
  app.Name = "greet"
  app.Usage = "fight the loneliness!"
  app.Action = func(c *cli.Context) error {
    fmt.Println("Hello friend!")
    return nil
  }

  app.Run(os.Args)
}
```

这是官网给出的一个例子，运行一下试试

```bash
» go build cli_main.go && ./cli_main
Hello friend!
```

实现我们上面的小程序需要用到 `flag` 这个功能，通过库实现的代码如下

```shell

func main() {
    app := cli.NewApp()
    app.Name = "weather-cli"
    app.Usage = "天气预报小程序"

    app.Flags = []cli.Flag{
        cli.StringFlag{
            Name:  "city, c",
            Value: "上海",
            Usage: "城市中文名",
        },
        cli.StringFlag{
            Name:  "day, d",
            Value: "今天",
            Usage: "可选: 今天, 昨天, 预测",
        },
    }

    app.Action = func(c *cli.Context) error {
        city := c.String("city")
        day := c.String("day")

        var body, err = Request(apiUrl + city)
        if err != nil {
            fmt.Printf("err was %v", err)
            return nil
        }

        var r Response
        err = json.Unmarshal([]byte(body), &r)
        if err != nil {
            fmt.Printf("\nError message: %v", err)
            return nil
        }
        if r.Status != 200 {
            fmt.Printf("获取天气API出现错误, %s", r.Message)
            return nil
        }
        Print(day, r)
        return nil
    }
    app.Run(os.Args)
}
```

我直接给出了全部代码，就 `40` 多行完成了~ 运行一下

```bash
» go build -o weather-cli utils.go types.go cli_main.go && ./weather-cli
城市: 上海
湿度: 72%
空气质量: 良
温馨提示: 极少数敏感人群应减少户外活动

» go build -o weather-cli utils.go types.go cli_main.go && ./weather-cli --city 北京
城市: 北京
湿度: 78%
空气质量: 轻度污染
温馨提示: 儿童、老年人及心脏、呼吸系统疾病患者人群应减少长时间或高强度户外锻炼
```

下面我们学习如何将这个小程序打包成二进制在各个平台下使用，以及如何压缩二进制包让它变得更小!

## 打包和压缩

打包为各个操作系统的程序

**Linux 64位**

```bash
GOOS=linux GOARCH=amd64 go build ...
```

**Windows 64位**

```bash
GOOS=windows GOARCH=amd64 go build ...
```

**MacOSX**

```bash
GOOS=darwin GOARCH=amd64 go build ...
```

如果你尝试打包后，生成的二进制文件大小大约是 `7.2M` 左右，这个体积有点大了，我们可以使用一些技术让它占用更小。

**首先加上编译参数 -ldflags**

```bash
go build -ldflags '-w -s' -o weather-cli utils.go types.go cli_main.go
```

执行后发现程序只有 `5.4M` 了，已经变小了，但是对于我们而言还是有点大，我就写几十行代码没必要生成这么大吧，下面我们使用另外一个神器 [upx](https://upx.github.io/)，如果你没安装可以在它的官网下载安装。

```bash
go build -ldflags '-w -s' -o weather-cli utils.go types.go cli_main.go && upx ./weather-cli
```

来看看

```bash
» ll -la
total 8.4M
drwxr-xr-x 12 biezhi staff  384 Nov  1 18:44 .
drwxr-xr-x 20 biezhi staff  640 Nov  1 16:45 ..
drwxr-xr-x 12 biezhi staff  384 Nov  1 18:44 .git
-rw-r--r--  1 biezhi staff 1.1K Sep  3 20:59 LICENSE
-rw-r--r--  1 biezhi staff  710 Nov  1 18:32 README.md
-rwxr-xr-x  1 biezhi staff 5.4M Nov  1 18:41 cli_main
-rw-r--r--  1 biezhi staff  905 Nov  1 18:18 cli_main.go
-rw-r--r--  1 biezhi staff  609 Nov  1 18:19 main.go
-rw-r--r--  1 biezhi staff  808 Nov  1 16:56 types.go
-rw-r--r--  1 biezhi staff 1.4K Nov  1 18:19 utils.go
-rwxr-xr-x  1 biezhi staff 2.0M Nov  1 18:44 weather-cli
```

只有 `2.0M` 了，这已经差不多了，各位司机学习快乐~ 想看更多有趣的开发姿势可关注我的专栏 [《王爵的技术小黑屋》](https://zhuanlan.zhihu.com/biezhi){:target="_blank"} 或者留言给我。