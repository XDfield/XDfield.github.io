---
title: Django开发环境搭建
category: Django
img: django.jpg
---
## 使用的软件:  
* 环境: Windows 10
* 语言: python3.5(安装Anaconda)
* IDE: Pycharm
* 数据库: MySQL
* 数据库管理工具: navicat
* web框架: Django

---  

## 软件安装:  

### 1.Anaconda安装:  

&emsp;&emsp;[Anaconda][ana]为python的一个发行版,包含了常用的Python库,自带conda命令工具,十分好用.现在anaconda最新版是自带python3.6的,如果想要使用python3.5版本,官方给出了三种途径:  

>1.安装最新版的anaconda后,重新创建一个python3.5的环境,具体方法[链接][py3env]  
>2.安装最新版的anaconda后,使用以下命令直接安装Python3.5:&ensp;`conda install python=3.5`  
>3.直接安装默认带python3.5版本的anaconda,即4.2.0版本(各个版本的下载地址在[这里][download_ana]),按自己的操作系统选择即可  

&emsp;&emsp;下载得到anaconda的安装包,安装完成后打开cmd,输入`python`,回车,显示如下即python安装成功:  
![python安装测试][img_1]  
(我自己用的是3.5.2版本)  

### 2.Pycharm安装:  

&emsp;&emsp;[Pycharm][pycharm]是JetBrains出品,专门写Python的IDE,有免费的社区版(Community)与收费的专业版(Professional),如果是新人入手练习感觉社区版也够用了,但有些专业要求的话还是安装[专业版][pro_ver]吧(可以直接新建django项目,用起来蛮省心,缺点就是比较占内存)  
&emsp;&emsp;那专业版收费怎么办呢,作为高校学生福利,如果有edu邮箱的话可以注册个账号,可以免费使用JetBrains的全家桶(申请edu账号[教程][edu]).简单粗暴的windows软件安装流程,没有什么好说的.

### 3.MySQL安装:  

&emsp;&emsp;[MySQL][mysql]是一个常用的数据库,安装也简单,我是win10就直接下载它的[Install][mysql_install]了,安装过程中会提示填写mysql服务的名字,最好记一下方便以后使用,然后是为root用户设置密码,这个密码要记住,蛮重要的.  
>&emsp;&emsp;特别提醒:win10下载的install程序有可能会有个bug(我安装的时候就遇到了),会有底下的按钮显示不了的,估计是系统字体的大小设置导致的,不过问题也不大,调出英文输入,通过X,C,B,N四个键可以进行控制输入.  

&emsp;&emsp;安装程序最后会自动帮你初始化数据库,以后要使用的时候自己打开Mysql服务就行了.(mysql默认的连接地址为'localhost:3306')  
&emsp;&emsp;最后找到mysql server的安装目录,将他的bin文件夹添加到系统的环境变量PATH中.

### 4.navicat安装(非必需):  

&emsp;&emsp;[navicat][navicat]是个数据库管理工具,有个gui客户端来管理自己的数据库还是蛮方便的,依旧简单粗暴的windows软件安装流程,没啥可说的.(自带中文版本,因为我们使用的是mysql数据库所以只安装navicat fot mysql就可以了)  

### 5.Django安装:  

&emsp;&emsp;在安装[Django][django]之前我们最好是先创建一个虚拟环境,以后所有关于Django的项目都在这里面进行,可以很好的与原先的python环境区分开.  
&emsp;&emsp;由于安装了anaconda,创建虚拟环境就十分简单,直接在cmd里输入:  
&emsp;&emsp;`conda create -n 你的虚拟环境名 python=3.5`  
&emsp;&emsp;(例如我的虚拟环境名字叫django-learning就输入`conda create -n django-learning python=3.5`)  
&emsp;&emsp;接着它会问你`Proceed ([y]/n)?`输入y回车确认就可以了.这样就创建了一个干净的python虚拟环境.然后使用`activate 你的虚拟环境名`来激活,可以看到类似如下显示:  
![进入虚拟环境][img_2]  
&emsp;&emsp;成功进入虚拟环境后就可以通过pip安装Django了(pip也是python的一个包管理工具),直接输入:  
&emsp;&emsp;`pip install Django`(若要指定Django的版本则输入`pip install Django==1.9.8`即可安装1.9.8版本的Django)  
&emsp;&emsp;对了,还要安装[mysqlclient][mysqlclient],用来驱动mysql可以被python调用,原先常用的是[mysqldb][mysqldb],但截止到目前(17年8月)mysqldb还不支持python3.5,所以只能用mysqlclient了.同样用pip安装(在同一个虚拟环境下):  
&emsp;&emsp;`pip install mysqlclient`

>一些常用的conda命令:  
>`conda list`&emsp;查看当前环境下已安装的包,anaconda中将python本身也当作一个包  
>`conda env list`&emsp;查看所有的虚拟环境  
>`activate 虚拟环境名`&emsp;激活指定的虚拟环境  
>`deactivate`&emsp;退出当前的虚拟环境  

---
### 到这里所有软件就都安装结束啦~  


[ana]: https://www.continuum.io/downloads "anaconda官网"
[py3env]: https://conda.io/docs/py2or3.html "anaconda创建python环境"  
[download_ana]: https://repo.continuum.io/archive/ "anaconda各个版本下载地址"  
[pycharm]: https://www.jetbrains.com/pycharm/ "Pycharm官网"  
[pro_ver]: https://www.jetbrains.com/pycharm/download/#section=windows "Pycharm下载"
[edu]: null "有时间再写"
[mysql]: https://www.mysql.com/ "MySQL官网"
[navicat]: https://www.navicat.com/en/products/navicat-for-mysql "navicat"
[mysql_install]: https://dev.mysql.com/downloads/installer/ "MySQL Install"  
[mysqlclient]: https://pypi.python.org/pypi/mysqlclient "mysqlclient"  
[mysqldb]: http://mysql-python.sourceforge.net/MySQLdb.html "mysqldb"
[django]: https://www.djangoproject.com/ "django官网"  
  
[img_1]: /assets/article_img/2017-08-11/python_test.png "python测试截图"
[img_2]: /assets/article_img/2017-08-11/env.jpg "进入虚拟环境"