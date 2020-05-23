---
layout: post
title:  【翻译】 A Web Crawler With Asyncio Coroutines
excerpt: ''
categories: 网络
tags: Python
---

## 概述

传统的计算机科学常强调有效的算法，来尽快地完成计算。但许多网络程序耗时不在计算上，而在保持大量缓慢的连接或低频事件上。这些程序提出了非常不同的挑战：有效地等待大量的网络事件。解决这个问题的一个现代方法是异步 I/O，或 `async`。

本文介绍一个简单的网络爬虫。网络爬虫等待许多响应，但只进行少量计算，是典型的的异步应用。一次抓取的页面越多则完成地越快。如果为每个进行中的请求分配一个线程，则随着并发请求数的增加，在耗尽套接字之前，内存或其他与线程相关的资源将被耗尽。通过使用异步 I/O，可避免了对线程的需求。

我们分三个阶段介绍该示例。 首先，我们展示一个异步事件循环，并简述一个使用事件循环和回调的爬虫：它非常有效，但将其扩展到更复杂的问题时会导致难以处理的面条式代码。因此，第二阶段我们将展示 Python 协程既高效又可扩展。我们使用生成器函数在 Python 中实现简单的协程。在第三阶段，我们使用 Python 标准的 `asyncio` 库中功能齐全的协程，并使用异步队列进行协调。

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

与上面的快速轮转的循环不同，此处对 `select` 的调用会暂停，以等待下一个 I/O 事件。然后执行运行等待这些事件的回调函数。未完成的操作将保持挂起状态，直到进入到下一个事件循环执行。

上文展示了什么？展示了如何开始操作并在操作就绪后执行回调。异步框架建立在上文展示的两个功能（无阻塞套接字和事件循环）的基础上，可以在单个线程上运行并发操作。

我们在这里实现了“并发”，但是没有实现传统上所谓的“并行”。也就是说，我们构建了一个很小的系统，该系统执行重叠的 I/O。它能够在其他操作还在进行时开始新的操作。它实际上并没有利用多个内核并行执行计算。此系统是针对 I/O 约束的问题而不是 CPU 约束的问题而设计的。

我们的事件循环在处理并发 I/O 上效率很高，因为它不会给每个连接分配线程资源。但是在继续之前，要先纠正一个常见的误解：异步比多线程要快。通常并非如此——实际上，在 Python 中，在处理少量非常活跃的连接时，像我们这样的事件循环比多线程要慢一些。在没有全局解释器锁的运行时中，线程在这种工作负载下会表现得更好。异步 I/O 最适用于事件很少、有许多慢速或睡眠连接的应用程序。


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

在一个套接字操作与下一个套接字操作之间，该函数记得什么状态？有套接字、URL和累积的响应。在线程中运行的函数使用编程语言的基本功能，将临时状态存储在其堆栈上的局部变量中。该函数还具有连续性，在 I/O 执行完成后，继续执行后续代码。运行时通过存储线程的指令指针来记住连续性。你无需考虑在 I/O 之后恢复这些局部变量和延续。它是语言内置的。

但是，对于基于回调的异步框架，这种语言特性不起作用。等待 I/O 时，函数必须显示地保存状态，因为在 I/O 完成之前，函数返回并丢失栈帧信息。在基于回调的示例中，为了代替局部变量，我们将 `sock` 和 `response` 作为 `Fetcher` 实例 `self` 的属性来存储。而随着应用程序功能变多，我们在回调之间手动保存的状态的复杂性也在增加。这种繁重的簿记工作让人头大。

更糟糕的是，如果在调度链中的下一个回调之前，回调引发异常会怎样？假设 `parse_links` 没处理好，在解析 HTML 时抛出了异常：

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
def gen_fn():
    result1 = yield 1
    print('result of yield: {}'.format(result1))
    result2 = yield 2
    print('result of 2nd yield: {}'.format(result2))
    return 'done'
    
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

当调用 `send` 时，生成器到达第一个 `yield`，然后暂停。`send` 的返回值是 1，因为 gen 传给 yield 表达式的值是 1：

```
>>> gen.send(None)
1
```

现在，生成器的指令指针距离初始点有 3 个字节码，部分经过编译后的 Python 的 56 个字节：

```
>>> gen.gi_frame.f_lasti
3
>>> len(gen.gi_code.co_code)
56
```

