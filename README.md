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

## 用Python定义数据模型 ##

我们早些时候谈到。MTV里的M代表模型。Django模型是用Python代码形式表述的数据在数据库中的定义。对数据层来说它等同于`CREATE TABLE`语句，只不过执行的是Python代码而不是SQL，而且还包含了比数据库字段定义更多的含义。Django用模型在后台执行SQL代码并把结果用Python的数据结构来描述。Django也使用模型来呈现SQL无法处理的高级概念*higher-level concepts*。

如果你对数据库很熟悉，你可能马上就会想到，不用SQL，而用Python定义数据模型是不是有点多余？Django这样做是有下面几个原因的：

- 自省*Introspection*（运行时自动识别数据库）会导致大量的系统开销*overhead*，而且有数据完整性问题*imperfect*。为了提供方便的数据访问API，Django需要以某种方式*somehow*知道数据库层内部信息。对于这个问题，有两种实现方式。第一种方式是用Python明确地定义数据模型，第二种方式是通过**自省**来**自动侦测**识别数据模型。

第二种方式看起来更清晰，因为数据表信息只存放在一个地方——数据库里，但是会带来一些问题。首先，运行时扫描数据库会带来大量的系统开销。如果每个请求都要扫描数据库的表结构，或者即便是服务启动时*initialized*做一次都是会带来不能接受的系统开销。（有人认为这个程度的系统过载是可以接受的，而Django开发者的目标是尽可能地降低框架的系统开销）。第二，某些数据库，尤其是老版本的MySQL,并未完整存储那些精确的自省元数据。【也就是说，第二种方法不好】

- 编写Python代码是非常有趣的，保持用Python的方式思考会避免你的大脑在不同领域来回切换。尽可能的保持在单一的编程环境/思想状态下可以帮助你提高生产率。不得不去重复写SQL，再写Python代码，再写SQL，…，会让你头都要裂了。【不可能的，不可能避免去学js html css，单独语言只有美国人才会习惯，21世纪谁不会几门外语】

- 把数据模型用代码的方式表述来让你可以容易对它们进行版本控制。 这样，你可以很容易了解数据结构*data layouts*的变动情况。

- SQL只能描述特定类型的数据字段。例如，大多数数据库都没有专用的字段类型来描述Email地址、URL。而用Django的模型可以做到这一点。好处就是高级的数据类型带来更高的效率和更好的代码复用。【抽象，细化，新的模型。嗯】

- SQL还有在不同数据库平台的兼容性问题。发布Web应用的时候，使用Python模块描述数据库结构信息可以避免为MySQL，PostgreSQL，SQLite编写不同的`CREATE TABLE`。【这个应该是第2重要，居然放最后】

当然，这个方法也有一个缺点，就是Python代码和数据库表的同步问题。如果你修改了一个Django模型，你要自己来修改数据库来保证和模型同步。我们将在稍后讲解解决这个问题的几种策略。【目前1.7已经自带migration工具】

最后，我们要提醒你Django提供了实用工具来从现有的数据库表中自动扫描生成模型。这对已有的数据库来说是非常快捷有用的。我们将在第18章中对此进行讨论。【将已有外部数据库，迁移、转化成新的Django数据模型，sqlalchemy也有类似的工具】

## 第一个模型 ##

在本章和后续章节里，我们把注意力放在一个基本的“书籍/作者/出版商”*book/author/publisher*的数据库结构上。我们这样做是因为，这是一个众所周知的例子，很多SQL有关的书籍也常用这个举例。你现在看的这本书也是由作者创作再由出版商出版的哦！【貌似是好例子，能讲一对一，多对多，外键之类的东西】

我们来假定下面的这些概念、字段和关系：

- 一个作者有姓，有名及email地址。

- 出版社有名称，地址，所在城市、省，国家，网站。

- 书籍有书名和出版日期。一本书，有一个或多个作者（书与作者是多对多*many-to-many*关系），一本书只有一个出版社（书与出版社是一对多*one-to-many*关系，也称外键`foreign key`）。【一本书可以有几个作者，一个作者也可以有几本书，所以是*many-to-many*；一个出版社可以有很多书，但一本已经出版了的书只可以有一个出版社，所以是*one-to-many*】

第一步是用Python代码来描述它们。打开由`startapp`命令创建的`models.py`并输入下面的内容：

```Python
from django.db import models

class Publisher(models.Model):
    name = models.CharField(max_length=30)
    address = models.CharField(max_length=50)
    city = models.CharField(max_length=60)
    state_province = models.CharField(max_length=30)
    country = models.CharField(max_length=50)
    website = models.URLField()

class Author(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=40)
    email = models.EmailField()

class Book(models.Model):
    title = models.CharField(max_length=100)
    authors = models.ManyToManyField(Author)
    publisher = models.ForeignKey(Publisher)
    publication_date = models.DateField()
```

