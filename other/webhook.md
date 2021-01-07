# 使用 webhook 部署 Gitbook 到服务器

## 概述

上一篇记录了创建和使用gitbook，现在将它部署到服务器。由于需要频繁更新博客，不可能每次更新都重新上传服务器，所以需要服务器自动拉取gitbook文件夹。可以将gitbook上传到github，然后使用 webhook 将push与服务器关联。这样每次更新后push到github，然后webhook执行服务器的脚本，将文件夹 pull 下来，重启gitbook 服务。

## 服务器配置

## docker

这里我们将gitbook 放到docker里，方便管理，不会影响服务器的其他应用。服务器 CentOS7 系统。

### 安装docker

```sh
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

教程很多，不再赘述。

### 创建容器

拉取 node 镜像

```sh
docker pull ubuntu
```

启动容器，绑定端口

```sh
docker run -itd --name blog -p 4000:4000 node /bin/bash
```

进入容器

```sh
docker exec -it blog /bin/bash
```

