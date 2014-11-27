# 第5章 模型 #

在第3章，我们讲述了用Django建造网站的基本途径：建立视图和URLConf。正如我们所阐述的，视图负责处理一些主观逻辑*some arbitrary logic*，然后返回响应结果*response*。作为例子之一，我们的主观逻辑是要计算当前的日期和时间。

在现代Web应用中，主观逻辑经常牵涉到与数据库的交互。数据库驱动网站*database-driven Web site*在后台连接数据库服务器，从中取出一些数据，然后在Web页面用漂亮的格式展示这些数据。这个网站也可能会向访问者提供修改数据库数据的方法。

许多复杂的网站都提供了以上两个功能的某种结合。例如Amazon.com就是一个数据库驱动站点的良好范例。本质上，每个产品页面都是数据库中数据以HTML格式进行的展现，而当你发表客户评论时，该评论被插入评论数据库中。

由于先天具备Python简单而强大的数据库查询执行方法，Django非常适合开发数据库驱动网站。本章深入介绍了该功能：Django数据库层。

（注意：尽管对Django数据库层的使用中并不特别强调这点，但是我们还是强烈建议您掌握一些数据库和SQL原理。对这些概念的介绍超越了本书的范围，但就算你是数据库方面的菜鸟，我们也建议你继续阅读。你也许能够跟上进度，并在上下文学习过程中掌握一些概念。）

在视图中进行数据库查询的笨方法

正如第3章详细介绍的那个在视图中输出HTML的笨方法（通过在视图里对文本直接硬编码HTML），在视图中也有笨方法可以从数据库中获取数据。很简单：用现有的任何Python类库执行一条SQL查询并对结果进行一些处理。

在本例的视图中，我们使用了`MySQLdb`类库（可以从 http://www.djangoproject.com/r/python-mysql/ 获得）来连接MySQL数据库，取回一些记录，将它们提供给模板以显示一个网页：【评论抱怨各种安装不顺利】【Python3.4用`mysql connector`代替mysqldb，亲测可以】

```Python
from django.shortcuts import render
import MySQLdb

def book_list(request):
    db = MySQLdb.connect(user='me', db='mydb', passwd='secret', host='localhost')
    cursor = db.cursor()
    cursor.execute('SELECT name FROM books ORDER BY name')
    names = [row[0] for row in cursor.fetchall()]
    db.close()
    return render(request, 'book_list.html', {'names': names})
```

这个方法可用，但很快一些问题将出现在你面前：

- 我们将数据库连接参数硬编码于代码之中。 理想情况下，这些参数应当保存在Django配置中。

- 我们不得不重复同样的代码：创建数据库连接、创建数据库游标、执行某个语句、然后关闭数据库。理想情况下，我们所需要应该只是指定所需的结果。

- 它把我们栓死在MySQL之上。如果过段时间，我们要从MySQL换到PostgreSQL，就不得不使用不同的数据库适配器（例如psycopg而不是MySQLdb），改变连接参数，根据SQL语句的类型可能还要修改SQL。理想情况下，应对所使用的数据库服务器进行抽象，这样一来只在一处修改即可变换数据库服务器。（如果你正在建立一个开源的Django应用程序来尽可能让更多人使用的话，这个特性是非常适当的。）

正如你所期待的，Django数据库层正是致力于解决这些问题。以下提前揭示了如何使用Django数据库API重写之前那个视图。

```Python
from django.shortcuts import render_to_response
from mysite.books.models import Book

def book_list(request):
    books = Book.objects.order_by('name')
    return render_to_response('book_list.html', {'books': books})
```

我们将在本章稍后的地方解释这段代码。目前而言，仅需对它有个大致的认识。

## MTV（或MVC）开发模式 ##

在钻研更多代码之前，让我们先花点时间考虑下Django数据驱动Web应用的总体设计。【思考者】

我们在前面章节提到过，Django的设计鼓励**松耦合**及对应用程序中不同部分的**严格分割**。遵循这个理念的话，要想修改应用的某部分而不影响其它部分就比较容易了。在视图函数中，我们已经讨论了通过模板系统把**业务逻辑**和**表现逻辑**分隔开的重要性。在数据库层中，我们对数据访问逻辑也应用了同样的理念。

把数据存取逻辑、业务逻辑和表现逻辑组合在一起的概念有时被称为软件架构的Model-View-Controller (MVC)模式。在这个模式中，Model代表数据存取层，View代表的是系统中选择显示什么和怎么显示的部分，Controller指的是系统中根据用户输入并视需要访问模型，以决定使用哪个视图的那部分。

* * *
为什么用缩写？
* * *

像MVC这样的明确定义模式的主要用于改善开发人员之间的沟通。比起告诉同事，“让我们采用抽象的数据存取方式，然后单独划分一层来显示数据，并且在中间加上一个控制它的层”，一个通用的说法会让你收益，你只需要说：“我们在这里使用MVC模式吧。”

