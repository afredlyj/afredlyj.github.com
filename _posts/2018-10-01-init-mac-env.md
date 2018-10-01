---
layout: post
title: Mac环境配置
category: reading 
---


### 安装homebrew

```java
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

安装时如果报错：

```java
Failed during: /usr/bin/sudo /usr/bin/xcode-select --switch /Library/Developer/CommandLineTools
```

需要先安装XCode CommandLineTools，在[开发者下载](https://developer.apple.com/download/more/)页面搜索`XCode`找到`command line tools`并下载安装

### 安装ITerms2

用homebrew安装：

```java
brew cask install iterm2 
```

安装oh-my-zsh

```java
# 1. 打开 iTerm2
# 2. 通过 git 下载：
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
# 3. 复制创建~/.zshrc配置文件：
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
# 4. bash 切换成 zsh ：
chsh -s /bin/zsh
# 5. 按照提醒输入密码，完全退出iTerm2

```
