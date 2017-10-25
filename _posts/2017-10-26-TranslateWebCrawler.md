---
title: 500行构造异步爬虫(个人翻译)
category: Scrapy
img: Scrapy.jpg
---
## 一个简单的异步爬虫
本文是"500行项目"中的异步爬虫项目的个人翻译,主要是出于学习目的.原文地址在[这里][link].  

---
### 介绍
传统的计算机科学强调的是有效的算法,来尽可能快的完成计算.但是很多网络程序花费的时间主要不是在计算上,而是保持开放大量的连接而导致的慢.这就提出了一个很有挑战性的问题:如何有效的等待大量的网络事件.现在针对这个问题比较有效的方案就是使用异步I/O,或称"async".  
  
这里我们要介绍一个简单的爬虫程序.这个爬虫只是一个异步程序框架,它只等待许多响应(response),但几乎没有什么计算.一次获取的页面越多,它能够越快的完成.如果它将每一个线程对应用于一个正在运行的请求,随着并发请求数量的增多,它将在耗光套接字(socket)之前把内存或线程相关的资源耗尽.而异步I/O就避免了对线程的依赖.  
  
我们的例子将分为三个阶段.第一阶段,我们会展示一个异步的事件循环,并构造一个爬虫来使用事件循环与回调(callbacks):这将很有效,但要将它扩展到更复杂的问题时将会催生一堆不好管理的代码.第二阶段,我们将因此引出python的协程(coroutines),它也是高效且可扩展的.在python中,我们将利用生成器来构造简单的协程.最后阶段,我们使用python的标准库"asyncio"里的功能完整的协程,并使用异步队列来进行协调.  

---
### 任务目标
一个爬虫程序所要做的就是在网站上查找并下载所有的页面,或许还需要进行归档或索引.从根url开始,它会抓取每个页面,解析它来查看其他未知的链接,并将其添加到队列中.直到它抓取到的页面里没有未获取的链接并且队列未空的时候才停止.  
  
我们可以通过同时下载多个页面来达到加速这个过程的目的.随着爬虫找到新的链接,它将同时在一个新的socket上启动对新页面的爬取工作.它在响应到达时进行解析并将新的链接添加到队列里.其中太多的并发数可能会降低性能,所以我们要限制一下并发请求的数量,并将剩余的链接留在队列中,直到一些正在运行的请求完成.  

---
### 比较传统的做法
如何做到爬虫的并发呢?我们一般是创建一个进程池(thread pool).每一个进程将会负责通过socket来下载一个页面.例如,来下载*xkcd.com*上的一个页面:  
```python
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {0} HTTP/1.1\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    whilt chunk:
        response += chunk
        chunk = sock.recs(4096)
    # 页面下载完成之后
    links = parse_links(response)
    q.add(links)
```  
一般来说,socket的操作是阻塞(blocking)的:当线程调用`connect`或者`recv`方法的时候,它会暂停直到操作完成.因此,为了一次性下载多个页面,我们需要许多线程.一个精明的程序会通过在线程池中保留空闲的线程来摊销线程创建的成本,然后对它们进行检查以便后续任务的循环利用;连接池中的socket也是如此.  
  
然而,线程是挺珍贵的,操作系统对程序,用户或者机器可能持有的线程数量将会强制执行限制.如果一个python程序启用了数以万计的线程那无疑会失败.如果我们在并发的socket上同时执行数万个并发的操作,那么在我们使用完socket之前,线程将会先被耗尽.线程的开销和系统对线程上限的限制将会是我们的瓶颈.  
  
诚然,我们的简易爬虫对线程的要求不会这么高.但对于一个非常大规模的应用程序来说,拥有成千上万的连接,将面临着这些限制:大多数系统依然可以继续创建socket,但线程已经被耗尽.我们该如何克服这个问题?  

---
### 异步(Async)
异步I/O框架在单一线程上利用非阻塞的socket来执行并发操作.我们先在与服务器建立连接之前将socket设置为非阻塞:  
```python
sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass
```  
需要注意的是,一个非阻塞的socket在连接时会抛出一个异常,即使它工作正常.这个异常来自于底层C函数的行为,只是提示你它已经开始运行了.  
  
现在我们的爬虫需要一个方法来知道连接什么时候被建立,从而可以发送http请求.我们可以简单的使用一个循环:  
```python
request = 'GET {0} HTTP/1.1\r\nHost: xkcd.com\r\n\r\n'.format(url)
encoded = request.encode('ascii')

while True:
    try:
        sock.send(encoded)
        break
    except OSError as e:
        pass

print('sent')
```  
这个方法不仅浪费电,而且不能有效的等待多个socket上的事件.以前,BSD Unix上对这个问题的解决方案是使用`select`,一个等待事件发生在非阻塞的socket上或者它们的一个小数组的C函数.现在,对具有大量连接需求的互联网应用有了代替品例如`poll`,或者BSD上的`kqueue`和Linux上的`epoll`.这些API跟`select`很像,但对拥有大量连接的应用支持更好.  
  
Python3.4的DefaultSelector使用系统上可用的最佳的`select`类功能.要注册一个关于网络I/O的通知,我们来创建一个非阻塞的socket并用默认的selector进行注册:  
```python
from selectors import DefaultSelector, EVENT_WRITE

selector = DefaultSelector()

sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass

def connected():
    selector.unregister(sock.fileno())
    print('connected!')

selector.register(sock.fileno(), EVENT_WRITE, connected)
```  
我们这里忽略了一个伪异常,调用`selector.register`,传入socket的文件描述符和一个表达我们等待的是什么事件的常数.通过传入`EVENT_WRITE`,当连接建立时我们会收到通知:就是我们想要知道这个socket是否是"可写的".当事件发生时,我们还传递了一个python方法`connected`来运行.这种功能被称为回调(callback).  
  
我们利用`selector`来接收这些I/O通知,构造一个循环:  
```python
def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()
```  
回调函数`connected`储存在`event_key.data`中,一旦非阻塞套接字建立了连接,我们便将它取回并执行.  
  
