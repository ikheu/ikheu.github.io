---
layout: post
title:  【翻译】 A Web Crawler With asyncio Coroutines
excerpt: ''
categories: 网络
tags: Python
---

## 概述

传统的计算机科学常强调有效的算法，来尽快地完成计算。但许多网络程序耗时不在计算上，而在保持大量缓慢的连接或低频事件上。这些程序提出了非常不同的挑战：有效地等待大量的网络事件。解决这个问题的一个现代方法是异步 I/O，或 `async`。

本文介绍一个简单的网络爬虫。网络爬虫等待许多响应，但只进行少量计算，是典型的的异步应用。一次抓取的页面越多则完成地越快。如果为每个进行中的请求分配一个线程，则随着并发请求数的增加，在耗尽套接字之前，内存或其他与线程相关的资源将被耗尽。通过使用异步I/O，可避免了对线程的需求。

我们分三个阶段介绍该示例。 首先，我们展示一个异步事件循环，并简述一个使用事件循环和回调的爬虫：它非常有效，但将其扩展到更复杂的问题时会导致难以处理的面条式代码。因此，第二阶段我们将展示 Python 协程既高效又可扩展。我们使用生成器函数在 Python 中实现简单的协程。 在第三阶段，我们使用 Python 标准的 `asyncio` 库中功能齐全的协程，并使用异步队列进行协调。

## 任务

网络爬虫应用寻找并下载网站的所有页面，可能对它们进行存储或索引。程序从根路径开始获取每个页面，解析页面以查找不可见的页面链接，并将它们添加到队列中。当页面没有不可见链接并且队列为空时，程序将停止。

同时下载多个页面可加速爬取过程。当爬虫程序找到新的链接时，在不同的套接字上同时对这些页面启动抓取操作。解析到达的响应，并将新链接添加到队列中。并发数太多会降低性能，可能造成收益递减的情况，因此可限制并发请求数量，将其余链接留在队列中，直到完成当前的请求。

## 传统方法

如何让爬虫并发？传统方法是创建一个线程池。每个线程通过一个套接字下载一个页面。例如要从 xkcd.com 下载页面：

```python
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)

    # Page is now downloaded.
    links = parse_links(response)
    q.add(links)
```

默认情况下，套接字操作处于阻塞状态：当线程调用诸如 `connect` 或 `recv` 之类的方法时，它将暂停直到操作完成。因此，要一次下载多个页面，我们需要多个线程。复杂的应用程序通过将空闲线程保留在线程池中，然后将其检出重新用于后续任务，以此摊销线程创建的成本；它与连接池中的套接字相同。

然而，线程成本很高，并且操作系统对进程、用户或机器可能具有的线程数施加各种硬性限制。在 Jesse 系统上，Python 线程消耗大约 50KB 内存，启动数万个线程会导致失败。如果我们在并发套接字上进行成千上万的同时操作，那么在套接字用完之前，线程就用完了。每个线程的开销或系统对线程的限制是瓶颈。

Dan Kegel 在其著名的 "C10K问题" 论文中，概述了多线程对 I/O 并发的局限性。他开头写道：
>是时候让 Web 服务器同时处理一万个客户端了，不是吗？ 今非昔比，如今网络世界太大了。

Kegel 于 1999 年创造了 "C10K" 一词。如今，一万个连接听起来很精确，但问题只有量的变化，而没有质的变化。 当时对于C10K 问题，在每个连接中创建线程是不切实际的。现在，上限提高了几个数量级。确实，基于多线程的网络爬虫可以正常工作，但是对于具有数十万个连接的超大型应用程序，上限仍然存在：这个限制是，大多数系统虽然还可以创建套接字，但线程用完了。我们该如何解决该问题呢？

## Async

异步 I/O 框架使用非阻塞套接字在单个线程上执行并发操作。在异步爬虫中，开始连接到服务器之前将套接字设置为非阻塞：

```python
sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass
```

烦人的是，即使非阻塞套接字正常运行，它也会从 `connect` 引发异常。此异常复制了底层 C 函数中令人厌恶的行为，该函数将`errno` 设置为 `EINPROGRESS` 以告知它已经开始。

