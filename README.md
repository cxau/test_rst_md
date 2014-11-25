# 第4章 模板 #

在前一章中，你可能已经注意到我们在例子视图中返回文本的方式有点特别。 也就是说，HTML 被直接硬编码在 Python 代码之中。

```python
def current_datetime(request):
    now = datetime.datetime.now()
    html = "<html><body>It is now %s.</body></html>" % now
    return HttpResponse(html)
```

尽管这种技术便于解释视图是如何工作的，但直接将HTML硬编码到你的视图里却并不是一个好主意。 让我们来看一下为什么：

- 对页面设计进行的任何改变都必须对 Python 代码进行相应的修改。站点设计的修改往往比底层 Python 代码的修改要频繁得多，因此如果可以在不进行 Python 代码修改的情况下变更设计，那将会方便得多。

- Python 代码编写和 HTML 设计是两项不同的工作，大多数专业的网站开发环境都将他们分配给不同的人员（甚至不同部门）来完成。设计者和 HTML/CSS 的编码人员不应该被要求去编辑 Python 的代码来完成他们的工作。

- 程序员编写 Python 代码和设计人员制作模板两项工作同时进行的效率是最高的，远胜于让一个人等待另一个人完成对某个既包含 Python 又包含 HTML 的文件的编辑工作。

基于这些原因，将页面的设计和Python的代码分离开会更干净简洁更容易维护。我们可以使用  Django 的模板系统 *Template System* 来实现这种模式，这就是本章要具体讨论的问题。

## 模板系统基本知识 ##

模板是一个文本，用于分离文档的表现形式和内容。 模板定义了占位符以及各种用于规范文档该如何显示的各部分基本逻辑（模板标签）。模板通常用于产生 HTML，但是 Django 的模板也能产生任何基于文本格式的文档。

让我们从一个简单的例子模板开始。该模板描述了一个向某个与公司签单人员致谢 HTML 页面。可将其视为一个格式信函：

```django
<html>
<head><title>Ordering notice</title></head>

<body>

<h1>Ordering notice</h1>

<p>Dear {{ person_name }},</p>

<p>Thanks for placing an order from {{ company }}. It's scheduled to
ship on {{ ship_date|date:"F j, Y" }}.</p>

<p>Here are the items you've ordered:</p>

<ul>
{% for item in item_list %}
    <li>{{ item }}</li>
{% endfor %}
</ul>

{% if ordered_warranty %}
    <p>Your warranty information will be included in the packaging.</p>
{% else %}
    <p>You didn't order a warranty, so you're on your own when
    the products inevitably stop working.</p>
{% endif %}

<p>Sincerely,<br />{{ company }}</p>

</body>
</html>
```

该模板是一段添加了些许变量和模板标签的基础  HTML。让我们逐步分析一下：

- 用两个大括号括起来的文字（如 
`{{ person_name }}` ）称为 变量 *variable*。这意味着“在此处插入指定变量的值。”（如何指定变量的值呢？稍后就会说明。）

- 被大括号和百分号包围的文本（如 `{% if ordered_warranty %}` ）是 模板标签 *template tag*。标签 *tag* 定义比较明确，即：标签通知模板系统去做某些事情。

       这个例子中的模板包含一个 `for` 标签（ `{% for item in item_list %}` ）和一个 `if` 标签（`{% if ordered_warranty %}` ）
	
       `for` 标签类似 Python 的 for 语句，可让你循环访问序列里的每一个项目。if 标签，正如你所料，是用来执行逻辑判断的。在这里，tag 标签检查 ordered_warranty 值是否为 `True`。如果是，模板系统将显示 `{% if ordered_warranty %}` 和 `{% else %}` 之间的内容；否则将显示 `{% else %}` 和 `{% endif %}` 之间的内容。注意 `{% else %}` 是可选的。

- 最后，这个模板的第二段中有一个关于 过滤器 *filter* 的例子，它是一种最便捷的转换变量输出格式的方式。 如这个例子中的 `{{ ship_date|date:"F j, Y" }}`，我们将变量 `ship_date` 传递给 `date` 过滤器，同时指定参数 `”F j,Y”`。过滤器 `date` 根据参数进行格式输出。 过滤器是用管道符(`|`)来调用的，具体可以参见 Unix 管道符。