让我们来快速讲解一下这些代码的含义。首先要注意的事是每个数据模型都是`django.db.models.Model`的子类。它的父类`Model`包含了所有必要的和数据库交互的方法，并提供了一个简洁漂亮的定义数据库字段的语法。信不信由你，这些就是我们需要编写的通过Django存取基本数据的所有代码。

每个模型相当于单个数据库表，每个属性也是这个表中的一个字段。属性名就是字段名，它的类型（例如`CharField`）相当于数据库的字段类型 （例如`varchar`）。例如，`Publisher`模块等同于下面这张表（用PostgreSQL的`CREATE TABLE`语法描述）：【自动生成的表名是app名称`books`和模型的小写名称`publisher`，`book`，`author`的组合】

```SQL
CREATE TABLE "books_publisher" (
    "id" serial NOT NULL PRIMARY KEY,
    "name" varchar(30) NOT NULL,
    "address" varchar(50) NOT NULL,
    "city" varchar(60) NOT NULL,
    "state_province" varchar(30) NOT NULL,
    "country" varchar(50) NOT NULL,
    "website" varchar(200) NOT NULL
);
```

事实上，正如过一会儿我们所要展示的，Django可以自动生成这些`CREATE TABLE`语句。

“每个数据库表对应一个类”这条规则的**例外情况**是多对多关系。在我们的范例模型中，`Book`有一个`ManyToManyField`叫做`authors`。该字段表明一本书有一个或多个作者，但`Book`数据表却并没有`authors`字段。相反，Django创建了一个额外的表，一个多对多连接表*many-to-many “join table”*，来处理书和作者之间的映射关系。【数据库里多了一个表（四个表）名为：`book_authors`，这就是那个“额外的表”，是因为“多对多”字段的原因创建的。】

请查看附录B了解所有的字段类型和模型语法选项。

最后需要注意的是，我们并没有显式地为这些模型定义任何主键。除非你单独指明，否则Django会自动为每个模型生成一个自增长的整数主键字段`id`。每个Django模型都要求有单独的主键。【这里如何单独指明一个自增长主键】

## 安装模型 ##

完成这些代码之后，现在让我们来在数据库中创建这些表。要完成该项工作，第一步是在Django项目中激活*activate*这些模型。将`books app`添加到配置文件的已安装应用列表中即可完成此步骤。

再次编辑`settings.py`文件，找到`INSTALLED_APPS`设置。`INSTALLED_APPS`告诉Django项目哪些app处于激活状态。缺省情况下如下所示：

```Python
# Django 1.6.8
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
)
```

把这些设置前面加*#*临时注释起来。（这几个app是经常使用到的，我们将在后续章节里讨论如何使用它们）。同时，注释掉`MIDDLEWARE_CLASSES`的默认设置条目，因为这些条目是依赖于刚才我们刚在`INSTALLED_APPS`注释掉的apps。然后，添加`books`到`INSTALLED_APPS`的末尾，此时设置的内容看起来应该是这样的：

```Python
# Django 1.6.8
INSTALLED_APPS = (
    # 'django.contrib.admin',
    # 'django.contrib.auth',
    # 'django.contrib.contenttypes',
    # 'django.contrib.sessions',
    # 'django.contrib.messages',
    # 'django.contrib.staticfiles',
    'books',
)

MIDDLEWARE_CLASSES = (
    # 'django.contrib.sessions.middleware.SessionMiddleware',
    # 'django.middleware.common.CommonMiddleware',
    # 'django.middleware.csrf.CsrfViewMiddleware',
    # 'django.contrib.auth.middleware.AuthenticationMiddleware',
    # 'django.contrib.messages.middleware.MessageMiddleware',
    # 'django.middleware.clickjacking.XFrameOptionsMiddleware',
)
```

（就像我们在上一章设置`TEMPLATE_DIRS`所提到的逗号，同样在`INSTALLED_APPS`的末尾也需添加一个逗号，因为这是个单元素的元组。 另外，本书的作者喜欢在元组的每一个*every*元素后面都加一个逗号，不管它是不是只有一个元素。这是为了避免忘了加逗号，而且也没什么坏处。）【我也喜欢这样】

'books'指示我们正在编写的名为`books`的app。`INSTALLED_APPS`中的每个app都使用Python的路径描述，包*package*的路径，用小数点（`.`）间隔。

现在我们可以创建数据库表了。首先，用下面的命令验证模型的有效性：【Django1.6显示"validate" has been deprecated in favor of "check". 貌似validate已经改成check了。】

```Shell
python manage.py validate
```

