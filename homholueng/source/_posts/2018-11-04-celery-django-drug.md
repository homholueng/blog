---
title: celery 与 django 的致命组合
date: 2018-11-04 12:44:47
categories: 
- Python Daily
tags:
- Celery
- Python
- Django
---


## 1. 背景说明



​	笔者在某个项目中中采用了 event-driven 的架构，并使用 Celery 来作为架构模式中的 worker 角色。Celery 在收到 master 发出的任务执行信号后，即会从数据库中读取任务信息并开始执行，于此同时，在执行过程中还会将必要的信息持久化到数据库中，而在这整个流程中，所有的数据库操作都通过 Django orm 框架来完成。



笔者的开发环境如下：



- mac OS 10.13.4 
- Python 2.7.9
- Django 1.8.11 
- Celery 3.1.18
- Django-celery 3.1.16
- Mysql 5.7.22



## 2. 出现问题



​	Celery worker 在执行的过程中，会偶发性的出现一些由 PyMySQL 抛出的异常，如（出现过的错误包括但不仅限于以下错误）：



- `error: Error -3 while decompressing data: incorrect header check `：读取数据库中数据出错
- `OperationalError: (2006, "MySQL server has gone away (error(32, 'Broken pipe'))") `：数据库连接中断报错
- `error: Command Out Of Sync`：多条进程占用同一条链接出错
- ....



​	通过多次测试和排查，发现问题是由于多个 Celery worker 使用了同一个数据库连接而导致的，由于多个 worker 使用了同一条连接，所以在进行数据库读写的时候，就很有可能会因为读到了由另一个 Worker 写入的脏数据或者是因为事务隔离而导致的数据不一致而引发异常，部分不可恢复的异常抛出后，由于连接被断开，就会导致其他 worker 在使用同一条链接时引发 `MySQL server has gone away` 的错误，而且由于并行程序存调度的不确定性，导致很难精确的复现某个错误，使得问题的根源具有一定的隐蔽性。那么，究竟是什么原因导致多个 worker 使用了同一条数据库连接呢？



## 3. 追根溯源



### 3.1 Celery 多进程（prefork）模式



> **By default multiprocessing is used to perform concurrent execution of tasks**, but you can also use Eventlet. 



Celery 的官方文档中有说明，Celery 默认使用 multiprocessing 库来实现任务的并行，而 multiprocessing 是通过 fork 来实现进程的创建操作的，那么 Celery 的 worker 进程会不会都是由一个父进程通过 fork 创建出来的呢？



在 celery worker 启动后，通过 `ps` 查看子进程的状态，结果如下:



```
$ ps -ef | grep celery
501 46229 54985   0  4:50下午 ttys004    0:01.41 python manage.py celery worker -c 4
501 46251 46229   0  4:50下午 ttys004    0:00.02 python manage.py celery worker -c 4
501 46252 46229   0  4:50下午 ttys004    0:00.02 python manage.py celery worker -c 4
501 46253 46229   0  4:50下午 ttys004    0:00.02 python manage.py celery worker -c 4
501 46254 46229   0  4:50下午 ttys004    0:00.02 python manage.py celery worker -c 4
```



通过进程状态查看工具，可以发现进程 `46251`，`46252`，`46253` ，`46254` 都是由进程 `46299` fork 出来的，而在 fork 操作时子进程会复制父进程内存空间中的所有数据，所以在这个时候，所有的子进程都拿到了父进程中的数据库连接，当多个子进程操作该连接时，就会引发上述问题。



至于为什么子进程会拿到和父进程一样的连接，还得从 Django ORM 连接管理部分的实现说起。



### 3.2 Django ORM 连接管理



既然问题是由于共享数据库连接导致的，Django ORM 中数据库连接管理的实现就是问题的根源所在。



首先，Django ORM 中所有的数据库操作都会通过 `django.db` 模块下的 `connections` 全局变量来获取特定数据库的连接，源码如下：



```python
# django/db/models/sql/query.py
from django.db import connections

def get_compiler(self, using=None, connection=None):
        if using is None and connection is None:
            raise ValueError("Need either using or connection")
        if using:
            connection = connections[using] # 关注点，在这里获取数据库连接
        return connection.ops.compiler(self.compiler)(self, connection, using)
```



而`connections` 声明的源码如下：



