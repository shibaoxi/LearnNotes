# Restful 介绍以及如何在Django中配置

## Restful介绍

### 协议

API与用户的通信协议，总是使用HTTPs协议。

### 域名

应该尽量将API部署在专用域名之下。

```bash
https://api.example.com
```

如果确定API很简单，不会有进一步扩展，可以考虑放在主域名下。

```bash
https://example.org/api/
```

### 版本

应该将API的版本号放入URL。

```bash
https://api.example.com/v1/
```

### 路径

路径又称"终点"（endpoint），表示API的具体网址。

在RESTful架构中，每个网址代表一种资源（resource），所以网址中不能有动词，只能有名词，而且所用的名词往往与数据库的表格名对应。一般来说，数据库中的表都是同种记录的"集合"（collection），所以API中的名词也应该使用复数。

举例来说，有一个API提供动物园（zoo）的信息，还包括各种动物和雇员的信息，则它的路径应该设计成下面这样。

```bash
https://api.example.com/v1/zoos
https://api.example.com/v1/animals
https://api.example.com/v1/employees
```

### https 动词

对于资源的具体操作类型，由HTTP动词表示。

常用的HTTP动词有下面五个（括号里是对应的SQL命令）

```bash
GET（SELECT）：从服务器取出资源（一项或多项）。
POST（CREATE）：在服务器新建一个资源。
PUT（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）。
PATCH（UPDATE）：在服务器更新资源（客户端提供改变的属性）。
DELETE（DELETE）：从服务器删除资源。
```

下面是一些例子:

```bash
GET /zoos：列出所有动物园
POST /zoos：新建一个动物园
GET /zoos/ID：获取某个指定动物园的信息
PUT /zoos/ID：更新某个指定动物园的信息（提供该动物园的全部信息）
PATCH /zoos/ID：更新某个指定动物园的信息（提供该动物园的部分信息）
DELETE /zoos/ID：删除某个动物园
GET /zoos/ID/animals：列出某个指定动物园的所有动物
DELETE /zoos/ID/animals/ID：删除某个指定动物园的指定动物
```

### 过滤信息

如果记录数量很多，服务器不可能都将它们返回给用户。API应该提供参数，过滤返回结果。

下面是一些常见的参数。

```bash
?limit=10：指定返回记录的数量
?offset=10：指定返回记录的开始位置。
?page=2&per_page=100：指定第几页，以及每页的记录数。
?sortby=name&order=asc：指定返回结果按照哪个属性排序，以及排序顺序。
?animal_type_id=1：指定筛选条件
```

参数的设计允许存在冗余，即允许API路径和URL参数偶尔有重复。比如，GET /zoo/ID/animals 与 GET /animals?zoo_id=ID 的含义是相同的。

### 状态码

服务器向用户返回的状态码和提示信息，常见的有以下一些（方括号中是该状态码对应的HTTP动词）。

```bash
200 OK - [GET]：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
204 NO CONTENT - [DELETE]：用户删除数据成功。
400 INVALID REQUEST - [POST/PUT/PATCH]：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
404 NOT FOUND - [*]：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
422 Unprocesable entity - [POST/PUT/PATCH] 当创建一个对象时，发生一个验证错误。
500 INTERNAL SERVER ERROR - [*]：服务器发生错误，用户将无法判断发出的请求是否成功
```

### 错误处理

如果状态码是4xx，就应该向用户返回出错信息。一般来说，返回的信息中将error作为键名，出错信息作为键值即可。

```bash
{
    error: "Invalid API key"
}
```

### 返回结果

针对不同操作，服务器向用户返回的结果应该符合以下规范。

```bash
GET /collection：返回资源对象的列表（数组）
GET /collection/resource：返回单个资源对象
POST /collection：返回新生成的资源对象
PUT /collection/resource：返回完整的资源对象
PATCH /collection/resource：返回完整的资源对象
DELETE /collection/resource：返回一个空文档
```

## DRF 的安装和快速实现

### 安装和注册DRF

```powershell
# 安装包
pip install djangorestframework
```

在 settings.py 中注册rest_framework

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework', # 注册 DRF
    'filemanager', # 注册app
]

```

### 序列化

主要负责对象和json格式的相互转换，例如

1. 获取数据： 对象 --> Json 返回给前端
2. 添加，修改： json --> 对象 存储在数据库中

在 app 中创建一个 serializer.py 文件，添加如下代码

```python
# ================导入模块===================
from rest_framework import serializers
from filemanager.models import FileInfo

# =================文件信息序列化类============

