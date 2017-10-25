---
title: Django的Model层
category: Django
img: diango.jpg
---
Django提供了一个用于构造和操纵数据的抽象层: Model层  
一个model可以简单的定义我们的数据对象,它包含着数据的属性与行为.通常来说,一个model可以映射为一张数据库表.  
要点:  
* 每一个model都是一个继承于django.db.models.Model的python类
* 该model类的每一个属性代表着数据库的一个列
* django提供了一些用于处理的默认方法,可以对数据库进行操作  

---
## 举个例子
但我们要创建一个名为'Person'的数据表时,一般使用SQL是这么创建的:  
```sql
CREATE TABLE Person(
    'id' serial NOT NULL PRIMARY KEY,
    'first_name' varchar(30) NOT NULL,
    'last_name' varchar(30) NOT NULL
);
```
而在django中,使用Model可以这么做:  
```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
```
first\_name与last\_name为model的类属性,其类型应该为Field,且每一个属性对应于数据库的一个列.  
> **注意:**  
>* 使用model构造数据的话,其表名是由model的meta信息自动生成的(但这个操作可以用户自定义),详情请看下文.
>* id会自动添加,该操作也可以被覆盖.详情请看下文.

---
## 如何使用models
一旦在django的app中创建了models.py并且构造好models类,只要将该app(例如叫myapp)注册到django中,即:
```python
INSTALLDED_APPS = [
    # ..
    'myapp',
    # ..
]
```
添加了新的app到INSTALL_APPS列表中后,不要忘了运行下`manage.py migrate`,还有`manage.py makemigrations`.该操作会在数据库中创建好与model相对应的表.  

---
## Fields
Fields是models种最主要的部分,它定义了数据库中的列的内容.需要注意的是,models默认已添加了一些关键操作,像是'clean','save'和'delete'.所以在定义新的fields时注意名称不要跟这些内定的操作相同.  

### Fields的类别
每一个在models中使用的field都是一个Field类的对象.由Django默认创建的field具有以下性质:  
* 它定义了数据库的列的内容,决定了存储的数据类型(INTEGER,VARCHAR,TEXT)
* 它定义了默认的HTML样式(主要用于后台管理)
* 一些基本的数据验证(主要用于表单的验证)

对于Django中Field都有什么类别,等以后更新.  

### Field的可选项
每一个field在创建的时候都可以带上一些可选的参数.比如CharFeld(和它的子类)在创建一个字符类型的数据时,需要带上`max_length`参数表示数据的最大长度(该参数为必填).  
其中有很多是field共有的参数设置:
* null:  
    若为True,django在数据库中存储空值的时候会填入NULL.该值默认为False.  
    对于一些字符类型的field要尽量避免使用null,否则但它为空值时,将会有两种可能的结果:null或者''.django中习惯使用''而不是null.不过也有例外,但一个CharField同时满足unique=True和blank=True时,需要有null=True来避免全取''值而违背了唯一性的约束.  
    当一个数据要取空值时,一般需要带上blank=True与null=True.  
    若为一个BoolenField,想取空值的话还是推荐使用NullBoolenField.
* blank:  
    若为True,该值允许为空.默认为False.
    需要注意的是,允许为空跟值为null是不等价的.null是数据库相关的,而blank是与验证相关.即,若blank=true,则在表单验证中,允许该值不填,否则该值必填.
* choices:  
    一个二维的元祖,它表示了该数据的可选项.若定义了choices,则该数据在页面上显示为一个可选框而不是一个普通的文本输入框.  
    ```python
    # 一个choices的值一般为这种形式:
    GENDER = (
        ('male', '男'),
        ('female', '女'),
    )
    ```
    其中,每一个元祖的第一个元素的值为实际存在数据库中的数据,而第二个元素的值为默认展示出来的数据.若要取得用于展示的数据,可以使用`get_FOO_display()`这种方法.例如:  
    ```python
    from django.db import models

    class Person(models.Model):
        SHIRT_SIZES = (
            ('S', 'Small'),
            ('M', 'Medium'),
            ('L', 'Large'),
        )
        name = models.CharField(max_length=60)
        shirt\_size = models.CharField(max\_length=1, choices=SHIRT_SIZES)
    ```
    ```python
    >> p = Person(name='Fred', shirt_size='L')
    >> p.save()
    >> p.shirt_size
    'L'
    >> p.get\_shirt\_size_display()
    'Large'
    ```  
    同时,choices也可以是一种分组形式:  
    ```python
    MEDIA_CHOICES = (
        # 第一个元素为组名,第二个元素为可选项
        ('Audio', (
            ('vinyl', 'Vinyl'),
            ('cd', 'CD'),
        )),
        ('Video', (
            ('vhs', ('VHS Tape')),
            ('dvd', ('DVD')),
        )),
        ('unknown', 'Unknown'),
    )
    ```
    choices不一定是列表或者元祖,可以是任何可迭代的对象.这可以让你动态的构建数据.但choices还是主要用于静态数据,这些数据的变动一般不会很大.  
    除非该field设定了blank=False并且带有默认值,不然在页面上选择框会是一个'-----'.要重写这项默认操作的话,可以添加一个None值(例如(None, '用于展示的说明')),在CharFeld中也可以是个''.