`validate`命令检查你的模型的语法和逻辑是否正确。如果一切正常，你会看到`0 errors found`消息。如果出错，请检查你输入的模型代码。错误输出会给出非常有用的错误信息来帮助你修正你的模型。

一旦你觉得你的模型可能有问题，运行`python manage.py validate`。它可以帮助你捕获一些常见的模型定义错误。

模型确认没问题了，运行下面的命令来生成`CREATE TABLE`语句（如果你使用的是Unix，那么可以启用语法高亮）：

```Shell
python manage.py sqlall books
```

在这个命令行中，`books`是app的名称。和你运行`manage.py startapp`中的一样。执行之后，输出如下：

```SQL
BEGIN;
CREATE TABLE "books_publisher" (
    "id" serial NOT NULL PRIMARY KEY,
    "name" varchar(30) NOT NULL,
    "address" varchar(50) NOT NULL,
    "city" varchar(60) NOT NULL,
    "state_province" varchar(30) NOT NULL,
    "country" varchar(50) NOT NULL,
    "website" varchar(200) NOT NULL
)
;
CREATE TABLE "books_author" (
    "id" serial NOT NULL PRIMARY KEY,
    "first_name" varchar(30) NOT NULL,
    "last_name" varchar(40) NOT NULL,
    "email" varchar(75) NOT NULL
)
;
CREATE TABLE "books_book" (
    "id" serial NOT NULL PRIMARY KEY,
    "title" varchar(100) NOT NULL,
    "publisher_id" integer NOT NULL REFERENCES "books_publisher" ("id") DEFERRABLE INITIALLY DEFERRED,
    "publication_date" date NOT NULL
)
;
CREATE TABLE "books_book_authors" (
    "id" serial NOT NULL PRIMARY KEY,
    "book_id" integer NOT NULL REFERENCES "books_book" ("id") DEFERRABLE INITIALLY DEFERRED,
    "author_id" integer NOT NULL REFERENCES "books_author" ("id") DEFERRABLE INITIALLY DEFERRED,
    UNIQUE ("book_id", "author_id")
)
;
CREATE INDEX "books_book_publisher_id" ON "books_book" ("publisher_id");
COMMIT;
```

注意下面几点：

- 自动生成的表名是app名称（`books`）和模型的小写名称（`publisher`，`book`，`author`）的组合。你可以参考附录B重写这个规则。

- 我们前面已经提到，Django为每个表格自动添加了一个`id`主键*primary key*，你可以重新设置它。

- 按约定，Django添加`"_id"`后缀到外键字段名。你猜对了，这个同样是可以自定义的。

- 外键是用`REFERENCES`语句明确定义的。

- 这些`CREATE TABLE`语句会根据你的数据库而作调整，这样象数据库特定的一些字段例如：（MySQL），`auto_increment`（PostgreSQL），`serial`（SQLite），都会自动生成。`integer primary key`同样的，字段名称也是自动处理（例如单引号还好是双引号）。例子中的输出是基于PostgreSQL语法的。

`sqlall`命令并没有在数据库中真正创建数据表，只是把SQL语句段打印出来，这样你可以看到Django究竟会做些什么。如果你想这么做的话，你可以把那些SQL语句复制到你的数据库客户端执行，或者通过Unix管道直接进行操作（例如，`python manager.py sqlall books | psql mydb`）。不过，Django提供了一种更为简易的提交SQL语句至数据库的方法：`syncdb`命令：【1.7版已将`syncdb`改为`migrate`，syncdb将在未来的版本(1.9)中移除】

```Shell
python manage.py syncdb
```

执行这个命令后，将看到类似以下的内容：

```Shell
Creating table books_publisher
Creating table books_author
Creating table books_book
Installing index for books.Book model
```

`syncdb`命令是同步你的模型到数据库的一个简单方法。它会根据`INSTALLED_APPS`里设置的app来检查数据库，如果表不存在，它就会创建表。需要注意的是，`syncdb`并不能*not*将模型的修改或删除同步到数据库；如果你修改或删除了一个模型，并想把它提交到数据库，`syncdb`并不会做出任何处理。（更多内容请查看本章最后的“修改数据库的架构”一段。）【在本章的最后，并没有“修改数据库的架构”这部分内容】【老版本DJB说是在第10章讲】【south组件，和1.7中的migrate功能可以搞定这个问题】

如果你再次运行`python manage.py syncdb`，什么也没发生，因为你没有添加新的模型或者添加新的app。因此，运行`python manage.py syncdb`总是安全的，因为它不会重复执行SQL语句。