每个 Django 模板含有很多内置的 tags 和 filters，我们将陆续进行学习. 附录F列出了很多的 tags 和 filters 的列表，熟悉这些列表对你来说是个好建议.你依然可以利用它创建自己的 filters 和 tags 。这些我们在第9章会讲到。

## 使用模板系统 ##

让我们深入研究模板系统，你将会明白它是如何工作的。但我们暂不打算将它与先前创建的视图结合在一起，因为我们现在的目的是了解它是如何独立工作的。 。 （换言之， 通常你会将模板和视图一起使用，但是我们只是想突出模板系统是一个Python库，你可以在任何地方使用它，而不仅仅是在Django视图中。）

在 Python 代码中使用 Django 模板的最基本方式如下：

1. 可以用原始的模板代码字符串创建一个 `Template` 对象。【语句不顺】

2. 调用模板 `Template` 对象的 `render()` 方法，并且传入一套变量context。它将返回一个基于模板的展现字符串，模板中的变量和标签会被context值替换。【不太懂】

在代码中，它长成这个样子：

	>>> from django import template
	>>> t = template.Template('My name is {{ name }}.')
	>>> c = template.Context({'name': 'Adrian'})
	>>> print t.render(c)
	My name is Adrian.
	>>> c = template.Context({'name': 'Fred'})
	>>> print t.render(c)
	My name is Fred.

下文将详细介绍每步的具体操作。

### 创建模板对象 ###

创建一个 `Template` 对象最简单的方法就是直接实例化它。 `Template` 类 *class* 就在 `django.template` 模块中，构造函数 *constructor*  接受一个参数，原始模板代码 *raw template code* 。 我们进入 Python 的交互解释器，来看看它在代码中是怎么运作的。

***
一个特殊的Python提示符
***

如果你曾经使用过Python，你一定好奇，为什么我们运行 `python manage.py shell` 而不是 `python` 。这两个命令都会启动交互解释器，但是`manage.py shell`命令有一个重要的不同：在启动解释器之前，它告诉Django使用哪个设置文件。Django框架的大部分子系统，包括模板系统，都依赖于配置文件，如果Django不知道使用哪个配置文件，这些系统将不能工作。

如果你好奇的话，这里解释一下它是怎样在幕后工作的。 Django搜索一个叫做`DJANGO_SETTINGS_MODULE`的环境变量，它应该位于`settings.py`设置文件中。例如，假设mysite在你的Python搜索路径中，那么`DJANGO_SETTINGS_MODULE`应该被设置为：`’mysite.settings’`。

当你运行命令：`python manage.py shell`，这个命令自动帮你设定`DJANGO_SETTINGS_MODULE`。 在当前的这些示例中，我们鼓励你使用` python manage.py shell`这个方法，这样可以免去你大费周章地去配置那些你不熟悉的环境变量。【啰嗦，过渡翻译了】

随着你越来越熟悉Django，你可能会偏向于废弃使用`manage.py shell`，而是在你的配置文件`.bash_profile`中手动添加 `DJANGO_SETTINGS_MODULE`这个环境变量。

让我们来了解一些模板系统的基本知识：

	>>> from django.template import Template
	>>> t = Template('My name is {{ name }}.')
	>>> print t

如果你还在交互解释器中，你会看到类似下面的东西：

	<django.template.Template object at 0xb7d5f24c>

那个`0xb7d5f24c`每次都会不一样，这没什么关系；这只是Python运行时 Template对象的ID。

当你创建一个`Template`对象，模板系统在内部编译这个模板到内部格式，并做优化，做好 渲染 *rendering* 的准备。如果你的模板语法有错误，那么在调用`Template()`时就会抛出`TemplateSyntaxError`异常：

	>>> from django.template import Template
	>>> t = Template('{% notatag %}')
	Traceback (most recent call last):
	  File "<stdin>", line 1, in ?
	  ...
	django.template.TemplateSyntaxError: Invalid block tag: 'notatag'

这里，“块标签” *block tag* 指向的是`{% notatag %}`，“块标签”与“模板标签”是同义的。

系统会在下面的情形抛出`TemplateSyntaxError`异常：

- 无效的标签 tags
- 无效的标签参数 arguments to valid tags
- 无效的过滤器 filters
- 无效的过滤器参数 arguments to valid filters
- 无效的模板语法 template syntax
- 未封闭的块标签（针对需要封闭的块标签）

