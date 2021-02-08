# 使用 Gitbook 搭建博客

gitbook 使用markdown 编写，简单易用。通过配置插件，也可以添加很多主题和小功能。适合搭建博客，电子书等。

## 安装 node.js

gitbook是一个基于Node.js的命令行工具，所以要先安装Node.js(下载地址[https://nodejs.org/en/](https://links.jianshu.com/go?to=https%3A%2F%2Fnodejs.org%2Fen%2F)，找到对应平台的版本安装即可)。

或使用包管理安装。

```sh
apt-get install nodejs
```

安装Gitbook：

```sh
npm install -g gitbook-cli
```

## 编辑工具

typora：[https://www.typora.io/](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.typora.io%2F)

```sh
apt-get install typora
```

## Gitbook 初始化

在空文件夹中执行

```sh
gitbook init
```

文件夹中将多两个文件

- `README.md` ：封面介绍
- `SUMMARY.md` ：配置目录结构

## 目录

编辑`SUMMARY.md`

```markdown
# Summary

* [Introduction](README.md)
* [前言](readme.md)
* [第一章](part1/README.md)
    * [第一节](part1/1.md)
    * [第二节](part1/2.md)
    * [第三节](part1/3.md)
    * [第四节](part1/4.md)
* [第二章](part2/README.md)
* [第三章](part3/README.md)
* [第四章](part4/README.md)
```

编辑后执行 `gitbook init` 将自动按以上目录寻找或创建文件。

每一篇文章都是一个.md文件，这样就可以开始写博客了。

## 启动服务

编辑好文件后，执行

```sh
gitbook init
gitbook serve
```

gitbook将在本地4000端口启动服务。浏览器访问 http://localhost:4000/

至此，已经可以使用 gitbook 搭建博客了。

## 插件

文件夹下创建 `book.json` ，如果已有，就直接打开。

配置插件等都是在这里，也可以复制别人的配置项。

`plugins` 是配置插件的位置，gitbook自带了5个插件，在名字前面加 - 可以禁用插件：

- sharing：右上角分享功能
- font-settings：字体设置（左上方的"A"符号）
- livereload：为 GitBook 实时重新加载
- highlight： 代码高亮
- search： 导航栏查询功能（不支持中文）

推荐几个我在使用的插件：

- page-treeview：每篇文档头部生成标题树
-  code：为代码块添加行号和复制按钮
- pageview-count：阅读量计数
- popup：插件用于点击图片时，打开新的网页用来查看高清大图。
- tbfed-pagefooter：在每个页面的最下方自动添加页脚信息
- favicon：修改网页标题的图标
- search-plus：原搜索插件不支持中文搜索，所以使用该插件进行替换。
- expandable-chapters 和 chapter-fold ：导航目录
- hide-element：隐藏界面元素
- back-to-top-button：返回顶部按钮
- splitter：侧边栏可自行调整宽度
- sharing-plus：分享当前页面，比默认的 sharing 插件多了一些分享方式。
- donate：打赏模块，在每篇文章底部都会加上一个按钮，点击显示图片
- github：右上角跳转到 github 主页

附上我的配置文件

```json
{
    "author": "Bourbon",
    "description": "学习，记录，分享，进步",
    "extension": null,
    "generator": "site",
    "isbn": "",
    "links": {
        "sharing": {
            "all": null,
            "facebook": null,
            "google": null,
            "twitter": null,
            "weibo": null
        }
    },
    "output": null,
    "pdf": {
        "fontSize": 12,
        "footerTemplate": null,
        "headerTemplate": null,
        "margin": {
            "bottom": 36,
            "left": 62,
            "right": 62,
            "top": 36
        },
        "pageNumbers": true,
        "paperSize": "a4"
    },
    "plugins": [
	    "page-treeview",
	    "code",
	    "pageview-count",
	    "popup",
	    "tbfed-pagefooter",
	    "favicon",
    	    "search-plus",
	    "expandable-chapters",
	    "hide-element",
	    "back-to-top-button",
	    "splitter",
    	    "-lunr",
	    "-search",
	    "-sharing",
	    "sharing-plus",
	    "chapter-fold",
	    "donate",
	    "github"
    ],
    "pluginsConfig": {
	"code": {
	    "copyButtons": false	
	},    
        "hide-element": {
            "elements": [".gitbook-link"]
        },
        "tbfed-pagefooter": {
            "copyright": "Copyright © 1141134779@qq.com 2020"
        },
        "favicon": {
            "shortcut": "assert/favicon.ico",
            "bookmark": "assert/favicon.ico",
            "appleTouch": "assert/favicon.ico",
            "appleTouchMore": {
                "120x120": "assert/favicon.ico",
                "180x180": "assert/favicon.ico"
            }
        },
        "fontsettings": {
            "theme": "white",
            "family": "sans",
            "size": 2
        },
        "page-treeview": {
            "copyright": "Copyright © 1141134779@qq,com 2020",
            "minHeaderCount": "2",
            "minHeaderDeep": "2"
        },
        "sharing": {
            "all": ["facebook", "google", "linkedin", "twitter", "weibo", "qq"]
        },
	"donate": {
	    "wechat": "/assert/wechat.jpg",
	    "alipay": "/assert/alipay.jpg",
	    "title": "",
            "button": "赏",
            "alipayText": "支付宝打赏",
            "wechatText": "微信打赏"
	},
	"github": {
	    "url": "https://github.com/BourbonWang"	
	}
    },
    "language": "zh-hans",
    "title": "Bourbon",
    "variables": {},
    "styles": {
        "website": "/assert/styles/website.css"
    }
}
```

## webhook 实现服务器自动更新博客

由于需要频繁更新博客，不可能每次更新都重新上传服务器，所以需要服务器自动拉取gitbook文件夹。可以将gitbook上传到github，然后使用 webhook 将push与服务器关联。这样每次更新后push到github，然后webhook执行服务器的脚本，将文件夹 pull 下来，重启gitbook 服务。

### docker

这里将博客放到docker里，方便管理，不会与其他服务冲突。

拉取 node 镜像，创建容器

```sh
docker run -itd --name blog -p 4000:4000 node:10.19 /bin/bash
docker exec -it blog /bin/bash
```

容器内同样方法安装gitbook。

### 部署 webhook

apt 换源(debian)后，安装 pip

```sh
apt-get install python3-pip
```

我使用的 webhookit：[https://github.com/hustcc/webhookit](https://github.com/hustcc/webhookit) 可根据文档自行安装。

由于这个webhook只能用python2, 注意使用 pip2。

```sh
pip2 install webhookit
webhookit_config > /home/webhook/config.py
```

编辑config.py ，只需要修改`repo_name/branch_name` 和 `SCRIPT` 

```python
# -*- coding: utf-8 -*-
'''
Created on Mar-03-17 15:14:34
@author: hustcc/webhookit
'''

# This means:
# When get a webhook request from `repo_name` on branch `branch_name`,
# will exec SCRIPT on servers config in the array.
WEBHOOKIT_CONFIGURE = {
    # a web hook request can trigger multiple servers.
    'repo_name/branch_name': [{
        # if exec shell on local server, keep empty.
        'HOST': '',  # will exec shell on which server.
        'PORT': '',  # ssh port, default is 22.
        'USER': '',  # linux user name
        'PWD': '',  # user password or private key.

        # The webhook shell script path.
        'SCRIPT': '/home/webhook/shell.sh'
    }, 
	...],
	...
}
```

创建shell.sh，就是自动执行的脚本，用来拉取博客，完成更新

```sh
cd /home/blog
git fetch --all
git reset --hard origin/master
git pull
gitbook install
gitbook init
gitbook serve
```

### 启动 webhook

```sh
webhookit -c /home/webhook/config.py -p 4001
```

监听4001端口，浏览器访问即可查看webhook URL以及配置信息。

### 配置 github

仓库 -> Settings -> Webhooks -> Add webhook

- payload URL：填写webhook URL
- Content type ：application/json
- 触发条件：Just the push event.