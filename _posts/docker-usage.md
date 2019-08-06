---
title: Docker 常用命令
date: 2019-04-12 15:18:31
categories:
- 笔记
tags:
- Docker
---

list docker cli commands

`docker --help`

`docker contianer --help`

docker version

`docker --version`

<!-- more -->

<!-- toc -->

docker info

`docker info`

list image

`docker image ls` or `docker images`

list container

`docker container ls`

`docker container ls --all` # all mode

`docker container ls -aq` # all in quite mode

run image

`docker run hello-world`

`docker run -d -p 6379:6379 redis`

login running container

`docker exec -it <name> bash`