渲染一个模板

一旦你创建一个`Template`对象，你可以用 context 来传递数据给它。 一个context是一系列变量和它们值的集合。【有一个 context小解 可以看看】

上下文 context 在Django里表现为`Context`类，在`django.template`模块里。 它的构造函数带有一个可选的参数：一个字典映射变量和它们的值。 调用`Template`对象的`render()`方法并传递context来填充模板：

	>>> from django.template import Context, Template
	>>> t = Template('My name is {{ name }}.')
	>>> c = Context({'name': 'Stephane'})
	>>> t.render(c)
	u'My name is Stephane.'

我们必须指出的一点是，`t.render(c)`返回的值是一个`Unicode对象`，不是普通的Python字符串【py3中如何呢？待研究】。 你可以通过字符串前的u来区分。 在整个Django框架中，使用的字符都是Unicode对象而不是普通的字符串。 如果你明白这样做给你带来了多大便利的话，尽可能地感激Django在幕后有条不紊地为你所做这这么多工作吧。 如果不明白你从中获益了什么，别担心。你只需要知道Django对Unicode的支持，将让你的应用程序轻松地处理各式各样的字符集，而不仅仅是基本的A-Z英文字符。

***
字典和Contexts
***

Python的字典数据类型就是关键字和它们值的一个映射。`Context`和字典很类似，不过`Context`还提供更多的功能，请看第九章。

变量名必须由英文字符开始 （A-Z或a-z）并可以包含数字字符、下划线和小数点。 （小数点在这里有特别的用途，稍后我们会讲到）变量是大小写敏感的。

下面是编写模板并渲染的例子：

	>>> from django.template import Template, Context
	>>> raw_template = """<p>Dear {{ person_name }},</p>
	...
	... <p>Thanks for placing an order from {{ company }}. It's scheduled to
	... ship on {{ ship_date|date:"F j, Y" }}.</p>
	...
	... {% if ordered_warranty %}
	... <p>Your warranty information will be included in the packaging.</p>
	... {% else %}
	... <p>You didn't order a warranty, so you're on your own when
	... the products inevitably stop working.</p>
	... {% endif %}
	...
	... <p>Sincerely,<br />{{ company }}</p>"""
	>>> t = Template(raw_template)
	>>> import datetime
	>>> c = Context({'person_name': 'John Smith',
	...     'company': 'Outdoor Equipment',
	...     'ship_date': datetime.date(2009, 4, 2),
	...     'ordered_warranty': False})
	>>> t.render(c)
	u"<p>Dear John Smith,</p>\n\n<p>Thanks for placing an order from Outdoor
	Equipment. It's scheduled to\nship on April 2, 2009.</p>\n\n\n<p>You
	didn't order a warranty, so you're on your own when\nthe products
	inevitably stop working.</p>\n\n\n<p>Sincerely,<br />Outdoor Equipment
	</p>"

让我们逐步来分析下这段代码：

- 首先我们导入`Template`和`Context`类 ，它们都在模块`django.template`里。
- 我们把模板的原始文本*raw text*保存到变量`raw_template`中。注意到我们使用了三个引号来标识这些文本，因为这样可以包含多行。
- 接下来，我们创建了一个模板对象`t`，把`raw_template`作为 `Template`类构造函数的参数。
- 我们从Python的标准库导入`datetime`模块，以后我们将会使用它。
- 然后，我们创建一个 Context 对象`c`。 Context构造的参数是Python**字典数据类型**。在这里，我们指定参数`person_name`的值是 `'John Smith'`, 参数`company`的值为`'Outdoor Equipment'`，等等。
- 最后，我们在模板对象上调用`render()`方法，传递context参数给它。 这是返回渲染后的模板的方法，它会替换模板变量为真实的值和执行块标签。
- 注意，warranty paragraph显示是因为 ordered_warranty 的值为 True . 注意时间的显示， April 2, 2009 , 它是按 'F j, Y' 格式显示的。
- 如果你是Python初学者，你可能在想为什么输出里有回车换行的字符('\n' )而不是 显示回车换行？因为这是Python交互解释器的缘故：调用 t.render(c)返回字符串，解释器缺省显示这些字符串的 真实内容呈现 ，而不是打印这个变量的值。 要显示换行而不是 '\n' ，使用 print 语句： print t.render(c) 。