与原先的无脑循环不同的是,这里我们调用`select`来暂停,并等待下一个I/O事件.然后该循环调用设定好的等待处理这些事件的回调函数.尚未完成的操作将在事件循环的某个未来答复之前保持等待.  
  
到目前为止我们已经完成了什么呢?我们展示了如何开始一个操作,并且当操作就绪时执行他的回调函数.一个异步框架建立在我们已经展示的两个功能上--非阻塞socket与事件循环--从而在单个线程上实现并发操作.  
  
我们已经做到了"并发",但还没有达到传统意义上的"并行性".我们只是搭建了一个不断进行I/O操作的微小系统.我们刚开始有能力去执行新的操作,而别人已经在飞了.实际上,它并没有使用多核进行并行计算.但,这个系统是针对I/O问题设计的,而不是针对CPU的限制.  
  
因此我们还是可以说我们的事件循环在并发I/O中是高效的,因为它不会将一个线程资源只对应一个连接.在我们继续之前,需要纠正一个常见的误解,即异步比多线程要快.通常情况下,在Python中,像我们这样的事件循环在为少量非常活跃的连接提供服务时,是比多线程要慢的.在没有全局解释器锁(GIL)的限制条件下,线程将在这样的工作中表现的更好.那异步I/O在哪方面才有利呢?答案是当应用程序服务于许多缓慢或者延时的连接或者不寻常的事件的时候.  

---
### 利用回调函数来编程
随着我们这个异步框架的搭建,我们该如何去写一个网络爬虫呢?甚至是一个url提取也不好写.  
  
我们从两个全局的集合(set)开始,一个是即将抓取的urls,一个是我们已经看过的urls:  
```python
urls_todo = set(['/'])
seen_urls = set(['/'])
```  
seen\_urls这个集合包含了urls\_todo与已完成的url.这两个集合用根url'/'进行初始化.  
  
抓取一个页面需要一系列的回调函数.`connected`这个回调函数在socket连接时被触发,并且发送一个GET请求给服务器.但此时它需要等待服务器的响应,所以要注册另一个回调函数.当该回调函数触发时,它无法读取完整的响应,所以它将重新注册,等等.  
  
现在将这些回调函数整合到一个`Fetcher`类里.它需要一个url,一个socket,还有一个存放响应的response字节:  
```python
class Fetcher:
    def __init__(self, url):
    self.response = b''  # 一个空的字节
    self.url = url
    self.sock = None
```  
首先调用`Fetcher.fetch`:  
```python
# 写在Fetcher里的类方法
def fetch(self):
    self.sock = socket.socket()
    self.sock.setbloking(False)
    try:
        self.sock.connect(('xkcd.com', 80))
    except BlockingIOError:
        pass

    # 注册下一个回调函数
    selector.register(self.sock.fileno(), EVENT_WRITE, self.connected)
```  
`fetch`方法首先连接一个socket.但请注意,这个方法在连接建立前已经返回.它将控制权返回给事件循环来等待连接.要知道为什么这样做的话,不妨想象一下我们整个应用程序的结构如下:  
```python
# 来抓取 http://xkcd.com/353/ 页面
fetcher = Fetcher('/353/')
fetcher.fetch()

while True:
    events = selector.select()
    for event_key, event_mask in events:
        callback = event_kay.data
        callback(event_key, event_mask)
```  
当事件循环调用`select`时,所有事件通知将会被处理.于是,`fetch`必须将控制权交给事件循环来让程序知道socket什么时候连接.只有这样在事件循环中才能调用`connected`这个在`fetch`结束时注册的回调函数.  
  
以下是`connected`方法的实现:  
```python
# 该方法也是在Fetcher类中:
def connected(self, key, mask):
    print('connected!')
    selector.unregister(key.fd)
    request = 'GET {0} HTTP/1.1\r\nHost: xkcd.com\r\n\r\n'.format(self.url)
    self.sock.send(request.encode('ascii'))

    # 注册下一个回调函数
    selector.register(key.fd, EVENT_READ, self.read_response)
```  
该方法发送了一个GET请求.一般情况下,应该检查`send`的返回值,以防不能一次性发送整条消息.但我们的请求比较短而且我们的应用程序没那么复杂.它只是简单的调用`send`,并等待响应.当然,最后还得注册另一个回调函数,并且将控制权返还给事件循环.下一个也是最后一个回调函数`read_response`将用来处理服务器的应答:  
```python
# 该方法也是在Fetcher类中
def read_response(self, key, mask):
    global stopped

    chunk = self.sock.recv(4096)  # 每次读4k
    if chunk:
        self.response += chunk
    else:
        selector.unresgister(key.fd)  # 读取完成
        links = self.parse_links()

        # Python的set操作
        for link in links.difference(seen_urls):
            urls_todo.add(link)
            Fetcher(link).fetch()

        seen_urls.update(links)
        urls_todo.remove(self.url)
        if not urls_todo:
            stopped = True
```  
每当`selector`发现socket是"可读的"的时候(此时意味着两种情况:该socket有数据或者它被关闭),这个回调函数将被执行.  
  
这个回调函数向socket的数据请求大致4k个字节.如果数据不够4k,`chunk`将取得所有能取的数据.如果数据超过4k,`chunk`长度将会是4k字节且socket尚有数据剩余可读,所以事件循环将再一次调用这个回调函数.但响应完成后,服务器关闭连接,`chunk`将为空.  
  
还未展示的`parse_links`方法将会返回一个urls的集合.我们为每一次新的url创建一个`fetcher`,现在没有设置并行上限.异步程序与回调的机制有个很棒的功能:例如当我们往seen_urls里添加新的链接时,不需要去考虑关于共享的数据上的修改而产生的逻辑错误.不过还需注意到,此时我们的代码不能中断.  
  
所以我们加入一个全局变量`stopped`,使用它来控制循环:  
```python
stopped = False

def loop():
    while not stopped:
        events = selector.select():
        for event_key, event_mask in events:
            callback = event_key.data
            callback()
```  
一旦所有页面下载完成,`fetcher`将会停止这个事件循环,并结束该程序.  
  
