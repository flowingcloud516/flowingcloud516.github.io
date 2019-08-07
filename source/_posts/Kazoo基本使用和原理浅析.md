---
title: Kazoo基本使用和原理浅析
date: 2019-08-04 11:02:21
tags:
- Python
- ZooKeeper
categories: Python
---

### 关于Kazoo

Kazoo是一个ZooKeeper的python客户端，它是一个纯python的应用，并在底层实现了与zookeeper通讯协议。

它主要有这些特性
- 底层通信异步化
- 基于Python装饰器，提供一次watch、终身受用的功能
- 提供 gevent/eventlet 兼容功能

其他的ZooKeeper-Python客户端<sup>[1]</sup>：
1. **txzookeeper** 基于Twisted/Python（对并发部分进行过更改的Python版本），2013年停止维护
2. **zc.zk** 2014年停止维护，zc.zk2 开始使用 kazoo

### 关于本文
1. 本文基于 kazoo 2.6.1 版本代码进行分析总结。
2. 本文中提到的“异步结构”，是根据handler选择变化的，在SequentialThreadingHandler中是异步线程，而在SequentialGeventHandler中是异步协程。

### 基本使用方式

1. 启动客户端
```Python
zk = KazooClient(hosts="127.0.0.1:2181") 
# can be multi hosts, like
# zk = KazooClient(hosts="192.168.0.1:2181,192.168.0.2:2181,192.168.0.3:2181")
zk.start()
```
2. CURD 命令
```python
# Ensure a path, create if necessary
zk.ensure_path("/path/to/my/node")

# Create a node with data
zk.create("/my/favorite/node", b"a value")

# read api
data, stat = zk.get("/my/favorite")
children = zk.get_children("/my/favorite")
if zk.exists("/my/favorite"):
    # Do something
    pass
```
3. watch 监听
- 方式1：采用ZooKeeper提供的默认watch监听事件，可使用的函数：
    - get()
    - get_children()
    - exists()
```python
def my_func(event):
    # check to see what the children are now

# Call my_func when the children change
children = zk.get_children("/my/favorite/node", watch=my_func)
```
   
- 方式2：采用kazoo提供的Watch装饰器，可使用的监听器有
    - ChildrenWatch 观察子节点变化
    - DataWatch  观察节点值的变化
```python
@zk.ChildrenWatch("/my/favorite/node")
def watch_children(children):
    print("Children are now: %s" % children)
```

4. Retry
在kazoo中，重试机制是通过抽象出一个类来进行控制的，这个类就是KazooRetry。KazooRetry实现了__call__函数，因此可以被直接当做函数使用。使用方式如下： 
```python
result = zk.retry(zk.get, "/path/to/node")

# Custom Retries
kr = KazooRetry(max_tries=3, ignore_expire=False)
result = kr(client.get, "/some/path")

# init client using Retry
rt = KazooRetry(max_tries=3, ignore_expire=False)
zk = KazooClient(hosts="127.0.0.1:2181", connection_retry=rt, command_retry=rt)
zk.start() 
```
    需要特别指出的是，在KazooRetry中会捕获一种ForceRetryError的异常来触发重试，这个机制被connection用于连接轮询。 

### Kazoo 状态转换
![image](/images/blog/python/kazoo_states.png "kazoo 状态转换")

Source/Target | LOST | CONNECTED | SUSPEND
---|---|---|---
LOST | 客户端初始化即为LOST状态 | 成功连接ZK服务端后，转移到CONNECTED | 
CONNECTED | 只有鉴权失败会返回LOST状态 |  | ZK Server 出现问题，重连ZK节点，先转移到SUSPEND
SUSPEND | 重建连接失败，如Session过期 | 重建连接成功，且鉴权有效 | 

### Kazoo 代码原理浅析
Kazoo底层通信用**异步实现**，但也提供一些同步命令，这些同步命令是在异步基础上增加等待来模拟同步的。

异步的实现方式可以通过handler类型进行选择。Kazoo提供了3种Handler：SequentialThreadingHandler、SequentialGeventHandler、SequentialEventletHandler，分别用线程、gevent、eventlet的方式来实现异步。

