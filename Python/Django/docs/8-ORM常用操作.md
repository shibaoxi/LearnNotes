# ORM 常用操作

## 增加

create 和 save 方法

- 增加一条作者记录

```python
from demoapp.models import *
Author.objects.create(name='胡大海')
AuthorDetail.objects.create(sex=False, email='hudahai@qq.com', address='北京市朝阳区XX', birthday='1993-03-04', author_id=1)
```

- 增加一条出版社记录

```python
pub = Publisher()
pub.name = '的空间的出版社' 
pub.address = "西安规程"
pub.city = "陕西" 
pub.state_province = "陕西"   
pub.website = "https://djjkc.unx.com"  
pub.country = "中国" 
pub.save()
```

- 增加一条书籍记录(外键和多对多关系)

```python
Book.objects.create(title="nginx实战", publisher=pub, publication_date="2013-05-07")
book = Book.objects.get(id=1) 
author = Author.objects.get(id=1) 
book.authors.add(author) 
```

>objects:model默认管理器
插入主外键关系的时候可以用对象的方式，也可以直接关联id的方式。
插入多对多关系的时候要分步操作。

## 修改

>update和save方法

- 修改id为1的作者名字为叶良辰，性别为女

```python
author = Author.objects.get(id=1)
author.name = "叶良辰"
author.save()
author_detail = AuthorDetail.objects.get(author= author.id) 
author_detail.sex = True
author_detail.save()
```

- 修改id为2的出版社，网址为www.baidu.com,城市为成都

```python
Publisher.objects.filter(id=2).update(website="www.baidu.com", city="成都") 
```

### 查询(惰性机制)

- 查询所有的出版社信息

```python
Publisher.objects.all()
<QuerySet [<Publisher: 北京惊天出版社>, <Publisher: 的空间的出版社>]>
```

>所谓惰性机制：Publisher.objects.all()只是返回了一个QuerySet（查询结果集对象）,并不会马上执行sql，而是当调用QuerySet的时候才执行。

### 删除

>delete方法

- 删除id为1的书籍信息

```python
Book.objects.filter(id=1).delete()
```

- 删除出版社为成都的记录

```python
Publisher.objects.filter(city="成都").delete()
```

>注意：django中的删除默认是级联删除。
