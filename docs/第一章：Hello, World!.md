本文翻译自[The Flask Mega-Tutorial Part I: Hello, World!](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world)

一趟愉快的学习之旅即将开始，跟随它你将学会用[Python](https://python.org/)和[Flask](http://flask.pocoo.org/)来创建Web应用。上面的视频包含了整个教程的内容预览（译者注：视频见原文）。通过学习本章内容，你将学会如何创建一个Flask项目，并在自己的电脑上运行一个简单的Flask Web应用。

教程中所有的代码示例都托管在GitHub上。虽然直接从GitHub下载代码可以节省写代码的步骤，但是我强烈建议你至少在前几章自己动手书写这些代码。一旦你熟悉了Flask和示例应用，一些繁琐重复的代码就可以直接从GitHub复制了。

在每章的开头，我都将提供三个GitHub的链接来帮助你顺畅地学习本章的内容。点击**Browse**链接会打开GitHub上Microblog项目在本章的对应代码库页面，不会包含之后章节的任何新增代码。而**Zip**链接则提供了这份代码库的zip打包文件的下载地址。如果点击**Diff**链接，打开的将会是本章节的代码变更信息。

*本章的GitHub链接为: [Browse](https://github.com/miguelgrinberg/microblog/tree/v0.1), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.1.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.0...v0.1).*

## 安装Python

你说你还没有安装Python？那还等什么！立马安装吧。如果操作系统默认没有提供Python安装包，可以从[Python官方网站](http://python.org/download/)下载。如果你使用Microsoft Windows操作系统并且打算使用WSL或者Cygwin，需要注意，不要在上面使用Windows版本的Python，而要使用类Unix版本，比如从Ubuntu获取（对应WSL）或从Cygwin上获取。

为了验证Python是否正确安装，你可以打开一个终端窗口并输入`python3`（如果不存在这个命令，那就输入`python`）。预期的输出如下：
```
$ python3
Python 3.5.2 (default, Nov 17 2016, 17:05:23)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> _
```
Python解释器中，光标不断闪烁，等待着你输入Python语句。在未来的章节中，你可以充分体会到交互式解释器的魅力。至少现在它能够帮你确认Python已经成功安装的事实。可以输入`exit()`并回车来退出交互式解释器。在Linux和Mac OS X操作系统上，按下快捷键Ctrl-D也可以快速退出交互式解释器。在Windows操作系统上，则是通过按下Ctrl-Z后跟上Enter快捷键来快速退出。

## 安装Flask

下一步开始安装Flask，在这之前我要告诉你安装Python三方包的最佳实践。

Python将所有三方包托管到一个公共仓库，任何人都能从这个公共仓库下载并安装所有的三方包。Python将三方包公共仓库命名为[PyPI](https://pypi.python.org/pypi)以表示Python Package Index的缩写(被一些人戏称为"cheese shop")。从PyPI上安装三方包非常简单，Python专门提供了一个名为`pip`的工具来解决这个问题（Python2.7中不含`pip`工具，需要单独安装）。

安装三方包时，使用`pip`命令如下：
```
$ pip install <package-name>
```

有趣的是，这个方法在大多数情况下不适用。假如Python解释器是全局安装的，所有用户都能使用，那么普通用户则没有权限来修改它，因此只能用管理员账户来执行安装操作。即使忽略操作的复杂性，使用这种全局安装的方式会发生什么？`pip`工具从PyPI上下载三方包并安装到全局Python目录下，即刻起，所有Python脚本都可以访问到这个三方包。想象这样一个场景，你之前用当时的最新版本Flask——0.11版本的Flask开发了一个Web应用，现在Flask已经更新到了0.12版本，你想要使用0.12版本的Flask开发第二个Web应用。但是，如果将Flask从0.11版本升级到0.12版本可能会导致第一个Web应用出现故障。解决这个问题的方法最好不过为旧Web应用安装和使用Flask0.11版本，为新Web应用安装和使用Flask0.12版本。

为了解决维护不同应用程序对应不同版本的问题，Python使用了*虚拟环境*的概念。 虚拟环境是Python解释器的完整副本。在虚拟环境中安装三方包时只会作用到虚拟环境，全局Python解释器不受影响。 那么，就为每个应用程序安装各自的虚拟环境吧。 虚拟环境还有一个好处，即它们由创建它们的用户所拥有，所以不需要管理员帐户。

我们先创建项目目录，我将这个应用命名为*microblog*：
```
$ mkdir microblog
$ cd microblog
```

如果你正在使用Python3，虚拟环境已经成为内置模块，可以直接通过如下命令来创建它：
```
$ python3 -m venv venv
```
> 译者注：这个命令不一定能够执行成功，比如译者在Ubuntu16.04环境下执行，提示需要先安装对应的依赖。`sudo apt-get install python3-venv`

使用这个命令来让Python运行`venv`包，它会创建一个名为`venv`的虚拟环境。 命令中的第一个“venv”是Python虚拟环境包的名称，第二个是要用于这个特定环境的虚拟环境名称。 如果你觉得这样很混乱，可以用你自定义的虚拟环境名字替换第二个`venv`。我习惯在项目目录中创建了名为`venv`的虚拟环境，所以无论何时`cd`到一个项目中，都会找到相应的虚拟环境。

请注意，在一些操作系统中，你可能需要在上面的命令中使用`python`而不是`python3`。 一些安装规范对Python 2.x版本使用`python`，对3.x版本使用`python3`，而另一些则将`python`映射到3.x版本。

命令执行完成后，当前目录下就会新增一个名为`venv`的目录来存储这个虚拟环境的相关文件。

如果你使用的Python版本低于3.4（包括2.7版本），则不会默认支持虚拟环境。 对于这些版本的Python，在创建虚拟环境之前，需要下载并安装称为[virtualenv](https://virtualenv.pypa.io/)的第三方工具。 一旦安装了virtualenv，你可以使用以下命令创建一个虚拟环境：
```
$ virtualenv venv
```

不管你用什么方法创建虚拟环境，创建完毕之后还需要激活才能够进入这个虚拟环境。 要激活你的全新虚拟环境，需使用以下命令：
```
$ source venv/bin/activate
(venv) $ _
```

如果你使用的是Microsoft Windows命令提示符窗口，则激活命令稍有不同：

```
$ venv\Scripts\activate
(venv) $ _
```

激活一个虚拟环境，终端会话的环境配置就会被修改，之后你键入`python`的时候，实际上是调用的虚拟环境中的Python解释器。 此外，终端提示符也被修改成包含被激活的虚拟环境的名称的格式。这种激活是临时的和私有的，因此在关闭终端窗口时它们将不会保留，也不会影响其他的会话。 那么，当你需要同时打开多个终端窗口来调试不同的应用时，每个终端窗口都可以激活不同的虚拟环境而不会相互影响。

成功创建和激活了虚拟环境之后，你可以安装Flask了，命令如下：
```
(venv) $ pip install flask
```

想要验证安装是否成功，可以打开Python解释器，并用*import*语句来导入它：
```
>>> import flask
>>> _
```

如果语句没有报错，那么恭喜你，Flask安装成功了！

## "Hello, World" Flask应用

[Flask网站](http://flask.pocoo.org/)展示了一个仅有五行代码的简单示例应用程序。 而我会告诉你一个稍微更复杂的例子，它将为你编写更大的应用程序提供一个很好的基础结构。

应用程序是存在于*包*中的。 在Python中，包含`__init__.py`文件的子目录被视为一个可导入的包。 当你导入一个包时，`__init__.py`会执行并定义这个包暴露给外界的属性。

那就创建一个名为`app`的包来存放整个应用吧。记得切换到*microblog*目录下，并执行如下命令：
```
(venv) $ mkdir app
```

并在其下创建文件`__init__.py`，输入如下的代码：
```
from flask import Flask

app = Flask(__name__)

from app import routes
```

上面的脚本仅仅是从flask中导入的类`Flask`，并以此类创建了一个应用程序对象。 传递给`Flask`类的`__name__`变量是一个Python预定义的变量，它表示当前调用它的模块的名字。当需要加载相关的资源，如我将在[第二章](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-ii-templates)讲到的模板文件，Flask就使用这个位置作为起点来计算绝对路径。 代码的最后，应用程序导入尚未存在的`routes`模块。

这段代码，乍一看可能会让人迷惑。

其一，这里有两个实体名为`app`。 `app`包由*app*目录和`__init__.py`脚本来定义构成，并在`from app import routes`语句中被引用。 `app`变量被定义为`__init__.py`脚本中的`Flask`类的一个实例，以至于它成为`app`包的属性。

其二，`routes`模块是在底部导入的，而不是在脚本的顶部。 最下面的导入是解决*循环导入*的问题，这是Flask应用程序的常见问题。 你将会看到`routes`模块需要导入在这个脚本中定义的`app`变量，因此将`routes`的导入放在底部可以避免由于这两个文件之间的相互引用而导致的错误。

那么在`routes`模块中有些什么？ 路由是应用程序实现的不同URL。 在Flask中，应用程序路由的处理逻辑被编写为Python函数，称为*视图函数*。 视图函数被映射到一个或多个路由URL，以便Flask知道当客户端请求给定的URL时执行什么逻辑。

这是需要写入到*app/routes.py*中的第一个视图函数的代码：
```
from app import app

@app.route('/')
@app.route('/index')
def index():
    return "Hello, World!"
```

这个视图函数简单到只返回一个字符串作为问候用语。 函数上面的两个奇怪的`＠app.route`行是*装饰器*，这是Python语言的一个独特功能。 装饰器会修改跟在其后的函数。 装饰器的常见模式是使用它们将函数注册为某些事件的回调函数。 在这种情况下，`＠app.route`修饰器在作为参数给出的URL和函数之间创建一个关联。 在这个例子中，有两个装饰器，它们将URL `/`和`/index`索引关联到这个函数。 这意味着，当Web浏览器请求这两个URL中的任何一个时，Flask将调用该函数并将其返回值作为响应传递回浏览器。这样做是为了在运行这个应用程序的时候会稍微有一点点意义。

要完成应用程序，你需要在定义Flask应用程序实例的顶层（译者注：也就是microblog目录下）创建一个命名为*microblog.py*的Python脚本。 它仅拥有一个导入应用程序实例的行：
```
from app import app
```

还记得两个`app`实体吗？ 在这里，你可以在同一句话中看到两者。 Flask应用程序实例被称为`app`，是`app`包的成员。`from app import app`语句从`app`包导入其成员`app`变量。 如果你觉得这很混乱，你可以重命名包或者变量。

只要确保所做的操作完全正确，那么你就可以看到如下面的项目结构图：
```
microblog/
  venv/
  app/
    __init__.py
    routes.py
  microblog.py
```

不管你信不信，这个应用的第一个版本现在完成了！ 但是在运行之前，需要通过设置`FLASK_APP`环境变量告诉Flask如何导入它：
```
(venv) $ export FLASK_APP=microblog.py
```

如果你使用Microsoft Windows操作系统，在上面的命令中使用`set`替换`export`。

万事俱备，只欠东风！运行如下命令来运行你的第一个Web应用吧：
```
(venv) $ flask run
 * Serving Flask app "microblog"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

服务启动后将处于阻塞监听状态，将等待客户端连接。 `flask run`的输出表明服务器正在运行在IP地址127.0.0.1上，这是本机的回环IP地址。 这个地址很常见，并有一个更简单的名字，你可能已经看过：*localhost*。 网络服务器监听在指定端口号等待连接。 部署在生产Web服务器上的应用程序通常会在端口443上进行监听，如果不执行加密，则有时会监听80，但启用这些端口需要root权限。 由于此应用程序在开发环境中运行，因此Flask使用自由端口5000。 现在打开您的网络浏览器并在地址栏中输入以下URL：
```
    http://localhost:5000/
```

或者，你也可以使用另一个URL：
```
    http://localhost:5000/index
```

应用程序路由映射执行了吗？ 第一个URL映射到`/`，而第二个映射到`/ index`。 这两个路由都与应用程序中唯一的视图函数相关联，所以它们产生相同的输出，即函数返回的字符串。 如果你输入任何其他网址，则会出现错误，因为只有这两个URL被应用程序识别。

![Hello, World!](http://upload-images.jianshu.io/upload_images/4961528-328651893e709b13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

完成演示之后，你可以按下Ctrl-C来停止Web服务。

真是可喜可贺！你已经成功地向成为一名Web开发者的道路上迈出了重要的第一步！


在结束本章节之前，我想提醒一下你，在终端会话中直接设置的环境变量不会永久生效，因此你不得不在每次新开终端时设定 `FLASK_APP` 环境变量，从 1.0 版本开始，Flask 允许你设置只会在运行`flask`命令时自动注册生效的环境变量，要实现这点，你需要安装 `python-dotenv`：
```
(venv) $ pip install python-dotenv
```

此时，在项目的根目录下新建一个名为 `.flaskenv` 的文件，其内容是：
```
FLASK_APP=microblog.py
```

通过此项设置，`FLASK_APP`就可以自动加载了，如果你钟爱手动设定环境变量，那也不错，只是记得每次启动终端后要设定它。

