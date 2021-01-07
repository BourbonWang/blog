# Git同步本地项目到Github

将本地的项目同步到github上，这样可以随时pull到本地，修改完后再push到github仓库。可以随时随地修改代码，也避免了项目的丢失的风险。

## 本地git获取github提交权限

### 设置用户名和邮箱 

```sh
git config --global user.name 'your_name'
git config --global user.email 'your_email'
```

### 生成ssh密钥

```sh
ssh-keygen -t rsa -C 'your_email'
```

提示设置存储位置和口令等，回车跳过。默认存储在 ~/.ssh/id_rsa.pub

### 复制到github

将生成的id_rsa.pub文件中的公钥复制到github的**setting / SSH AND GPG KEY / SSH keys**

### 测试完成

```sh
ssh git@github.com
```

提示 `successfully authenticated`  则成功。

## 创建本地仓库

- cd到项目目录
- `git init` 初始化git仓库
- `git add .` 把所有项目文件添加到提交暂存区
- `git commit -m '提交说明'` 把暂存区中的内容提交到仓库

## 创建远程仓库

github新建仓库，假设仓库名为[resName]

## 将本地仓库同步到远程仓库

### 添加远程仓库

```sh
git remote add origin git@github.com:[githubUerName]/[resName]
```

### 将本地仓库的内容push到远程仓库的master分支

```sh
git push -u origin master
```

`push`的`-u`参数是设置本地仓库默认的`upstream`,这里就是把本地仓库同远程仓库的master分支进行关联，之后你在这个仓库pull时不带参数也默认从master分支拉取.