生成器可以随时从任何函数中恢复，因为生成器的栈帧实际上不在堆栈上：它在堆上。它在调用层次结构中的位置不是固定的，并且不需要遵循常规函数执行的先进先后顺序。

我们可以将值 “hello” 发送到生成器中，它成为 `yield` 表达式的结果，并且生成器继续进行直到生成 2：

```
>>> gen.send('hello')
result of yield: hello
2
```

现在，栈帧中包含局部变量 `result`：

```
>>> gen.gi_frame.f_locals
{'result': 'hello'}
```

再次调用 send，生成器从第二个 yield 那里继续运行1，通过引发 `StopIteration` 结束。

```
>>> gen.send('goodbye')
result of 2nd yield: goodbye
Traceback (most recent call last):
  File "<input>", line 1, in <module>
StopIteration: done
```

异常具有值，即生成器的返回值 `'done'`。

## 使用生成器创建协程

因此，生成器可以暂停，可以使用一个值进行恢复，并且它具有返回值。听起来像是构建异步编程模型的好方法，它没有面条式的回调！我们要构建一个“协程”：一个与程序中其他例程协同调度的例程。我们的协程将是 Python 标准 asyncio 库中的协程的简化版本。与 asyncio 一样，我们将使用生成器、asyncio 和 yield from 语句。

首先，我们需要一种方式来表示协程正在等待的某些未来的结果。下面是一个精简版的实现：

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

future 实例初始时为等待 (pending)，通过调用 `set_result` 设置为被处理 (resolved)。

让我们调整 fetcher  程序以使用 futures 和协程。我们用回调编写了 fetch：

```python
class Fetcher:
    def fetch(self):
        self.sock = socket.socket()
        self.sock.setblocking(False)
        try:
            self.sock.connect(('xkcd.com', 80))
        except BlockingIOError:
            pass
        selector.register(self.sock.fileno(),
                          EVENT_WRITE,
                          self.connected)

    def connected(self, key, mask):
        print('connected!')
        # And so on....
```

fetch 方法首先连接套接字，然后注册在套接字准备好时执行的回调 `connected`。现在，我们可以将这两个步骤合并为一个协程：

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
            f.set_result(None) # 设置结果，并执行回调

        selector.register(sock.fileno(),
                          EVENT_WRITE,
                          on_connected)
        yield f
        selector.unregister(sock.fileno())
        print('connected!')
```

现在 fetch 是生成器函数，而非普通的函数，因其包含一个 yield 表达式。我们创建一个等待的 future，然后 yield 它以暂停 fetch，直到套接字准备好。内部函数 `on_connected` 处理这个 future。

但处理好 future 后，如何恢复生成器呢？我们需要一个协程的 driver。我们称其为 task:

```python
class Task:
    def __init__(self, coro):
        self.coro = coro # 协程
        f = Future() # 创建一个 future
        f.set_result(None) # 这里有必要吗？
        self.step(f) # 驱动这个协程

    def step(self, future):
        try:
            next_future = self.coro.send(future.result) # 驱动协程，下次再执行时，会接着上次的继续
        except StopIteration:
            return

        next_future.add_done_callback(self.step) # future 增加一个回调

# Begin fetching http://xkcd.com/353/
fetcher = Fetcher('/353/')
Task(fetcher.fetch())

loop()
```

`task` 通过向生成器传递 `None` 来启动生成器。`fetch` 运行直至 yield 一个 future，该 future 被 `task` 捕获为 `next_future`。当套接字完成连接后，事件循环运行回调 `on_connected`，以处理 `future`、调用 `step` 和恢复 `fetch`。

## 使用 `yield from` 分解协程

连接套接字后，我们将发送 HTTP GET 请求并读取服务器响应。这些步骤不再需要分散在回调间。我们将它们收集到相同的生成器函数中：

```python
    def fetch(self):
        # ... connection logic from above, then:
        sock.send(request.encode('ascii'))

        while True:
            f = Future()

            def on_readable():
                f.set_result(sock.recv(4096))

            selector.register(sock.fileno(),
                              EVENT_READ,
                              on_readable)
            chunk = yield f
            selector.unregister(sock.fileno())
            if chunk:
                self.response += chunk
            else:
                # Done reading.
                break