如果你有兴趣，花点时间用你的SQL客户端登录进数据库服务器看看刚才Django创建的数据表。你可以手动启动命令行客户端（例如，执行PostgreSQL的`psql`命令），也可以执行`python manage.py dbshell`，这个命令将依据`DATABASE_SERVER`的里设置自动检测使用哪种命令行客户端。是两种方法中，后者（`manage.py dbshell`）总是较为方便。

## 基本数据访问 ##

一旦你创建了模型，Django自动为这些模型提供了高级的Python API。 运行`python manage.py shell`并输入下面的内容试试看：【注意import路径】

```Python
>>> from books.models import Publisher
>>> p1 = Publisher(name='Apress', address='2855 Telegraph Avenue',
...     city='Berkeley', state_province='CA', country='U.S.A.',
...     website='http://www.apress.com/')
>>> p1.save()
>>> p2 = Publisher(name="O'Reilly", address='10 Fawcett St.',
...     city='Cambridge', state_province='MA', country='U.S.A.',
...     website='http://www.oreilly.com/')
>>> p2.save()
>>> publisher_list = Publisher.objects.all()
>>> publisher_list
[<Publisher: Publisher object>, <Publisher: Publisher object>]
```

这短短几行代码干了不少的事。这里简单的说一下：

- 首先，导入`Publisher`模型类，通过这个类，我们可以与包含出版社*publishers*信息的数据表进行交互。

- 接着，创建一个`Publisher`类的实例，并设置了字段`name`，`address`等的值。

- 调用该对象的`save()`方法，将对象保存到数据库中。Django会在后台执行一条`INSERT`语句。

- 最后，使用`Publisher.objects`属性从数据库取出出版商的信息，这个属性可以认为是包含出版商的记录集。这个属性*attribute*有许多方法，这里先介绍调用`Publisher.objects.all()`方法获取数据库中`Publisher`类的所有对象。这个操作的幕后，Django执行了一条SQL语言的`SELECT`语句。【重要，这个是直接调用**类属性**】

这里有一个值得注意的地方，在这个例子可能并未清晰地展示。当你使用Django model API创建对象时，Django并未将对象保存至数据库内，除非你调用`save()`方法：

```Python
p1 = Publisher(...)
# At this point, p1 is not saved to the database yet!
p1.save()
# Now it is.
```

如果需要**一步完成**对象的创建与存储至数据库，就使用`objects.create()`方法。下面的例子与之前的例子等价：【重要，实际情况，这个方法运用的更多吧】

```Python
>>> p1 = Publisher.objects.create(name='Apress',
...     address='2855 Telegraph Avenue',
...     city='Berkeley', state_province='CA', country='U.S.A.',
...     website='http://www.apress.com/')
>>> p2 = Publisher.objects.create(name="O'Reilly",
...     address='10 Fawcett St.', city='Cambridge',
...     state_province='MA', country='U.S.A.',
...     website='http://www.oreilly.com/')
>>> publisher_list = Publisher.objects.all()
>>> publisher_list
```

当然，你可以执行更多的Django数据库API来玩，不过，还是让我们先解决掉一个烦人的小问题。

## 添加模块的字符串表现 ##

当我们打印整个publisher列表时，我们没有得到想要的有用信息，无法把`Publisher`对象区分开来：

```Python
[<Publisher: Publisher object>, <Publisher: Publisher object>]
```

我们可以简单解决这个问题，只需要为Publisher对象添加一个方法`__unicode__()`。`__unicode__()`方法告诉Python如何将对象以unicode的方式显示出来。为以上三个模型添加`__unicode__()`方法后，就可以看到效果了：【翻看官网的1.7文档，py3中是否还有这个？或者是改成了__str__()】


```Python
from django.db import models

class Publisher(models.Model):
    name = models.CharField(max_length=30)
    address = models.CharField(max_length=50)
    city = models.CharField(max_length=60)
    state_province = models.CharField(max_length=30)
    country = models.CharField(max_length=50)
    website = models.URLField()

    def __unicode__(self):
        return self.name

class Author(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=40)
    email = models.EmailField()

    def __unicode__(self):
        return u'%s %s' % (self.first_name, self.last_name)

class Book(models.Model):
    title = models.CharField(max_length=100)
    authors = models.ManyToManyField(Author)
    publisher = models.ForeignKey(Publisher)
    publication_date = models.DateField()

    def __unicode__(self):
        return self.title
```

* * *
笔记：注意，Python3中，果然已经改成__str__()了。

```Python
from django.db import models

class Person(models.Model):
    name = models.CharField(max_length=128)

    def __str__(self):              # __unicode__ on Python 2
        return self.name
```
* * *

