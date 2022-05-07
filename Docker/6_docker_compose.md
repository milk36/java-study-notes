DockerCompose
===
Compose 是用于定义和运行多容器 Docker 应用程序的工具.通过 Compose,您可以使用 YML 文件来配置应用程序需要的所有服务.

然后,使用一个命令,就可以从 YML 文件配置中创建并启动所有服务

​一键启动所有的服务

DockerCompose的使用步骤

1. 创建对应的DockerFile文件
1. 创建yml文件，在yml文件中编排我们的服务
1. 通过`docker-compose up`命令 一键运行我们的容器

[官网教程](https://docs.docker.com/compose)

[compose 官方github](https://github.com/docker/compose)

#### 官方github 手动安装 docker-compose命令 v1版:

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

```shell
curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

```

修改文件夹权限

```shell
chmod +x /usr/local/bin/docker-compose
```

建立软连接

```shell
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

校验是否安装成功

```shell
docker-compose --version
```
#### 官方github 手动安装 V2版安装
```shell
mkdir -p /usr/local/lib/docker/cli-plugins
#拉取到宿主机
curl -SL https://github.com//docker/compose/releases/download/v2.5.0/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
#修改文件夹权限
chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
#安装成功
docker compose version
```
#### Compose文件的定义级使用

[参考官网入门教程](https://docs.docker.com/compose/gettingstarted/)

* 创建Dockerfile 
  ```shell
  # syntax=docker/dockerfile:1
  FROM python:3.7-alpine
  WORKDIR /code
  ENV FLASK_APP=app.py
  ENV FLASK_RUN_HOST=0.0.0.0
  RUN apk add --no-cache gcc musl-dev linux-headers
  COPY requirements.txt requirements.txt
  RUN pip install -r requirements.txt
  EXPOSE 5000
  COPY . .
  CMD ["flask", "run"]
  ```
  含义:
    1. 从 Python 3.7 映像开始构建映像。
    1. 将工作目录设置为/code.
    1. 设置命令使用的环境变量flask。
    1. 安装 gcc 和其他依赖项
    1. 复制requirements.txt并安装 Python 依赖项。
    1. 向镜像添加元数据以描述容器正在侦听端口 5000
    1. 将项目中的当前目录复制.到镜像中的workdir .。
    1. 将容器的默认命令设置为flask run.
* 创建 Compose文件 docker-compose.yml
  ```yml
  version: "3.9"
  services:
    web:
      build: .
      ports:
        - "8080:5000"
    redis:
      image: "redis:alpine"
  ```
  这个 Compose 文件定义了两个服务：web和redis.

  该服务使用从当前目录中web构建的图像。Dockerfile然后它将容器和主机绑定到暴露的端口，8000. 此示例服务使用 Flask Web 服务器的默认端口，5000.
* 执行 Compose

  `docker compose up`  
* 另一个 Compose例子
  ```yml
  version: '3'
  services:
    nginx:
      image: nginx #nginx镜像
      ports:
        - 80:80 #端口映射
      links:
        - app #定义另一个服务中容器的网络链接
      volumes:
        - ./nginx/conf.d:/etc/nginx/conf.d #数据卷挂载
      app:
        image: app #自定义app镜像
        expose:
          - "8080" #暴露端口
  ```
  
  nginx/conf.d 配置
  ```shell
  server {
      listen 80;
      access_log off;

      location / {
          proxy_pass http://app:8080;
      }
    
  }
  ```

#### Compose 参考

  [docker-compose CLI 概述](https://docs.docker.com/compose/reference/)

  [Compose 和 Docker 版本兼容性](https://docs.docker.com/compose/compose-file/compose-file-v2/)

  [Compose 编写规范](https://docs.docker.com/compose/compose-file/)