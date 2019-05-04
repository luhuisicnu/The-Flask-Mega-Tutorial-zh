本文翻译自[The Flask Mega-Tutorial Part XVII: Deployment on Linux](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)

这是Flask Mega-Tutorial系列的第十七部分，我将把Microblog部署到Linux服务器。

在本章中，我将谈到Microblog应用生命周期中的一个里程碑，因为我将讨论如何将应用部署到生产服务器上，以便真实用户可以访问它。

部署的主题非常广泛，因此不可能在这里涵盖所有范畴。 本章致力于探讨传统托管方式，包括Ubuntu发行版的Linux服务器和流行的树莓派微机。 我将在后面的章节中介绍其他选项，例如云和容器部署。

*本章的GitHub链接为：[Browse](https://github.com/miguelgrinberg/microblog/tree/v0.17), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.17.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.16...v0.17).*

## 传统托管

当提到“传统托管”时，意思是应用是手动或通过原始服务器机器上的脚本安装部署的。 该过程涉及安装应用程序、其依赖项和生产规模的Web服务器，并配置系统以确保其安全。

当你要部署自己的项目时，要问的第一个问题是在哪找服务器。 目前有很多经济的托管服务。 只需每月5美元，[Digital Ocean](https://www.digitalocean.com/)，[Linode](https://www.linode.com/)或[Amazon Lightsail](https://amazonlightsail.com/)就可以租借一台虚拟化Linux服务器（Linode和Digital Ocean为其入门级服务器提供1GB RAM，而亚马逊仅提供512MB）给你运行部署实验。 如果你一分钱都不愿意花，那么[Vagrant](https://www.vagrantup.com/)和[VirtualBox](https://www.virtualbox.org/)组合而成的工具，可以让你在自己的计算机上创建一个与付费服务器类似的虚拟服务器。

就技术角度而言，该应用可以部署在任何主流操作系统上，包括各种开放源代码的Linux和BSD发行版以及商用的OS X（OS X是一个开源和商业的混种，因为它基于开源BSD衍生产品Darwin）和Microsoft Windows。

由于OS X和Windows是的桌面操作系统，不是作为服务器的最佳选择，因此不是首选。 Linux或BSD操作系统之间的选择很大程度上取决于爱好，所以我将选择其中更受欢迎的Linux。 而Linux发行版中，我将再次选择受欢迎的Ubuntu。

## 创建Ubuntu服务器

如果你有兴趣与我一起部署，那么就需要一台服务器才能开始工作。 为你推荐两种选择，一种是付费的，另一种是免费的。 如果你愿意花一点钱，可以在Digital Ocean，Linode或Amazon Lightsail上注册一个账户，并创建一个Ubuntu 16.04镜像的虚拟服务器。 你应该使用最低配置的服务器，在我写这篇文章的时候，三家的最低配置都是每月5美元。 开销是按照服务器启动的小时数进行比例计算的，因此，如果你创建服务器后，使用几个小时然后删除它，那么有可能你只需支付美分级别的费用。

免费的方案基于你的计算机上可以运行虚拟机。 要使用此选项，请在你的机器上安装[Vagrant](https://www.vagrantup.com/)和[VirtualBox](https://www.virtualbox.org/)，然后创建一个名为*Vagrantfile*的文件并用以下内容来描述虚拟机的规格：

*Vagrantfile*：Vagrant配置。

```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end
end
```

该文件配置了一个带有1GB RAM的Ubuntu 16.04服务器，你可以用其IP地址192.168.33.10来访问该服务器。 要创建服务器，请运行以下命令：

```
$ vagrant up
```

请参阅Vagrant [命令行文档](https://www.vagrantup.com/docs/cli/)了解其他管理虚拟服务器的选项。

## 使用SSH客户端

你的服务器处于后端，所以不需要像个人计算机上那样拥有桌面。 你可以通过SSH客户端连接到服务器，并运行命令行进行交互。 如果你使用的是Linux或Mac OS X，则可能已经安装了[OpenSSH](http://www.openssh.org/)。 如果你使用Microsoft Windows，[Cygwin](https://www.cygwin.com/)，[Git](https://git-scm.com/)和[Windows Subsystem for Linux](https://msdn.microsoft.com/en-us/commandline/wsl/about)提供OpenSSH，因此你可以安装这些选项中的任何一个。

如果你正在使用来自第三方提供商的虚拟服务器，则在创建服务器时，会为其分配IP地址。 你可以使用以下命令打开终端会话来连接到该服务器：

```
$ ssh root@<server-ip-address>
```

系统会提示你输入密码。密码已在创建服务器后自动生成并显示给你，或者你自己指定了密码。

如果你使用的是Vagrant VM，则可以使用以下命令打开终端会话：

```
$ vagrant ssh
```

如果你使用的是Windows并且拥有Vagrant虚拟机，请注意你需要从可以调用`ssh`命令的shell运行上述命令。

## 免密登录

如果你使用的是Vagrant虚拟机，那么可以跳过本节，因为你的虚拟机已正确配置为使用名为`ubuntu`的非root帐户，Vagrant不用输入密码就可以自动登录。

要是你使用的是虚拟服务器，则建议创建一个常规用户来完成你的部署工作，并配置此帐户以便在不使用密码的情况下登录，这么做最初看起来似乎是一个糟糕的主意， 之后你会发现它不仅更方便，而且更安全。

我将创建一个名为`ubuntu`的用户帐户（如果你愿意，可以使用其他名称）。 要创建这个用户，请使用前一节中的`ssh`指令登录到你的服务器的root帐户，然后键入以下命令来创建用户，给它`sudo`权限并最终切换到它：

```
$ adduser --gecos "" ubuntu
$ usermod -aG sudo ubuntu
$ su ubuntu
```

现在我要配置这个新的`ubuntu`帐户来使用[public key](http://en.wikipedia.org/wiki/Public-key_cryptography)认证，以便你可以免密登录。

先不管服务器上打开的终端会话，然后在本地计算机上启动第二个终端。 如果你使用的是Windows，这需要是可以访问`ssh`命令的终端，所以它可能是一个`bash`或者类似的提示符的终端，而不是本地的Windows终端。 在该终端会话中，检查 *~/.ssh* 目录的内容：

```
$ ls ~/.ssh
id_rsa  id_rsa.pub
```

如果目录列表显示如上所述的名为*id_rsa*和*id_rsa.pub*的文件，那么你已经有一个密钥。 如果没有这两个文件，或者根本没有 *~/.ssh* 目录，则你需要运行以下命令（也是OpenSSH工具集的一部分）来创建SSH密钥对：

```
$ ssh-keygen
```

此应用程序将提示你输入一些内容，为此我建议你在所有提示中按Enter以接受默认设置。 你当然也可以做一些设置，如果你知道这么做意味着什么的话。

运行此命令后，应该有上面列出的两个文件了。 文件*id_rsa.pub*是你的*公钥*，这是一个你将提供给第三方的文件，用于识别你的身份。 *id_rsa*文件是你的*私钥*，不应与任何人共享。

你现在需要将公钥配置为服务器中的*授权主机*。 在你自己的计算机上打开的终端上，将公钥打印到屏幕上：

```
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCjw....F8Xv4f/0+7WT miguel@miguelspc
```

这将是一个非常长的字符序列，显示时可能跨越多行（但实际上只有一行）。 你需要将此数据复制到剪贴板，然后切换回远程服务器上的终端，你将在其中运行以下命令来存储公钥：

```
$ echo <paste-your-key-here> >> ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
```

免密登录现在应该可以工作了。 背后逻辑是，你机器上的`ssh`会用私钥执行加密操作来向服务器标识自己。 然后服务器使用你的公钥验证操作是否有效。

你现在可以注销`ubuntu`会话，然后注销`root`会话，然后尝试直接登录到`ubuntu`帐户：

```
$ ssh ubuntu@<server-ip-address>
```

这一次不用输入密码就登录了！

## 保护你的服务器

为了最大限度地降低服务器受到攻击的风险，你可以采取一些措施来关闭攻击者可能访问的大量潜在漏洞。

我要做的第一个更改是禁用root用户通过SSH登录。 你现在可以无密码地访问`ubuntu`帐户，并且可以通过`sudo`从该帐户运行管理员命令，因此实际上不需要暴露root帐户。 要禁用root登录，你需要编辑服务器上的 */etc/ssh/sshd_config* 文件。 你可能在你的服务器上安装了`vi`和`nano`文本编辑器，你可以用它来编辑文件（如果你不熟悉这两种文件编辑器，可以首先尝试`nano`）。 由于SSH配置对普通用户是不可访问的，所以你需要在编辑器命令前添加`sudo`（即`sudo vi /etc/ssh/sshd_config`）。 你需要更改此文件中的单行：

*/etc/ssh/sshd_config*：禁止root登录。

```
PermitRootLogin no
```

请注意，要进行此更改，你需要找到以`PermitRootLogin`开头的行（找不到就新建一行）并将该值更改为`no`。

下一个更改在同一个文件中。 现在我要为所有帐户禁用密码登录。 你有一个无密码的登录设置，所以没有必要允许密码。 如果你对完全禁用密码感到紧张，可以跳过此更改，但对于生产服务器来说，这是一个非常好的主意，因为攻击者经常在所有服务器上尝试随机帐户名和密码并希望能中奖。 要禁用密码登录，请在 */etc/ssh/sshd_config* 中更改以下行：

*/etc/ssh/sshd_config*：禁用密码登录。

```
PasswordAuthentication no
```

完成编辑SSH配置后，需要重新启动ssh服务以使更改生效：

```
$ sudo service ssh restart
```

我要做的第三个改变是安装*防火墙*。 这是一个阻止在任何未明确启用的端口上访问服务器的软件：

```
$ sudo apt-get install -y ufw
$ sudo ufw allow ssh
$ sudo ufw allow http
$ sudo ufw allow 443/tcp
$ sudo ufw --force enable
$ sudo ufw status
```

这些命令会安装[ufw](https://wiki.ubuntu.com/UncomplicatedFirewall)（简单防火墙），并将其配置为仅允许端口22（ssh），80（http）和443（https）上的外部通信。 任何其他端口将不被允许。

## 安装基础依赖

如果你遵循了我的建议并配置了Ubuntu 16.04发行版的服务器，那么你的系统完全支持Python 3.5，因此这是我将用于部署的Python版本。

基础的Python解释器可能已经预先安装在你的服务器上，但有一些额外的软件包可能却没有，而且Python之外还有一些其他软件包可用于创建健壮的生产环境部署。 对于数据库服务器，我将从SQLite切换到MySQL。 Postfix包是一个邮件传输代理，我将用它来发送电子邮件。 Supervisor工具将监视Flask服务器进程，并在其崩溃时自动重启，并当Supervisor服务重启后自动启动其监视的服务。 Nginx服务器将接受来自外部世界的所有请求，并将它们转发给应用程序。 最后，我将使用git来从git仓库下载应用程序。

```
$ sudo apt-get -y update
$ sudo apt-get -y install python3 python3-venv python3-dev
$ sudo apt-get -y install mysql-server postfix supervisor nginx git
```

这些安装大部分是无人值守的，但是在运行第三条安装语句到一定进度时，系统会提示你为MySQL服务选择一个root密码，并且还会询问关于安装postfix软件包的一些问题，你可以接受他们的默认答案。

请注意，对于此部署，我选择不安装Elasticsearch。 这项服务需要大量的RAM，所以只有拥有超过2GB内存的大型服务器时才可以考虑。 为了避免服务器内存不足的问题，我将停用搜索功能。 如果你有高配的服务器，可以从[Elasticsearch站点](https://elastic.co/)下载官方的.deb软件包，并按照其安装说明将其添加到你的服务器。 请注意，Ubuntu 16.04软件包存储库中提供的Elasticsearch软件包太旧，无法运行，你需要6.x或更高版本。

我还注意到，默认安装的postfix可能不足以在生产环境中发送电子邮件。 为了避免垃圾邮件和恶意邮件，很多服务器都要求发件人服务器通过安全扩展标识自己，这意味着至少你必须拥有与你的服务器相关联的域名。 如果你想了解如何完全配置电子邮件服务器以使其通过标准安全测试，请参阅以下Digital Ocean的指南：

*   [Postfix Configuration](http://do.co/2FhdIes)
*   [Adding an SPF Record](http://do.co/2Ff8ksk)
*   [DKIM Installation and Configuration](http://do.co/2HW2oTD)

## 安装应用

现在我要使用`git`从我的GitHub代码库下载Microblog源代码。 如果你不熟悉git源码控制，我建议你阅读[git for beginners](http://ryanflorence.com/git-for-beginners/)。

要将应用下载到服务器，请确保你位于`ubuntu`用户的主目录中，然后运行：

```
$ git clone https://github.com/miguelgrinberg/microblog
$ cd microblog
$ git checkout v0.17
```

这会将代码克隆到你的服务器上，并将其同步到本章的内容。 如果你在学习本教程的过程中维护了自己的git代码库，则可以将代码库URL更改为你的URL，在这种情况下，你可以跳过`git checkout`命令。

现在我需要创建一个虚拟环境并使用所有的包依赖项来填充它，在[第十五章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e4%ba%94%e7%ab%a0%ef%bc%9a%e4%bc%98%e5%8c%96%e5%ba%94%e7%94%a8%e7%bb%93%e6%9e%84.md)中，我已将依赖包的列表保存到*requirements.txt*文件中：

```
$ python3 -m venv venv
$ source venv/bin/activate
(venv) $ pip install -r requirements.txt
```

除了*requirements.txt*中的包之外，我还将使用此生产部署指定的两个包，因此它们不包含在*requirements.txt*文件中。 `gunicorn`软件包是Python应用程序的生产Web服务器。 `pymysql`软件包包含MySQL驱动程序，它使SQLAlchemy能够与MySQL数据库一起工作：

```
(venv) $ pip install gunicorn pymysql
```

我需要创建一个 *.env* 文件，其中包含所有需要的环境变量：

*/home/ubuntu/microblog/.env*：环境配置。

```
SECRET_KEY=52cb883e323b48d78a0a36e8e951ba4a
MAIL_SERVER=localhost
MAIL_PORT=25
DATABASE_URL=mysql+pymysql://microblog:<db-password>@localhost:3306/microblog
MS_TRANSLATOR_KEY=<your-translator-key-here>
```

这个 *.env* 文件与我在[第十五章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e4%ba%94%e7%ab%a0%ef%bc%9a%e4%bc%98%e5%8c%96%e5%ba%94%e7%94%a8%e7%bb%93%e6%9e%84.md)展示的非常类似，但是我为SECRET_KEY使用了一个随机字符串。 为了生成这个随机字符串，我使用了下面的命令：

```
python3 -c "import uuid; print(uuid.uuid4().hex)
```

对于`DATABASE_URL`变量，我定义了一个MySQL URL。 我将在下一节中向你介绍如何配置数据库。

我需要将`FLASK_APP`环境变量设置为应用程序的入口点以启用`flask`命令，但在解析 *.env* 文件之前需要此变量，因此需要手动设置。 为避免每次都设置它，我把它添加到`ubuntu`帐户的 *~/.profile* 文件的底部，以便每次登录时自动设置它：

```
$ echo "export FLASK_APP=microblog.py" >> ~/.profile
```

如果你注销并重新登录，现在`FLASK_APP`就已经设置好了。 你可以通过运行`flask --help`来确认它是否已经设置好了。 如果帮助信息显示应用程序已添加的`translate`命令，那么你就知道应用程序已被找到。

现在`flask`命令是有效的，我可以编译语言翻译：

```
(venv) $ flask translate compile
```

## 设置MySQL

我在开发过程中使用过的sqlite数据库非常适合简单的应用程序，但是当部署可能需要一次处理多个请求的健壮Web服务器时，最好使用更强大的数据库。 出于这个原因，我要建立一个名为'microblog'的MySQL数据库。

要管理数据库服务器，我将使用`mysql`命令，该命令应该已经安装在你的服务器上：

```
$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.19-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

请注意，你需要键入你在安装MySQL时选择的MySQL root密码才能访问MySQL命令提示符。

这些是创建名为`microblog`的新数据库的命令，以及具有完全访问权限的同名用户：

```
mysql> create database microblog character set utf8 collate utf8_bin;
mysql> create user 'microblog'@'localhost' identified by '<db-password>';
mysql> grant all privileges on microblog.* to 'microblog'@'localhost';
mysql> flush privileges;
mysql> quit;
```

你将需要用你选择的密码来替换`<db-password>`。 这将是`microblog`数据库用户的密码，所以不要使用你已为root用户选择的密码。 `microblog`用户的密码需要与你包含在 *.env* 文件中的`DATABASE_URL`变量中的密码相匹配。

如果你的数据库配置是正确的，你现在应该能够运行数据库迁移以创建所有的表：

```
(venv) $ flask db upgrade
```

继续下一步之前，确保上述命令成功完成且不会产生任何错误。

## 设置Gunicorn和Supervisor

当你使用`flask run`运行服务器时，正在使用的是Flask附带的Web服务器。 该服务器在开发过程中非常有用，但它不适合用于生产服务器，因为它不考虑性能和稳健性。 取而代之，我决定使用[gunicorn](http://gunicorn.org/)，它是一个纯粹的Python Web服务器，但与Flask不同，它是一个支持高并发的强大生产服务器，同时它也非常容易使用。

要在gunicorn下启动Microblog，你可以使用以下命令：

```
(venv) $ gunicorn -b localhost:8000 -w 4 microblog:app
```

`-b`选项告诉gunicorn在哪里监听请求，我在8000端口上监听了内部网络接口。 在没有外部访问的情况下运行Python Web应用程序通常是一个好主意，然后还需要一个非常快速的Web服务器，它可以优化来自客户端的所有静态文件的请求。 这个快速的Web服务器将直接提供静态文件，并将用于应用程序的任何请求转发到内部服务器。 我将在下一节中向你展示如何将nginx设置为面向公众的服务器。

`-w`选项配置gunicorn将运行多少*worker*。 拥有四个进程可以让应用程序同时处理多达四个客户端，这对于Web应用程序通常足以处理大量客户端请求，因为并非所有客户端都在不断请求内容。 根据服务器的RAM大小，你可能需要调整worker数量，以免内存不足。

`microblog:app`参数告诉gunicorn如何加载应用程序实例。 冒号前的名称是包含应用程序的模块，冒号后面的名称是此应用程序的名称。

虽然gunicorn的设置非常简单，但从命令行运行服务器在生产服务器实际上不是一个恰当的方案。 我想要做的是让服务器在后台运行，并持续监视，因为如果由于某种原因导致服务器崩溃并退出，我想确保新的服务器自动启动以取代它。 而且我还想确保如果机器重新启动，服务器在启动时自动运行，而无需人工登录和启动。 我将使用上面安装的[supervisor](http://supervisord.org/)包来执行此操作。

Supervisor使用配置文件定义它要监视什么程序以及如何在必要时重新启动它们。 配置文件必须存储在 */etc/supervisor/conf.d* 中。 这是Microblog的配置文件，我将其称为*microblog.conf*：

*/etc/supervisor/conf.d/microblog.conf*：Supervisor配置。

```
[program:microblog]
command=/home/ubuntu/microblog/venv/bin/gunicorn -b localhost:8000 -w 4 microblog:app
directory=/home/ubuntu/microblog
user=ubuntu
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
```

`command`，`directory`和`user`设置告诉supervisor如何运行应用程序。 如果计算机启动或崩溃，`autostart`和`autorestart`设置会使microblog自动重新启动。 `stopasgroup`和`killasgroup`选项确保当supervisor需要停止应用程序来重新启动它时，它仍然会调度成顶级gunicorn进程的子进程。

编写此配置文件后，必须重载supervisor服务的配置才能导入它：

```
$ sudo supervisorctl reload
```

像这样，这个gunicorn web服务器就已经启动和运行，并处于监控之中！

## 设置Nginx

由gunicorn启动的microblog应用服务器现在运行在本地端口8000。 我现在需要做的是将应用程序暴露给外部世界，为了使面向公众的web服务器能够被访问，我在防火墙上打开了两个端口（80和443）来处理应用程序的Web通信。

我希望这是一个安全的部署，所以我要配置端口80将所有流量转发到将要加密的端口443。 我将首先创建一个SSL证书。创建一个*自签名SSL证书*，这对于测试是可以的，但对于真正的部署不太好，因为Web浏览器会警告用户，证书不是由可信证书颁发机构颁发的。 创建microblog的SSL证书的命令是：

```
$ mkdir certs
$ openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
  -keyout certs/key.pem -out certs/cert.pem
```

该命令将要求你提供关于应用程序和你自己的一些信息。 这些信息将包含在SSL证书中，如果用户请求查看它，Web浏览器则会向用户显示它们。上述命令的结果将是名为*key.pem*和*cert.pem*的两个文件，我将其放置在Microblog根目录的*certs*子目录中。

要有一个由nginx服务的网站，你需要为它编写配置文件。 在大多数nginx安装中，这个文件需要位于 */etc/nginx/sites-enabled* 目录中。Nginx在这个位置安装了一个我不需要的测试站点，所以我将首先删除它：

```
$ sudo rm /etc/nginx/sites-enabled/default
```

下面你可以看到Microblog的nginx配置文件，它在 */etc/nginx/sites-enabled/microblog* 中：

*/etc/nginx/sites-enabled/microblog*：Nginx配置。

```
server {
    # listen on port 80 (http)
    listen 80;
    server_name _;
    location / {
        # redirect any requests to the same URL but on https
        return 301 https://$host$request_uri;
    }
}
server {
    # listen on port 443 (https)
    listen 443 ssl;
    server_name _;

    # location of the self-signed SSL certificate
    ssl_certificate /home/ubuntu/microblog/certs/cert.pem;
    ssl_certificate_key /home/ubuntu/microblog/certs/key.pem;

    # write access and error logs to /var/log
    access_log /var/log/microblog_access.log;
    error_log /var/log/microblog_error.log;

    location / {
        # forward application requests to the gunicorn server
        proxy_pass http://localhost:8000;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /static {
        # handle static files directly, without forwarding to the application
        alias /home/ubuntu/microblog/static;
        expires 30d;
    }
}
```

Nginx的配置不易理解，但我添加了一些注释，至少你可以知道每个部分的功能。 如果你想获得关于特定指令的信息，请参阅[nginx官方文档](https://nginx.org/en/docs/)。

添加此文件后，你需要告诉nginx重新加载配置以激活它：

```
$ sudo service nginx reload
```

现在应用程序应该部署成功了。 在你的Web浏览器中，可以键入服务器的IP地址（如果使用的是Vagrant VM，则为192.168.33.10），然后该服务器将连接到应用程序。 由于你使用的是自签名证书，因此将收到来自Web浏览器的警告，你必须解除该警告。

使用上述说明为自己的项目完成部署之后，我强烈建议你将自签名证书替换为真实的证书，以便浏览器不会在用户访问你的网站时发出警告。 为此，你首先需要购买域名并将其配置为指向你的服务器的IP地址。 一旦你有一个域名，你可以申请一个免费的[Let's Encrypt](https://letsencrypt.org/) SSL证书。 我在博客上写了一篇关于如何[通过HTTPS运行你的Flask应用程序](https://blog.miguelgrinberg.com/post/running-your-flask-application-over-https)的详细文章。

## 部署应用更新

我想讨论的基于Linux的部署的最后一个主题是如何处理应用程序升级。 应用程序源代码通过`git`安装在服务器中，因此，无论何时想要将应用程序升级到最新版本，都可以运行`git pull`来下载自上次部署以来的新提交。

当然，下载新版本的代码不会导致升级。 当前正在运行的服务器进程将继续运行，旧代码已被读取并存储在内存中。 要触发升级，你必须停止当前的服务器并启动一个新的服务器，以强制重新读取所有代码。

进行升级通常比重新启动服务器更为复杂。 你可能需要应用数据库迁移或编译新的语言翻译，因此实际上，执行升级的过程涉及一系列命令：

```
(venv) $ git pull                              # download the new version
(venv) $ sudo supervisorctl stop microblog     # stop the current server
(venv) $ flask db upgrade                      # upgrade the database
(venv) $ flask translate compile               # upgrade the translations
(venv) $ sudo supervisorctl start microblog    # start a new server
```

## 树莓派托管

[树莓派](http://www.raspberrypi.org/)是一款革命性低成本的小型Linux计算机，功耗非常低，因此它是托管家庭在线服务器的理想设备，可以全天候在线而无需捆绑你的台式电脑或笔记本电脑。 有几个Linux发行版可以在树莓派上运行。 我的选择是[Raspbian](http://www.raspbian.org/)，这是树莓派基金会的官方发行版。

为了准备树莓派的环境，我要安装一个新的Raspbian版本。 我将使用2017年9月版的Raspbian Stretch Lite，但在阅读本文时，可能会有更新的版本，请查看官方[下载页面](https://www.raspberrypi.org/downloads/raspbian/)获得最新版本。

Raspbian镜像需要安装在SD卡上，然后插入树莓派，以便它启动时可以识别到。 在[树莓派站点](https://www.raspberrypi.org/documentation/installation/installing-images/)上可以查看到从Windows，Mac OS X和Linux将Raspbian镜像复制到SD卡的方法。

当你第一次启动树莓派时，请在连接到键盘和显示器时进行操作，以便你可以进行设置。 至少应该启用SSH，以便你可以从计算机登录并方便地执行部署任务。

和Ubuntu一样，Raspbian也是Debian的衍生产品，所以上面针对的Ubuntu Linux的说明，大部分都可以在树莓派上生效。 但是，如果你计划在家庭网络上运行小型应用程序而无需外部访问时，则可以跳过某些步骤。 例如，你可能不需要防火墙或无密码登录。 你可能想在这样一台小型的计算机上使用SQLite而不是MySQL。 你可以选择不使用nginx，并且让gunicorn服务器直接监听来自客户端的请求。 你可能只想要一个gunicorn worker进程。 Supervisor服务对于确保应用程序始终处于运行状态非常有用，因此我建议你仍然在树莓派上使用它。

