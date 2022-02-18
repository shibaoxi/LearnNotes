# QuerySet 常用的API

> QuerySet 特点
>
> * 是可迭代的
> * 可切片

查询相关的API

* **Get(\**kwargs)**:返回与所给的筛选条件相匹配的对象，返回结果有且只有一个。如果符合筛选条件的对象超过一个，就会抛出MultipleObjectsReturned异常，如果没有找到符合筛选条件的对象，就会抛出DoesNotExist异常
* **all()**：查询所有结果.
* **filter(\*\*kwargs)**:它包含了与所给的筛选条件相匹配的对象。
* **exclude(\*\*kwargs)**:它包含了那些与所给筛选条件不匹配的对象
* **order_by(\*fields)**:对查询结果排序
* **reverse()**: 对查询结果反向排序
* **distinct()**: 从返回结果中剔除重复记录
* **values(*fields)**:返回一个ValuesQuerySet（一个特殊的QuerySet）, 运行后得到的并不是一系列model的实例化对象，而是一个可迭代的字典序列。
* **values_list(\*fields)**:它与values()非常相似，只不过后者返回的结果是字典序列，而values_list()返回的结果是元组序列。
* **count()**:返回数据库中匹配查询（QuerySet）的对象数量
* **first()**: 返回第一条记录，等价于[:1][0]
* **last()**: 返回最后一条记录，等价于[::-1][0]
* **exists()**: 如果QuerySet包含有数据，就返回True 否则就返回False。

## 实例

查询id为1的书籍信息，并且只显示书籍名称和出版日期

```python
Book.objects.filter(id=2).values("title","publication_date")
```

查询所有出版社信息，并按id降序排列，并尝试使用reverse方法进行反向排序

```python
# 升序排列
Publisher.objects.all().order_by('id')
# 降序排列
Publisher.objects.all().order_by('-id')
# 反向排序
Publisher.objects.all().order_by('id').reverse()
```

查询出版社所在的城市信息，城市信息不要重复

```python
Publisher.objects.all().values('city').distinct()
```

查询城市是北京的出版社，尝试使用exclude方法

```python
Publisher.objects.filter(city='北京')
# 查询不是北京的出版时
Publisher.objects.exclude(city='北京')
```

查询男作者的数量

```python
AuthorDetail.objects.filter(sex=0).count()
```
