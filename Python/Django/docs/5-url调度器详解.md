# URL 调度器详解

## 简介

为了给一个应用设计 URL，你需要创建一个 Python 模块，通常被称为 URLconf （URL configuration）。这个模块是纯粹的 Python 代码，包含 URL 模式（简单的正则表达式）到 Python 函数（你的视图）的简单映射。

## Django 如何处理一个请求

当一个用户请求 Django 站点的一个页面，下面是 Django 系统决定执行哪个 Python 代码使用的算法：

1. Django 确定使用根 URLconf 模块。通常，这是 ROOT_URLCONF 设置的值，但如果传入 HttpRequest 对象拥有 urlconf 属性（通过中间件设置），它的值将被用来代替 ROOT_URLCONF 设置。

2. Django 加载该 Python 模块并寻找可用的 urlpatterns 。它是 django.urls.path() 和(或) django.urls.re_path()。

3. Django 会按顺序遍历每个 URL 模式，然后会在所请求的URL匹配到第一个模式后停止，并与 path_info 匹配。

4. 一旦有 URL 匹配成功，Djagno 导入并调用相关的视图，这个视图是一个Python 函数。

5. 如果没有 URL 被匹配，或者匹配过程中出现了异常，Django 会调用一个适当的错误处理视图。

## URLconf 示例

```python
from django.urls import path

from . import views

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    path('articles/<int:year>/', views.year_archive),
    path('articles/<int:year>/<int:month>/', views.month_archive),
    path('articles/<int:year>/<int:month>/<slug:slug>/', views.article_detail),
]
```

注意：

- 要从 URL 中取值，使用尖括号。
- 捕获的值可以选择性地包含转换器类型。比如，使用 <int:name> 来捕获整型参数。如果不包含转换器，则会匹配除了 / 外的任何字符。
- 这里不需要添加反斜杠，因为每个 URL 都有。比如，应该是 articles 而不是 /articles 。
一些请求的例子：

- /articles/2005/03/ 会匹配 URL 列表中的第三项。Django 会调用函数 views.month_archive(request, year=2005, month=3) 。
- /articles/2003/ 将匹配列表中的第一个模式不是第二个，因为模式按顺序匹配，第一个会首先测试是否匹配。请像这样自由插入一些特殊的情况来探测匹配的次序。在这里 Django 会调用函数 views.special_case_2003(request)
- /articles/2003 不匹配任何一个模式，因为每个模式要求 URL 以一个斜线结尾。
- /articles/2003/03/building-a-django-site/ 会匹配 URL 列表中的最后一项。Django 会调用函数 views.article_detail(request, year=2003, month=3, slug="building-a-django-site") 。

## 使用正则表达式

如果路径和转化器语法不能很好的定义你的 URL 模式，你可以可以使用正则表达式。如果要这样做，请使用 re_path() 而不是 path() 。

在 Python 正则表达式中，命名正则表达式组的语法是 (?P\<name>pattern) ，其中 name 是组名，pattern 是要匹配的模式。

这里是先前 URLconf 的一些例子，现在用正则表达式重写一下：

```python
from django.urls import path, re_path

from . import views

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    re_path(r'^articles/(?P<year>[0-9]{4})/$', views.year_archive),
    re_path(r'^articles/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/$', views.month_archive),
    re_path(r'^articles/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/(?P<slug>[\w-]+)/$', views.article_detail),
]
```

这实现了与前面示例大致相同的功能，除了:

- 将要匹配的 URLs 将稍受限制。比如，10000 年将不在匹配，因为 year 被限制长度为4。
- 无论正则表达式进行哪种匹配，每个捕获的参数都作为字符串发送到视图。