Django紧紧地遵循这种MVC模式，可以称得上是一种MVC框架。以下是Django中M、V和C各自的含义：

- *M*，数据存取，由Django数据库层处理，本章要讲述的内容。

- *V*，选择哪些数据要显示以及怎样显示，由视图和模板处理。

- *C*，根据用户输入委派视图，由Django框架根据URLconf设置，对给定URL调用适当的Python函数。

由于*C*部分由框架自行处理，而Django里更关注的是模型*models*、模板*templates*和视图*views*，所以Django也被称为*MTV*框架。在MTV开发模式中：

- *M*代表模型，即数据存取层。该层处理与数据相关的所有事务：如何存取、如何验证有效性、包含哪些行为以及数据之间的关系等。

- *T*代表模板，即表现层。该层处理与表现相关的决定：如何在页面或其他类型文档中进行显示。

- *V*代表视图，即**业务逻辑层**。该层包含存取模型及调取恰当模板的相关逻辑。你可以把它看作模型与模板之间的桥梁。【python代码主要在view里面啊】

如果你熟悉其它的MVC Web开发框架，比方说Ruby on Rails，你可能会认为Django视图是控制器，而Django模板是视图。很不幸，这是对MVC不同诠释所引起的错误认识。在Django对MVC的诠释中，视图用来描述要展现给用户的数据；不是数据如何*how*展现，而是展现 哪些*which*数据。相比之下，Ruby on Rails及一些同类框架提倡控制器负责决定向用户展现哪些数据，而视图则仅决定如何*how*展现数据，而不是展现哪些*which*数据。

两种诠释中没有哪个更加正确一些。重要的是要理解底层概念。

## 数据库配置 ##

记住这些理念之后，让我们来开始Django数据库层的探索。首先，我们需要做些初始配置；我们需要告诉Django使用什么数据库以及如何连接数据库。

我们假定你已经完成了数据库服务器的安装和激活，并且已经在其中创建了数据库（例如，用`CREATE DATABASE`语句）。如果你使用SQLite，不需要这步安装，因为SQLite使用文件系统上的独立文件来存储数据。

像前面章节提到的`TEMPLATE_DIRS`一样，数据库配置也是在Django的配置文件里，缺省是`settings.py`。打开这个文件并查找数据库配置：

```Python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.', # Add 'postgresql_psycopg2', 'mysql', 'sqlite3' or 'oracle'.
        'NAME': '',                      # Or path to database file if using sqlite3.
        'USER': '',                      # Not used with sqlite3.
        'PASSWORD': '',                  # Not used with sqlite3.
        'HOST': '',                      # Set to empty string for localhost. Not used with sqlite3.
        'PORT': '',                      # Set to empty string for default. Not used with sqlite3.
    }
}
```

配置纲要如下：

- `DATABASE_ENGINE`告诉Django使用哪个数据库引擎。如果你在Django中使用数据库，`DATABASE_ENGINE`必须是表5-1中所列出的值。

    **表5-1 数据库引擎设置**

    | 设置                                   | 数据库     | 适配器                  | 
    | :------------------------------------- | :--------- | :---------------------- |
    | django.db.backends.postgresql_psycopg2 | PostgreSQL | `psycopg` version 2.x, http://www.djangoproject.com/r/python-pgsql/
    | django.db.backends.mysql               | MySQL      | `MySQLdb`, http://www.djangoproject.com/r/python-mysql/
    | django.db.backends.sqlite3             | SQLite     | 不需要
    | django.db.backends.oracle              | Oracle     | `cx_Oracle`, http://www.djangoproject.com/r/python-oracle/
    
    注意，无论选择使用哪个数据库服务器，都必须下载和安装对应的数据库适配器。访问表5-1中“适配器”一栏中的链接，可通过互联网免费获取这些适配器。如果你使用Linux，你的发布包管理系统会提供合适的包。比如说查找`python-postgresql`或者`python-psycopg`的软件包。

    配置示例：

    ```Python
    'ENGINE': 'django.db.backends.postgresql_psycopg2',
    ```

- `DATABASE_NAME`将数据库名称告知Django。例如：

    ```Python
    'NAME': 'mydb',
    ```

    如果使用SQLite，请对数据库文件指定完整的文件系统路径。例如：

    ```Python
    'NAME': '/home/django/mydata.db',
    ```

    在这个例子中，我们将SQLite数据库放在`/home/django`目录下，你可以任意选用最合适你的目录。

- `DATABASE_USER`告诉Django用哪个用户连接数据库。如果使用SQLite，留空即可。

- `DATABASE_PASSWORD`告诉Django连接用户的密码。SQLite用空密码即可。