就象你看到的一样，`__unicode__()`方法可以进行任意处理，来返回对一个对象的字符串表示。`Publisher`和`Book`对象的`__unicode__()`方法简单地返回各自的名称和标题，`Author`对象的`__unicode__()`方法则稍微复杂一些，它将`first_name`和`last_name`字段值以空格连接后再返回。【Python3中的`__unicode__()`不起作用，用`__str__()`就可以】

对`__unicode__()`的唯一要求就是它要返回一个unicode对象 如果`__unicode__()`方法未能返回一个Unicode对象，而返回比如说，一个整型数字，那么Python将抛出一个`TypeError`错误，并提示："coercing to Unicode: need string or buffer, int found".

* * *
Unicode对象
* * *

什么是Unicode对象呢？【Py3大法用上以后，这个就不是问题了，全部直接__str()__】

你可以认为unicode对象就是一个Python字符串，它可以处理上百万不同类别的字符——从古老版本的Latin字符到非Latin字符，再到曲折的引用和艰涩的符号。

普通的python字符串是经过编码的，意思就是它们使用了某种编码方式（如ASCII，ISO-8859-1或者UTF-8）来编码。如果你把奇特的字符（其它任何超出标准128个如0-9和A-Z之类的ASCII字符）保存在一个普通的Python字符串里，你一定要跟踪你的字符串是用什么编码的，否则这些奇特的字符可能会在显示或者打印的时候出现乱码。当你尝试要将用某种编码保存的数据结合到另外一种编码的数据中，或者你想要把它显示在已经假定了某种编码的程序中的时候，问题就会发生。我们都已经见到过网页和邮件被“??? ??????”弄得乱七八糟，或者其它出现在奇怪位置的字符，这一般来说就是存在编码问题了。

但是Unicode对象并没有编码*encoding*。它们使用Unicode，一个一致的、通用的字符编码集。当你在Python中处理Unicode对象的时候，你可以直接将它们混合使用和互相匹配而不必去考虑编码细节。

Django在其内部的各个方面都使用到了Unicode对象。Model对象中，检索匹配方面的操作使用的是Unicode对象，View函数之间的交互使用的是Unicode对象，Template的渲染也是用的Unicode对象。通常，我们不必担心编码是否正确，后台会处理的很好。

注意，我们这里只是对Unicode对象进行非常浅显的概述，若要深入了解你可能需要查阅相关的资料。这是一个很好的起点： http://www.joelonsoftware.com/articles/Unicode.html 。

为了让我们的修改生效，先退出Python Shell，然后再次运行`python manage.py shell`进入。（这是保证代码修改生效的最简单方法。）现在`Publisher`对象列表容易理解多了：

```Python
>>> from books.models import Publisher
>>> publisher_list = Publisher.objects.all()
>>> publisher_list
[<Publisher: Apress>, <Publisher: O'Reilly>]
```

请确保你的每一个模型里都包含`__unicode__()`方法，这不只是为了交互时方便，也是因为Django会在其他一些地方用__unicode__()来显示对象。【重要，我以为只是用来print()玩的呢】

最后，`__unicode__()`也是一个很好的例子来演示我们怎么添加行为*behavior*到模型里。Django的模型不只是为对象定义了数据库表的结构，还定义了对象的行为。__unicode__()就是一个例子来演示模型知道怎么显示它们自己。

## 插入和更新数据 ##

你已经知道怎么做了，要插入一条数据，先使用一些kwarg参数*keyword arguments*创建对象实例，如下：

```Python
>>> p = Publisher(name='Apress',
...         address='2855 Telegraph Ave.',
...         city='Berkeley',
...         state_province='CA',
...         country='U.S.A.',
...         website='http://www.apress.com/')
```

如前文所述，这个实例化模型类的操作，并没有*not*对数据库本身做修改。在调用`save()`方法之前，记录并没有保存至数据库，除非你这样做：

```Python
>>> p.save()
```

在SQL里，这大致可以转换成这样：

```SQL
INSERT INTO books_publisher
    (name, address, city, state_province, country, website)
VALUES
    ('Apress', '2855 Telegraph Ave.', 'Berkeley', 'CA',
     'U.S.A.', 'http://www.apress.com/');
```

因为`Publisher`模型有一个自动增加的主键`id`，所以第一次调用`save()`还多做了一件事，计算这个主键的值并把它赋值给这个对象实例：【重要】【Py3因为是长整型默认，所以可能是52L】

```Python
>>> p.id
52    # this will differ based on your own data
```

接下来再调用`save()`将不会创建新的记录，而只是修改记录内容（也就是执行一条`UPDATE`的SQL语句，而不是`INSERT`语句）：

```Python
>>> p.name = 'Apress Publishing'
>>> p.save()
```

上面执行的`save()`相当于下面的SQL语句：

