## 问题现象
Fastapi使用Gunicorn+Uvicorn部署生产环境，使用Logrotate压缩access日志后，新的Access日志无法正常输出。

> [Uvicorn issue]

### 1 启动gunicorn+uvicorn
   
```shell
(venv) [root@localhost task_registry_center]# gunicorn -w 2 -b 0.0.0.0:8000 -k uvicorn.workers.UvicornWorker --access-logfile log/access.log main:app
[2020-12-16 17:09:41 +0800] [3330] [INFO] Starting gunicorn 20.0.4
[2020-12-16 17:09:41 +0800] [3330] [INFO] Listening at: http://0.0.0.0:8000 (3330)
[2020-12-16 17:09:41 +0800] [3330] [INFO] Using worker: uvicorn.workers.UvicornWorker
[2020-12-16 17:09:41 +0800] [3332] [INFO] Booting worker with pid: 3332
[2020-12-16 17:09:41 +0800] [3333] [INFO] Booting worker with pid: 3333
```

### 2 清空access.log文件
   
```shell
now access log file is empty
[root@localhost log]# pwd
/data/task_registry_center/log
[root@localhost log]# ll
total 20
-rw-r--r--. 1 root root    0 Dec 16 16:56 access.log
-rw-r--r--. 1 root root    0 Dec 16 16:41 client.log
-rw-r--r--. 1 root root    0 Dec 16 16:56 debug.log
-rw-r--r--. 1 root root    0 Dec 16 16:41 elk_client.log
-rw-r--r--. 1 root root 5402 Dec 16 17:08 error.log
-rw-r--r--. 1 root root    0 Dec 16 16:41 id_server.log
-rw-r--r--. 1 root root 4029 Dec 16 17:09 main.log
-rw-r--r--. 1 root root 4480 Dec 16 17:10 tasks.log
```

### 3 调用10次Api，此时access.log内容如下：
   
```shell
192.168.11.1:60477 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60478 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60479 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60480 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60481 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60482 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60483 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60484 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60485 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60486 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
```

### 4 logrotate 配置如下：
   
```shell
[root@localhost ~]# cat /etc/logrotate.d/task_registry_center                                                                                                                                                   
/data/task_registry_center/log/*.log {                                                                                                                                  
        size 1M                                                                                                                                                         
        rotate 20                                                                                                                                                       
        compress                                                                                                                                                        
        nodelaycompress                                                                                                                                                 
        dateformat -%Y-%m-%d-%s                                                                                                                                         
        dateext                                                                                                                                                         
        missingok                                                                                                                                                       
        notifempty                                                                                                                                                      
        postrotate                                                                                                                                                      
            killall -s USR1 gunicorn                                                                                                                  13,9          All
        endscript
}
```
### 5 手动调用logrotate命令进行日志压缩或者直接像gunicorn发送USR1信号:

    手动调用logrotate命令： `[root@localhost ~]# /usr/sbin/logrotate -f /etc/logrotate.d/task_registry_center`
   
    发送USR1信号： `killall -s USR1 gunicorn`

### 6 Gunicorn日志提示已经正常处理了USR1信号（在gunicorn中，USR1信号表示日志文件发生变更，需要重新打开日志文件）：
    
```shell
[2020-12-16 17:12:16 +0800] [3330] [INFO] Handling signal: usr1
[2020-12-16 17:12:16 +0800] [3330] [INFO] Handling signal: usr1
[2020-12-16 17:12:16 +0800] [3330] [INFO] Handling signal: usr1
[2020-12-16 17:12:16 +0800] [3330] [INFO] Handling signal: usr1
```
   
### 7 再次调用10次Api，access.log并未记录到最新的调用记录。
   
- 日志目录，可见新的access.log日志为空：

```shell
[root@localhost log]# ll
total 20
-rw-r--r--. 1 root root   0 Dec 16 17:12 access.log
-rw-r--r--. 1 root root 125 Dec 16 17:11 access.log-2020-12-16-1608109936.gz
```
- `access.log-2020-12-16-1608109936.gz`文件内容依旧为前10次的API调用记录:
```shell
192.168.11.1:60477 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60478 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60479 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60480 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60481 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60482 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60483 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60484 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60485 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
192.168.11.1:60486 - "GET /api/client?page_size=10&page=1 HTTP/1.1" 200
```

## 环境信息
- 系统 Centos7
- Python 3.8.3
- gunicorn 20.0.4
- uvicorn 0.13.1
- fastapi 0.62.0

## logrotate
logrotate有两个工作模式，分别为`copytruncate`和`create`，简单说下两种模式的区别。

