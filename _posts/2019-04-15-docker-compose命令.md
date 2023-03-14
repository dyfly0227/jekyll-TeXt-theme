---
title: 'docker compose命令'
excerpt: 'docker compose常用命令'
tags: 'docker'
---

##### 生成容器并启动
```bash
# 第一次生成或者镜像有变动，必须要有--build
docker-compose up -d --build
```

##### 查看容器
```bash
# -aq 查看所有容器，查看运行容器去掉
docker ps -aq
```
##### 停止容器
```bash
docker stop 容器id
# 停止所有容器
docker stop $(docker ps -aq)
```
##### 重启容器
```bash
docker restart 容器id
# 重启所有容器
docker restart $(docker ps -aq)
```
##### 删除容器
```bash
docker rm 容器id
# 删除所有容器
docker rm $(docker ps -aq)
# 删除所有停止的容器
docker container prune
```
##### 进入容器
```bash
docker exec -it 容器id /bin/bash
```
##### 容器日志
```bash
docker logs 容器id
```
##### 查看镜像
```bash
docker images
```
##### 删除镜像
```bash
docker rmi 镜像id
# 删除所有镜像
docker rmi $(docker images -q)
# 删除所有未使用的镜像
docker image prune --force --all或者docker image prune -f -a
```
##### 复制文件
```bash
 docker  cp 容器名:  容器路径       宿主机路径 
 docker cp 宿主路径中文件      容器名  容器路径   
```
