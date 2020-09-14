
## 1. Nginx架构

### 1.1. Nginx处理流程

Nginx使用Epoll的非租塞事件驱动模型，内部有三种状态机：

1. 传输层状态机，接收四层请求
2. HTTP状态机，处理七层事件
3. Mail状态机，处理邮件

基于状态机，在nginx接收请求处理静态资源的时候，在静态目录找到静态资源，如果做反向代理的时候，从磁盘中缓存取，当nginx内存不足以缓存文件和信息的时候，sendfile活AIO会退化成阻塞的IO调用，因此需要一个线程池来处理调用。对于处理完成的请求会记录access和error日志。当进行反向代理的时候，可以通过协议或应用层代理到后端服务器。

### 1.2. Nginx进程结构

Nginx采用多进程而不是多线程模型的原因是进程之间是独立内存空间的，而多线程是共享内存地址空间的，如果某个线程出现内存问题，会导致整个nginx进程挂掉，而多进程则不会出现这样的问题。

Nginx进程包含一个Master的父进程和多个子进程，子进程包含worker进程和cache进程。master进程的目的是管理worker进程，所有的worker进程是处理真正的请求，master进程监控worker进程的运行，是不是需要载入新的配置文件、是不是要进行热部署。

缓存是不仅要在多个worker进程之间共享，而且也要被cache manager和cache loader进程使用，cache manager和cache load也是为反向代理时，后端发来的动态请求做缓存使用的。cache loader做缓存的载入，manager做缓存的管理。

进程之间的通信是使用共享内存来解决的。worker进程很多的原因是nginx采用事件驱动模型以后，希望nginx每个nginx进程都占用一颗CPU，所以不仅nginx一般配置进程数等于CPU核心数，而且最好为每个进程分配固定的CPU核更有效的利用CPU上的缓存来减少缓存失效降低命中率。

Tip：nginx reload和kill -SIGHUP pid是一样的

### 1.3. 使用信号管理进程

1. Master进程：

   - 监控worker进程CHLD信号，如果worker进程终止，会发送CHLD信号给master，如果worker进程意外终止，master可以立刻通过CHLD发现worker进程异常，会立刻拉起worker进程
   - 管理worker进程
   - 接收信号：
     - TERM，INT立刻停止进程
     - 、QUIT优雅停止，保证不会对用户发reset报文
     - HUP重载配置
     - USR1重新打开日志文件，对日志文件切割
     - USR2平滑升级
     - WINCH优雅关闭旧的进程(配合USR2来进行升级)

2. Worker进程

   接收信号：TERM、INT、QUIT、USR1、WINCH，虽然worker也可以接收信号，但是通常不这么操作，都是有master来管理的

3. Nginx命令行

   - reload：HUP
   - reopen：USER1
   - stop：TERM
   - quit：QUIT

nginx接收命令和使用kill+信号处理的效果是一样的

### 1.4. reload的过程

在配置文件变更后，执行nginx -s reload会将新的配置文件应用，reload的流程：

1. 想master进程发送HUP信号（reload指令）
2. master进程校验配置语法是否正确（不一定非要执行-t指令校验）
3. master进程打开可能引入的新的监听端口
4. master进程用新的配置文件启动新的worker子进程
5. master进程向老的worker子进程发送QUIT信号优雅关闭。为了保证nginx的平滑，关闭老进程之前一定要保证新进程启动
6. 老worker进程收到QUIT信号后，关闭监听句柄，处理完当前连接后结束进程。正常情况下老的worker进程都会正常退出。但是在一些特殊情况，比如连接出了问题，客户端长时间没有处理，导致老worker进程一直存在，新版的nginx通过参数**worker_shutdown_timeout**来释放异常超时的worker进程连接

### 1.5. 热升级以及回滚

1. 将旧的Nginx二进制文件替换成新的二进制文件（注意备份；替换时，由于文件被进程使用，所以用cp -f参数强制替换）
2. 想master进程发送USER2信号
3. master进程修改pid文件名，加后缀.oldbin
4. 用新Nginx文件启动新的master进程
5. 想老的master进程发送WINCH信号（从第三部的备份pid文件找到进程号，或者ps查看），关闭老的worker进程
6. 回滚：向老master发送HUP信号，向新master发送QUIT

### 1.6. 优雅关闭worker进程

 如果直接关闭worker进程，会导致正在连接的用户收到异常信息，优雅的关闭是指worker进程可以识别出当前的连接没有处理请求，然后把进程关闭。优雅关闭只针对HTTP请求

1. 设置定时器work_shutdown_timeout
2. 关闭监听句柄
3. 关闭空闲连接
4. 在循环中等待全部链接关闭，这个等待时间可能会比较长，可能会超过work_shutdown_timeout设置时间，当达到work_shutdown_timeout时间，就会被强制关闭
5. 退出进程

### 1.7. Nginx事件驱动模型Epoll

在建立了百万并发的连接时，同时活跃的连接可能只有几百个，只需要处理着几百个请求，而select或poll每次取操作系统事件时，会把这百万连接都交给操作系统，由操作系统判断哪些连接有请求进来。Epoll就是用了前者。

Epoll的实现：Epoll维护了eventpoll，它通过两个数据结构将两个事件分开，nginx每次取活跃连接的时候，会遍历一个链表，这个链表里只有活跃的连接，从内核态读取到用户态，这样效率就很高。

通常可以将worker进程的优先级调高（比如-19），这样CPU会给worker进程分配的时间片比较大，避免进程切换，一个县城仅处理一个连接，不做连接切换，依赖OS的进程调度实现并发。一线程同时处理多连接，在用户态代码完成连接切换，尽量减少OS进程切换。

### 1.8. 同步&异步、阻塞&非租塞的区别

以Accept为例：

#### 阻塞调用

阻塞Accept请求，调用Accept，正常Accept不为空会立即返回；但是如果Accept队列为空时，操作系统会等待新的三次握手的连接到达内核中去唤醒Accept调用。这个过程会导致进程的切换，对于nginx是不能容忍进程切换的。

#### 非租塞调用

调用Accept时，监听套接字设置为非租塞，如果Accept队列为空，则不等待，立刻返回EAGAIN错误，代码会进行处理。所以，这里由代码决定是否切换新任务。

### 1.9. Nginx的模块

- 前提编译进nginx
- 提供哪些配置项
- 模块何时被使用（编译即使用还是显示配置）
- 提供哪些变量

 ### 1.10. Nginx如何通过连接池处理网络请求

syntax: worker_connections |default 512 | context events

配置一个worker进程最大同时建立连接数，该连接数用于客户端与nginx、nginx与上游服务器的连接，因此，一个客户端的请求需要消耗两个连接。每个连接对应一个读事件和写事件。

内存池配置：

Syntax: connection_pool_size

### 1.11. worker进程间的协同工作

Nginx进程间的通信方式

1. 信号
2. 共享内存，多个进程可以同时使用，避免竞争问题需要加锁 



