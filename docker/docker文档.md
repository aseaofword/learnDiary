

# docker文档

## 一、docker安装过程

### 1.更新yum源

```
yum update
```

### 2.安装docker所需依赖

```java
 yum install -y yum-utils device-mapper-persistent-data lvm2
```

### 3.设置Docker的yum的源

```
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 4.查看仓库所有Docker版本

```java
yum list docker-ce --showduplicates | sort -r
```

### 5.安装Docker

```
[root@CodeGuide ~]# sudo yum install docker-ce
[root@CodeGuide ~]# 推荐；sudo yum install -y docker-ce-25.0.5 docker-ce-cli-25.0.5 containerd.io
```

### 6.安装docker-compose

官网地址：

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

```

镜像地址：

```
# 指定路径【推荐】
sudo curl -L https://gitee.com/fustack/docker-compose/releases/download/v2.24.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
# 设置权限
sudo chmod +x /usr/local/bin/docker-compose

```

### 7.启动Docker并添加开机自启动

启动docker：

```
[root@CodeGuide ~]# sudo systemctl start docker
```

开机自启动：

```
[root@CodeGuide ~]# systemctl enable docker
```

重启docker：

```
sudo systemctl restart docker
```

### 8.卸载docker

```
[root@CodeGuide ~]# sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

### 9.docker常用指令

```
[root@CodeGuide ~]# docker --help				#Docker帮助
[root@CodeGuide ~]# docker --version			#查看Docker版本
[root@CodeGuide ~]# docker search <image>		#搜索镜像文件，如：docker search mysql
[root@CodeGuide ~]# docker pull <image>		#拉取镜像文件， 如：docker pull mysql
[root@CodeGuide ~]# docker images				#查看已经拉取下来的所以镜像文件
[root@CodeGuide ~]# docker rmi <image>		#删除指定镜像文件
[root@CodeGuide ~]# docker run --name <name> -p 80:8080 -d <image>		#发布指定镜像文件
[root@CodeGuide ~]# docker ps					#查看正在运行的所有镜像
[root@CodeGuide ~]# docker ps -a				#查看所有发布的镜像
[root@CodeGuide ~]# docker rm <image>			#删除执行已发布的镜像
[root@CodeGuide ~]# docker exec -it mysql bash			#进入容器
```

### 10.设置国内源

阿里云提供了镜像源：[https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors (opens new window)](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)- 登录后你会获得一个专属的地址。

使用以下命令来设置 Docker 国内源：- 或者你可以通过 `vim /etc/docker/daemon.json` 进入修改添加 registry-mirrors 内容后重启 Docker

```yaml
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://ll5nv541.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
 
```

## 二、Dockerfile

### 1.常用命令

| 命令       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| FROM       | 基于哪个镜像来实现                                           |
| MAINTAINER | 作者                                                         |
| ENV        | 环境变量                                                     |
| ADD        | 添加宿主机文件到容器中，有需要解压的文件会自动解压           |
| COPY       | 添加宿主机文件到容器中                                       |
| WORKDIR    | 设置后续指令的工作目录                                       |
| EXPOSE     | 容器内需要暴露的端口                                         |
| CMD        | 容器启动后会执行的命令，如果执行docker run会被覆盖           |
| ENTRYPOINT | 和CMD功能相同，但是docker run命令不会覆盖，如果需要覆盖可以用-entrypoint来覆盖 |
| VOLUME     | 挂载目录                                                     |
| USER       | 指定后续指令的用户上下文。                                   |
| ARG        | 定义在构建过程中传递给构建器的变量，可使用 "docker build" 命令设置。 |

**RUN指令的两种格式：**

shell 格式：RUN <命令行命令>

exec 格式：RUN ["可执行文件", "参数1", "参数2"]

## 三、docker compsose

### 1.Compose 安装

 安装

```shell
$ sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

添加可执行权限

```shell
$ sudo chmod +x /usr/local/bin/docker-compose
```

创建软链

```shell
$ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

### 2.使用

**depends_on设置依赖关系。**

- docker-compose up ：以依赖性顺序启动服务。在以下示例中，先启动 db 和 redis ，才会启动 web。
- docker-compose up SERVICE ：自动包含 SERVICE 的依赖项。在以下示例中，docker-compose up web 还将创建并启动 db 和 redis。
- docker-compose stop ：按依赖关系顺序停止服务。在以下示例中，web 在 db 和 redis 之前停止。

```
version: "3.7"
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```

注意：web 服务不会等待 redis db 完全启动 之后才启动。

**env_file**

从文件添加环境变量。可以是单个值或列表的多个值。

```
env_file: .env
```

也可以是列表格式：

```
env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

**expose**

暴露端口，但不映射到宿主机，只被连接的服务访问。

仅可以指定内部端口为参数：

```
expose:
 - "3000"
 - "8000"
```

**extra_hosts**

添加主机名映射。类似 docker client --add-host。

```
extra_hosts:
 - "somehost:162.242.195.82"
 - "otherhost:50.31.209.229"
```

以上会在此服务的内部容器中 /etc/hosts 创建一个具有 ip 地址和主机名的映射关系：

```
162.242.195.82  somehost
50.31.209.229   otherhost
```

**healthcheck**

用于检测 docker 服务是否健康运行。

```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"] # 设置检测程序
  interval: 1m30s # 设置检测间隔
  timeout: 10s # 设置检测超时时间
  retries: 3 # 设置重试次数
  start_period: 40s # 启动后，多少秒开始启动检测程序
```

**image**

指定容器运行的镜像。以下格式都可以：

```
image: redis
image: ubuntu:14.04
image: tutum/influxdb
image: example-registry.com:4000/postgresql
image: a4bc65fd # 镜像id
```

**logging**

服务的日志记录配置。

driver：指定服务容器的日志记录驱动程序，默认值为json-file。有以下三个选项

```
driver: "json-file"
driver: "syslog"
driver: "none"
```

仅在 json-file 驱动程序下，可以使用以下参数，限制日志得数量和大小。

```
logging:
  driver: json-file
  options:
    max-size: "200k" # 单个文件大小为200k
    max-file: "10" # 最多10个文件
```

当达到文件限制上限，会自动删除旧的文件。

syslog 驱动程序下，可以使用 syslog-address 指定日志接收地址。

```
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```

**restart**

- no：是默认的重启策略，在任何情况下都不会重启容器。
- always：容器总是重新启动。
- on-failure：在容器非正常退出时（退出状态非0），才会重启容器。
- unless-stopped：在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器

```
restart: "no"
restart: always
restart: on-failure
restart: unless-stopped
```

注：swarm 集群模式，请改用 restart_policy。

**secrets**

存储敏感数据，例如密码：

```
version: "3.1"
services:

mysql:
  image: mysql
  environment:
    MYSQL_ROOT_PASSWORD_FILE: /run/secrets/my_secret
  secrets:
    - my_secret

secrets:
  my_secret:
    file: ./my_secret.txt
```

**volumes**

将主机的数据卷或者文件挂载到容器里。

```
version: "3.7"
services:
  db:
    image: postgres:latest
    volumes:
      - "/localhost/postgres.sock:/var/run/postgres/postgres.sock"
      - "/localhost/data:/var/lib/postgresql/data"
```

00:11:34.728 logback [main] INFO  o.s.b.w.s.c.ServletWebServerApplicationContext - Root WebApplicationContext: initialization completed in 1470 ms 	at sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1162) 