```SQL
UPDATE books_publisher SET
    name = 'Apress Publishing',
    address = '2855 Telegraph Ave.',
    city = 'Berkeley',
    state_province = 'CA',
    country = 'U.S.A.',
    website = 'http://www.apress.com'
WHERE id = 52;
```

注意，并不是只更新修改过的那个字段，**所有**的字段都会被更新。这个操作有可能引起**竞态条件**，这取决于你的应用程序。请参阅后面的“更新多个对象”小节以了解如何实现这种轻量的修改（只修改对象的部分字段）。【竞态条件（race condition），从多进程间通信的角度来讲，是指两个或多个进程对共享的数据进行读或写的操作时，最终的结果取决于这些进程的执行顺序。--internet】【下面是，小的更改情况的语句】

```SQL
UPDATE books_publisher SET
    name = 'Apress Publishing'
WHERE id=52;
```

## 选择对象 ##

当然，创建新的数据库，并更新之中的数据是必要的，但是，对于Web应用程序来说，更多的时候是在检索查询数据库。我们已经知道如何从一个给定的模型中取出所有记录：

```Python
>>> Publisher.objects.all()
[<Publisher: Apress>, <Publisher: O'Reilly>]
```

这相当于这个SQL语句：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher;
```

* * *
注意

请注意，在Django在选择所有数据时并没有使用`SELECT *`，而是显式列出了所有字段。设计的时候就是这样：`SELECT *`会更慢，而且最重要的是列出所有字段遵循了Python界的一个信条：明言胜于暗示*Explicit is better than implicit*。【Python不建议用*】

有关Python之禅，在Python提示行输入`import this`试试看。
* * *

让我们来仔细看看`Publisher.objects.all()`这行的每个部分：

- 首先，我们有一个已定义的模型`Publisher`。没什么好奇怪的：你想要查找数据，你就用模型来获得数据。

- 然后，是`objects`属性。它被称为管理器*manager*，我们将在第10章中详细讨论它。目前，我们只需了解管理器管理着所有针对数据包含*data including*、还有最重要的，数据查询*data lookup*的表格级操作。

    所有的模型都自动拥有一个`objects`管理器；你可以在想要查找数据时使用它。

- 最后，还有`all()`方法。这个方法返回数据库中所有的记录。尽管这个对象看起来`looks like`像一个列表`list`，它实际是一个*QuerySet*对象，这个对象是数据库中一些记录的集合。附录C将详细描述QuerySet。现在，我们就先当它是一个**仿真列表**对象好了。

任何对数据库的查询操作都遵循这样的模式：即调用绑定在该模型上的管理器`objects`的相应方法（如`all()`）。

### 数据过滤 ###

我们很少会一次性从数据库中取出所有的数据；通常都只针对一部分数据进行操作。在Django API中，我们可以使用`filter()`方法对数据进行过滤：

```Python
>>> Publisher.objects.filter(name='Apress')
[<Publisher: Apress>]
```

`filter()`根据关键字参数来转换成`WHERE`SQL语句。前面这个例子相当于这样：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
WHERE name = 'Apress';
```

你可以传递多个参数到`filter()`来缩小选取范围：

```Python
>>> Publisher.objects.filter(country="U.S.A.", state_province="CA")
[<Publisher: Apress>]
```

多个参数会被转换成`AND`SQL从句，因此上面的代码可以转化成这样：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
WHERE country = 'U.S.A.'
AND state_province = 'CA';
```

【提问：AND有了，OR怎么办？】【回答，如下：

```Python
>>> from django.db.models import Q
>>> Publisher.objects.filter(Q(name='Apress')|Q(name="O'Reilly"))
```

详情参看： https://docs.djangoproject.com/en/1.6/topics/db/queries/ 】

注意，SQL缺省的`=`操作符是精确匹配的，其他类型的查找也可以使用：【注意双下划线】

```Python
>>> Publisher.objects.filter(name__contains="press")
[<Publisher: Apress>]
```

在`name`和`contains`之间有双下划线。和Python一样，Django也使用双下划线来表明会进行一些魔术般操作的方法。这里，`contains`部分会被Django翻译成`LIKE`语句：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
WHERE name LIKE '%press%';
```

其他的一些查找类型有：`icontains`（大小写不敏感的`LIKE`），`startswith`和`endswith`，还有`range`（SQL语言的`BETWEEN`查询）。附录C详细描述了所有的查找类型。【参见文档 https://docs.djangoproject.com/en/1.6/topics/db/queries/ 】

### 获取单个对象 ###

上面的例子中`filter()`方法返回一个记录集，这个记录集是一个列表。相对列表来说，有些时候我们更需要获取单个的对象，`get()`方法就是在此时使用的：

```Python
>>> Publisher.objects.get(name="Apress")
<Publisher: Apress>
```