这个例子也暴露出异步程序的一个问题:[面条式的代码][sopaghetii code](指的是源代码的控制流程复杂混乱).我们需要一些方法来表达一系列的计算与I/O操作,并且能够将多个这样的一系列操作调度并发运行.但没有线程的话,一系列的操作不能被集合到一个函数当中:每当一个函数开始执行一个I/O操作时,它将明确的保存将来可能会用到的任何"状态",然后返回.你应该多考虑并写下这部分保存状态的代码.  
  
让我们来看看这是什么意思.用常规的阻塞型socket,思考下如何简单地在一个线程上抓取一个url:  
```python
# 阻塞版本(Blocking)
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {0} HTTP/1.1\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)
    
    # 页面下载完成
    links = parse_links(response)
    q.add(links)
```  
在一个socket操作与下一个socket之间,该记住哪些状态呢?它拥有socket,url和response.在线程上运行的函数,使用编程语言的基本功能将这些临时的状态存储到其堆栈中的局部变量中.该函数还拥有一个"继续部分"--即在I/O操作完成后执行的代码.在运行时,通过存储线程的指令指针来记住这个"继续部分".你不需要去考虑恢复这些局部变量和I/O操作之后的"继续部分".这些都在编程语言里建立好了.  
  
但使用基于回调的异步框架的话,这些语言的功能就帮不上忙了.在等待I/O时,函数必须明确保存其状态,因为该函数在I/O完成之前就返回并丢失了堆栈帧.我们这个基于回调的例子将socket和response作为Fetcher实例的属性存储来代替局部变量;通过注册`connected`与`read_response`来存储他的"继续部分"来代替指令指针.随着应用程序的功能越来越复杂,我们在回调之间手动保存的状态也将变得复杂起来.这就让编写代码越发让人头疼.  
  
更糟糕的是,如果回调引发了异常,则在调用下一个回调之前会发生什么?它会说我们在`parse_links`这个方法里的工作做得不好,会抛出一个HTML解析的异常.  
  
而这样的报错信息仅显示事件循环正在运行回调.我们无法知道是什么导致的错误.整个回调链在两端都发生了错误:我们忘了我们从哪里来,要到哪里去.这种上下文的丢失被称为"栈的断裂(stack ripping)",在许多情况下会混淆我们.这个问题也阻碍了我们在一系列回调中添加异常处理,像是"try/except"的调用.  
  
因此,除了争论多线程与异步哪个效率更快,哪个更容易出错也是需要我们关心的:如果都有错误的情况下,线程更容易受到数据竞争的问题,而由于"栈的断裂"回调机制会固执与调试问题.(这段实在不知道该怎么翻译)  
>threads are susceptible to data races if you make a mistake synchronizing them, but callbacks are stubborn to debug due to stack ripping.  

---  
### 协程(Coroutines)
我们向你保证.将回调机制的效率性与多线程程序的传统代码条理性相结合是有可能的.这种结合需要通过一种称为"协程"的模式.利用Python3.4的标准库asyncio,与一个叫做aiohttp的包,用协程来抓取一个url将会非常直观:  
```python
@asyncio.coroutine
def fetch(self, url):
    response = yield from self.session.get(url)
    body = yield from response.read()
```  
它同样是可扩展的.与线程的开销相比,协程占用的内存会更小.Python可以轻易的开启数十万的协程.
  
与其他计算机科学相比,协程的概念比较简单:它是一个可以暂停和恢复的子程序.而线程由操作系统抢先多任务,协程协作多任务:他们选择何时暂停,与下一步运行哪个协程.  
  
有很多协程的实现,即使在Python自身中.在Python3.4的标准库"asyncio"中的协程是基于生成器,Future类以及"yield from"语句.从Python3.5开始,协程就作为语言的基本特征;然而,作为在Python3.4中预先存在的语言特性,理解协程是解决Python3.5中的本地协程程序的基础.  
  
为了解释Python3.4中基于生成器的协程,我们将展示下对生成器的探索,以及它们如何用于asyncio中的协程,相信你会喜欢上它的.我们在解释完基于生成器的协程后,再在异步爬虫中来使用它们.  

---
### Python的协程是如何工作的
在掌握Python的生成器之前,你需要了解常规的Python函数是怎么工作的.一般来说,当一个Python方法调用一个子方法的时候,该子方法保留控制权直到它返回或者抛出异常.这时控制权回到调用者这里:  
```python
>>> def foo():
...     bar()
...
>>> def bar():
...     pass
```  
标准的Python解释器是用C写的.执行Python函数的C函数叫做PyEval_EvalFrameEx.它需要一个Python堆栈帧(stack frame)的对象,并在帧的上下文中评估(evaluate)Python字节码(bytecode).以下是`foo`的字节码:
```python
>>> import dis
>>> dis.dis(foo)
    2       0 LOAD_GLOBAL       0 (bar)
            3 CALL_FUNCTION     0 (0 positional, 0 keyword pair)
            6 POP_TOP       
            7 LOAD_CONST        0 (None)
           10 RETURN_VALUE
```  
`foo`函数将`bar`加载到堆栈中并调用它,然后从堆栈中弹出(pop)其返回值,将None加载到堆栈上并返回None.  
  
当PyEval\_EvalFrameEx遇到CALL\_FUNCTION字节码时,它会创建一个新的Python堆栈帧并递归:即,用于执行`bar`的新帧递归地调用PyEval_EvalFrameEx.  
  
了解Python堆栈帧在堆内存中是非常重要的!Python的解释器是一个普通的C程序,所以它的堆栈帧是正常的堆栈帧.但它自己操作的堆栈帧是在堆之上的.除了其他的惊喜,这意味着一个Python堆栈帧可以超过其函数调用.要以交互的方式查看,请从`bar`内保存当前帧:  
```python
>>> import inspect
>>> frame = None
>>> def foo():
...     bar()
...
>>> def bar():
...     global frame
...     frame = inspect.currentframe()
...
>>> foo()
>>> # 这一帧执行'bar'的代码
>>> frame.f_code.co_name
'bar'
>>> # 它的后指针指向了'foo'
>>> caller_frame = frame.f_back
>>> caller_frame.f_code.co_name
'foo'
```  
![函数调用](function-calls.png)  
  
