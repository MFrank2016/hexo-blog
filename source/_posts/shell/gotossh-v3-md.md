---
title: 【效率工具】史上最好用的SSH一键登录脚本，第三版更新！
date: 2019-04-04 19:10:39
tags:
categorys:
---

## 说明

时隔一周，GotoSSH又迎来了一次重大更新，让这个史诗级的shell工具变得更加丝般顺滑了~

这次的主要更新是对自定义全局命令以及自定义属性的支持，让设置更加灵活，此外，对各个细节进行了调整，并修复了一些极少数情况下可能会发生的bug。

另外，最重要的一点是，对代码进行了大量优化和注释，让小白也能很轻松的看懂各个地方是在做什么事情，毕竟对于服务器信息这么隐私的信息，交给一个第三方shell来管理，大家难免会有些不放心嘛，这个可以理解，所以特意做了这个更新，让大家能放心食用。

有能力的小伙伴也可以把这个shell脚本自行改进，让它变得更加好用，如果有其他想法，欢迎提出，会考虑在后续更新中进行添加。

## 更新后样式

普通的一键登录到服务器:

![](https://i.loli.net/2019/04/08/5caa300e3e11c.gif)

先登录跳板机，然后自动跳转到线上服务器：

![](https://i.loli.net/2019/04/08/5caa305847211.gif)

![](https://i.loli.net/2019/04/08/5caa30d31db3a.gif)

登录服务并查看日志：

![](https://i.loli.net/2019/04/08/5caa36045135c.gif)

![](https://i.loli.net/2019/04/08/5caa3603366a4.gif)

登录跳板机，然后跳转线上服务器并查看指定日志：

![](https://i.loli.net/2019/04/08/5caa3ac235cf7.gif)

从服务器复制文件到本地：

![](https://i.loli.net/2019/04/08/5caab56a6e3c3.gif)

从线上服务器复制文件到跳板机，然后再复制到本地：

![](https://i.loli.net/2019/04/08/5caab6078e829.gif)

列举所有服务器:

![](https://i.loli.net/2019/04/08/5caab7530dc2c.gif)

列举服务器支持的所有命令:

![list-all-the-commands.gif](https://i.loli.net/2019/04/08/5caab7d733ef1.gif)

## v3版本更新功能

1. 新增了两个命令，一个是查看服务器列表，一个是查看支持的命令列表。
   
因为有小伙伴反映说，记不住哪个服务器是几号，每次需要先输入`gotossh`来查看，然后再`ctrl + c`退出，之后再进行长命令操作，感觉不太优雅。emmmm，于是就有了这么个功能：

![](https://i.loli.net/2019/04/08/5caab7530dc2c.gif)

现在可以使用`gotossh -l`查看所有的服务器列表了。

此外，顺便增加了对自定义命令的更友好支持，一是在选择完服务器后，会显示该服务器支持的命令列表，包括该服务器的自定义命令，以及全局命令。

![list-all-the-commands.gif](https://i.loli.net/2019/04/08/5caab7d733ef1.gif)

2. 配置文件中，新增了`setting`节点和`common-command`节点。
   
前者是用于设置全局配置信息，目前仅有version信息，用于之后的升级迭代。后续会考虑加入如颜色，显示方案等自定义配置。

后者即全局公用命令，可以看做是模板命令，为什么要做这个功能呢？

很多服务的日志地址其实是类似的，比如A服务的日志地址也许是：`/var/log/server-a/service-a.log`，B服务的日志地址也许是：`/var/log/server-b/server-b.log`，它们的大致路径其实是差不多的，所以如果有了模板命令，我们便不需要给每个服务器来单独设置一个自定义命令了，只需要在该自定义属性中配置相应属性即可。

比如设置一条模板命令：

```
[common-command]
log=/var/log/[service-name]/[service-name].log
```

再为服务a和服务b设置相应的属性：

```
[Server-Attribute-service-a]
service-name=service-a

[Server-Attribute-service-b]
service-name=service-b
```

这样一来，使用就更加优雅了，管理起来也更加方便。

3. 配置文件中，服务器信息的分割符由原来的“|”改成了“||”
   
因为考虑到密码中可能含有“|”，所以进行了上述调整，不过仍旧没法解决密码中存在“||”的情况，emmm，这种情况应该不多，暂时先不考虑了。

4. 配置文件中，改用`link_name`作为服务器标识

之前配置自定义命令时，使用的是`Server-ServerNo`的形式，但如果服务器数量比较多，删除前面的服务器配置后，会导致后面的服务器编号改变，这样就需要对自定义节点进行调整，比较麻烦，所以使用`Server-link_name`来作为节点名称就是来解决这个问题的。

5. 配置文件中，新增了自定义属性

上面其实已经看到过了，可以新增`Server-Attribute-link_name`节点来设置服务器的自定义属性，这个自定义属性可以用在自定义命令或者全局公用命令中进行替换。

另外，还新增了两个特殊的自定义属性`[P1][P2]`，分别代表传入脚本的第三个和第四个参数，举个栗子：

```
[Server-service-a]
cd=cd [P1]
```

使用如上配置后，当输入`gotossh 1 cd /var/log/service-a`（假设service-a是第一台服务器）后，将会先登录该服务器，然后执行`cd /var/log/service-a`命令，这里`[P1]`将会被传入脚本的第三个参数`/var/log/service-a`所替代，同理，还可以在命令中使用`[P2]`，它将被第四个参数替代。

6. 新增了大量注释，让代码看起来更加清晰

目的在前面已经说过了，这里就不再赘述了，希望大家多提建议，一起来让这个shell脚本变得更好好用。

## 旧版本升级

如果你已经使用了之前的版本，那么使用新版本的话，你需要进行以下操作：

1. 进入`/usr/local/bin/`删除原来的shell。
2. 安装依赖

```shell
$ brew install gnu-sed --with-default-names
$ export PATH="$(brew --prefix coreutils)/libexec/gnubin:/usr/local/bin:$PATH"
$ export MANPATH="/usr/local/opt/coreutils/libexec/gnuman:$MANPATH"
```
3. 拉取最新代码并安装

```shell
$ git clone https://github.com/MFrank2016/GotoSSH.git
$ cd GotoSSH
$ chmod a+x gotossh
$ sudo cp gotossh /usr/local/bin/
```

shell里已经写好了配置升级的函数，所以不用太担心旧配置的调整。如果想要使用新功能的话，参照上面的说明，添加相应的节点，如`common-command`即可。

## 小结

`GotoSSH`虽然只是一个小的脚本，但是说实话，这个几百行的脚本调试起来可真的不容易，没法打断点就只能用输出的方式一点点的排查问题，比较蛋疼，清明节花了一整天的时间才调试好，希望大家能多多支持一下，给项目加个star的话就非常感谢啦。

