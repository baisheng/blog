---
title: 折腾 Bitbucket Pipeline + Docker + Gradle 自动部署
date: 2018-11-23 00:23:00
tags: [docker, gradle, springboot, bitbucket]
---

> 一直依赖部署项目都是非常令我苦恼的事情. 每次要部署的时候都偷懒觉得部署一下得了, 懒得折腾自动部署(持续集成), 但是等到下次要部署的时候就得再头疼一次.

## 构想

今天咬咬牙, 一定要配置好自动部署. 由于自己的需求比较简单, 就是部署一下单机的服务, 所以也没有上Jenkins, 没必要大炮打蚊子. 刚好代码托管在 Bitbucket 上, 就使用了 Bitbucket Pipeline 服务, 可以定义一些简单的工作流任务, 每次 push 代码之后可以自动构建和部署.

*(Bitbucket Pipeline 免费额度是每月 50 分钟的构建时间, 对于个人开发者应该是够了的, 但是也尽量不要在 pipeline 上执行耗时操作)*

另外, 为了 防止部署过程中 build 发生错误 以及 未来可能的架构升级, 我是将项目打包成 docker 镜像, 将镜像独立托管, 然后让服务器通过拉去最新的 docker 镜像来进行更新代码.

## 零件

有了构想, 就要准备开始"组装玩具"了, 但是在这之前, 需要先确认一下手上的素材

- 代码托管平台+构建平台: Bitbucket
- docker 镜像托管: Google Container Registry (gcr.io)
- vps 实例一个
- 待部署软件: Gradle + 多模块 Spring Boot 项目

## 组装

### 配置 GCR

直接根据 Google 的 [文档](https://cloud.google.com/container-registry/docs/quickstart) 来为项目启用 gcr 即可.

### 配置 Gradle 打包 Docker

使用 [bmuschko/**gradle-docker-plugin**](https://github.com/bmuschko/gradle-docker-plugin) 来处理 Gradle Docker Build. 根据 [文档](https://bmuschko.github.io/gradle-docker-plugin/#spring_boot_application_plugin), 这个插件刚好支持零配置打包 Spring Boot 应用, 非常方便.

在项目根目录的`build.gradle`中引入插件

```groovy
buildscript {
    ext {
        ...
    	gradleDockerPluginVersion = '4.0.4' // 查看插件的 Github Release 页面获得最新版本
        ...
    }
    repositories {
        ...
        gradlePluginPortal()
        ...
    }
    dependencies {
		...
        classpath("com.bmuschko:gradle-docker-plugin:${gradleDockerPluginVersion}")
        ...
    }
}
```

在需要打包的子模块的`build.gradle`中使用插件

```groovy
apply plugin: 'com.bmuschko.docker-remote-api'
apply plugin: 'com.bmuschko.docker-spring-boot-application'
```

子模块的`build.gradle`中配置 Docker 打包选项

```groovy
docker {
    registryCredentials {
        url = 'https://gcr.io'
        username = '_json_key'
        password = file(project.rootDir.path + '/gcr_keyfile.json').text
    }
    springBootApplication {
        baseImage = 'openjdk:8-alpine'
        ports = [8080]
        tag = "gcr.io/project/image:" + version
    }
}
```

解释一下 registryCredentials, 这是用于配置镜像托管的节点. url 直接填写对应的相应的 gcr 地址, 有多个地址, 请根据实际托管位置决定. username 直接为 _json_key, 说明认证方式为 json 文件. password 是 keyfile.json 当中的内容, json 配置在项目根目录, 子模块 gradle 进行调用, 需要配置相对路径. 另外 keyfile.json 的获取方式请参考 gcr [文档](https://cloud.google.com/container-registry/docs/advanced-authentication#json_key_file)

而 springBootApplication 节点就是配置最终的镜像信息, 不多解释

这样就可以正常打包了, 如果你本机上安装了 docker 并且开启了本地 2375 端口, 可以直接进行测试

```shell
gradlew :app:dockerPushImage
```

### 配置 vps

需要在 vps 配置好 docker 则不必多说, 另外为了顺利拉去进行, 还需要配置好 GCloud SDK, 根据 Google [文档](https://cloud.google.com/container-registry/docs/pushing-and-pulling) 即可.

#### 配置自动部署脚本

仅仅使用 pipeline 进行 ssh 操作, 无法对发生错误的情况进行处理, 所以需要在主机上配置好一键部署脚本 `deploy.sh`, 到时候直接 ssh 执行这个脚本就好了

以下自定义`${需要自定义的内容} `

```shell
cd ~
vi deploy.sh
```

```shell
#!/usr/bin/env bash

if [[  "$(docker ps -q -f name=${container name})" ]]; then
    docker update --restart=no ${container name}
    docker stop ${container name}
    docker rm ${container name}
fi

docker pull ${image name}
docker run -d --restart always -p 8080:8080 --name ${container name} ${image name}
```

```shell
chmod 777 deploy.sh
```

好了, 接下去只要在主机上执行`~/deploy.sh`就能自动部署最新的镜像

### 连接 Bitbucket 和 计算实例

进入 Bitbucket 对应项目的 `设置`->`SSH keys` 中生成一个新的 SSH key, 复制 public key 并加入 vps 中.  

另外还需要将计算实例的 host 地址加入到 Bitbucket 的 Known hosts 里面.

### 编写 Pipeline 脚本

可以说这是最重要的一步, 因为是通过脚本来将之前准备的资源串联起来. 但是这又偏偏放在了最后, 反而觉得似乎没有那么重要.

由于 push 镜像的授权已经配置在 gradle 里面了, 所有就没必要在 pipeline 环境里面进行 gcr 授权了

所以整个脚本就分为两步

1. build + push 镜像
2. ssh 到 vps 上, 停止并删除正在运行的旧容器, 拉去新镜像并重新部署

```yaml
pipelines:
  default:
    - step:
        name: Deploy to Docker
        image: openjdk:8
        caches:
          - gradle
          - docker
        services:
          - docker
        script:
          - chmod +x gradlew
          - bash ./gradlew :app:dockerPushImage
    - step:
        name: Deploy to Production
        deployment: production
        trigger: manual # 配置为手动, 使上线可控
        script:
          - ssh -T user@host deploy.sh
          - echo "Successful deployment to Production"
```