```

该代码从套接字读取整个消息，通常看起来很有用。我们如何将其从 fetch 分解为子程序呢？现在，轮到 Python 3 中的 yield from 登场了。它使一个生成器可以委派给另一个生成器。

要了解怎么做到，让我们回到简单的生成器示例：

```
>>> def gen_fn():
...     result = yield 1
...     print('result of yield: {}'.format(result))
...     result2 = yield 2
...     print('result of 2nd yield: {}'.format(result2))
...     return 'done'
...   
```

要从另一个生成器调用此生成器，可以使用 yield from 委托给它：

```
>>> # Generator function:
>>> def caller_fn():
...     gen = gen_fn()
...     rv = yield from gen
...     print('return value of yield-from: {}'
...           .format(rv))
...
>>> # Make a generator from the
>>> # generator function.
>>> caller = caller_fn()
```

caller 生成器的行为就像 gen 一样，它委托给的生成器是：

```
>>> caller.send(None)
1
>>> caller.gi_frame.f_lasti
15
>>> caller.send('hello')
result of yield: hello
2
>>> caller.gi_frame.f_lasti  # Hasn't advanced.
15
>>> caller.send('goodbye')
result of 2nd yield: goodbye
return value of yield-from: done
Traceback (most recent call last):
  File "<input>", line 1, in <module>
StopIteration
```

虽然 `caller` 被 `gen` 所 `yield`，但 `caller` 并没有前进。注意。即使内部生成器 `gen` 从一个 `yield` 语句前进到下一个 yield 语句，它的指令指针仍位于 `yield from` 语句的位置 15 处。在外部的 `caller` 看来，我们无法确定 `yield` 的值是来自 `caller` 还是它所委托的生成器。从 `gen` 内部，我们无法判断该值由 `caller` 发送还是由它的外部发送。`yield from` 语句是一个光滑的管道，通过该通道，`yield` 的值可以流入或流出 `gen`，直到 `gen` 完成工作。

协程可以通过 `yield from` 将任务委托给子程序，并收到任务的结果。注意，在上面，`claller `打印了 "return value of yield-from: done"。当 `gen` 完成时，它的返回值变成了 `caller` 中 `yield from` 的值：

```
    rv = yield from gen
```

之前，当我们批评基于回调的异步编程时，最直接的抱怨是关于“堆栈撕裂”：当回调引发异常时，堆栈跟踪通常无法发挥作用，仅能显示事件循环正在运行回调，却不知道为什么。协程是如何应对的呢？

```
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

这有用多了！堆栈跟踪显示，当抛出错误时，`caller_fn` 委托给 `gen_fn`。更好的是，我们可以将对子协程的调用包装在异常处理程序中，这与普通子例程相同：

```
>>> def gen_fn():
...     yield 1
...     raise Exception('uh oh')
...
>>> def caller_fn():
...     try:
...         yield from gen_fn()
...     except Exception as exc:
...         print('caught {}'.format(exc))
...
>>> caller = caller_fn()
>>> caller.send(None)
1
>>> caller.send('hello')
caught uh oh
```

因此，就像常规子程序一样，可以使用子协程分解逻辑。让我们从 fetcher 中分解一些有用的子协程。下面编写 `read` 协程以接收一个块：

```python
def read(sock):
    f = Future()

    def on_readable():
        f.set_result(sock.recv(4096))

    selector.register(sock.fileno(), EVENT_READ, on_readable)
    chunk = yield f  # Read one chunk.
    selector.unregister(sock.fileno())
    return chunk
```

在 `read` 的基础上使用 `read_all` 协程来接收整个消息：

```python
def read_all(sock):
    response = []
    # Read whole response.
    chunk = yield from read(sock)
    while chunk:
        response.append(chunk)
        chunk = yield from read(sock)

    return b''.join(response)
```

如果以正确的方法看待，`yield from `语句将消失，并且这些看起来就像常规函数在阻塞 I/O。但实际上， `read` 和 `read_all` 是协程。`read` 中的 yield 会暂停 `read_all`，直到 I/O 完成。当 `read_all` 暂停时，事件循环进行别的工作并等待别的 I/O 事件；一旦事件就绪，`read_all` 将在下一个循环周期中使用 `read` 的结果进行恢复。

在堆栈的起始点，`fetch` 调用 `read_all`：