这样，就返回了单个对象，而不是列表（更准确的说，QuerySet)。所以，如果结果是多个对象，会导致抛出异常：

```Python
>>> Publisher.objects.get(country="U.S.A.")
Traceback (most recent call last):
    ...
MultipleObjectsReturned: get() returned more than one Publisher --
    it returned 2! Lookup parameters were {'country': 'U.S.A.'}
```

如果查询没有返回结果也会抛出异常：

```Python
>>> Publisher.objects.get(name="Penguin")
Traceback (most recent call last):
    ...
DoesNotExist: Publisher matching query does not exist.
```

这个`DoesNotExist`异常是`Publisher`这个Model类的一个属性，即`Publisher.DoesNotExist`。在你的应用中，你可以捕获并处理这个异常，像这样：【只是演示，实际少用】

```Python
try:
    p = Publisher.objects.get(name='Apress')
except Publisher.DoesNotExist:
    print "Apress isn't in the database yet."
else:
    print "Apress is in the database."
```

### 数据排序 ###

在运行前面的例子中，你可能已经注意到返回的结果是无序的。我们还没有告诉数据库怎样对结果进行排序，所以我们返回的结果是无序的。

在你的Django应用中，你或许希望根据某字段的值对检索结果排序，比如说，按字母顺序。那么，使用`order_by()`这个方法就可以搞定了：

```Python
>>> Publisher.objects.order_by("name")
[<Publisher: Apress>, <Publisher: O'Reilly>]
```

【笔记，这个操作和`Publisher.objects.all().order_by('name')`等价，有没有`all()`，都可以】

