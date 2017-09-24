---
title: 从一个表单例子引出Django
category: Django
img: django.jpg
---  
{% raw %}  
&emsp;&emsp;这次我们从一个表单的例子引入Django的大致开发过程.其中主要涉及到表单与数据库之间的数据存取.  

---
### 目录结构
&emsp;&emsp;上一篇我们实现了Django项目的新建,现在来看看它的目录结构:  
> * projectname/ (项目的主目录)
>   * projectname/ (主要配置文件的存放地址)
>     * \_\_init__.py (来将该文件夹视为一个包)
>     * setting.py (全局配置)
>     * urls.py (Django主要的urls配置入口)
>     * wsgi.py (Django启动的wsgi文件,暂时不管)
>   * templates/ (主要的html文件模板)
>   * manage.py (启动Django的主要文件)  

&emsp;&emsp;除了这些文件夹之外,还有一些常用的文件夹需要手动创建:  
> * app\ (各种独立出来的app应用)
> * static\ (存放静态文件,即css js 或者主要的图片)
> * log\ (存放运行日志)
> * media\ (一般用来存放用户上传的文件)  
> ps: *若以后app多了,全部放在项目的根目录会比较乱,可以再创建一个apps文件夹来存放各个app(记得在里面放上一个\_\_init__.py文件)*  

&emsp;&emsp;注意,如果是用Django自带的命令创建的项目,则不会自带templates文件夹,所以即使手动创建了也需要配置一下,static文件夹也是需要进行配置.  
&emsp;&emsp;打开settings.py文件,找到TEMPLATES,为一个list,里面放着一个dic,而其中的'DIRS'段即为自定义模板的保存位置,修改为:  
```
'DIRS': [os.path.join(BASE_DIR, 'templates')],
```
> 可见其实'BASE_DIR'是该文件定义好了的项目根目录位置.  

&emsp;&emsp;然后是static文件夹,在settings.py文件最后补上:
```
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'static'),
]
```
&emsp;&emsp;即可指引到根目录下创建好的那个static文件夹.

---
### 创建app
&emsp;&emsp;利用Pycharm可以很方便的创建app.依次点击菜单栏上的 Tools->Run manage.py Task... 然后在打开的窗口下输入`start appname`即可创建指定名称的app.  
> 若是没有使用Pycharm的用户,则可以通过cmd,在manage.py的同目录下,输入: `python manage.py startapp appname`  

&emsp;&emsp;这里我们的例子是个表单,就创建一个名为Form的app,创建好后可以看到app的目录结构如下:  
> * Form/
>   * migrations/
>     * \_\_init__.py
>   * \_\_init__.py
>   * admin.py
>   * apps.py
>   * models.py
>   * tests.py
>   * views.py  

&emsp;&emsp;创建完app后记得在setting.py里注册一下.在INSTALLED_APPS这个list里补上'message',不然该项目不会识别这个app,不方便后面更新数据库.

---
### 创建表单页面
&emsp;&emsp;这里简单编写一个html表单页面,然后简单配置一下css文件检验下效果.新建一个form.html文件,内容如下:  
```
<!DOCTYPE html>
<html>
<head>
    <title>form test</title>
</head>
<form action="" method="post">
    <table align="center">
        <caption>Test Form</caption>
        <tr>
            <td>name:</td>
            <td><input type="text" name="name" /></td>
        </tr>
        <tr>
            <td>password:</td>
            <td><input type="text" name="password" /></td>
        </tr>
        <tr>
            <td>email:</td>
            <td><input type="text" name="email" /></td>
        </tr>
        <tr>
            <td colspan="2" align="center">
                <input type="submit" value="Submit" />
            </td>
        </tr>
    </table>
</form>
</html>
```  
&emsp;&emsp;然后是style.css文件(简单的配置下表单):  
```
form {
    border:1px solid #000;
    border-radius: 5px;
    margin: 0 auto;
    max-width: 300px;
}
```  
&emsp;&emsp;然后将form.html文件放到templates文件夹中,style.css文件放到static\css文件夹中(css文件夹新建,用一个文件夹保存同个类型的文件比较方便).然后在form.html中添加对style.css文件的引用,在\<head>里添加:  
```
<link rel="stylesheet" href="\static\css\style.css">
```
> 记住路径一定是'\\'开头  

---
### 配置该页面  
&emsp;&emsp;要绑定这个html文件,就需要进urls.py文件里绑定与之对应的url规则.而一个url规则需要对应一个view方法,在对应app里定义.  
&emsp;&emsp;现在新增一个view方法,在message这个app里的views.py里新增以下代码:  
```
def getForm(request):
    return render(request, 'form.html')
```  
&emsp;&emsp;这就创建好了一个view方法,他的第一个默认参数是一个WSGIRequest对象,即一个对应的网页请求,这里直接返回原先定义的那个html文件,使用render()方法.  
&emsp;&emsp;接下来填写urls.py,新增一个url绑定,在urls.py文件的urlpatterns列表里新增:  
```
url(r'^form/$', views.getForm, name='getForm'),
```  
> 文件开头要记得import一下message的views文件