```python
class Fetcher:
    def fetch(self):
         # ... connection logic from above, then:
        sock.send(request.encode('ascii'))
        self.response = yield from read_all(sock)
```

很神奇，`Task` 类不需要修改。它像以前一样驱动外部的 `fetch` 协程：

```python
Task(fetcher.fetch())
loop()
```

当 read 函数 yield 一个 future 时，task 通过 yield from 语句的通道接收它，就像 future 从 fetch 中直接 yield 一样。当事件循环处理 future 时，task 将它的忽而过发送给 fetch，并且该值通过 read 接受，就像 task 被 read 直接驱动一样：


为了完善的协程实现，我们弥补了如下缺陷：我们的代码在等待 future 时使用 yield，但在将其委托给子协程时使用 yield from。如果每当协程暂停时都使用 yield from，则会更加完善。这样，协程就不必关心它等待什么类型的东西。

我们利用了 Python 在生成器和迭代器之间的深层对应特性。对于调用者而言，推进生成器与推进迭代器相同。因此，我们通过实现一种特殊的方法使 Future 类变得可迭代：

```python
    # Method on Future class.
    def __iter__(self):
        # Tell Task to resume me here.
        yield self
        return self.result
```
future 的 __iter__ 方法是一个 yield 自身的协程。我们将以下代码：

```python
# f is a Future.
yield f
```

取代为：

```python
# f is a Future.
yield from f
```

而外部其他部分不变！驱动者 Task 通过调用 send 接收 future，当 future 处理完后，它将新结果发送回协程。

全部使用 yield from 的优点是什么？为什么比在等待 future 时使用 yield 以及在委托协程时使用 yield from 这种方式更好？那是因为现在方法的实现不会调用者的影响，可以自由更改。被调用方可能是一个返回 future （用于处理值）的常规方法，也可能是一个包含 yield from 语句并返回值的协程。不管哪种情况，调用方只需要 yield from 该方法并等待结果。

对于异步用户来说，使用协程进行编码比在此处看到的要简单得多。在上面的代码中，我们从第一原则实现了协程，因此你看到了回调、tasks 和 futures。你甚至看到了非阻塞套接字和 select 调用。但是，当在使用 asyncio 构建应用程序时，这些都不会出现在代码中。如我们所承诺的，可以轻松获取URL：

```python
    @asyncio.coroutine
    def fetch(self, url):
        response = yield from self.session.get(url)
        body = yield from response.read()
```

目前一切顺利，现在回到了最初的任务：使用 asyncio 编写一个异步 Web 爬虫。

## 协程协调

我们在一开始描述了希望爬虫如何工作，现在可以用 asyncio 协程库来实现它了。

爬虫程序将获取第一页，解析其链接，并将其添加到队列中。之后，它爬取整个网站，并发获取页面。但是为了降低客户端和服务器上的负载，我们限制最大并发数量。每当任务获取到页面后，都立即从队列中取出下一个链接。有一段时期程序没太多事可干，因此一些任务必须停下来。但当访问包含新链接的页面时，队列突然增加，暂停的工作人员会醒来并开始抓取网页。最后，一旦完成工作，程序必须退出。

想象一下，如果工作者是线程，我们怎样编写爬虫？我们可以使用 Python 标准库中同步的 queue。每次将项目放入队列时，队列都会增加其 “tasks” 计数。线程在完成一项任务后调用 task_done。Queue.join上的主线程将阻塞，直到队列中的每一项都被 task_done 调用匹配，然后退出。

协程使用与异步队列完全相同的模式！首先我们引入：

```python
try:
    from asyncio import JoinableQueue as Queue
except ImportError:
    # In Python 3.5, asyncio.JoinableQueue is
    # merged into Queue.
    from asyncio import Queue
```

我们在 crawler 类中 worker 的共享状态，并在 `crawl` 中编写主逻辑。我们从一个协程开始启动 crawl，并运行 asyncio 的事件循环，直到 crawl 结束。

```python
loop = asyncio.get_event_loop()

crawler = crawling.Crawler('http://xkcd.com',
                           max_redirect=10)

loop.run_until_complete(crawler.crawl())
```

爬虫从一个根 URL 和 max_redirect 开始，max_redirect 是为获取任一个链接的最大重定向次数。它将 (URL, nax_redirect) 对放入队列中。（下文将解释原因）

