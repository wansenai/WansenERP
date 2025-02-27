# Eairp是什么

[Eairp](https://github.com/wansenai/eairp) (Enterprise AI Resource Planning) 是一套面向企业的综合资源规划系统，旨在优化整合企业各项运营流程，提高管理效率，降低运营成本。
EAIRP包含物料采购、财务预算、库存管理、账单管理、用户角色组织管理等多项功能，并引入先进的AI助手，为企业提供智能管理解决方案。

# 目录
- [简介](#简介)
- [镜像使用指南](#镜像使用指南)
    -	[拉取现有的镜像](#拉取现有的镜像)
        -	[使用docker run](#使用docker run)
        -	[使用docker-compose](#使用docker-compose)
    -	[构建镜像](#构建镜像)
- [升级Eairp](#升级Eairp)
- [故障排查](#故障排查)
- [镜像细节说明](#镜像细节说明)
    -	[配置选项](#配置选项)
    -	[配置文件](#配置文件)
    -	[其他说明](#其他说明)
- [许可证](#许可证)
- [支持](#支持)
- [贡献指南](#贡献指南)
- [致谢](#致谢)

# 简介

目标是提供在 Docker 中运行的可用于生产的 Eairp 系统。原因如下：

-	该操作系统基于 Debian，而不是基于一些占用空间较小的发行版，如 Alpine
-	Docker Compose 使用了多个容器：一个用于 mysql，一个用于 Redis，另一个用于 Eairp + Nginx。这样就可以在不同的机器上运行它们。

# 镜像使用指南

请先安装 [Docker](https://www.docker.com/) 随后可选择：

1.	从DockerHub拉取现成镜像。
2.	下载[项目源码](https://github.com/wansenai/eairp)自行构建。

## 拉取现有的镜像

需运行三个容器：

-	Eairp主应用容器
-	MySQL数据库容器
-	Redis数据库容器

### 使用docker run

首先创建一个专用的docker网络：

```console
docker network create -d bridge eairp-nw
```
然后为MySQL数据库运行一个容器，并确保其配置为使用UTF8编码。

#### 启动MySQL

我们将使用的本地目录挂载MySQL容器的配置文件：
-	一个用于在数据库初始化时设置权限（见下文），
-	另一个是将Eairp放在MySQL数据库中的数据包含在内，这样当你停止并重新启动MySQL时，你就不会发现自己没有任何数据。

例如：
-	`/usr/local/mysql-init`
-	`/usr/local/mysql`

您需要确保这些目录存在，然后将`eairp.sql`文件复制到`/usr/local/mysql-int`目录。（您可以根据需要重新命名它，例如`init.sql `）。

注意：确保您挂载到容器中的目录是绝对路径，而不是相对路径。

```console
docker run --net=eairp-nw \
           -d \
           --name mysql-eairp \
           -p 3306:3306 \
           -v /usr/local/mysql:/var/lib/mysql \
           -v /usr/local/mysql-init:/docker-entrypoint-initdb.d \
           -e MYSQL_ROOT_PASSWORD=123456 \
           -e MYSQL_USER=eairp \
           -e MYSQL_PASSWORD=123456 \
           -e MYSQL_DATABASE=eairp \
           mysql:8.3 \
           --character-set-server=utf8mb4 \
           --collation-server=utf8mb4_bin \
           --explicit-defaults-for-timestamp=1
```
或
```console
docker run --net=eairp-nw -d --name mysql-eairp -p 3306:3306 -v /usr/local/mysql:/var/lib/mysql -v /usr/local/mysql-init:/docker-entrypoint-initdb.d -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_USER=eairp -e MYSQL_PASSWORD=123456 -e MYSQL_DATABASE=eairp -d mysql:8.3 --character-set-server=utf8mb4 --collation-server=utf8mb4_bin --explicit-defaults-for-timestamp=1
```

您应该根据本地MySQL数据库密码，修改命令 `MYSQL_PASSWORD` 参数。

#### 启动Redis

我们还需要使用的本地目录挂载Redis容器的配置文件：

-	`/usr/local/redis/data`
-	`/usr/local/redis/redis.conf`

在继续之前，您需要确保此目录存在。

注意：确保您挂载到容器中的目录是绝对路径，而不是相对路径。

```console
docker run --net=eairp-nw \
           -d \
           --name redis-eairp \
           -p 6379:6379 \
           -v /usr/local/redis/redis.conf:/etc/redis/redis.conf \
           -v /usr/local/redis/data:/data \
           -e SPRING_REDIS_PASSWORD=123456
           redis:latest
```
或
```console
docker run --net=eairp-nw -d --name redis-eairp -v /usr/local/redis/redis.conf:/etc/redis/redis.conf -v /usr/local/redis/data:/data -p 6379:6379 redis:latest
```

您应该根据本地Redis数据库密码，修改命令 `SPRING_REDIS_PASSWORD` 参数，如果没有就不用传递该环境变量。

#### 启动Eairp

我们还将为Eairp日志（包含应用程序配置和状态）绑定挂载一个本地目录，例如：

-	`/usr/local/eairp`

注意：确保您挂载到容器中的目录是绝对路径，而不是相对路径。

确保此目录存在，然后通过运行以下命令之一在容器中运行Eairp。

```console
docker run --net=eairp-nw \
           -d \
           --name eairp \
           -p 8088:8088 \
           -p 3000:80 \
           -v /usr/local/eairp:/application/log \
           -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-eairp:3306/eairp \
           -e SPRING_DATASOURCE_USERNAME=eairp \
           -e SPRING_DATASOURCE_PASSWORD=123456 \
           -e SPRING_REDIS_HOST=redis-eairp \
           -e SPRING_REDIS_PORT=6379 \
           -e SPRING_REDIS_PASSWORD=123456 \
           -e API_BASE_URL=http://localhost:8088/erp-api \
           wansenai/eairp:latest
```
或
```console
docker run --net=eairp-nw -d --name eairp -p 8088:8088 -p 3000:80 -v /usr/local/eairp:/application/log -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-eairp:3306/eairp -e SPRING_DATASOURCE_USERNAME=eairp -e SPRING_DATASOURCE_PASSWORD=123456 -e SPRING_REDIS_HOST=redis-eairp  -e SPRING_REDIS_PORT=6379 -e SPRING_REDIS_PASSWORD=123456 -e API_BASE_URL=https://eairp.cn/erp-api wansenai/eairp:latest
```

注意：Eairp在 `Spring Boot` 中使用 `Spring DataSource` 作为连接到数据库的数据源。

请不要忘记使用之前创建的MySQL容器和Redis容器的名称添加MySQL数据库连接环境变量（`SPRING_DATASOURCE_URL`）和Redis数据库连接环境参数（`SPRING_REDIS_HOST `），以便Eairp知道其数据库的位置。

如果要在服务器上部署，请修改 `API_BASE_URL` 环境变量的值，例如：
-	`http://eairp.cn/erp-api`
-	`https://eairp.cn/erp-api`

### 使用docker-compose

另一种解决方案是使用我们提供的Docker Compose文件。

首先，您需要下载[eairp源代码](https://github.com/eairps/eairp/releases)到您的本地计算机，然后您必须从[docker](https://github.com/eairps/eairp/tree/master/docker)文件夹下载5个文件，它们是：

-	`.env`
-	`Dockerfile`
-	`docker-compose.yaml`
-	`start.sh`
-	`mysql-scripts/eairp.sql`

然后将这四个文件和 `mysql-scripts` 文件夹复制到Eairp源代码目录结构中，注意 `mysql-scscripts` 是一个文件夹。您需要复制整个文件夹，而不仅仅是其中的sql文件。

完整的目录结构如下：

```markdown
eairp/
├── core/
├── desktop/
├── docs/
├── images/
├── web/
├── mysql-scripts/
│ ├── eairp.sql
│
└── .env
└── docker-compose.yaml
└── Dockerfile
└── start.sh
```

作为参考，这里有一个最小的Docker Compose文件，您可以将其用作示例（完整示例再[此处](https://github.com/eairps/eairp/tree/master/docker/latest/docker-compose.yaml)):

```yaml
version: '3.8'
networks:
  bridge:
    driver: bridge
services:
  eairp:
    image: wansenai/eairp:latest
    container_name: eairp
    ports:
      - "3000:80"
      - "8088:8088"
    environment:
      SPRING_DATASOURCE_URL: "${SPRING_DATASOURCE_URL}"
      SPRING_DATASOURCE_USERNAME: "${SPRING_DATASOURCE_USERNAME}"
      SPRING_DATASOURCE_PASSWORD: "${SPRING_DATASOURCE_PASSWORD}"
      SPRING_REDIS_HOST: "${SPRING_REDIS_HOST}"
      SPRING_REDIS_PORT: "${SPRING_REDIS_PORT}"
      SPRING_REDIS_PASSWORD: "${SPRING_REDIS_PASSWORD}"
      SPRING_PROFILE: "docker"
    depends_on:
      - mysql
      - redis
    networks:
      - bridge

  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
      MYSQL_DATABASE: "eairp"
      MYSQL_USER: "${MYSQL_USER}"
      MYSQL_PASSWORD: "${MYSQL_PASSWORD}"
    command:
      - "--character-set-server=utf8mb4"
      - "--collation-server=utf8mb4_bin"
      - "--explicit-defaults-for-timestamp=1"
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql-scripts:/docker-entrypoint-initdb.d
    cap_add:
      - SYS_NICE
    networks:
      - bridge

  redis:
    image: redis:7.0
    container_name: redis
    command: redis-server --requirepass 123456
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - bridge

volumes:
  mysql-data:
  redis-data:
```

## 构建镜像

这允许您在本地重建 `Eairp-docker` 映像。以下是步骤：
-	```shell
    docker-compose up
    ```
-	启动浏览器并将其指向`http://localhost:3000`

请注意，`docker-compose up` 将在第一次运行时自动构建Eairp映像。如果你需要重新构建，你可以运行 `docker compose up--build`。您还可以使用 `docker build . -t eairp:latest` 例如最新的版本。

你也可以通过运行 `docker build -t eairp` 来构建镜像然后使用上面的命令启动Eairp，并使用`docker run ...`

# 升级Eairp

您已经安装了Eairp docker镜像并使用了它，现在是您想将Eairp升级到较新版本的时候了。

如果您遵循了上述说明，则已将Eairp本地目录映射到主机上。

升级所需做的就是停止正在运行的Eairp容器，并启动要升级到的新版本。您应该保持Mysql容器和Redis容器运行。

请注意，您当前的Eairp配置文件（`start.sh`、`.env` 和 `application.yml`）将被保留。

您应该始终查看[发行说明](https://github.com/wansenai/eairp/releases)对于当前版本和要升级到的新版本之间发生的所有变化，可能需要执行一些手动步骤（例如更新Eairp配置文件）。

# 镜像细节说明

## 配置选项

第一次从Eairp映像创建容器时，会在容器中执行一个shell脚本（`/usr/local/bin/docker-entrypoint.sh`）来设置一些配置。可以传递以下环境变量：

-	`SPRING_DATASOURCE_URL`: 用于连接Mysql数据库的JDBC地址，默认端口为3306。
-	`SPRING_DATASOURCE_USERNAME`: 用于连接MySQL数据库的用户名
-	`SPRING_DATASOURCE_PASSWORD`: 连接MySQL数据库的密码
-	`SPRING_REDIS_HOST`: Eairp用于连接Redis数据库的主机地址
-	`SPRING_REDIS_PORT`: Redis数据库的端口，默认为6379。
-	`SPRING_REDIS_PASSWORD`: 连接Redis数据库的密码
-	`API_BASE_URL`: 用于前端访问后端API地址，可以设置域名或服务器IP，默认为*http://localhost:8088/erp-api*.

如果你需要执行一些高级配置，你可以通过运行以下命令在正在运行的Eairp容器内获得一个shell（但请注意，如果你删除容器，这些命令将不会被保存）：

```console
docker exec -it <eairp container id> bash -l
```

## 配置文件

您可能需要修改Eairp的4个重要配置文件：
-	`.env`
-	`start.sh`
-	`Dockerfile`
-	`docker-compose.yaml`

## 其他说明
暂时没有

# 许可证

Eairp使用 [Apache-2.0](https://github.com/eairps/eairp/blob/master/LICENSE-APACHE) 和 [MIT](https://github.com/eairps/eairp/blob/master/LICENSE-MIT) 许可。

# 支持

-	如果你想提出一个问题或改进的想法，请打开一个[Issues](https://github.com/eairps/eairp/issues)。
-	如果您有任何疑问，请使用[Eairp Discussions](https://github.com/eairps/eairp/discussions)。

# 贡献指南

-	如果你想在代码方面提供帮助，请在[Eairp](https://github.com/eairps/eairp)上发送Pull Requests。
-	请注意，更改需要合并到所有其他有意义的分支中，如果它们对现有标记有意义，则必须删除并重新创建这些标记。
-	此外，每当修改分支或标签时，[Eairp DockerHub官方镜像](https://hub.docker.com/repository/docker/wansenai/eairp/general)上都会显示Pull Request.

# 致谢

-	最初由 [James Zow](https://github.com/Jzow) 创建。