```python
# django/db/__init__.py
from django.db.utils import ConnectionHandler
# ...

connections = ConnectionHandler()

# ...
```



原来 `connections` 这个全局变量是一个 `ConnectionHandler` 类，那么这个 `ConnectionHandler` 又是何方神圣呢，`ConnectionHandler` 实现部分源码如下：



```python
# django/db/utils.py
from threading import local

class ConnectionHandler(object):
    def __init__(self, databases=None):
        self._databases = databases
        self._connections = local() # django 通过 ThreadLocal 来管理 [线程 -> 数据库连接] 的映射关系
	
    # ....

    def __getitem__(self, alias): # 关注点
        if hasattr(self._connections, alias):
            return getattr(self._connections, alias)

        self.ensure_defaults(alias)
        self.prepare_test_settings(alias)
        db = self.databases[alias]

        backend = load_backend(db['ENGINE']) # 加载特定的数据库后端
        conn = backend.DatabaseWrapper(db, alias) # 初始化 DatabaseWrapper
        setattr(self._connections, alias, conn) # 存到 ThreadLocal 中
        return conn

    def __setitem__(self, key, value):
        setattr(self._connections, key, value)

    def __delitem__(self, key):
        delattr(self._connections, key)
     # ...
```



当我们通过 `connections[usings]` 获取数据库连接时，就会调用 `ConnectionHandler` 的 `__getitem__()` 方法，此时 `ConnectionHandler` 会先通过我们在 `settings.py` 中的配置的数据库信息加载相应的数据库后端，同时初始化该后端实现的 `DatabaseWrapper` 对象并返回。



`DatabaseWrapper` 是 Django 用于表示数据库连接的一个包装类，由数据库后端自行实现，并且该必须继承自 `BaseDatabaseWrapper`，该类的部分实现如下：



```python
# django/db/backends/base/base.py

class BaseDatabaseWrapper(object):
    """
    Represents a database connection.
    """
    
    # ...

    def __init__(self, settings_dict, alias=DEFAULT_DB_ALIAS,
                 allow_thread_sharing=False):
        self.connection = None # 真正的数据库连接对象
        self.close_at = None # 关注点，连接到期时间，下文会有详细讲解
        self.allow_thread_sharing = allow_thread_sharing
        
        # ...

    def get_connection_params(self):
        raise NotImplementedError('subclasses of BaseDatabaseWrapper may require a get_connection_params() method')

    def get_new_connection(self, conn_params):
        raise NotImplementedError('subclasses of BaseDatabaseWrapper may require a get_new_connection() method')
        
        # ...
	
    def ensure_connection(self): # 在每次使用之前都会确保当前连接是否有效
        if self.connection is None:
            with self.wrap_database_errors:
                self.connect()

    def connect(self):
        conn_params = self.get_connection_params()
        self.connection = self.get_new_connection(conn_params) # 在这里真正建立数据库连接 
        
        # ...
```



绕来绕去，笔者终于找到了真正的数据库连接的藏身之处，Django 通过 `BaseDatabaseWrapper` 来实现对单个数据库连接的封装和管理，而获取数据库连接的方法则交给子类来实现。笔者本地的数据库后端配置如下：



```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': '',  
        'USER': '',  
        'PASSWORD': '',  
        'HOST': 'localhost',
        'PORT': '3306',
        'CONN_MAX_AGE': 3600,
    },
}
```



顺着声明，找到相应后端的 `DatabaseWrapper` 的实现：



```python
# django/db/backends/mysql

try:
    import MySQLdb as Database
except ImportError as e:
    from django.core.exceptions import ImproperlyConfigured
    raise ImproperlyConfigured("Error loading MySQLdb module: %s" % e)

class DatabaseWrapper(BaseDatabaseWrapper):

    def get_new_connection(self, conn_params):
        conn = Database.connect(**conn_params) # 真正的数据库连接在此处建立
        conn.encoders[SafeText] = conn.encoders[six.text_type]
        conn.encoders[SafeBytes] = conn.encoders[bytes]
        return conn
```



至此，Django ORM 中对数据库连接的管理已经非常清晰了：Django 通过 `ConnectionHandler` 来管理整个 Django APP 中的所有数据库连接，并且使用 `ThreadLocal` 来隔离不同线程所能获取到的数据库连接，同时，通过 `DatabaseWrapper` 封装了对真正的数据库连接的操作，整个结构关系如下图所示：