```python
class Crawler:
    def __init__(self, root_url, max_redirect):
        self.max_tasks = 10
        self.max_redirect = max_redirect
        self.q = Queue()
        self.seen_urls = set()

        # aiohttp's ClientSession does connection pooling and
        # HTTP keep-alives for us.
        self.session = aiohttp.ClientSession(loop=loop)

        # Put (URL, max_redirect) in the queue.
        self.q.put((root_url, self.max_redirect))
```

队列中未完成的 tasks 的数量现在是 1。回到主程序中，启用事件循环和 crawl 方法：

```python
loop.run_until_complete(crawler.crawl())
```

crawl 协程启动了 workers。它就像一个主线程：所有 tasks 完成前在 join 处阻塞住，而 workers 在后台运行。

```python
    @asyncio.coroutine
    def crawl(self):
        """Run the crawler until all work is done."""
        workers = [asyncio.Task(self.work())
                   for _ in range(self.max_tasks)]

        # When all work is done, exit.
        yield from self.q.join()
        for w in workers:
            w.cancel()
```

如果 workers 是线程，那么我们不会一次全部启动。为了避免线程创建开销，线程池通常按需增长。但协程开销很小，只需要设置最大数量即可。

有趣的是，我们该怎么关闭哦爬虫。当 join future resolves，worker tasks 仍处于活动状态单已暂停：它们等待更多 URL，但不会出现。因此主协程在退出前将其取消。不然 Python 解释器关闭并调用所有对象的销毁器时，活动的 task 将抗议：

```
ERROR:asyncio:Task was destroyed but it is pending!
```

cancel 如何工作呢？生成器还有我们尚未展示的功能。你可以从外部想向生成器中抛出异常：

```
>>> gen = gen_fn()
>>> gen.send(None)  # Start the generator as usual.
1
>>> gen.throw(Exception('error'))
Traceback (most recent call last):
  File "<input>", line 3, in <module>
  File "<input>", line 2, in gen_fn
Exception: error
```

生成器通过 throw 来抛出，但现在却引发了异常。若生成器的调用栈中未捕获该异常，则异常会冒泡到顶部。因此要取消 task 的协程：

```python
    # Method of Task class.
    def cancel(self):
        self.coro.throw(CancelledError)
```

无论生成器在哪里暂停，或者处于 yield from 语句，他都将恢复或抛出异常。我们通过 task 的 step 方法来处理取消：

```python
    # Method of Task class.
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

现在 task 知道它已被取消，因此当它被销毁时，不再惧怕死亡。一旦 crawl 取消了 workers，它便会退出。事件循环看到协程完成，也会退出。

```python
loop.run_until_complete(crawler.crawl())
```

crawl 方法做了主协程必须做的所有事情：从队列中获取 URL，爬取并解析新的链接。每个 worker 独立运行 work 协程：

```python
    @asyncio.coroutine
    def work(self):
        while True:
            url, max_redirect = yield from self.q.get()

            # Download page and add new links to self.q.
            yield from self.fetch(url, max_redirect)
            self.q.task_done()
```

Python 解释器看到代码中包含 yield from 语句时，会将其编译为生成器函数。所以在 crawl 中，当主协程调用 slef.work 10 次时，它并不会运行该方法，只会使用该段代码的引用创建 10 个生成器对象。它将每个包装在一个 Task 中。Task 接收生成器 yield 的每个 future，通过调用 send 并传递每个 future 处理后的结果来驱动生成器。由于生成器有各自的栈帧，所以他们使用独立的局部变量和指令指针来独立运行。

worker 之间通过队列来协调。通过下面这句来获取新链接：

```python
    url, max_redirect = yield from self.q.get()
