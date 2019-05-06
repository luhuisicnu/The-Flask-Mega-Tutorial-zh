本文翻译自[The Flask Mega-Tutorial Part XIX: Deployment on Docker Containers](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xix-deployment-on-docker-containers)

这是Flask Mega-Tutorial系列的第十九部分，我将在其中部署Microblog到Docker容器平台。

在[第十七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E4%B8%83%E7%AB%A0%EF%BC%9ALinux%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)中，你了解了传统部署，使用这种部署方式，你必须关注服务器配置的每个细节。 然后在[第十八章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%EF%BC%9AHeroku%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)我带你到另一个极端——Heroku ，这是一项完全掌控配置和部署任务的服务，使你能够全神贯注于应用程序。 在本章中，你将学习基于*容器*（尤其是在[Docker](https://www.docker.com/)容器平台）的第三种应用程序部署策略。 这种部署的工作量，介于另外两个选项之间。

容器建立在轻量级虚拟化技术的基础上，允许应用程序及其依赖和配置完全隔离宿主机地运行，而不需要使用虚拟机等完整的虚拟化解决方案。使用虚拟机需要更多的资源，并且有时可能与宿主机相比，性能显著下降。 配置为容器宿主机的系统可以运行大量容器，所有这些容器共享主机的内核并直接访问主机的硬件。 这与虚拟机不同，虚拟机必须模拟完整的系统，包括CPU，磁盘，其他硬件，内核等。

尽管必须共享内核，但容器中的隔离级别非常高。 容器具有自己的文件系统，并且可以基于容器宿主机使用不同的操作系统。 例如，你可以在Fedora宿主机上运行基于Ubuntu Linux的容器，反之亦然。 尽管容器是Linux操作系统上诞生的技术，但由于虚拟化的原因，也可以在Windows和Mac OS X宿主机上运行Linux容器。 这允许你在开发系统上测试部署操作，并且如果你愿意的话，还可以将容器合并到开发工作流程中去。

*本章的GitHub链接为：[Browse](https://github.com/miguelgrinberg/microblog/tree/v0.19), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.19.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.18...v0.19).*

## 安装Docker社区版

尽管Docker不是唯一的容器平台，但它是迄今为止最受欢迎的，所以我选择了它。 有两个版本的Docker，免费的社区版（CE）和付费的企业版（EE）。 对于本教程来说，Docker CE就够了。

要使用Docker CE，首先必须将其安装在系统上。 在[Docker网站](https://www.docker.com/community-edition)上有适用于Windows，Mac OS X和多个Linux发行版的安装程序。 如果你正在使用Microsoft Windows系统，请务必注意Docker CE依赖Hyper-V。 如有必要，安装程序将为你启用此功能，但请记住，启用Hyper-V会限制诸如VirtualBox等其他虚拟化技术产品的运行。

一旦Docker CE安装在你的系统上，你可以通过在终端窗口或命令提示符处输入以下命令来验证安装是否成功：

```
$ docker version
Client:
 Version:      17.09.0-ce
 API version:  1.32
 Go version:   go1.8.3
 Git commit:   afdb6d4
 Built:        Tue Sep 26 22:40:09 2017
 OS/Arch:      darwin/amd64

Server:
 Version:      17.09.0-ce
 API version:  1.32 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   afdb6d4
 Built:        Tue Sep 26 22:45:38 2017
 OS/Arch:      linux/amd64
 Experimental: true
```

## 构建容器镜像

为Microblog创建容器的第一步是为它构建一个*镜像*。 容器镜像是用于创建容器的模板。 它包含容器文件系统的完整表示，以及与网络，启动选项等相关的各种设置。

为应用程序创建容器镜像的最基本方法是启动一个要使用的基本操作系统（Ubuntu，Fedora等）容器，连接到运行在其中的bash shell进程，然后手动安装应用程序，可以参照我在[第十七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E4%B8%83%E7%AB%A0%EF%BC%9ALinux%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)中介绍的流程进行传统部署。 安装完所有内容后，你可以保存容器的快照，并生成容器镜像。 `docker`命令支持这种类型的工作流，但我不打算讨论这种方法，因为它非常不便，每次需要生成新镜像时都必须手动安装应用程序。

更好的方法是通过脚本生成容器镜像。 创建脚本化容器镜像的命令是`docker build`。 该命令从一个名为*Dockerfile*的文件读取并执行构建指令（我需要创建这些指令）。 Dockerfile基本上可以认为是一个安装程序脚本，它执行安装步骤来部署应用程序，以及一些容器特定的设置。

这是Microblog的一份基础的*Dockerfile*：

*Dockerfile*: Microblog的Dockerfile。

```
FROM python:3.6-alpine

RUN adduser -D microblog

WORKDIR /home/microblog

COPY requirements.txt requirements.txt
RUN python -m venv venv
RUN venv/bin/pip install -r requirements.txt
RUN venv/bin/pip install gunicorn

COPY app app
COPY migrations migrations
COPY microblog.py config.py boot.sh ./
RUN chmod +x boot.sh

ENV FLASK_APP microblog.py

RUN chown -R microblog:microblog ./
USER microblog

EXPOSE 5000
ENTRYPOINT ["./boot.sh"]
```

Dockerfile中的每一行都是一条命令。 `FROM`命令指定将在其上构建新镜像的基础容器镜像。 这样一来，你从一个现有的镜像开始，添加或改变一些东西，并最终得到一个派生的镜像。 镜像由名称和标签来标记，它们之间用冒号分隔。 该标签用作版本控制机制，允许容器镜像提供多个版本。 我选择的镜像的名称是`python`，它是Python的官方Docker镜像。 该镜像的标签允许你指定解释器版本和基础操作系统。 `3.6-alpine`标签选择安装在Alpine Linux上的Python 3.6解释器。 由于其体积小，Alpine Linux发行版比起更常见的发行版（例如Ubuntu）会更多地被使用。 你可以在[Python镜像库](https://hub.docker.com/r/library/python/tags/)中查看Python镜像可用的标签。

`RUN`命令在容器的上下文中执行任意命令。 这与你在shell提示符下输入命令相似。 `adduser -D microblog`命令创建一个名为`microblog`的新用户。 大多数容器镜像都使用`root`作为默认用户，但以root身份运行应用程序并不是一个好习惯，所以我创建了自己的用户。

`WORKDIR`命令设置将要安装应用程序的默认目录。 当我在上面创建`microblog`用户时，会自动创建了一个主目录，所以现在我将该目录设置为默认目录。 在Dockerfile中的任何剩余命令执行以及运行容器时，其当前目录为这个默认目录。

`COPY`命令将文件从你的机器复制到容器文件系统。 该命令需要两个或更多参数，源文件/目录和目标文件/目录。 源文件必须与Dockerfile所在的目录相关。 目的地可以是绝对路径，也可以是相对于在之前的`WORKDIR`命令中设置的目录的路径。 在这第一个`COPY`命令中，我将*requirements.txt*文件复制到容器文件系统的`microblog`用户的主目录中。

容器中有了*requirements.txt*文件，我就可以使用`RUN`命令创建一个虚拟环境。 首先我创建它，然后在其中安装所有依赖。 由于依赖文件仅包含通用依赖项，因此我明确安装*gunicorn*，以将其用作Web服务器。 当然，我也可以在我的*requirements.txt*文件中添加gunicorn。

接下来的三个`COPY`命令从顶级目录中复制*app*包，含有数据库迁移的*migrations*目录以及中的*microblog.py*和*config.py*脚本。 我还复制了一个新文件，*boot.sh*，我将在下面讨论它。

`RUN chmod`命令确保将这个新的*boot.sh*文件正确设置为可执行文件。 如果你使用的是基于Unix的文件系统，并且你的源文件已被标记为可执行文件，则复制的文件将会已是可执行的。 我显式地对其进行授权，是因为在Windows上很难设置可执行位。 如果你正在使用Mac OS X或Linux，你可能不需要这个步骤，但有了它也不会有什么问题。

`ENV`命令在容器中设置环境变量。我需要设置`FLASK_APP`，它是`flask`命令所依赖的。

下面的`RUN chown`命令将存储在 */home/microblog* 中的所有目录和文件的所有者设置为新的`microblog`用户。 尽管我在Dockerfile的顶部附近创建了该用户，但所有命令的默认用户仍为`root`，因此所有这些文件的属主都需要切换到`microblog`用户，以便在容器启动时该用户可以正确运行这些文件。

下一行中的`USER`命令使得这个新的`microblog`用户成为任何后续指令的默认用户，并且也是容器启动时的默认用户。

`EXPOSE`命令配置该容器将用于服务的端口。 这是必要的，以便Docker可以适当地在容器中配置网络。 我选择了标准的Flask端口5000，但这其实可以是任意端口。

最后，`ENTRYPOINT`命令定义了容器启动时应该执行的默认命令。 这是启动应用程序Web服务器的命令。 为了保持良好的代码组织逻辑，我决定为此创建一个单独的脚本，正是我之前复制到容器的*boot.sh*文件。 这里是这个脚本的内容：

*boot.sh*：Docker容器启动脚本。

```
#!/bin/sh
source venv/bin/activate
flask db upgrade
flask translate compile
exec gunicorn -b :5000 --access-logfile - --error-logfile - microblog:app
```

这是一个相当标准的启动脚本，与[第十七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E4%B8%83%E7%AB%A0%EF%BC%9ALinux%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)和[第十八章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%EF%BC%9AHeroku%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)的部署启动十分类似。 激活虚拟环境，执行迁移框架升级数据库，编译语言翻译，最后用gunicorn运行服务器。

请注意gunicorn命令之前的`exec`。 在shell脚本中，`exec`触发正在运行脚本的进程被给定的命令来替换掉，而不是将这个命令作为新进程启动。 这很重要，因为Docker会将容器的生命与其上运行的第一个进程关联起来。 在像这样的情况下，启动进程不是容器的主进程，你需要确保主进程取代启动进程，以确保容器不会提前停止。

Docker的一个有趣的方面是容器写入`stdout`或`stderr`的任何内容都将被捕获并存储为容器的日志。 出于这个原因，`-access-logfile`和`--error-logfile`都配置为`-`，它将日志发送到标准输出，以便它们作为日志由Docker存储。

Dockerfile写好后，我现在可以构建容器镜像了：

```
$ docker build -t microblog:latest .
```

我给`docker build`命令的`-t`参数设置了新容器镜像的名称和标签。 `.`表示容器构建的基础目录，这就是*Dockerfile*所在的目录。 构建过程将执行*Dockerfile*中的所有命令并创建镜像，该镜像将存储在你自己的机器上。

你可以使用`docker images`命令获取本地镜像的列表：

```
$ docker images
REPOSITORY    TAG          IMAGE ID        CREATED              SIZE
microblog     latest       54a47d0c27cf    About a minute ago   216MB
python        3.6-alpine   a6beab4fa70b    3 months ago         88.7MB
```

此列表将包含你的新镜像以及它的基础镜像。 每当你对应用程序进行更改后，都可以通过再次运行build命令来更新容器镜像。

## 启动容器

使用已创建的镜像，你现在可以运行应用程序的容器版本。 通过`docker run`命令，通常再搭配大量的参数，就可以完成容器的启动。 我将首先向你展示一个基本的例子：

```
$ docker run --name microblog -d -p 8000:5000 --rm microblog:latest
021da2e1e0d390320248abf97dfbbe7b27c70fefed113d5a41bb67a68522e91c
```

`--name`选项为新容器提供了一个名称。 `-d`选项告诉Docker在后台运行容器。 如果没有`-d`，容器将作为前台应用程序运行，从而阻塞你的命令提示符。 `-p`选项将容器端口映射到主机端口。 第一个端口是主机上的端口，右边的端口是容器内的端口。 上面的例子暴露了主机端口8000，其对应容器中的端口5000，因此即使内部容器使用5000，你也将在宿主机上访问端口8000来访问应用程序。 一旦容器停止，`--rm`选项将使其自动被删除。 虽然这不是必需的，但完成或中断的容器通常不再需要，因此可以自动删除。 最后一个参数是容器使用的容器镜像名称和标签。 运行上述命令后，可以在 *`http://localhost:8000`* 上访问该应用程序。

docker run的输出是分配给新容器的ID。 这是一个很长的十六进制字符串，在随后的命令中你可以使用它来引用容器。 实际上，只有前几个字符是必需的，足以保证ID的唯一性。

如果你想看看哪些容器正在运行，你可以使用`docker ps`命令：

```
$ docker ps
CONTAINER ID  IMAGE             COMMAND      PORTS                   NAMES
021da2e1e0d3  microblog:latest  "./boot.sh"  0.0.0.0:8000->5000/tcp  microblog
```

你可以看到，其实`docker ps`命令显示的是缩短了的容器ID。 如果你现在想停止容器，你可以使用`docker stop`：

```
$ docker stop 021da2e1e0d3
021da2e1e0d3
```

回顾一下，应用程序配置中有许多来自环境变量的选项。 例如，Flask密钥，数据库URL和电子邮件服务器选项都是从环境变量中导入的。 在上面的`docker run`例子中，我没有考虑这些，因此所有这些配置选项都将使用默认值。

在更实际的例子中，你将在容器内设置这些环境变量。 你在前面的章节看到，*Dockerfile*中的ENV命令设置了环境变量，对于将变为静态的变量来说，这是一个方便的选项。 但是，对于依赖于安装的变量，将它们作为构建过程的一部分并不方便，因为你希望容器镜像具有良好的可移植性。 如果你想将应用程序作为容器镜像提供给另一个人，你希望该人员能够按原样使用它，而不必使用不同的变量重新构建它。

所以构建时的环境变量可能很有用，但是也需要有可以通过`docker run`命令设置的运行时环境变量，对于这些变量，可以使用`-e`选项来设置。 以下示例设置了密钥和gmail帐户：

```
$ docker run --name microblog -d -p 8000:5000 --rm -e SECRET_KEY=my-secret-key \
    -e MAIL_SERVER=smtp.googlemail.com -e MAIL_PORT=587 -e MAIL_USE_TLS=true \
    -e MAIL_USERNAME=<your-gmail-username> -e MAIL_PASSWORD=<your-gmail-password> \
    microblog:latest
```

由于具有许多环境变量定义，`docker run`命令行非常长的情况并不罕见。

## 使用第三方“容器化”服务

Microblog的容器版本看起来不错，但我还没有真正考虑过很多关于存储的问题。 实际上，由于我没有设置`DATABASE_URL`环境变量，因此应用程序正在使用默认SQLite数据库并将数据存储在容器内部的文件系统上。 当你停止并删除容器时，你认为数据去哪里了？ 数据也会被删除！

容器中的文件系统是*临时的*，这意味着它随着容器的删除而删除。 你可以将数据写入容器内的文件系统，并且容器可以正常读写数据，但如果出于任何原因需要回收容器并将其替换为新的容器，则应用程序保存到容器内的任何数据将永远丢失。

容器应用程序的一个好的设计策略是保持应用程序容器*无状态*。 如果你的应用程序代码和数据容器没有任何问题，可以将其丢弃并替换为新的容器，容器变为真正的一次性容器，这在简化升级部署方面非常有用。

但是，这意味着数据必须放在应用程序容器之外的某个位置。 这就是神奇的Docker生态系统发挥作用的地方了。 Docker容器镜像仓库包含大量的容器镜像。你已经了解了Python容器镜像，我正在使用它作为我的Microblog容器的基础镜像。 除此之外，Docker还为Docker容器镜像仓库中的许多其他语言，数据库和其他服务维护镜像，如果这还不够，Docker容器镜像仓库还允许公司为其产品发布容器镜像，并且像你我这样的常规用户也可以发布自己的镜像。 这意味着安装第三方服务需要做出的努力会减少成只需在Docker容器镜像仓库中找到合适的镜像，并通过带有适当参数的`docker run`命令启动它。

所以我现在要做的是创建两个额外的容器，一个用于MySQL数据库，另一个用于Elasticsearch服务，然后我将加长启动Microblog容器的命令， 以使其能够访问这两个新的容器。


### 添加MySQL容器

像许多其他产品和服务一样，MySQL在Docker镜像仓库中提供了公共容器镜像。 就像我自己的Microblog容器一样，MySQL依赖于需要传递给`docker run`的环境变量。 他们配置了密码，数据库名称等。在镜像仓库中有许多MySQL镜像时，我决定使用由MySQL官方团队维护的镜像。 你可以在其镜像仓库页面找到有关MySQL容器镜像的详细信息： *https://hub.docker.com/r/mysql/mysql-server/* 。

回顾一下在[第十七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E4%B8%83%E7%AB%A0%EF%BC%9ALinux%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)中设置MySQL的繁琐过程，你就会赞叹在Docker中部署MySQL的轻松体验。 这里是启动MySQL服务器的`docker run`命令：

```
$ docker run --name mysql -d -e MYSQL_RANDOM_ROOT_PASSWORD=yes \
    -e MYSQL_DATABASE=microblog -e MYSQL_USER=microblog \
    -e MYSQL_PASSWORD=<database-password> \
    mysql/mysql-server:5.7
```

这就对了！ 在安装了Docker的任何机器上，你可以运行上面的命令，就会得到一个完成安装的MySQL服务器，它具有一个随机生成的root密码，一个名为`microblog`的全新数据库和一个名字相同的用户，该用户具备访问这个数据库的所有权限。 请注意，你需要输入正确的密码，以便它可以从`MYSQL_PASSWORD`环境变量获得。

现在在应用程序方面，我需要添加一个MySQL客户端软件包，就像我在Ubuntu上进行传统部署一样。 我将再次使用`pymysql`，我可以将它添加到*Dockerfile*中：

*Dockerfile*：添加pymysql到Dockerfile中。

```
# ...
RUN venv/bin/pip install gunicorn pymysql
# ...
```

任何时候对应用程序或*Dockerfile*进行更改后，都需要重建容器镜像：

```
$ docker build -t microblog:latest .
```

现在我可以再次启动Microblog，但是这次连接到数据库容器，以便两者都可以通过网络进行通信：

```
$ docker run --name microblog -d -p 8000:5000 --rm -e SECRET_KEY=my-secret-key \
    -e MAIL_SERVER=smtp.googlemail.com -e MAIL_PORT=587 -e MAIL_USE_TLS=true \
    -e MAIL_USERNAME=<your-gmail-username> -e MAIL_PASSWORD=<your-gmail-password> \
    --link mysql:dbserver \
    -e DATABASE_URL=mysql+pymysql://microblog:<database-password>@dbserver/microblog \
    microblog:latest
```

`--link`选项告诉Docker让正要运行的容器可以访问参数中指定的容器。 该参数包含由冒号分隔的两个名称。 第一部分是要链接的容器的名称或ID，在本例中是我在上面创建的一个名为`mysql`的容器。 第二部分定义了一个可以在这个容器中用来引用链接的主机名。 这里我使用`dbserver`作为代表数据库服务器的通用名称。

通过建立两个容器之间的链接，我可以设置`DATABASE_URL`环境变量，以便SQLAlchemy被引导使用其他容器中的MySQL数据库。 数据库URL将使用`dbserver`作为数据库主机名，`microblog`作为数据库名称和用户，以及你在启动MySQL时选择的密码。

我在试用MySQL容器时注意到的一件事是，这个容器需要几秒钟才能完全运行并准备好接受数据库连接。 如果启动MySQL容器，然后立刻启动应用容器，在*boot.sh*脚本尝试运行`flask db migrate`时，则可能会因数据库未准备好接受连接而失败。 为了使我的解决方案更加健壮，我决定在*boot.sh*中添加一个重试循环：

*boot.sh*：重试数据库连接。

```
#!/bin/sh
source venv/bin/activate
while true; do
    flask db upgrade
    if [[ "$?" == "0" ]]; then
        break
    fi
    echo Upgrade command failed, retrying in 5 secs...
    sleep 5
done
flask translate compile
exec gunicorn -b :5000 --access-logfile - --error-logfile - microblog:app
```

此循环检查`flask db upgrade`命令的退出代码，如果它不为零，则认为出现了问题，因此它会等待5秒钟然后重试。

### 添加Elasticsearch容器

[Elasticsearch Docker文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)演示了如何将该服务作为单一节点以用于开发模式，以及部署两个节点的生产环境服务。 现在，我将使用单节点模式，并使用引擎开源的“oss”镜像。 容器使用以下命令启动：

```
$ docker run --name elasticsearch -d -p 9200:9200 -p 9300:9300 --rm \
    -e "discovery.type=single-node" \
    docker.elastic.co/elasticsearch/elasticsearch-oss:6.1.1
```

这个`docker run`命令与我用于Microblog和MySQL的命令有很多相似之处，但是有一些有趣的区别。 首先，有两个`-p`选项，这意味着这个容器将在两个端口上而不是一个端口上进行监听。 端口9200和9300都映射到主机中的相同端口。

另一个区别在于用于引用容器镜像的语法。 对于我在本地构建的镜像，语法是`<name>:<tag>`。 MySQL容器使用格式为稍微更完整的`<account>/<name>:<tag>`语法，适用于在Docker镜像仓库中引用容器镜像。 我使用的Elasticsearch镜像遵循模式`<registry>/<account><name>:<tag>`，其中包括镜像仓库的地址作为第一个组件。 此语法用于未托管在Docker镜像仓库中的镜像。 在本处，Elasticsearch在*docker.elastic.co*上运行自己的容器镜像仓库服务，而不是使用由Docker维护的主镜像仓库。

所以，现在我已经启动并运行了Elasticsearch服务，我可以修改Microblog容器的启动命令以创建指向它的链接并设置Elasticsearch服务URL：

```
$ docker run --name microblog -d -p 8000:5000 --rm -e SECRET_KEY=my-secret-key \
    -e MAIL_SERVER=smtp.googlemail.com -e MAIL_PORT=587 -e MAIL_USE_TLS=true \
    -e MAIL_USERNAME=<your-gmail-username> -e MAIL_PASSWORD=<your-gmail-password> \
    --link mysql:dbserver \
    -e DATABASE_URL=mysql+pymysql://microblog:<database-password>@dbserver/microblog \
    --link elasticsearch:elasticsearch \
    -e ELASTICSEARCH_URL=http://elasticsearch:9200 \
    microblog:latest
```

在运行此命令之前，如果你仍然在运行Microblog容器，请先停止它。 还要仔细操作来为数据库设置正确的密码，并让Elasticsearch服务的参数处于命令中的恰当位置。

现在你应该可以访问 *`http://localhost:8000`* 并使用搜索功能。 如果你遇到任何错误，可以通过查看容器日志来对其进行排查。 你很可能希望查看Microblog容器的日志，其中将显示任何Python堆栈跟踪：

```
$ docker logs microblog
```

## Docker容器镜像仓库

现在我已经在Docker上使用三个容器来运行了完整的应用程序，其中两个容器来自公开的第三方镜像。 如果你想提供自己的容器镜像给其他人，那么你必须将它们推送到任何人都可以获取到的Docker镜像仓库中。

要访问Docker镜像仓库，你需要转到 *https://hub.docker.com* 并为自己创建一个帐户。 确保你选择一个你喜欢的用户名，因为这将用于你发布的所有镜像。

为了能够从命令行访问你的账户，你需要使用`docker login`命令登录：

```
$ docker login
```

如果你一直跟随我的引导，现在你的计算机上已经有一个名为`microblog:latest`的镜像存储在本地。 为了能够将这个镜像推送到Docker镜像仓库中，它需要重新命名以包含该帐户，正如来自MySQL的镜像。 这是通过`docker tag`命令完成的：

```
$ docker tag microblog:latest <your-docker-registry-account>/microblog:latest
```

如果你再次用`docker images`列出你的镜像，你会看到两个Microblog条目，一个是`microblog:latest`，另一个还包括你的帐户名。 它们实际上是同一镜像的两个别名。

要将镜像发布到Docker镜像仓库，请使用`docker push`命令：

```
$ docker push <your-docker-registry-account>/microblog:latest
```

现在你的镜像被公开了，你可以像MySQL和服务那样，说明如何安装它并从Docker镜像仓库运行。

## 容器化应用的部署

让你的应用程序在Docker容器中运行的最大的好处之一是，一旦该容器在你的本地测试通过了，就可以将它们运行到任何提供Docker支持的平台。 例如，你可以使用[第十七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E4%B8%83%E7%AB%A0%EF%BC%9ALinux%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)中推荐的Digital Ocean，Linode或Amazon Lightsail上的相同服务器。 即使这些提供商提供的最便宜的产品也足以让Docker运行一些容器。

[Amazon Container Service（ECS）](https://aws.amazon.com/ecs/)使你能够创建一个容器宿主机集群，以在其中运行容器。在集成完备的AWS环境中，提供了水平扩展和负载平衡，以及为容器镜像使用私有容器镜像仓库的功能。

最后，容器编排平台例如[Kubernetes](https://kubernetes.io/)通过允许你以简单的YAML格式文本文件描述你的多容器部署逻辑，来提供了更高级别的自动化和便利性， 负载均衡，水平扩展，密钥的安全管理以及滚动升级和回滚。

