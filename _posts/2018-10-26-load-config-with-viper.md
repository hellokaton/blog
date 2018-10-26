---
layout: post
title: 在 Go 中使用 Viper 加载配置文件
tags: ['golang']
---

使用任何编程语言开发工程化的项目都缺少不了配置，我们可能要存储一些数据库信息、邮件配置、其他的第三方服务密钥等，而配置文件的类型又有很多种，比如 `XML`、`JSON`、`YML`、`INI` 等，配置文件又可能分为不同的环境，如 `dev`、`test`、`prod`，这篇文章中带你了解在 Go 中加载配置信息那些事儿。

<!-- more -->

## 加载配置的方式

所有的程序一开始都是没有框架的，那我们怎么做呢？还好 Go 语言的标准库封装的足够优秀，标准库已经有了 `json` 解析的包，如果你只想解析 `yaml` 可以试试 [yaml](https://github.com/go-yaml/yaml){:target="_blank"} 这个库。

我们假设现在有一份 `json` 格式的配置文件 `config.json`

```json
{
  "port": 10666,
  "mysql": {
    "url": "(127.0.0.1:3306)/biezhi",
    "username": "root",
    "password": "123456"
  }
}
```

这个配置的结构有两个层级，第一层只有一个端口号，第二层是 MySQL 数据库的信息。在 Go 代码中可以这样读取

```go
import (
    "encoding/json"
    "fmt"
    "io/ioutil"
)

// 定义配置文件解析后的结构
type MySQLConfig struct {
    URL      string
    Username string
    Password string
}

type Config struct {
    Port  int
    MySQL MySQLConfig
}

func main() {
    var config Config

    // ReadFile 函数会读取文件的全部内容，返回一个字节数组
    data, err := ioutil.ReadFile("config.json")
    if err != nil {
        panic(err)
    }

    //读取的数据为json格式，需要进行解码
    err = json.Unmarshal(data, &config)
    if err != nil {
        panic(err)
    }
    fmt.Println(config)
}

// output
» {10666 {(127.0.0.1:3306)/biezhi root 123456}}
```

这样就可以读取到我们自定义的配置文件了，非常方便。那有人就问了，既然我可以这样读取为什么还用你说的这个 Viper 呢？

这个问题问的好。任何框架、产品出现必然有它的意义，我问你几个问题

**1. 如果现在有 10 个配置文件的结构都不同你怎么解决呢？**

_这还不好办，我把结构体改为 `map` 不就行了。_

**2. 我有好几个运行环境，这些配置的结构是类似的，名称或后缀不同，我想动态读取你怎么办？**

_简单，我根据你命令行传入的参数读取相应的配置不就行了。_

**3. 我想修改配置后可以自动刷新配置到内存中怎么办？**

_我在编写一个监听器自动监听这些文件的变动（小伙子说的挺简单的哈）_

**4. 由于某些原因把之前的所有 JSON 文件要换成 YAML 文件怎么办？**

_(#‵′)靠，你怎么这么多事儿。。。我不干了。_

----

哈哈，看来读取配置文件这件小事也不是那么简单，问题还挺多。其实这些需求在你编码能力足够的情况下其实都是可以解决的，只不过我们自己写一遍除非比别人的更有特色或优势，否则做的都是无用功。

## 为什么是 Viper？

先说说它这个名字，中国人乍一看马上想到尊贵的各种会员。Viper 的中文是毒蛇，因为作者之前编写过一个项目叫 [Cobra](https://github.com/spf13/cobra){:target="_blank"}（一个非常流行的命令行库），而 Cobra 的中文是眼镜蛇，他说 Viper 是 Cobra 的同伴，果然大神都很有趣。

其实读取配置文件的库也不少，我们来看看常用的一些。

- [yaml](https://github.com/go-yaml/yaml){:target="_blank"}：用于解析 `yaml` 文件的库
- [ini](https://github.com/go-ini/ini){:target="_blank"}：用于解析 `ini` 配置的库
- [store](https://github.com/tucnak/store){:target="_blank"}：用于解析 `toml` 配置的库
- [env](https://github.com/caarlos0/env){:target="_blank"}：用于解析环境变量的库
- [查看更多配置库](https://github.com/avelino/awesome-go#configuration){:target="_blank"}

这些库无疑都是优秀的，但它们更像是一些小工具，在 Github 上的 star 数也都较低。那么 Viper 是什么呢？

![viper]({{ "/public/images/2018/10/viper.png" | prepend: site.cdnurl }} "viper")

viper 的 Github 是：[https://github.com/spf13/viper](https://github.com/spf13/viper){:target="_blank"}

有非常多优秀的 Go 语言项目都使用了 Viper，如：

* [Hugo](http://gohugo.io){:target="_blank"}
* [EMC RexRay](http://rexray.readthedocs.org/en/stable/){:target="_blank"}
* [Imgur’s Incus](https://github.com/Imgur/incus){:target="_blank"}
* [Nanobox](https://github.com/nanobox-io/nanobox)/[Nanopack](https://github.com/nanopack){:target="_blank"}
* [Docker Notary](https://github.com/docker/Notary){:target="_blank"}
* [BloomApi](https://www.bloomapi.com/){:target="_blank"}
* [doctl](https://github.com/digitalocean/doctl){:target="_blank"}
* [Clairctl](https://github.com/jgsqware/clairctl){:target="_blank"}

它解决了什么问题呢？或者说它比这些工具库有什么优势？

- 为各种配置项设置默认值
- 加载并解析JSON、TOML、YAML、HCL 或 Java properties 格式的配置文件
- 可以监视配置文件的变动、重新读取配置文件
- 从环境变量中读取配置数据
- 从远端配置系统中读取数据，并监视它们（比如etcd、Consul）
- 从命令参数中读物配置
- 从 buffer 中读取
- 调用函数设置配置信息

**为什么不支持 INI 文件？**

作者认为 `INI` 文件非常差（我觉得还好），没有标准格式，而且很难验证。Viper 设计使用 JSON、TOML 或 YAML 文件。如果你愿意为 `INI` 配置提交 [PR](https://github.com/spf13/viper/pulls){:target="_blank"}，作者很乐意合并它，支持一个新的格式是简单的，必须考虑到兼容以及其他的一些通用问题才是关键。

在构建一个应用的时候，你不用关心配置文件的格式，你应该专注于业务本身，其他的让 Viper 帮你完成。

## 如何使用

说了这么多，我们快来试试吧，先把前面的例子重新实现一下。

```go
func main() {
    var config Config
    viper.SetConfigName("config")   // 设置配置文件名 (不带后缀)
    viper.AddConfigPath(".")        // 第一个搜索路径
    err := viper.ReadInConfig()     // 读取配置数据
    if err != nil {
        panic(fmt.Errorf("Fatal error config file: %s \n", err))
    }
    viper.Unmarshal(&config)        // 将配置信息绑定到结构体上
    fmt.Println(config)
}

// output
» {10666 {(127.0.0.1:3306)/biezhi root 123456}}
```

> Viper 可以搜索多个路径，但目前单个 Viper 实例仅支持单个配置文件，Viper默认不搜索任何路径。

如果这个时候我不想用 JSON 文件，把它换成 YAML 文件，那么格式变成这样了

```yml
port: 10666
mysql:
  url: "(127.0.0.1:3306)/biezhi"
  username: root
  password: 123456
```

我们将该文件命名为 `config2.yaml`

在没有 Viper 的时候我们恐怕需要再写一套实现了，那么在 Viper 中如何操作呢？

```go
viper.SetConfigName("config2")   // 设置配置文件名 (不带后缀)
```

没错，改这一行代码就可以了。

## 其他用法

### 设置默认值

默认值不是必须的，如果配置文件、环境变量、远程配置系统、命令行参数、Set 函数都没有指定时，默认值将起作用。

```go
viper.SetDefault("ContentDir", "content")
viper.SetDefault("LayoutDir", "layouts")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})
```

### 监听配置变化

Viper 支持在程序运行时动态加载配置，只需要调用 viper 实例的 `WatchConfig` 函数，你也可以指定一个回调函数来获得变动的通知。

```go
viper.WatchConfig()
viper.OnConfigChange(func(e fsnotify.Event) {
    fmt.Println("配置发生变更：", e.Name)
})
```

### 获取值

在Viper中，有一些根据值的类型获取值的方法，存在以下方法：

- `Get(key string) : interface{}`
- `GetBool(key string) : bool`
- `GetFloat64(key string) : float64`
- `GetInt(key string) : int`
- `GetString(key string) : string`
- `GetStringMap(key string) : map[string]interface{}`
- `GetStringMapString(key string) : map[string]string`
- `GetStringSlice(key string) : []string`
- `GetTime(key string) : time.Time`
- `GetDuration(key string) : time.Duration`
- `IsSet(key string) : bool`

如果 `Get` 函数未找到值，则返回对应类型的一个零值。可以通过 `IsSet()` 方法来检测一个健是否存在。

```go
viper.GetString("logfile") // Setting & Getting 不区分大小写
if viper.GetBool("verbose") {
    fmt.Println("verbose enabled")
}
```

对应的修改配置

```go
viper.Set("Verbose", true)
viper.Set("LogFile", LogFile)
```

### 访问嵌套 Key

访问方法也支持嵌套的键，如直接读取我们前面的 YAML 配置中的 MySQL 用户名

```go
GetString("mysql.username") // root
```

### 获取子级配置

当配置层级关系较多的时候，有时候我们需要直接获取某个子级的所有配置，可以这样操作：

```yaml
app:
  cache1:
    max-items: 100
    item-size: 64
  cache2:
    max-items: 200
    item-size: 80
```

```go
subv := viper.Sub("app.cache1")
```

subv 就代表：

```yaml
max-items: 100
item-size: 64
```

### 解析配置

可以将配置绑定到某个结构体、map上，有两个方法可以做到这一点：

- `Unmarshal(rawVal interface{}) : error`
- `UnmarshalKey(key string, rawVal interface{}) : error`

```go
var config Config
var mysql MySQL

err := Unmarshal(&config)            // 将配置解析到 config 变量
if err != nil {
    t.Fatalf("unable to decode into struct, %v", err)
}

err := UnmarshalKey("mysql", &mysql) // 将配置解析到 mysql 变量
if err != nil {
    t.Fatalf("unable to decode into struct, %v", err)
}
```

### 设置别名

我们可以为 key 设置别名，当别名的值被重置后，原 key 对应的值也会发生变化。别名可以实现多个 key 引用单个值。

```go
viper.RegisterAlias("loud", "Verbose")

viper.Set("verbose", true) 
viper.Set("loud", true)     // 这两句设置的都是同一个值

viper.GetBool("loud")       // true
viper.GetBool("verbose")    // true
```

### 从 `io.Reader` 中读取配置

Viper 预先定义了许多配置源，例如文件、环境变量、命令行参数和远程K / V存储系统，但您并未受其约束。
您也可以实现自己的配置源，并提供给 viper。

```go
viper.SetConfigType("yaml") // 这里不区分大小写

var yamlExample = []byte(`
Hacker: true
name: steve
hobbies:
- skateboarding
- snowboarding
- go
clothing:
  jacket: leather
  trousers: denim
age: 35
eyes : brown
beard: true
`)

viper.ReadConfig(bytes.NewBuffer(yamlExample))

viper.Get("name") // 返回 "steve"
```

### 从环境变量中读取

Viper 支持环境变量，使得我们可以开箱即用，很多时候环境参数是从命令行传入的。有四个和环境变量有关的方法：

- `AutomaticEnv()`
- `BindEnv(string...) error`
- `SetEnvPrefix(string)`
- `SetEnvKeyReplacer(string...) *strings.Replacer`

> 注意，环境变量时区分大小写的。

Viper 提供了一种机制来确保 `Env` 变量是唯一的。通过设置环境变量前缀 `SetEnvPrefix`，在从环境变量读取时会添加设置的前缀。`BindEnv` 和 `AutomaticEnv` 函数都会使用到这个前缀。

`BindEnv` 需要一个或两个参数。第一个参数是键名，第二个参数是环境变量的名称。环境变量的名称区分大小写。如果没有提供 ENV 的变量名，Viper 会自动假定该键名称与 ENV 变量名称匹配，并且 ENV 变量为全部大写。当你显式提供 ENV 变量名称时，它不会自动添加前缀。

使用 ENV 变量时要注意，当关联后，每次访问时都会读取该 ENV 值。Viper 在 `BindEnv` 调用时不读取 ENV 值。

`AutomaticEnv` 与 `SetEnvPrefix` 结合将会特别有用。当 `AutomaticEnv` 被调用时，任何 `viper.Get` 请求都会去获取环境变量。环境变量名为 `SetEnvPrefix` 设置的前缀，加上对应名称的大写。

`SetEnvKeyReplacer` 允许你使用一个 `strings.Replacer` 对象来将配置名重写为 Env 名。如果你想在`Get()` 中使用包含-的配置名 ，但希望对应的环境变量名包含 `_` 分隔符，就可以使用该方法。使用它的一个例子可以在项目中 `viper_test.go` 文件里找到。

```go
SetEnvPrefix("spf")       // 将会自动转为大写
BindEnv("id")             // 必须要绑定后才能获取

BindEnv("loglevel", "LOG_LEVEL"); //直接指定了loglevel所对应的环境变量，则不会去补全前缀

os.Setenv("SPF_ID", "13") // 通常通过系统环境变量来设置
id := Get("id")           // 13
```

### 读取远程 Key/Value

启用该功能，需要导入 `viper/remot` 包：

```go
import _ "github.com/spf13/viper/remote"
```

Viper 可以从如 `etcd`、`Consul` 的远程 `Key/Value` 存储系统的一个路径上，读取一个配置字符串（JSON, TOML, YAML 或 HCL格式）。

这些值会优于默认值，但会被从磁盘文件、命令行 flag、环境变量的配置所覆盖。

Viper 使用 [crypt](https://github.com/xordataexchange/crypt) 从 `K/V` 存储系统里读取配置，意味着你可以加密储存你的配置信息，并且可以自动解密配置信息，加密是可选项。

你可以将远程配置与本地配置结合使用，也可以独立使用。

`crypt` 有一个命令行工具可以帮助你存储配置信息到 K/V 存储系统，`crypt` 在 http://127.0.0.1:4001 上默认使用 etcd。

```bash
$ go get github.com/xordataexchange/crypt/bin/crypt
$ crypt set -plaintext /config/biezhi.json /Users/biezhi/settings/config.json
```

确认你的值被设置：

```bash
$ crypt get -plaintext /config/biezhi.json
```

有关 `crypt` 如何设置加密值或如何使用 Consul 的示例，请参考文档。

**远程 Key/Value 存储例子 - 未加密的**

```go
viper.AddRemoteProvider("etcd", "http://127.0.0.1:4001", "/config/biezhi.json")
viper.SetConfigType("json") // 因为不知道格式，所以需要指定
err := viper.ReadRemoteConfig()
```

**远程 Key/Value 存储例子 - 加密的**

```go
viper.AddSecureRemoteProvider("etcd", "http://127.0.0.1:4001",
"/config/biezhi.json","/etc/secrets/mykeyring.gpg")

viper.SetConfigType("json") // 因为不知道格式，所以需要指定
err := viper.ReadRemoteConfig()
```

更多例子可以参考 Github 主页。