# models.py 模型类的定义

## 一、 创建数据模型

### 实例

我们来假定下面的这些概念，字段和关系：

- **作者模型**: 一个作者有姓名。
- **作者详情模型**： 把作者的详情放到详情表，包含性别、email、地址和出生日期，作者详情模型和作者模型之间是一对一的关系(OneToOneField)
- **出版商模型**： 出版商有名称，地址，所在城市、省、国家和网站等
- **书籍模型**： 书籍有书名和出版日期。一本书可能会有多个作者，一个作者也可以写多本书，所以作者和书籍的关系是多对多的关系(manay-to-manay)，一本书只应该由一个出版商出版，所以出版商和书籍是一对多的关系(one-to-manay)，也被称作外键(ForeignKey).

```python
from django.db import models
class Publisher(models.Model):
    #字段，字符串，最长长度为30
    name = models.CharField(max_length=30)
    address = models.CharField(max_length=30)
    city = models.CharField(max_length=60)
    state_province = models.CharField(max_length=30)
    country = models.CharField(max_length=50)
    website = models.URLField() #类型为URL

class Author(models.Model):
    name = models.CharField(max_length=30)

class AuthorDetail(models.Model):
    # 性别 布尔类型，0为男，1为女
    sex = models.BooleanField(max_length=1, choices=((0,'男'),(1,'女'),))
    email = models.EmailField() # 邮件类型
    address = models.CharField(max_length=50)
    birthday = models.DateField()
    # 和作者是一对一的关系
    author = models.OneToOneField(Author, on_delete=models.CASCADE)

class Book(models.Model):
    title = models.CharField(max_length=100)
    # 书籍和作者是多对多的关系
    authors = models.ManyToManyField(Author)
    # 书籍和出版商是一对多的关系，用外键
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
    publication_date = models.DateField()
```

在上面代码中我们需要注意以下几点：

- **每个数据模型都是 django.db.models.Model 的子类**，它的父类 Model 包含了所有必要的和数据库交互的方法，并提供了一个简洁漂亮的定义数据库字段的语法。
- **每个模型相当于单个数据库表** (这条规则例外情况是多对多关系，多对多关系的时候会多生成一张关系表)， 每个属性也是这个表中的一个字段。属性名就是字段名，它的类型(例如 CharField)相当于数据库的字段类型(例如 varchar)。
- **模型之间的三种关系**：一对一（OneToOneField), 一对多（ForeignKey）和多对多（ManyToManyField）

### 模型的常用字段类型

- BooleanField：布尔类型字段
- CharField：字符串类型
- DateField: 日期字段
- DateTimeField: 日期时间字段
- DecimalField：（精确）小数字段
- EmailField：Email字段
- FileField：文件字段
- FloatField：（浮点数）小数字段
- ImageField：图片字段
- IntegerField：整数字段
- IPAddressField：IP字段
- SmallIntegerField：小整数字段
- TextField：文本字段
- URLField：网页地址字段

### 模型常用的字段选项

- null: (null = True|False) 数据库字段的设置是否可以为空(数据库进行验证)
- blank (blank = True|False) 字段是否为空django会进行校验(表单验证)
- choices 轻量级的配置字段可选属性的定义
- default 字段的默认值
- help_text 字段文字帮助
- primary_key (=True|False)一般情况下不需要进行定义，是否主键，如果没有显示指明主键的话，django会自动增加一个默认主键：id = models.AutoField(primary=True)
- unique 是否唯一，对于数据表而言。
- verbose_name 字段的详细名称，如果不指定该属性，默认使用字段的属性名称。

### 定义数据模型的扩展属性

通过内部类Meta给数据模型类增加扩展属性：

```python
class Meta:
        verbose_name = '名称'
        verbose_name_plural = '名称复数形式'
        ordering = ['排序字段']
```

### 定义模型的方法

定义模型的方法和普通python类方法没有太大的差别，定义模型方法可以将当前对应的数据，组装成具体的业务逻辑。

```python
def __str__(self):
    return self.name
```
