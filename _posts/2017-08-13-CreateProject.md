---
title: Django新建项目
category: Django
---
&emsp;&emsp;上一篇我们搭建完了django的开发环境,现在就可以来新建个django项目来检验下了.  
  
&emsp;&emsp;打开Pycharm(专业版才有各种模板,[后面](#j)会介绍没有专业版pycharm的创建方法),依次点击File->New Project,打开新建项目的页面如图:  
![新建项目][img_1]  
&emsp;&emsp;左侧选择Django,右侧Location选择项目要保存的位置,Interpreter选择原先创建好的虚拟环境.  

>如果Interpteter里找不到自己的虚拟环境的话可以点击右侧的齿轮,选择"Add Local",然后去anaconda的安装目录下的envs文件夹里找,里面有创建了的所有虚拟环境,找到装有django的那个,点击进去选择python.exe即可.  

&emsp;&emsp;然后直接点Create创建项目.可以看到创建好后的项目结构如下:  
>* mysite/
>   * templates/
>   * mysite/
>     * settings.py
>     * urls.py
>     * wsgi.py
>     * \_\_init__.py
>   * manage.py  

&emsp;&emsp;现在来运行下服务,点击右上角的绿色三角Run,可以看到控制台的输出如下:  
>D:\PyCharm\bin\runnerw.exe D:\Anaconda3\envs\django-learning\python.exe H:/Django/TestSite/manage.py runserver 8000
>Performing system checks...
>
>System check identified no issues (0 silenced).  
> 
>You have 13 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.  
>Run 'python manage.py migrate' to apply them.  
>August 12, 2017 - 22:09:23  
>Django version 1.11.4, using settings 'TestSite.settings'  
>Starting development server at http://127.0.0.1:8000/  
>Quit the server with CTRL-BREAK.  

&emsp;&emsp;现在打开浏览器,在地址栏上输入'http://127.0.0.1:8000'或者'http://localhost:8000',可以看到该项目成功运行:  
![django成功运行][img_2]  

---
&emsp;&emsp;接下来配置一下mysql数据库,在使用mysql数据库之前需要启动mysql的服务,打开cmd(可能需要管理员权限),输入以下命令:`net start mysql`(*这里的mysql是你原先填的mysql的服务名*),成功的话可以看到'MYSQL 服务已经启动成功'的字样.(关闭服务的命令为`net stop mysql`)  
&emsp;&emsp;然后打开navicat,点击连接,创建一个mysql的连接,填写:  
* 连接名: 随意
* 主机名: localhost
* 端口: 3306
* 用户名: root
* 密码: 原先设置的密码  

&emsp;&emsp;然后点击连接测试看成不成功(成功的话可以看到'连接成功'的提示),接着点'确定'就行了.  
![navicat连接][img_3]  
&emsp;&emsp;连接到数据库后,可以在右边看到该连接,右键它选择'新建数据库':  
* 数据库名: django_learning(任写)
* 字符集: utf8 -- UTF-8 Unicode
* 排序规则: utf8\_general\_ci  

> 字符集选择utf8可以很好的支持中文,排序规则常用的选择general\_ci,要记住这个里面是不区分大小写的,例如'Dog'与'dog'等同,所以不要用这种字段作为表的id,不然请选择utf8\_bin  

---
&emsp;&emsp;创建完数据库后,就要将该数据库与django项目进行绑定了.打开项目里mysite文件夹下的settings.py文件,找到以下字段:  
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
```  
&emsp;&emsp;这里默认的是使用python3自带的sqlite数据库,一个十分轻量级的数据库,如果不是很大的数据,对并发存储要求不高的话用这个也是可以的.现在我们将它改为使用我们创建好的mysql数据库.修改见以下:  
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'django_learning',
        'USER': 'root',
        'PASSWORD': '',
        'HOST': '127.0.0.1',
        'PORT': '3306',
    }
}
```  
* ENGINE: 选择mysql数据库
* NAME: 你创建的数据库名
* USER与PASSWORD按自己设定的填写(一般都是填root用户)
* HOST与PORT都是用的默认的  

&emsp;&emsp;设置完后我们来检验一下能否正常使用,由于我们在上一篇已经安装了mysqlclient,所以正常来说python可以调用mysql.在该项目目录下打开cmd,输入以下命令: `python manager.py shell`  
&emsp;&emsp;接着输入以下代码:  
```
from django.db import connection  
cursor = connection.cursor()  
```  
&emsp;&emsp;如果没有报错那说明数据库使用正常.  

---  
<span id="j"></span>
### 最后提一下如果没有专业版Pycharm的话如何新建项目:
&emsp;&emsp;django提供了很方便的创建项目的功能,首先打开cmd,cd到想要存放项目的位置,然后输入以下命令: `django-admin startproject mysite` ('mysite'可以换为任意的项目名)  
&emsp;&emsp;可以看到创建后的项目结构和用Pycharm创建的一致:  
> * mysite/
>   * manage.py
>   * mysite/
>     * \_\_init__.py
>     * settings.py
>     * urls.py
>     * wsgi.py  

&emsp;&emsp;然后是不使用pycharm的话如何运行项目呢,很简单:  
&emsp;&emsp;`python manager.py runserver`  

> 后面也可以带上自定义的参数,例如让其他电脑也可以访问网页(即使用0.0.0.0地址),或者改为自定义端口号, `python manager.py 0.0.0.0 9588` 即可在0.0.0.0:9588的地址开启服务.  

---  
### 如何新建项目就写到这里了~



[img_1]: /assets/article_img/2017-08-13/pycharm_new.png "Pycharm新建项目"
[img_2]: /assets/article_img/2017-08-13/django_run.png "django项目成功运行"  
[img_3]: /assets/article_img/2017-08-13/navicat.png "navicat连接测试"