这就是使用Django模板系统的基本规则：写模板，创建`Template`对象，创建`Context`，调用`render()`方法。

### 多个上下文，同一个模板 ###

一旦有了`Template`对象，你就可以通过它渲染多个context，例如：

	>>> from django.template import Template, Context
	>>> t = Template('Hello, {{ name }}')
	>>> print t.render(Context({'name': 'John'}))
	Hello, John
	>>> print t.render(Context({'name': 'Julie'}))
	Hello, Julie
	>>> print t.render(Context({'name': 'Pat'}))
	Hello, Pat

无论何时我们都可以像这样使用同一模板源渲染多个context，只进行**一次**`Template`对象的创建，然后多次调用`render()`方法渲染，这样会更高效：

	# Bad
	for name in ('John', 'Julie', 'Pat'):
	    t = Template('Hello, {{ name }}')
	    print t.render(Context({'name': name}))
	
	# Good
	t = Template('Hello, {{ name }}')
	for name in ('John', 'Julie', 'Pat'):
	    print t.render(Context({'name': name}))

Django的模板解析相当快捷。大部分的解析工作都是在后台通过对简短正则表达式一次性调用来完成。这和基于XML的模板引擎形成鲜明对比，那些引擎承担了XML解析器的开销，且往往比Django模板渲染引擎要慢上几个数量级。【没有jinja2牛逼】

### Context变量的查找 ###

在到目前为止的例子中，我们通过 context 传递的简单参数值主要是字符串，还有一个`datetime.date`例子。 然而，模板系统能够非常简洁地处理更加复杂的数据结构，例如列表*list*、字典*dictionary*和自定义的对象。

在Django模板中遍历复杂数据结构的关键是句点字符 (`.`)。

最好是用几个例子来说明一下。比如，假设你要向模板传递一个Python字典。要通过**字典键**访问该字典的值，可使用一个句点：

	>>> from django.template import Template, Context
	>>> person = {'name': 'Sally', 'age': '43'}
	>>> t = Template('{{ person.name }} is {{ person.age }} years old.')
	>>> c = Context({'person': person})
	>>> t.render(c)
	u'Sally is 43 years old.'

同样，也可以通过句点来访问对象的属性。比方说， Python的`datetime.date`对象有`year`、`month`和`day`几个属性，你同样可以在模板中使用句点来访问这些属性*attributes*：

	>>> from django.template import Template, Context
	>>> import datetime
	>>> d = datetime.date(1993, 5, 2)
	>>> d.year
	1993
	>>> d.month
	5
	>>> d.day
	2
	>>> t = Template('The month is {{ date.month }} and the year is {{ date.year }}.')
	>>> c = Context({'date': d})
	>>> t.render(c)
	u'The month is 5 and the year is 1993.'

这个例子使用了一个自定义类*custom class*，演示了通过实例变量加点(dot)来访问它的属性，这个方法适用于任意的对象：

	>>> from django.template import Template, Context
	>>> class Person(object):
	...     def __init__(self, first_name, last_name):
	...         self.first_name, self.last_name = first_name, last_name
	>>> t = Template('Hello, {{ person.first_name }} {{ person.last_name }}.')
	>>> c = Context({'person': Person('John', 'Smith')})
	>>> t.render(c)
	u'Hello, John Smith.'

点语法也可以用来引用对象的**方法***methods*。 例如，每个Python字符串都有`upper()`和`isdigit()`方法，你在模板中可以使用同样的句点语法来调用它们：

	>>> from django.template import Template, Context
	>>> t = Template('{{ var }} -- {{ var.upper }} -- {{ var.isdigit }}')
	>>> t.render(Context({'var': 'hello'}))
	u'hello -- HELLO -- False'
	>>> t.render(Context({'var': '123'}))
	u'123 -- 123 -- True'

注意这里调用方法时**并没有**使用圆括号,而且也无法给该方法传递参数；你只能调用**不需参数**的方法。（我们将在本章稍后部分解释该哲学。）

最后，句点也可用于访问列表索引*list indices*，例如：

	>>> from django.template import Template, Context
	>>> t = Template('Item 2 is {{ items.2 }}.')
	>>> c = Context({'items': ['apples', 'bananas', 'carrots']})
	>>> t.render(c)
	u'Item 2 is carrots.'

