Docker 应用部署
===
### 部署 mysql
* 搜索mysql镜像
  ```shell
  docker search mysql
  ```
* 拉取mysql镜像
  ```shell
  docker pull mysql:5.6
  ```  
* 创建容器 设置端口 目录映射
  ```shell
  mkdir /data/mysql
  cd /data/mysql

  docker run -id \
  -p 3306:3306 \
  --name=c_mysql \
  -v $PWD/conf:/etc/mysql/conf.d \
  -v $PWD/logs:/logs \
  -v $PWD/data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=123456 \
  mysql:5.6
  ```  
  * 参数说明:
    - **-id** :保持STDIN打开，即使没有连接 ; 在后台运行容器并打印容器ID
    - **-p 3306:3306** :将容器的 3306 端口映射到宿主机的 3306 端口。
    - **-v $PWD/conf:/etc/mysql/conf.d** :将主机当前目录下的 conf/my.cnf 挂载到容器的 /etc/mysql/my.cnf。配置目录
    - **-v $PWD/logs:/logs** :将主机当前目录下的 logs 目录挂载到容器的 /logs。日志目录
    - **-v $PWD/data:/var/lib/mysql** :将主机当前目录下的data目录挂载到容器的 /var/lib/mysql 。数据目录
    - **-e MYSQL_ROOT_PASSWORD=123456** :初始化 root 用户的密码。
### 部署 mongodb
* 拉取mysql镜像
  ```shell
  docker pull mongo:3.4
  ```  
* 创建容器 设置端口 目录映射
  ```shell
  mkdir /data/mongo
  cd /data/mongo

  docker run -id \
  -p 27017:27017 \
  --name c_mongo \
  -v $PWD/db:/data/db \
  mongo:3.4 --auth

  #实际运行指令
  docker run -id -p 27017:27017 --name c_mongo -v /data/docker/mongo/db:/data/db mongo:3.4 --auth
  docker run -id -p 27017:27017 --name d_mongo -v /data/docker/mongo/db_4.2:/data/db mongo:4.2 --auth
  ```  
* 配置mongodb 超级用户
  
  进入命令行工具配置 :

  `docker exec -it c_mongo mongo`  
  
  4.x以上版本使用 :
  
  `docker exec -it c_mongo mongosh`

  ```shell
  use admin

  db.createUser( 
      { 
          user: "mongo-admin", 
          pwd: "123456", 
          roles: [ 
              { role: "root", db: "admin" } 
          ] 
      } 
  ) 

  db.auth("mongo-admin","123456")
  ```
  [参考 Linux-Tutorial install MongoDB](https://github.com/judasn/Linux-Tutorial/blob/955ff70778c388c807eaf51eb29ae5cfbb75eb60/markdown-file/MongoDB-Install-And-Settings.md)
### 部署 nginx  
* 拉取镜像
  ```shell
  docker pull nginx
  ```
* 创建容器 设置端口 目录映射
  ```shell
  mkdir /data/docker/nginx
  mkdir /data/docker/nginx/conf  
  ```
  * nginx 配置文件
    ```shell
    cd /data/docker/nginx/conf
    vim nginx.conf
    ```
    配置:

    ```shell    
    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
    }


    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        keepalive_timeout  65;

        #gzip  on;

        include /etc/nginx/conf.d/*.conf;
    }
    ```
  * 启动参数 
    ```shell
    cd /data/docker/nginx

    docker run -id --name=c_nginx \
    -p 80:80 \
    -v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf \
    -v $PWD/logs:/var/log/nginx \
    -v $PWD/html:/usr/share/nginx/html \
    nginx
    ```

    * 参数说明：
      * **-p 80:80**：将容器的 80端口映射到宿主机的 80 端口。
      * **-v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf**：将主机当前目录下的 /conf/nginx.conf 挂载到容器的 :/etc/nginx/nginx.conf。配置目录
      * **-v $PWD/logs:/var/log/nginx**：将主机当前目录下的 logs 目录挂载到容器的/var/log/nginx。日志目录
  * 设置默认访问 `index.html`
    ```shell
    cd /data/docker/nginx/html
    echo hi nginx docker >> index.html
    ```
### 部署 redis    
* 拉取redis镜像
  ```shell
  docker pull redis:5.0
  ```
* 创建容器 设置启动参数
  ```shell
  docker run -id --name=c_redis -p 6379:6379 redis:5.0
  ```  
* 修改redis密码
  ```shell
  docker exec -it c_redis redis-cli #进入redis命令行控制台

  redis-cli

  #查看现有密码
  127.0.0.1:6379> config get requirepass

  #设置密码
  127.0.0.1:6379>config set requirepass XXX(XXX为你要设置的密码)

  #带权限进入redis命令行
  redis-cli -h 127.0.0.1 -p 6379 -a XXX
  ```
  