- `DATABASE_HOST`告诉Django连接哪一台主机的数据库服务器。如果数据库与Django安装于同一台计算机（即本机），可将此项保留空白。如果你使用SQLite，此项留空。

    此处的MySQL是一个特例。如果使用的是MySQL且该项设置值由斜杠（`'/'`）开头，MySQL将通过Unix socket来连接指定的套接字，例如：

    ```Python
    DATABASE_HOST = '/var/run/mysql'
    ```

一旦在输入了那些设置并保存之后应当测试一下你的配置。 我们可以在`mysite`项目目录下执行上章所提到的`python manage.py shell`来进行测试。（我们上一章提到过在，`manager.py shell`命令是以正确Django配置启用Python交互解释器的一种方法。这个方法在这里是很有必要的，因为Django需要知道加载哪个配置文件来获取数据库连接信息。）

输入下面这些命令来测试你的数据库配置：

```Python
>>> from django.db import connection
>>> cursor = connection.cursor()
```

如果没有显示什么错误信息，那么你的数据库配置是正确的。否则，你就得查看错误信息来纠正错误。表5-2是一些常见错误。【下面已经更新至1.4的版本，相对DBv1新一点】【附上1.6的文档 https://docs.djangoproject.com/en/1.6/ref/databases/ 随时查看是否变化】

**表5-2 数据库配置错误信息**

| 错误信息 | 解决方法 | 
| :------- | :------- |
| You haven’t set the ENGINE setting yet. | 不要以空字符串配置`DATABASE_ENGINE`的值。使用表格5-1列出可用的值。 |
| Environment variable DJANGO_SETTINGS_MODULE is undefined. | 使用`python manager.py shell`命令启动交互解释器，不要以`python`命令直接启动交互解释器。 |
| Error loading _____ module: No module named _____. | 未安装合适的数据库适配器（例如, `psycopg`或`MySQLdb`）。Django并不自带适配器，所以你得自己下载安装。 |
| _____ isn’t an available database backend. | 把`ENGINE`配置成前面提到的合法的数据库引擎。也许是拼写错误？ |
| database _____ does not exist | 设置`NAME`指向存在的数据库，或者先在数据库客户端中执行合适的`CREATE DATABASE`语句创建数据库。 |
| role _____ does not exist | 设置`USER`指向存在的用户，或者先在数据库客户端中执创建用户。 |
| could not connect to server | 查看`HOST`和`PORT`是否已正确配置，并确认数据库服务器是否已正常运行。 |

## 第一个应用程序 ##

你现在已经确认数据库连接正常工作了，让我们来创建一个Django app，一个包含模型、视图和Django代码，并且形式为独立Python包的完整Django应用。

在这里要先解释一些术语，初学者可能会混淆它们。在第二章我们已经创建了*project*, 那么*project*和*app*之间到底有什么不同呢？它们的区别就是，一个是配置*configuration*，而另一个是代码*code*：

- project包含很多个Django app以及对它们的配置。

    技术上来讲，project的作用是提供配置文件，比方说哪里定义数据库连接信息, 安装的app列表，`TEMPLATE_DIRS`，等等。

- app是一套Django功能的集合，通常包括模型和视图，按Python的包结构的方式存在。

    例如，Django本身内建有一些app，例如注释系统和自动管理界面。app的一个关键点是它们是很容易移植到其他project和被多个project复用。

对于如何架构Django代码并没有快速成套的规则。如果你只是建造一个简单的Web站点，那么可能你只需要一个app就可以了。但如果是一个包含许多不相关的模块的复杂的网站，例如电子商务和社区之类的站点，那么你可能需要把这些模块划分成不同的app，以便以后复用。

不错，你**可以不用创建app**，这一点已经被我们之前编写的视图函数的例子证明了。在那些例子中，我们只是简单的创建了一个称为`views.py`的文件，编写了一些函数并在URLconf中设置了各个函数的映射。这些情况都不需要使用apps。【练习操作，不用创建app也行】

但是，系统对app有一个约定：如果你使用了Django的数据库层（模型*models*），你**必须**创建一个Django app。模型**必须**存放在apps中。因此，为了开始建造我们的模型，我们必须创建一个新的app。

在`mysite`项目文件下输入下面的命令来创建一个叫`books`的app：

```Shell
python manage.py startapp books
```


这个命令并没有输出什么，它只在`mysite`的目录里创建了一个`books`目录。让我们来看看这个目录的内容：【1.7版新增了`/migrations/`目录】

```Shell
books/
    __init__.py
    models.py
    tests.py
    views.py
```

这个目录包含了这个app的模型和视图。

使用你最喜欢的文本编辑器查看一下`models.py`和`views.py`文件的内容。它们都是空的，除了`models.py`里有一个`import`。这就是你Django app的基础。

## 在Python代码里定义模型 ##











