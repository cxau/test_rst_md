配置纲要如下：

- `DATABASE_ENGINE`告诉Django使用哪个数据库引擎。如果你在Django中使用数据库，`DATABASE_ENGINE`必须是表5-1中所列出的值。

    | 表5-1                                                                         |
    | 设置                                   | 数据库     | 适配器                  | 
    | -------------------------------------- | ---------- | :---------------------- |
    | django.db.backends.postgresql_psycopg2 | PostgreSQL | `psycopg` version 2.x, http://www.djangoproject.com/r/python-pgsql/
    | django.db.backends.mysql               | MySQL      | `MySQLdb`, http://www.djangoproject.com/r/python-mysql/
    | django.db.backends.sqlite3             | SQLite     | 不需要
    | django.db.backends.oracle              | Oracle     | `cx_Oracle`, http://www.djangoproject.com/r/python-oracle/

    注意，无论选择使用哪个数据库服务器，都必须下载和安装对应的数据库适配器。访问表5-1中“适配器”一栏中的链接，可通过互联网免费获取这些适配器。如果你使用Linux，你的发布包管理系统会提供合适的包。比如说查找`python-postgresql`或者`python-psycopg`的软件包。