现在，爬虫程序需要一种方法来知道何时建立连接，以便它可以发送 HTTP 请求。 可以简单在循环中进行尝试：

```python
request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
encoded = request.encode('ascii')

while True:
    try:
        sock.send(encoded)
        break  # Done.
    except OSError as e:
        pass

print('sent')
```

这种方法不仅成本高，而且不能有效地等待多个套接字上的事件。在远古时代，BSD Unix 解决此问题的方法是使用 C 函数 `select`，该函数在非阻塞套接字或其中的一小部分套接字上等待事件发生。 如今，对具有大量连接的互联网应用程序的需求，导致了一些程序取代了 `select`, 例如 `poll` BSD 上的 `kqueue` 和 Linux 上的 `epoll`。 这些 API 与 select 类似，但是在处理大量连接时表现更佳。

Python 3.4 的 `DefaultSelector` 使用系统上可用的最佳的类 `select` 函数。为注册有关网络 I/O 的通知，我们创建一个非阻塞套接字，并使用默认选择器注册它：

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

我们忽略了虚假错误，并调用 `selector.register`，传入套接字的文件描述符和一个常量，该常量表示正在等待的事件。为了在建立连接时得到通知，我们传递 `EVENT_WRITE` ：也就是说，我们想知道套接字何时是“可写的”。 我们还会传递一个 Python 函数 `connected`，以在该事件发生时运行。这种功能称为回调。

当选择器接收到 I/O 通知时，将对其进行循环处理：

```python
def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()
```

`connected` 的回调存储为 `event_key.data`，一旦连接了非阻塞套接字，便检索并执行该回调。

与上面的快速旋转循环不同，此处对 `select` 的调用会暂停，以等待下一个 I/O 事件。然后循环运行等待这些事件的回调。未完成的操作将保持挂起状态，直到事件循环的将来某个时刻。

上文展示了什么？展示了如何开始操作并在操作就绪后执行回调。异步框架建立在上文展示的两个功能（无阻塞套接字和事件循环）的基础上，可以在单个线程上运行并发操作。

我们在这里实现了“并发”，但是没有实现传统上所谓的“并行”。也就是说，我们构建了一个很小的系统，该系统执行重叠的 I/O。它能够在其他操作还在进行时开始新的操作。它实际上并没有利用多个内核并行执行计算。此系统是针对 I/O 约束的问题而不是 CPU 约束的问题而设计的。

所以，我们的事件循环在处理并发 I/O 上效率很高，因为它不会给每个连接分配线程资源。但是在继续之前，要先纠正一个常见的误解：异步比多线程要快。通常并非如此——实际上，在 Python 中，在处理少量非常活跃的连接时，像我们这样的事件循环比多线程要慢一些。在没有全局解释器锁的运行时中，线程在这种工作负载下会表现得更好。异步 I/O z最适用于事件很少、有许多慢速或睡眠连接的应用程序。


## 使用回调编程

使用目前构建的简短的异步框架，如何完成爬虫程序呢？实际上，即使是简单的 URL 提取程序也很难编写。

首先，我们有一个尚未获取的 URL 集合，和一个已经解析过得 URL 集合：

```python
urls_todo = set(['/'])
seen_urls = set(['/'])
```

`seen_urls` 集合包含 `urls_todo` 和已经完成的 URL。使用很路径 URL "/" 初始化这两个集合。

抓取一个网页需要一系列的回调。套接字连接将触发 `connect` 回调，向服务器发送 GET 请求。但随后必须等待响应，于是需要注册另一个回调。如果这个回调触达了，还无法读取整个响应，因此又要注册一个回调，等等。

我们使用 `Fetcher` 对象来收集这些回调。它需要一个 URL，一个 socket 对象以及一个累积响应字节的位置。

```python
class Fetcher:
    def __init__(self, url):
        self.response = b''  # Empty array of bytes.
        self.url = url
        self.sock = None
```

首先调用 `Fetcher.fetch`：

```python
# Method on Fetcher class.
def fetch(self):
    self.sock = socket.socket()
    self.sock.setblocking(False)
    try:
        self.sock.connect(('xkcd.com', 80))
    except BlockingIOError:
        pass

    # Register next callback.
    selector.register(self.sock.fileno(),
                        EVENT_WRITE,
                        self.connected)
```

