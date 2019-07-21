本文翻译自[The Flask Mega-Tutorial Part XX: Some JavaScript Magic](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xx-some-javascript-magic)

这是Flask Mega-Tutorial系列的第二十部分，我将添加一个功能，当你将鼠标悬停在用户的昵称上时，会弹出一个漂亮的窗口。

现在，构建一个Web应用而不使用JavaScript是不可能的。 你一定知道，JavaScript是Web浏览器中可本地运行的唯一语言。 在[第十四章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e5%9b%9b%e7%ab%a0%ef%bc%9aAjax.md)中，你看到我在Flask模板中添加了一个简单的JavaScript的启用链接，以提供实时的用户动态的语言翻译。 而在本章中，我将深入探讨该主题，并向你展示另一个有用的JavaScript技巧，给应用程序增添趣味来吸引用户。

社交网站的常见用户交互模式是，当你将鼠标悬停在用户的名称上时，可以在弹出窗口中显示用户的主要信息。 如果你从未注意到这一点，请访问Twitter，Facebook，LinkedIn或任何其他主要社交网站，当你看到用户名时，只需将鼠标指针放在上面几秒钟即可看到弹出窗口。 本章将致力于为Microblog实现该功能，你可以在下面看到预览效果：

![User Popup](http://upload-images.jianshu.io/upload_images/4961528-d34608ca599c0ca2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*本章的GitHub链接为：[Browse](https://github.com/miguelgrinberg/microblog/tree/v0.20), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.20.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.19...v0.20).*

## 服务端的支持

在深入研究客户端之前，让我们先了解一下支持这些用户弹窗所需的服务器端的工作。 用户弹窗的内容将由新路由返回，它是现有个人主页路由的简化版本。 视图函数如下：

*app/main/routes.py*：用户弹窗视图函数。

```
@bp.route('/user/<username>/popup')
@login_required
def user_popup(username):
    user = User.query.filter_by(username=username).first_or_404()
    return render_template('user_popup.html', user=user)
```

该路由将被附加到 */user/\<username\>/popup* URL，并且将简单地加载所请求的用户，然后渲染到模板中。 该模板是个人主页的简化版本：

*app/templates/user_popup.html*：用户弹窗模板。

```
<table class="table">
    <tr>
        <td width="64" style="border: 0px;"><img src="{{ user.avatar(64) }}"></td>
        <td style="border: 0px;">
            <p>
                <a href="{{ url_for('main.user', username=user.username) }}">
                    {{ user.username }}
                </a>
            </p>
            <small>
                {% if user.about_me %}<p>{{ user.about_me }}</p>{% endif %}
                {% if user.last_seen %}
                <p>{{ _('Last seen on') }}: 
                   {{ moment(user.last_seen).format('lll') }}</p>
                {% endif %}
                <p>{{ _('%(count)d followers', count=user.followers.count()) }},
                   {{ _('%(count)d following', count=user.followed.count()) }}</p>
                {% if user != current_user %}
                    {% if not current_user.is_following(user) %}
                    <a href="{{ url_for('main.follow', username=user.username) }}">
                        {{ _('Follow') }}
                    </a>
                    {% else %}
                    <a href="{{ url_for('main.unfollow', username=user.username) }}">
                        {{ _('Unfollow') }}
                    </a>
                    {% endif %}
                {% endif %}
            </small>
        </td>
    </tr>
</table>
```

当用户将鼠标指针悬停在用户名上时，随后小节中编写的JavaScript代码将会调用此路由。客户端将服务器端返回的响应中的html内容显示在弹出窗口中。 当用户移开鼠标时，弹出窗口将被删除。 听起来很简单，对吧？

如果你想了解弹窗像什么样，现在可以运行应用，跳转到任何用户的个人主页，然后在地址栏的URL中追加 */popup* 以查看全屏版本的弹出窗口内容。

## Bootstrap Popover组件简介

在[第十一章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e4%b8%80%e7%ab%a0%ef%bc%9a%e7%be%8e%e5%8c%96.md)中，我向你介绍了可便捷地创建精美网页的Bootstrap框架。 到目前为止，我只使用了这个框架的一小部分。 Bootstrap捆绑了许多常见的UI元素，所有这些元素都在地址为 *https://getbootstrap.com* 的Bootstrap文档中有demo和示例。 其中一个组件是[Popover](https://getbootstrap.com/docs/3.3/javascript/#popovers)（弹窗），在文档中将其描述为“用于容纳辅助信息的小的覆盖窗口”。 这正是我需要的！  

大多数bootstrap组件都是通过HTML标记定义的，该标记引用Bootstrap CSS的定义内容来添加漂亮的样式。 一些高级的组件还需要JavaScript。 应用程序在网页中包含这些组件的标准方式是在适当的位置添加HTML，然后为需要脚本支持的组件调用JavaScript函数，以便初始化或激活它。 popover组件确实需要JavaScript的支持。

要做弹窗的HTML部分非常简单，你只需要定义将触发弹窗的元素。 就我而言，就是处理每条用户动态中出现的可点击的用户名。
*app/templates/_post.html*子模板具有已定义的用户名：

```
            <a href="{{ url_for('main.user', username=post.author.username) }}">
                {{ post.author.username }}
            </a>
```

现在根据popover文档，我需要调用每个链接上的`popover()` JavaScript函数，就像上面出现在页面上的链接一样，这才能初始化弹出窗口。 初始化调用接受许多配置弹出窗口的选项，包括传递想要在弹出窗口中显示的内容，以及使用什么方法触发弹出窗口出现或消失（单击，悬停在元素上等），如果内容是纯文本或HTML，那么在文档中可以找到更多的选项。不幸的是，在阅读完这些信息之后，我的疑惑更多了，因为这个组件看起来并没有按照我需要的方式工作。 以下是我实现此功能需要解决的问题列表：

* 在页面中会有很多用户名链接，每条用户动态都会显示一个。我需要有一种方法可以在页面渲染后用JavaScript中找到所有这些链接，以便我可以将它们初始化为弹出窗口。
* Bootstrap文档中的popover示例都将目标HTML元素的`data-content`属性设置为popover的内容，因此当触发悬停事件时，Bootstrap需要做的只是显示弹出窗口。这对我来说要做的就不止这些了，因为我想对服务器进行Ajax调用以获取内容，并且只有当收到服务器的响应时，我才希望弹出窗口出现。
* 使用“悬停”模式时，只要你将鼠标指针放在目标元素中，弹出窗口就会保持可见状态。当你移开鼠标时，弹出窗口将消失。这具有糟糕的副作用，即如果用户想要将鼠标指针移动到弹出窗口中，弹出窗口将消失。我需要找出一种方法来将悬停行为扩展为包含弹出窗口，以便用户可以移动到弹出窗口中，例如，单击那里的链接。

在开发基于浏览器的应用程序时，事情变得越来越复杂的情况，实际上并不罕见。 你必须非常仔细地考虑DOM元素如何相互作用，并使其行为方式提供良好的用户体验。

## 在页面加载完成后执行函数

很明显，我将需要在每个页面加载后立即运行一些JavaScript代码。 我要运行的函数将搜索页面中用户名的所有链接，并使用Bootstrap中的弹出窗口组件配置它们。

jQuery JavaScript库作为Bootstrap的依赖项加载，因此我将利用它。 当使用jQuery时，你可以用`$(...)`封装来注册一个函数，函数将会在页面加载完毕后运行。 我可以将它添加到*app/templates/base.html*模板中，以便它可以在应用程序的每个页面上运行：

*app/templates/base.html*：页面加载完毕后运行函数。

```
...
<script>
    // ...

    $(function() {
        // write start up code here
    });
</script>
```

正如你所看到的，我已经在`<script>`元素中添加了我的启动函数，而在[第十四章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e5%9b%9b%e7%ab%a0%ef%bc%9aAjax.md)中，我已在该元素中定义了中的`translate()`函数。

## 使用选择器查找DOM元素

第一个要解决的问题是创建一个JavaScript函数来查找页面中的所有用户链接。 这个函数将在页面加载完成时运行，并且当完成时，将为所有页面配置悬停和弹出行为。 现在我要集中精力来寻找链接。

回顾[第十四章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e5%9b%9b%e7%ab%a0%ef%bc%9aAjax.md)，在实时翻译中被调用的HTML元素具有唯一的ID。 例如，ID = 123的用户动态中具有`id="post123"`属性。 然后使用jQuery，在JavaScript中使用表达式`$('#post123')`在DOM中定位此元素。 `$()`函数功能非常强大，并且具有相当复杂的查询语言来搜索DOM元素，可以参考[CSS Selectors](https://api.jquery.com/category/selectors/)。

我用于翻译功能的选择器旨在使用`id`属性查找一个具有唯一标识符的特定元素。 识别元素的另一种方法是使用`class`属性，它可以分配给页面中的多个元素。 例如，我可以用`class="user_popup"`标记所有的用户链接，然后我可以通过`$('.user_popup')`获取这些元素的列表（CSS选择器中，`#`前缀代表查询id属性，`.`前缀代表查询class属性）。 在本处，返回值将是具有该类的所有元素的集合。

## 弹窗和DOM元素

通过使用Bootstrap文档中的弹出窗口示例并在浏览器的调试器中检查DOM，我确定Bootstrap将弹出窗口组件创建为DOM中目标元素的同级元素。 正如我上面提到的，这会影响悬停事件的行为，只要用户将鼠标从`<a>`链接移动到弹出窗口本身，就会触发“鼠标移出”事件。

我可以扩展悬停事件以包含弹出窗口，就是将弹出窗口作为目标元素的子元素，这样悬停事件就会继承。 通过查看文档中的弹出选项，可以通过在`container`选项中传递父元素来完成此操作。

将popover作为悬停元素的子元素可以很好地用于按钮或一般的`<div>`或`<span>`元素，但在我的情况下，popover的target将是显示用户名的可点击链接的`<a> `元素。 使popover成为`<a>`元素的子元素的问题是，弹出窗口将获得`<a>`父元素的链接行为。 最终的结果是这样的：

```
        <a href="..." class="user_popup">
            username
            <div> ... popover elements here ... </div>
        </a>
```

为了避免弹出窗口出现在`<a>`元素中，我要使用的是另一个技巧。 我要将`<a>`元素封装在`<span>`元素中，然后将悬停事件和弹出窗口与`<span>`相关联。 由此产生的结构将是：

```
        <span class="user_popup">
            <a href="...">
                username
            </a>
            <div> ... popover elements here ... </div>
        </span>
```

`<div>`和`<span>`元素是不可见的，因此它们是用于帮助组织和构建DOM的重要元素。 `div`元素是*块元素*，有点像HTML文档中的段落，而`<span>`元素是*行内元素*，它可以用于字词级别。 本处，我决定使用`<span>`元素，因为我要包装的`<a>`元素也是行内元素。

因此，我将继续并重构我的*app/templates/_post.html*子模板以包含`<span>`元素：

```
...
                {% set user_link %}
                    <span class="user_popup">
                        <a href="{{ url_for('main.user', username=post.author.username) }}">
                            {{ post.author.username }}
                        </a>
                    </span>
                {% endset %}
...
```

如果你想知道弹出式HTML元素在哪里，好消息是我不必操心这一点。 当我在刚刚创建的`<span>`元素上调用`popover()`初始化函数时，Bootstrap框架会为我动态地插入弹出组件。

## 悬停事件

正如我上面提到的，Bootstrap中的popover组件使用的悬停行为不够灵活，无法满足我的需求，但如果你查看`trigger`选项的文档，则`hover`只是其中一个可能的值。 一个引起我注意的是`manual`模式，在这种模式下，可以通过JavaScript调用手动显示或删除弹出窗口，这种模式可以让我自由地实现悬停逻辑，所以我将使用该选项并实现我自己的悬停事件处理程序，并以我需要的方式工作。

所以我的下一步是将一个“hover”事件附加到页面中的所有链接。 使用jQuery，可以通过调用`element.hover(handlerIn, handlerOut)`将悬停事件附加到任何HTML元素。 如果在元素集合上调用这个函数，jQuery方便地将事件附加到所有元素上。 这两个参数是两个函数，分别在用户将鼠标指针移入和移出目标元素时调用对应的函数。

*app/templates/base.html*：悬停事件。

```
    $(function() {
        $('.user_popup').hover(
            function(event) {
                // mouse in event handler
                var elem = event.currentTarget;
            },
            function(event) {
                // mouse out event handler
                var elem = event.currentTarget;
            }
        )
    });
```

事件参数是一个事件对象，它包含了一些有用的信息。 在本处，我使用`event.currentTarget`来提取事件的目标元素。

浏览器在鼠标进入受影响的元素后立即调度悬停事件。 针对弹出行为，你只想鼠标停留在元素上一段时间才能激活，以防当鼠标指针短暂通过元素但不停留在元素上时出现弹出闪烁。 由于该事件不支持延迟，因此这是我需要自己实现的另一件事情。 所以我打算在“鼠标进入”事件处理程序中添加一秒计时器：

*app/templates/base.html*：悬停延迟。

```
    $(function() {
        var timer = null;
        $('.user_popup').hover(
            function(event) {
                // mouse in event handler
                var elem = event.currentTarget;
                timer = setTimeout(function() {
                    timer = null;
                    // popup logic goes here
                }, 1000);
            },
            function(event) {
                // mouse out event handler
                var elem = event.currentTarget;
                if (timer) {
                    clearTimeout(timer);
                    timer = null;
                }
            }
        )
    });
```

`setTimeout()`函数在浏览器环境中才可用。 它需要两个参数，函数和毫秒单位的时间。 `setTimeout()`的效果是函数在给定的延迟后被调用。 所以我添加了一个函数（现在是空的），将在悬停事件的一秒钟后被调用。 由于JavaScript语言中的闭包机制，此函数可以访问在外部作用域中定义的变量，例如`elem`。

我将timer对象存储在`hover()`调用之外定义的`timer`变量中，以使timer对象也可以被“mouse out”处理程序访问。 我需要这么做的原因是为了获得良好的用户体验。 如果用户将鼠标指针移动到其中一个用户链接中，并在移动它之前停留了半秒钟，我不希望该timer继续运行并调用显示弹出窗口的函数。 所以我的鼠标移出事件处理程序检查是否有一个活动的timer对象，如果有，就取消它。

## Ajax请求

Ajax请求不是一个新话题了，因为我已经在[第十四章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%e7%ac%ac%e5%8d%81%e5%9b%9b%e7%ab%a0%ef%bc%9aAjax.md)中已介绍过这个主题，来作为实时语言翻译功能。 当使用jQuery时，`$.ajax()`函数向服务器发送一个异步请求。

我要发送到服务器的请求将具有类似 */user/\<username\>/popup* 模式的URL，在本章开始时我已经将该URL添加到应用程序中。 这个请求的响应将包含我需要在弹出窗口中插入的HTML。

关于这个请求的直接问题是我需要知道包含在URL中的“username”的值是什么。 鼠标进入的事件处理函数是通用的，它将在页面中找到的所有用户链接，所以该函数需要从其上下文中确定用户名。

`elem`变量包含悬停事件中的目标元素，它是包裹`<a>`元素的`<span>`元素。 为了提取用户名，我可以从`<span>`开始浏览DOM，移至第一个子元素，即`<a>`元素，然后从中提取文本，这就是在网址中要使用的用户名 。 使用jQuery的DOM遍历函数，可以很简单地做到：

```
elem.first().text().trim()
```

应用于DOM节点的`first()`函数返回其第一个子节点。 `text()`函数返回节点的文本内容。 该函数不会对文本进行任何修剪，例如，如果在一行中有`<a>`，在下一行中有文本，在另一行中有`</a>`，`text()`将返回文本周围的所有空白。 为了消除所有空白并只留下文本，我使用了名为`trim()`的JavaScript函数。

这就是我需要向服务器发出请求的所有信息：

*app/templates/base.html*：XHR请求。

```
    $(function() {
        var timer = null;
        var xhr = null;
        $('.user_popup').hover(
            function(event) {
                // mouse in event handler
                var elem = $(event.currentTarget);
                timer = setTimeout(function() {
                    timer = null;
                    xhr = $.ajax(
                        '/user/' + elem.first().text().trim() + '/popup').done(
                            function(data) {
                                xhr = null
                                // create and display popup here
                            }
                        );
                }, 1000);
            },
            function(event) {
                // mouse out event handler
                var elem = $(event.currentTarget);
                if (timer) {
                    clearTimeout(timer);
                    timer = null;
                }
                else if (xhr) {
                    xhr.abort();
                    xhr = null;
                }
                else {
                    // destroy popup here
                }
            }
        )
    });
```

代码中，我在外部范围中定义了一个新变量`xhr`。 这个变量将保存我通过调用`$.ajax()`来初始化的异步请求对象。 不幸的是，当直接在JavaScript端构建URL时，我无法使用Flask中的`url_for()`，所以在这种情况下，我必须显式连接URL的各个部分。

`$.ajax()`调用返回一个promise，这是一个代表异步操作的特殊JavaScript对象。 我可以通过添加`.done(function)`来附加一个完成回调函数，所以一旦请求完成，我的回调函数就会被调用。 回调函数将接收到的响应作为参数，你可以在上面的代码中看到，我将其命名为`data`。 这将是我要放入popover的HTML内容。

但在我们获得弹窗之前，还有一个细节需要处理，以便给予用户一个良好的体验。 回想一下之前添加的逻辑，如果用户在触发鼠标进入事件之后的一秒内将鼠标指针移出`<span>`，将触发取消弹窗的逻辑。 同样的逻辑也需要应用于异步请求，所以我添加了第二个子句来放弃我的`xhr`请求对象（如果存在）。

## 弹窗的创建和销毁

最后我使用在Ajax回调函数中传递给我的`data`参数来创建我的弹窗组件：

*app/templates/base.html*：显示弹窗。

```
                                function(data) {
                                    xhr = null;
                                    elem.popover({
                                        trigger: 'manual',
                                        html: true,
                                        animation: false,
                                        container: elem,
                                        content: data
                                    }).popover('show');
                                    flask_moment_render_all();
                                }
```

弹出窗口的实际创建非常简单，Bootstrap的`popover()`函数完成设置所需的所有工作。 弹出窗口的选项作为参数给出。 我已经用`manual`触发模式，HTML内容，没有淡入淡出的动画（这样它就会更快地出现和消失）配置了这个弹出窗口，并且我已经将父元素设置为`<span>`元素本身，所以悬停行为通过继承扩展到弹出窗口。 最后，我将Ajax回调函数的`data`参数作为`content`参数的值。

`popover()`调用创建了一个弹窗组件，该组件也具有一个名为`popover()`的方法来显示弹窗。因此我不得不添加第二个`popover('show')`调用来将弹窗显示到页面中。

弹出窗口的内容包括[第十二章](https://github.com/luhuisicnu/The-Flask-Mega-Tutorial-zh/blob/master/docs/%E7%AC%AC%E5%8D%81%E4%BA%8C%E7%AB%A0%EF%BC%9A%E6%97%A5%E6%9C%9F%E5%92%8C%E6%97%B6%E9%97%B4.md)中通过Flask-Moment插件生成的“最后访问”日期。 [文档](https://github.com/miguelgrinberg/Flask-Moment#ajax-support)中提到，当通过Ajax添加新的Flask-Moment元素时，需要调用`flask_moment_render_all()`函数来适当地渲染这些元素。

现在剩下的就是完善鼠标移出事件处理程序上的删除弹出窗口逻辑。 如果用户将鼠标移出目标元素，该处理程序已经具有中止弹出操作的逻辑。 如果这些条件都不适用，那么这意味着弹出窗口当前显示并且用户正在离开target区域，所以在这种情况下，对目标元素的`popover('destroy')`调用将正确地执行移除和清理。

*app/templates/base.html*：销毁弹窗。

```
                function(event) {
                    // mouse out event handler
                    var elem = $(event.currentTarget);
                    if (timer) {
                        clearTimeout(timer);
                        timer = null;
                    }
                    else if (xhr) {
                        xhr.abort();
                        xhr = null;
                    }
                    else {
                        elem.popover('destroy');
                    }
                }
```

