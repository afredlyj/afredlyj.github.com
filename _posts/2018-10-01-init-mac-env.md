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

### 安装oh-my-zsh

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

### 配置GitHUB环境

生成ssh公钥：

```java
$ cd ~/.ssh
$ ssh-keygen -t rsa -C "your_email@example.com"
```

将生成之后的`id_rsa.pub`文件内容复制到github pub key中，然后本地测试：

```java
ssh -T git@github.com
```

如果出现如下内容，说明pub key配置成功：

```java
Warning: Permanently added the RSA host key for IP address '' to the list of known hosts.
Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
```

### 安装JDK

通过如下命令安装最新的JDK：
```java
$ brew cask install java
```

安装成功提示：

```java
installer: Package name is JDK 11
installer: Installing at base path /
installer: The install was successful.
🍺  java was successfully installed!
```

JDK11太新，线上用的是JDK8，所以我需要装JDK8:

```java
$ brew tap caskroom/versions
$ brew cask install java8
```

### 配置Maven

```java
export M2_HOME="~/Documents/program_tools/maven-3.5.4"
export M2=$M2_HOME/bin
export MAVEN_OPTS="-Xms256m -Xmx512m"
export PATH=$M2:$PATH
```