Django不允许使用负数列表索引。像`{{ items.-1 }}`这样的模板变量将会引发`TemplateSyntaxError`。

***
Python列表类型
***

一点提示：Python的列表是从0开始索引。第一项的索引是0，第二项的是1，依此类推。

句点查找的使用，可归纳如下，当模板系统在一个变量名中遇到一个点的时候，它会按照下面的顺序进行查找：

- 字典查找　　　如`foo["bar"]`
- 属性查找　　　如`foo.bar`
- 方法调用　　　如`foo.bar()`
- 列表索引查找　如`foo[2]`

系统会默认使用第一个实用的查找方式。按短路逻辑`short-circuit logic`。

句点查找可以多级深度嵌套。例如在下面这个例子中 `{{person.name.upper}}`会转换成字典类型查找（`person['name']`)然后是方法调用（`upper()`):

	>>> from django.template import Template, Context
	>>> person = {'name': 'Sally', 'age': '43'}
	>>> t = Template('{{ person.name.upper }} is {{ person.age }} years old.')
	>>> c = Context({'person': person})
	>>> t.render(c)
	u'SALLY is 43 years old.'

### 方法调用行为 ###

方法调用比其他类型的查找略为复杂一点。以下是一些注意事项：

- 在方法查找过程中，如果某方法抛出一个异常，除非该异常有一个`silent_variable_failure`属性并且它的值为`True`，否则的话它将被传播*propagated*。如果异常确实有`silent_variable_failure`属性，那么模板里的指定变量会被置为空字符串*empty string*，比如:

		>>> t = Template("My name is {{ person.first_name }}.")
		>>> class PersonClass3:
		...     def first_name(self):
		...         raise AssertionError, "foo"
		>>> p = PersonClass3()
		>>> t.render(Context({"person": p}))
		Traceback (most recent call last):
		...
		AssertionError: foo
		
		>>> class SilentAssertionError(AssertionError):
		...     silent_variable_failure = True
		>>> class PersonClass4:
		...     def first_name(self):
		...         raise SilentAssertionError
		>>> p = PersonClass4()
		>>> t.render(Context({"person": p}))
		u'My name is .'

- 仅在方法无需传入参数时，其调用才有效。 否则，系统将会转移到下一个查找类型（列表索引查找）。

- 显然，有些方法是有副作用的，允许模板系统访问它们，在好的情况下可能只是干件蠢事，坏的情况下甚至会引发安全漏洞*security hole*。

	例如，你有一个 `BankAccount` 对象，它有一个 `delete()` 方法。 如果某个模板中包含了像 `{{ account.delete }}`这样的标签，其中`account`又是`BankAccount`的一个实例，请注意在渲染这个模板时，该`account`对象将被删除！

	要防止这样的事情发生，必须设置该方法的 `alters_data` 函数属性：

		def delete(self):
		    # Delete the account
		delete.alters_data = True

	模板系统不会执行任何以该方式进行标记的方法。接上面的例子，如果模板文件里包含了 `{{ account.delete }}` ，对象又具有 `delete()`方法，而且`delete()`有`alters_data=True`这个属性，那么在模板载入时，`delete()`方法将不会被执行。它将静静地错误退出。

### Django如何处理无效变量 ###

默认情况下，如果一个变量不存在，模板系统会把它展示为**空字符串**，不做任何事情来表示失败。 例如：

	>>> from django.template import Template, Context
	>>> t = Template('Your name is {{ name }}.')
	>>> t.render(Context())
	u'Your name is .'
	>>> t.render(Context({'var': 'hello'}))
	u'Your name is .'
	>>> t.render(Context({'NAME': 'hello'}))
	u'Your name is .'
	>>> t.render(Context({'Name': 'hello'}))
	u'Your name is .'

系统静悄悄地表示失败，而不是抛出一个异常，因为这通常是人为错误造成的。这种情况下，因为变量名有错误的状况或名称，所有的查询都会失败。 现实世界中，对于一个web站点来说，如果仅仅因为一个小的模板语法错误而造成无法访问，这是不能接受的。

### 与Context对象一同玩耍 ###