{% asset_img 1532015206_73_w1123_h403.png [类图] %}


### 3.3 验证猜想



在了解了 Django ORM 对数据库连接的管理方式之后，我们就能够来探讨导致多个 worker 共享同一个数据库连接的真正原因了。为了验证不同的子进程确实使用了同一个数据库连接，笔者监听了 Celery worker 进程初始化的信号，并在 worker 进程初始化完成后查看当前进程的相关信息



```python
import logging
    import threading
    import os

from celery import signals
from django.db import connections

@signals.worker_process_init.connect
def worker_init(**kwargs):
    import threading
    import os
    for conn in connections.all(): # 这里会读取 settings 中所有数据库配置对应的连接
        logger.error(
            '\n <pid>: %s\n '
            '---------------------------------- \n '
            '<connection>: %s\n '
            '<wrapper>: %s\n '
            '---------------------------------- \n '
            '<thread>: %s\n '
			'<local>: %s\n '
            '<handler>: %s\n '
            '<parent pid>: %s \n '
            '<allow_thread_sharing>: %s\n' % (
                os.getpid(), # 获取当前进程 ID
                conn.connection, # 获取 DatabaseWrapper 中存储的真正的数据库连接对象
                conn, # 当前取得的 DatabaseWrapper
                threading.currentThread(), 
				threading.local(),
                connections, # 当前取得的 ConnectionHandler 对象
                os.getppid(), # 获取父进程的 ID
                conn.allow_thread_sharing)) # 查看当前 DatabaseWrapper 是否是线程安全的
```

`ConnectionHandler` 的 `all()` 方法实现如下：

```python
# django/db/utils.py

DEFAULT_DB_ALIAS = 'default'


class ConnectionHandler(object):
    @cached_property
    def databases(self):
        if self._databases is None:
            self._databases = settings.DATABASES
        if self._databases == {}:
            self._databases = {
                DEFAULT_DB_ALIAS: {
                    'ENGINE': 'django.db.backends.dummy',
                },
            }
        if DEFAULT_DB_ALIAS not in self._databases:
            raise ImproperlyConfigured("You must define a '%s' database" % DEFAULT_DB_ALIAS)
        return self._databases
    
    def __iter__(self):
        return iter(self.databases)

    def all(self):
        return [self[alias] for alias in self]
```



输出如下：


```
[2018-07-20 09:50:05,601: ERROR/Worker-3]
 <pid>: 41293
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x107b6e350>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10739d250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x107df54d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x1061edd50>
 <parent pid>: 41281
 <allow_thread_sharing>: False

[2018-07-20 09:50:05,601: ERROR/Worker-2]
 <pid>: 41292
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x107b6e350>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10739d250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x107df44d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x1061edd50>
 <parent pid>: 41281
 <allow_thread_sharing>: False

[2018-07-20 09:50:05,604: ERROR/Worker-1]
 <pid>: 41291
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x107b6e350>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10739d250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x107df44d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x1061edd50>
 <parent pid>: 41281
 <allow_thread_sharing>: False

[2018-07-20 09:50:05,604: ERROR/Worker-4]
 <pid>: 41294
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x107b6e350>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10739d250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x107df54d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x1061edd50>
 <parent pid>: 41281
 <allow_thread_sharing>: False
```

可以看到，4个 worker 进程 `41291`, `41292`, `41293`, `41294` 被启动了，随后通过 `lsof` 查看这些进程打开的 mysql 连接：

```
$ lsof -p 41291,41292,41293,41294 | grep :mysql
python2.7 41291 lianghonghao   10u   IPv6 0xb37e546feaec2979        0t0     TCP localhost:62502->localhost:mysql (ESTABLISHED)
python2.7 41292 lianghonghao   10u   IPv6 0xb37e546feaec2979        0t0     TCP localhost:62502->localhost:mysql (ESTABLISHED)
python2.7 41293 lianghonghao   10u   IPv6 0xb37e546feaec2979        0t0     TCP localhost:62502->localhost:mysql (ESTABLISHED)
python2.7 41294 lianghonghao   10u   IPv6 0xb37e546feaec2979        0t0     TCP localhost:62502->localhost:mysql (ESTABLISHED)
```

果然，四个进程同时使用了 `localhost:62502->localhost:mysql` 这条 TCP 连接。

