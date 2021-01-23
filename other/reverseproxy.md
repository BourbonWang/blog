# Docker 安装 Nginx 并实现反向代理

## 安装 Nginx 镜像

### 拉取镜像

```sh
docker pull nginx
```

![](https://s3.ax1x.com/2021/01/22/solJBT.png)

查看镜像

```sh
docker images
```

### 创建实例

```sh
docker run --name nginx-test -p 80:80 -d nginx
docker start nginx-test
```

浏览器访问服务器ip，运行nginx欢迎页面

![](https://s3.ax1x.com/2021/01/22/so3wOx.png)

### 创建关键目录映射点

本地创建文件夹

- html: nginx存储网页的目录
- logs：日志目录
- conf：配置文件目录

```sh
mkdir -p nginx/html nginx/logs nginx/conf
```

将刚才创建的nginx容器的配置文件拷贝到本地

```sh
docker cp nginx-test:/etc/nginx ~/nginx/conf
```

创建新的nginx容器

```sh
docker run -d -p 80:80 --name nginx -v ~/nginx/html:/usr/share/nginx/html -v ~/nginx/logs:/var/log/nginx -v ~/nginx/conf:/etc/nginx nginx
docker start nginx
```

## 反向代理

反向代理就是将访问本机的数据代理转发到本机的其他端口。比如，一个网站在4000端口上，而浏览器默认访问而的是80端口。就可以用nginx将访问80端口的数据转发到4000端口上，然后浏览器直接访问80端口就可以看到网站了。

所以要先知道要反向代理的服务的地址。由于我们要代理的是容器，先查看容器的ip

```sh
docker inspect 容器名
```

在最后的`Networks`下看到`IPAddress`

![](https://s3.ax1x.com/2021/01/22/soJmfe.png)

编辑`conf/conf.d/default.conf`，修改`proxy_pass` 为目标容器的ip和端口号

```
location / {
        #root   /usr/share/nginx/html;
        #index  index.html index.htm;
        proxy_pass http://172.17.0.2:4000;
    }
```

访问浏览器，成功跳转到目标网页