多数时间，你可以通过传递一个完全填充*full populated*的字典给 `Context()` 来初始化 `Context`对象。 但是初始化以后，你也可以使用标准的Python字典句法*syntax*向`Context`对象添加或者删除条目：

	>>> from django.template import Context
	>>> c = Context({"foo": "bar"})
	>>> c['foo']
	'bar'
	>>> del c['foo']
	>>> c['foo']
	Traceback (most recent call last):
	  ...
	KeyError: 'foo'
	>>> c['newvariable'] = 'hello'
	>>> c['newvariable']
	'hello'

## 基本的模板标签和过滤器 ##

如前所述，模板系统带有内置的标签*tags*和过滤器*filters*。下面的章节提供了一个多数通用标签和过滤器的简要说明。

### 标签 ###

#### if/else ####

`{% if %}`标签检查一个变量，如果这个变量为真（即，变量存在，非空，不是布尔值假），系统会显示在`{% if %}`和`{% endif %}`之间的任何内容，例如：

	{% if today_is_weekend %}
	    <p>Welcome to the weekend!</p>
	{% endif %}

`{% else %}`标签是可选的：

	{% if today_is_weekend %}
	    <p>Welcome to the weekend!</p>
	{% else %}
	    <p>Get back to work.</p>
	{% endif %}

***
Python的“真值”
***

在Python和Django模板系统中，以下这些对象相当于布尔值的False：

- 空列表　　[]
- 空元组　　()
- 空字典　　{} 
- 空字符串　''
- 零值　　　0
- 特殊对象　None
- 对象　　　False
- 自定义对象，定义了自己的布尔值上下文行为（属于Python高级用法）

除以上之外的所有东西都视为`True`。

`{% if %}`标签接受`and`，`or`或者`not`关键字来对多个变量做判断，或者对变量取反，例如：

	{% if athlete_list and coach_list %}
	    Both athletes and coaches are available.
	{% endif %}
	
	{% if not athlete_list %}
	    There are no athletes.
	{% endif %}
	
	{% if athlete_list or coach_list %}
	    There are some athletes or some coaches.
	{% endif %}
	
	{% if not athlete_list or coach_list %}
	    There are no athletes or there are some coaches.
	{% endif %}
	
	{% if athlete_list and not coach_list %}
	    There are some athletes and absolutely no coaches.
	{% endif %}

{% if %}标签不允许在同一个标签中同时使用`and` 和`or`，因为逻辑上可能模糊的。例如，下面的代码是非法的：

	{% if athlete_list and coach_list or cheerleader_list %}

系统不支持用圆括号来处理比较操作的顺序。如果你确实需要用到圆括号来组合表达你的逻辑式，考虑将它移到模板之外处理，然后以模板变量的形式传入结果吧。或者，仅仅用嵌套的`{% if %}`标签替换吧，就像这样：

	{% if athlete_list %}
	    {% if coach_list or cheerleader_list %}
	        We have athletes, and either coaches or cheerleaders!
	    {% endif %}
	{% endif %}

多次使用同一个逻辑操作符是没有问题的，但是我们不能把不同的操作符组合起来。例如，这是合法的：

	{% if athlete_list or coach_list or parent_list or teacher_list %}

Django模板没有`{% elif %}`标签，请使用嵌套的`{% if %}`标签来达成同样的效果：

	{% if athlete_list %}
	    <p>Here are the athletes: {{ athlete_list }}.</p>
	{% else %}
	    <p>No athletes are available.</p>
	    {% if coach_list %}
	        <p>Here are the coaches: {{ coach_list }}.</p>
	    {% endif %}
	{% endif %}

一定要用`{% endif %}`关闭每一个`{% if %}`标签。

#### for ####

`{% for %}`允许我们在一个序列上迭代。与Python的`for`语句的情形类似，循环语法是`for X in Y`，`Y`是要迭代的序列，而`X`则是在每一个特定的循环中使用的变量名称。每一次循环中，模板系统会渲染在`{% for %}`和`{% endfor %}`之间的所有内容。

例如，给定一个运动员列表`athlete_list`变量，我们可以使用下面的代码来显示这个列表：

	<ul>
	{% for athlete in athlete_list %}
	    <li>{{ athlete.name }}</li>
	{% endfor %}
	</ul>

在标签末尾添加一个`reversed`参数，使得该列表被反向迭代：

	{% for athlete in athlete_list reversed %}
	...
	{% endfor %}