如果我们在 Worker 进程初始化之后手动关闭 `DatabseWrapper` 中的连接，然后重新建立呢？修改代码：

```python
@signals.worker_process_init.connect
def worker_init(**kwargs):
    import threading
    import os
    for conn in connections.all():
        conn.close() # 关闭连接
        conn.connect() # 重新建立
        logger.error(
            '\n <pid>: %s\n '
            '---------------------------------- \n '
            '<connection>: %s\n '
            '<wrapper>: %s\n '
            '---------------------------------- \n '
            '<thread>: %s\n '
            '<local>: %s\n '
            '<handler>: %s\n '
            '<parent pid>: %s \n '
            '<allow_thread_sharing>: %s\n' % (
                os.getpid(),  # 获取当前进程 ID
                conn.connection,  # 获取 DatabaseWrapper 中存储的真正的数据库连接对象
                conn,  # 当前取得的 DatabaseWrapper
                threading.currentThread(),
                threading.local(),
                connections,  # 当前取得的 ConnectionHandler 对象
                os.getppid(),  # 获取父进程的 ID
                conn.allow_thread_sharing))  # 查看当前 DatabaseWrapper 是否是线程安全的
```

运行程序，输出如下：

```
[2018-07-20 09:55:49,137: ERROR/Worker-2]
 <pid>: 42715
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x10fb87b50>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10f33a250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x10fd904d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x10e18ad50>
 <parent pid>: 42705
 <allow_thread_sharing>: False

[2018-07-20 09:55:49,138: ERROR/Worker-3]
 <pid>: 42716
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x10fb87b50>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10f33a250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x10fd914d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x10e18ad50>
 <parent pid>: 42705
 <allow_thread_sharing>: False

[2018-07-20 09:55:49,140: ERROR/Worker-1]
 <pid>: 42714
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x10fb87b50>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10f33a250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x10fd904d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x10e18ad50>
 <parent pid>: 42705
 <allow_thread_sharing>: False

[2018-07-20 09:55:49,141: ERROR/Worker-4]
 <pid>: 42717
 ----------------------------------
 <connection>: <pymysql.connections.Connection object at 0x10fb87b50>
 <wrapper>: <django.db.backends.mysql.base.DatabaseWrapper object at 0x10f33a250>
 ----------------------------------
 <thread>: <_MainThread(MainThread, started 140735510033280)>
 <local>: <thread._local object at 0x10fd914d0>
 <handler>: <django.db.utils.ConnectionHandler object at 0x10e18ad50>
 <parent pid>: 42705
 <allow_thread_sharing>: False

```

查看 worker 进程打开的连接：

```
$ lsof -p 42714,42715,42716,42717 | grep :mysql
python2.7 42714 lianghonghao    3u   IPv6 0xb37e546fd9bde279        0t0     TCP localhost:62813->localhost:mysql (ESTABLISHED)
python2.7 42715 lianghonghao    3u   IPv6 0xb37e546feaec34f9        0t0     TCP localhost:62811->localhost:mysql (ESTABLISHED)
python2.7 42716 lianghonghao    3u   IPv6 0xb37e546feaec4079        0t0     TCP localhost:62812->localhost:mysql (ESTABLISHED)
python2.7 42717 lianghonghao    3u   IPv6 0xb37e546fd9bdff39        0t0     TCP localhost:62814->localhost:mysql (ESTABLISHED)
```

可以看到，在子进程初始化后断开并重新建立之后，就不会出现多个 worker 同时占用一个连接的情况了。

那么，到底是什么原因导致了子进程拿到了和父进程一样的连接呢？



### 3.4 致命组合



首先，普及一个知识，Django 中是没有数据库连接池的，这就意味着每次请求都需要经历一次连接建立和连接断开的过程。



