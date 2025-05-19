---
title: 07 使用 Docker Compose - 入门
date: 2025-4-27
updated: 2025-4-27
categories: 计算机
tags:
  - docker
  - 学习
---

## 🐳 使用 Docker Compose

[Docker Compose](https://docs.docker.com/compose/) 是一个用于定义和共享多容器应用程序的工具。使用 Compose，我们可以创建一个 YAML 文件来定义服务，然后使用一个命令就可以启动或关闭所有服务。

*使用 Compose 的一大* 优势 在于，您可以在一个文件中定义应用程序堆栈，并将其保存在项目仓库的根目录下（现在已受版本控制），并轻松允许其他人为您的项目做出贡献。其他人只需克隆您的仓库并启动 Compose 应用即可。事实上，您可能会看到 GitHub/GitLab 上现在有不少项目正在这样做。

那么，我们该如何开始呢？

## 🛠️ 安装 Docker Compose

如果您已在 Windows、Mac 或 Linux 系统上安装了 Docker Desktop，则说明您已经安装了 Docker Compose！Play-with-Docker 实例也已安装 Docker Compose。如果您使用的是其他系统，可以按照 [此处的说明](https://docs.docker.com/compose/install/) 安装 Docker Compose 。

## 📄 创建我们的Compose文件

1. 在应用程序文件夹中，创建一个名为（ 和文件 `docker-compose.yml` 旁边 ）的文件。 `Dockerfile` `package.json`
2. 在撰写文件中，我们首先定义想要作为应用程序的一部分运行的服务（或容器）列表。
	```js
	services:
	```

现在，我们将开始将一项服务迁移到撰写文件中。

## 🚀 定义应用服务

要记住，这是我们用来定义应用程序容器的命令。

```js
docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:18-alpine \
  sh -c "yarn install && yarn run dev"
```
1. 首先，让我们定义容器的服务入口和镜像。我们可以为服务选择任意名称。该名称将自动成为网络别名，这在定义 MySQL 服务时非常有用。
	```js
	services:
	  app:
	    image: node:18-alpine
	```
2. 通常，您会在定义附近看到该命令 `image` ，尽管对排序没有要求。所以，我们继续将其移动到文件中。
	```js
	services:
	  app:
	    image: node:18-alpine
	    command: sh -c "yarn install && yarn run dev"
	```
3. 让我们通过定义 服务的来迁移 `-p 3000:3000` 命令的一部分。这里我们将使用 [简短的语法](https://docs.docker.com/compose/compose-file/#short-syntax-2) ，但您也可以使用更详细的 [长语法](https://docs.docker.com/compose/compose-file/#long-syntax-2) 。 `ports`
	```js
	services:
	  app:
	    image: node:18-alpine
	    command: sh -c "yarn install && yarn run dev"
	    ports:
	      - 3000:3000
	```
4. 接下来，我们将使用 和定义 迁移工作目录 ( `-w /app`) 和卷映射 ( ) 。卷也有 [短语](https://docs.docker.com/compose/compose-file/#short-syntax-4) 法和 [长](https://docs.docker.com/compose/compose-file/#long-syntax-4) 语法。 `-v "$(pwd):/app"` `working_dir` `volumes`
	Docker Compose 卷定义的一个优点是我们可以使用当前目录的相对路径。
	```js
	services:
	  app:
	    image: node:18-alpine
	    command: sh -c "yarn install && yarn run dev"
	    ports:
	      - 3000:3000
	    working_dir: /app
	    volumes:
	      - ./:/app
	```
5. 最后，我们需要使用密钥迁移环境变量定义 `environment` 。
	```js
	services:
	  app:
	    image: node:18-alpine
	    command: sh -c "yarn install && yarn run dev"
	    ports:
	      - 3000:3000
	    working_dir: /app
	    volumes:
	      - ./:/app
	    environment:
	      MYSQL_HOST: mysql
	      MYSQL_USER: root
	      MYSQL_PASSWORD: secret
	      MYSQL_DB: todos
	```

### 🗄️ 定义 MySQL 服务

现在，是时候定义 MySQL 服务了。我们用于该容器的命令如下：

```js
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mysql:8.0
```
1. 我们首先定义新服务并命名， `mysql` 以便它自动获取网络别名。我们接下来还会指定要使用的镜像。
	```js
	services:
	  app:
	    # The app service definition
	  mysql:
	    image: mysql:8.0
	```
2. 接下来，我们将定义卷映射。当我们使用 运行容器时 `docker run` ，会自动创建指定的卷。但是，使用 Compose 运行时不会发生这种情况。我们需要在顶层 `volumes:`部分中定义卷，然后在服务配置中指定挂载点。只需提供卷名称，即可使用默认选项。 当然， [还有更多可用选项。](https://docs.docker.com/compose/compose-file/#volumes-top-level-element)
	```js
	services:
	  app:
	    # The app service definition
	  mysql:
	    image: mysql:8.0
	    volumes:
	      - todo-mysql-data:/var/lib/mysql
	volumes:
	  todo-mysql-data:
	```
3. 最后我们只需要指定环境变量即可。
	```js
	services:
	  app:
	    # The app service definition
	  mysql:
	    image: mysql:8.0
	    volumes:
	      - todo-mysql-data:/var/lib/mysql
	    environment:
	      MYSQL_ROOT_PASSWORD: secret
	      MYSQL_DATABASE: todos
	volumes:
	  todo-mysql-data:
	```

此时，我们的完整内容 `docker-compose.yml` 应该是这样的：

```js
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

## 🏁 运行我们的应用程序堆栈

现在我们有了 `docker-compose.yml` 文件，我们可以启动它了！

1. 首先确保没有其他 app/db 副本正在运行（ `docker ps` 和 `docker rm -f <ids>` ）。
2. 使用命令启动应用程序堆栈 `docker compose up` 。我们将添加 `-d` 标志以在后台运行所有内容。
	```js
	docker compose up -d
	```
	当我们运行这个程序时，我们应该看到如下输出：
	```js
	[+] Running 3/3
	⠿ Network app_default    Created                                0.0s
	⠿ Container app-mysql-1  Started                                0.4s
	⠿ Container app-app-1    Started                                0.4s
	```
	你会注意到，卷和网络都已创建！默认情况下，Docker Compose 会自动为应用程序堆栈专门创建一个网络（这就是我们没有在 Compose 文件中定义网络的原因）。
3. 让我们使用命令查看日志 `docker compose logs -f` 。您将看到来自每个服务的日志交错成单个流。当您想要观察与时间相关的问题时，这非常有用。该 `-f` 标志“跟随”日志，因此会在日志生成时为您提供实时输出。
	如果您还没有看到，您将看到如下输出...
	```js
	mysql_1  | 2022-11-23T04:01:20.185015Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.31'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
	app_1    | Connected to mysql db at host mysql
	app_1    | Listening on port 3000
	```
	服务名称显示在行首（通常以彩色显示），以帮助区分消息。如果要查看特定服务的日志，可以将服务名称添加到 logs 命令的末尾（例如 `docker compose logs -f app` ）。
4. 此时，你应该可以打开应用并看到它正在运行了。嘿！我们只剩下一个命令了！

## 在 Docker Dashboard 中查看我们的应用程序堆栈

**如果我们查看 Docker Dashboard，我们会看到一个名为app** 的组 。这是 Docker Compose 中的“项目名称”，用于将容器分组。默认情况下，项目名称就是 `docker-compose.yml` 所在目录的名称。

![带有应用程序项目的 Docker 仪表板](http://localhost/tutorial/using-docker-compose/dashboard-app-project-collapsed.png)

如果你向下旋转应用程序，你会看到我们在 Compose 文件中定义的两个容器。它们的名称也更具描述性，因为它们遵循 的模式 `<project-name>_<service-name>_<replica-number>` 。因此，你可以很容易地快速查看哪个容器是我们的应用程序，哪个容器是 MySQL 数据库。

![Docker 仪表板，其中应用程序项目已展开](http://localhost/tutorial/using-docker-compose/dashboard-app-project-expanded.png)

## 彻底摧毁一切

当你准备彻底卸载它时，只需运行 `docker compose down` 或点击 Docker Dashboard 上整个应用程序的垃圾桶即可。容器将停止运行，网络也将被移除。

拆掉之后，你就可以切换到另一个项目，运行它 `docker compose up` ，然后就可以为该项目做贡献了！真的没有比这更简单的了！

## 🔍 回顾

在本节中，我们了解了 Docker Compose，以及它如何帮助我们显著简化多服务应用程序的定义和共享。我们将使用的命令转换为适当的 Compose 格式，从而创建了一个 Compose 文件。

至此，本教程即将结束。由于我们使用的 Dockerfile 存在一个很大的问题，我们想介绍一些关于镜像构建的最佳实践。那就一起来看看吧！