可以嵌套使用`{% for %}`标签：【重要】

	{% for athlete in athlete_list %}
	    <h1>{{ athlete.name }}</h1>
	    <ul>
	    {% for sport in athlete.sports_played %}
	        <li>{{ sport }}</li>
	    {% endfor %}
	    </ul>
	{% endfor %}

在执行循环之前先检测列表的大小是一个通常的做法，当**列表为空**时输出一些特别的提示：【重要】

	{% if athlete_list %}
	    {% for athlete in athlete_list %}
	        <p>{{ athlete.name }}</p>
	    {% endfor %}
	{% else %}
	    <p>There are no athletes. Only computer programmers.</p>
	{% endif %}

因为这种做法十分常见，所以`for`标签支持一个可选的`{% empty %}`分句，通过它我们可以定义当列表为空时的输出内容。下面的例子与之前那个等价：

	{% for athlete in athlete_list %}
	    <p>{{ athlete.name }}</p>
	{% empty %}
	    <p>There are no athletes. Only computer programmers.</p>
	{% endfor %}

Django模板**不支持**退出循环操作。如果我们想退出循环，可以改变正在迭代的变量，让其仅仅包含需要迭代的项目。 同理，Django也不支持continue语句，我们无法让当前迭代操作跳回到循环头部。（请参看本章稍后的**设计哲学和限制**小节，了解一下决定这个设计的背后原因）

在每个`{% for %}`循环里有一个称为`forloop`的模板变量。这个变量有一些提示循环进度信息的属性：【重要】

- `forloop.counter`总是一个表示当前循环的执行次数的整数计数器。这个计数器是从1开始的，所以在第一次循环时`forloop.counter`将会被设置为`1`。请看例子：

		{% for item in todo_list %}
		    <p>{{ forloop.counter }}: {{ item }}</p>
		{% endfor %}

- `forloop.counter0`类似于`forloop.counter`，但是它是从0计数的。第一次执行循环时这个变量会被设置为`0`。

- `forloop.revcounter`是表示循环中剩余项的整型变量。在循环初次执行时`forloop.revcounter`将被设置为序列中项的总数。最后一次循环执行中，这个变量将被置1。

- `forloop.revcounter0`类似于`forloop.revcounter`，但它以0做为结束索引。 在第一次执行循环时，该变量会被置为序列的项的个数减1。最后一次循环执行中，这个变量将被置`0`。

- `forloop.first`是一个布尔值，如果该迭代是第一次执行，那么它被置为`True`。在下面的特殊案例中这个变量是很有用的：【把第一个设置成active？】

		{% for object in objects %}
		    {% if forloop.first %}<li class="first">{% else %}<li>{% endif %}
		    {{ object }}
		    </li>
		{% endfor %}

- `forloop.last`是一个布尔值，在最后一次执行循环时被置为True。一个常见用法是在一系列的链接之间放置管道符（|）：【if嵌套在for内部判断这个变量的属性】

		{% for link in links %}{{ link }}{% if not forloop.last %} | {% endif %}{% endfor %}

	上面的模板可能会产生如下的结果：

		Link1 | Link2 | Link3 | Link4

	另一个常见的用途是为列表中的每个单词的加上逗号：

		Favorite places:
		{% for p in places %}{{ p }}{% if not forloop.last %}, {% endif %}{% endfor %}

- `forloop.parentloop`是一个指向当前循环的上一级循环的`forloop`对象的引用（在嵌套循环的情况下）。 例子在此：

		```django
		{% for country in countries %}
		    <table>
		    {% for city in country.city_list %}
		        <tr>
		        <td>Country #{{ forloop.parentloop.counter }}</td>
		        <td>City #{{ forloop.counter }}</td>
		        <td>{{ city }}</td>
		        </tr>
		    {% endfor %}
		    </table>
		{% endfor %}

`forloop`魔法变量，仅能在循环中使用。在模板解析器碰到`{% endfor %}`标签后，`forloop`就消失了。

***
Context和forloop变量
***

在`{% for %}`块中，已存在的变量会被移除，以避免`forloop`魔法变量被覆盖。Django会把这个变量移动到`forloop.parentloop`中。通常我们不用担心这个问题，但是一旦我们在模板中定义了`forloop`这个变量（当然我们反对这样做），在`{% for %}`块中它会在`forloop.parentloop`被重新命名。

#### ifequal/ifnotequal ####







