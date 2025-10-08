---
title: "第一篇 blog ，纪录部署 hugo 过程"
date: 2022-09-25T10:23:16Z
tags: 
  - hugo
  - docker-compose
  - blog
categories: IT技术
author: Bonfire8458
draft: false
---
嗯呢，一直以来想搭建一个 blog ，纪录一些东西。

这是 blog 搭建的一些简略纪录，作为第一篇好像也还不错。

<!--more-->


# 一、编写 dockerfile ，构建基础镜像

docker 和 docker-compose 安装配置略.

首先新建一个文件夹存放项目文件。

`<>` 尖括号需要替换成自己的。

```dockerfile
#  使用采用了debian 做基镜的 golong 镜像作为构建基础
FROM golang:1.19.1-bullseye

#  network=hosts 时，暴露的端口
EXPOSE 1313

#  使用中科大源
#  创建普通用户，使用普通用户运行
#  使用宿主机的 group_id、group_name、user_id、user_name，方便后面使用宿主密钥文件
#  更新源缓存
#  安装 vim、hugo
#  配置 encoding=utf-8 防止 vim 中文乱码
#  创建 /src 目录，存放 hugo
#  更改 /src 目录权限为运行用户权限
RUN sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list \
    && groupadd -g <group_id> <group_name> && useradd -u <user_id> -g <group_name> <user_name> -m \
    && apt-get update -y \
    && apt-get install vim hugo -y \
    && echo "set encoding=utf-8" >> /etc/vim/vimrc \
    && mkdir -p /src \
    && chown -R <user_name>: <group_name> /src

#  复制 shell 初始化文件到 ～/init
COPY init /home/<user_name>/init

#  指定用户运行
USER <user_name>

# 指定工作目录
WORKDIR /src

#  docker 容器启动后，运行的命令。运行 init.sh 初始化脚本，hugo 服务器监听 0.0.0.0 ，指定 hugo 测试环境 baseURL。
CMD bash ~/init/init.sh && hugo server --buildDrafts --bind="0.0.0.0" --baseURL=http://<your_ip>:1313
```

# 二、编写 `./init/init.sh`
```bash
#  判断空文件夹函数，防止重启容器导致数据被删除
is_empty_dir(){
    return `ls -A $1|wc -w`
}
if is_empty_dir '/src' ; then
    # 判断为空，则为新容器。
    hugo new site /src
    cd /src/
fi
```

# 三、编写 docker-compose
```yaml
version: '3'
services:
  blog:
    build:
      context: .
    ports:
      - "1313:1313"
    volumes:
      - ~/.ssh:/home/<user_name>/.ssh
    restart: always
    container_name: blog
    privileged: false
    user: <user_name>
```


# 四、配置hugo
## 配置 ssh 密钥和 创建 github 仓库
宿主机  `<user_name>` 用户密钥配置到 github。这样配置容器使用宿主机密钥，可以让其他 docker 项目可以使用同一个密钥。

在 github 创建名为`<your_github_name>.github.io`的空仓库，不能包含 README.md 文件

## 运行docker
```bash
# 构建并运行，成功后显示日志
docker-compose up -d --build && docker-compose logs -f
# 无报错成功后，按 Ctrl + C  退出日志显示，然后进入容器
docker-compose exec blog bash
```
## 配置 git 并初始化
```bash
# 初始化git
cd /src/
git config --global user.name "用户名"
git config --global user.email "自己的邮箱"
git init
# 验证本地仓库与Github之间传输是否成功
ssh -T git@github.com
# 安装主题并添加为 submodule
git submodule add https://github.com/vaga/hugo-theme-m10c.git themes/m10c
#配置添加 github 仓库
git remote add github git@github.com:<your_github_name>/<your_github_name>.github.io.git
```
## 配置hugo，编辑 config.toml，指定主题为 m10c，并进行其他配置
```bash
# 配置 hugo ，
vim config.toml
# 新建一个about页面，并写一些东西，草稿状态 draft 改为 false，theme = "m10c"。
hugo new about.md
# 生成html文件
hugo
```

## git 上传 github 仓库
```bash
# 创建  README.md 文件 ， baseURL 填写 <your_github_name>.github .io
echo "# <your_github_name>.github.io" >> README.md
# 添加本文件夹下所有文件
git add .
# 运行git status 指令可看到文件被跟踪处于暂存状态
git status
# 本地提交
git commit -m "first commit"
# 推送到 github
git push -u github master

```

# 五、发布 blog
进入 `<your_github_name>`.github.io 仓库，进入setting-->pages-->Build and deployment-->Github Actions-->save-->不需要自己编写 deploy 文件，可以选择hugo模板，然后保存提交到github仓库。

# 六、修改`./init/init.sh`，以方便迁移
```bash
# 判断空文件夹函数，防止重启容器导致数据被删除
is_empty_dir(){
    return `ls -A $1|wc -w`
}
if is_empty_dir '/src' ; then
    # 判断为空，则为新容器。
    git config --global user.name "用户名"
    git config --global user.email "自己的邮箱"
    cd /src/
    # 克隆 github 上的 hugo
    git clone  git@github.com:<your_github_name>/<your_github_name>.github.io.git  ../src/
    # 初始化引入的 m10c 模块
    git submodule update --init --recursive
fi
```

# 七、结束
## 备份与迁移
这样，本地只需要备份 dockerfile 、docker-compose.yml 、init/init.sh 三个文件/文件夹就可以迁移部署到有 docker 的机器上了。

重新部署只需要运行 `docker-compose up -d --build` 就可以了。

把这三个文件提交到私有 git 仓库里面。

## 写文章
写文章不太方便，需要使用vim，后续可能需要改用 python 的镜像做基础版本，写一个页面。在这点上 hugo 不太方便，hexo 有 hexo-admin 可以使用可是方便多了。

# 评论系统使用推特
{{< X 1574019861847502848>}}
