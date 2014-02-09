---
layout: post
title: git使用
category: funny
---

### windows下修改git的工作目录
安装git后，它的默认路径会在`C:/User/Administrator`下，如果设置的工作目录不在这个文件夹，每次打开Git Bash，都要各种`cd`操作，很麻烦，故google找方法解决这个问题。  

 - 打开Git的安装目录，比如我的安装目录`D:/Program Files (x86)/Git`，打开`etc/profile`，找到  

~~~~  
# normalize HOME to unix path  
HOME="$(cd "$HOME" ; pwd)"  
export PATH="$HOME/bin:$PATH"  
~~~~  

 - 添加两行代码，最后代码如下：  

~~~~  
# normalize HOME to unix path  
HOME="E:/workspace/gitrepo"  
HOME="$(cd "$HOME" ; pwd)"  
cd
export PATH="$HOME/bin:$PATH"
~~~~  

 也就是我把默认目录`C:/User/Administrator`，切换到了`E:/workspace/gitrepo`，保存，重启`Git Bash`：  

~~~~  
$ pwd  
/e/workspace/gitrepo  
~~~~  

 OK，搞定

### git的一些命令说明

 *　Clone现有仓库到本地  

 ~~~~

git clone https://github.com/alibaba/dubbo.git

 ~~~~

 该命令将在本地的当前目录创建一个dubbo文件夹，并把该仓库的所有文件都clone到该文件夹下，如果想要将该文件夹重命名，则可以用以下命令：

 ~~~~

git clone https://github.com/alibaba/dubbo.git mydubbo

 ~~~~
