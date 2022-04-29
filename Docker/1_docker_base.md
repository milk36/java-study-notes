Docker基础篇
===
Docker 架构

![Docker 架构](https://docs.docker.com/engine/images/architecture.svg "Docker 架构")
#### 相关资源

[官网](http://www.docker.com)

[仓库](https://hub.docker.com)

[docker 使用安装教程 参考](https://github.com/judasn/Linux-Tutorial/blob/955ff70778/markdown-file/Docker-Install-And-Usage.md)

* CentOS 安装过程：

  * 卸载原有的环境：

    ```shell
    sudo yum remove docker \
                      docker-client \
                      docker-client-latest \
                      docker-common \
                      docker-latest \
                      docker-latest-logrotate \
                      docker-logrotate \
                      docker-engine
    ```
  * 安装指令
    安装对应的依赖环境和镜像地址

    ```shell
    yum install -y yum-utils
    #默认国内访问官方镜像地址很慢
    yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```
    安装过慢设置阿里云镜像

    ```shell
    yum-config-manager \
        --add-repo \
        http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    #yum更新下即可
    yum makecache fast
    ```

    直接安装docker CE

    ```shenll
    sudo yum install -y docker-ce docker-ce-cli containerd.io
    ```
  * Docker卸载

    ```shell
    systemctl stop docker
    yum -y remov docker-ce
    rm -rf /var/lib/docker
    # 重启服务
    sudo systemctl restart docker
    ```
* 查看配置文件位置：systemctl show --property=FragmentPath docker

* 启动 Docker：`systemctl start docker.service` 或 `systemctl start docker`

* 停止 Docker：`systemctl stop docker.service` 或 `systemctl stop docker`
`
* 查看状态：`systemctl status docker.service` 或 `systemctl status docker`

* 开机启动docker `systemctl enable docker`

* 运行 hello world 镜像：`docker run hello-world`
#### Docker基本结构
* 镜像(image)

  Docker 镜像（Image）就是一个只读的模板。镜像可以用来创建 Docker 容器，一个镜像可以创建很多容器。

  | docker | 面向对象 |
  | :----- | :------- |
  | 容器(container)   | 对象     |
  | 镜像(image)   | 类       |
* 容器(container)

  Docker 利用容器（Container）独立运行的一个或一组应用。容器是用镜像创建的运行实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。可以把容器看做是一个简易版的 Linux 环境（包括root用户权限、进程空间、用户空间和网络空间等）和运行在其中的应用程序。容器的定义和镜像几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。  
* 仓库(repository)

  仓库（Repository）是集中存放镜像文件的场所。

  仓库(Repository)和仓库注册服务器（Registry）是有区别的。仓库注册服务器上往往存放着多个仓库，每个仓库中又包含了多个镜像，每个镜像有不同的标签（tag）。
  
  仓库分为公开仓库（Public）和私有仓库（Private）两种形式。
  
  最大的公开仓库是 Docker Hub(https://hub.docker.com/)，存放了数量庞大的镜像供用户下载。国内的公开仓库包括阿里云 、网易云 等  

* 阿里云仓库配置

  访问: https://cr.console.aliyun.com/cn-hangzhou/instances

  镜像工具 - > 镜像加速器

  ```shell
  sudo mkdir -p /etc/docker
  sudo tee /etc/docker/daemon.json <<-'EOF'
  {
    "registry-mirrors": ["填写私有的加速器地址"]
  }
  EOF
  sudo systemctl daemon-reload
  sudo systemctl restart docker
  ```
### Docker命令
![Docker command dragram](../img/dockercommand.png "Docker command dragram")
* Docker帮助命令
  | 命令           | 说明                                       |
  | -------------- | ------------------------------------------ |
  | docker version | 查看docker的版本信息                       |
  | docker info    | 查看docker详细的信息                       |
  | docker --help  | docker的帮助命令，可以查看到相关的其他命令 |
#### 镜像相关命令

  | 镜像命令               | 说明                     |
  | ---------------------- | ------------------------ |
  | docker images          | 列出本地主机上的镜像     |
  | docker search 镜像名称 | 从 docker hub 上搜索镜像 |
  | docker pull 镜像名称   | 从 docker hub 上下载镜像  |
  | docker rmi 镜像名称    | 删除本地镜像             |
* Docker images 镜像信息
 
  镜像表格信息说明

  | 选项       | 说明             |
  | ---------- | ---------------- |
  | REPOSITORY | 表示镜像的仓库源 |
  | TAG        | 镜像的标签       |
  | IMAGE ID   | 镜像ID           |
  | CREATED    | 镜像创建时间     |
  | SIZE       | 镜像大小         |

  | 参数       | 说明               |
  | ---------- | ------------------ |
  | -a         | 列出本地所有的镜像 |
  | -q         | 只显示镜像ID       |
  | --digests  | 显示镜像的摘要信息 |
  | --no-trunc | 显示完整的镜像信息 |
  ```shell
  docker images -a //列出本地所有的镜像
  docker images --digests //显示镜像的摘要信息
  docker images --no-trunc //显示完整的镜像信息
  ```
* Docker search  在线仓库搜索

  docker hub是Docker的在线仓库，我们可以通过docker search 在上面搜索我们需要的镜像

  [hub 仓库](https://hub.docker.com)

  | 参数名称   | 描述                                      |
  | ---------- | ----------------------------------------- |
  | --no-trunc | 显示完整的描述信息                        |
  | --limit    | 分页显示                                  |
  | -f         | 过滤条件  docker search -f STARS=5 tomcat |

  ```shell
  docker search centos
  docker search redis
  ```
* Docker pull 拉取镜像

  从Docker hub 上下载镜像文件

  ```shell
  docker pull tomcat
  docker pull redis
  ```
* Docker rmi 删除镜像

  | 删除方式 | 命令                               |
  | -------- | ---------------------------------- |
  | 删除单个 | docker rmi -f 镜像ID               |
  | 删除多个 | docker rmi -f 镜像1:TAG 镜像2:TAG  |
  | 删除全部 | docker rmi -f $(docker images -qa) |  

  ```shell
  docker rmi -f hello-world
  docker rmi -f redis:latest tomcat:latest
  ```
#### 容器命令

  有镜像才能创建容器
* 启动容器

  `docker run [OPTIONS] IMAGE [COMMAND]`

  OPTIONS中的一些参数

  | options | 说明                                                         |
  | ------- | :----------------------------------------------------------- |
  | -\-name | "容器新名字": 为容器指定一个名称                             |
  | -d      | 后台运行容器，并返回容器ID，也即启动守护式容器               |
  | `-i`    | `以交互模式运行容器，通常与 -t 同时使用`                     |
  | `-t`    | `为容器重新分配一个伪输入终端，通常与 -i 同时使用`           |
  | -P:     | 随机端口映射                                                 |
  | -p      | 指定端口映射，有以下四种格式 ip:hostPort:containerPort<br>ip::containerPort<br>`hostPort:containerPort`<br>containerPort<br> |

  交互式的容器

  ```shell
  //创建centos容器 并进入centos shell命令行
  docker run -it centos /bin/bash
  ```
* 列举运行容器信息

  我们要查看当前正在运行的容器有哪些，可以通过ps 命令来查看

  `docker ps [OPTIONS]`

  OPTONS可用的参数

  | OPTIONS    | 说明                                      |
  | ---------- | ----------------------------------------- |
  | -a         | 列出当前所有正在运行的容器+历史上运行过的 |
  | -l         | 显示最近创建的容器。                      |
  | -n         | 显示最近n个创建的容器。                   |
  | -q         | 静默模式，只显示容器编号。                |
  | --no-trunc | 不截断输出。                              |
* 退出容器命令
  
  启动了一个容器后，如何退出容器

  | 退出方式 | 说明           |
  | -------- | -------------- |
  | `exit`     | 容器停止退出   |
  | ctrl+p+q | 容器不停止退出 |
* 启动容器
  ```shell
  docker start 容器ID或者名称
  ```
* 重启容器
  ```shell
  docker restart 容器ID或者名称
  ```  
* 停止容器
  ```shel
  docker stop 容器ID或者名称

  //强制停止容器处理方式
  docker kill 容器ID或者名称
  ```  
* 删除容器

  当不需要容器的时候, 可以进行删除操作, 使用 rm 命令
  ```shell
  docker rm 容器ID
  //删除所有 ps 查讯到的容器
  docker rm -f $(dokcer ps -qa)
  //通过管道的方式删除 ps 查询到的容器
  docker ps -a -q | xargs docker rm
  ```
* 守护方式启动容器

  |OPTIONS|说明|
  |--|--|
  |-d|在后台运行容器并打印容器ID|
  |-i|保持STDIN打开，即使没有连接|
  |-t|分配一个pseudo-TTY|
  |--rm|在容器退出时自动移除容器|
  |--name|为容器指定一个名称|
  ```shell
  docker run -d 容器名称

  docker run -dit openeuler/openeuler

  //启动容器 并运行一个在后台循环输出的脚本
  docker run -d openeuler/openeuler /bin/bash -c 'while true;do echo hello docker;sleep 2;done'
  ```
  查看运行日志
  ```shell
  docker logs -t -f --tail 3(显示行数) 容器ID
  ```
  查看容器中运行的进程
  ```shell
  docker top 容器ID
  ```
* 查看容器细节
  ```shell
  docker inspect 容器ID
  ```
* 进入运行的容器

  | 进入方式 | 说明                                         |
  | -------- | -------------------------------------------- |
  | exec     | 在容器中打开新的终端,并且可以启动新的进程    |
  | attach   | 直接进入容器启动命令的终端，不会启动新的进程 | 

  ```shell
  //进入容器命令行
  docker exec -it 容器ID /bin/bash
  docker exec -it f5dd bash

  //附属容器终端
  docker attach 容器ID
  docker attach f5dd
  ```
* 文件复制

  把容器中的文件复制到宿主机器
  ```shell
  docker cp 容器ID:容器内路径 目的地路径

  docker cp f5dd:/root/test.txt /data/
  ```