现在再来看Python的生成器,它也是利用相同的构建块--代码对象(code object)和堆栈帧--来达到奇妙的效果.  
  
以下是一个生成器函数:  
```python
>>> def gen_fn():
...     result = yield 1
...     print('result of yield: {0}'.format(result))
...     result2 = yield 2
...     print('result of 2nd yield: {0}'.format(result2))
...     return 'done'
...
```  
当Python将`gen_fn`编译为字节码时,它会看到`yield`语句,从而知道`gen_fn`是一个生成器函数,而不是一个常规函数.它设置一个标志来记住这个事实:  
```python
>>> # 生成器的标志位为第五位
>>> generator_bit = 1 << 5
>>> bool(gen_fn.__code__.co_flags & generator_bit)
True
```  
当你调用一个生成器函数的时候,Python会看到这个生成器标志位,实际上并不会运行这个函数.取而代之的,它会创建一个生成器:  
```python
>>> gen = gen_fn()
>>> type(gen)
<class 'generator'>
```  
一个Python生成器包含一个堆栈帧与一段代码,即`gen_fn`的主体部分:  
```python
>>> gen.gi_code.co_name
'gen_fn'
```  
来自`gen_fn`的生成器都指向同一段代码.但每一个都有各自的堆栈帧.这个堆栈帧不在任何的实际堆栈里,它呆在堆内存中等待被使用:  
![generator](generator.png)  
该帧有一个"最后指令(last instruction)"指针,它是最近执行的指令.一开始,这个指针为-1,表明该生成器还未开始:  
```python
>>> gen.gi_frame.f_lasti
-1
```  
当我们调用`send`的时候,生成器会到达它第一个`yield`处并暂停.`send`的返回值为1,这是`gen`传递给`yield`表达式的值:  
```python
>>> gen.send(None)
1
```  
生成器的指令指针现在从一开始的-1变为3,这是因为代码部分经过编译后变为了56个字节.(大概就这个意思吧...) 
> The generator's instruction pointer is now 3 bytecodes from the start, part way through the 56 bytes of compiled Python:  
```python
>>> gen.gi_frame.f_lasti
3
>>> len(gen.gi_code.co_code)
56
```  
可以随时从任何方法里恢复生成器,因为它的堆栈帧实际上并不在堆栈上:它在堆内存上(it is on the heap).它在调用层次结构中的位置不是固定的,它不需要遵循常规函数执行的先进后出的顺序.它像是被流放了一般.  
  
我们现在传输"hello"的值到生成器中,它将成为`yield`表达式的结果,而生成器会继续执行直到遇到`yield`2:  
```python
>>> gen.send('hello')
result of yield: hello
2
```  
它的堆栈帧现在包含了局部变量`result`:  
```python
>>> gen.gi_frame.f_locals
{'result': 'hello'}
```  
其他由`gen_fn`得到的生成器将会有他们各自的堆栈帧与局部变量.  
  
当我们再次调用`send`方法的时候,生成器会从第二个`yield`的位置继续执行,然后抛出一个特殊的"StopIteration"异常来结束:  
```python
>>> gen.send('goodbye')
result of 2nd yield: goodbye
Traceback (mose recent call last):
    File "<input>", line 1, in <module>
StopIteration: done
```  
该异常携带一个值,它是该生成器最后的返回结果:字符串"done"  

---
### 用生成器来构造协程
所以说,一个生成器可以暂停,并且可以用一个值来恢复且带有一个返回值.听起来是一个很好的用于构建异步程序模型的语言,而且不会有面条式的回调.我们要建立一个"协程":一个可以与其他程序协调安排的程序.我们这个协程将是Python标准库"asyncio"中的简化版本.正如asyncio中一样,我们将会用到生成器,futures和"yield from"语句.  
  
首先,我们需要一种方法来表示该协程正在等待的未来的结果.以下是一个简易版本:  
```python
class Future:
    def __init__(self):
        self.result = None
        self._callbacks = []

    def add_done_callback(self, fn):
        self._callbacks.append(fn)

    def set_result(self, result):
        self.result = result
        for fn in self._callbacks:
            fn(self)
```  
一个`Future`一开始是"待定"状态.直到调用`set_result`方法.  
  
让我们适配下我们的爬虫,利用`Future`与协程.先用回调写`fetch`方法:  
```python
class Fetcher:
    def fetch(self):
        self.sock = socket.socket()
        self.sock.setblocking(False)
        try:
            self.sock.connect(('xkcd.com', 80))
        except BlockingIOError:
            pass
        selector.register(self.sock.fileno(), EVENT_WRITE, self.connected)

    def connected(self, key, mask):
        print('connected!')
        # Balabala...
```  
`fetch`方法首先建立一个socket的连接,然后注册`connected`这个回调函数,当socket准备好时将被执行.现在我们可以用协程来将这两步结合在一起:  
```python
def fetch(self):
    sock = socket.socket()
    sock.setblocking(False)
    try:
        sock.connect(('xkcd.com', 80))
    except BlockingIOError:
        pass
    
    f = Future()
    
    def on_connected():
        f.set_result(None)
    
    selector.register(sock.fileno(), EVENT_WRITE, on_connected)
    
    yield f
    selector.unregister(sock.fileno())
    print('connected!')
```  
现在`fetch`已经成为了生成器而不是一个常规的函数,因为它带有"yield"语句.我们创建一个待定的`future`,然后运行到yield处暂停`fetch`,直到socket准备好.内部的`on_connected`方法会处理这个`future`.  
  
但当`future`被处理后,谁来恢复这个生成器呢?我们需要一个协程驱动.称之为"task":  
```python
class Task:
    def __init__(self, coro):
        self.coro = coro
        f = Future()
        f.set_result(None)
        self.step(f)

    def step(self, future):
        try:
            next_future = self.coro.send(future.result)
        except StopIteration:
            return

        next_future.add_done_callback(self.step)

# 开始抓取 http://xkcd.com/353/
fetcher = Fetcher('/353/')
Task(fetcher.fetch())

loop()
```  
这个task通过传入一个`None`来启动`fetch`生成器.然后`fetch`开始运行到yield处,此时task将会得到next_future的值.当socket建立连接后,事件循环将调用回调函数`on_connected`,该函数将处理`future`,从而调用`step`,也就恢复了`fetch`.  

