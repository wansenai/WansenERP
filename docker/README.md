# What is Eairp

[Eairp](https://github.com/wansenai/eairp) (Enterprise AI Resource Planning) is a comprehensive resource planning system for enterprises, aimed at optimizing and integrating various operational processes, improving management efficiency, and reducing operating costs. 
EAIRP includes multiple functions such as material procurement, financial budgeting, inventory management, billing management, and user role organization management. 
It also introduces advanced AI assistants to provide intelligent management solutions for enterprises.

# Table of contents
- [Introduction](#introduction)
- [How to use this image](#how-to-use-this-image)
    -	[Pulling existing image](#pulling-an-existing-image)
        -	[Using docker run](#using-docker-run)
        -	[Using docker-compose](#using-docker-compose)
    -	[Building](#building)
- [Upgrading Eairp](#upgrading-eairp)
- [Troubleshooting](#troubleshooting)
- [Details for the eairp image](#details-for-the-eairp-image)
    -	[Configuration Options](#configuration-options)
    -	[Configuration Files](#configuration-files)
    -	[Miscellaneous](#miscellaneous)
- [License](#license)
- [Support](#support)
- [Contribute](#contribute)
- [Credits](#credits)

# Introduction

The goal is to provide a production-ready Eairp system running in Docker. This is why:

-	The OS is based on Debian and not on some smaller-footprint distribution like Alpine
-	Several containers are used with Docker Compose: one for the mysql one for Redis and another for Eairp + Nginx. This allows the ability to run them on different machines.

# How to use this image

You should first install [Docker](https://www.docker.com/) on your machine.

Then there are several options:

1.	Pull the eairp image from DockerHub.
2.	Get the [sources of this project](https://github.com/wansenai/eairp) and build them.

## Pulling an existing image

You need to run 3 containers:

-	One for the Eairp image
-	One image for Eairp to connect to a MySQL database
-	One image for Eairp to connect to a Redis database

### Using docker run

Start by creating a dedicated docker network:

```console
docker network create -d bridge eairp-nw
```

Then run a container for your MySQL database and make sure it is configured to use UTF8 encoding.

#### Starting MySQL

We will bind mount two local directories to be used by the MySQL container:
-	one to be used at database initialization to set permissions (see below), 
-	another to contain the data put by Eairp inside the MySQL database, so that when you stop and restart MySQL you don't find yourself without any data.

For example:
-	`/usr/local/mysql-init`
-	`/usr/local/mysql`

You need to make sure these directories exist, and then copy the `eairp.sql` file to the `/usr/local/mysql-init` directory. (you can name it the way you want, for example `init.sql`).

Note: Make sure the directories you are mounting into the container are fully-qualified, and aren't relative paths.

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
OR
```console
docker run --net=eairp-nw -d --name mysql-eairp -p 3306:3306 -v /usr/local/mysql:/var/lib/mysql -v /usr/local/mysql-init:/docker-entrypoint-initdb.d -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_USER=eairp -e MYSQL_PASSWORD=123456 -e MYSQL_DATABASE=eairp -d mysql:8.3 --character-set-server=utf8mb4 --collation-server=utf8mb4_bin --explicit-defaults-for-timestamp=1
```

You should adapt the command line to use the passwords that you wish for the MySQL root password.

#### Starting Redis

We will bind mount a local directory to be used by the Redis container to contain the data put by Eairp inside the database, so that when you stop and restart Eairp you don't find yourself without any data. For example:

-	`/usr/local/redis/data`
-	`/usr/local/redis/redis.conf`

You need to make sure this directory exists, before proceeding.

Note Make sure the directory you specify is specified with the fully-qualified path, not a relative path.

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
OR
```console
docker run --net=eairp-nw -d --name redis-eairp -v /usr/local/redis/redis.conf:/etc/redis/redis.conf -v /usr/local/redis/data:/data -p 6379:6379 redis:latest
```

You should adapt the command line to use the passwords that you wish for the Redis password.

#### Starting Eairp

We will also bind mount a local directory for the Eairp log permanent directory (contains application config and state), for example:

-	`/usr/local/eairp`

Note Make sure the directory you specify is specified with the fully-qualified path, not a relative path.

Ensure this directory exists, and then run Eairp in a container by issuing one of the following command.

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
OR
```console
docker run --net=eairp-nw -d --name eairp -p 8088:8088 -p 3000:80 -v /usr/local/eairp:/application/log -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-eairp:3306/eairp -e SPRING_DATASOURCE_USERNAME=eairp -e SPRING_DATASOURCE_PASSWORD=123456 -e SPRING_REDIS_HOST=redis-eairp  -e SPRING_REDIS_PORT=6379 -e SPRING_REDIS_PASSWORD=123456 -e API_BASE_URL=https://eairp.cn/erp-api wansenai/eairp:latest
```

Note the Eairp uses Spring DataSource in Spring Boot as the data source to connect to the database

Please don’t forget to add the MySQL database connection environment variables (`SPRING_DATASOURCE_URL`) and the Redis database connection environment variables (`SPRING_REDIS_HOST`) with the names of the MySQL container and Redis container created earlier so that Eairp knows the location of its databases.

If you want to deploy on your server, please modify the value of the `API_BASE_URL` environment variable, for example:
-	`http://eairp.cn/erp-api`
-	`https://eairp.cn/erp-api`

### Using docker-compose

Another solution is to use the Docker Compose files we provide.

First you need to download the [eairp source code](https://github.com/wansenai/eairp/releases) to your local machine, then you must download 5 files from [eairp-docker](https://github.com/wansenai/eairp-docker) repository, they are:

-	`.env`
-	`Dockerfile`
-	`docker-compose.yaml`
-	`start.sh`
-	`mysql-scripts/eairp.sql`

Then copy these four files and the `mysql-scripts` folder to the Eairp source code directory structure, Note that `mysql-scripts` is a folder. You need to copy the folder in its entirety instead of the sql file inside.

The complete directory structure is as follows:

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

For reference here's a minimal Docker Compose file that you could use as an example (full example [here](https://github.com/wansenai/eairp-docker/blob/master/latest/docker-compose.yaml)):

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

## Building

This allows you to rebuild the Eairp docker image locally. Here are the steps:
-	```shell
    docker-compose up
    ```
-	Start a browser and point it to `http://localhost:3000`

Note that `docker-compose up` will automatically build the Eairp image on the first run. If you need to rebuild it you can issue `docker-compose up --build`. You can also build the image with `docker build . -t eairp:latest` for example.

You can also just build the image by issuing `docker build -t eairp .` and then use the instructions from above to start Eairp and the database using `docker run ...`

# Upgrading Eairp

You've installed an Eairp docker image and used it and now comes the time when you'd like to upgrade Eairp to a newer version.

If you've followed the instructions above you've mapped the Eairp permanent directory to a local directory on your host.

All you need to do to upgrade is to stop the running Eairp container and start the new version of it that you want to upgrade to. You should keep your Mysql container and Redis container running.

Note that your current Eairp configuration files (`start.sh`, `.env` and `application.yml`) will be preserved. 

You should always check the [Release Notes](https://github.com/wansenai/eairp/releases) for all releases that happened between your current version and the new version you're upgrading to, as there could be some manual steps to perform (such as updating your Eairp configuration files).

# Details for the eairp image

## Configuration Options

The first time you create a container out of the eairp image, a shell script (`/usr/local/bin/docker-entrypoint.sh`) is executed in the container to setup some configuration. The following environment variables can be passed:

-	`SPRING_DATASOURCE_URL`: Used to connect to the Mysql database JDBC address, Defaults port to 3306.
-	`SPRING_DATASOURCE_USERNAME`: Username for connecting to the MySQL database
-	`SPRING_DATASOURCE_PASSWORD`: Password for connecting to the MySQL database
-	`SPRING_REDIS_HOST`: Host address used by Eairp to connect to the Redis database
-	`SPRING_REDIS_PORT`: The port number of the Redis database,  Defaults to 6379.
-	`SPRING_REDIS_PASSWORD`: Password for connecting to the Redis database
-	`API_BASE_URL`: Used for front-end access to back-end API address, you can set your domain name or server IP, Defaults to *http://localhost:8088/erp-api*.

If you need to perform some advanced configuration, you can get a shell inside the running Eairp container by issuing the following (but note that these won't be saved if you remove the container):

```console
docker exec -it <eairp container id> bash -l
```

## Configuration Files

There are 4 important configuration files for Eairp that you may want to modify:
-	`.env`
-	`start.sh`
-	`Dockerfile`
-	`docker-compose.yaml`

## Miscellaneous
Other additional instructions will be added later

# License

Eairp is licensed under the [Apache-2.0](https://github.com/wansenai/eairp/blob/master/LICENSE-APACHE) and [MIT](https://github.com/wansenai/eairp/blob/master/LICENSE-MIT).

The Dockerfile repository is also licensed under the [Apache-2.0](https://github.com/wansenai/eairp-docker/blob/master/LICENSE).

# Support

-	If you wish to raise an issue or an idea of improvement use [Eairp Issues](https://github.com/wansenai/eairp/issues)
-	If you have questions, use the [Eairp Discussions](https://github.com/wansenai/eairp/discussions)

# Contribute

-	If you wish to help out on the code, please send Pull Requests on [Eairp Docker GitHub project](https://github.com/wansenai/eairp-docker)
-	Note that changes need to be merged to all other branches where they make sense and if they make sense for existing tags, those tags must be deleted and recreated.
-	In addition, whenever a branch or tag is modified, a Pull Request on the [DockerHub Eairp official image](https://hub.docker.com/repository/docker/wansenai/eairp/general) must be made 

# Credits

-	Originally created by [James Zow](https://github.com/Jzow)