以下我们将kazoo的主要模块拆开，分别讲解原理，然后通过一个kazoo.get()命令的流程来说明kazoo具体是如何工作的。

_P.S. 如果觉得代码太多不想细究，只想理解大致原理，可跳过中间的模块解析部分，只看**模块架构**和**get工作流程**~_

#### 模块架构
![image](/images/blog/python/kazoo_struct.png "kazoo 模块架构")

Kazoo 主要有这3大模块：client、异步handler、连接connection，其关系如上图所示。   
1. client：管理配置项，并对外提供get、set等操作命令；
2. connection：通过handler提供的异步功能，与ZK进行通信，而通信过程会经过kazoo自己实现的serialization方法进行；
3. handler：管理异步队列和异步操作，kazoo客户端初始化，后主要有2个异步队列和1个异步结果操作：
    1. zk_loop：connection模块通过**异步轮询**的方式与ZK Server进行通信，处理request和response；
    2. AsyncResult：异步结果，不同的handler有不同的实现，SequentialGeventHandler中直接调用gevent.event.AsyncResult，在SequentialThreadingHandler中kazoo模拟了一个异步请求，并通过completion_queue队列的线程处理异步结果的callback；
    3. callback_queue：处理watch回调。

#### Connection
connection模块中有ConnectionHandler类，用于处理与ZK的异步连接。我们来看看start函数：
```python
def start(self):
    """Start the connection up"""
    if self.connection_closed.is_set():
        rw_sockets = self.handler.create_socket_pair()   # 调用本地方法
        self._read_sock, self._write_sock = rw_sockets   # 获得一个 socket pair
        self.connection_closed.clear()
    if self._connection_routine:                         # 验证当前是否存在异步通信线程/协程
        raise Exception("Unable to start, connection routine already "
                        "active.")
    self._connection_routine = self.handler.spawn(self.zk_loop)
                                                         # 启动一个异步通信线程/协程 

# ....

def zk_loop(self):
    """Main Zookeeper handling loop"""
    self.logger.log(BLATHER, 'ZK loop started')

    self.connection_stopped.clear()

    retry = self.retry_sleeper.copy()                   # 获取client的retry方法
    try:
        while not self.client._stopped.is_set():        # 注意，轮询不是靠这个while进行的，而是靠下面的retry
            # If the connect_loop returns STOP_CONNECTING, stop retrying
            if retry(self._connect_loop, retry) is STOP_CONNECTING:         # 主要功能函数，连接轮询，处理请求和反馈
                break                                                       # 默认只会retry 1次
    except RetryFailedError:
        self.logger.warning("Failed connecting to Zookeeper "
                            "within the connection retry policy.")
    finally:
        self.connection_stopped.set()
        self.client._session_callback(KeeperState.CLOSED)
        self.logger.log(BLATHER, 'Connection stopped')  
# ...

def _connect_loop(self, retry):
    # Iterate through the hosts a full cycle before starting over
    status = None
    host_ports = self._expand_client_hosts()        # 获取 zk_server list

    # Check for an empty hostlist, indicating none resolved
    if len(host_ports) == 0:
        return STOP_CONNECTING

    for host, hostip, port in host_ports:           # 对于每个 zk_server，轮询尝试建立连接
        if self.client._stopped.is_set():
            status = STOP_CONNECTING
            break
        status = self._connect_attempt(host, hostip, port, retry)   # 处理request和response的主体函数
        if status is STOP_CONNECTING:
            break

    if status is STOP_CONNECTING:                   # 如果当前连接断掉，则返回
        return STOP_CONNECTING
    else:
        raise ForceRetryError('Reconnecting')       # 抛出异常，强行触发上层的retry，实现轮询
```
可以看到，connection调用start()启动之后，进行了3步主要操作：
1. 生成一个 socket_pair；
2. 验证当前是否已经有异步结构，如果有，则抛出异常并退出；
3. 通过client的异步handler.spawn()生成一个异步线程，然后通过zk_loop函数与ZK Server进行通信。