`fetch` 方法首先连接套接字，但注意在连接建立之前，该方法就返回了。它将等待连接的控制权交给了事件循环。要了解原因，想象我们的整个应用程序有这样的结构：

```python
# Begin fetching http://xkcd.com/353/
fetcher = Fetcher('/353/')
fetcher.fetch()
while True:
    events = selector.select()
    for event_key, event_mask in events:
        callback = event_key.data
        callback(event_key, event_mask)
```

调用 `select` 时，将在事件循环中处理所有事件通知。因此 `fetch` 必须将控制权交给事件循环，这样程序就知道套接字何时连接了。只有这样，事件循环才会运行 `connected` 回调，该回调上面的 `fetch` 结束时注册。

这是 `connected` 的实现：

```python
# Method on Fetcher class.
def connected(self, key, mask):
    print('connected!')
    selector.unregister(key.fd)
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(self.url)
    self.sock.send(request.encode('ascii'))

    # Register the next callback.
    selector.register(key.fd,
                        EVENT_READ,
                        self.read_response)
```

该方法发送一个 GET 请求。如果无法立即发送整个消息，则实际的应用会检查 `send` 的返回值。但是我们的请求很小，而且最后一个回调 `read_response` 处理服务器的响应：

```python
# Method on Fetcher class.
def read_response(self, key, mask):
    global stopped

    chunk = self.sock.recv(4096)  # 4k chunk size.
    if chunk:
        self.response += chunk
    else:
        selector.unregister(key.fd)  # Done reading.
        links = self.parse_links()

        # Python set-logic:
        for link in links.difference(seen_urls):
            urls_todo.add(link)
            Fetcher(link).fetch()  # <- New Fetcher.

        seen_urls.update(links)
        urls_todo.remove(self.url)
        if not urls_todo:
            stopped = True
```

每当选择器发现套接字是“可读的”时后，都会执行该回调，这可能意味着两件事：套接字具有数据或已关闭。

回调最多从套接字读取 4K 的数据。如果不足 4K，则 `chunk` 读取所有。如果比 4K 多，则 `chunk` 的长度为 4K，并且套接字保持可读状态，因此事件循环在下一个周期会再次运行此回调。响应完成后，服务器关闭这个套接字， `chunk` 为空。

这里没有展示 `parse_links` 方法，它返回一个 URL 集合。我们为每个新的 URL 启动一个 fetcher，没有并发上限。注意使用回调的异步编程的一个很棒的特性：修改共享数据时无需互斥锁，例如，向 `seen_urls` 中添加链接时。

增加全局变量 `stopped` 来控制事件循环：

```python
stopped = False

def loop():
    while not stopped:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()
```

下载所有页面后，fetcher 将停止全局事件循环，程序退出。

这个例子暴露了异步编程的一个问题：面条代码。我们需要某种方式来表示一系列计算和 I/O 操作，并调度多个此类操作以同时运行。但是，如果没有线程，就无法将一系列操作写在一个函数中：每当一个函数开始 I/O 操作时，它将显式保存未来运行所需要的任何状态，然后返回。你需要考虑并编写此状态保存的代码。

来解释下这到底是什么意思。考虑一下，传统的线程中使用阻塞套接字在上抓取 URL 是多么简单：

```python
# Blocking version.
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)

    # Page is now downloaded.
    links = parse_links(response)
    q.add(links)
```

在一个套接字操作与下一个套接字操作之间，该函数记得什么状态？有套接字、URL和累计的响应。在线程中运行的函数使用编程语言的基本功能，将临时状态存储在其堆栈上的局部变量中。该函数还具有连续性，在 I/O 执行完成后，继续执行后续代码。运行时通过存储线程的指令指针来记住连续性。您无需考虑在 I/O 之后恢复这些局部变量和延续。它是语言内置的。

但是，对于基于回调的异步框架，这些语言特性无能为力。等待 I/O 时，函数必须显示地保存状态，因为在 I/O 完成之前，函数返回并丢失栈帧信息。在基于回调的示例中，为了代替局部变量，我们将 `sock` 和 `response` 作为 `Fetcher` 实例 `self` 的属性来存储。而随着应用程序功能变多，我们在回调之间手动保存的状态的复杂性也在增加。这种繁重的簿记工作让人头大。