class FileInfoSerializer(serializers.ModelSerializer):

    class Meta:
        model = FileInfo
        fields = "__all__"
```

### 视图

实现后台功能的核心

在 views.py 中添加如下代码

```python
# ===================导入模块=======================

from rest_framework.viewsets import ModelViewSet # 封装完成的ModelViewSet视图集
from filemanager.models import FileInfo
from filemanager.serializer import FileInfoSerializer


# ============文件信息视图=================

class FileInfoViewSet(ModelViewSet):
    queryset = FileInfo.objects.all()
    serializer_class = FileInfoSerializer
```

### 路由

路由的匹配

在app中创建 urls.py

```python
# ===============导入模块===================
from django.urls import path
from rest_framework.routers import DefaultRouter
from filemanager.views import FileInfoViewSet

#  实例化一个DefaultRouter对象
router = DefaultRouter()

#  注册相应的url

# 注册fileinfo对象

router.register('files', FileInfoViewSet, basename='files') # https://127.0.0.1:8000/api/v1/files/


urlpatterns = [

]

#  附加到urlpatterns集合中

urlpatterns += router.urls
```

修改主目录下面的 urls.py

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('filemanager.urls'))
]
```

## DRF 筛选

### 安装

```powershell
pip install django-filter
```

### 注册到installed_apps中

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework', # 注册 DRF
    'django_filters', # DRF 筛选
    'filemanager', # 注册app
]

```

### 完成filter的筛选类

在 app 文件夹下面创建一个 filter.py 文件，添加如下代码：

```python
# ============导入模块==================
from django_filters import filterset
from filemanager.models import FileInfo

# 文件信息的筛选类
class FileInfoFilter(filterset):
    # 重写需要支持模糊匹配的字段
    name = filters.CharFilter(field_name='name', lookup_expr="icontains")
    type = filters.CharFilter(field_name='type', lookup_expr='icontains')
    id = filters.NumberFilter(field_name='id', lookup_expr='icontains')
    pid = filters.NumberFilter(field_name='pid')
    class Meta:
        model = FileInfo
        fields = ('name', 'type', 'id', 'pid')
```

### 在viewsset中添加筛选类

```python
from filemanager.filter import FileInfoFilter # 导入筛选类

class FileInfoViewSet(ModelViewSet):
    queryset = FileInfo.objects.all()
    serializer_class = FileInfoSerializer

    # 制定筛选的类
    filter_class =FileInfoFilter
```

### 在 setting 中添加全局的Rest framework 设置

```python
# =========== REST Framework 全局设置 ===============
REST_FRAMEWORK = {

    # 设置全局的filters backend
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
    ]

}
```

### DRF 中的筛选和查找

- 筛选【filter】： 一个值只能对应一个字段，需要django-filter
- 搜索【search】： 一个值对应多个字段，DRF自带

## 在项目中添加DRF 查找功能

在 settings 中修改如下代码

```python
# =========== REST Framework 全局设置 ===============
REST_FRAMEWORK = {

    # 设置全局的filters backend
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
    ]

}
```

在view中添加如下代码

```python
class FileInfoViewSet(ModelViewSet):
    queryset = FileInfo.objects.all()
    serializer_class = FileInfoSerializer
    # 制定筛选的类
    filter_class =FileInfoFilter

    # 指定查找匹配的字段
    search_fields = ('name', 'author')
```

## DRF 添加分页功能

在app 目录下面创建新的 paginations.py 文件，添加如下代码：

```python
from rest_framework.pagination import PageNumberPagination

class PageNumberPagination(PageNumberPagination):
    page_size = 5
    page_query_param = 'page'
    page_size_query_param = 'size'
    max_page_size = 25
```

在view 文件下面添加如下代码：

```python
from filemanager.paginations import PageNumberPagination # 导入分页的类
# 分页设置
    pagination_class = PageNumberPagination
```

## 自动化生成API文档

### 安装swagger

```powershell
pip install drf-yasg
```

### 配置

在根urls文件中添加如下代码：

```python
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

# swagger 配置
schema_view = get_schema_view(
    openapi.Info(
        title="文件管理系统API接口文档平台", #必传
        default_version='v1',    #必传
        description="这是一个接口文档",
        terms_of_service="",
        contact=openapi.Contact(email='shibaoxi@live.com'),
        license=openapi.License(name="BSD License"),
    ),
    public=True,
    # permission_classes=()
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('filemanager.urls')),
    path('docs/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
]
```

在 settings 文件中注册应用

```python
'drf_yasg', # 注册swagger
```