在 Django 1.6 版本之后，加入了一个新的功能：[persistent connections](https://docs.djangoproject.com/en/1.8/ref/databases/)，开发者能够通过在数据库配置中设置 `CONN_MAX_AGE` 变量的值来指定一个数据库从建立到销毁前能够被保留的时间。至于 Django 为什么不采用数据库连接池而是使用连接持久化的方案，可以参考一下这个帖子中的讨论：[Database pooling vs. persistent connections](https://groups.google.com/forum/#!topic/django-developers/NwY9CHM4xpU)。



Django 建立每个连接时都会将该连接到期的时间设置到 `DatabaseWrapper` 的 `close_at` 属性中：

```python
# django/db/backends/base/base.py

class BaseDatabaseWrapper(object):

    def connect(self):
        # ...

        max_age = self.settings_dict['CONN_MAX_AGE']
        self.close_at = None if max_age is None else time.time() + max_age # 上文提到的连接到期时间

        # ...
```



同时，在每次请求开始前和结束后，检测当前内存中数据库连接的可用性，并关闭过期和不可用的连接：



> Persistent connections avoid the overhead of re-establishing a connection to the database in each request. They’re controlled by the [`CONN_MAX_AGE`](https://docs.djangoproject.com/en/1.8/ref/settings/#std:setting-CONN_MAX_AGE) parameter which defines the maximum lifetime of a connection. It can be set independently for each database.



实现如下：



```python
# django/db/__init__.py
from django.core import signals

def close_old_connections(**kwargs):
    for conn in connections.all():
        conn.close_if_unusable_or_obsolete()

# 监听请求开始和结束的事件，并在触发时检测连接的有效性
signals.request_started.connect(close_old_connections)
signals.request_finished.connect(close_old_connections)
```



```python
# django/db/backends/base/base.py

class BaseDatabaseWrapper(object):
	
    # ...
    
    def close_if_unusable_or_obsolete(self):
        if self.connection is not None:
            if self.get_autocommit() != self.settings_dict['AUTOCOMMIT']:
                self.close()
                return
			
            # 如果连接在上次使用过程中遇到了无法恢复的错误也会被关闭
            # If an exception other than DataError or IntegrityError occurred
            # since the last commit / rollback, check if the connection works.
            if self.errors_occurred:
                if self.is_usable():
                    self.errors_occurred = False
                else:
                    self.close()
                    return
            
            # 重点在这里
            # 如果连接没过期就不会被关闭
            # persistent connections 的使用影响了这段逻辑
            if self.close_at is not None and time.time() >= self.close_at:
                self.close()
                return
    
    def close(self):
        self.validate_thread_sharing()
        if self.closed_in_transaction or self.connection is None:
            return
        
        # 关闭数据库连接并置空
        try:
            self._close()
        finally:
            if self.in_atomic_block:
                self.closed_in_transaction = True
                self.needs_rollback = True
            else:
                self.connection = None 

    def _close(self):
        if self.connection is not None:
            with self.wrap_database_errors:
                return self.connection.close() # 真正的数据库连接在这里关闭
```

























其实这个功能的出发点是好的，但是和 multiprocessing 结合使用，就有可能导致致命的错误，通过查阅资料和阅读 Celery 和 djcelery 的源码后，笔者发现两者都在某些时刻尝试去清理子进程从父进程处获得的数据库连接以保证正常运行，如果没有使用 persistent connections 功能，就能够在子进程**初始化完成后**和**任务开始执行前**成功关闭从父进程处继承来的连接，当子进程需要进行数据库操作时，就会重新建立新的连接，也就不会出现多个进程同时使用一条连接的情况。以下为 `celery` 中的部分源码：



```python
# celery.fixups.django.py

# ...

class DjangoWorkerFixup(object):

    def __init__(self, app):
        
        # ...
        
        try:
            self._close_old_connections = symbol_by_name(
                'django.db:close_old_connections', # 重点
            )
        except (ImportError, AttributeError):
            self._close_old_connections = None
	
    def install(self):
		
        # ...
        signals.task_prerun.connect(self.on_task_prerun) # 在任务开始前执行
        signals.task_postrun.connect(self.on_task_postrun) # 在任务结束后执行
        # ...
        
        self.close_database()
        self.close_cache()
        return self
    
    def on_task_prerun(self, sender, **kwargs):
        """Called before every task."""
        if not getattr(sender.request, 'is_eager', False):
            self.close_database()

    def close_database(self, **kwargs):
        if self._close_old_connections:
            return self._close_old_connections()  # Django 1.6
        
        # ...
```



以下为 `djcelery` 的部分源码：



```python
# djcelery.loaders.py

class DjangoLoader(BaseLoader):
    
    # ...
    
    def on_process_cleanup(self):
        """Does everything necessary for Django to work in a long-living,
        multiprocessing environment.

        """
        # See http://groups.google.com/group/django-users/
        #            browse_thread/thread/78200863d0c07c6d/
        self.close_database()
        self.close_cache()
    
    # 在每个任务开始前执行
    def on_task_init(self, task_id, task):
        """Called before every task."""
        try:
            is_eager = task.request.is_eager
        except AttributeError:
            is_eager = False
        if not is_eager:
            self.close_database()

    def _close_database(self):
        try:
            funs = [conn.close for conn in db.connections]
        except AttributeError:
            if hasattr(db, 'close_old_connections'):  # Django 1.6+
                funs = [db.close_old_connections] # 在这里获取到 django.db:close_old_connections
            else:
                funs = [db.close_connection]  # pre multidb

        for close in funs:
            try:
                close()
            except DATABASE_ERRORS as exc:
                str_exc = str(exc)
                if 'closed' not in str_exc and 'not connected' not in str_exc:
                    raise

    def close_database(self, **kwargs):
        db_reuse_max = self.conf.get('CELERY_DB_REUSE_MAX', None)
        if not db_reuse_max:
            return self._close_database()
        if self._db_reuse >= db_reuse_max * 2:
            self._db_reuse = 0
            self._close_database()
        self._db_reuse += 1
```



而在笔者的数据库配置中，已经设置了 `CONN_MAX_AGE` 为 3600（一个连接从建立成功后有 3600s 的有效期），而 `djcelery` 和 `celery` 清理都是通过调用 `django.db.close_old_connections()` 来清理数据库连接的，正是因为该方法在检查没有出现异常的连接时，若该连接没有过期则不会进行关闭，才导致了 `djcelery` 和 `celery` 的准备工作没有起作用，关键代码如下：



```python
# django/db/backends/base/base.py

if self.close_at is not None and time.time() >= self.close_at:
    self.close()
    return
```



所以一个完美的 BUG 就这么出现了：



1. 父进程在进行子进程初始化工作之前调用了 Django ORM 进行数据库操作，导致父进程中的 `DatabaseWrapper` 中的连接被初始化。
2. 子进程被 fork 出来后，由于完全复制了父进程的内存数据，导致所有 worker 共享了同一个 MySQL 连接（同一个 socket file）。
3. djcelery 和 Celery 虽然在子进程初始化和任务开始时对子进程中的连接进行了清理，但是由于 persistent connections 功能的存在，导致数据库连接没有被关闭。
4. 当多个子进程同时使用时，极有可能原地爆炸。



### 3.5 结论及解决方案



所以，Django 的 persistent connections 在与 `multiprocessing` 同时使用时，要多加注意，一定要在把父进程中的数据库连接关闭之后再创建子进程。



#### 3.5.1 关闭 persistent connections 功能



关闭 persistent connections 功能，这样在子进程初始化完成后和任务开始前会把从父进程中继承过来的连接关闭。



#### 3.5.2 手动关闭 Django ORM 中维护的连接



不关闭 persistent connections 功能，监听子进程初始化完成和任务开始的信号，并在收到信号时手动强制关闭当前进程中 Django ORM 的连接，实现如下：



```python
from django.db import connections

@signals.task_prerun.connect
def task_prerun(**kwargs):
    for conn in connections.all():
        conn.close() # 这里会调用 DatabaseWrapper 的 close() 方法

@signals.worker_process_init.connect
def worker_init(**kwargs):
    for conn in connections.all():
        conn.close()
```



#### 3.5.4 使用其他的并发实现方式



目前，只有在多进程模式下使用 Django ORM 会出现上述问题，Django 通过 ThreadLocal 来管理不同线程所对应的数据库连接，如果不使用多进程模式就不会出现内存数据相同的问题，也就不会出现共享文件描述符的问题，而 Celery 提供了若干种并发的模式：eventlet,
gevent, solo, threads，在能够满足功能性需求和非功能性需求的前提下，不妨考虑其他的并发模式。



## 4. 后记



这个 BUG 是两种框架的特性冲突而导致，但是 Django 官方文档在介绍 persistent connections 功能时并没有提醒开发者注意该事项，而出现错误后又很难定位问题，感觉在一定程度上对开发者有些不友好。

希望本次总结能够给各位读者打预防针，使得读者在日后的使用中尽量避免遇到这样的问题，或对读者在解决其他问题时提供一定的参考。