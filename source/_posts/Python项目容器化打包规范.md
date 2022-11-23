title: Python项目容器化打包规范
author: 几时西风
tags:
  - docker
  - python
categories:
  - docker
date: 2022-11-23 16:55:00
---
## 引言

一个成熟的python项目可能会依赖很多特定的环境，然而项目运行的结果不仅取决于代码，和运行代码的环境也息息相关。这很有可能会造成，开发环境上的运行结果和测试环境、线上环境上的结果都不一致的现象。为了解决这个问题，我们可以将python项目打包成docker镜像，这样即使在不同的机器上运行打包后的项目，我们也能够得到一致的运行结果。



## 准备工作

### 安装docker

可以使用如下脚本一键安装

```shell
sudo curl -sSL https://get.daocloud.io/docker | sh
```

也可参考 CentOS Docker基础环境安装



### 准备python项目

在项目根目录下添加以下两个文件

- requirements.txt 该文件描述了python项目的依赖环境，使用如下命令即可导出相关库的信息并自动生成依赖文件

  ```shell
  pip freeze > requirements.txt
  ```

- Dockerfile 

  ```yaml
  # 将官方 Python 运行时用作父镜像，根据需要选择python版本
  FROM python:3.7
  # 记录维护这信息
  MAINTAINER zhangjie@parkson.com.cn
  # 将工作目录设置为 /myapps
  WORKDIR ./myapps
  # 将当前目录内容复制到位于 /myapps 中的容器中
  ADD . .
  # 安装 requirements.txt 中指定的任何所需软件包
  RUN pip install -r requirements.txt
  # 使用匿名卷持久化存储日志或者其它数据，匿名卷实际地址在docker安装目录下，可以用docker inspect查看
  VOLUME [/logs1,/logs2]
  # 定义环境变量
  ENV name1=var1 name2=var2
  # 暴露端口配置，默认TCP,可指定为udp
  EXPOSE [80/tcp,80/udp]
  # 在容器启动时运行 hello_world.py
  ENTRYPOINT ["python", "./myapps/hello_world.py"]
  ```

  note:有时候pip install -r requirements.txt很慢，可以考虑换源，只需将原命令改为如下即可

  ```shell
  RUN pip install -r requirements.txt -i https://pypi.douban.com/simple
  ```



## 打包镜像

在Dockerfile所在的目录下运行

```shell
sudo docker build -t demo:v1 .
```

其中demo为镜像名称，v1为镜像tag用于版本管理



## 运行容器

```shell
sudo docker run demo:v1
```

其它常用参数，根据需要使用：

- -d: 后台运行容器，并返回容器ID；
- -v /opt/logs:/logs  挂载目录,将容器中的目录映射到宿主机中，格式为宿主机目录:容器中的目录
- -p 80：80  指定端口映射，格式为：主机(宿主)端口:容器端口
- -e username="ritchie"   设置环境变量；
- --env-file .env  使用.env文件设置环境变量
- --name nginx1.18 为容器指定一个名称；
- --link mysql:mysql  添加链接到另一个容器；



## 标准化自动打包、自动部署流程

本规范仅专注于如何打包python容器，如果需要结合jenkins自动化部署流程，则需要参照以下步骤

### 添加docker.sh文件

需要在项目根目录添加docker.sh文件，参照如下:

> 设置镜像名称BUILD_IMAGE_NAME
> 不需要设置tag，统一使用构建时间生成，构建成功后保存在.docker_build_version文件中
> 根据项目设置run()函数容器的映射端口

