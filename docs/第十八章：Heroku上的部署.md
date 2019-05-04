本文翻译自[The Flask Mega-Tutorial Part XVIII: Deployment on Heroku](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xviii-deployment-on-heroku)

这是Flask Mega-Tutorial系列的第十八部分，我将在其中部署Microblog到Heroku云平台。

在前面的文章中，我向你展示了托管Python应用程序的“传统”方式，并且我演示了两个部署到Linux的服务器的实际示例。 如果你不曾管理过Linux系统，那么你可能认为需要投入大量工作到这项任务中，而且肯定会有一个更简单的方法。

在本章中，我将向你展示一种完全不同的部署方法，该方法依赖第三方*云*托管提供程序来执行大部分管理任务，从而使你能够腾出更多时间处理应用程序。

许多云托管提供商提供了一个应用程序可以运行的托管平台。 你只需提供部署到这些平台上的实际应用程序，因为硬件，操作系统，脚本语言解释器，数据库等都由该服务管理。 这种服务称为[平台即服务](https://en.wikipedia.org/wiki/Platform_as_a_service)（PaaS）。

是不是感到难以置信？

我将把Microblog部署到[Heroku](http://heroku.com/)，这是一种流行的云托管服务，对Python应用程序也非常友好。 我选择Heroku不仅仅是因为它非常受欢迎，还因为它有一个免费的服务级别，可以让你跟随我并在不花钱的情况下完成部署。

*本章的GitHub链接为：[Browse](https://github.com/miguelgrinberg/microblog/tree/v0.18), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.18.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.17...v0.18).*

## 托管于Heroku

Heroku是首批PaaS平台之一。 它以Ruby的应用程序的托管服务开始，随后逐渐发展到支持诸多其他语言，如Java，Node.js和Python。

在Heroku中部署Web应用程序主要是通过`git`版本控制工具完成的，因此你必须将应用程序放在git代码库中。 Heroku在应用程序的根目录中查找名为*Procfile*的文件，以获取有关如何启动应用程序的描述。 对于Python项目，Heroku还期望*requirements.txt*文件列出需要安装的所有模块依赖项。 在通过git将应用程序上传到Heroku的服务器之后，你的工作基本就完成了，只需等待几秒钟，应用程序就会上线。 整个操作流程就是这么简单。

Heroku提供不同的服务级别，允许你自主选择为应用程序提供多少计算能力和运行时间，随着用户群的增长，你需要购买更多的“dynos”计算单元。

准备好了吗？让我们开始吧！

## 创建Heroku账户

在部署应用到Heroku之前，你需要拥有一个帐户。 所以请访问[heroku.com](https://id.heroku.com/signup)并创建一个免费账户。 一旦注册成功并登录到Heroku，你将可以访问一个dashboard，其中列出了你的所有应用程序。

## 安装Heroku命令行客户端

Heroku提供了一个名为[Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)的命令行工具来与服务交互，可安装于Windows，Mac OS X和Linux。 该文档包括了支持的所有平台的安装说明。 如果你计划部署应用程序以测试该服务，请将其安装在你的系统上。

安装CLI后应该做的第一件事是登录到你的Heroku帐户：

```
$ heroku login
```

Heroku CLI会要求你输入电子邮件地址和帐户密码。 你的身份验证状态将在随后的命令中被记住。

## 设置Git

`git`工具是Heroku应用程序部署的核心，因此如果你还没有安装它的话，则必须将它安装到你的系统上。 如果你没有可用于你的操作系统的安装包，可以访问[git site](https://git-scm.com/)下载安装程序。

使用`git`的原因很多并且都理由充分。 如果你打算部署应用到Heroku，那么这些原因就要又增加一个，因为要部署应用到Heroku，你的应用程序必须在`git`代码库中。 如果你要为Microblog执行测试部署，可以从GitHub克隆应用程序：

```
$ git clone https://github.com/miguelgrinberg/microblog
$ cd microblog
$ git checkout v0.18
```

`git checkout`命令将代码库切换到指定的历史提交点，也就是本章所处的位置。

如果更喜欢使用你自己的代码，你可以通过在顶层目录中运行`git init .`来将你自己的项目转换成`git`代码库（注意`init`后面的句号，它告诉git你想要在当前目录中初始化代码库）。

## 创建Heroku应用

要用Heroku注册一个新应用，需要在应用程序根目录下使用`apps:create`子命令，并将应用程序名称作为唯一参数传递：

```
$ heroku apps:create flask-microblog
Creating flask-microblog... done
http://flask-microblog.herokuapp.com/ | https://git.heroku.com/flask-microblog.git
```

Heroku要求应用程序的名称具有唯一性。 我上面已使用了`flask-microblog`这个名称，所以你需要为你的部署选择一个不同的名称。

该命令的输出将包含Heroku分配给应用程序的URL以及git代码库。 你的本地git代码库将配置一个额外的*remote*，称为`heroku`。 你可以用`git remote`命令验证它是否存在：

```
$ git remote -v
heroku  https://git.heroku.com/flask-microblog.git (fetch)
heroku  https://git.heroku.com/flask-microblog.git (push)
```

根据你创建git代码库的方式，上述命令的输出还可能包含另一个名为`origin`的远程仓库地址。

## 临时文件系统

Heroku平台与其他部署平台不同之处在于它在虚拟化平台上运行的文件系统是*临时*的。 那是什么意思？ 这意味着Heroku可以随时将运行你的应用的虚拟服务器重置为干净状态。 你不该天真地认为你保存到文件系统的任何数据都会被持久存储，事实上，Heroku经常回收服务器。

在这种条件下工作会为我的应用程序带来一些问题，因为它使用了如下的几个文件：

*   默认的SQLite数据库引擎将数据写入磁盘文件中
*   应用程序的日志也写入磁盘文件中
*   编译的语言翻译存储库同样是本地文件

以下部分将针对这三个方面提出解决方案。

## 使用Heroku Postgres数据库

为了解决第一个问题，我将切换到不同的数据库引擎。 在[第十七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E4%B8%83%E7%AB%A0%EF%BC%9ALinux%E4%B8%8A%E7%9A%84%E9%83%A8%E7%BD%B2.md)中，你看到我使用MySQL数据库为Ubuntu部署添加健壮性。 Heroku基于Postgres数据库提供了自己的数据库产品，因此我将转而使用它来避免使用基于文件的SQLite。

Heroku应用的数据库使用相同的Heroku CLI进行设置。 在本章中，我将创建一个免费级别的数据库：

```
$ heroku addons:add heroku-postgresql:hobby-dev
Creating heroku-postgresql:hobby-dev on flask-microblog... free
Database has been created and is available
 ! This database is empty. If upgrading, you can transfer
 ! data from another database with pg:copy
Created postgresql-parallel-56076 as DATABASE_URL
Use heroku addons:docs heroku-postgresql to view documentation
```

新创建的数据库的URL存储在`DATABASE_URL`环境变量中，该变量在应用程序运行时将可用。 这就非常方便了，因为应用程序已经设定为在该变量中查找数据库URL。

## 输出日志到标准输出

Heroku希望应用程序直接输出日志到`stdout`。 当你使用`heroku logs`命令时，应用程序打印到标准输出的任何内容都将被保存并返回。 所以我要添加一个配置变量，指示我是要输出日志到`stdout`，还是像我之前那样输出到文件。 这是配置的变化：

*config.py*：输出日志到标准输出的选项。

```
class Config(object):
    # ...
    LOG_TO_STDOUT = os.environ.get('LOG_TO_STDOUT')
```

然后在应用工厂函数中，我会检查此配置以了解应该如何配置应用程序的日志记录器：

`app/__init__.py`：输出日志到标准输出或文件。

```
def create_app(config_class=Config):
    # ...
    if not app.debug and not app.testing:
        # ...

        if app.config['LOG_TO_STDOUT']:
            stream_handler = logging.StreamHandler()
            stream_handler.setLevel(logging.INFO)
            app.logger.addHandler(stream_handler)
        else:
            if not os.path.exists('logs'):
                os.mkdir('logs')
            file_handler = RotatingFileHandler('logs/microblog.log',
                                               maxBytes=10240, backupCount=10)
            file_handler.setFormatter(logging.Formatter(
                '%(asctime)s %(levelname)s: %(message)s '
                '[in %(pathname)s:%(lineno)d]'))
            file_handler.setLevel(logging.INFO)
            app.logger.addHandler(file_handler)

        app.logger.setLevel(logging.INFO)
        app.logger.info('Microblog startup')

    return app
```

所以现在我需要在Heroku中运行应用程序时设置`LOG_TO_STDOUT`环境变量，但在其他配置中则不需要。 Heroku CLI使得做到这一点变得简单，因为它提供了一个选项来设置运行时使用的环境变量：

```
$ heroku config:set LOG_TO_STDOUT=1
Setting LOG_TO_STDOUT and restarting flask-microblog... done, v4
LOG_TO_STDOUT: 1
```

## 编译翻译

Microblog依赖本地文件的第三个方面是编译后的语言翻译文件。 确保这些文件永远不会从临时文件系统中消失的粗暴做法是将编译后的语言文件添加到git代码库，以便在部署到Heroku后它们成为应用程序初始状态的一部分。

在我看来，更优雅的选择是在Heroku的启动命令中包含`flask translate compile`命令，以便在服务器重新启动时再次编译这些文件。 我打算选择这个方案，因为我知道启动过程需要多个命令，至少我还需要运行数据库迁移。 所以现在，我将把这个问题放在一边，稍后当我写*Procfile*的时候会重新讨论它。

## 托管Elasticsearch

Elasticsearch是可以添加到Heroku项目中的众多服务之一，但与Postgres不同的是，这不是由Heroku提供的服务，而是由与Heroku合作提供附加组件的第三方提供的。 在我写这篇文章的时候，有三个不同的集成Elasticsearch服务提供商。

在配置Elasticsearch之前，请注意，Heroku要求你的帐户在安装任何第三方附加组件之前添加信用卡信息，即使你仍处于在免费级别中。 如果你不想将信用卡信息提供给Heroku，请跳过此部分。 你仍然可以部署应用程序，但搜索功能不起作用。

在可作为附加组件提供的Elasticsearch选项中，我决定尝试[SearchBox](https://elements.heroku.com/addons/searchbox)，它附带一个免费的初试计划。 要将SearchBox添加到你的帐户，你必须在登录到Heroku后运行以下命令：

```
$ heroku addons:create searchbox:starter
```

该命令将部署一个Elasticsearch服务，并将该服务的连接URL保存在与你的应用程序关联的`SEARCHBOX_URL`环境变量中。 请记住，除非将你的信用卡信息添加到你的Heroku帐户中，否则此命令将失败。

回忆一下[第十六章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e5%85%ad%e7%ab%a0%ef%bc%9a%e5%85%a8%e6%96%87%e6%90%9c%e7%b4%a2.md)，我的应用程序在Elasticsearch连接URL中查找的是`ELASTICSEARCH_URL`变量，所以我需要添加这个变量并将其设置为由SearchBox分配的连接URL：

```
$ heroku config:get SEARCHBOX_URL
<your-elasticsearch-url>
$ heroku config:set ELASTICSEARCH_URL=<your-elasticsearch-url>
```

在这里，我首先要求Heroku打印`SEARCHBOX_URL`的值，然后将其添加到一个名为`ELASTICSEARCH_URL`的新环境变量中。

## 更新依赖

Heroku期望依赖关系在*requirements.txt*文件中，就像我在[第十五章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e4%ba%94%e7%ab%a0%ef%bc%9a%e4%bc%98%e5%8c%96%e5%ba%94%e7%94%a8%e7%bb%93%e6%9e%84.md)中定义的那样。 但是为了在Heroku上运行应用程序，我需要为这个文件添加两个新的依赖关系。

Heroku不提供自己的Web服务器。 相反，它希望应用程序根据环境变量`$PORT`中给出的端口号启动自己的Web服务器。 由于Flask开发Web服务器不足以用于生产，因此我将再次使用[gunicorn](http://gunicorn.org/)，这是Heroku为Python应用程序推荐的服务器。

该应用程序还将连接到Postgres数据库，为此SQLAlchemy依赖`psycopg2`软件包的安装。

`gunicorn` 和`psycopg2` 都需要添加到*requirements.txt*文件中。

## Procfile

Heroku需要知道如何执行应用程序，并且它会在应用程序的根目录中使用名为*Procfile*的文件。 这个文件的格式很简单，每行包含一个进程名称，一个冒号，然后是启动进程的命令。 在Heroku上运行的最常见的应用程序类型是一个Web应用程序，对于这种类型的应用程序，进程名称应该是`web`。 下面你可以看到Microblog的*Procfile*：

*Procfile*：Heroku Procfile。

```
web: flask db upgrade; flask translate compile; gunicorn microblog:app
```

在这里，我定义的启动命令中将按顺序执行三个命令作以启动Web应用程序。 首先，我运行数据库迁移升级，然后编译语言翻译，最后启动服务器。

因为前两个子命令是基于`flask`命令的，所以我需要添加`FLASK_APP`环境变量：

```
$ heroku config:set FLASK_APP=microblog.py
Setting FLASK_APP and restarting flask-microblog... done, v4
FLASK_APP: microblog.py
```

`gunicorn`命令比我用于Ubuntu部署的还要简单，因为这个服务与Heroku环境有很好的集成。 例如，`$PORT`环境变量默认会被设置，取代使用`-w`选项来设置worker的数量，heroku推荐添加一个名为`WEB_CONCURRENCY`的环境变量，在`-w`参数没有提供的时候，就会使用这个环境变量，因此你可以灵活地控制worker的数量而无需修改Procfile。

## 部署应用

所有准备步骤都已完成，所以现在是时候执行部署了。 要将应用程序上传到Heroku的服务器进行部署，需要使用`git push`命令。 这与你将本地git代码库中的更改推送到GitHub或其他远程git服务器的方式类似。

现在我已经达到了最有趣的部分，就是将应用程序推送到我们的Heroku托管帐户。 这其实很简单，我只需要使用`git`将应用程序推送到Heroku git代码库的主分支就行了。 关于如何做到这一点有几种方法，取决于你是如何创建你的git代码库的。 如果你使用我的`v0.18`代码，那么你需要基于此标记创建一个分支，并将其作为远程主分支推送，如下所示：

```
$ git checkout -b deploy
$ git push heroku deploy:master
```

相反，如果你正在使用自己的代码库，那么你的代码已经在`master`分支中，所以你首先需要确保你的更改已经提交：

```
$ git commit -a -m "heroku deployment changes"
```

然后运行如下命令启动部署：

```
$ git push heroku master
```

无论你如何推送分支，都应该看到Heroku的以下输出：

```
$ git push heroku deploy:master
Counting objects: 247, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (238/238), done.
Writing objects: 100% (247/247), 53.26 KiB | 3.80 MiB/s, done.
Total 247 (delta 136), reused 3 (delta 0)
remote: Compressing source files... done.
remote: Building source:
remote:
remote: -----> Python app detected
remote: -----> Installing python-3.6.2
remote: -----> Installing pip
remote: -----> Installing requirements with pip
...
remote:
remote: -----> Discovering process types
remote:        Procfile declares types -> web
remote:
remote: -----> Compressing...
remote:        Done: 57M
remote: -----> Launching...
remote:        Released v5
remote:        https://flask-microblog.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy... done.
To https://git.heroku.com/flask-microblog.git
 * [new branch]      deploy -> master
```

我们在`git push`命令中使用的标签`heroku`是在创建应用程序时由Heroku CLI自动添加的远程代码库。 `deploy:master`参数意味着我将代码从本地代码库的`deploy`分支推送到Heroku代码库上的`master`分支。 当你使用自己的项目时，你可能会用`git push heroku master`命令推动你的本地`master`分支。 由于这个项目的代码库分支结构，我推送了一个非`master`的分支，但Heroku侧要求的目标分支是'master'，因为这是Heroku唯一接受部署的分支。

就这样，应用程序现在应该已经部署在创建应用程序的命令的输出中给出的URL上了。 在我的案例中，URL是 *https://flask-microblog.herokuapp.com* ，所以这就是我需要键入和访问该应用程序的URL。

如果你想查看正在运行的应用程序的日志，请使用`heroku logs`命令。 如果由于任何原因导致应用程序无法启动，该命令可能很有用。 如果有任何错误，将在日志中显示。

## 部署应用更新

要部署新版本的应用程序，只需要使用`git push`命令将新的代码库推送到Heroku即可。 这将重复部署过程，关停旧部署，然后用新代码替换它。 Procfile中的命令将作为新部署的一部分再次运行，因此在此过程中将更新任何新的数据库迁移或翻译内容。

