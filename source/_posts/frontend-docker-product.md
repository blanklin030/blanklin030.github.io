---
title: 前端基于docker的生产环境部署方案
date: 2020-07-20 19:28:55
tags:
  - docker
categories:
  - tool
---
+ docker安装
    + brew install
        ```
        brew install docker
        ```
    + Docker Desktop install
   [download](https://hub.docker.com/editions/community/docker-ce-desktop-mac/)
+ Dockerfile  

```
# 基础镜像
FROM node as build

# 设置工作目录
WORKDIR /app

# 把 `/app/node_modules/.bin` 加到$PATH中
ENV PATH /app/node_modules/.bin:$PATH

# 安装并缓存应用依赖
COPY ./package.json /app/package.json
COPY ./package-lock.json /app/package-lock.json
# 切换npm源
RUN npm config set registry https://registry.npm.taobao.org
# 安装npm包
RUN npm install

# 编译应用
COPY . /app
RUN npm run build

# 删除无用文件
RUN rm -fr /app/src
RUN rm -fr /app/public
RUN rm -fr /app/node_modules

# production environment
FROM nginx
COPY --from=build /app/build /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/conf.d
EXPOSE 8082
CMD ["nginx", "-g", "daemon off;"]
```  

+ nginx.conf配置  

```
server {

  listen 8082;
  # 访问根目录强制到index.html
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
  }

  # 访问v1前缀都反向代理到server端地址
  location ~ ^/v1/ {
    rewrite ^/(.*)$ /$1 break;
    proxy_pass http://server:8081;
  }
}
```  
+ Makefile  

```
.PHONY: clean build

SHELL := /bin/bash

NAME=aere
VERSION=0.1
MODULE=git.xxx.com/xx/frontend
BUILD_DATE=$(shell date '+%Y%m%d')
BUILD_VERSION=$(shell git log -1 --pretty=format:%h)
REGISTRY=10.xx.xx.17:8001

release:
	docker build --no-cache -f Dockerfile -t $(REGISTRY)/$(NAME)/frontend:$(BUILD_DATE)-$(BUILD_VERSION) .
	docker tag $(REGISTRY)/$(NAME)/frontend:$(BUILD_DATE)-$(BUILD_VERSION) $(REGISTRY)/$(NAME)/frontend

push:
	docker push $(REGISTRY)/$(NAME)/frontend:$(BUILD_DATE)-$(BUILD_VERSION)
	docker push $(REGISTRY)/$(NAME)/frontend
```  
+ how to use  

    + 本地开发环境编排容器
    ```
    make release
    make push
    ```
    + 生产环境运行docker
    ```
        # 注意docker run的参数,具体参数定义自行查找
        docker run -it --rm -p 8082:8082 10.xx.xx.17:8001/aere/frontend
    ```
    
    + 浏览器访问
    ```
    http://ip:8082
    ```

+ 问题与思考
    + npm打包太慢，每次build都得好几分钟，不知道各位有没有好办法进行优化
    + 可以使用docker compose进行编排，就不需要每次在生产环境输入长串命令