跟以前的`all()`例子差不多，SQL语句里多了指定排序的部分：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
ORDER BY name;
```

我们可以对任意字段进行排序：

```Python
>>> Publisher.objects.order_by("address")
[<Publisher: O'Reilly>, <Publisher: Apress>]

>>> Publisher.objects.order_by("state_province")
[<Publisher: Apress>, <Publisher: O'Reilly>]
```

如果需要以多个字段为标准进行排序（第二个字段会在第一个字段的值相同的情况下被使用到），使用多个参数就可以了，如下：

```Python
>>> Publisher.objects.order_by("state_province", "address")
 [<Publisher: Apress>, <Publisher: O'Reilly>]
```

我们还可以指定逆向排序，在前面加一个减号`-`前缀：【这个高级】

```Python
>>> Publisher.objects.order_by("-name")
[<Publisher: O'Reilly>, <Publisher: Apress>]
```

尽管很灵活，但是每次都要用`order_by()`显得有点啰嗦。大多数时间你通常只会对某些字段进行排序。在这种情况下，Django让你可以指定模型的缺省排序方式：

```Python
class Publisher(models.Model):
    name = models.CharField(max_length=30)
    address = models.CharField(max_length=50)
    city = models.CharField(max_length=60)
    state_province = models.CharField(max_length=30)
    country = models.CharField(max_length=50)
    website = models.URLField()

    def __unicode__(self):
        return self.name

    class Meta:                # 注意这里！
        ordering = ['name']
```

现在，让我们来接触一个新的概念。`class Meta`，内嵌于`Publisher`这个类的定义中（如果`class Publisher`是顶格的，那么`class Meta`在它之下要缩进4个空格－－按Python的传统）。你可以在任意一个模型 类中使用Meta类，来设置一些与特定模型相关的选项。在附录B中有Meta中所有可选项的完整参考，现在，我们关注`ordering`这个选项就够了。如果你设置了这个选项，那么除非你检索时特意额外地使用了`order_by()`，否则，当你使用Django的数据库API去检索时，`Publisher`对象的相关返回值默认地都会按`name`字段排序。【变成默认ordering了】

### 链式查询 ###

我们已经知道如何对数据进行过滤和排序。当然，通常我们需要同时进行过滤和排序查询的操作。因此，你可以简单地写成这种“链式”的形式：

```Python
>>> Publisher.objects.filter(country="U.S.A.").order_by("-name")
[<Publisher: O'Reilly>, <Publisher: Apress>]
```

你应该没猜错，转换成SQL查询就是`WHERE`和`ORDER BY`的组合：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
WHERE country = 'U.S.A'
ORDER BY name DESC;
```

### 限制返回的数据 ###

另一个常用的需求就是取出固定数目的记录。想象一下你有成千上万的出版商在你的数据库里，但是你只想显示第一个。你可以使用标准的Python列表裁剪语句：

```Python
>>> Publisher.objects.order_by('name')[0]
<Publisher: Apress>
```

这相当于：【Djago应该是先将Python语法转换成标准的SQL语句，再去操作数据库。】

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
ORDER BY name
LIMIT 1;
```

类似的，你可以用Python的*range-slicing*语法来取出数据的特定子集：

```Python
>>> Publisher.objects.order_by('name')[0:2]
```

这个例子返回两个对象，等同于以下的SQL语句：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
ORDER BY name
OFFSET 0 LIMIT 2;
```

注意；**不支持**Python的负索引*negative slicing*：

```Python
>>> Publisher.objects.order_by('name')[-1]
Traceback (most recent call last):
  ...
AssertionError: Negative indexing is not supported.
```

虽然不支持负索引，但是我们可以使用其他的方法。比如，稍微修改`order_by()`语句来实现：

```Python
>>> Publisher.objects.order_by('-name')[0]
```

### 更新多个对象 ###

在“插入和更新数据”小节中，我们有提到模型的`save()`方法，这个方法只会更新一行里的所有列。而某些情况下，我们只需要更新行里的某几列。【在Django1.5中，可以采用类似：product.save(update_fields=['name']) 的方式来强制更新指定字段。】

例如说我们现在想要将Apress `Publisher`的名称由原来的`'Apress'`更改为`'Apress Publishing'`。若使用`save()`方法，如下所示：

```Python
>>> p = Publisher.objects.get(name='Apress')
>>> p.name = 'Apress Publishing'
>>> p.save()
```

这等同于如下SQL语句：

```SQL
SELECT id, name, address, city, state_province, country, website
FROM books_publisher
WHERE name = 'Apress';

UPDATE books_publisher SET
    name = 'Apress Publishing',
    address = '2855 Telegraph Ave.',
    city = 'Berkeley',
    state_province = 'CA',
    country = 'U.S.A.',
    website = 'http://www.apress.com'
WHERE id = 52;
```

（注意在这里我们假设Apress的ID为`52`）

在这个例子里我们可以看到Django的`save()`方法更新了不仅仅是`name`列的值，还有更新了所有的列。若`name`以外的列有可能会被其他的进程所改动的情况下，只更改`name`列显然是更加明智的。更改某一指定的列，我们可以调用结果集*QuerySet*对象的`update()`方法，示例如下：

```Python
>>> Publisher.objects.filter(id=52).update(name='Apress Publishing')
```

与之等同的SQL语句变得更高效，并且不会引起竞态条件：

```SQL
UPDATE books_publisher
SET name = 'Apress Publishing'
WHERE id = 52;
```

`update()`方法对于任何结果集*QuerySet*均有效，这意味着你可以同时更新多条记录。以下示例演示如何将所有`Publisher`的`country`字段值由`'U.S.A'`更改为`'USA'`：【注意，update只针对“结果集”，如果是get()返回的就不能update】

```Python
>>> Publisher.objects.all().update(country='USA')
2
```

`update()`方法会返回一个整型数值，表示受影响的记录条数。在上面的例子中，这个值是2。【Py3长整型】

## 删除对象 ##

删除数据库中的对象只需调用该对象的`delete()`方法即可：

```Python
>>> p = Publisher.objects.get(name="O'Reilly")
>>> p.delete()
>>> Publisher.objects.all()
[<Publisher: Apress Publishing>]
```

同样我们可以在结果集上调用`delete()`方法同时删除多条记录。这一点与我们上一小节提到的`update()`方法相似：【delete()可用于单个对象也可用于结果集】

```Python
>>> Publisher.objects.filter(country='USA').delete()
>>> Publisher.objects.all().delete()
>>> Publisher.objects.all()
[]
```

删除数据时要谨慎！为了预防误删除掉某一个表内的所有数据，Django要求在删除表内所有数据时**显式**使用`all()`。比如，下面的操作将会出错：

```Python
>>> Publisher.objects.delete()
Traceback (most recent call last):
  File "<console>", line 1, in <module>
AttributeError: 'Manager' object has no attribute 'delete'
```

而一旦使用`all()`方法，所有数据将会被删除：

```Python
>>> Publisher.objects.all().delete()
```

如果只需要删除部分的数据，就不需要调用`all()`方法。再看一下之前的例子：

```Python
>>> Publisher.objects.filter(country='USA').delete()
```

## 下一章 ##

通过本章的学习，你应该可以熟练地使用Django模型来编写一些简单的数据库应用程序。在第10章我们将讨论Django数据库层的高级应用。

一旦你定义了你的模型，接下来就是要把数据导入数据库里了。你可能已经有现成的数据了，请看18章以获得有关如何集成现有数据库的建议。也可能数据是用户提供的，第7章中还会教你怎么处理用户提交的数据。

有时候，你和你的团队成员也需要手工输入数据，这时候如果有一个基于Web的数据输入和管理的界面就会很有帮助。下一章将介绍解决手工录入问题的方法——Django Admin管理界面。
