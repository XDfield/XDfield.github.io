---
title: Scrapy学习:官方例子
category: Scrapy
img: Scrapy.jpg
---
[Scrapy](https://scrapy.org/)是一个强大的爬虫框架,现在开始来学习.  

---
## 安装
scrapy支持python2.7和python3.3+  
跟普通的python库安装一样,可以直接在命令行输入: `pip install scrapy`  
在windows下,如果有安装Anaconda或者Miniconda的,也可以输入: `conda install -c conda-forge scrapy`  

---
## 官方例子
这里是[官方文档](https://doc.scrapy.org/en/latest/)提供的一个学习例子,学习使用下.  
### 目的
爬取 *quotes.toscrape.com* 上的信息,该网站主要显示了一些知名作者写的话.  
该教程将会按一下顺序来完成:  
1. 创建一个新的Scrapy项目
2. 写一个spider来爬取一个站点并提取数据
3. 使用命令行来导出爬取到的数据
4. 调整spider来让它能够沿着页面顺序爬取
5. 使用spider的参数  

---
### 创建项目
在开始写爬取数据前,先创建一个Scrapy项目.在命令行输入: `scrapy startproject projectname`  
该命令会创建一个名为'projectname'的文件夹,该文件夹的结构如下:  
```  
projectname/
    scrapy.cfg  # 配置文件 
    projectname/  # 一个python包,你的代码应从这里导入
        __init__.py
        iterms.py  # 项目中对'物件'的定义
        pipelines.py  # 项目的pipelines
        setting.py  # 项目的设置
        spiders/  # 待会你的spider存放的位置
            __init__.py
```  

### 来写第一个Spider
Spider是我们来定义的一个类,Scrapy将会调用它来从网站上(或者是一堆网站上)抓取信息.该类必须是`scrapy.Spider`的子类,并且要定义最初的请求目标,至于定义如何按顺序爬取还有解析规则都是可选的.  
现在在projectname/spders文件夹里创建一个名为quotes_spider.py的文件,这将存放我们的第一个Spider的代码,内容如下:
```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'
    
    def start_requests(self):
        urls = [
            'http://quotes.toscrape.com/page/1/',
            'http://quotes.toscrape.com/page/2/'
        ]
        for url in urls:
            yield scrapy.Request(url=url, callback=self.parse)

    def parse(self, response):
        page = response.url.split('/')[-2]
        filename = 'quotes-%s.html' % page
        with open(filename, 'wb') as f:
            f.write(response.body)
        self.log('Save file %s' % filename)
```  
正如你所看到的,我们的Spider继承于`scrapy.Spider`,并且定义了一些属性与方法:  
* name: Spider的标识符.它在一个项目中应该是唯一的,也就是说对于两个不同的Spider不能有同样的name.
* start_requests(): 必须返回一个可迭代的Requests(可以是返回一个'请求'的列表或者一个生成器),用于让Spider开始爬取.随后,接下来要的请求会从这些初始的请求中连续生成.  
* parse(): 在每一个请求得到响应后,将会调用该方法来进行处理.它会接收一个TextResponse的实例为参数,该实例包含了响应页面的主体.  
> parse()方法通常会用来解析响应页面,提取需要爬取的信息,还有寻找接下来要爬取的链接并为他们创建新的请求.  

### 运行Spider
要让spider开始工作,只需回到我们的项目目录,并在命令行输入: `scrapy crawl quotes`  
该命令会运行我们刚才创建的名为`quotes`的spider,它会发送一个域名为'quotes.toscrape.com'的请求,在命令行内可以看到相应的输出信息.  
运行完后,检查当前目录下,会生成两个新的文件:'quotes-1.html'和'quotes-2.html',其内容为各个请求的响应,正如我们的`parse()`方法所写的.  
> 现在还未开始解析得到的html页面,稍后会进行这方面的操作.  

### 爬取过程的大致流程
Scrapy处理了我们的Spider中的`start_requests`方法所返回的`scrapy.Request`对象.在接收到响应后,实例化一个`Response`对象,然后调用与请求相关联的回调函数(在这个例子中,就是`parse`方法),将响应传递过去.  

### 关于start_requests方法的简化
我们可以定义一个带有一系列url值,名为`start_urls`的列表作为类属性,而不用实现一个`start_requests`方法来生成各个url的`scrapy.Request`对象.这个列表中的值将会被默认的`start_requests`方法使用并在你的spider中创建初始请求:  
```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        # balabalaba...
```  
至于`parse()`方法,即使我们没有自己将它注册为各个请求的回调函数,Scrapy还是会自己调用它来处理请求,因为Scrapy默认的就是调用名为`parse()`的方法.  

### 提取数据
先在Scrapy的终端界面测试下选择器的使用.命令行输入: `scarpy shell 'http://quotes.toscrape.com/page/1/'`  
> **注意:** 在输入url的时候最后要记得闭合(带上'/'),否则会有各种问题.  
在Windows上,使用双引号: `scrapy shell "http://quotes.toscrape.com/page/1/"`  

接着在终端可以看到一些响应的输出.接下来可以使用CSS来对`response`对象进行元素选择:  
```python
>> response.css('title')
[<Selector xpath='descendant-or-self::title'data='<title>Quotes to Scrape</title>'>]
```  
运行`response.css('title')`的结果会返回一个列表类型的对象`SelectorList`,它呈现了包含着XML/HTML元素的一些`Selector`对象,可以对它进行进一步的选择提取.  
要提取标题内部的文字,可以这么做:  
```python
>> response.css('title::text').extract()
['Quotes to Scrape']
```  
有两点需要注意:  
1. 在css的提取中添加了'::text',这意味着只提取\<title\>元素内部的文字.如果不加的话会返回整个元素,包括它的标签:
```python
>> response.css('title').extract()
['<title>Quotes to Scrape</title>']
```
2. 调用`extract()`会返回一个列表,因为处理的是一个`SelectorList`对象.如果只想要提取第一个结果,在该例子中,可以直接使用:
```python
>> response.css('title::text').extract_first()
'Quotes to Scrape'
```
或者是:  
```python
>> response.css('title::text')[0].extract()
'Quotes to Scrape'
```
> 不过还是推荐使用`extract_first()`方法,可以避免发生`IndexError`错误,即使是在未找到匹配的元素的情况下,也会返回None.

对大多数爬取代码来说,若在页面上未发现要找的信息我们就需要对这些错误更加包容,即使是爬取失败,我们也至少得到一些信息.  
除了使用`extract()`和`extract_first()`方法,还可以使用`re()`方法来通过正则表达式进行提取:  
```python
>> response.css('title::text').re(r'Quotes.*')
['Quotes to Scrape']
>> response.css('title::text').re(r'Q\w+')
['Quotes']
```
为了寻找合适的css选择器,可以输入`view(response)`,在默认的浏览器上打开该页面,然后使用浏览器上的开发者工具进行查看.  

### XPath的简单介绍
除了css选择器,scrapy也支持xpath语句:  
```python
>> response.xpath('//title')
[<Selector xpath='//title'data'<title>Quotes to Scrape</title>'>]
>> response.xpath('//title/text()').extract_first()
'Quotes to Scrape'
```
XPath语句十分强大,也是Scrapy选择器里使用的内部语句.实际上,CSS选择也是转变为XPath.可以在先前css选择的结果中看出来.  
虽然不像css那么流行,但XPath语句提供了更强大的功能,除了按元素构造进行导航,还可以解析元素的内容.使用XPath,可以这么选: **选择包含'下一页'的链接**.这种便利使得XPath十分适合用于爬取工作,即使已经学会了css选择器的用法也还是推荐学习下XPath的语法.  

### 提取论点与作者信息
现在已经知道了如何选择与提取,接下来完善我们的spider.  
在'http://quotes.toscrape.com'上的每个页面的HTML元素大致如下:  
```html
<div class='quote'>
    <span class='text'> "The world as we have created it is a process of our thinking.It cannot be changed without changing our thinking."</span>
    <span>
        by<small class='author'>Albert Einstein</small>
        <a href='/author/Albert-Einstein'>(about)</a>
    </span>
    <div class='tag'>
        Tag:
        <a class='tag' href='/tag/change/page/1'>change</a>
        <a class='tag' href='/tag/deep-thoughtes/page/1'>deep-thoughtes</a>
        <a class='tag' href='/tag/thinking/page/1'>thinking</a>
        <a class='tag' href='/tag/world/page/1'>world</a>
    </div>
</div>
```
跟原先一样,打开终端来分析如何提取: `scrapy shell 'http://quotes.toscrape.com'`  
先提取'quote'元素:  
```python
>> response.css('div.quote')
```
每次选择器会返回一个对象,让我们进一步进行元素细分.所以我们先将提取到的整个'quote'元素保存到一个变量中,下面可以直接使用这个变量来提取:  
```python
>> quote = response.css('div.quote')[0]
```
接下来提取'title','author'和'tag':  
```python
>> title = quote.css('span.text::text').extract_first()
>> title
'"The world as we have created it is a process of our thinking.It cannot be changed without changing our thinking."'
>> author = quote.css('small.author::text').extract_first()
>> author
'Albert Einstein'
```
由于'tag'是有多个,我们可以使用`extract()`来提取全部:
```python
>> tag = quote.css('div.tags a.tag::text').extract()
>> tag
['change', 'deep-thoughts', 'thinking', 'world']
```
知道了如何提取所有信息后,可以将他们整合起来并放到一个python的字典中:  
```python
>> for quote in response.css('div.quote'):
...     text = quote.css('span.text::text').extract_first()
...     author = quote.css('small.author::text').extract_first()
...     tags = quote.css('div.tags a.tag::text').extract()
...     print(dict=(text=text, author=author, tags=tags))
{'tags':['change', 'deep-thoughts', 'thinking', 'world'],'author':'Albert Einstein','text':'"The world as we have created it is a process of our thinking.It cannot be changed without changing our thinking."'}
{...}
...
```

### 在spider中提取数据
回到我们的spider中,直到现在它都没有提取什么特别的数据,只是将整个HTML页面保存在本地.我们将上面的代码整合到spider中.  
典型的Scrapy的spider会生成许多包含有从页面中提取出来的数据的字典.我们可以在回调函数中使用`yield`关键字来达到这个目的:  
```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').extract_first(),
                'author': quote.css('small.author::text').extract_first(),
                'tags': quote.css('div.tags a.tag::text').extract(),
            }
```

### 储存爬取到的数据
存储爬取的到的数据,最简单的方法是直接导出保存.可以在运行脚本时如下输入: `scrapy crawl quotes -o quotes.json`  
这将会生成一个quotes.json的文件,其包含所有抓取到的数据,以json的格式存储.  
基于一些历史原因,Scrapy会在提供的文件上添加内容而不是重写它的内容.如果你使用这条命令两次而没有移除第一次生成的json文件,这将导致第二次的json文件出错.  
为此,可以选择使用另一种格式,像是[JSON Lines](http://jsonlines.org):  
`scrapy crawl quotes -o quotes.jl`  
JSON Lines格式十分好用,他是一种数据流的形式,你可以很方便的在原有数据后面添加新的数据.即使运行同一个命令两次,也不会遇到像JSON那样的问题.同时,因为每一个记录都为独立的一行,你可以读写一个大文件而不用将它整个放到内存里,有很多像是[JQ](https://stedolan.github.io/jq)这样的工具能在命令行里使用.  
在一些小型的项目(就像我们这个)里,这样就够了.但,如果你想要适应更加复杂的情况,可以写一个`Item Pipeline`.如果只是想要存储一些爬取的数据,倒可不必关心这些.  

### 爬取接下来的链接
不止是爬取两页,我们一般更想要爬取一个网站上的所有页面.  
我们已经知道如何提取数据了,现在来看看如何设置接下来要爬取的方法.  
要爬取下一页内容,首先就要找到'下一页'的链接.还是看我们这个例子,可以看到在页面中有这样一段标明了下一页的链接:  
```html
<ul class='pager'>
    <li class='next'>
        <a href='/page/2/'>Next <span aria-hidden='true'>&rarr;</span></a>
    </li>
</ul>
```
在终端中进行如下尝试:  
```python
>> response.css('li.next a').extract_first()
'<a href="/page/2/">Next <span aria-hidden="true">-></span></a>'
```
这将得到整个a元素,但我们只想要他的`href`属性.为此,Scrapy提供一个CSS的方法扩展来让用户选择属性的内容,就像这样:  
```python
>> response.css('li.next a::attr(href)').extract_first()
'/page/2/'
```
现在让我们的Spider来自动获取下一页的链接并进行爬取:  
```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').extract_first(),
                'author': quote.css('small.author::text').extract_first(),
                'tags': quote.css('div.tags a.tag::text').extract(),
            }

        next_page = response.css('li.next a::attr(href)').extract_first()
        if next_page is not None:
            next_page = response.urljoin(next_page)
            yield scrapy.Request(next_page, callback=self.parse)
```
现在,在提取完页面的数据后,`parse()`方法会寻找下一页的链接,通过`urljoin()`方法构造绝对路径url,然后进行新的页面的请求,将自己注册为回调函数来处理下一页的数据提取以此来顺序抓取所有页面.  
这就是Scrapy中顺序抓取的大致过程:当在回调函数中返回一个请求,Scrapy将会安排这个请求的发送与回调函数的注册.  
通过这种方法,就可以自定义规则来构造复杂的爬虫代码,并解析不同类型的数据.  
这个例子中,会产生多重循环来抓取所有页面直到结束--对于抓取博客,论坛或者其他带分页的站点就十分方便.  

### 生成请求的简化
我们可以使用`response.follow`来简化生成请求的操作:  
```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').extract_first(),
                'author': quote.css('small.author::text').extract_first(),
                'tags': quote.css('div.tags a.tag::text').extract(),
        
        next_page = response.css('li.next a::attr(href)').extract_first()
        if next_page is not None:
            yield response.follow(next_page, callback=self.parse)
```
跟`scrapy.Request`不同,`response.follow`方法支持直接传进相对路径url--不需要调用`urljoin`方法.需要注意的是,`response.follow`方法只会返回一个请求的实例,所以还要来发出这个请求.  
你也可以传入一个选择器给`response.follow`来代替字符串;该选择器将会提取必要的属性:  
```python
for href in response.css('li.next a::attr(href)'):
    yield response.follow(href, callback=self.parse)
```
> 针对\<a>元素,`response.follow`会自动寻找它的`href`属性.所以该代码可以简化为:
```python
for a in response.css('li.next a'):
    yield response.follow(a, callback=self.parse)
```
> **注意:** `response.follow(response.css('li.next a'))`这样写是不对的,因为`response.css`返回的是一个列表对象而不是单独一个选择器.使用for循环或者像是`response.follow(response.css('li.next a')[0])`才是对的.  

### 更多例子
接下来是另一个spider,它指明不同的回调函数与下一页链接,对作者信息进行抓取:  
```python
import scrapy

class AuthorSpider(scrapy.Spider):
    name = 'author'
    start_urls = ['http://quotes.toscrape.com/']

    def parse(self, response):
        # 爬取作者页面
        for href in response.css('.author + a::attr(href)'):
            yield response.follow(href, callback=self.parse_author)

        # 爬取下一页
        for href in response.css('li.next a::attr(href)'):
            yield response.follow(href, callback=self.parse)
        
    def parse_author(self, response):
        def extract_with_css(query):
            return response.css(query).extract_first().strip()

        yield {
            'name': extract_with_css('h3.author-title::text'),
            'birthdate': extract_with_css('.author-born-date::text'),
            'bio': extract_with_css('.author-description::text'),
        }
```
这个spider将会从主页面开始,每次遇到指向作者页面的链接时就调用`parse_author`方法进行处理,遇到下一页的链接时则调用`parse`方法进行处理.  
这里我们将回调函数传入`response.follow`来让代码更简洁,使用`scrapy.Request`也是可以的.  
回调函数`parse_author`内部定义了一个辅助函数用来提取与清理从css选择器得到的数据,并生成一个python的字典来保存作者的信息.  
还有一件很有趣的事情就是,即使许多页面是出自同一个作者,我们也不用担心会多次访问到相同的作者页面.Scrapy会自动过滤掉已经访问过了的重复的页面,避免这种程序上的错误导致的多次访问服务器.这项设置可以通过更改`DUPEFILTER_CLASS`来配置.  

### 使用一些spider的参数
在运行爬虫时,可以加入`-a`参数:  
`scrapy crawl quotes -o quotes-humor.json -a tag=humor`  
这些参数会传递到spider的初始化方法中作为spider的默认属性.这这个例子中,通过tag传入的值会通过`self.tag`发生作用,可以使用这个方法来指定特定的tag下的quote:  
```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'

    def start_requests(self):
        url = 'http://quotes.toscrape.com/'
        tag = getattr(self, 'tag', None)
        if tag is not None:
            url = url + 'tag/' + tag
            yield scrapy.Request(url, self.parse)

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').extract_first(),
                'author': quote.css('small.author::text').extract_first(),
                'tags': quote.css('div.tags a.tag::text').extract(),
        
        next_page = response.css('li.next a::attr(href)').extract_first()
        if next_page is not None:
            yield response.follow(next_page, callback=self.parse)
```
若将`tag=humor`参数传入spider,它将只抓取带有'humor'标签的页面,比如:'http://quotes.toscrape.com/tag/humor'.

---
## 结尾
关于Scrapy的官方例子学习就到这里,本篇只涉及了scrapy的基础使用,关于一些高级方法的使用学习有空再更吧.