在介绍create模式之前，需要先介绍下linux系统中文件是通过inode编号来定位的，和文件名称没有关系。
### create
1. 将当前程序输出的日志文件重命名为压缩文件，但是inode编号不变。程序仅通过inode编号定位日志文件，所以此时到程序重新打开日志文件，所有生成的日志都输出到了重命名后的文件中。
2. 创建新的同名日志文件，虽然新的日志文件和原来日志文件的名字一样，但是inode编号不一样，所以程序输出的日志还是往原日志文件输出。
3. 通过某些方式通知程序，重新打开日志文件。程序重新打开日志文件，靠的是文件路径而不是inode编号，所以打开的是新的日志文件。
    
### copytruncate
1. 拷贝当前日志文件到压缩文件，这期间程序照常输出日志到原来的文件中，原来的文件名也没有变。
2. 清空程序正在输出的日志文件。清空后程序输出的日志还是输出到这个日志文件中，因为清空文件只是把文件的内容删除了，文件的inode编号并没有发生变化，变化的是元信息中文件内容的信息。

此方案存在一个问题，就是在日志拷贝开始，到清空日志文件的这段时间内，程序输出的日志没有存储，存在日志丢失的风险。

> [logrotate机制和原理]

## 问题修复

### Gunicorn源码分析
在gunicorn(版本20.0.4)中可以查找到和USR1信号处理的相关代码如下：

`venv/Lib/site-packages/gunicorn/arbiter.py` 196行：

```python
def run(self):
    "Main master loop."
    self.start()
    util._setproctitle("master [%s]" % self.proc_name)

    try:
        self.manage_workers()

        while True:
            self.maybe_promote_master()

            sig = self.SIG_QUEUE.pop(0) if self.SIG_QUEUE else None
            if sig is None:
                self.sleep()
                self.murder_workers()
                self.manage_workers()
                continue

            if sig not in self.SIG_NAMES:
                self.log.info("Ignoring unknown signal: %s", sig)
                continue

            signame = self.SIG_NAMES.get(sig)
            handler = getattr(self, "handle_%s" % signame, None)
            if not handler:
                self.log.error("Unhandled signal: %s", signame)
                continue
            self.log.info("Handling signal: %s", signame)
            handler()
            self.wakeup()

def handle_usr1(self):
    """
    SIGUSR1 handling.
    Kill all workers by sending them a SIGUSR1
    """
    self.log.reopen_files()
    self.kill_workers(signal.SIGUSR1)
```

### Uvicorn源码分析
Uvicorn只处理了程序终止相关的信号：
```python
HANDLED_SIGNALS = (
    signal.SIGINT,  # Unix signal 2. Sent by Ctrl+C.
    signal.SIGTERM,  # Unix signal 15. Sent by `kill <pid>`.
)
```

`venv/Lib/site-packages/uvicorn/workers.py` 64行：
```python
    def init_process(self):
        self.config.setup_event_loop()
        super(UvicornWorker, self).init_process()

    def init_signals(self):
        pass

    def run(self):
        self.config.app = self.wsgi
        server = Server(config=self.config)
        loop = asyncio.get_event_loop()
        loop.run_until_complete(server.serve(sockets=self.sockets))
```
可以看到UvicornWorker的init_signals函数被重写为不执行任何操作了，其父类gunicorn.workers.base.Worker的init_signals方法为：
```python
    def init_signals(self):
        # reset signaling
        for s in self.SIGNALS:
            signal.signal(s, signal.SIG_DFL)
        # init new signaling
        signal.signal(signal.SIGQUIT, self.handle_quit)
        signal.signal(signal.SIGTERM, self.handle_exit)
        signal.signal(signal.SIGINT, self.handle_quit)
        signal.signal(signal.SIGWINCH, self.handle_winch)
        signal.signal(signal.SIGUSR1, self.handle_usr1)
        signal.signal(signal.SIGABRT, self.handle_abort)

        # Don't let SIGTERM and SIGUSR1 disturb active requests
        # by interrupting system calls
        signal.siginterrupt(signal.SIGTERM, False)
        signal.siginterrupt(signal.SIGUSR1, False)

        if hasattr(signal, 'set_wakeup_fd'):
            signal.set_wakeup_fd(self.PIPE[1])
```

## 解决方案
让UvicornWorker正常处理USR1信号即可，将`venv/Lib/site-packages/uvicorn/workers.py` 63行由：
```python
def init_signals(self):
    pass
```
修改为
```python
import signal

def init_signals(self):
    signal.signal(signal.SIGUSR1, self.handle_usr1)
```
即可。

[Uvicorn issue]: https://github.com/encode/uvicorn/issues/896
[logrotate机制和原理]: https://www.lightxue.com/how-logrotate-works