```

队列的 get 方法本身就是一个协程：它将元素放入队列之前暂停，然后恢复运行并返回该元素。

顺便一提，当主协程取消时，worker 会在 crawl 结束的地方暂停。从协程的角度来看，当 yield from 抛出 CancelledError 时，他在循环的最后行程结束。

当 worker 获取页面时，它将解析链接并塞入队列，然后调用 task_done 减少计数器。最终，一个 worker 获取到了其包含的所有 URL 都已经获取过的页面，并且队列中不再有任务。因此该 worker 对 task_done 的调用将计数器减少到 0。然后等待队列的 join 方法的 crawl 结束运行。

我们承诺过要解释为什么队列中的元素是成对的，例如：

```
# URL to fetch, and the number of redirects left.
('http://xkcd.com/353', 10)
```

新的URL剩余 10 次重定向。提取此特定的 URL 会导致重定向到带有斜杠的新位置。我们减少剩余的重定向次数，并将下一个位置放入队列：

```
# URL with a trailing slash. Nine redirects left.
('http://xkcd.com/353/', 9)
```

我们使用的 aiohttp 库，在默认情况下使用重定向，并提供最终的响应。但我们不用这种默认行为，而将重定向交给爬虫程序处理，这样就可以合并相同的目标地址：如果 URL 已经在 self.seen_urls 中了，并且在别的入口点已经开始抓取该链接：

爬虫程序获取 “foo”，并发现它重定向到 “baz”，因此将 “baz” 添加到队列和 seen_urls 中。如果获取的下一页是 “bar” 也重定向到 “baz”，则程序不会再次将 “baz” 塞入队列。如果响应是页面而不是重定向，fetch 则对其进行解析以获取链接并将新链接放入队列。

```python
    @asyncio.coroutine
    def fetch(self, url, max_redirect):
        # Handle redirects ourselves.
        response = yield from self.session.get(
            url, allow_redirects=False)

        try:
            if is_redirect(response):
                if max_redirect > 0:
                    next_url = response.headers['location']
                    if next_url in self.seen_urls:
                        # We have been down this path before.
                        return

                    # Remember we have seen this URL.
                    self.seen_urls.add(next_url)

                    # Follow the redirect. One less redirect remains.
                    self.q.put_nowait((next_url, max_redirect - 1))
             else:
                 links = yield from self.parse_links(response)
                 # Python set-logic:
                 for link in links.difference(self.seen_urls):
                    self.q.put_nowait((link, self.max_redirect))
                self.seen_urls.update(links)
        finally:
            # Return connection to pool.
            yield from response.release()
```

如果这是多线程代码，则在竞争条件下会很糟糕。例如，worker 检查链接是否在 seen_urls 中，如果不在，则将其放入队列并添加到 seen_urls中。如果在两个操作间产生中断，则另一个 worker 可能会从不同的页面解析相同的链接，同样在发现它不在see_urls 中时，会将其添加到队列中。现在，同一链接放入队列两次，从而（最好情况下）导致重复的工作和错误的统计信息。

但是，协程仅容易受 yield from 语句的干扰。这个关键区别，使协程代码比多线程代码更不容易出现竞争：多线程代码必须通过获取锁来显式地进入临界区，否则将被中断。默认情况下，Python 协程是不可中断的，只有在显式地 yield 时才会让出控制权。

我们不再需要像基于回调的程序中那样的 fetcher 类。该类是减少回调的一种解决方法：在等待 I/O 时，由于局部变量不会在调用之间保留，它们需要一些位置来存储状态。然而，fetcher 协程可以像常规函数一样将其状态存储在局部变量中，因此不再需要这种类。

fetch 处理完服务器响应后，将返回到调用方 work。work 方法调用队列的 task_done，然后从队列中获取下一个要提取的 URL。

当 fetch 将新的链接放入队列时，它将增加未完成任务的数量，并使等待 q.join 的主协程暂停。但是，如果没有看不见的链接，并且这是队列中的最后一个 URL，那么当 work 调用 task_done 时，未完成的任务的数量将降至零。该事件将使 join 暂停并且主协程完成。

当访存将新的链接放入队列时，它将增加未完成任务的数量，并使等待q.join的主协程暂停。 但是，如果没有看不见的链接，并且这是队列中的最后一个URL，那么当工作调用task_done时，未完成的任务的数量将降至零。 该事件将使连接暂停并且主要协程完成。

协调 work 与 主协程的队列代码如下：

```python
class Queue:
    def __init__(self):
        self._join_future = Future()
        self._unfinished_tasks = 0
        # ... other initialization ...

    def put_nowait(self, item):
        self._unfinished_tasks += 1
        # ... store the item ...

    def task_done(self):
        self._unfinished_tasks -= 1
        if self._unfinished_tasks == 0:
            self._join_future.set_result(None)

    @asyncio.coroutine
    def join(self):
        if self._unfinished_tasks > 0:
            yield from self._join_future