---
### 利用yield from来管理协程
一旦socket连接成功,我们就发送一个GET请求然后读取服务器的响应.这些步骤不再需要用回调机制而分散;我们可以将它们整合到一个生成器函数中:  
```python
def fetch(self):
    # 连接socket的逻辑见上面
    sock.send(request.encode('ascii'))

    while True:
        f = Future()

        def on_readable():
            f.set_result(sock.recv(4096))

        selector.register(sock.fileno(), EVENT_READ, on_readbale)
        chunk = yield f
        selector.unregister(sock.fileno())
        if chunk:
            self.response += chunk
        else:
            # 进行读取
            break
```  
这个从socket中读取整段信息的代码看起来蛮有用的.我们该怎样利用它从`fetch`过到子程序呢?(不会翻译..)  
> How can wo factor it from `fetch` into a subroutine?  

现在Python3.x有`yield from`来解决这个问题.它将一个生成器委托给了另一个.  
  
要知道该怎么做,先回到原先的那个简单的生成器例子:  
```python
>>> def gen_fn():
...     result = yield 1
...     print('result of yield: {}'.format(result))
...     result2 = yield 2
...     print('result of 2nd yield: {}'.format(result2))
...     return 'done'
...
```  
要从另一个生成器调用它,只需用`yield from`进行委托:  
```python
>>> # 一个生成器函数
>>> def caller_fn():
...     gen = gen_fn()
...     rv = yield from gen
...     print('return value of yield-from: {}'.format(rv))
...
>>> # 从一个生成器函数来生成一个生成器
>>> caller = caller_fn()
```  
`caller`这个委托的生成器表现的跟`gen`一样:  
```python
>>> caller.send(None)
1
>>> caller.gi_frame.f_lasti
15
>>> caller.send('hello')
result of yield: hello
2
>>> caller.gi_frame.f_lasti  # 没有进一步
15
>>> caller.send('goofbye')
result of 2nd yield: goodbye
return value of yield-from: done
Traceback (most recent call last):
    File "<input>", line 1, in <module>
StopIteration
```  
当`caller`从`gen`返回时,`caller`并没有"前进".值得注意的是它的指令指针保持在15的位置,也正是`yield from`语句的所在,即使内部的生成器`gen`已经从一个`yield`语句进行到下一个`yield`语句.从`caller`的角度看,我们无法判断它产生的值是来自`caller`还是其委托的生成器.而从内部的`gen`的角度看,我们也不能确定值是从`caller`来的还是外部发送的.`yield from`语句是一个"无摩擦(frictionless)"的通道,通过它值(value)可以流进流出直到`gen`完成.  
  
一个协程通过`yield from`可以将工作委托给一个子协程,从而获得工作的结果.可以注意到,在上面的例子中,`caller`打印出了"return value of yield-from: done".当`gen`完成的时候,由于`caller`内部的`yield from`语句它(`gen`)的结果值得以返回:  
```python
rv = yield from gen
```  
在早些时候,但我们批评基于回调的异步程序时,最主要的一点就是关于"堆栈断裂(stack ripping)":当一个回调抛出异常时,堆栈跟踪便没用了.它只会显示事件循环在运行回调而没有显示为什么.那协程会怎么做呢?  
```python
>>> def gen_fn():
...     raise Exception('my error')
>>> caller = caller_fn()
>>> caller.send(None)
Traceback (most recent call last):
    File "<input>", line 1, in <module>
    File "<input>", line 3, in caller_fn
    File "<input>", line 2, in gen_fn
Exception: my error
```  
这就很有用了!当抛出错误时,堆栈跟踪会显示是`caller_fn`在委托`gen_fn`.更令人欣慰的是,我们可以在异常处理程序中,将调用包装到子协程中,就跟普通的子程序一样:  
```python
>>> def gen_fn():
...     yield 1
...     raise Exception('uh oh')
...
>>> def caller_fn():
...     try:
...         yield from gen_fn()
...     except Exception as err:
...         print('caught {}'.format(err))
...
>>> caller = caller_fn()
>>> caller.send(None)
1
>>> caller.send('hello')
caught un oh
```  
这样的话我们的逻辑就跟普通子程序的逻辑一样了.让我们往爬虫里添加一些有用的子协程.写一个`read`协程来接收固定块:  
```python
def read(sock):
    f = Future()

    def on_readbale():
        f.set_result(sock.recv(4096))
    
    selector.register(sock.fileno(), EVENT_READ, on_readable)
    chunk = yield f  # 读取一块
    selector.unregister(sock.fileno())
    return chunk
```  
再构造一个`read_all`在负责接收整条信息:  
```python
def read_all(sock):
    response = []
    # 读取整条信息
    chunk = yield from read(sock)
    while chunk:
        response.append(chunk)
        chunk = yield from read(sock)
    
    return b''.join(response)
```  
把`yield from`语句去掉的话,这些就像是传统的阻塞型I/O的函数.但实际上,`read`和`read_all`都是协程.`read`暂停了`read_all`直到I/O操作完成.当`read_all`暂停时,asyncio的事件循环就会开始其他工作并等待其他I/O事件;一旦事件准备完成,`read_all`将会被下一个循环中的`read`的结果恢复.  
  