```sh
#!/usr/bin/env bash

# 获取脚本所在文件件
BUILD_DIRECTORY=$(dirname $(readlink -f "$0"))

# docker仓库相关信息
BUILD_REGISTRY=$DOCKER_REGISTRY
BUILD_REGISTRY_USER=$DOCKER_REGISTRY_USER
BUILD_REGISTRY_PASS=$DOCKER_REGISTRY_PASS

# docker生产仓库相关信息
BUILD_REGISTRY_PROD=$DOCKER_REGISTRY_PROD
BUILD_REGISTRY_PROD_USER=$DOCKER_REGISTRY_PROD_USER
BUILD_REGISTRY_PROD_PASS=$DOCKER_REGISTRY_PROD_PASS

# 命名空间
BUILD_NAMESPACE=""

# 镜像名称
BUILD_IMAGE_NAME=aglaia-web-re

if [ "$2" ]; then
  BUILD_IMAGE_NAME=$2
fi

# 镜像版本
BUILD_VERSION=""

# 最终镜像名
BUILD_IMAGE=$BUILD_IMAGE_NAME
if [ "$BUILD_NAMESPACE" ]; then
  BUILD_IMAGE=$BUILD_NAMESPACE/$BUILD_IMAGE
fi

# 检查docker仓库配置
check_docker_registry() {
  if [ "$BUILD_REGISTRY" == "" ]; then
    echo "请配置docker仓库环境变量：DOCKER_REGISTRY"
    exit 1
  fi
}

# 检查docker仓库认证信息
check_docker_registry_user_and_pass() {
  if [ "$BUILD_REGISTRY_USER" == "" -o "$BUILD_REGISTRY_PASS" == "" ]; then
    echo "请配置docker仓库认证信息环境变量：DOCKER_REGISTRY_USER、DOCKER_REGISTRY_PASS"
    exit 1
  fi
}

# 检查docker生产仓库配置
check_docker_registry_prod() {
  if [ "$BUILD_REGISTRY_PROD" == "" ]; then
    echo "请配置生产docker仓库环境变量：DOCKER_REGISTRY_PROD"
    exit 1
  fi
}

# 检查docker生产仓库认证信息
check_docker_registry_prod_user_and_pass() {
  if [ "$BUILD_REGISTRY_PROD_USER" == "" -o "$BUILD_REGISTRY_PROD_PASS" == "" ]; then
    echo "请配置生产docker仓库认证信息环境变量：DOCKER_REGISTRY_PROD_USER、DOCKER_REGISTRY_PROD_PASS"
    exit 1
  fi
}

# 检查镜像
check_image() {
  if [ "$BUILD_IMAGE" == "" ]; then
    echo "请配置镜像名称：BUILD_IMAGE_NAME，或通过参数指定 docker.sh <command> <image>"
    exit 1
  fi
}

# 检查版本
check_version() {
  if [ "$BUILD_VERSION" == "" ]; then
    echo "请先执行build操作"
    exit 1
  fi
}

# 生产版本号
gen_version() {
  echo $(date "+%Y%m%d%H%M%S")
}

# 获取版本号
get_version() {
  local FILE=$BUILD_DIRECTORY/.docker_build_version
  if [ -f "$FILE" ]; then
    echo $(cat "$FILE")
  else
    echo ""
  fi
}

# 当需要时执行构建
build_when_necessary() {
  if [ "$(get_version)" == "" ]; then
    build
  fi
}

# 构建镜像
build() {
  echo "构建镜像"
  check_docker_registry
  check_image
  BUILD_VERSION=$(gen_version)
  check_version
  docker build --no-cache --force-rm $BUILD_DIRECTORY -t $BUILD_REGISTRY/$BUILD_IMAGE:$BUILD_VERSION
  if [ "$?" != "0" ]; then
    echo "构建镜像失败"
    exit 1
  fi
  echo $BUILD_IMAGE >$BUILD_DIRECTORY/.docker_build_image
  echo $BUILD_VERSION >$BUILD_DIRECTORY/.docker_build_version
  echo "构建镜像成功"
}

# 执行镜像
run() {
  echo "执行镜像"
  check_docker_registry
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker run --rm -p 80:80 $BUILD_REGISTRY/$BUILD_IMAGE:$BUILD_VERSION
  if [ "$?" != "0" ]; then
    echo "执行镜像失败"
    exit 1
  fi
  echo "执行镜像成功"
}

# 推送镜像
push() {
  echo "推送镜像"
  check_docker_registry
  check_docker_registry_user_and_pass
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker login $BUILD_REGISTRY -u $BUILD_REGISTRY_USER -p $BUILD_REGISTRY_PASS
  if [ "$?" != "0" ]; then
    echo "登录docker仓库失败"
    exit 1
  fi
  docker push $BUILD_REGISTRY/$BUILD_IMAGE:$BUILD_VERSION
  if [ "$?" != "0" ]; then
    echo "推送镜像失败"
    exit 1
  fi
  echo "推送镜像成功"
}

# 标记为最新版本
tag_latest() {
  echo "标记为最新版本"
  check_docker_registry
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker tag $BUILD_REGISTRY/$BUILD_IMAGE:$BUILD_VERSION $BUILD_REGISTRY/$BUILD_IMAGE:latest
  if [ "$?" != "0" ]; then
    echo "标记为最新版本失败"
    exit 1
  fi
  echo "标记为最新版本成功"
}

# 推送最新版本
push_latest() {
  echo "推送最新版本"
  check_docker_registry
  check_docker_registry_user_and_pass
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker login $BUILD_REGISTRY -u $BUILD_REGISTRY_USER -p $BUILD_REGISTRY_PASS
  if [ "$?" != "0" ]; then
    echo "登录docker仓库失败"
    exit 1
  fi
  docker push $BUILD_REGISTRY/$BUILD_IMAGE:latest
  if [ "$?" != "0" ]; then
    echo "推送最新版本失败"
    exit 1
  fi
  echo "推送最新版本成功"
}

# 标记为生产版本
tag_prod() {
  echo "标记为生产版本"
  check_docker_registry
  check_docker_registry_prod
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker tag $BUILD_REGISTRY/$BUILD_IMAGE:$BUILD_VERSION $BUILD_REGISTRY_PROD/$BUILD_IMAGE:$BUILD_VERSION
  if [ "$?" != "0" ]; then
    echo "标记为生产版本失败"
    exit 1
  fi
  echo "标记为生产版本成功"
}

# 推送生产版本
push_prod() {
  echo "推送生产版本"
  check_docker_registry
  check_docker_registry_prod
  check_docker_registry_prod_user_and_pass
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker login $BUILD_REGISTRY_PROD -u $BUILD_REGISTRY_PROD_USER -p $BUILD_REGISTRY_PROD_PASS
  if [ "$?" != "0" ]; then
    echo "登录docker生产仓库失败"
    exit 1
  fi
  docker push $BUILD_REGISTRY_PROD/$BUILD_IMAGE:$BUILD_VERSION
  if [ "$?" != "0" ]; then
    echo "推送生产版本失败"
    exit 1
  fi
  echo "推送生产版本成功"
}

# 标记为生产最新版本
tag_prod_latest() {
  echo "标记为生产最新版本"
  check_docker_registry
  check_docker_registry_prod
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker tag $BUILD_REGISTRY/$BUILD_IMAGE:$BUILD_VERSION $BUILD_REGISTRY_PROD/$BUILD_IMAGE:latest
  if [ "$?" != "0" ]; then
    echo "标记为生产最新版本失败"
    exit 1
  fi
  echo "标记为生产最新版本成功"
}

# 推送生产最新版本
push_prod_latest() {
  echo "推送生产最新版本"
  check_docker_registry
  check_docker_registry_prod
  check_docker_registry_prod_user_and_pass
  check_image
  BUILD_VERSION=$(get_version)
  check_version
  docker login $BUILD_REGISTRY_PROD -u $BUILD_REGISTRY_PROD_USER -p $BUILD_REGISTRY_PROD_PASS
  if [ "$?" != "0" ]; then
    echo "登录docker生产仓库失败"
    exit 1
  fi
  docker push $BUILD_REGISTRY_PROD/$BUILD_IMAGE:latest
  if [ "$?" != "0" ]; then
    echo "推送生产最新版本失败"
    exit 1
  fi
  echo "推送生产最新版本成功"
}

RC=0

case "$1" in
build)
  build
  ;;
run)
  run
  ;;
push)
  push
  ;;
tag-latest)
  tag_latest
  ;;
push-latest)
  push_latest
  ;;
tag-prod)
  tag_prod
  ;;
push-prod)
  push_prod
  ;;
tag-prod-latest)
  tag_prod_latest
  ;;
push-prod-latest)
  push_prod_latest
  ;;
pipeline)
  build_when_necessary
  push
  tag_latest
  push_latest
  ;;
pipeline-prod)
  build_when_necessary
  tag_prod
  push_prod
  tag_prod_latest
  push_prod_latest
  ;;
pipeline-all)
  build_when_necessary
  push
  tag_latest
  push_latest
  tag_prod
  push_prod
  tag_prod_latest
  push_prod_latest
  ;;
pipeline-rebuild)
  build
  push
  tag_latest
  push_latest
  ;;
pipeline-prod-rebuild)
  build
  tag_prod
  push_prod
  tag_prod_latest
  push_prod_latest
  ;;
pipeline-all-rebuild)
  build
  push
  tag_latest
  push_latest
  tag_prod
  push_prod
  tag_prod_latest
  push_prod_latest
  ;;
*)
  echo $"Usage 1: $0 {build|run|push|tag-latest|push-latest|tag-prod|push-prod|tag-prod-latest|push-prod-latest}"
  echo $"Usage 2: $0 {pipeline|pipeline-prod|pipeline-all|pipeline-rebuild|pipeline-prod-rebuild|pipeline-all-rebuild}"
  echo $"Tips  1: 使用前请设置脚本变量：BUILD_NAMESPACE、BUILD_IMAGE_NAME"
  echo $"Tips  2: 使用前请设置环境变量：DOCKER_REGISTRY、DOCKER_REGISTRY_USER、DOCKER_REGISTRY_PASS、DOCKER_REGISTRY_PROD、DOCKER_REGISTRY_PROD_USER、DOCKER_REGISTRY_PROD_PASS"
  exit 2
  ;;
esac

exit $RC
```



