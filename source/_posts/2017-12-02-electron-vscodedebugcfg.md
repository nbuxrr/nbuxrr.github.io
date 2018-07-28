---
title: 使用VSCode调试electron项目
categories: 技术
tags: [electron, vscode]
keywords: Electron
description: Electron project debug
---

使用新版本的vscode调试各种语言的项目（包扩electron项目）的配置貌似便捷了很多，有点小惊喜。以electron的quick-start项目为例，小记一下，O(∩_∩)O哈哈~
## vscode版本

![](http://pic.xrr.fun/blog/20171202/vscodeversion.png)

## 项目代码
```shell
# 克隆示例项目的仓库
$ git clone https://github.com/electron/electron-quick-start

# 进入这个仓库
$ cd electron-quick-start

# 安装依赖
$ npm install

# 运行
$ npm start
```
![](http://pic.xrr.fun/blog/20171202/project.png)

## vscode调试配置
刚开始项目没有调试配置，需要先选择调试器环境，例如C++代码就可以选C++(GDB/LLDB)，这里electron运行的环境是nodejs，则选Node.js。

![](http://pic.xrr.fun/blog/20171202/cfg1.png)

选择好调试环境后，在工程目录下会自动生成.vscode/launch.json文件。接着就是点“添加配置”按钮，使用提示下拉框方便的选择调试工程的类型，自动填充相应配置。

![](http://pic.xrr.fun/blog/20171202/cfg2.png)
![](http://pic.xrr.fun/blog/20171202/cfgfinish.png)

注：其中默认的legacy协议在实际运行调试时vscode报如下错误

![](http://pic.xrr.fun/blog/20171202/err.png)

根据提示将协议修改为inspector，就调试正常了。

## 调试
vscode调试的快捷键与号称宇宙第一IDE的visual studio一致

![](http://pic.xrr.fun/blog/20171202/debug.png)
