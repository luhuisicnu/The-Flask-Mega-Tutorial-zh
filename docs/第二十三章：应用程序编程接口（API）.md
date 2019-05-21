本文翻译自[The Flask Mega-Tutorial Part XXIII: Application Programming Interfaces (APIs)](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xxiii-application-programming-interfaces-apis)

我为此应用程序构建的所有功能都只适用于特定类型的客户端：Web浏览器。 但其他类型的客户端呢？ 例如，如果我想构建Android或iOS APP，有两种主流方法可以解决这个问题。 最简单的解决方案是构建一个简单的APP，仅使用一个Web视图组件并用Microblog网站填充整个屏幕，但相比在设备的Web浏览器中打开网站，这种方案几乎没有什么卖点。 一个更好的解决方案（尽管更费力）将是构建一个本地APP，但这个APP如何与仅返回HTML页面的服务器交互呢？

这就是应用程序编程接口（API）的能力范畴了。 API是一组HTTP路由，被设计为应用程序中的低级入口点。与定义返回HTML以供Web浏览器使用的路由和视图函数不同，API允许客户端直接使用应用程序的*资源*，从而决定如何通过客户端完全地向用户呈现信息。 例如，Microblog中的API可以向用户提供用户信息和用户动态，并且它还可以允许用户编辑现有动态，但仅限于数据级别，不会将此逻辑与HTML混合。

如果你研究了应用程序中当前定义的所有路由，会注意到其中的几个符合我上面使用的API的定义。 找到它们了吗？ 我说的是返回JSON的几条路由，比如[第十四章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e5%9b%9b%e7%ab%a0%ef%bc%9aAjax.md)中定义的 */translate* 路由。 这种路由的内容都以JSON格式编码，并在请求时使用`POST`方法。 此请求的响应也是JSON格式，服务器仅返回所请求的信息，客户端负责将此信息呈现给用户。

虽然应用程序中的JSON路由具有API的“感觉”，但它们的设计初衷是为支持在浏览器中运行的Web应用程序。 设想一下，如果智能手机APP想要使用这些路由，它将无法使用，因为这需要用户登录，而登录只能通过HTML表单进行。 在本章中，我将展示如何构建不依赖于Web浏览器的API，并且不会假设连接到它们的客户端的类型。