### 赋予docker.sh执行权限

```shell
sudo chmod a+x docker.sh
```



### 设置dockerhub相关环境变量

> 这些环境变量在docker.sh中会用到，并且所有docker打包发布相关操作都会用到，因此强烈建议直接配置到/etc/profile

```
# docker仓库相关信息
export DOCKER_REGISTRY=docker.parkson.net.cn
export DOCKER_REGISTRY_USER=<账号>
export DOCKER_REGISTRY_PASS=<密码>

# docker生产仓库相关信息
export DOCKER_REGISTRY_PROD=docker-registry.parkson.net.cn
export DOCKER_REGISTRY_PROD_USER=<账号>
export DOCKER_REGISTRY_PROD_PASS=<密码>
```



### docker.sh使用方法

#### 查看使用方法

```
./docker.sh
```

#### 构建镜像

```
./docker.sh build
```

#### 运行镜像

```
./docker.sh run
```

#### 推送镜像

```
./docker.sh push
```

#### 标记为最新版本

```
./docker.sh tag-latest
```

#### 推送最新版本

```
./docker.sh push-latest
```

#### 标记为生产版本

```
./docker.sh tag-prod
```

#### 推送生产版本

```
./docker.sh push-prod
```

#### 标记为生产最新版本

```
./docker.sh tag-prod-latest
```