在zk_loop函数中，通过while循环进行轮询，通过_connect_loop处理请求信息。默认retry只会重试1次，重试之前会默认sleep一个随机时间。

在_connect_loop函数中，拿到zk_server的host list，并通过_connect_attempt函数，逐一尝试建立连接。_connect_attempt函数中会进行如下操作：
1. 收集当前命令，通过写socket发送请求给ZK Server；
2. 如果当前没有命令，发送一个ping过去，保持连接活性；
3. 通过_read_socket函数，从读socket接受response；
4. 在_read_socket函数中，把返回值写入异步结果AsyncResult，如果返回值中存在watch，则调用相应的callback。

关于connection机制的介绍，可参考引用[3]，介绍的很详细。

#### Handler
Kazoo使用handler来进行异步操作，提供了3种异步操作的handler：SequentialThreadingHandler、SequentialGeventHandler、SequentialEventletHandler，分别用线程、gevent、eventlet的方式来实现异步。

由于默认使用SequentialThreadingHandler提供异步方案，我们以SequentialThreadingHandler来进行分析。

先看看handler提供的函数接口：
```python
def start(self):
    """Start the worker threads."""
    # 代码略

def stop(self):
    """Stop the worker threads and empty all queues."""
    # 代码略

def select(self, *args, **kwargs):
    # if we have epoll, and select is not expected to work
    # use an epoll-based "select". Otherwise don't touch
    # anything to minimize changes
    # 代码略

def socket(self):
    return utils.create_tcp_socket(socket)

def create_connection(self, *args, **kwargs):
    return utils.create_tcp_connection(socket, *args, **kwargs)

def create_socket_pair(self):
    return utils.create_socket_pair(socket)

def event_object(self):
    """Create an appropriate Event object"""
    return threading.Event()

def lock_object(self):
    """Create a lock object"""
    return threading.Lock()

def rlock_object(self):
    """Create an appropriate RLock object"""
    return threading.RLock()

def async_result(self):
    """Create a :class:`AsyncResult` instance"""
    return AsyncResult(self)

def spawn(self, func, *args, **kwargs):
    t = threading.Thread(target=func, args=args, kwargs=kwargs)
    t.daemon = True
    t.start()
    return t

def dispatch_callback(self, callback):
    """Dispatch to the callback object

    The callback is put on separate queues to run depending on the
    type as documented for the :class:`SequentialThreadingHandler`.

    """
    self.callback_queue.put(lambda: callback.func(*callback.args))
```

1. **spawn() 函数**
这些函数接口中，最重要的函数是spawn()，它经常在各处被调用，用于提供异步的功能。具体在SequentialThreadingHandler中，spawn()会调用python的threading模块，生成一个线程，并将线程设置为守护线程。

2. **start() 函数**
kazoo客户端初始化后会调用start()函数，启动异步。在SequentialThreadingHandler中，start()函数会初始化2个队列，并为其各自生成一个线程进行消费：
```python
def start(self):
    """Start the worker threads."""
    with self._state_change:
        if self._running:
            return

        # Spawn our worker threads, we have
        # - A callback worker for watch events to be called         用于唤醒callback
        # - A completion worker for completion events to be called  用于设置异步结果AsyncResult的值
        for queue in (self.completion_queue, self.callback_queue):   
            w = self._create_thread_worker(queue)                 # 此处将调用_create_thread_worker生成2个worker，持续消费队列
            self._workers.append(w)
        self._running = True
        python2atexit.register(self.stop)
        
def _create_thread_worker(self, queue):
    def _thread_worker():  # pragma: nocover    消费队列的函数
        while True:
            try:
                func = queue.get()
                try:
                    if func is _STOP:
                        break
                    func()
                except Exception:
                    log.exception("Exception in worker queue thread")
                finally:
                    queue.task_done()
                    del func  # release before possible idle
            except self.queue_empty:
                continue
    t = self.spawn(_thread_worker)          # 生成消费队列的线程
    return t
```

