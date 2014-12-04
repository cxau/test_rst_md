# Day 1

## 准备工作

练习环境设定在Django框架中，编辑使用PyCharm。

- Django提供`本地服务器`
- PyCharm提供`LiveEdit`功能，可以配合`Chrome`浏览器方便地实验
- PyCharm有`Zencoding`功能，熟悉使用它编辑HTML。

注意：在Django中创建视图时，记住用`render()`而不是`render_to_response()`，后者会让`{{ STATIC_URL }}`解析不正确，导致不能正确载入js。

## 安装jQuery库

### 获取

官网获取，在 http://jquery.com

### 版本选择

官方提供min版和dev版。min版用在生产环境。dev版用在开发环境。dev版可以用来学习源码。

### CDN还是本地

鉴于墙的存在，我选择本地引用jQuery。

### 导入jQuery库

新建一个html5文档，在`</body>`标签上方引入jQuery库。这样可以让网页的加载在感觉上好像快一点。

```HTML
<script src="{{ STATIC_URL }}js/jquery.js"></script>

</body>
```

注意：你自己的jQuery代码，要放在这个引用`<script>`的下面。不然会提示jQuery不存在。**导入**后才能使用。

## 初识jQuery

### 函数名

`jQuery()`，有两个别名，等价于`window.jQuery()`和`$()`。大家都喜欢用美元。

```Javascript
	// Map over jQuery in case of overwrite
	_jQuery = window.jQuery,

	// Map over the $ in case of overwrite
	_$ = window.$,
```

### 初试选择器

`$()`可以做选择器用。方便选取dom元素，给它添加/更改属性。

### CSS操作

尽量使用`addClass()`方法，使用`类`来更改式样，而不是直接通过`css()`方法。用`类`实现内容·样式·逻辑`分离`的理念。松耦合。