在堆栈根部(stack's root),`fetch`调用`read_all`:  
```python
class Fetcher:
    def fetch(self):
        # 连接逻辑见上
        sock.send(request.encode('ascii'))
        self.response = yield from read_all(sock)
```  
而Task这个类却不需要修改.它用跟之前一样的方式来驱动`fetch`:  
```python
Task(fetcher.fetch())
loop()
```  
当`read`产生一个`future`时,任务通过`yield from`语句这个通道来接收它,就像它是直接从`fetch`中产生的一样.当事件循环"解决"了一个`future`,任务就发送它的结果给`fetch`,这个值直接被`read`接收,就像该任务直接驱动了`read`:  
![yield-from机制](yield-from.png)  
为了更好的实现我们的协程,我们来优化一下:当我们的代码等待一个`future`时使用`yield`,但在委托到一个子协程时使用`yield from`.无论何时要协程暂停时都使用`yield from`的话会更加优雅.那样的话协程就不需要关心它在等待什么类型的事件.  
  
我们利用Python在生成器与迭代器之间的深层通信.对调用者来说,升级一个生成器就跟升级一个迭代器一样.所以我们通过实现一个特殊的方法将我们的`Future`类变为可迭代:  
```python
# 在Future类中的方法
def __iter__(self):
    # 让Task知道在这里恢复
    yield self
    return self.result
```  
`future`的`__iter__`方法是一个协程,它会返回`future`自身.现在我们将代码从这样:  
```python
# f是一个Future实例
yield f
```
替换为这样:  
```python
# f是一个Future实例
yield from f
```  
输出将不变!用于驱动的Task在调用`send`时接收到一个`future`,当`future`得到解决时它会发送一个新的结果回到协程中.  
  
到处使用`yield from`的好处是啥呢?为什么它就比用`yield`来等待`future`和用`yield from`来委托子协程要更好?好处就是,一个方法可以自由的改变而不会影响到他的调用者:它可能是会返回一个用来解决值的`future`的普通方法,亦或者一个使用`yield from`语句来返回值的协程.不管是哪种情况,调用者只需要使用`yield from`方法来等待结果.  
  
亲爱的读者,我们即将结束这次异步中的协程的探索之旅.我们观察了生成器中的原理,编写了对`future`与`task`的实现.我们讲述了asyncio如何获得着两个最佳的优点:比线程更有效,比回调更清晰的并发I/O.当然,实际的asyncio比我们的实践更加复杂.真正的框架解决了I/O的零拷贝(zero-copy I/O),公平调度,异常处理和其他丰富的功能.  
  
对一个asyncio的使用者来说,用协程写代码比你在这里看到的更简单.从我们上面实现的协程中,你看到了回调,`task`与`future`.甚至是非阻塞I/O和对`select`的调用.但当使用asyncio构建应用程序时,这些都不会出现在你的代码中.正如我们承诺的,你现在可以真正的抓取一个URL了:  
```python
@asyncio.corountine
def fetch(self, url):
    response = yield from self.session.get(url)
    body = yield from response.read()
```  
对本次探索感到满意的话,我们回到原来的工作中:使用asyncio来写一个网络爬虫.  

---
### 协调协程
我们先来描述下我们想要我们的爬虫如何工作.现在该用asyncio的协程来实现它了.  
  
我们的爬虫先抓取第一个页面,解析它的链接然后将它们添加到队列中.然后它会分布开来,同时抓取页面.但为了限制客户端与服务器的负载,我们希望有个最大值来限制可运行的"工人"数量.每当一个工人完成抓取页面时,应立即从队列中拉出下一个链接.当没有可以爬取的地方后,经过一段时间,这些工人必须暂停.但当一个工人遇到一个满是新链接的页面的时候,队列开始增长,所有暂停的工人都应该唤醒并开始爬取.最后,一旦它的工作完成,我们的程序必须退出.  
  
想象一下,如果这些工人是线程.我们该如何编写爬虫的算法?我们可以从Python的标准库中使用一个同步队列.每一次有项目被放进队列里,该队列就增加"任务"的计数.工作线程在完成该工作后就调用`task_done`方法.主线程在`Queue.join`中阻塞,直到每一个放进队列的项目都被`task_done`调用,然后退出.  
  
协程通过asyncio队列,使用着完全相同的模式!首先我们导入它:  
```python
try:
    from asyncio import JoinableQueue as Queue
except ImportError:
    # 在Python3.5中,asyncio.JoinableQueue合并在Queue里
    from asyncio import Queue
```  
我们在一个爬虫类中收集工人们的共享状态,在一个`crawl`方法中写入主要逻辑.用一个协程启动`crawl`然后运行asyncio的事件循环直到`crawl`结束:  
```python
loop = asyncio.get_event_loop()
crawler = crawling.Crawler('http://xkcd.com', max_redirect=10)
loop.run_untul_complete(crawler.crawl())
```  
`crawler`用一个根URL与`max_redirect`参数初始化,重定向的数量这个参数将会作用于每一个url的抓取.它将每一个(URL, max_redirect)对放入队列中.(至于为什么要这么做,敬请期待):  
```python
class Crawler:
    def __init__(self, root_url, max_redirect):
        self.max_tasks = 10
        self.max_redirect = max_redirect
        self.q = Queue()
        self.seen_urls = set()

        # aiohttp库的ClientSession会有一个连接池并且为我们保持连接(HTTP keep-alives)
        self.session = aiohttp.ClientSession(loop=loop)

        # 将(URL, max_redirect)对放入队列
        self.q.put((root_url, self.max_redirect))
```  
队列里未完成的任务数量现在是一个.回到我们的脚本,我们启动一个事件循环与`crawl`方法:  
```python
loop.run_until_complete(crawler.crawl())
```  
`crawl`这个协程可以踢掉工人.就像是一个主线程一样:它在`join`这里阻塞直到所有任务完成,而工人们在后台运行.  
```python
@asyncio.coroutine
def crawl(self):
    '''运行crawler直到所有工作完成'''
    workers = [asyncio.Task(self.work()) for _ in range(self.max_tasks)]
    # 当所偶工作完成后,退出
    yield from self.q.join()
    for w in workers:
        w.cancel()
```  
如果工人们(workers)是线程那我们估计不会想要一次性全部启动.为了避免创建昂贵的线程,除非确定它们是有必要的,线程池通常按需增长.但协程很廉价,所以我们只需在开始时简单的设定下允许的最大数量.  
  
我们如何关闭爬虫是个有趣的问题.当`join`这个future被解决后,任务依然存在但已被挂起:他们等待更多的url,但并没有到来.所以主协程在退出前先取消他们.否则,当Python解释器关闭并且销毁所有对象时,尚存的任务会发出警告:  
```python
ERROR:asyncio:Task was destroyed but it is pending!
```  
那`cancel`该如何工作呢?生成器还有个我们未展示的功能.你可以从外面抛一个异常到生成器里:  
```python
>>> gen = gen_fn()
>>> gen.send(None)  # 跟往常一样启动生成器
1
>>> gen.throw(Exception('error'))
Traceback (most recent call last):
    File "<input>", line 3, in <module>
    File "<input>", line 2, in gen_fn
Exception: error
```  
生成器现在被`throw`恢复了,但现在它抛出了个异常.如果在生成器的调用堆栈中没有代码来捕获它,则异常会回到顶部.所以为了取消一个任务协程:  
```python
# 在Task类中的方法
def cancel(self):
    self.coro.throw(CancelledError)
```  
不管生成器由于`yield from`语句暂停在什么地方,它都会恢复并抛出一个异常.我们在`task`的`step`方法中处理这个异常:  
```python
# Task类中的方法
def step(self, future):
    try:
        next_future = self.coro.send(future.result)
    except CancelledError:
        self.cancelled = True
        return
    except StopIteration:
        return
    
    next_future.add_done_callback(self.step)
```  
现在任务就知道它被取消了,所以当它被销毁时也不会再挣扎了.  
  
一旦`crawl`取消了这些工人,它就会退出.事件循环看到协程已经完成(我们待会就会知道该怎么做),然后它也一起退出了:  
```python
loop.run_until_complete(crawler.crawl())
```  
`crawl`方法中包含了我们的主协程需要做的所有操作.任务协程该做的就是从队列中获取URL,抓取它,然后解析出新的链接.每一个工人都各自运行`work`协程:  
```python
@asyncio.coroutine
def work(self):
    while True:
        url, max_redirect = yield from self.q.get()

        # 下载页面并添加新链接到self.q中
        yield from self.fetch(url, max_redirect)
        self.q.task_done()
```  
Python发现这个代码中包含`yield from`语句,所以将它编译为生成器函数.所以在`crawl`中,当主线程调用`self.work`十次时,它实际上并不会执行这个方法:它只会创建十个引用这段代码的生成器对象.然后将每一个都包含到一个`Task`中.`Task`接收到由生成器传递来的`future`,然后当`future`被解决时通过使用每一个`future`的结果来调用`send`从而驱动生成器.因为生成器有他们各自的堆栈帧,所以将会带有各自的局部变量与指令指针去独立运行.  
  
工人通过队列与其他人合作.它通过以下方式等待新的URL:  
```python
url, max_redirect = yield from self.q.get()
```  
队列的`get`方法本身也是一个协程:它会暂停直到有项目进入队列,然后恢复并返回该项目.  
  
顺便,当主协程取消工人的时候,在`crawl`的结尾,这里也就是它们会暂停的地方.从协程的角度看,当`yield from`抛出一个CancelledError异常时,它最后一次循环也就结束了.  
  
当一个工人爬取一个页面时,它解析出链接并将新的放进队列中,然后调用`task_done`来将计数减一.最后,一个工人爬取了一页上所有url都被爬取了的页面,而队列中也没有任务了.因此这个工人调用`task_done`将计数减到零.在等待队列的`join`方法的`crawl`,就解除了暂停并结束.  
  
我们说好会解释下为什么队列中的项目是成对的,就像这样:  
```python
# 需要爬取的URL,与剩下的重定向数目
('http://xkcd.com/353', 10)
```  
新的URL会有10次重定向剩余.当爬取这个URL时会在结尾附上一个斜杠而重定向到一个新的位置.此时减少一次重定向数量,然后放到队列的下一个位置:  
```python
# 带有斜杠的URL.九次重定向剩余
('http://xkcd.com/353/', 9)
```  
我们使用的aiohttp库将默认遵循重定向,并给我们最终的响应.但是,我们让他不这么做,而是在爬虫中处理重定向,所以它可以合并指向相同目的地的重定向路径:如果我们已经处理过这个URL,它将存在于self.seen_urls中而我们已经从不同的点进入这条路径:  
![redirect](redirects.png)  
爬虫爬取到"foo"然后看到它指向了"baz",所以就将"baz"添加到队列和`seen_urls`中.如果在下一页爬取到"bar",也指向了"baz",爬虫就不会又将它添加到队列里.如果响应是一个页面而不是重定向,`fetch`便解析它得到链接将新的放到队列里:  
```python
@asyncio.coroutine
def fetch(self, url, max_redirect):
    # 我们自己处理重定向
    response = yield from self.session.get(url, allow_redirects=False)

    try:
        if is_redirect(response):
            if max_redirect > 0:
                next_url = response.headers['location']
                if next_url in self.seen_urls:
                    # 我们之前已经处理过这个路径
                    return
                
                # 记住我们见过这个URL
                self.seen_urls.add(next_url)

                # 遵循重定向.重定向次数减一
                self.q.put_nowait((next_url, max_redirect-1))
        else:
            links = yield from self.parse_links(response)
            # Python的集合操作
            for link in links.difference(self.seen_urls):
                self.q.put_nowait((link, self.max_redirect))
            self.seen_urls.update(links)
    finally:
        # 将连接返回到连接池
        yield from response.release()
```  
如果是多线程的代码,那将会因竞争条件(race conditions)而变得很糟糕.例如,一个工人在检查一个链接是否在`seen_urls`里,如果没有的话便将它放进队列然后添加到`seen_urls`里.如果它在两个操作之间被中断,那另一个工人可能会从一个不同的页面解析到同一个链接,同时也观察到它不在`seen_urls`中,并将其添加到队列中.现在同一个链接被放进队列两次,(至少)导致了重复的工作与统计错误.  
  
然而,一个协程只会轻易地被`yield from`语句中断.这就是协程代码远不如多线程的关键所在:多线程代码必须通过抓取一个锁定,明确地输入一个关键部分,否则它就是可中断的.一个Python协程默认情况下是不中断的,只有在明确的`yield`时才能释放控制权.  
  
我们不在像之前基于回调的程序那样需要一个`Fetcher`类.这个类只是为了弥补回调函数的不足:在等待I/O的时候,它们需要一个地方来保存状态,因为他们的局部变量在调用之间不被保留.但`fetch`协程可以先一个常规的函数那样保存自己的状态到局部变量中,所以我们不在需要这个类.  
  
当`fetch`完成对服务器响应的处理时,它会返回给调用者,`work`.`work`方法在队列上调用`task_done`然后从队列里得到下一个URL来抓取.  
  
当`fetch`将新的链接放到队列里时,它将未完成的任务数加一,并且将正在等待`q.join`的主协程暂停.然而,如果已经没有未处理的URL或者说这是队列里最后一个URL时,`work`调用`task_done`使得未完成的任务数变为零.这个事件恢复了`join`从而主协程得以完成.  
  
与工人和主协程协调合作的队列的代码就像下面这样:  
```python
class Queue:
    def __init__(self):
        self._join_future = Future()
        self._unfinished_tasks = 0
        # ...其他初始化操作...

    def put_nowait(self, item):
        self._unfinished_tasks += 1
        # ...储存该项目...

    def task_done(self):
        self._unfinished_tasks -= 1
        if self._unfinished_tasks == 0:
            self._join_future.set_result(None)

    @asyncio.coroutine
    def join(self):
        if self._unfinished_tasks > 0:
            yield from self._join_future
```  
主协程`crawl`,从`join`来恢复.所以当最后一个工人将未完成的任务数变为0时,它标志着`crawl`恢复,并且结束.  
  
旅途就要结束了.我们的程序通过调用`crawl`来开始:  
```python
loop.run_until_complete(self.crawler.crawl())
```  
那程序该怎么结束呢?由于`crawl`是一个生成器函数,调用它返回一个生成器.为了驱动这个生成器,asyncio将它包含在一个任务中:  
```python
class EventLoop:
    def run_until_complete(self, coro):
        '''运行知道协程结束'''
        task = Task(coro)
        task.add_done_callback(stop_callback)
        try:
            self.run_forever()
        except StopError:
            pass
        
class StopError(BaseException):
    '''在停止事件循环时抛出'''

def stop_callback(future):
    raise StopError
```  
在任务完成时,它抛出StopError异常,循环将它视为一个任务正常完成的信号.  
  
但这又是什么?该任务调用了`add_done_callback`和`result`?你可能会想这个任务酷似`future`.你的直觉是对的.关于这个`Task`类,我们必须承认一个我们没有告诉你的细节:他就是`future`:  
```python
class Task(Future):
    '''一个包含在Future里的协程'''
```  
一般来说一个`future`会被别人通过调用`set_result`来解决.但一个`task`在它的协程结束时,它自己解决.回想起我们很早前对Python生成器的探索,当它返回时,它会抛出一个特殊的异常StopIteration:  
```python
# Task类里的方法
def step(self, future):
    try:
        next_future = self.coro.send(future.result)
    except CancelledError:
        self.cancelled = True
        return
    except StopIteration as err:
        # 通过协程的返回值Task解决自己
        self.set_result(err.value)
        return

    next_future.add_down_callback(self.step)
```  
所以当事件循环调用`task.add_done_callback(stop_callback)`时,它准备被`task`所停止.这里再次`run_until_complete`:  
```python
# 事件循环里的方法
def run_until_complete(self, coro):
    task = Task(coro)
    task.add_done_callback(stop_callback)
    try:
        self.run_forever()
    except StopError:
        pass
```  
当`task`截取到`StopIteration`然后解决自身时,回调函数会在循环中抛出`StopError`异常.循环停止,调用堆栈解卷(unwound)到`run_until_complete`.我们的程序就结束了.  

---
### 结论
越来越多的,现代程序由I/O绑定取代了CPU绑定.对这样的程序来说,Python的线程有着最差的两点:全局解释器锁妨碍了他们并行计算,抢占式切换(preemptive switching)让他们更容易发生竞争(race).异步通常是正确的模式.但正如基于回调的异步代码的增长,它逐渐变的混乱.协程是一个整洁的代替者.他们自然地被子程序所决定,具有清晰的异常处理与堆栈跟踪.  
  
我们忽略掉`yield from`语句来看,一个协程看起来就像是一个执行阻塞I/O的线程一般.我们甚至可以使用多线程的经典模式来协调协程.不再需要重造.因此,跟回调比较,协程对于一个有着多线程编程经验的程序员来说是更加友好的.  
  
但当我们关注到`yield from`语句的时候,我们看到它标识出了协程在什么时候失去控制权与什么时候运行其他.不像线程,协程显现出我们的代码在哪里可以中断与哪里不能.正如Glyph Lefkowitz写的,"线程使得本地推理(local reasoning)变得困难,而这或许是软件开发中最重要的事."然而,明确的中断,让它可以"通过检视一个例行程序本身而不是整个系统,来理解一个例行程序的行为(从而理解它的正确性)"  
  
这一章是写在Python与异步的复兴时期.基于生成器的协程,正如你刚刚学到的,于2014年三月发布在Python3.4的"asyncio"模块.到了2015年九月,Python3.5被发布,其语言本身内置了协程.这些自带的协同程序用新的语法`async def`声明,他们使用新的`await`关键字来委托协程或者等待一个`Future`而不是使用`yield form`.  
  
尽管有着这些更新,但核心思想不变.Python新的内置协程从语法上将会和生成器不同,但工作非常相似;确实,他们将在Python的解释器上分享一个实现.`Task`,`Future`和事件循环将会在asyncio中扮演他们自己的角色.  
  
现在你知道如何用asyncio的协程工作了,你可以丢掉大部分细节.他的工作原理被藏在了精简的表面下.但你掌握的基础使你能够在现代异步环境中正确高效的编码.  

---

[link]: http://aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html  
[sopaghetii code]: https://zh.wikipedia.org/wiki/%E9%9D%A2%E6%9D%A1%E5%BC%8F%E4%BB%A3%E7%A0%81
