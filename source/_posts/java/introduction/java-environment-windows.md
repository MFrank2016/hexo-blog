---
title: 【Java入门篇】二、Java开发环境搭建——Windows篇
tags: 
 - Java入门
 - Java
 - 环境搭建
categories: 编程
date: 2018-12-28 19:00:02
---

> 你为了你的正义，我为了我的正义。 -- 《火影忍者》

## 一、安装JDK

官网下载链接：https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

{% asset_img 1-jdk-windows.png jdk-windows %}

## 二、配置环境变量

需要配置以下几个环境变量：

JAVA_HOME     配置JDK安装路径

PATH                  配置JDK命令文件的位置

CLASSPATH      配置类库文件的位置

### 1、我的电脑（右键）-->属性-->高级系统设置

{% asset_img 2-jdk-windows.png jdk-windows %}

### 2、环境变量-->新建

{% asset_img 3-jdk-windows.png jdk-windows %}

{% asset_img 4-jdk-windows.png jdk-windows %}

{% asset_img 5-jdk-windows.png jdk-windows %})

(1)新建->变量名"JAVA_HOME"，变量值"C:/Java/jdk1.8.0_144"（即JDK的安装路径） 

(2)编辑->变量名"Path"，在原变量值的最后面加上“;%JAVA_HOME%/bin;%JAVA_HOME%/jre/bin” 

(3)新建->变量名“CLASSPATH”,变量值“.;%JAVA_HOME%/lib;%JAVA_HOME%/lib/dt.jar;%JAVA_HOME%/lib/tools.jar”

### 3、确认环境配置是否正确

在控制台分别输入java，javac，java -version 命令，出现如下所示的JDK的编译器信息，包括修改命令的语法和参数选项等信息。

java命令：

{% asset_img 6-jdk-windows.png jdk-windows %}

javac命令：

{% asset_img 7-jdk-windows.png jdk-windows %}

java -version命令：

{% asset_img 8-jdk-windows.png jdk-windows %}

### 4、在控制台下验证第一个java程序：

右键--》新建--》文本文档

```java
public class Test {
    public static void main(String[] args) {
    System.out.println("Hello World!");
    }
}
```

用记事本编写好，点击“保存”，并保存在桌面后，先在控制台中进入桌面的目录。

```bash
C:
cd /Users/[用户名]/Desktop
```

上面的[用户名]改成你的计算机用户名即可，不清楚的话打开我的电脑，进C盘目录：C:/Users 找一下。

输入javac Test.java和java Test命令，即可运行程序（打印出结果“Hello Java”）。

{% asset_img 9-jdk-windows.png jdk-windows %}

至此，Java开发环境搭建成功。