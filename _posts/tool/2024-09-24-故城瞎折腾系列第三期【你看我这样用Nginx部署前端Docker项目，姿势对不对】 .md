---
title: 故城瞎折腾系列第三期【你看我这样用Nginx部署前端Docker项目，姿势对不对】
date: 2024-09-24
categories: [tool]
tags: [docker,工具,部署]
description: 如何利用Nginx + Docker部署前端项目
---

## 1. 背景
- 一次偶然的机会公司要升级部署机制，整体部署流程进行升级：jenkins部署升级为pipeline + k8s + docker
- 使用容器化部署在部署和回滚上面，都会更方便管理，尤其是环境变量众多，且在部署多集群和命名空间时非常有优势
- 其实代码能跑就行，项目能部署就行，都是看公司决策，我们先做好自己本职工作就行


## 2. 导语
- 此篇文章分享如何利用 Nginx + Docker 管理前端项目部署(个人自用版，企业参考公司部署体系)
- 其实我也没有探究过公司具体的企业部署流程，只是公司部署人员告诉我们提供前端项目Dockerfile，以及前端环境变量如何利用Docker注入，其他的就交给他们了，然后我抽象了 Docker 容器，以及如何接入的一些配置供自己使用，这绝对是全网首个这么去利用 Docker 部署前端项目 + 结合 Docker 环境变量注入前端项目环境变量的，发出来也只是给大家交流^_^

以下是我的【故城瞎折腾系列】文章

