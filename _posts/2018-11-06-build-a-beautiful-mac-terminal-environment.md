---
layout: post
title: 打造 Mac 下高颜值好用的终端环境
---

如果你觉得当前的终端操作不符合你的气质，可以看看我今天来介绍的 Mac 终端利器，不过不会介绍太多细节操作。

![biezhi iterm]({{ "/public/images/2018/11/biezhi_iterm2.png" | prepend: site.cdnurl }} "biezhi")

<!-- more -->

# 它们是谁？

- iTerm2：号称 Mac 下最好的终端工具（嗯，我也这么认为，毕竟我不会别的了）
- zsh：一款强大的终端工具，能帮助你更高效地编写和执行命令。

# 安装 iTerm2

下面的安装我几乎都用 [brew](https://brew.sh/){:target="_blank"} 方式了，如果你还不懂什么是 brew 可以看看 [这个](http://zhailiange.com/2016/06/04/Homebrew/){:target="_blank"}。

所以下面我假设你已经安装了 `Homebrew`。

如果你从来没有运行过 `brew cask` 命令，可以先执行：

```shell
brew tap caskroom/cask
```

> 多执行也不会怀孕的，放心！

然后开始安装 `iTerm2`

```shell
brew cask install iterm2
```

安装成功后在 Launchpad 中可以看到有一个新图标出现，打开 iTerm2。

## 代码配色

默认的界面还是略显丑陋的，我们来设置一下代码配色吧。

<img src='{{ "/public/images/2018/11/open_settings.png" | prepend: site.cdnurl }}'  alt="iTerm2 设置" width="400px"/>

先检查下终端颜色配置为 `xterm-256color`，位置在 `iTerm2 -> Preferences -> Profiles -> Terminal`。

![iTerm2 终端颜色值]({{ "/public/images/2018/11/iterm_terminal_color.png" | prepend: site.cdnurl }} "iTerm2 终端颜色值")

然后就可以设置配色了，默认情况下 `iTerm2` 只有 7 种自带的配色，当然满足不了我们高颜值的需求了。有人就开源了一款叫 [iTerm2-Color-Schemes](https://github.com/mbadolato/iTerm2-Color-Schemes){:target="_blank"} 的配色合集，里面有各种经典、常用的配色方案，来使用 Git 下载到本地。

```shell
mkdir ~/.iterm2 && cd ~/.iterm2

git clone https://github.com/mbadolato/iTerm2-Color-Schemes
```

这里我创建了一个 `~/.iterm2` 的目录，放在别的目录都可以，它的目录结构是这样的：

```shell
 ~/.iterm2/iTerm2-Color-Schemes $ ls -la
total 72
-rw-r--r--    1 biezhi  staff  34131 Nov  6 11:34 README.md
drwxr-xr-x  198 biezhi  staff   6336 Nov  6 11:34 Xresources
drwxr-xr-x    3 biezhi  staff     96 Nov  6 11:34 backgrounds
drwxr-xr-x  198 biezhi  staff   6336 Nov  6 11:34 konsole
drwxr-xr-x  198 biezhi  staff   6336 Nov  6 11:34 putty
drwxr-xr-x  198 biezhi  staff   6336 Nov  6 11:34 remmina
drwxr-xr-x  198 biezhi  staff   6336 Nov  6 11:34 schemes
drwxr-xr-x  200 biezhi  staff   6400 Nov  6 11:34 screenshots
drwxr-xr-x  180 biezhi  staff   5760 Nov  6 11:34 terminal
drwxr-xr-x  198 biezhi  staff   6336 Nov  6 11:34 terminator
drwxr-xr-x  199 biezhi  staff   6368 Nov  6 11:34 termite
drwxr-xr-x  162 biezhi  staff   5184 Nov  6 11:34 tilda
drwxr-xr-x   19 biezhi  staff    608 Nov  6 11:34 tools
drwxr-xr-x    3 biezhi  staff     96 Nov  6 11:34 xfce4terminal
drwxr-xr-x  198 biezhi  staff   6336 Nov  6 11:34 xrdb
```

下面需要导入配色方案。

![导入配色方案]({{ "/public/images/2018/11/import_color_schemes.png" | prepend: site.cdnurl }} "导入配色方案")

![导入配色方案]({{ "/public/images/2018/11/choose_color_schemes.png" | prepend: site.cdnurl }} "导入配色方案")

> 选择 `schemes` 文件夹内的所有配色方案。

导入成功后就可以选择一些流行的配色方案了。

![选择配色方案]({{ "/public/images/2018/11/select_color_scheme.png" | prepend: site.cdnurl }} "选择配色方案")

选择配色后再去你的 iTerm 里面看会发现，已经好看了那么一点。

## 安装字体

为什么要安装字体呢？我们电脑的字体其实是可以用的，但是想要图标的这种字体就没法儿了：

<img src='{{ "/public/images/2018/11/hack_font.png" | prepend: site.cdnurl }}'  alt="iTerm2 设置" width="70%"/>

而这些图标字体其实是非 `ASCII` 码字体，在 iTerm2 中可以进行配置，所以先要安装这个字体。这款字体叫 [nerd-fonts](https://github.com/ryanoasis/nerd-fonts){:target="_blank"}，它支持下面这么多种图标。

![nerd-fonts]({{ "/public/images/2018/11/sankey-glyphs-combined-diagram.svg" | prepend: site.cdnurl }} "nerd-fonts")

使用 brew 安装

```shell
brew tap caskroom/fonts
brew cask install font-hack-nerd-font
```

> 注意：安装的时候会去 Github 下载字体，如果你下载失败可能是被墙了。
> 
> 那么可以通过 `https_proxy=127.0.0.1:1087 brew cask reinstall font-hack-nerd-font` 的方式安装，前提是你开启了代理。

安装成功后需要在 iTerm2 中配置一下，在 `iTerm2 -> Preferences -> Profiles -> Text -> Font -> Change Font` 栏位中，Text 下面勾选 `Use a different font for non-ASCII text`，然后在 `Non-ASCII font` 点击 `Change font` 修改：

![设置字体]({{ "/public/images/2018/11/settings_hack_font.png" | prepend: site.cdnurl }} "设置字体")

![选择字体]({{ "/public/images/2018/11/choose_font.png" | prepend: site.cdnurl }} "选择字体")

这里选择的字体是非 ASCII 码字符的字体，不要设置错了！选择好之后关闭即可。

# 安装 zsh

```shell
brew install zsh
```

![安装 zsh]({{ "/public/images/2018/11/install_zsh.png" | prepend: site.cdnurl }} "安装 zsh")

默认的 shell 是 bash，需要修改为 zsh：

```shell
sudo sh -c "echo $(which zsh) >> /etc/shells"
chsh -s $(which zsh)
```

修改时会提示你输入密码。

现在 zsh 安装完成了，安装虽简单，可配置麻烦啊，这你能忍吗？？当然不能！

于是，[oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh){:target="_blank"} 出现了，有了它 zsh 配置起来就方便多了，来安装一下它。

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

![安装 oh-my-zsh]({{ "/public/images/2018/11/install_oh_my_zsh.png" | prepend: site.cdnurl }} "安装 oh-my-zsh")

安装好之后可以看到界面发生了一点点变化，同时会产生一个名为 `.zshrc` 的配置文件，在用户家目录下面，我们以后主要就是修改它了。

# 配置主题

上面看到界面发生变化是因为 `oh-my-zsh` 默认帮我们配置了一个终端主题，你可以打开 `~/.zshrc` 文件看看：

```shell
ZSH_THEME="robbyrussell"
```

这些主题文件存储在 `~/.oh-my-zsh/themes` 目录下，你也可以使用其他的。

为了实现前面想要的酷炫的终端主题，有人写了一个名为 [powerlevel9k](https://github.com/bhilburn/powerlevel9k){:target="_blank"} 的高颜值主题。

![nerd-fonts]({{ "/public/images/2018/11/powerlevel9k.gif" | prepend: site.cdnurl }} "nerd-fonts")

看到这么骚的操作，赶紧来安装吧！先将主题下载到本地的主题目录中：

```shell
git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
```

然后修改 `zsh` 主题配置：

```shell
ZSH_THEME="powerlevel9k/powerlevel9k"
```

修改配置文件后一定要记得让配置生效，使用 `source` 命令：

```shell
source ~/.zshrc
```

现在来看看终端变成什么样子了！

![powerlevel9k]({{ "/public/images/2018/11/powerlevel9k.png" | prepend: site.cdnurl }} "powerlevel9k")

> 我这里 iTerm2 的代码配色选择的是：**Dracula**

如果你喜欢这个风格的话可以不用进行其他主题设置了，为了让它看起来简洁一点，我在 `.zshrc` 配置中又添加了几行：

```shell
POWERLEVEL9K_MODE="nerdfont-complete"
# Customise the Powerlevel9k prompts
POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(ssh dir vcs newline status)
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=()
POWERLEVEL9K_PROMPT_ADD_NEWLINE=true
```

- `POWERLEVEL9K_MODE`：设置 powerlevel9k 的字体是我们前面下载的
- `POWERLEVEL9K_LEFT_PROMPT_ELEMENTS`：将前面居右的几个元素放在左边了
- `POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS`：右边不放置任何元素（如果你喜欢在右边也可以加）
- `POWERLEVEL9K_PROMPT_ADD_NEWLINE`：在每个提示之前添加换行符

现在它变成这样了

![simple powerlevel9k]({{ "/public/images/2018/11/simple_powerlevel9k.png" | prepend: site.cdnurl }} "simple powerlevel9k")

更详细的配置可以参考 [Prompt Customization](https://github.com/bhilburn/powerlevel9k#prompt-customization){:target="_blank"} 和 [Stylizing Your Prompt](https://github.com/bhilburn/powerlevel9k/wiki/Stylizing-Your-Prompt){:target="_blank"}。

# 别名设置

装好 zsh 之后顺手就添加一下我自己常用的别名：

```shell
alias cls='clear'
alias ll='ls -l'
alias la='ls -a'
alias vi='vim'
alias ssr="http_proxy=http://127.0.0.1:1087 https_proxy=http://127.0.0.1:1087"
alias grep='grep --color=auto'
```

这样我们只需要输入较短的命令就可以干大事情了！当然这里你可以设置更多自己熟悉的一些操作，比如和编程语言相关的等等。

# zsh 插件推荐

zsh 那些酷插件可多了去了，我只推荐几个我认为比较实用的。

## extract

这个插件是用于解压的，解压各种包命令多可能会手误，用它只需要输入 `x biezhi.zip` 即可。

在 `.zshrc` 的 plugins 中添加 `extract` 配置即可，它支持解压 [这些](https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins/extract){:target="_blank"} 文件。

## autojump

这个插件主要帮助我们记住目录，一键直达。只要你脑海里有目录的几个字母，然后使用 `j [你知道的]` 按下 tab 即可，不用 `cd cd cd` 慢慢找。举个栗子：

我使用 `cd` 进入了 `blog` 这个目录，还进入了 `gitmoji` 目录。

```shell
cd workspace/projects/github/blog
```

如果用 `autojump` 的话，现在想进入 `blog` 目录只需要 `j blog` 即可，一般我们都会按下 `tab` 确定目录位置，当遇到多个类似的目录名的时候它会提示你输入数字进入。

**安装**

```shell
brew install autojump
```

安装后添加到 `autojump` 到 `zsh` 的 插件配置（plugins）里，再追加一句命令：

```shell
[[ -s $(brew --prefix)/etc/profile.d/autojump.sh ]] && . $(brew --prefix)/etc/profile.d/autojump.sh
```

让配置文件生效即可。

## zsh-syntax-highlighting

zsh-syntax-highlighting 用于高亮你的 zsh 可用命令，比如输入 `sleep`、`cat` 这些命令的时候就会高亮（功能上确实没啥乱用）。

```shell
brew install zsh-syntax-highlighting
```

安装好就行了，不用在 plugins 中追加。

## zsh-autosuggestions

这是一个神奇的终端自动提示插件，当你输入 `ps` 的时候它可能会出现 `ps -ef | grep helloworld`。是因为它会记住你曾经输入过的命令，当你再次输入前几个命令的时候帮你自动匹配，让你工作更高效。下面是一个演示：

<script id="asciicast-37390" data-size="big" src="https://asciinema.org/a/37390.js" async></script>

你可以直接使用 brew 安装

```shell
brew install zsh-autosuggestions
```

## colors

[colors](https://github.com/athityakumar/colorls){:target="_blank"} 是一个 Ruby 实现的脚本，它可以配合 powerlevel9k 显示电脑上的文件图标（应该是通过后缀判断的），使用的效果如下：

![colors]({{ "/public/images/2018/11/colors.png" | prepend: site.cdnurl }} "colors")

安装后就可以使用了

```shell
gem install colorls
```

# 其他技巧

- 连续按两次 `tab` 会补全列表，补全项可以使用 `ctrl+n/p/f/b` 上下左右切换
- 输入目录名即可进入，不用 `cd` 了，输入 `..` 即可到上级目录，返回上次目录输入 `-`
- 输入 `d` 即可看到目录列表
- 智能的命令纠错功能（需开启 `ENABLE_CORRECTION` 配置）

# 注意点

这样配置后打开 VSCode 就变成这幅样子：

<img src='{{ "/public/images/2018/11/vscode_font.png" | prepend: site.cdnurl }}'  alt="vscode 字体错误" width="450px"/>

如何修复呢？只需要下载 [Menlo-for-Powerline](https://github.com/abertsch/Menlo-for-Powerline) 里的 ttf 字体文件，然后双击安装。安装成功后在 vscode 中配置一项：

```vscode
"terminal.integrated.fontFamily": "Menlo for Powerline",
```

这样就会变成下面这个样子了。

<img src='{{ "/public/images/2018/11/vscode_new_font.png" | prepend: site.cdnurl }}'  alt="修复 vscode 字体" width="450px"/>