---
title: "Deploy Xxl-Job With Docker"
date: 2022-04-21T15:34:40+08:00
draft: false
tags: ["xxl-job", "java", "Docker"]
categories: ["技术"]
---


> 基础环境：
> M1 MacBook Pro Monterey 12.3.1
> Java azul-1.8.0_322
> xxl-job v2.3.0


本文以Docker 运行的方式在本地部署运行xxl-job 并简单测试任务的执行。

xxl-job 官方文档对Docker 形式运行有所介绍，但内容不够详细。

官方提供的xxl-job 镜像中不包含MySQL，因此需要单独启动MySQL服务，并保证xxl-job-admin 服务能够连接MySQL。

# 启动并配置MySQL

MySQL同样以Docker镜像方式运行，参考了[Mac m1 docker 安装mysql](https://www.jianshu.com/p/eb3d9129d880)

M1 的Mac 还要注意一点不能直接`docker pull mysql`，这样会报'no matching manifest for linux/arm64/v8 in the manifest list entries'

## 启动MySQL服务

```shell
docker pull mysql/mysql-server:5.7.30
```

镜像下载之后，启动MySQL服务。

```shell
docker run --platform linux/amd64 --name mysql -p 3306:3306 -v /tmp:/opt -e MYSQL_ROOT_PASSWORD=123456 -d mysql/mysql-server:5.7.30
```

## 设置MySQL远程连接

进入MySQL容器

```shell
docker exec -it mysql /bin/bash
```

登录MySQL

```shell
mysql -u root -p
```

输入密码。并依次执行以下命令：

```shell
CREATE USER 'root'@'%' IDENTIFIED BY 'root';
GRANT ALL ON *.* TO 'root'@'%';                  //远程连接授权
flush privileges;                                //刷新权限
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';  //更新root用户密码
flush privileges;                                //刷新权限
```

## 初始化xxl-job数据库

```shell
mysql> source /path/to/xxl-job/doc/db/tables_xxl_job.sql
```

# 启动xxl-job-admin

## 下载官方镜像

```shell
docker pull xuxueli/xxl-job-admin:2.3.0
```

## 启动容器

按照官方文档中示例的命令启动xxl-job-admin容器镜像，但是无法正常访问web界面，日志中报错：

```shell
External DTD: Failed to read external DTD 'mybatis-3-mapper.dtd', because 'http' access is not allowed due to restriction set by the accessExternalDTD property.
```

解决这个问题需要在服务启动的时候添加Java参数'-Djavax.xml.accessExternalDTD=all'

另外，因为是在一个容器访问另一个容器中的MySQL，因此需要通过link的方式连接。

最终启动命令如下：

```shell
docker run --platform linux/amd64 -e PARAMS="--spring.datasource.url=jdbc:mysql://mysql:3306/xxl-job?Unicode=true&characterEncoding=UTF-8 --spring.datasource.username=root --spring.datasource.password=123456" -e JAVA_OPTS="-Djavax.xml.accessExternalDTD=all" -p 8080:8080 -v /tmp:/data/applogs --name xxl-job-admin -d --link mysql:mysql xuxueli/xxl-job-admin:2.3.0
```

观察日志，服务启动成功之后，浏览器访问'http://localhost:8080/xxl-job-admin'，可以看到web界面。


# 执行shell任务

admin服务启动成功之后，执行器的启动比较简单，Idea打开源代码中的'xxl-job-executor-samples/xxl-job-executor-sample-springboot'项目，编译打包成jar包，然后运行即可。

添加'GLUE(shell)'类型的任务之后，需要在Glue IDE中编辑任务的具体内容，然后可以执行任务。