3. **event_object() 函数**
生成一个锁，用在并发安全控制。由于需要考虑在gevent/eventlet并发模型下的锁控制，所以将锁生成抽象到handler来处理。在SequentialThreadingHandler中直接返回threading.Event()。

#### Watch
Kazoo提供了2种watch的方式，我们分别介绍其原理。
1. Default watch
该方式通过将watch的回调函数作为参数，传递给get()、get_children()和exists()接口完成watch的功能。   
以get()为例，通过源码，可以看到get()函数的操作流程如下：
```python
def get(self, path, watch=None):
    """Get the value of a node.
       .. (the doc) ..
    """
    return self.get_async(path, watch=watch).get()

def get_async(self, path, watch=None):
    """
       .. (the doc) ..
    """
    if not isinstance(path, string_types):      # 进行参数类型校验
        raise TypeError("Invalid type for 'path' (string expected)")
    if watch and not callable(watch):
        raise TypeError("Invalid type for 'watch' (must be a callable)")

    async_result = self.handler.async_result()
    self._call(GetData(_prefix_root(self.chroot, path), watch),     # GetData 会将get请求序列化成命令，用于与ZK Server直接通信
               async_result)
    return async_result
```
实际上，watch的步骤如下：
    1. get()会调用get_async()，将参数和回调函数watcher传递过去；   
    2. 而get_async()函数会将get命令与watcher封装，通过_call()函数放入自身的请求队列；
    3. 后面的流程一分为二：一方面，序列化的命令会将watch的标志位置为1，而在处理请求的时候watcher函数会被放到client的watcher中；
    4. 当读到的请求发回一个需要唤醒watcher的路径时，会调用client中对应的watcher，执行回调函数。
2. Decorator watch
Kazoo提供了2个watch装饰器，使用装饰器可以达到一次watch永远生效的作用。一般情况下ZooKeeper的watch事件只会生效1次，可以想象，这里其实只是一个语法糖：在初始化装饰器时注册一次watch，当触发watch事件后，再注册同样的watch即可。   
原理很简单，但是具体的实现还是有学习价值的。以DataWatch为例：
```python
def __init__(self, client, path, func=None, *args, **kwargs):
    # ...
    # 篇幅有限，此处省略了很多文档和代码
    # ...
    if func is not None:
        self._used = True
        self._client.add_listener(self._session_watcher)    # 注册session listener，当状态改变时触发，下同
        self._get_data()                # 获取数据

def __call__(self, func):
    # ...
    # 篇幅有限，此处省略了很多文档和代码
    # ...
    self._client.add_listener(self._session_watcher)
    self._get_data()
    return func
    
@_ignore_closed
def _get_data(self, event=None):
    # ...
    # 篇幅有限，此处省略了很多文档和代码
    # ...
    try:
        data, stat = self._retry(self._client.get,
                                 self._path, self._watcher)     # 通过retry，调用get命令，并将watcher注册
    except NoNodeError:
        data = None

        # This will set 'stat' to None if the node does not yet
        # exist.
        stat = self._retry(self._client.exists, self._path,     # 如果当前node不存在，会先尝试watch节点
                           self._watcher)                       # 然后生成一个异步结构来持续监听节点data变化
        if stat:
            self._client.handler.spawn(self._get_data)
            return
    # ...
    # 码多不看
    # ...
```

#### 通信流程
至此，kazoo的主要模块介绍完毕。下面通过一个get()命令的执行顺序，将kazoo工作机制展现出来。不多说，上图：
![image](/images/blog/python/kazoo_invoke_flow.png "get请求流程(图中蓝色为同步过程，红色为异步过程)")


### _** Annotations & References: **_
[1] [ZKClient Bindings](https://cwiki.apache.org/confluence/display/ZOOKEEPER/ZKClientBindings)
[2] [Kazoo Official](https://kazoo.readthedocs.io/en/latest/)
[3] [zookeeper client API实现(python kazoo 的实现)](https://www.cnblogs.com/cloud-zhao/p/5688619.html)