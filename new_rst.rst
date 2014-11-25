

1.1 每次处理一个字符
====================
感谢：Luther Blissett

测试 `各种` 空格 ``问题`` 。看看 *能不能* 搞定。**黑体**呢？


任务
----
用每次处理一个字符的方式处理字符串。


解决方案
--------
可以创建一个列表，列表的子项是字符串的字符（意思是每个子项是一个字符串，长度为一。Python 实际上并没有一个特别的类型来对应“字符”并以此和字符串区分开来）。我们可以调用内建的 ``list``，用字符串作为参数，如下： ::

   thelist = list(thestring)

也可以不创建一个列表，而直接用 ``for`` 语句完成对该字符串的循环遍历： ::

   for c in thestring:
       do_something_with(c)

或者用列表推导中的 ``for`` 来遍历： ::

   results = map(do_something, thestring)


讨论
----
在 Python 中，字符就是长度为 1 的字符串。可以循环遍历一个字符串，依次访问它的每个字符。也可以用 ``map`` 来实现差不多的功能，只要你愿意每取到一个字符就调用一次处理函数。还可以用内建的 ``list`` 类型来获得该字符串的所有字符的集合，还可以调用 ``sets.Set``，并将该字符串作为参数（在 Python 2.4 中，可以用同样的方式直接调用内建的 ``set``）： ::

   import sets
   magic_chars = sets.Set('abracadabra')
   poppins_chars = sets.Set('supercalifragilisticexpialidocious')
   print ''.join(magic_chars & poppins_chars)  # 集合的交集 [intersection]

   acrd


更多资料
--------
`Library Reference` 中关于序列的章节； `Perl Cookbook` ，1.5 节。


笔记
----
摘自 `Python Cookbook` 第 2 版 中文版

1. Python 没有 C 语言中的 ``char`` 字符型变量，即便是 a = 'a' 也是一个 str，我们可以用 type(a) 和 type(a[0]) 获得它的类型 --> <type 'str'>
2. 列表推导，原文 list comprehension
3. Python 的 set 和其他语言类似, 是一个基本功能包括关系测试和消除重复元素。集合对象还支持 union [联合]，intersection [交]，difference [差] 和 sysmmetric difference [对称差集] 等数学运算
4. ``map()`` 函数，还不太会用，待研究

Python 2.7 运行成功，化简为如下写法： ::

   >>> magic_chars = set('abracadabra')  # 2.4 版以后，内建 set()
   >>> poppins_chars = set('supercalifragilisticexpialidocious')
   >>> print ''.join(magic_chars & poppins_chars)
   acrd
