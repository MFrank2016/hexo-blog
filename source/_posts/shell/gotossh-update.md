---
title: 【效率工具】史上最好用的SSH一键登录脚本，超强更新！
date: 2019-03-29 00:26:30
tags:
categorys:
---

## 说明

虽然已经是凌晨，但丝毫不能掩盖我激动的心情，今天完成了对GotoSSH的一次大更新，新增了两个肥肠实用的功能，是真的好用，话不多说，先来看效果图：

普通的一键登录：

![](https://i.loli.net/2019/03/29/5c9cf654b339b.gif)

一键登录跳板机，然后跳转登录线上服务器：

![](https://i.loli.net/2019/03/29/5c9cf6c81ad1f.gif)

![](https://i.loli.net/2019/03/29/5c9cf6ee38d18.gif)

一键登录跳板机查看指定日志：

![](https://i.loli.net/2019/03/29/5c9cf72b7b707.gif)

一键登录跳板机后跳转线上服务器查看指定日志：

![](https://i.loli.net/2019/03/29/5c9cf76aad6ac.gif)

然后是更加劲爆内容，一键从跳板机复制指定文件到本地：

![](https://i.loli.net/2019/03/29/5c9cf782c0db4.gif)

一键从生产环境复制指定文件到本地：

![](https://i.loli.net/2019/03/29/5c9cf7b506db0.gif)

![20190329003615.png](https://i.loli.net/2019/03/29/5c9cf80147c31.png)

我只能说，是真的强。

## Shell脚本

Shell脚本已经发布到了`github`上，链接在此：https://github.com/MFrank2016/GotoSSH

可自行前往下载，好用的话别忘了给个star。

## 安装依赖

CentOS :
```
$ sudo yum install -y expect
```

Ubuntu :
```
$ sudo apt-get install tcl tk expect
```

Mac :
```
$ sudo brew install expect
```

## 安装 GotoSSH

```
$ git clone https://github.com/MFrank2016/GotoSSH.git
$ cd GotoSSH
$ chmod a+x gotossh
$ sudo cp gotossh /usr/local/bin/
```

## 配置

```
$ vim ~/.gotossh_config
server_name|ip|username|password|port|rely_server_no

[Server1]
commend=tail -f -n 10 testlog.log

[scp]
log1=~/testlog.log
```

配置文件由三部分组成。

第一部分是服务器的基本信息。

```
server_name|ip|username|password|port|rely_server_no
```

举个栗子：

```
JumpServer1|118.24.163.31|root|testpassword|22|0
OnlineServerB|111.231.59.85|root|testpassword2|22|1
```

最后一列是代表该服务器依赖于哪个服务器，如果该列的值设置为0，代表不依赖于其他服务器，否则代表需要先登录其他服务器后才能登录该服务器，目前暂时只支持二连跳，不支持多跳转。

第二部分是自定义命令，你可以在这里为每台服务器单独设置一些自定义命令。

```
[Server1]
commend=tail -f -n 10 testlog.log
```

Server1 表示以下是为第一台服务器设置的命令，同理Server2则表示为第二台设置的命令。对于顺序没有要求，只要为需要设置自定义命令的服务器添加该选项即可。

commend 是命令的名字，可以随意取名，最好简单一点，方便输入，等号后面是实际执行的命令。

举个栗子：

```
gotossh 1 commend
```

只要你小手一点回车，脚本便会自动帮你登录到第一台服务器，然后执行上面的命令`tail -f -n 10 testlog.log`。

注意，如果你输入的命令需要密码的话，需要在命令后面把密码也带上，并且用|分隔。

举个栗子：

```
[Server1]
commend=scp root@111.231.59.85:/var/log/test-service/test-service.log ./test-server.log|testpassword2
```

当然，强烈建议不要将类似`rm -rf xxx`等敏感操作放到这里，因为如果配置不当，容易引发事故。

配置文件的最后一部分是对于scp命令的支持。

```
[scp]
log1=~/testlog.log
log2=/var/log/test-service/test-service.log
```

log1 和 log2 都是随意起的名字，后面是服务器上你想要复制的文件路径，配置好之后，你就可以这样使用：

```
gotossh 1 scp log1
```

它就会自动把第一台服务器上的`~/testlog.log`文件复制到你的本地。

```
gotossh 2 scp log2
```

这个操作就更厉害了，因为第二台服务器设置了对第一台服务器的依赖，所以它会先登录第一台服务器，然后再复制第二台服务器上的文件到第一台服务器上，最后，退出服务器到本地，将第一台服务器上的复制品再拷贝到本地。

## 配置文件举例
```
$ vim ~/.gotossh_config
JumpServer1|118.24.163.31|root|testpassword|22|0
OnlineServerB|111.231.59.85|root|testpassword2|22|1

[Server1]
log=tail -f -n 20 testlog.log

[Server2]
log=tail -f -n 20 /var/log/test-service/test-service.log
cd=cd /var/log/test-service/

[scp]
log3=~/testlog.log
log4=/var/log/test-service/test-service.log
```

## 用法
```
$ gotossh
######################################################################################
#                                  [GOTO SSH]                                        #
#                                                                                    #
#                                                                                    #
# [1] test_server - 192.168.0.1:root                                                 #
# [2] online_server - 192.168.2.2:root                                               #
#                                                                                    #
#                                                                                    #
######################################################################################
Server Number:(Input Server Number Here)
```

```
gotossh 1
gotossh 2
gotossh 1 log
gotossh 2 log
gotossh 2 cd
gotossh 1 scp log3
gotossh 2 scp log4
```

## 解决了什么问题

1. 查询线上服务器日志的时候，需要先登录跳板机，然后再登录服务器，过程比较麻烦。需要多次查看服务器信息，如，ip，用户名，密码等，查看后还需要来回进行复制。利用GotoSSH，配置好服务器信息之后，可以直接一键跳转。
2. 增加了登录服务器后执行自定义命令，这一点主要是在查看日志的时候，还需要先去查看一下服务的日志路径，然后再切回来看日志，既然每次都是模板式操作，为何不简化一下呢？
3. 服务器上有时候操作很不方便，因为对权限做了严格的限制，很多命令无法使用，所以增加了对`scp`命令的支持，可以将线上服务器日志一键拷贝到本地，岂不是美滋滋。

最后再贴一下项目地址：https://github.com/MFrank2016/GotoSSH

如果觉得还不错，别忘了加个star✨也欢迎关注我的公众号留言交流。

![](https://i.loli.net/2019/03/14/5c8a58ba229ca.png)
