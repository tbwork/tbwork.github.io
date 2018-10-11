---
layout: post
title:  "关于MacOS下双击多开Eclipse的Automator方案"
date:   2018-10-11 09:58:18 +0800
categories: 码农手册
tags: Eclipse MacOS
author: Tommy.Tesla
mathjax: true
---

## 问题背景

相信很多同学看到标题后就没兴趣看了，因为他们都用Intellij（IDEA），确实大部分选择Mac的Java Coder选用都是IDEA，哈哈哈。

但也有一些像我一样的古董IDE铁粉（IDEA也有在用，作为补充），包括在跟一些较为著名的github项目（e.g. [zipkin](https://github.com/openzipkin/zipkin) )作者们的交流过程中发现还是有很多Eclipse国际使用者的。Eclipse也有其自身的一些优势和特点，比如开源并永久免费、独特的用户习惯（比如git项目在改动后项目名上会多个尖括号）等。

有些IDE自己提供了多开程序的方案，比如IDEA等。Eclipse自己内置了WorkingSet等类似多项目功能，所以并没有提供MacOS下的现成多开方案。实际场景下，尤其在网络通信开发或者微服务框架体系下，多开Eclipse进行应用间调试是很常见的需求。

工欲善其事，必先利其器，虽然本文说的并不是啥高深技术，但事实中我发现知道这样方案的很少。仅以此文献给那些在Mac上使用Eclipse的Coder们 :)，这是一个值得一用的方案，每次打开节省1秒钟、、、一天、一月、一年下来就多了，不要让已经很繁琐的编码工作再节外生枝、、、

## 现有的流行方案

通过打开iTerm或者Terminal定位到Eclipse应用所在目录，然后使用open命令打开一个新的窗口，例如：
```
cd /Users/tangbo/eclipse/jee-2018-092
open -n Eclipse.app

```
> 假设您的Mac系统已经安装了Eclipse。

## Automator方案

大致思路：使用Automator创建一个Eclipse应用的壳应用，并且在实际使用时双击该应用进行使用。
理论上这个思路适用于为任何应用创建多开功能的壳应用。
以下是Step by Step教程：

### 1. 找到Applications中的Automator应用，双击打开。
### 2. 如下图所示，双击**Run Shell Script**，这时候旁边会弹出一个输入框，输入要执行的脚本[1]。
![image.png](/image/reopen-eclipse-in-macos/automator.png)
### 3. 输入脚本:
```
cd /Users/tangbo/eclipse/jee-2018-092
open -n Eclipse.app
```
### 4. "Command + S"保存为名为Eclipse应用，放到桌面上。
![image.png](/image/reopen-eclipse-in-macos/eclipse-app.png)

### 5. 这时候双击这个图标已经可以双击无限打开多个Eclipse窗口了。但可能我们觉得这个应用的Logo实在太丑，换成Eclipse的Logo不是更好？这只需要右击这个应用，选择**Get info**，点下红框里的图标，如下图所示：
![image.png](/image/reopen-eclipse-in-macos/eclipse-info.png)

### 6. Bing搜索个喜欢的Eclipse无版权图标，[这里有个现成的](https://cn.pling.com/img/hive/content-pre1/87185-1.png)，Command + C，回到刚刚的红框，点击下，Command + V即可完成图标替换。最终效果如下图：

![image.png](/image/reopen-eclipse-in-macos/result-demonstration.png)


## 参考资料

[1] Run a shell script on OS X without having a terminal window appear? https://superuser.com/questions/360247/run-a-shell-script-on-os-x-without-having-a-terminal-window-appear.




* content
{:toc}