本文翻译自[The Flask Mega-Tutorial Part XIV: Ajax](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiv-ajax)

这是Flask Mega-Tutorial系列的第十四部分，我将使用Microsoft翻译服务和少许JavaScript来添加实时语言翻译功能。

在本章中，我将从服务器端开发的“安全区域”脱离，研究与服务器端同样重要的客户端组件的功能。 你是否看到过某些网站在用户生成的内容旁边显示的“翻译”链接？ 这些链接会触发非用户本地语言内容的实时自动翻译。 翻译的内容通常插入原始版本的下方。 Google将其显示为外语搜索结果。 Facebook在用户动态上使用它。 Twitter在推文上使用它。 今天我将向你展示如何将相同的功能添加到Microblog！

*本章的GitHub链接为：[Browse](https://github.com/miguelgrinberg/microblog/tree/v0.14), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.14.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.13...v0.14).*

## 服务器端与客户端

迄今为止，在我遵循的传统服务器端模型中，有一个客户端（由用户驱动的Web浏览器）向应用服务器发出HTTP请求。 请求可以简单地请求HTML页面，例如当你单击“个人主页”链接时，或者它可以触发一个操作，例如在编辑你的个人信息之后单击提交按钮。 在这两种类型的请求中，服务器通过直接发送新的网页或通过发送重定向来完成请求。 然后客户端用新的页面替换当前页面。 只要用户停留在应用的网站上，该周期就会重复。 在这种模式下，服务器完成所有工作，而客户端只显示网页并接受用户输入。

有一种不同的模式，客户端扮演更积极的角色。 在这个模式中，客户端向服务器发出一个请求，服务器响应一个网页，但与前面的情况不同，并不是所有的页面数据都是HTML，页面中也有部分代码，通常用Javascript编写。 一旦客户端收到该页面，它就会显示HTML部分，并执行代码。 从那时起，你就拥有了一个可以独立工作的活动客户端，而无需与服务器进行联系或只有很少联系。 在严格的客户端应用中，整个应用通过初始页面请求下载到客户端，然后应用完全在客户端上运行，只有在查询或者变更数据时才与服务器联系。 这种类型的应用称为[单页应用](http://en.wikipedia.org/wiki/Single-page_application)（SPAs）。

大多数应用是这两种模式的混合，并结合了两者的技术特点。 我的Microblog应用主要是服务器端应用，但今天我将添加一些客户端操作。 为了实时翻译用户动态，客户端浏览器将*异步请求*发送到服务器，服务器将响应该请求而不会导致页面刷新。然后客户端将动态地将翻译插入当前页面。 这种技术被称为[Ajax](http://en.wikipedia.org/wiki/Ajax_(programming))，这是Asynchronous JavaScript和XML的简称（尽管现在XML常常被JSON取代）。

## 实时翻译的工作流程

由于使用了Flask-Babel，本应用对外语有很好的支持，可以支持尽可能多的语言，只要我找到了对应的译文。 但是遗漏了一个元素，用户将会用他们自己的语言发表动态，所以用户很可能会用应用未知的语言发表动态。 自动翻译的质量大多数情况下不怎么样，但在，如果你只想对另一种语言的文本了解其基本含义，这已经足够了。

这正是Ajax大展身手的好机会！ 设想主页或发现页面可能会显示若干用户动态，其中一些可能是外语。 如果我使用传统的服务器端技术实现翻译，则翻译请求会导致原始页面被替换为新页面。 事实是，要求翻译诸多用户动态中的一条，并不是一个足够大的动作来要求整个页面的更新，如果翻译文本可以被动态地插入到原始文本下方，而剩下的页面保持原样，则用户体验更加出色。

实施实时自动翻译需要几个步骤。 首先，我需要一种方法来识别要翻译的文本的源语言。 我还需要知道每个用户的首选语言，因为我想仅为使用其他语言发表的动态显示“翻译”链接。 当提供翻译链接并且用户点击它时，我需要将Ajax请求发送到服务器，服务器将联系第三方翻译API。 一旦服务器发送了带有翻译文本的响应，客户端JavaScript代码将动态地将该文本插入到页面中。 你一定注意到了，这里有一些特殊的问题。 我将逐一审视这些问题。

## 语言识别

第一个问题是确定一条用户动态的语言。这不是一门精确的科学，因为不能确保监测结果绝对正确，但是对于大多数情况，自动检测的效果相当好。 在Python中，有一个称为`guess_language`的语言检测库，还算好用。 这个软件包的原始版本相当陈旧，从未被移植到Python 3，因此我将安装支持Python 2和3的派生版本：

```
(venv) $ pip install guess-language_spirit
```

计划是将每条用户动态提供给这个包，以尝试确定语言。 由于做这种分析有点费时，我不想每次把帖子呈现给页面时重复这项工作。 我要做的是在提交时为帖子设置源语言。 检测到的语言将被存储在post表中。

第一步，添加`language`字段到`Post`模型：

*app/models.py*：添加监测到的语言到`Post`模型：

```
class Post(db.Model):
    # ...
    language = db.Column(db.String(5))
```

你一定还记得，每当数据库模型发生变化时，都需要生成数据库迁移：

```
(venv) $ flask db migrate -m "add language to posts"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added column 'post.language'
  Generating migrations/versions/2b017edaa91f_add_language_to_posts.py ... done
```

然后将迁移应用到数据库：

```
(venv) $ flask db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Upgrade ae346256b650 -> 2b017edaa91f, add language to posts
```

我现在可以在提交帖子时检测并存储语言：

*app/routes.py*：为新的用户动态保存语言字段。

```
from guess_language import guess_language

@app.route('/', methods=['GET', 'POST'])
@app.route('/index', methods=['GET', 'POST'])
@login_required
def index():
    form = PostForm()
    if form.validate_on_submit():
        language = guess_language(form.post.data)
        if language == 'UNKNOWN' or len(language) > 5:
            language = ''
        post = Post(body=form.post.data, author=current_user,
                    language=language)
        # ...
```

有了这个变更，每次发表动态时，都会通过`guess_language`函数测试文本来尝试确定语言。 如果语言监测为未知，或者如果我得到意想不到的长字符串的结果，我会将一个空字符串保存到数据库中以安全地使用它。 我将采用约定，将任何将把语言设置为空字符串的帖子假定为未知语言。

## 展示一个“翻译”链接

第二步很简单。 我现在要做的是在任何不是当前用户的首选语言的用户动态下，添加一个“翻译”链接。

*app/templates/_post.html*：给用户动态添加翻译链接。

```
                {% if post.language and post.language != g.locale %}
                <br><br>
                <a href="#">{{ _('Translate') }}</a>
                {% endif %}
```

我在`_post.html`子模板中执行此操作，以便此功能出现在显示用户动态的任何页面上。 翻译链接只会出现在检测到语言种类的动态下，并且必须满足的条件是，这种语言与用Flask-Babel的`localeselector`装饰器装饰的函数选择的语言不匹配。 回想一下[第十三章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e4%b8%89%e7%ab%a0%ef%bc%9a%e5%9b%bd%e9%99%85%e5%8c%96%e5%92%8c%e6%9c%ac%e5%9c%b0%e5%8c%96.md)所选语言环境存储为`g.locale`。 链接文本需要以Flask-Babel可以翻译的方式添加，所以我在定义它时使用了`_()`函数。

请注意，我还没有关联此链接的操作。 首先，我想弄清楚如何进行实际的翻译。

## 使用第三方翻译服务

两种主要的翻译服务是[Google Cloud Translation API](https://developers.google.com/translate/)和[Microsoft Translator Text API](http://www.microsofttranslator.com/dev/)。 两者都是付费服务，但微软为低频少量的翻译提供了免费的入门级选项。 谷歌过去提供免费翻译服务，但现在，即使是最低层次的服务也需要付费。 因为我希望能够在不产生费用的情况下尝试翻译，我将实施Microsoft的解决方案。

在使用Microsoft Translator API之前，你需要先获得微软云服务[Azure](https://azure.com/)的帐户。 你可以选择免费套餐，但在注册过程中系统会要求你提供信用卡号，但在你保持该级别的服务时，你的卡不会被收取费用。

获得Azure帐户后，转到Azure门户并单击左上角的“New”按钮，然后键入或选择“Translator Text API”。 当你点击“Create”按钮时，将看到一个表单，并可以在其中定义一个新的翻译器资源，然后将其添加到你的帐户中。 你可以在下面看到我是如何完成表单的：

![Azure Translator](http://upload-images.jianshu.io/upload_images/4961528-4f9a52cdbf937a60..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当你再次点击“Create”按钮时，翻译器API资源将被添加到你的帐户中。几秒钟之后，你将在顶栏中收到通知，说明部署了翻译器资源。 点击通知中的“Go to resource”按钮，然后点击左侧栏上的“Keys”选项。 你现在将看到两个Key，分别标记为“Key 1”和“Key 2”。 将其中一个Key复制到剪贴板，然后将其设置到终端的环境变量中（如果使用的是Microsoft Windows，请用`set`替换`export`）：

```
(venv) $ export MS_TRANSLATOR_KEY=<paste-your-key-here>
```

该Key用于验证翻译服务，因此需要将其添加到应用配置中：

*config.py*: 添加Microsoft Translator API key到配置中。

```
class Config(object):
    # ...
    MS_TRANSLATOR_KEY = os.environ.get('MS_TRANSLATOR_KEY')
```

与很多配置值一样，我更喜欢将它们安装在环境变量中，并从那里将它们导入到Flask配置中。 对于允许访问第三方服务的密钥或密码等敏感信息，这一点尤为重要。 你绝对不想在代码中明确写出它们。

Microsoft Translator API是一个接受HTTP请求的Web服务。 Python中有若干HTTP客户端，但最常用和最简单的就是`requests`包。 所以让我们将其安装到虚拟环境中：

```
(venv) $ pip install requests
```

在下面，你可以看到我使用Microsoft Translator API编写翻译文本的功能。 我来新增一个*app/translate.py*模块：

*app/translate.py*：文本翻译函数。

```
import json
import requests
from flask_babel import _
from app import app

def translate(text, source_language, dest_language):
    if 'MS_TRANSLATOR_KEY' not in app.config or \
            not app.config['MS_TRANSLATOR_KEY']:
        return _('Error: the translation service is not configured.')
    auth = {'Ocp-Apim-Subscription-Key': app.config['MS_TRANSLATOR_KEY']}
    r = requests.get('https://api.microsofttranslator.com/v2/Ajax.svc'
                     '/Translate?text={}&from={}&to={}'.format(
                         text, source_language, dest_language),
                     headers=auth)
    if r.status_code != 200:
        return _('Error: the translation service failed.')
    return json.loads(r.content.decode('utf-8-sig'))
```

该函数定义需要翻译的文本、源语言和目标语言为参数，并返回翻译后文本的字符串。 它首先检查配置中是否存在翻译服务的Key，如果不存在，则会返回错误。 错误也是一个字符串，所以从外部看，这将看起来像翻译文本。 这可确保在出现错误时用户将看到有意义的错误消息。

`requests`包中的`get()`方法向作为第一个参数给定的URL发送一个带有GET方法的HTTP请求。 我使用*/v2/Ajax.svc/Translate* URL，它是翻译服务中的一个端点，它将翻译内容荷载为JSON返回。文本、源语言和目标语言都需要在URL中分别命名为`text`，`from`和`to`作为查询字符串参数。 要使用该服务进行身份验证，我需要将我添加到配置中的Key传递给该服务。 该Key需要在名为`Ocp-Apim-Subscription-Key`的自定义HTTP头中给出。 我创建了`auth`字典，然后将它通过`headers`参数传递给`requests`。

`requests.get()`方法返回一个响应对象，它包含了服务提供的所有细节。 我首先需要检查和确认状态码是200，这是成功请求的代码。 如果我得到任何其他代码，我就知道发生了错误，所以在这种情况下，我返回一个错误字符串。 如果状态码是200，那么响应的主体就有一个带有翻译的JSON编码字符串，所以我需要做的就是使用Python标准库中的`json.loads()`函数将JSON解码为我可以使用的Python字符串。 响应对象的`content`属性包含作为字节对象的响应的原始主体，该属性是UTF-8编码的字符序列，需要先进行解码，然后发送给`json.loads()`。

下面你可以看到一个Python控制台会话，我演示了如何使用新的`translate()`函数：

```
>>> from app.translate import translate
>>> translate('Hi, how are you today?', 'en', 'es')  # English to Spanish
'Hola, ¿cómo estás hoy?'
>>> translate('Hi, how are you today?', 'en', 'de')  # English to German
'Are Hallo, how you heute?'
>>> translate('Hi, how are you today?', 'en', 'it')  # English to Italian
'Ciao, come stai oggi?'
>>> translate('Hi, how are you today?', 'en', 'fr')  # English to French
"Salut, comment allez-vous aujourd'hui ?"
```

很酷，对吧？ 现在是时候将此功能与应用集成在一起了。

## 来自服务器的Ajax

我将从实现服务器端部分开始。 当用户单击动态下方显示的翻译链接时，将向服务器发出异步HTTP请求。 我将在下一节中向你展示如何执行此操作，因此现在我将专注于实现服务器处理此请求的操作。

异步（Ajax）请求类似于我在应用中创建的路由和视图函数，唯一的区别是它不返回HTML或重定向，而是返回数据，格式为[XML](http://en.wikipedia.org/wiki/XML)或更常见的[JSON](http://en.wikipedia.org/wiki/JSON)。 你可以在下面看到翻译视图函数，该函数调用Microsoft Translator API，然后返回JSON格式的翻译文本：

*app/routes.py*：文本翻译视图函数。

```
from flask import jsonify
from app.translate import translate

@app.route('/translate', methods=['POST'])
@login_required
def translate_text():
    return jsonify({'text': translate(request.form['text'],
                                      request.form['source_language'],
                                      request.form['dest_language'])})
```

如你所见，相当简单。 我以`POST`请求的形式实现了这条路由。 关于什么时候使用`GET`或`POST`（或者还没有见过的其他请求方法），真的没有绝对的规则。 由于客户端将发送数据，因此我决定使用`POST`请求，因为它与提交表单数据的请求类似。 `request.form`属性是Flask用提交中包含的所有数据暴露的字典。 当我使用Web表单工作时，我不需要查看`request.form`，因为Flask-WTF可以为我工作，但在这种情况下，实际上没有Web表单，所以我必须直接访问数据。

所以我在这个函数中做的是调用上一节中的`translate()`函数，直接从通过请求提交的数据中传递三个参数。 将结果合并到单个键`text`下的字典中，字典作为参数传递给Flask的`jsonify()`函数，该函数将字典转换为JSON格式的有效载荷。 `jsonify()`返回的值是将被发送回客户端的HTTP响应。

例如，如果客户希望将字符串“Hello，World！”翻译成西班牙语，则来自该请求的响应将具有以下有效载荷：

```
{ "text": "Hola, Mundo!" }
```

## 来自客户端的Ajax

因此，现在服务器能够通过*/translate* URL提供翻译，当用户单击我上面添加的“翻译”链接时，我需要调用此URL，传递需要翻译的文本、源语言和目标语言。 如果你不熟悉在浏览器中使用JavaScript，这将是一个很好的学习机会。

在浏览器中使用JavaScript时，当前显示的页面在内部被表示为文档对象模型（DOM）。 这是一个引用页面中所有元素的层次结构。 在此上下文中运行的JavaScript代码可以更改DOM以触发页面中的更改。

我们首先需要讨论的是，在浏览器中运行的JavaScript代码如何获取需要发送到服务器中运行的翻译函数的三个参数。 为了获得文本，我需要找到包含用户动态正文的DOM内的节点并获取它的内容。 为了便于识别包含用户动态的DOM节点，我将为它们附加一个唯一的ID。 如果你查看*_post.html*模板，则呈现用户动态正文的行只会读取`{{post.body}}`。 我要做的是将这些内容包装在一个`<span>`元素中。 这不会在视觉上改变任何东西，但它给了我一个可以插入标识符的地方：

*app/templates/_post.html*：给每条用户动态添加ID。

```
                <span id="post{{ post.id }}">{{ post.body }}</span>
```

这将为每条用户动态分配一个唯一标识符，格式为`post1`，`post2`等，其中数字与每条用户动态的数据库标识符相匹配。 现在每条用户动态都有一个唯一的标识符，给定一个ID值，我可以使用jQuery定位`<span>`元素并提取其中的文本。 例如，如果我想获得ID为123的用户动态的文本，我可以这样做：

```
$('#post123').text()
```

这里的$符号是jQuery库提供的函数的名称。 这个库被Bootstrap使用，所以它已经被Flask-Bootstrap包含。 `＃`是jQuery使用的“选择器”语法的一部分，这意味着接下来是元素的ID。

我也希望有一个地方可以在我从服务器收到翻译文本后插入翻译文本。 我要做的是将“翻译”链接替换为翻译文本，因此我还需要为该节点提供唯一标识符：

*app/templates/_post.html*：为翻译链接添加ID。

```
                <span id="translation{{ post.id }}">
                    <a href="#">{{ _('Translate') }}</a>
                </span>
```

因此，现在对于一个给定的用户动态ID，我有一个用于用户动态的`post <ID>`节点和一个对应的`translation <ID>`节点，我可以在用翻译后的文本替换翻译链接时用到它们。

下一步是编写一个可以完成所有翻译工作的函数。 该函数将利用输入和输出DOM节点以及源语言和目标语言，向服务器发出携带必须的三个参数的异步请求，并在服务器响应后用翻译后的文本替换翻译链接。 这听起来像很多工作，但实现相当简单：

*app/templates/base.html*：客户端翻译函数。

```
{% block scripts %}
    ...
    <script>
        function translate(sourceElem, destElem, sourceLang, destLang) {
            $(destElem).html('<img src="{{ url_for('static', filename='loading.gif') }}">');
            $.post('/translate', {
                text: $(sourceElem).text(),
                source_language: sourceLang,
                dest_language: destLang
            }).done(function(response) {
                $(destElem).text(response['text'])
            }).fail(function() {
                $(destElem).text("{{ _('Error: Could not contact server.') }}");
            });
        }
    </script>
{% endblock %}
```

前两个参数是用户动态和翻译链接节点的唯一ID，后两个参数是源语言和目标语言代码。

该函数从一个很好的接触开始：它添加一个*加载器*替换翻译链接，以便用户知道翻译正在进行中。 这是通过使用`$(destElem).html()`函数完成的，它用基于`<img>`元素的新HTML内容替换定义为翻译链接的原始HTML。 对于加载器，我将使用一个小的动画GIF，它已添加到Flask为静态文件保留的*app/static*目录中。 为了生成引用这个图像的URL，我使用`url_for()`函数，传递特殊的路由名称`static`并给出图像的文件名作为参数。 你可以在本章的[下载包](https://github.com/miguelgrinberg/microblog/tree/v0.14)中找到*loading.gif*图像。

现在我用一个优雅的加载器代替了翻译链接，以便用户知道要等待翻译出现。 下一步是将POST请求发送到我在前一节中定义的*/translate* URL。 为此，我也将使用jQuery，本处使用`$ .post()`函数。 这个函数以一种类似于浏览器提交Web表单的格式向服务器提交数据，这很方便，因为它允许Flask将这些数据合并到`request.form`字典中。 `$ .post()`的参数是两个，第一个是发送请求的URL，第二个是包含服务器期望的三个数据项的字典（或者称之为对象，因为这些是在JavaScript中调用的）。

你可能知道JavaScript对回调函数（或者称为*promises*的更高级的回调形式）友好。 现在要做的就是说明一旦这个请求完成并且浏览器接收到响应，我想完成的事情。 在JavaScript中没有需要等待的事情，一切都是*异步*。 我需要做的是提供一个回调函数，浏览器在接收到响应时调用它。 而且，为了使所有内容尽可能健壮，我想指出在出现错误的情况下该怎么做，以作为处理错误的第二个回调函数。 有几种方法可以指定这些回调，但在这种情况下，使用promises可以使代码更加清晰。 语法如下：

```
$.post(<url>, <data>).done(function(response) {
    // success callback
}).fail(function() {
    // error callback
})
```

promise语法允许将`$ .post()`调用的返回值“传入”回调函数作为参数。 在成功回调中，我所需要做的就是使用翻译后的文本调用`$(destElem).text()`，该文本在字典中`text`键下。 在出现错误的情况下，我也是这样做的，但是我显示的文本是一条通用的错误消息，我会确保它会作为可翻译的文本编入基础模板中。

所以现在唯一剩下的就是通过用户点击翻译链接来触发具有正确参数的`translate()`函数。 存在若干方法可以做到这一点，我要做的是将该函数的调用嵌入链接的`href`属性中：

*app/templates/_post.html*：翻译链接处理器。

```
                <span id="translation{{ post.id }}">
                    <a href="javascript:translate(
                                '#post{{ post.id }}',
                                '#translation{{ post.id }}',
                                '{{ post.language }}',
                                '{{ g.locale }}');">{{ _('Translate') }}</a>
                </span>
```

链接的`href`元素可以接受任何JavaScript代码，如果它带有`javascript:`前缀的话，那么这是一种方便的方式来调用翻译函数。 因为这个链接将在客户端请求页面时在服务器端渲染，所以我可以使用`{{}}`表达式来为函数生成四个参数。 每条用户动态都有自己的翻译链接，以及其唯一生成的参数。 `post <ID>`和`translation <ID>`需要渲染具体的ID，它们都需要在被使用时加上`#`前缀。

现在实时翻译功能已经完成！ 如果你在环境中设置了有效的Microsoft Translator API Key，则现在应该能够触发翻译。 假设你的浏览器设置为偏好英语，则需要使用其他语言撰写文章以查看“翻译”链接。 下面你可以看到一个例子：

![Translation](http://upload-images.jianshu.io/upload_images/4961528-1b85536f66d9c7f3..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在本章中，我介绍了一些需要翻译成应用支持的所有语言的新文本，因此有必要更新翻译目录：

```
(venv) $ flask translate update
```

对于你自己的项目，需要编辑每个语言存储库中的*messages.po*文件以包含这些新测试的翻译，不过我已经在本章的下载包或GitHub存储库中创建了西班牙语翻译。

要完成新的翻译，还需要执行编译：

```
(venv) $ flask translate compile
```

