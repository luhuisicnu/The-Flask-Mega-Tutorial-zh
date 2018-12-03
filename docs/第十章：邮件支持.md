本文翻译自[The Flask Mega-Tutorial Part X: Email Support](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-x-email-support)

这是Flask Mega-Tutorial系列的第十部分，在其中我将告诉你，应用如何向你的用户发送电子邮件，以及如何在电子邮件支持之上构建密码重置功能。

现在，应用在数据库方面做得相当不错，所以在本章中，我想抛开这个主题，开始添加发送电子邮件的功能，这是大多数Web应用必需的另一个重要部分。

为什么应用需要发送电子邮件给用户？ 原因很多，但其中一个常见的原因是解决与认证相关的问题。 在本章中，我将为忘记密码的用户添加密码重置功能。 当用户请求重置密码时，应用将发送包含特制链接的电子邮件。 用户然后需要点击该链接才能访问设置新密码的表单。

*本章的GitHub链接为：[Browse](https://github.com/miguelgrinberg/microblog/tree/v0.10), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.10.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.9...v0.10).*

## Flask-Mail简介

就实际的邮件发送而言，Flask有一个名为[Flask-Mail](https://pythonhosted.org/Flask-Mail/)的流行插件，可以使任务变得非常简单。 和往常一样，该插件是用pip安装的：
```
(venv) $ pip install flask-mail
```

密码重置链接将包含有一个安全令牌。 为了生成这些令牌，我将使用[JSON Web Tokens](https://jwt.io/)，它也有一个流行的Python包：
```
(venv) $ pip install pyjwt
```

Flask-Mail插件是通过`app.config`对象来配置的。还记得在[第七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E4%B8%83%E7%AB%A0%EF%BC%9A%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86.md)中，我添加了用于在生产环境中发生错误时发送电子邮件的配置项? 当时我没有告诉你，不过，我选择的配置变量都是Flask-Mail的需求的，所以不需要任何额外的工作，配置的活已经完工。

像大多数Flask插件一样，你需要在Flask应用创建之后创建一个邮件实例。 本处，`mail`是类`Mail`的一个实例：
```
# ...
from flask_mail import Mail

app = Flask(__name__)
# ...
mail = Mail(app)
```

[第七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E4%B8%83%E7%AB%A0%EF%BC%9A%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86.md)中我提到过，测试发送电子邮件的方式有两种。 如果你想使用一个模拟的电子邮件服务器，Python提供了一个非常好用的方法，你可以使用下面的命令在第二个终端中启动它：
```
(venv) $ python -m smtpd -n -c DebuggingServer localhost:8025
```

要配置此服务器，需要设置两个环境变量：
```
(venv) $ export MAIL_SERVER=localhost
(venv) $ export MAIL_PORT=8025
```

如果你希望真实地发送电子邮件，则需要使用真实的电子邮件服务器。 那么你只需要为它设置`MAIL_SERVER`、`MAIL_PORT`、`MAIL_USE_TLS`、`MAIL_USERNAME`和`MAIL_PASSWORD`环境变量。 如果你想要快速解决方案，可以使用Gmail帐户发送电子邮件，并使用以下设置：
```
(venv) $ export MAIL_SERVER=smtp.googlemail.com
(venv) $ export MAIL_PORT=587
(venv) $ export MAIL_USE_TLS=1
(venv) $ export MAIL_USERNAME=<your-gmail-username>
(venv) $ export MAIL_PASSWORD=<your-gmail-password>
```

如果你使用的是Microsoft Windows，则需要在上面的每个`export`语句中将`export`替换为`set`。

Gmail帐户中的安全功能可能会阻止应用通过它发送电子邮件，除非你明确允许“安全性较低的应用程序”访问你的Gmail帐户。 可以阅读[此处](https://support.google.com/accounts/answer/6010255?hl=en)来了解具体情况，如果你担心帐户的安全性，可以创建一个辅助邮箱帐户，配置它来仅用于测试电子邮件功能，或者你可以暂时启用允许不太安全的应用程序来运行此测试，完成后恢复为默认值。

## Flask-Mail的使用

为了学习Flask-Mail如何工作，我将向你展示如何用Python shell发送电子邮件。那么，运行`flask shell`以激活Python，然后运行下面的命令：
```
>>> from flask_mail import Message
>>> from app import mail
>>> msg = Message('test subject', sender=app.config['ADMINS'][0],
... recipients=['your-email@example.com'])
>>> msg.body = 'text body'
>>> msg.html = '<h1>HTML body</h1>'
>>> mail.send(msg)
```

上面的代码片段将发送一个电子邮件到你在`recipients`参数中设置的电子邮件地址列表。发件人配置项我在[第七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E4%B8%83%E7%AB%A0%EF%BC%9A%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86.md)中已经配置过了，是`ADMINS`。 该电子邮件将具有纯文本和HTML版本，所以根据你的电子邮件客户端的配置，可能会看到它们之中的其中之一。

如你所见，相当简单。现在让我们将电子邮件整合到应用中。

## 简单的电子邮件框架

我将从编写一个发送电子邮件的帮助函数开始，这个函数基本上是上一节中shell函数的通用版本。 我将把这个函数放在一个名为`app/email.py`的新模块中：
```
from flask_mail import Message
from app import mail

def send_email(subject, sender, recipients, text_body, html_body):
    msg = Message(subject, sender=sender, recipients=recipients)
    msg.body = text_body
    msg.html = html_body
    mail.send(msg)
```

Flask-Mail支持一些我不在这里使用的功能，如抄送和密件抄送列表。 如果你对这些选项感兴趣，务必查阅[Flask-Mail文档](https://pythonhosted.org/Flask-Mail/)。

## 请求重置密码

我上面提到过，用户有权利重置密码。因此我将在登录页面提供一个链接：
```
    <p>
        Forgot Your Password?
        <a href="{{ url_for('reset_password_request') }}">Click to Reset It</a>
    </p>
```

当用户点击链接时，会出现一个新的Web表单，要求用户输入注册的电子邮件地址，以启动密码重置过程。 这里是表单类：
```
class ResetPasswordRequestForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Email()])
    submit = SubmitField('Request Password Reset')
```

这里是相应的HTML模板：
```
{% extends "base.html" %}

{% block content %}
    <h1>Reset Password</h1>
    <form action="" method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.email.label }}<br>
            {{ form.email(size=64) }}<br>
            {% for error in form.email.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

当然也需要一个视图函数来处理表单：
```
from app.forms import ResetPasswordRequestForm
from app.email import send_password_reset_email

@app.route('/reset_password_request', methods=['GET', 'POST'])
def reset_password_request():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    form = ResetPasswordRequestForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if user:
            send_password_reset_email(user)
        flash('Check your email for the instructions to reset your password')
        return redirect(url_for('login'))
    return render_template('reset_password_request.html',
                           title='Reset Password', form=form)
```

该视图函数与其他的表单处理视图函数非常相似。 我从确保用户没有登录开始，如果用户登录，那么使用密码重置功能就没有意义，所以我重定向到主页。

当表格被提交并验证通过，我使用表格中的用户提供的电子邮件来查找用户。 如果我找到用户，就发送一封密码重置电子邮件。 我执行此操作使用的`send_password_reset_email()`辅助函数，将在下面向你展示。

电子邮件发送后，我会闪现一条消息，指示用户查看电子邮件以获取进一步说明，然后重定向回登录页面。 你可能会注意到，即使用户提供的电子邮件不存在，也会显示闪现的消息，这样的话，客户端就不能用这个表单来判断一个给定的用户是否已注册。

## 密码重置令牌

在实现`send_password_reset_email()`函数之前，我需要一种方法来生成密码重置链接，它将被通过电子邮件发送给用户。 当链接被点击时，将为用户展现设置新密码的页面。 这个计划中棘手的部分是确保只有有效的重置链接可以用来重置帐户的密码。

生成的链接中会包含*令牌*，它将在允许密码变更之前被验证，以证明请求重置密码的用户是通过访问重置密码邮件中的链接而来的。JSON Web Token（JWT）是这类令牌处理的流行标准。 JWTs的优点是它是自成一体的，不但可以生成令牌，还提供对应的验证方法。

如何运行JWTs？让我们通过Python shell来学习一下：
```
>>> import jwt
>>> token = jwt.encode({'a': 'b'}, 'my-secret', algorithm='HS256')
>>> token
b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhIjoiYiJ9.dvOo58OBDHiuSHD4uW88nfJikhYAXc_sfUHq1mDi4G0'
>>> jwt.decode(token, 'my-secret', algorithms=['HS256'])
{'a': 'b'}
```

`{'a'：'b'}`字典是要写入令牌的示例有效载荷。 为了使令牌安全，需要提供一个秘密密钥用于创建加密签名。 在这个例子中，我使用了字符串`'my-secret'`，但是在应用中，我将使用配置中的`SECRET_KEY`。`algorithm`参数指定使用什么算法来生成令牌，而`HS256`是应用最广泛的算法。

如你所见，得到的令牌是一长串字符。 但是不要认为这是一个加密的令牌。 令牌的内容，包括有效载荷，可以被任何人轻易解码（不相信我？复制上面的令牌，然后粘贴在[JWT调试器](https://jwt.io/#debugger-io)上就可以看到它的内容）。 使令牌安全的是，有效载荷是*被签名的*。 如果有人试图伪造或篡改令牌中的有效载荷，则签名将会无效，并且生成新的签名依赖秘密密钥。 令牌验证通过时，有效负载的内容将被解码并返回给调用者。 如果令牌的签名验证通过，有效载荷才可以被认为是可信的。

我要用于密码重置令牌的有效载荷格式为`{'reset_password'：user_id，'exp'：token_expiration}`。 `exp`字段是JWTs的标准，如果它存在，则表示令牌的到期时间。 如果一个令牌有一个有效的签名，但是它已经过期，那么它也将被认为是无效的。 对于密码重置功能，我会给这些令牌10分钟的有效期。

当用户点击电子邮件链接时，令牌将被作为URL的一部分发送回应用，处理这个URL的视图函数首先要做的就是验证它。 如果签名是有效的，则可以通过存储在有效载荷中的ID来识别用户。 一旦得知用户的身份，应用可以要求一个新的密码，并将其设置在用户的帐户上。

由于这些令牌属于用户，因此我将在`User`模型中编写令牌生成和验证的方法：
```
from time import time
import jwt
from app import app

class User(UserMixin, db.Model):
    # ...

    def get_reset_password_token(self, expires_in=600):
        return jwt.encode(
            {'reset_password': self.id, 'exp': time() + expires_in},
            app.config['SECRET_KEY'], algorithm='HS256').decode('utf-8')

    @staticmethod
    def verify_reset_password_token(token):
        try:
            id = jwt.decode(token, app.config['SECRET_KEY'],
                            algorithms=['HS256'])['reset_password']
        except:
            return
        return User.query.get(id)
```

`get_reset_password_token()`函数以字符串形式生成一个JWT令牌。 请注意，`decode('utf-8')`是必须的，因为`jwt.encode()`函数将令牌作为字节序列返回，但是在应用中将令牌表示为字符串更方便。

`verify_reset_password_token()`是一个静态方法，这意味着它可以直接从类中调用。 静态方法与类方法类似，唯一的区别是静态方法不会接收类作为第一个参数。 这个方法需要一个令牌，并尝试通过调用PyJWT的`jwt.decode()`函数来解码它。 如果令牌不能被验证或已过期，将会引发异常，在这种情况下，我会捕获它以防止出现错误，然后将`None`返回给调用者。 如果令牌有效，那么来自令牌有效负载的`reset_password`的值就是用户的ID，所以我可以加载用户并返回它。

## 发送密码重置电子邮件

现在我有了令牌，可以生成密码重置电子邮件。 `send_password_reset_email()`函数依赖于上面写的`send_email()`函数。
```
from flask import render_template
from app import app

# ...

def send_password_reset_email(user):
    token = user.get_reset_password_token()
    send_email('[Microblog] Reset Your Password',
               sender=app.config['ADMINS'][0],
               recipients=[user.email],
               text_body=render_template('email/reset_password.txt',
                                         user=user, token=token),
               html_body=render_template('email/reset_password.html',
                                         user=user, token=token))
```

这个函数中有趣的部分是电子邮件的文本和HTML内容是使用熟悉的`render_template()`函数从模板生成的。 模板接收用户和令牌作为参数，以便可以生成个性化的电子邮件消息。 以下是重置密码电子邮件的文本模板：
```
Dear {{ user.username }},

To reset your password click on the following link:

{{ url_for('reset_password', token=token, _external=True) }}

If you have not requested a password reset simply ignore this message.

Sincerely,

The Microblog Team
```

这是更美观的的HTML版本：
```
<p>Dear {{ user.username }},</p>
<p>
    To reset your password
    <a href="{{ url_for('reset_password', token=token, _external=True) }}">
        click here
    </a>.
</p>
<p>Alternatively, you can paste the following link in your browser's address bar:</p>
<p>{{ url_for('reset_password', token=token, _external=True) }}</p>
<p>If you have not requested a password reset simply ignore this message.</p>
<p>Sincerely,</p>
<p>The Microblog Team</p>
```

请注意，这两个电子邮件模板中的`url_for()`调用中引用的`reset_password`路由尚不存在，这将在下一节中添加。在这两个模板中，`url_for()`函数中的`_external=True`参数是一个新玩意儿。不带这个参数的情况下，`url_for()`函数生成的是相对路径。例如`url_for('user', username='susan')`生成`/user/susan`。这样的路径在本站的Web页面中使用是完全足够的，因为其余的协议、主机、端口部分，会沿用本站的当前值。一旦通过邮件发送时，就脱离了这个上下文，这时候就需要URL的完全路径了。一旦传入`_external=True`参数给`url_for()`函数，就会生成一个URL的完全路径。本处示例为`http://localhost:5000/user/susan`。如果应用被部署到一个域名下，则协议、主机名和端口会发生对应的变化。

## 重置用户密码

当用户点击电子邮件链接时，会触发与此功能相关的第二个路由。 这是密码重置视图函数：
```
from app.forms import ResetPasswordForm

@app.route('/reset_password/<token>', methods=['GET', 'POST'])
def reset_password(token):
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    user = User.verify_reset_password_token(token)
    if not user:
        return redirect(url_for('index'))
    form = ResetPasswordForm()
    if form.validate_on_submit():
        user.set_password(form.password.data)
        db.session.commit()
        flash('Your password has been reset.')
        return redirect(url_for('login'))
    return render_template('reset_password.html', form=form)
```

在这个视图函数中，我首先确保用户没有登录，然后通过调用`User`类的令牌验证方法来确定用户是谁。 如果令牌有效，则此方法返回用户；如果不是，则返回`None`，并将重定向到主页。

如果令牌是有效的，那么我向用户呈现第二个表单，需要用户其中输入新密码。 这个表单的处理方式与以前的表单类似，表单提交验证通过后，我调用`User`类的`set_password()`方法来更改密码，然后重定向到登录页面，以便用户登录。

这是`ResetPasswordForm`类：
```
class ResetPasswordForm(FlaskForm):
    password = PasswordField('Password', validators=[DataRequired()])
    password2 = PasswordField(
        'Repeat Password', validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Request Password Reset')
```

这是相应的HTML模板：
```
{% extends "base.html" %}

{% block content %}
    <h1>Reset Your Password</h1>
    <form action="" method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.password.label }}<br>
            {{ form.password(size=32) }}<br>
            {% for error in form.password.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>
            {{ form.password2.label }}<br>
            {{ form.password2(size=32) }}<br>
            {% for error in form.password2.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

密码重置功能现已完成，一定要多尝试几次。

## 异步电子邮件

如果你正在使用Python提供的模拟电子邮件服务器，可能没有注意到这一点，那就是发送电子邮件会大大减慢应用的速度，原因是发送电子邮件时所发生的和电子邮件服务器的网络交互。通常需要几秒钟的时间才能收到电子邮件，如果收件人的电子邮件服务器速度较慢，或者收件人有多个，则可能会更久。

我真正想要的`send_email()`函数是*异步的*。 那是什么意思？ 这意味着当这个函数被调用时，发送邮件的任务被安排在后台进行，释放`send_email()`函数以立即返回，以便应用可以在发送邮件的同时继续运行。

Python实际上有多种方式支持运行异步任务，`threading`和`multiprocessing`模块都可以做到这一点。 为发送电子邮件启动一个后台线程，比开始一个全新的进程需要的资源少得多，所以我打算采用这种方法：
```
from threading import Thread
# ...

def send_async_email(app, msg):
    with app.app_context():
        mail.send(msg)

def send_email(subject, sender, recipients, text_body, html_body):
    msg = Message(subject, sender=sender, recipients=recipients)
    msg.body = text_body
    msg.html = html_body
    Thread(target=send_async_email, args=(app, msg)).start()
```

`send_async_email`函数现在运行在后台线程中，它通过`send_email()`的最后一行中的`Thread()`类来调用。 有了这个改变，电子邮件的发送将在线程中运行，并且当进程完成时，线程将结束并自行清理。 如果你已经配置了一个真正的电子邮件服务器，当你按下密码重置请求表单上的提交按钮时，肯定会注意到访问速度的提升。

你可能预期只有`msg`参数会被发送到线程，但正如你在代码中所看到的那样，我也传入了应用实例。 使用线程时，需要牢记Flask的一个重要设计方面。 Flask使用*上下文*来避免必须跨函数传递参数。 我不打算详细讨论这个问题，但是需要知道的是，有两种类型的上下文，即*应用上下文*和*请求上下文*。 在大多数情况下，这些上下文由框架自动管理，但是当应用启动自定义线程时，可能需要手动创建这些线程的上下文。

许多Flask插件需要应用上下文才能工作，因为这使得他们可以在不传递参数的情况下找到Flask应用实例。这些插件需要知道应用实例的原因是因为它们的配置存储在`app.config`对象中，这正是Flask-Mail的情况。`mail.send()`方法需要访问电子邮件服务器的配置值，而这必须通过访问应用属性的方式来实现。 使用`with app.app_context()`调用创建的应用上下文使得应用实例可以通过来自Flask的`current_app`变量来进行访问。

