**[Archived]**

![Docker + Mysql](https://miro.medium.com/max/875/1*p3cYVYA9Gj8oCEf5tkkuow.png)

Hello, in this tutorial I will explain how to host MySQL Server in Docker using terminal and docker-compose only. Before we start, I assume you have understood about Docker and Docker Compose. If not, you can check various docker and docker compose tutorial on internet like [this one](https://www.youtube.com/watch?v=fqMOX6JJhGo) and [this one](https://docs.docker.com/compose/gettingstarted/).

# Prerequisites

- Linux Server (I'll use [Ubuntu 18.04 LTS](https://releases.ubuntu.com/18.04.4/))
- Docker ([get it here](https://docs.docker.com/get-docker/))
- Docker Compose ([get it here](https://docs.docker.com/compose/install/))
- Text Editor (I'll use Nano)

# Configure Docker Compose

Create a **docker.compose.yml** and write docker compose configurations using your favorite Text Editor like this.

```env
version: '3'
services:
    mysql:
        image: mysql:8
        environment:
            MYSQL_DATABASE: 'your_database'
            MYSQL_USER: 'daniel'
            MYSQL_PASSWORD: 'daniel'
            MYSQL_ROOT_PASSWORD: 'root'
    ports:
        - "3306:3306"
    volumes:
        - mysql_volume:/var/lib/mysql
volumes:
    mysql_volume:
```

Here's the explanation:

- **version**, in this file we will use version 3 compose.
- **image**, in this case we will use MySQL image version 8. If you need another version of the MySQL image, you can check other versions here.
- **environment**, this is the environment variable needed to configure database and access. We will create database with name "your_database", then we will create an user with name "daniel" and password "daniel" and also configure superuser account password with "root".
- **ports**, this port will be used to map your container host to your host port (in this case, your linux server port).
- **volumes**, we will store persist mysql data from container filesystem /var/lib/mysql. So, whenever mysql container is restarted or stopped, the data won't be erased.

Save the file then run docker-compose up in detached mode in docker-compose.yml directory.

```cmd
docker-compose up -d
```

All right, you can wait until your docker is pulling MySQL image from docker repository and configure it.

# Accessing MySQL Instance

So you have configured Docker Compose and run it. Now there are few ways to access MySQL:

- Access MySQL from Host
- Access MySQL from inside container
- Access MySQL from MySQL Workbench or any other GUI Tools.

## Access MySQL from Host

This one is a little bit tricky because you have to install mysql-shell in your host then you can access it from host terminal. You can follow instructions here.

After installation, try to access mysql container from host with following command.

```cmd
$ mysql -h localhost -P 3306 --protocol=tcp -u root -p
```

## Access MySQL from inside container

This is most easiest way to access MySQL and check whether your MySQL instance is okay or not without installing other applications. You can use container's bash.

First, see MySQL Container ID or Container Name using docker ps.

```cmd
$ docker ps
```

Then execute below command and fill container with either Container ID or Container Name.

Finally you can use MySQL inside container using following command.

```cmd
/# mysql -u root -p
```

## Access MySQL from MySQL Workbench or any other GUI Tools

If you prefer to use GUI Tools such as MySQL Workbench, you can connect it with following configurations:

- Connection Method : Standard (TCP/IP)
- Hostname : 127.0.0.1 (or your remote server address)
- Port : 3306
- Username: root (or using other user)
- Password: root (your root password)

That's all. Any comments or suggestions are welcome. Thanks for reading :)