* default:  
    该field的默认值.若为一个可调用的对象,则在每一个新的数据对象创建的时候,都会自动调用一个该对象来生成值填入默认值.(可以令default=datetime.now用来设置对象的生成时间)  
    该值不应为一个可变对象(eg:model实例,列表,集合).对于ForeignKey,其默认值应为其引用的field的值,而不是model的实例.
* help_text:  
    用于展示表单页面中的帮助文本.需要注意的是该值不会自动转化为HTML代码,即可以在里面添加需要的HTML样式.
* primary_key:
    若为True,该值为该model的主键.  
    并不需要在每一个model里设置一个primary_key=True的field,因为如果没有定义主键的话,django会自动添加一个IntergerField的属性设置为主键,除非你想要自定义一个属性为主键.  
    ```python
    id = models.AutoField(primary_key=True)
    ```
    主键应该是唯一的,若在创建一个实例后,改变了该实例的主键值并保存,则在数据库中会创建一份新的记录而不是覆盖原纪录.  
* unique:
    若为True,该值必须为表中的唯一值.  
    关于唯一性限制还有unique\_for\_date,unique\_for\_month,unique\_for\_year.  
* db_column:  
    表示该field在数据库中的列名,若不设置则默认为field名.
* db_index:  
    若为True,将会为这个field创建一个数据库索引.
* db_tablespace:  
    若该field建了索引则该值指定其表空间.原本默认值为项目中的'DEFAULT\_INDEX\_TABLESPACE'中设定.  
* editable:  
    若为False,该值不会在表单中显示,在model的验证中也会被忽略.默认为True.
* error_messages:  
    替换默认的报错信息,其值应为一个字典,包含报错关键字.  
    各个报错关键字为:null,blank,invalid,invalid\_choice,unique和unique\_for\_data.  
    这些错误信息通常不会传给表单.  
* validators:  
    该field的一系列验证内容.  
* verbose_name:  
    供用户进行理解,用于展示的field名.详情看下文.  

### 详细的字段名
除了ForeignKey,ManyToManyField和OneToOneField,其余每一个field类型第一个参数都为--'verbose_name'.若该参数为设置,则django会根据field名自动创建,其中下划线会被转变为空格.例如:  
```python
first_name = models.CharField("person's first name", max_length=30)
```
该例子中的详细名即为"person's first name".  
```python
first_name = models.CharField(max_length=30)
```
在这个例子中,详细字段名被自动设置为'first name'.
至于上面提到的Foreigney三个,由于他们的第一个参数位置需要的是一个model类,所以应该用verbose_name参数来设置详细字段名.  

### 关系
在关系型数据库中,表间的关系是很重要的.django提供了三种主要的数据库关系模式:1对1,多对1,多对多.  

#### 多对1关系
要定义一个多对1的关系,可以使用django.db.models.ForeignKey.像普通的field类型一样设置为类属性即可.  
ForeignKey的第一个参数为:需要关联的model类.  
例如,一种汽车对应一个工厂,而一个工厂对应多种汽车.就像下面这样:  
```python
from django.db import models

class Manufacturer(models.Model):
    # ..
    pass

class Car(models.Model):
    manufacturer = models.ForeignKey(Manufacturer, on_delete=models.CASCADE)
```
也有一些关系类型是与自身一对多,称为recursive relationship(递归关系).此时可以使用: `models.ForeignKey('self', on_delete=models.CASCADE)`  
这里涉及到,当相关联的类为定义时如何进行关联?答案是可以直接填写类的名字:  
```python
from django.db import models

class Car(models.Model):
    manufacturer = models.ForeignKey(
        'Manufacturer',
        on_delete=models.CASCADE,
    )
    # ..

class Manufacturer(models.Model):
    # ..
    pass
```
至于要与其他app内定义的model相关联的话,可以像这样使用:  
```python
class Car(models.Model):
    manufacturer = models.ForeignKey(
        'production.Manufacturer',
        on_delete=models.CASCADE,
    )
```
这种关联被称为lazy relationship(懒联系),对于不同app之间的互相调用十分有用.  

#### 多对多关系
使用ManyToManyField来定义一个多对多关系.跟其他field的定义一样.  
ManyToManyField的第一个参数也是一个需要与之关联的model类.  
例如,一种披萨可以有多种配料,而一种配料可以用于多种披萨:  
```python
from django.db import models

class Topping(models.Model):
    # ..
    pass

class Pizza(models.Model):
    # ..
    toppings = models.MansToManyField(Topping)
```
跟ForeignKey一样,ManyToManyField也可以创建递归关系.  
两个model之间谁拥有ManyToManyField不是很重要,但只能单方带有该属性,不能两个都带.  
一般来说,需要在表单中编辑的才带有ManyToManyField.对于上面的例子,一般会在Pizza中加入toppings,因为我们一般只主要考虑披萨而不是主考虑配料.像上面这样设计的话,表单中就能让用户选择加什么配料了.  

#### 多对多关系中的扩展
若只是设计像披萨配料这种简单的多对多关系的话,普通的ManyToManyField就足以满足需求.但如果你想要将数据与两个model之间关联起来就需要一些额外的内容.  