```

主协程 crawl 被 join 所 yield。因此，当最后一个 worker 将未完成的任务计数减为零时，表示 crawl 恢复并完成了工作。

```python
loop.run_until_complete(self.crawler.crawl())
```

程序如何结束？由于 crawl 是生成器函数，因此对其进行调用将返回生成器。为了驱动生成器，asyncio 将其包装在一个 task 中：

```python
class EventLoop:
    def run_until_complete(self, coro):
        """Run until the coroutine is done."""
        task = Task(coro)
        task.add_done_callback(stop_callback)
        try:
            self.run_forever()
        except StopError:
            pass

class StopError(BaseException):
    """Raised to stop the event loop."""

def stop_callback(future):
    raise StopError
```

task 完成后将抛出 StopError，循环将 StopError 视作已正常完成的信号。

task 的 add_done_callback 和 restult 方法又是什么？你或许认为 task 类似于 future。你的直觉是对的。我们必须承认隐藏的 Task 类的细节：一个 task 也是一个 future。

```python
class Task(Future):
    """A coroutine wrapped in a Future."""
```

通常，future 可以通过某个地方调用 set_result 来处理。但是，当协程停止时，task 会自行处理。请记住，从我们之前对 Python 生成器的探索中可以知道，当生成器返回时，它会抛出特殊的 StopIteration 异常。

```python
    # Method of class Task.
    def step(self, future):
        try:
            next_future = self.coro.send(future.result)
        except CancelledError:
            self.cancelled = True
            return
        except StopIteration as exc:

            # Task resolves itself with coro's return
            # value.
            self.set_result(exc.value)
            return

        next_future.add_done_callback(self.step)
```

因此，当事件循环调用 task.add_done_callback（stop_callback）时，它准备被 task 停止。这里再次是 run_until_complete：

```python
    # Method of event loop.
    def run_until_complete(self, coro):
        task = Task(coro)
        task.add_done_callback(stop_callback)
        try:
            self.run_forever()
        except StopError:
            pass
```

当 task 捕获 StopIteration 并自行处理时，回调将从循环内引发 StopError。循环停止，调用堆栈退回到 run_until_complete。我们的程序完成了。

## 结论

现代程序越来越经常受 I/O 约束，而不是受 CPU 约束。对于这样的程序，Python 线程是个两头吃亏的选择：全局解释器锁阻止它们真实的并行计算，而抢占式切换使它们易于发生竞争。异步通常是正确的模式。但是随着基于回调的异步代码的增长，它往往会变得混乱不堪。协程是一个整洁的选择。它们自然地包含在子例程中，并具有健全的异常处理和堆栈跟踪功能。

如果我们斜视以使语句的收益模糊，那么协程看起来就像是执行传统阻塞 I/O 的线程。 我们甚至可以将协程与来自多线程编程的经典模式进行协调。 无需重新发明。 因此，与回调相比，协程对于使用多线程的程序员是一个诱人的习惯用法。

但是，当我们睁开眼睛，专注于语句的收益时，我们看到它们在协程让步并允许其他人运行时标记了点。 与线程不同，协程会显示我们的代码可以在何处中断和无法在何处中断。 格里夫·莱夫科维茨（Glyph Lefkowitz）在他的启发性文章“坚强” 14中写道：“线程使本地推理变得困难，本地推理可能是软件开发中最重要的事情。” 但是，显式屈服可以“通过检查例程本身而不是整个系统来了解例程的行为（从而，正确性）”。

本章是在Python和异步技术的复兴中撰写的。 您刚刚学会了设计的基于生成器的协程，已于2014年3月在Python 3.4的“异步”模块中发布。2015年9月，Python 3.5随语言本身内置的协程一起发布。 这些本机协程以新语法“ async def”声明，而不是“ yield from”，而是使用新的“ await”关键字委派给协程或等待Future。

尽管取得了这些进步，但核心思想仍然存在。 Python的新本机协程在语法上将与生成器不同，但工作方式非常相似。 实际上，他们将在Python解释器中共享一个实现。 任务，未来和事件循环将继续在异步中发挥作用。

既然您知道了异步协程的工作原理，就可以在很大程度上忘记细节。 机器被塞在一个精巧的接口后面。 但是，您掌握了基础知识后，便可以在现代异步环境中正确有效地进行编码。