&emsp;&emsp;url的填写顺序: 1.一段正则表达式用来匹配输入的url('^'表示从这里开始,'/$'表示从这里结束);2.对应的view方法;3.可以给该url指定一个名称(可选但最好带上)  
> 这里新增了url后,原先默认的起始页就404了  

&emsp;&emsp;现在启动django项目,地址输入 `http://127.0.0.1:8000/form` ,正常的话可以看到原先建立的表单页面了.  
![表单][form]  
> 可见css文件有正常导入

---
### 创建model  
&emsp;&emsp;既然是表单,就涉及到数据的存取,也就涉及到数据库的操作.在开发环境搭建那一篇文章里装过mysql的驱动,这里就不用装了.  
&emsp;&emsp;有了数据库就得创建model模型,这就涉及到django的orm模型.  
> 这里提一下一般的数据库操作流程:
> ```
>def book_list():
>	db = mysqldb.connect(user='', db='', passwd='', host='')
>	cursor = db.cursor()
>	cursor.excute('SELECT name FROM books ORDER BY name')
>	names = [row[0] for row in cursor.fetchall()]
>	db.close()
>```
> 即连接数据库->创建cursor->执行sql语句->取回值, 而orm则是将数据库的表结构映射为一个类来操作,以类的方式来完成这些繁琐的工作.  

&emsp;&emsp;现在来创建对应的model模型,在message的models.py文件里新增以下代码:  
```
class User(models.Model):
    name = models.CharField(max_length=10, verbose_name='用户名')
    password = models.CharField(max_length=50, verbose_name='密码')
    email = models.EmailField(verbose_name='邮箱')

    class Meta:
        verbose_name = '用户信息'
```  
&emsp;&emsp;这里新增了一个User类用来定义与操作对应的数据,按照原先定义的表单的内容这里定义了name, password, email三个变量,其中按他们应有的类型选用了不同的模型字段(Field).  
&emsp;&emsp;([这里][Field]是官方各种字段(Field)类型的文档.)
> CharField顾名思义为字符串字段,name与password都属于这一类,注意该字段有个必填的属性为max\_length,而verbose\_name为对该变量的注释;而email也有专门的字段EmailField.定义完一个model类后要定义一个Meta信息,其中为对该类型(User)的注释定义  

&emsp;&emsp;现在运行命令 `makemigrations message` (在Tools->Run manage.py Task...里)生成数据库更新记录,如果没问题就再输入 `migrate message` .可以用navicat看表是否创建成功.  
> 生成表的默认命名结构为: appname_modelname (全为小写)  

---
### 数据表的增删改查
&emsp;&emsp;最后让表单可以与数据库进行数据存取.在views.py中导入models.py文件,然后就可以在view方法中对该数据表进行实例化.  
> 例如:  
> `all = User.objects.all()`  
> 这里的'objects'为一个默认的数据表管理器,它的'all()'方法会返回全部的数据,且可迭代.得到的实例,可以使用类似item.name的方法来取得数据.  
> 'objects'还有个方法'filter()'可用来筛选数据.  
> 使用方法: `pp = User.objects.filter(name='pp')`  
> 而存储数据可以通过创建一个User对象并赋上各个值,最后调用'save()'方法进行保存.删除数据则通过'delete()'方法.  

&emsp;&emsp;现在要让html页面可以提交数据,必须在html文件里写上 `{% csrf_token %}` ,不然django不会让他提交,这是一种安全机制.  
&emsp;&emsp;而表单的 `action` 属性则需要填上要提交的url,这里还是form.html自己,而先前我们给它定义了一个名称为 'getForm' ,所以这里我们可以填为:  
```
<form action="{% url 'getForm' %}" method="post">
```
> 这样以后我们修改form的url规则的话,就不用连同html文件内部一起修改了.  

&emsp;&emsp;现在我们修改下getForm这个view方法,实现提交后数据可以保存到数据库:  
```
def getForm(request):

    if request.method == 'POST':
        name = request.POST.get('name', '')
        password = request.POST.get('password', '')
        email = request.POST.get('email', '')
        user = User()
        user.name = name
        user.password = password
        user.email = email
        user.save()

    return render(request, 'test.html)
```  
> 这里用 `request.method` 来判断进来的是post请求还是get请求(默认为get请求,当表单提交的时候是post请求),post时该request对象带有一个POST属性,此为一个dict,包含着填入的数据.用相应的get方法取得数据并赋值给新建的User对象并save一下就行了.  
> 可以试试填入数据并提交然后用navicat来检查是否已经保存进数据库.  

---
### 这个表单的例子就写到这里了~  
{% endraw %}

[Field]: https://docs.djangoproject.com/en/1.11/ref/models/fields/ "Field type"  
[form]: /assets/article_img/2017-08-19/form.png "表单页面"