> 第一篇：[故城瞎折腾系列第一期【你要不要动手封装个前端Docker容器玩一玩】](https://juejin.cn/post/7414045785636454440) <br>
> 第二篇：[故城瞎折腾系列第二期【都2024年了，你还在手动部署前端项目吗】](https://juejin.cn/post/7416922427156348963)<br>
> 第三篇：故城瞎折腾系列第三期【你看我这样用Nginx部署前端Docker项目，姿势对不对】


## 3. 首先在服务器上安装 Nginx + Docker

安装 Nginx(以 centos 为例子)
```bash
# 确保系统包是最新的
sudo yum update -y

# 安装 EPEL 仓库
sudo yum install epel-release -y

# 从 EPEL 仓库安装 Nginx
sudo yum install nginx -y

# 启动并启用 Nginx
sudo systemctl start nginx

# 将 Nginx 设置为开机自启动
sudo systemctl enable nginx

# 验证 Nginx 状态
sudo systemctl status nginx

# 如果你的 CentOS 系统启用了防火墙（如果需要），你需要开放 HTTP (80) 和 HTTPS (443) 端口
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# 通过浏览器访问服务器的 IP 地址或 localhost 来查看 Nginx 欢迎页面
# http://your_server_ip
```

安装 Docker(以 centos 为例子)
```bash
# 更新系统并移除旧版本
sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

# 安装依赖的工具和包以确保 Docker 能够正确安装
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# 使用 yum-config-manager 添加 Docker 官方的仓库
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 从官方仓库安装 Docker
sudo yum install docker-ce docker-ce-cli containerd.io -y

# 安装完成后，启动 Docker 并设置为开机自启动
sudo systemctl start docker
sudo systemctl enable docker

# 检查 Docker 是否已成功安装
sudo docker --version
```


## 4. 用 Docker 部署前端项目

### 4.1. 手动运行前端项目镜像

- 构建 Docker 镜像并推送到 github 容器中心化仓库（[需要提前配置好 Github-Token](https://juejin.cn/post/7414045785636454440#heading-6)）
- 你也可以使用 `docker build` 或者 `docker buildx` [本地构建 Docker 镜像](https://github.com/rookie-luochao/nginx-runner?tab=readme-ov-file#%E6%89%93%E5%8C%85%E6%9E%84%E5%BB%BA%E9%95%9C%E5%83%8F)

```yml
name: Docker Image CI

on:
  push:
    tags:
      - v*

  # 这个选项可以使你手动在 Action tab 页面触发工作流
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: get version
        id: vars
        run: echo ::set-output name=version::${GITHUB_REF/refs\/tags\/v/}

      - uses: actions/checkout@v4

      - name: set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: set up docker buildx
        uses: docker/setup-buildx-action@v3

      - name: login ghrc hub
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GHCR_TOKEN }}

      - name: build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          platforms: linux/amd64,linux/arm64
          tags: |
            ghcr.io/${{ github.repository_owner }}/webapp:${{ steps.vars.outputs.version }}
            ghcr.io/${{ github.repository_owner }}/webapp:latest
```

以 Docker 环境变量注入模式运行前端镜像（以 `ghcr.io/rookie-luochao/openapi-ui` 为例子）：
```bash
# pull Docker image
docker pull ghcr.io/rookie-luochao/openapi-ui:latest

# start container
docker run -d -p 3000:80 -e APP_CONFIG=env=zh,appNameZH=简洁美观的接口文档 ghcr.io/rookie-luochao/openapi-ui:latest
```


### 4.2. 利用 github-action 自动连接服务器并运行前端项目镜像

Docker 镜像构建完成后利用 appleboy/ssh-action 进行远程自动部署（需要提前设置 github 项目的环境变量）：
```yml
name: Deploy CI

on:
  workflow_run:
    workflows: [Docker Image CI]
    types:
      - completed

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to remote server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.REMOTE_HOST }}
          username: ${{ secrets.REMOTE_USERNAME }}
          password: ${{ secrets.REMOTE_PASSWORD }}
          port: ${{ secrets.REMOTE_PORT }}
          command_timeout: 30m
          script: |
            docker login ghcr.io -u rookie-luochao -p ${{ secrets.GHCR_TOKEN }}
            echo "---------docker login success--------"
            docker pull ghcr.io/rookie-luochao/openapi-ui
            echo "---------docker pull success--------"
            docker stop ${{ secrets.IMAGE_NAME }}
            echo "---------docker stop container success--------"
            docker rm ${{ secrets.IMAGE_NAME }}
            echo "---------docker rm container success--------"
            docker run -dp ${{ secrets.HOST_PORT }}:80 -e ${{ secrets.ENVS }} --name ${{ secrets.IMAGE_NAME }} ghcr.io/rookie-luochao/openapi-ui
            echo "---------docker deploy success--------"
```


### 4.3. 开放服务器端口，并验证前端项目是否成功启动

如果用下面的命令运行前端镜像，我需要开方服务器的 3000 端口，并且前端服务启动成功后，在浏览器输入ip地址+端口号进行验证是否启动成功：`http://your_server_ip:3000`
```bash
docker run -d -p 3000:80 -e APP_CONFIG=env=zh,appNameZH=简洁美观的接口文档 ghcr.io/rookie-luochao/openapi-ui:latest
```


## 5. 购买域名和域名证书

- [前往阿里云购买域名](https://wanwang.aliyun.com/domain)
- [前往腾讯云购买域名](https://buy.cloud.tencent.com/domain)
- [前往华为云购买域名](https://www.huaweicloud.com/product/domain.html)


## 6. 使用 Nginx 配置域名端口解析并转发到 Docker 前端服务

- 阿里云域名控制台配置好域名解析规则，使用 `A` 记录类型将域名解析指向到自己的服务器 IP 地址
- 下载域名证书
- 在服务器 `etc/nginx/nginx.conf` 文件增加如下域名转发配置

```
	server {
	    listen       80;
	    server_name  www.openapi-ui.com;
	    return 301 https://$server_name$request_uri;
	}

	server {
	    listen       443 ssl;
	    server_name  www.openapi-ui.com;

          ssl_certificate     /etc/nginx/ssl/openapi-ui/openapi-ui.com_bundle.crt;
    	    ssl_certificate_key /etc/nginx/ssl/openapi-ui/openapi-ui.com.key;
    	    
    	    ssl_protocols TLSv1.2 TLSv1.3;
    	    ssl_prefer_server_ciphers off;
    	    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';
    	    ssl_session_cache shared:SSL:10m;
    	    ssl_session_timeout 10m;
	
	    location / {	        
	        proxy_set_header Host $proxy_host;
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

	        proxy_buffering on;
		      proxy_buffers 8 16k;
		      proxy_buffer_size 16k;

	        # 启用 Gzip 压缩
	        gzip on;
	        gzip_min_length 1000;
	        gzip_types text/plain application/javascript application/x-javascript text/javascript text/xml text/css;
	        # 选择性地设置 Gzip 压缩等级
	        gzip_comp_level 6;
	        # 启用 Gzip 静态文件预压缩
	        gzip_static on;
	
	        # 启用 Gzip 压缩代理响应
	        proxy_http_version 1.1;
	        proxy_set_header Upgrade $http_upgrade;
	        proxy_set_header Connection 'upgrade';

	        proxy_pass http://127.0.0.1:3000;
	    }
	}

```

## 7. 结语
* 介绍了如何在服务器上安装 Nginx + Docker
* 介绍了如何用 Docker 部署前端项目
* 介绍了如何使用 Nginx 配置域名端口解析并转发到 Docker 前端服务
* 看都看完了，还不动手操作一波