更糟糕的是，如果在调度链中的下一个回调之前，回调引发异常会怎样？假设 `parse_links` 没写好，在解析 HTML 时抛出了异常：

```
Traceback (most recent call last):
  File "loop-with-callbacks.py", line 111, in <module>
    loop()
  File "loop-with-callbacks.py", line 106, in loop
    callback(event_key, event_mask)
  File "loop-with-callbacks.py", line 51, in read_response
    links = self.parse_links()
  File "loop-with-callbacks.py", line 67, in parse_links
    raise Exception('parse error')
Exception: parse error
```

堆栈跟踪仅显示事件循环正在运行回调。我们无从得知什么导致了异常。链的两端都断了：我们忘了从哪儿来，要到哪去。这种上下文的丢失成为“堆栈撕裂”，在很多情况下让人困惑。堆栈撕裂还阻止我们为一连串的回调设置异常处理，即使用“try/except”块包装函数调用及其调用树。

因此，多线程和异步的长期争论中，除了哪个更高效外，另一个是关于哪个更易出错：在同步上，线程很容易受数据竞争的影响而出错，而回调方法因为堆栈撕裂难以调试。

## 协程

可以编写将回调效率与多线程编程经典外观完美结合的异步代码。这种结合通过协程模式实现。使用 Python 3.4 标准的协程库，以及 aiohttp 库，在协程中抓取 URL 十分直接：

```python
@asyncio.coroutine
    def fetch(self, url):
        response = yield from self.session.get(url)
        body = yield from response.read()
```

它也是可扩展的。与每个线程 50k 的内存和操作系统对线程的严格限制相比，Python协程在 Jesse 的系统上仅占用 3k 的内存。Python可以轻松启动成千上万个协程。

协程的概念可以追溯到计算机科学的早期，它很简单：它是可以暂停和恢复的子程序。线程由操作系统抢占式地执行多任务，而协程协同执行多任务：它们选择何时暂停以及接下来运行哪个协程。

协程有许多实现方式。即使在 Python 中也有几个。 Python 3.4 中标准 asyncio 库中的协程建立于生成器、Future 类和 yield from 语句的基础上。从Python 3.5 开始，协程是语言的原生特性。然而需要了解在 Python 3.4 中协程最初是使用已有语言工具实现的，这是应对 Python 3.5 原生协程的基础。

为了解释 Python 3.4 中基于生成器的协程，我们将对生成器以及它们如何用于异步的协程中进行说明，相信您会喜欢阅读它，就像我们编写它一样。解释了基于生成器的协程之后，我们将在异步 Web 爬虫中使用它们。

## Python 生成器如何工作

在掌握 Python 生成器之前，必须了解常规函数的工作方式。 通常，当 Python 函数调用函数时，该函数会保留控制权，直到它返回或引发异常为止。然后将控制权返回给调用者：

```
>>> def foo():
...     bar()
...
>>> def bar():
...     pass
```

标准的 Python 解释器用 C 编写。执行 Python 函数的 C 函数是 `PyEval_EvalFrameEx`。它接受一个 Python 栈帧对象，并在栈帧上下文中评估 Python 字节码。这是 `foo` 的字节码：

```
>>> import dis
>>> dis.dis(foo)
  2           0 LOAD_GLOBAL              0 (bar)
              3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
              6 POP_TOP
              7 LOAD_CONST               0 (None)
             10 RETURN_VALUE
```

`foo` 函数在栈上载入 bar 并调用它，然后 `bar` 的返回值出栈，`None` 入栈，并返回 `None`。

当 `PyEval_EvalFrameEx` 遇到 `CALL_FUNCTION` 字节码时，它将创建一个新的 Python 栈帧并进行递归：即，它使用新的栈帧递归地调用 `PyEval_EvalFrameEx`，用于执行 `bar`。

至关重要的是要了解 `Python` 栈帧是在堆内存中分配的！Python 解释器是普通的 C 程序，因此其栈帧是普通的栈帧。但是它操纵的 Python 栈帧在堆上。除其他惊喜外，这意味着 Python 栈帧可以超过其函数调用的寿命。为了交互式地查看它，可从 `bar` 中保存当前帧：

