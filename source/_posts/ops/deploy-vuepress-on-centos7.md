---
title: 在centos7 上部署 vuepress
tags: 
 - 环境搭建
categories: 编程
date: 2018-12-29 00:01:00
---

## 前言

> vuepress是一款十分优秀简洁的文档生成器，可以根据目录下的md文档自动生成对应的html文件，界面简洁大方。每一个由 VuePress 生成的页面都带有预渲染好的 HTML，也因此具有非常好的加载性能和搜索引擎优化（SEO）。本文将介绍如何在CentOS7环境下部署vuepress。

官方主页：https://vuepress.vuejs.org/zh/  

官方文档：https://vuepress.vuejs.org/zh/guide/（官方文档就是用Vuepress搭建的，包括上面的主页）

项目地址：https://github.com/vuejs/vuepress

## 一、安装nodejs

``` bash
curl -sL https://rpm.nodesource.com/setup_8.x | sudo bash -
yum install nodejs
```

## 二、安装vuepress

``` bash
npm install -g vuepress
```

## 三、创建工作目录

``` bash
mkdir project
cd project
mkdir docs
```

## 四、初始化前

``` bash
npm init -y
vim package.json
```

编辑成如下内容，这里其实是设置命令别名。

```javascript
{
    "scripts": {
    	"docs:dev": "vuepress dev docs",
    	"docs:build": "vuepress build docs"
    }
}
```

创建.vuepress目录。

``` bash
mkdir .vuepress
cd .vuepress
```

创建config.js，这是vuepress的全局配置文件，大部分属性在这里设置。

``` bash
mkdir public
vim config.js
```

修改成如下内容，对应内容可以自行修改。

``` bash
module.exports = {
	title: '清风wiki',
	description: '我在等风，也在等你',
	// 相对于git仓库的路径 如全路径为：https://mfrank2016.github.io/wikibook/ 则设置为'/wikibook/'
	base: '/wikibook/',
	host: '0.0.0.0',
	// 运行端口
	port: 8081,

	themeConfig: {
		//gitc 仓库地址
    	repo: 'https://github.com/MFrank2016/wikibook',
    	// 如果你的文档不在仓库的根部
   		docsDir: 'docs',
    	// 可选，默认为 master
    	docsBranch: 'master',
    	// 默认为 true，设置为 false 来禁用
    	editLinks: true,
    	//导航栏
    	nav: [
      		{ text: 'Home', link: '/' },
      		{ text: 'Guide', link: '/guide/' },
      		{ text: 'External', link: 'https://google.com' },
      		{ text: 'Languages',
      		items: [
      		{ text: 'Chinese', link: '/language/chinese' },
      		{ text: 'Japanese', link: '/language/japanese' }
      		]}],
      	sidebar: [{
        	title: 'Group 1',
        	collapsable: false,
        	children: [
          		'/'
        		]
      		},
      		{
        	title: 'Group 2',
        	children: [
            	'/'
        		]
      		}
    	]
  	},  
    //搜索
    search: true,
    searchMaxSuggestions: 10,
    lastUpdated: 'Last Updated', // string | boolean
}
```

整体结构

``` bash
project
├─── docs
│ ├── README.md
│ ├── .vuepress
│   ├── config.js
│   └── public
│     └── hero.png
│ └── guide
│   └── README.md 
└── package.json
```

## 五、初始化

在docs目录下创建README.md

```bash
---
home: true
heroImage: /hero.png
actionText: 点击阅读
actionLink: /guide/
footer: MIT Licensed | Copyright © 2018-present Frank
---
```

然后回到project目录

```bash
# 开启调试模式，运行服务，此时打开 http://localhost:8081 (这里即上面设置的端口) 即能看到最简单的页面
vuepress dev

# 构建，此时会将md文档转化成html文件存储在docs/.vuepress/dist目录
vuepress build
```

## 六、调试部署

此时静态网页已经生成在了**docs/.vuepress/dist**目录下，可以先开启调试模式，然后使用ftp等软件先对服务器进行远程连接，修改docs下面的文档，每次修改上传后，会自动重新编译，当然整个过程需要一两分钟时间，这取决于服务器的性能。调整到合适的程度即可将其移动到nginx或者apache相应目录下即可。