#### 推送生产最新版本

```
./docker.sh push-prod-latest
```

#### 执行流水线作业

> build(如果已经构建不会重新构建)->push->tag-latest->push-latest

```
./docker.sh pipeline
```

#### 执行生产流水线作业

> build(如果已经构建不会重新构建)->tag-prod>push-prod->tag-latest->push-latest

```
./docker.sh pipeline-prod
```
#### 执行全部流水线作业

> build(如果已经构建不会重新构建)->push->tag-latest->push-latest->tag-prod>push-prod->tag-latest->push-latest

```
./docker.sh pipeline-all
```

#### 执行流水线作业（强制重新构建镜像）

> build(每次都重新构建镜像)->push->tag-latest->push-latest

```
./docker.sh pipeline-rebuild
```

#### 执行生产流水线作业（强制重新构建镜像）

> build(每次都重新构建镜像)->tag-prod>push-prod->tag-latest->push-latest

```
./docker.sh pipeline-prod-rebuild
```
#### 执行全部流水线作业（强制重新构建镜像）

> build(每次都重新构建镜像)->push->tag-latest->push-latest->tag-prod>push-prod->tag-latest->push-latest

```
./docker.sh pipeline-all-rebuild
```



### 项目相关环境变量配置

在项目根目录下添加.env文件（用于设置环境变量，更多环境变量请根据项目进行设置）

> .env

```shell
# dev、test、prod
ENV=dev
# DBURI
HOST=10.88.1.12
```



### 添加docker-compose.yml文件

在项目根目录下添加docker-compose.yml文件

> docker-compose.yml
> 设置镜像名称（格式：项目代号，形如xxx-yyy-zzz，例如aglaia-ui-re）
> 容器名称与镜像名称保持一致
> 主机名称与镜像名称保持一致
> 根据项目设置容器的映射端口
> 大部分基底容器默认不支持中文，为了保持统一只设置时区为Asia/Shanghai，语言方面以英语作为默认语言

```yaml
version: "3.5"
services:
  <项目代号>:
    image: ${DOCKER_REGISTRY:-docker.parkson.net.cn}/<项目代号>:${VERSION:-latest}
    container_name: <项目代号>
    hostname: <项目代号>
    ports:
      - 8080:8080
    ulimits:
      memlock:
        soft: -1
        hard: -1
    env_file:
      - .env
    environment:
      - TZ=Asia/Shanghai
      - LANG=en_US.UTF-8
      - LANGUAGE=en_US:en
      - LC_ALL=en_US.UTF-8
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "128m"
        max-file: "20"
```



### docker-compose.yaml使用方法

该文件主要用于部署配合Jenkins自动化部署使用

> 将docker-compose.yml上传到服务器的"/docker/compose/<项目代号>/"目录下

#### 启动容器

> 前台

```
docker-compose up
```

#### 启动容器

> 后台

```
docker-compose up -d
```

#### 销毁容器

```
docker-compose down
```

#### 更新镜像

```
docker-compose pull
```

#### 查看日志

> 查看最后500行日志

```
docker-compose logs --tail 500
```

> 查看最后500行日志（滚动刷新）

```
docker-compose logs --tail 500 -f
```