```
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
>>> # The frame was executing the code for 'bar'.
>>> frame.f_code.co_name
'bar'
>>> # Its back pointer refers to the frame for 'foo'.
>>> caller_frame = frame.f_back
>>> caller_frame.f_code.co_name
'foo'
```

现在该讲 Python 生成器了，它使用相同的构建块——代码对象和栈帧——产生了惊人的效果。

下面是个生成器函数：

```python
>>> def gen_fn():
...     result = yield 1
...     print('result of yield: {}'.format(result))
...     result2 = yield 2
...     print('result of 2nd yield: {}'.format(result2))
...     return 'done'
...     
```

当 `Python` 将 `gen_fn` 编译为字节码时，遇到 `yield` 表达式时将 `gen_fn` 视作生成器函数，而非一般的函数。设置标志记住这是个生成器函数：

```
>>> # The generator flag is bit position 5.
>>> generator_bit = 1 << 5
>>> bool(gen_fn.__code__.co_flags & generator_bit)
True
```

当调用生成器函数时，`Python` 会看到 `generator` 标志，并且实不会运行该函数。 相反，它将创建一个生成器：

```
>>> gen = gen_fn()
>>> type(gen)
<class 'generator'>
```

Python 生成器封装了栈帧以及对某些代码的引用，即 gen_fn 的主体：

```
>>> gen.gi_code.co_name
'gen_fn'
```

调用 `gen_fn` 的所有生成器都指向相同的代码。但是每个都有自己的栈帧。该栈帧不在任何实际的堆栈上，它位于堆内存中等待使用：


框架具有“最后一条指令”指针，该指针是最近执行的指令。 刚开始时，最后一条指令指针为 -1，表示生成器尚未开始：

```
>>> gen.gi_frame.f_lasti
-1
```


## 使用生成器创建协程

## 使用 `yield from` 分解协程

## 协程协调

## 结论

现代程序越来越经常受 I/O 约束，而不是受 CPU 约束。对于这样的程序，Python 线程是个两头吃亏的选择：全局解释器锁阻止它们真实的并行计算，而抢占式切换使它们易于发生竞争。异步通常是正确的模式。但是随着基于回调的异步代码的增长，它往往会变得混乱不堪。协程是一个整洁的选择。它们自然地包含在子例程中，并具有健全的异常处理和堆栈跟踪功能。

如果我们斜视以使语句的收益模糊，那么协程看起来就像是执行传统阻塞I / O的线程。 我们甚至可以将协程与来自多线程编程的经典模式进行协调。 无需重新发明。 因此，与回调相比，协程对于使用多线程的程序员是一个诱人的习惯用法。

但是，当我们睁开眼睛，专注于语句的收益时，我们看到它们在协程让步并允许其他人运行时标记了点。 与线程不同，协程会显示我们的代码可以在何处中断和无法在何处中断。 格里夫·莱夫科维茨（Glyph Lefkowitz）在他的启发性文章“坚强” 14中写道：“线程使本地推理变得困难，本地推理可能是软件开发中最重要的事情。” 但是，显式屈服可以“通过检查例程本身而不是整个系统来了解例程的行为（从而，正确性）”。

本章是在Python和异步技术的复兴中撰写的。 您刚刚学会了设计的基于生成器的协程，已于2014年3月在Python 3.4的“异步”模块中发布。2015年9月，Python 3.5随语言本身内置的协程一起发布。 这些本机协程以新语法“ async def”声明，而不是“ yield from”，而是使用新的“ await”关键字委派给协程或等待Future。

尽管取得了这些进步，但核心思想仍然存在。 Python的新本机协程在语法上将与生成器不同，但工作方式非常相似。 实际上，他们将在Python解释器中共享一个实现。 任务，未来和事件循环将继续在异步中发挥作用。

既然您知道了异步协程的工作原理，就可以在很大程度上忘记细节。 机器被塞在一个精巧的接口后面。 但是，您掌握了基础知识后，便可以在现代异步环境中正确有效地进行编码。