*本章的GitHub链接为：[Browse](https://github.com/miguelgrinberg/microblog/tree/v0.23), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.23.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.22...v0.23).*

## REST API设计风格
## REST as a Foundation of API Design

有些人可能会强烈反对上面提到的 */translate* 和其他JSON路由是API路由。 其他人可能会同意，但也会认为它们是一个设计糟糕的API。 那么一个精心设计的API有什么特点，为什么上面的JSON路由不是一个好的API路由呢？

你可能听说过[REST API](https://en.wikipedia.org/wiki/Representational_state_transfer)。 REST（Representational State Transfer）是Roy Fielding在[博士论文](http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)中提出的一种架构。 该架构中，Dr. Fielding以相当抽象和通用的方式展示了REST的六个定义特征。

除了Dr.Fielding的论文外，没有关于REST的权威性规范，从而留下了许多细节供读者解读。 一个给定的API是否符合REST规范的话题往往是REST“纯粹主义者”之间激烈争论的源头，REST“纯粹主义者”认为REST API必须以非常明确的方式遵循全部六个特征，而不像REST“实用主义者”那样，仅仅将Dr. Fielding在论文中提出的想法作为指导原则或建议。Dr.Fielding站在纯粹主义阵营的一边，并在博客文章和在线评论中的撰写了一些额外的见解来表达他的愿景。

目前实施的绝大多数API都遵循“实用主义”的REST实现。 包括来自Facebook，GitHub，Twitter等“大玩家”的大部分API都是如此。很少有公共API被一致认为是纯REST，因为大多数API都没有包含纯粹主义者认为必须实现的某些细节。 尽管Dr. Fielding和其他REST纯粹主义者对评判一个API是否是REST API有严格的规定，但软件行业在实际运用中引用REST是很常见的。

为了让你了解REST论文中的内容，以下各节将介绍Dr. Fielding列举的六项原则。

### 客户端－服务器

客户端－服务器原则相当简单，正如其字面含义，在REST API中，客户端和服务器的角色应该明确区分。 在实践中，这意味着客户端和服务器都是单独的进程，并在大多数情况下，使用基于TCP网络上的HTTP协议进行通信。

### 分层系统

分层系统原则是说当客户端需要与服务器通信时，它可能最终连接到代理服务器而不是实际的服务器。 因此，对于客户端来说，如果不直接连接到服务器，它发送请求的方式应该没有什么区别，事实上，它甚至可能不知道它是否连接到目标服务器。 同样，这个原则规定服务器兼容直接接收来自代理服务器的请求，所以它绝不能假设连接的另一端一定是客户端。

这是REST的一个重要特性，因为能够添加中间节点的这个特性，允许应用程序架构师使用负载均衡器，缓存，代理服务器等来设计满足大量请求的大型复杂网络。

### 缓存

该原则扩展了分层系统，通过明确指出允许服务器或代理服务器缓存频繁且相同请求的响应内容以提高系统性能。 有一个你可能熟悉的缓存实现：所有Web浏览器中的缓存。 Web浏览器缓存层通常用于避免一遍又一遍地请求相同的文件，例如图像。

为了达到API的目的，目标服务器需要通过使用*缓存控制*来指示响应是否可以在代理服务器传回客户端时进行缓存。 请注意，由于安全原因，部署到生产环境的API必须使用加密，因此，除非此代理服务器*terminates* SSL连接，或者执行解密和重新加密，否则缓存通常不会在代理服务器中完成。

### 按需获取客户端代码（Code On Demand）

这是一项可选要求，规定服务器可以提供可执行代码以响应客户端，这样一来，就可以从服务器上获取客户端的新功能。 因为这个原则需要服务器和客户端之间就客户端能够运行的可执行代码类型达成一致，所以这在API中很少使用。 你可能会认为服务器可能会返回JavaScript代码以供Web浏览器客户端执行，但REST并非专门针对Web浏览器客户端而设计。 例如，如果客户端是iOS或Android设备，执行JavaScript可能会带来一些复杂情况。

### 无状态

无状态原则是REST纯粹主义者和实用主义者之间争论最多的两个中心之一。 它指出，REST API不应保存客户端发送请求时的任何状态。 这意味着，在Web开发中常见的机制都不能在用户浏览应用程序页面时“记住”用户。 在无状态API中，每个请求都需要包含服务器需要识别和验证客户端并执行请求的信息。这也意味着服务器无法在数据库或其他存储形式中存储与客户端连接有关的任何数据。

如果你想知道为什么REST需要无状态服务器，主要原因是无状态服务器非常容易扩展，你只需在负载均衡器后面运行多个服务器实例即可。 如果服务器存储客户端状态，则事情会变得更复杂，因为你必须弄清楚多个服务器如何访问和更新该状态，或者确保给定客户端始终由同一服务器处理，这样的机制通常称为*粘性会话*。

再思考一下本章介绍中讨论的 */translate* 路由，就会发现它不能被视为*RESTful*，因为与该路由相关的视图函数依赖于Flask-Login的`@login_required`装饰器， 这会将用户的登录状态存储在Flask用户会话中。

### 统一接口

最后，最重要的，最有争议的，最含糊不清的REST原则是统一接口。 Dr. Fielding列举了REST统一接口的四个特性：唯一资源标识符，资源表示，自描述性消息和超媒体。

唯一资源标识符是通过为每个资源分配唯一的URL来实现的。 例如，与给定用户关联的URL可以是 */api/users/\<user-id\>* ，其中 *\<user-id\>* 是在数据库表主键中分配给用户的标识符。 大多数API都能很好地实现这一点。

资源表示的使用意味着当服务器和客户端交换关于资源的信息时，他们必须使用商定的格式。 对于大多数现代API，JSON格式用于构建资源表示。 API可以选择支持多种资源表示格式，并且在这种情况下，HTTP协议中的*内容协商*选项是客户端和服务器确认格式的机制。

自描述性消息意味着在客户端和服务器之间交换的请求和响应必须包含对方需要的所有信息。 作为一个典型的例子，HTTP请求方法用于指示客户端希望服务器执行的操作。 `GET`请求表示客户想要检索资源信息，`POST`请求表示客户想要创建新资源，`PUT`或`PATCH`请求定义对现有资源的修改，`DELETE` 表示删除资源的请求。 目标资源被指定为请求的URL，并在HTTP头，URL的查询字符串部分或请求主体中提供附加信息。

超媒体需求是最具争议性的，而且很少有API实现，而那些实现它的API很少以满足REST纯粹主义者的方式进行。由于应用程序中的资源都是相互关联的，因此此要求会要求将这些关系包含在资源表示中，以便客户端可以通过遍历关系来发现新资源，这几乎与你在Web应用程序中通过点击从一个页面到另一个页面的链接来发现新页面的方式相同。理想情况下，客户端可以输入一个API，而不需要任何有关其中的资源的信息，就可以简单地通过超媒体链接来了解它们。但是，与HTML和XML不同，通常用于API中资源表示的JSON格式没有定义包含链接的标准方式，因此你不得不使用自定义结构，或者类似[JSON-API](http://jsonapi.org/)，[HAL](http://stateless.co/hal_specification.html)，[ JSON-LD](https://json-ld.org/)这样的试图解决这种差距的JSON扩展之一。

## 实现API Blueprint

为了让你体验开发API所涉及的内容，我将在Microblog添加API。 我不会实现所有的API，只会实现与用户相关的所有功能，并将其他资源（如用户动态）的实现留给读者作为练习。

为了保持组织有序，并遵循我在[第十五章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e4%ba%94%e7%ab%a0%ef%bc%9a%e4%bc%98%e5%8c%96%e5%ba%94%e7%94%a8%e7%bb%93%e6%9e%84.md)中描述的结构， 我将创建一个包含所有API路由的新blueprint。 所以，让我们从创建blueprint所在的目录开始：

```
(venv) $ mkdir app/api
```

在blueprint的`__init__.py`文件中创建blueprint对象，这与应用程序中的其他blueprint类似：

`app/api/__init__.py`: API blueprint 构造器。

```
from flask import Blueprint

bp = Blueprint('api', __name__)

from app.api import users, errors, tokens
```

你可能会记得有时需要将导入移动到底部以避免循环依赖错误。 这就是为什么*app/api/users.py*，*app/api/errors.py*和*app/api/tokens.py*模块（我还没有写）在blueprint创建之后导入的原因。

API的主要内容将存储在*app/api/users.py*模块中。 下表总结了我要实现的路由：

| HTTP 方法 | 资源 URL | 注释 |
| --- | --- | --- |
| `GET` | */api/users/\<id\>* | 返回一个用户|
| `GET` | */api/users* | 返回所有用户的集合 |
| `GET` | */api/users/\<id\>/followers* | 返回某个用户的粉丝集合 |
| `GET` | */api/users/\<id\>/followed* | 返回某个用户关注的用户集合 |
| `POST` | */api/users* | 注册一个新用户 |
| `PUT` | */api/users/\<id\>* | 修改某个用户|

现在我要创建一个模块的框架，其中使用占位符来暂时填充所有的路由：

*app/api/users.py*：用户API资源占位符。

```
from app.api import bp

@bp.route('/users/<int:id>', methods=['GET'])
def get_user(id):
    pass

@bp.route('/users', methods=['GET'])
def get_users():
    pass

@bp.route('/users/<int:id>/followers', methods=['GET'])
def get_followers(id):
    pass

@bp.route('/users/<int:id>/followed', methods=['GET'])
def get_followed(id):
    pass

@bp.route('/users', methods=['POST'])
def create_user():
    pass

@bp.route('/users/<int:id>', methods=['PUT'])
def update_user(id):
    pass
```

*app/api/errors.py*模块将定义一些处理错误响应的辅助函数。 但现在，我使用占位符，并将在之后填充内容：

*app/api/errors.py*：错误处理占位符。

```
def bad_request():
    pass
```

*app/api/tokens.py*是将要定义认证子系统的模块。 它将为非Web浏览器登录的客户端提供另一种方式。现在，我也使用占位符来处理该模块：

*app/api/tokens.py*: Token处理占位符。

```
def get_token():
    pass

def revoke_token():
    pass
```

新的API blueprint需要在应用工厂函数中注册：

`app/__init__.py`：应用中注册API blueprint。

```
# ...

def create_app(config_class=Config):
    app = Flask(__name__)

    # ...

    from app.api import bp as api_bp
    app.register_blueprint(api_bp, url_prefix='/api')

    # ...
```

## 将用户表示为JSON对象

实施API时要考虑的第一个方面是决定其资源表示形式。 我要实现一个用户类型的API，因此我需要决定的是用户资源的表示形式。 经过一番头脑风暴，得出了以下JSON表示形式：

```
{
    "id": 123,
    "username": "susan",
    "password": "my-password",
    "email": "susan@example.com",
    "last_seen": "2017-10-20T15:04:27Z",
    "about_me": "Hello, my name is Susan!",
    "post_count": 7,
    "follower_count": 35,
    "followed_count": 21,
    "_links": {
        "self": "/api/users/123",
        "followers": "/api/users/123/followers",
        "followed": "/api/users/123/followed",
        "avatar": "https://www.gravatar.com/avatar/..."
    }
}
```

许多字段直接来自用户数据库模型。 `password`字段的特殊之处在于，它仅在注册新用户时才会使用。 回顾[第五章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e4%ba%94%e7%ab%a0%ef%bc%9a%e7%94%a8%e6%88%b7%e7%99%bb%e5%bd%95.md)，用户密码不存储在数据库中，只存储一个散列字符串，所以密码永远不会被返回。`email`字段也被专门处理，因为我不想公开用户的电子邮件地址。 只有当用户请求自己的条目时，才会返回`email`字段，但是当他们检索其他用户的条目时不会返回。`post_count`，`follower_count`和`followed_count`字段是“虚拟”字段，它们在数据库字段中不存在，提供给客户端是为了方便。 这是一个很好的例子，它演示了资源表示不需要和服务器中资源的实际定义一致。

请注意`_links`部分，它实现了超媒体要求。 定义的链接包括指向当前资源的链接，用户的粉丝列表链接，用户关注的用户列表链接，最后是指向用户头像图像的链接。 将来，如果我决定向这个API添加用户动态，那么用户的动态列表链接也应包含在这里。

JSON格式的一个好处是，它总是转换为Python字典或列表的表示形式。 Python标准库中的`json`包负责Python数据结构和JSON之间的转换。因此，为了生成这些表示，我将在`User`模型中添加一个名为`to_dict()`的方法，该方法返回一个Python字典：

*app/models.py*：User模型转换成表示。

```
from flask import url_for
# ...

class User(UserMixin, db.Model):
    # ...

    def to_dict(self, include_email=False):
        data = {
            'id': self.id,
            'username': self.username,
            'last_seen': self.last_seen.isoformat() + 'Z',
            'about_me': self.about_me,
            'post_count': self.posts.count(),
            'follower_count': self.followers.count(),
            'followed_count': self.followed.count(),
            '_links': {
                'self': url_for('api.get_user', id=self.id),
                'followers': url_for('api.get_followers', id=self.id),
                'followed': url_for('api.get_followed', id=self.id),
                'avatar': self.avatar(128)
            }
        }
        if include_email:
            data['email'] = self.email
        return data
```

该方法一目了然，只是简单地生成并返回用户表示的字典。正如我上面提到的那样，`email`字段需要特殊处理，因为我只想在用户请求自己的数据时才包含电子邮件。 所以我使用`include_email`标志来确定该字段是否包含在表示中。

注意一下`last_seen`字段的生成。 对于日期和时间字段，我将使用[ISO 8601](https://en.wikipedia.org/wiki/ISO_8601)格式，Python的`datetime`对象可以通过`isoformat()`方法生成这样格式的字符串。 但是因为我使用的`datetime`对象的时区是UTC，且但没有在其状态中记录时区，所以我需要在末尾添加`Z`，即ISO 8601的UTC时区代码。

最后，看看我如何实现超媒体链接。 对于指向应用其他路由的三个链接，我使用`url_for()`生成URL（目前指向我在*app/api/users.py*中定义的占位符视图函数）。 头像链接是特殊的，因为它是应用外部的Gravatar URL。 对于这个链接，我使用了与渲染网页中的头像的相同`avatar()`方法。

`to_dict()`方法将用户对象转换为Python表示，以后会被转换为JSON。 我还需要其反向处理的方法，即客户端在请求中传递用户表示，服务器需要解析并将其转换为`User`对象。 以下是实现从Python字典到`User`对象转换的`from_dict()`方法：

*app/models.py*：表示转换成User模型。

```
class User(UserMixin, db.Model):
    # ...

    def from_dict(self, data, new_user=False):
        for field in ['username', 'email', 'about_me']:
            if field in data:
                setattr(self, field, data[field])
        if new_user and 'password' in data:
            self.set_password(data['password'])
```

本处我决定使用循环来导入客户端可以设置的任何字段，即`username`，`email`和`about_me`。 对于每个字段，我检查它是否存在于`data`参数中，如果存在，我使用Python的`setattr()`在对象的相应属性中设置新值。

`password`字段被视为特例，因为它不是对象中的字段。 `new_user`参数确定了这是否是新的用户注册，这意味着`data`中包含`password`。 要在用户模型中设置密码，需要调用`set_password()`方法来创建密码哈希。

## 表示用户集合

除了使用单个资源表示形式外，此API还需要一组用户的表示。 例如客户请求用户或粉丝列表时使用的格式。 以下是一组用户的表示：

```
{
    "items": [
        { ... user resource ... },
        { ... user resource ... },
        ...
    ],
    "_meta": {
        "page": 1,
        "per_page": 10,
        "total_pages": 20,
        "total_items": 195
    },
    "_links": {
        "self": "http://localhost:5000/api/users?page=1",
        "next": "http://localhost:5000/api/users?page=2",
        "prev": null
    }
}
```

在这个表示中，`items`是用户资源的列表，每个用户资源的定义如前一节所述。 `_meta`部分包含集合的元数据，客户端在向用户渲染分页控件时就会用得上。 `_links`部分定义了相关链接，包括集合本身的链接以及上一页和下一页链接，也能帮助客户端对列表进行分页。

由于分页逻辑，生成用户集合的表示很棘手，但是该逻辑对于我将来可能要添加到此API的其他资源来说是一致的，所以我将以通用的方式实现它，以便适用于其他模型。 可以回顾[第十六章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e5%85%ad%e7%ab%a0%ef%bc%9a%e5%85%a8%e6%96%87%e6%90%9c%e7%b4%a2.md)，就会发现我目前的情况与全文索引类似，都是实现一个功能，还要让它可以应用于任何模型。 对于全文索引，我使用的解决方案是实现一个`SearchableMixin`类，任何需要全文索引的模型都可以从中继承。 我会故技重施，实现一个新的mixin类，我命名为`PaginatedAPIMixin`：

*app/models.py*：分页表示mixin类。

```
class PaginatedAPIMixin(object):
    @staticmethod
    def to_collection_dict(query, page, per_page, endpoint, **kwargs):
        resources = query.paginate(page, per_page, False)
        data = {
            'items': [item.to_dict() for item in resources.items],
            '_meta': {
                'page': page,
                'per_page': per_page,
                'total_pages': resources.pages,
                'total_items': resources.total
            },
            '_links': {
                'self': url_for(endpoint, page=page, per_page=per_page,
                                **kwargs),
                'next': url_for(endpoint, page=page + 1, per_page=per_page,
                                **kwargs) if resources.has_next else None,
                'prev': url_for(endpoint, page=page - 1, per_page=per_page,
                                **kwargs) if resources.has_prev else None
            }
        }
        return data
```

`to_collection_dict()`方法产生一个带有用户集合表示的字典，包括`items`，`_meta`和`_links`部分。 你可能需要仔细检查该方法以了解其工作原理。 前三个参数是Flask-SQLAlchemy查询对象，页码和每页数据数量。 这些是决定要返回的条目是什么的参数。 该实现使用查询对象的`paginate()`方法来获取该页的条目，就像我对主页，发现页和个人主页中的用户动态所做的一样。

复杂的部分是生成链接，其中包括自引用以及指向下一页和上一页的链接。 我想让这个函数具有通用性，所以我不能使用类似`url_for('api.get_users', id=id, page=page)`这样的代码来生成自链接（译者注：因为这样就固定成用户资源专用了）。 `url_for()`的参数将取决于特定的资源集合，所以我将依赖于调用者在`endpoint`参数中传递的值，来确定需要发送到`url_for()`的视图函数。 由于许多路由都需要参数，我还需要在`kwargs`中捕获更多关键字参数，并将它们传递给`url_for()`。 `page`和`per_page`查询字符串参数是明确给出的，因为它们控制所有API路由的分页。

这个mixin类需要作为父类添加到User模型中：

*app/models.py*：添加PaginatedAPIMixin到User模型中。

```
class User(PaginatedAPIMixin, UserMixin, db.Model):
    # ...
```

将集合转换成json表示，不需要反向操作，因为我不需要客户端发送用户列表到服务器。

## 错误处理

我在[第七章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E4%B8%83%E7%AB%A0%EF%BC%9A%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86.md)中定义的错误页面仅适用于使用Web浏览器的用户。当一个API需要返回一个错误时，它需要是一个“机器友好”的错误类型，以便客户端可以轻松解释这些错误。 因此，我同样设计错误的表示为一个JSON。 以下是我要使用的基本结构：

```
{
    "error": "short error description",
    "message": "error message (optional)"
}
```

除了错误的有效载荷之外，我还会使用HTTP协议的状态代码来指示常见错误的类型。 为了帮助我生成这些错误响应，我将在*app/api/errors.py*中写入`error_response()`函数：

*app/api/errors.py*：错误响应。

```
from flask import jsonify
from werkzeug.http import HTTP_STATUS_CODES

def error_response(status_code, message=None):
    payload = {'error': HTTP_STATUS_CODES.get(status_code, 'Unknown error')}
    if message:
        payload['message'] = message
    response = jsonify(payload)
    response.status_code = status_code
    return response
```

该函数使用来自Werkzeug（Flask的核心依赖项）的`HTTP_STATUS_CODES`字典，它为每个HTTP状态代码提供一个简短的描述性名称。 我在错误表示中使用这些名称作为`error`字段的值，所以我只需要操心数字状态码和可选的长描述。 `jsonify()`函数返回一个默认状态码为200的Flask`Response`对象，因此在创建响应之后，我将状态码设置为对应的错误代码。

API将返回的最常见错误将是代码400，代表了“错误的请求”。 这是客户端发送请求中包含无效数据的错误。 为了更容易产生这个错误，我将为它添加一个专用函数，只需传入长的描述性消息作为参数就可以调用。 下面是我之前添加的`bad_request()`占位符：

*app/api/errors.py*：错误请求的响应。

```
# ...

def bad_request(message):
    return error_response(400, message)
```

## 用户资源Endpoint

必需的用户JSON表示的支持已完成，因此我已准备好开始对API endpoint进行编码了。

### 检索单个用户

让我们就从使用给定的`id`来检索指定用户开始吧：

*app/api/users.py*：返回一个用户。

```
from flask import jsonify
from app.models import User

@bp.route('/users/<int:id>', methods=['GET'])
def get_user(id):
    return jsonify(User.query.get_or_404(id).to_dict())
```

视图函数接收被请求用户的`id`作为URL中的动态参数。 查询对象的`get_or_404()`方法是以前见过的`get()`方法的一个非常有用的变体，如果用户存在，它返回给定`id`的对象，当id不存在时，它会中止请求并向客户端返回一个404错误，而不是返回`None `。 `get_or_404()`比`get()`更有优势，它不需要检查查询结果，简化了视图函数中的逻辑。

我添加到User的`to_dict()`方法用于生成用户资源表示的字典，然后Flask的`jsonify()`函数将该字典转换为JSON格式的响应以返回给客户端。

如果你想查看第一条API路由的工作原理，请启动服务器，然后在浏览器的地址栏中输入以下URL：

```
http://localhost:5000/api/users/1
```

浏览器会以JSON格式显示第一个用户。 也尝试使用大一些的`id`值来查看SQLAlchemy查询对象的`get_or_404()`方法如何触发404错误（我将在稍后向你演示如何扩展错误处理，以便返回这些错误 JSON格式）。

为了测试这条新路由，我将安装[HTTPie](https://httpie.org/)，这是一个用Python编写的命令行HTTP客户端，可以轻松发送API请求：

```
(venv) $ pip install httpie
```

我现在可以请求`id`为`1`的用户（可能是你自己），命令如下：

```
(venv) $ http GET http://localhost:5000/api/users/1
HTTP/1.0 200 OK
Content-Length: 457
Content-Type: application/json
Date: Mon, 27 Nov 2017 20:19:01 GMT
Server: Werkzeug/0.12.2 Python/3.6.3

{
    "_links": {
        "avatar": "https://www.gravatar.com/avatar/993c...2724?d=identicon&s=128",
        "followed": "/api/users/1/followed",
        "followers": "/api/users/1/followers",
        "self": "/api/users/1"
    },
    "about_me": "Hello! I'm the author of the Flask Mega-Tutorial.",
    "followed_count": 0,
    "follower_count": 1,
    "id": 1,
    "last_seen": "2017-11-26T07:40:52.942865Z",
    "post_count": 10,
    "username": "miguel"
}
```

### 检索用户集合

要返回所有用户的集合，我现在可以依靠`PaginatedAPIMixin`的`to_collection_dict()`方法：

*app/api/users.py*：返回所有用户的集合。

```
from flask import request

@bp.route('/users', methods=['GET'])
def get_users():
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 10, type=int), 100)
    data = User.to_collection_dict(User.query, page, per_page, 'api.get_users')
    return jsonify(data)
```

对于这个实现，我首先从请求的查询字符串中提取`page`和`per_page`，如果它们没有被定义，则分别使用默认值1和10。 `per_page`具有额外的逻辑，以100为上限。 给客户端控件请求太大的页面并不是一个好主意，因为这可能会导致服务器的性能问题。 然后`page`和`per_page`以及query对象（在本例中，该查询只是`User.query`，是返回所有用户的最通用的查询）参数被传递给`to_collection_query()`方法。 最后一个参数是`api.get_users`，这是我在表示中使用的三个链接所需的endpoint名称。

要使用HTTPie测试此endpoint，请使用以下命令：

```
(venv) $ http GET http://localhost:5000/api/users
```

接下来的两个endpoint是返回粉丝集合和关注用户集合。 与上面的非常相似：

*app/api/users.py*：返回粉丝列表和关注用户列表。

```
@bp.route('/users/<int:id>/followers', methods=['GET'])
def get_followers(id):
    user = User.query.get_or_404(id)
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 10, type=int), 100)
    data = User.to_collection_dict(user.followers, page, per_page,
                                   'api.get_followers', id=id)
    return jsonify(data)

@bp.route('/users/<int:id>/followed', methods=['GET'])
def get_followed(id):
    user = User.query.get_or_404(id)
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 10, type=int), 100)
    data = User.to_collection_dict(user.followed, page, per_page,
                                   'api.get_followed', id=id)
    return jsonify(data)
```

由于这两条路由是特定于用户的，因此它们具有`id`动态参数。 `id`用于从数据库中获取用户，然后将`user.followers`和`user.followed`关系查询提供给`to_collection_dict()`，所以希望现在你可以看到，花费一点点额外的时间，并以通用的方式设计该方法，对于获得的回报而言是值得的。 `to_collection_dict()`的最后两个参数是endpoint名称和`id`，`id`将在`kwargs`中作为一个额外关键字参数，然后在生成链接时将它传递给`url_for()` 。

和前面的示例类似，你可以使用HTTPie来测试这两个路由，如下所示：

```
(venv) $ http GET http://localhost:5000/api/users/1/followers
(venv) $ http GET http://localhost:5000/api/users/1/followed
```

由于超媒体，你不需要记住这些URL，因为它们包含在用户表示的`_links`部分。

### 注册新用户

*/users*路由的`POST`请求将用于注册新的用户帐户。 你可以在下面看到这条路由的实现：

*app/api/users.py*：注册新用户。

```
from flask import url_for
from app import db
from app.api.errors import bad_request

@bp.route('/users', methods=['POST'])
def create_user():
    data = request.get_json() or {}
    if 'username' not in data or 'email' not in data or 'password' not in data:
        return bad_request('must include username, email and password fields')
    if User.query.filter_by(username=data['username']).first():
        return bad_request('please use a different username')
    if User.query.filter_by(email=data['email']).first():
        return bad_request('please use a different email address')
    user = User()
    user.from_dict(data, new_user=True)
    db.session.add(user)
    db.session.commit()
    response = jsonify(user.to_dict())
    response.status_code = 201
    response.headers['Location'] = url_for('api.get_user', id=user.id)
    return response
```

该请求将接受请求主体中提供的来自客户端的JSON格式的用户表示。 Flask提供`request.get_json()`方法从请求中提取JSON并将其作为Python结构返回。 如果在请求中没有找到JSON数据，该方法返回`None`，所以我可以使用表达式`request.get_json() or {}`确保我总是可以获得一个字典。

在我可以使用这些数据之前，我需要确保我已经掌握了所有信息，因此我首先检查是否包含三个必填字段，`username`， `email`和`password`。 如果其中任何一个缺失，那么我使用*app/api/errors.py*模块中的`bad_request()`辅助函数向客户端返回一个错误。 除此之外，我还需要确保`username`和`email`字段尚未被其他用户使用，因此我尝试使用获得的用户名和电子邮件从数据库中加载用户，如果返回了有效的用户，那么我也将返回错误给客户端。

一旦通过了数据验证，我可以轻松创建一个用户对象并将其添加到数据库中。 为了创建用户，我依赖`User`模型中的`from_dict()`方法，`new_user`参数被设置为`True`，所以它也接受通常不存在于用户表示中的`password`字段。

我为这个请求返回的响应将是新用户的表示，所以使用`to_dict()`产生它的有效载荷。 创建资源的`POST`请求的响应状态代码应该是201，即创建新实体时使用的代码。 此外，HTTP协议要求201响应包含一个值为新资源URL的`Location`头部。

下面你可以看到如何通过HTTPie从命令行注册一个新用户：

```
(venv) $ http POST http://localhost:5000/api/users username=alice password=dog \
    email=alice@example.com "about_me=Hello, my name is Alice!"
```

### 编辑用户

示例API中使用的最后一个endpoint用于修改已存在的用户：

*app/api/users.py*：修改用户。

```
@bp.route('/users/<int:id>', methods=['PUT'])
def update_user(id):
    user = User.query.get_or_404(id)
    data = request.get_json() or {}
    if 'username' in data and data['username'] != user.username and \
            User.query.filter_by(username=data['username']).first():
        return bad_request('please use a different username')
    if 'email' in data and data['email'] != user.email and \
            User.query.filter_by(email=data['email']).first():
        return bad_request('please use a different email address')
    user.from_dict(data, new_user=False)
    db.session.commit()
    return jsonify(user.to_dict())
```

一个请求到来，我通过URL收到一个动态的用户`id`，所以我可以加载指定的用户或返回404错误（如果找不到）。 就像注册新用户一样，我需要验证客户端提供的`username`和`email`字段是否与其他用户发生了冲突，但在这种情况下，验证有点棘手。 首先，这些字段在此请求中是可选的，所以我需要检查字段是否存在。 第二个复杂因素是客户端可能提供与目前字段相同的值，所以在检查用户名或电子邮件是否被采用之前，我需要确保它们与当前的不同。 如果任何验证检查失败，那么我会像之前一样返回400错误给客户端。

一旦数据验证通过，我可以使用`User`模型的`from_dict()`方法导入客户端提供的所有数据，然后将更改提交到数据库。 该请求的响应会将更新后的用户表示返回给用户，并使用默认的200状态代码。

以下是一个示例请求，它用HTTPie编辑`about_me`字段：

```
(venv) $ http PUT http://localhost:5000/api/users/2 "about_me=Hi, I am Miguel"
```

## API 认证

我在前一节中添加的API endpoint当前对任何客户端都是开放的。 显然，执行这些操作需要认证用户才安全，为此我需要添加*认证*和*授权*，简称“AuthN”和“AuthZ”。 思路是，客户端发送的请求提供了某种标识，以便服务器知道客户端代表的是哪位用户，并且可以验证是否允许该用户执行请求的操作。

保护这些API endpoint的最明显的方法是使用Flask-Login中的`@login_required`装饰器，但是这种方法存在一些问题。 装饰器检测到未通过身份验证的用户时，会将用户重定向到HTML登录页面。 在API中没有HTML或登录页面的概念，如果客户端发送带有无效或缺少凭证的请求，服务器必须拒绝请求并返回401状态码。 服务器不能假定API客户端是Web浏览器，或者它可以处理重定向，或者它可以渲染和处理HTML登录表单。 当API客户端收到401状态码时，它知道它需要向用户询问凭证，但是它是如何实现的，服务器不需要关心。

### User模型中实现Token

对于API身份验证需求，我将使用*token*身份验证方案。 当客户端想要开始与API交互时，它需要使用用户名和密码进行验证，然后获得一个临时token。 只要token有效，客户端就可以发送附带token的API请求以通过认证。 一旦token到期，需要请求新的token。 为了支持用户token，我将扩展`User`模型：

*app/models.py*：支持用户token。

```
import base64
from datetime import datetime, timedelta
import os

class User(UserMixin, PaginatedAPIMixin, db.Model):
    # ...
    token = db.Column(db.String(32), index=True, unique=True)
    token_expiration = db.Column(db.DateTime)

    # ...

    def get_token(self, expires_in=3600):
        now = datetime.utcnow()
        if self.token and self.token_expiration > now + timedelta(seconds=60):
            return self.token
        self.token = base64.b64encode(os.urandom(24)).decode('utf-8')
        self.token_expiration = now + timedelta(seconds=expires_in)
        db.session.add(self)
        return self.token

    def revoke_token(self):
        self.token_expiration = datetime.utcnow() - timedelta(seconds=1)

    @staticmethod
    def check_token(token):
        user = User.query.filter_by(token=token).first()
        if user is None or user.token_expiration < datetime.utcnow():
            return None
        return user
```

我为用户模型添加了一个`token`属性，并且因为我需要通过它搜索数据库，所以我为它设置了唯一性和索引。 我还添加了`token_expiration`字段，它保存token过期的日期和时间。 这使得token不会长时间有效，以免成为安全风险。

我创建了三种方法来处理这些token。 `get_token()`方法为用户返回一个token。 以base64编码的24位随机字符串来生成这个token，以便所有字符都处于可读字符串范围内。 在创建新token之前，此方法会检查当前分配的token在到期之前是否至少还剩一分钟，并且在这种情况下会返回现有的token。

使用token时，有一个策略可以立即使token失效总是一件好事，而不是仅依赖到期日期。 这是一个经常被忽视的安全最佳实践。 `revoke_token()`方法使得当前分配给用户的token失效，只需设置到期时间为当前时间的前一秒。

`check_token()`方法是一个静态方法，它将一个token作为参数传入并返回此token所属的用户。 如果token无效或过期，则该方法返回`None`。

由于我对数据库进行了更改，因此需要生成新的数据库迁移，然后使用它升级数据库：

```
(venv) $ flask db migrate -m "user tokens"
(venv) $ flask db upgrade
```

### 带Token的请求

当你编写一个API时，你必须考虑到你的客户端并不总是要连接到Web应用程序的Web浏览器。 当独立客户端（如智能手机APP）甚至是基于浏览器的单页应用程序访问后端服务时，API展示力量的机会就来了。 当这些专用客户端需要访问API服务时，他们首先需要请求token，对应传统Web应用程序中登录表单的部分。

为了简化使用token认证时客户端和服务器之间的交互，我将使用名为[Flask-HTTPAuth](https://flask-httpauth.readthedocs.io/)的Flask插件。 Flask-HTTPAuth可以使用pip安装：

```
(venv) $ pip install flask-httpauth
```

Flask-HTTPAuth支持几种不同的认证机制，都对API友好。 首先，我将使用[HTTPBasic Authentication](https://en.wikipedia.org/wiki/Basic_access_authentication)，该机制要求客户端在标准的[Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization)头部中附带用户凭证。 要与Flask-HTTPAuth集成，应用需要提供两个函数：一个用于检查用户提供的用户名和密码，另一个用于在认证失败的情况下返回错误响应。这些函数通过装饰器在Flask-HTTPAuth中注册，然后在认证流程中根据需要由插件自动调用。 实现如下：

*app/api/auth.py*：基本认证支持。

```
from flask import g
from flask_httpauth import HTTPBasicAuth
from app.models import User
from app.api.errors import error_response

basic_auth = HTTPBasicAuth()

@basic_auth.verify_password
def verify_password(username, password):
    user = User.query.filter_by(username=username).first()
    if user is None:
        return False
    g.current_user = user
    return user.check_password(password)

@basic_auth.error_handler
def basic_auth_error():
    return error_response(401)
```

Flask-HTTPAuth的`HTTPBasicAuth`类实现了基本的认证流程。 这两个必需的函数分别通过`verify_password`和`error_handler`装饰器进行注册。

验证函数接收客户端提供的用户名和密码，如果凭证有效则返回`True`，否则返回`False`。 我依赖`User`类的`check_password()`方法来检查密码，它在Web应用的认证过程中，也会被Flask-Login使用。 我将认证用户保存在`g.current_user`中，以便我可以从API视图函数中访问它。

错误处理函数只返回由*app/api/errors.py*模块中的`error_response()`函数生成的401错误。 401错误在HTTP标准中定义为“未授权”错误。 HTTP客户端知道当它们收到这个错误时，需要重新发送有效的凭证。

现在我已经实现了基本认证的支持，因此我可以添加一条token检索路由，以便客户端在需要token时调用：

*app/api/tokens.py*：生成用户token。

```
from flask import jsonify, g
from app import db
from app.api import bp
from app.api.auth import basic_auth

@bp.route('/tokens', methods=['POST'])
@basic_auth.login_required
def get_token():
    token = g.current_user.get_token()
    db.session.commit()
    return jsonify({'token': token})
```

这个视图函数使用了`HTTPBasicAuth`实例中的`@basic_auth.login_required`装饰器，它将指示Flask-HTTPAuth验证身份（通过我上面定义的验证函数），并且仅当提供的凭证是有效的才运行下面的视图函数。 该视图函数的实现依赖于用户模型的`get_token()`方法来生成token。 数据库提交在生成token后发出，以确保token及其到期时间被写回到数据库。

如果你尝试直接向token API路由发送POST请求，则会发生以下情况：

```
(venv) $ http POST http://localhost:5000/api/tokens
HTTP/1.0 401 UNAUTHORIZED
Content-Length: 30
Content-Type: application/json
Date: Mon, 27 Nov 2017 20:01:00 GMT
Server: Werkzeug/0.12.2 Python/3.6.3
WWW-Authenticate: Basic realm="Authentication Required"

{
    "error": "Unauthorized"
}
```

HTTP响应包括401状态码和我在`basic_auth_error()`函数中定义的错误负载。 下面请求带上了基本认证需要的凭证：

```
(venv) $ http --auth <username>:<password> POST http://localhost:5000/api/tokens
HTTP/1.0 200 OK
Content-Length: 50
Content-Type: application/json
Date: Mon, 27 Nov 2017 20:01:22 GMT
Server: Werkzeug/0.12.2 Python/3.6.3

{
    "token": "pC1Nu9wwyNt8VCj1trWilFdFI276AcbS"
}
```

现在状态码是200，这是成功请求的代码，并且有效载荷包括用户的token。 请注意，当你发送这个请求时，你需要用你自己的凭证来替换`<username>:<password>`。 用户名和密码需要以冒号作为分隔符。

### 使用Token机制保护API路由

客户端现在可以请求一个token来和API endpoint一起使用，所以剩下的就是向这些endpoint添加token验证。 Flask-HTTPAuth也可以为我处理的这些事情。 我需要创建基于`HTTPTokenAuth`类的第二个身份验证实例，并提供token验证回调：

*app/api/auth.py*: Token认证支持。

```
# ...
from flask_httpauth import HTTPTokenAuth

# ...
token_auth = HTTPTokenAuth()

# ...

@token_auth.verify_token
def verify_token(token):
    g.current_user = User.check_token(token) if token else None
    return g.current_user is not None

@token_auth.error_handler
def token_auth_error():
    return error_response(401)
```

使用token认证时，Flask-HTTPAuth使用的是`verify_token`装饰器注册验证函数，除此之外，token认证的工作方式与基本认证相同。 我的token验证函数使用`User.check_token()`来定位token所属的用户。 该函数还通过将当前用户设置为`None`来处理缺失token的情况。返回值是`True`还是`False`，决定了Flask-HTTPAuth是否允许视图函数的运行。

为了使用token保护API路由，需要添加`@token_auth.login_required`装饰器：

*app/api/users.py*：使用token认证保护用户路由。

```
from app.api.auth import token_auth

@bp.route('/users/<int:id>', methods=['GET'])
@token_auth.login_required
def get_user(id):
    # ...

@bp.route('/users', methods=['GET'])
@token_auth.login_required
def get_users():
    # ...

@bp.route('/users/<int:id>/followers', methods=['GET'])
@token_auth.login_required
def get_followers(id):
    # ...

@bp.route('/users/<int:id>/followed', methods=['GET'])
@token_auth.login_required
def get_followed(id):
    # ...

@bp.route('/users', methods=['POST'])
def create_user():
    # ...

@bp.route('/users/<int:id>', methods=['PUT'])
@token_auth.login_required
def update_user(id):
    # ...
```

请注意，装饰器被添加到除`create_user()`之外的所有API视图函数中，显而易见，这个函数不能使用token认证，因为用户都不存在时，更不会有token了。

如果你直接对上面列出的受token保护的endpoint发起请求，则会得到一个401错误。为了成功访问，你需要添加`Authorization`头部，其值是请求 */api/tokens* 获得的token的值。Flask-HTTPAuth期望的是"不记名"token，但是它没有被HTTPie直接支持。就像针对基本认证，HTTPie提供了`--auth`选项来接受用户名和密码，但是token的头部则需要显式地提供了。下面是发送不记名token的格式：

```
(venv) $ http GET http://localhost:5000/api/users/1 \
    "Authorization:Bearer pC1Nu9wwyNt8VCj1trWilFdFI276AcbS"
```

### 撤销Token

我将要实现的最后一个token相关功能是token撤销，如下所示:

*app/api/tokens.py*：撤销token。

```
from app.api.auth import token_auth

@bp.route('/tokens', methods=['DELETE'])
@token_auth.login_required
def revoke_token():
    g.current_user.revoke_token()
    db.session.commit()
    return '', 204
```

客户端可以向 */tokens* URL发送`DELETE`请求，以使token失效。此路由的身份验证是基于token的，事实上，在`Authorization`头部中发送的token就是需要被撤销的。撤销使用了`User`类中的辅助方法，该方法重新设置token过期日期来实现撤销操作。之后提交数据库会话，以确保将更改写入数据库。这个请求的响应没有正文，所以我可以返回一个空字符串。Return语句中的第二个值设置状态代码为204，该代码用于成功请求却没有响应主体的响应。

下面是撤销token的一个HTTPie请求示例：

```
(venv) $ http DELETE http://localhost:5000/api/tokens \
    Authorization:"Bearer pC1Nu9wwyNt8VCj1trWilFdFI276AcbS"
```

## API友好的错误消息

你是否还记得，在本章的前部分，当我要求你用一个无效的用户URL从浏览器发送一个API请求时发生了什么?服务器返回了404错误，但是这个错误被格式化为标准的404 HTML错误页面。在API blueprint中的API可能返回的许多错误可以被重写为JSON版本，但是仍然有一些错误是由Flask处理的，处理这些错误的处理函数是被全局注册到应用中的，返回的是HTML。

HTTP协议支持一种机制，通过该机制，客户机和服务器可以就响应的最佳格式达成一致，称为*内容协商*。客户端需要发送一个`Accept`头部，指示格式首选项。然后，服务器查看自身格式列表并使用匹配客户端格式列表中的最佳格式进行响应。

我想做的是修改全局应用的错误处理器，使它们能够根据客户端的格式首选项对返回内容是使用HTML还是JSON进行内容协商。这可以通过使用Flask的`request.accept_mimetypes`来完成:

*app/errors/handlers.py*：为错误响应进行内容协商。

```
from flask import render_template, request
from app import db
from app.errors import bp
from app.api.errors import error_response as api_error_response

def wants_json_response():
    return request.accept_mimetypes['application/json'] >= \
        request.accept_mimetypes['text/html']

@bp.app_errorhandler(404)
def not_found_error(error):
    if wants_json_response():
        return api_error_response(404)
    return render_template('errors/404.html'), 404

@bp.app_errorhandler(500)
def internal_error(error):
    db.session.rollback()
    if wants_json_response():
        return api_error_response(500)
    return render_template('errors/500.html'), 500
```

`wants_json_response()`辅助函数比较客户端对JSON和HTML格式的偏好程度。 如果JSON比HTML高，那么我会返回一个JSON响应。 否则，我会返回原始的基于模板的HTML响应。 对于JSON响应，我将使用从API blueprint中导入`error_response`辅助函数，但在这里我要将其重命名为`api_error_response()`，以便清楚它的作用和来历。

