---
layout: post
title: 打造漂亮好用的终端
---

今天我介绍在 Mac 下打造漂亮的终端环境。

# 配置 iTerm2

## 安装 iTerm2

# 如果你從來沒有用過 brew cask 的話需要先跑這行

```shell
$ brew tap caskroom/cask
```

```shell
$ brew cask install iterm2
```

## 配色

```shell
$ mkdir -p ~/.iterm2/iTerm2-Color-Schemes
$ git clone https://github.com/mbadolato/iTerm2-Color-Schemes ~/.iterm2/iTerm2-Color-Schemes
```

全选导入

## 安装字体

默认情况下我们电脑的字体是无法显示那些漂亮的图标的，沒有安裝的話畫面會長這樣，遇到 icon 會變框框問號。
所以需要安装如下字体。

```shell
$ brew tap caskroom/fonts
$ brew cask install font-hack-nerd-font
```

> 注意：安装的时候其实会去 Github 下载字体，如果你下载失败可能是被墙了。
> 那么可以通过 `https_proxy=127.0.0.1:1087 brew cask reinstall font-hack-nerd-font` 的方式安装，前提是你开启了代理。

安装成功后我们在 iTerm2 - Text 下面勾选 `Use a different font for non-ASCII text`。

然后在 `Non-ASCII font` 点击 `Change font` 修改：


选择 OK 之后关闭即可。

# 安装 ZSH

```shell
$ brew install zsh
```

默认的 shell 是 bash，我们需要修改为 zsh：

```shell
$ sudo sh -c "echo $(which zsh) >> /etc/shells"
$ chsh -s $(which zsh)
```

修改时会提示你输入密码。

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

```shell
$ git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
```

```shell
ZSH_THEME="powerlevel9k/powerlevel9k"
```

```shell
source ~/.zshrc
```

让配置文件生效后就变成这样了！

## 安装 zsh 插件

```shell
$ brew install zsh-syntax-highlighting
$ brew install zsh-autosuggestions
```