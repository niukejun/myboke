---
title: 08 Image 构建最佳实践 - 入门
date: 2025-4-27
updated: 2025-4-27
categories: 计算机
tags:
  - docker
  - 学习
---

## 🛠️ 图像构建最佳实践

## 🔒 安全扫描

构建镜像后，最好使用 `docker scan` 命令扫描其中的安全漏洞。Docker 已与 [Snyk](http://snyk.io/) 合作提供漏洞扫描服务。

例如，要扫描 `getting-started` 您在教程中前面创建的图像，您只需输入

```js
docker scan getting-started
```

扫描使用不断更新的漏洞数据库，因此随着新漏洞的发现，您看到的输出会有所不同，但它可能看起来像这样：

```js
✗ Low severity vulnerability found in freetype/freetype
  Description: CVE-2020-15999
  Info: https://snyk.io/vuln/SNYK-ALPINE310-FREETYPE-1019641
  Introduced through: freetype/freetype@2.10.0-r0, gd/libgd@2.2.5-r2
  From: freetype/freetype@2.10.0-r0
  From: gd/libgd@2.2.5-r2 > freetype/freetype@2.10.0-r0
  Fixed in: 2.10.0-r1

✗ Medium severity vulnerability found in libxml2/libxml2
  Description: Out-of-bounds Read
  Info: https://snyk.io/vuln/SNYK-ALPINE310-LIBXML2-674791
  Introduced through: libxml2/libxml2@2.9.9-r3, libxslt/libxslt@1.1.33-r3, nginx-module-xslt/nginx-module-xslt@1.17.9-r1
  From: libxml2/libxml2@2.9.9-r3
  From: libxslt/libxslt@1.1.33-r3 > libxml2/libxml2@2.9.9-r3
  From: nginx-module-xslt/nginx-module-xslt@1.17.9-r1 > libxml2/libxml2@2.9.9-r3
  Fixed in: 2.9.9-r4
```

输出列出了漏洞的类型、了解更多信息的 URL，以及重要的是哪个版本的相关库修复了该漏洞。

[还有其他几个选项，您可以在docker scan 文档](https://docs.docker.com/engine/scan/) 中阅读 。

除了在命令行上扫描新构建的镜像之外，您还可以 [配置 Docker Hub](https://docs.docker.com/docker-hub/vulnerability-scanning/) 以自动扫描所有新推送的镜像，然后您可以在 Docker Hub 和 Docker Desktop 中查看结果。

![中心漏洞扫描](http://localhost/tutorial/image-building-best-practices/hvs.png)

## 图像分层

你知道吗？你可以查看图像的组成方式。使用该 `docker image history` 命令，你可以看到用于创建图像中每个图层的命令。

1. 使用该 `docker image history` 命令查看 `getting-started` 您在本教程前面创建的图像中的图层。
	```js
	docker image history getting-started
	```
	您应该得到类似这样的输出（日期/ID 可能不同）。
	```js
	IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
	05bd8640b718   53 minutes ago   CMD ["node" "src/index.js"]                     0B        buildkit.dockerfile.v0
	<missing>      53 minutes ago   RUN /bin/sh -c yarn install --production # b…   83.3MB    buildkit.dockerfile.v0
	<missing>      53 minutes ago   COPY . . # buildkit                             4.59MB    buildkit.dockerfile.v0
	<missing>      55 minutes ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
	<missing>      10 days ago      /bin/sh -c #(nop)  CMD ["node"]                 0B
	<missing>      10 days ago      /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B
	<missing>      10 days ago      /bin/sh -c #(nop) COPY file:4d192565a7220e13…   388B
	<missing>      10 days ago      /bin/sh -c apk add --no-cache --virtual .bui…   7.85MB
	<missing>      10 days ago      /bin/sh -c #(nop)  ENV YARN_VERSION=1.22.19     0B
	<missing>      10 days ago      /bin/sh -c addgroup -g 1000 node     && addu…   152MB
	<missing>      10 days ago      /bin/sh -c #(nop)  ENV NODE_VERSION=18.12.1     0B
	<missing>      11 days ago      /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
	<missing>      11 days ago      /bin/sh -c #(nop) ADD file:57d621536158358b1…   5.29MB
	```
	每条线代表图像中的一个层。此处的显示将基础层显示在底部，将最新层显示在顶部。使用此功能，您还可以快速查看每层的大小，从而帮助诊断大型图像。
2. 你会注意到有几行被截断了。如果你加上这个 `--no-trunc` 标志，你就能得到完整的输出（是的……用截断标志就能得到完整的输出，真有意思，不是吗？）
	```js
	docker image history --no-trunc getting-started
	```

## 层缓存

现在您已经看到了分层的实际作用，接下来需要学习一个重要的课程来帮助减少容器镜像的构建时间。

> 一旦某个层发生变化，所有下游层也必须重新创建

让我们再看一下我们使用的 Dockerfile...

```js
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
```

回到镜像历史记录的输出，我们看到 Dockerfile 中的每个命令都变成了镜像中的一个新层。你可能还记得，当我们修改镜像时，必须重新安装 yarn 依赖项。有没有什么办法可以解决这个问题？每次构建都传递相同的依赖项不太合理，对吧？

为了解决这个问题，我们需要重构 Dockerfile 以支持依赖项的缓存。对于基于 Node 的应用程序，这些依赖项在 Dockerfile `package.json` 文件中定义。那么，如果我们先只复制 Dockerfile 文件，安装依赖项，然后再复制其他所有文件，会怎么样 *？* 这样，只有在 Dockerfile 文件发生更改时，我们才需要重新创建 Yarn 依赖项 `package.json` 。这样就合理了吗？

1. 更新 Dockerfile 以首先复制 `package.json` ，安装依赖项，然后复制其他所有内容。
	```js
	FROM node:18-alpine
	WORKDIR /app
	COPY package.json yarn.lock ./
	RUN yarn install --production
	COPY . .
	CMD ["node", "src/index.js"]
	```
2. `.dockerignore` 在与 Dockerfile 相同的文件夹中 创建一个文件，其内容如下。
	```js
	node_modules
	```
	`.dockerignore` 文件是一种简单的方法，可以选择性地仅复制与图像相关的文件。您可以 [在此处](https://docs.docker.com/engine/reference/builder/#dockerignore-file) 阅读更多相关信息。在这种情况下， `node_modules` 第二步应该省略文件夹 `COPY` ，否则可能会覆盖该步骤中命令创建的文件。有关为什么建议在 Node.js 应用程序中使用此方法的更多详细信息以及其他最佳实践，请参阅 [Dockerizing a Node.js web app](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/) `RUN` 指南 。
3. 使用 构建一个新图像 `docker build` 。
	```js
	docker build -t getting-started .
	```
	您应该看到如下输出...
	```js
	[+] Building 16.1s (10/10) FINISHED
	=> [internal] load build definition from Dockerfile                                               0.0s
	=> => transferring dockerfile: 175B                                                               0.0s
	=> [internal] load .dockerignore                                                                  0.0s
	=> => transferring context: 2B                                                                    0.0s
	=> [internal] load metadata for docker.io/library/node:18-alpine                                  0.0s
	=> [internal] load build context                                                                  0.8s
	=> => transferring context: 53.37MB                                                               0.8s
	=> [1/5] FROM docker.io/library/node:18-alpine                                                    0.0s
	=> CACHED [2/5] WORKDIR /app                                                                      0.0s
	=> [3/5] COPY package.json yarn.lock ./                                                           0.2s
	=> [4/5] RUN yarn install --production                                                           14.0s
	=> [5/5] COPY . .                                                                                 0.5s
	=> exporting to image                                                                             0.6s
	=> => exporting layers                                                                            0.6s
	=> => writing image sha256:d6f819013566c54c50124ed94d5e66c452325327217f4f04399b45f94e37d25        0.0s
	=> => naming to docker.io/library/getting-started                                                 0.0s
	```
	你将看到所有层都已重建。这很好，因为我们对 Dockerfile 做了相当多的修改。
4. 现在，对文件进行更改 `src/static/index.html` （例如更改 `<title>` 为“The Awesome Todo App”）。
5. 现在再次使用以下命令构建 Docker 镜像 `docker build -t getting-started .`。这次，你的输出应该会有所不同。
	```js
	[+] Building 1.2s (10/10) FINISHED
	=> [internal] load build definition from Dockerfile                                               0.0s
	=> => transferring dockerfile: 37B                                                                0.0s
	=> [internal] load .dockerignore                                                                  0.0s
	=> => transferring context: 2B                                                                    0.0s
	=> [internal] load metadata for docker.io/library/node:18-alpine                                  0.0s
	=> [internal] load build context                                                                  0.2s
	=> => transferring context: 450.43kB                                                              0.2s
	=> [1/5] FROM docker.io/library/node:18-alpine                                                    0.0s
	=> CACHED [2/5] WORKDIR /app                                                                      0.0s
	=> CACHED [3/5] COPY package.json yarn.lock ./                                                    0.0s
	=> CACHED [4/5] RUN yarn install --production                                                     0.0s
	=> [5/5] COPY . .                                                                                 0.5s
	=> exporting to image                                                                             0.3s
	=> => exporting layers                                                                            0.3s
	=> => writing image sha256:91790c87bcb096a83c2bd4eb512bc8b134c757cda0bdee4038187f98148e2eda       0.0s
	=> => naming to docker.io/library/getting-started                                                 0.0s
	```
	首先，你应该注意到构建速度快了很多！你会发现有几个步骤使用了之前缓存的层。所以，太棒了！我们使用了构建缓存。推送和拉取镜像以及更新镜像的速度也会快很多。太棒了！

## 多阶段构建

虽然我们不会在本教程中深入探讨它，但多阶段构建是一个非常强大的工具，它通过使用多个阶段来帮助我们创建镜像。它们有几个优点，包括：

- 将构建时依赖项与运行时依赖项分开
- *仅* 发送 应用程序运行所需的内容，以减少整体图像大小

### Maven/Tomcat 示例

构建基于 Java 的应用程序时，需要 JDK 将源代码编译为 Java 字节码。但是，生产环境中不需要该 JDK。您可能还会使用 Maven 或 Gradle 等工具来构建应用。我们的最终镜像中也不需要这些工具。多阶段构建可以提供帮助。

```js
FROM maven AS build
WORKDIR /app
COPY . .
RUN mvn package

FROM tomcat
COPY --from=build /app/target/file.war /usr/local/tomcat/webapps
```

在此示例中，我们使用一个阶段（称为 `build` ）来使用 Maven 执行实际的 Java 构建。在第二个阶段（从 开始 `FROM tomcat` ），我们从该阶段复制文件 `build` 。最终镜像只是最后一个正在创建的阶段（可以使用 `--target` 标志覆盖）。

### React 示例

构建 React 应用时，我们需要一个 Node 环境来将 JS 代码（通常是 JSX）、SASS 样式表等编译成静态 HTML、JS 和 CSS。但如果我们不执行服务器端渲染，生产构建甚至不需要 Node 环境。为什么不将静态资源放在静态 nginx 容器中呢？

在这里，我们使用 `node:18` 镜像执行构建（最大化层缓存），然后将输出复制到 nginx 容器中。很酷吧？

## 🔍 回顾

通过了解镜像的结构，我们可以更快地构建镜像并减少变更。扫描镜像让我们确信正在运行和分发的容器是安全的。多阶段构建还可以通过将构建时依赖项与运行时依赖项分离，帮助我们减小镜像整体大小并提高最终容器的安全性。
