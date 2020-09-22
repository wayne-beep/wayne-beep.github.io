
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

## 2. HTTP模块

### 2.1. 冲突的配置指令优先级

#### 配置快的嵌套

```
main
http{
  upstream{}
  split_client{}
  map{}
  geo{}
  server{
    if(){}
    location{
      limit_except{}
    }
    location {
      location {     
      }
    }
  }
  server{}
}
```

#### 指令的Context

```yaml
Syntax: log_format name;
Default: log_format combined "...";
Context: http

Syntax: access_log
Default: access_log logs/access.log combined;
Context: http,server,location,if in location,limit_except
```

log_format只能存在于http块，而access_log可以存在于很多模块内

#### 指令的合并

- 直指令：存储配置项的值

  不同块下可以合并，存储值指令继承规则向上覆盖，子配置不存在，直接使用父配置；子配置存在则覆盖父配置。例如root；access_log；gzip。

- 动作类指令：指定行为

  不同块下不可以合并，例如：rewrite，proxy_pass，当指令执行到这个位置时，必须立刻执行这种行为。生效阶段：server_rewrite阶段；rewrite阶段；content阶段

### 2.2. listen指令的用法

- listen unix:/var/run/nginx.sock;
- listen 127.0.0.1:8080;
- listen 127.0.0.1;
- isten 8080;
- listen *:8080;
- listen localhost:8080 bind;
- listen [::]8080 ipv6only=on;
- listen[::1];

### 2.3. 正则表达式

| 代码  | 说明                         |
| :---: | ---------------------------- |
|   .   | 匹配除换行符以外的任意字符   |
|  \w   | 匹配字母或数字或下划线或汉字 |
|  \s   | 匹配任意空白字符             |
|  \d   | 匹配数字                     |
|  \b   | 匹配单词的开始或结束         |
|   ^   | 匹配字符串的开始             |
|   $   | 匹配字符串结束               |
|   *   | 重复0或更多次                |
|   +   | 重复1或多次                  |
|  ？   | 重复0或1次                   |
|  {n}  | 重复n次                      |
| {n,}  | 重复n或多次                  |
| {n,m} | 重复n到m次                   |
|   \   | 转译                         |
| （）  | 分组                         |

### 2.4. server指令块查找

#### server_name指令

-  *泛域名：仅支持最前或最后，例如：server_name *.example.com

- 正则表达式：server_name www.example.com ~^www\d+\\.example.com，使用正则表达式，前面需要加上~^
- .example.com 可以匹配 example.com  *.example.com
- server_name 后面写 "_" 表示匹配所有域名

指令后可以跟多个域名，第一个是主域名

```
Syntax server_name_in_redirect on|off
Default server_name_in_redirect off
Context http,server,location
```

在多域名的时候，当使用return指令开启重定向时，默认的请求返回的重定向地址就是请求的地址，当设置server_name_in_redirect为on时，请求任一域名都会重定向到主域名

用正则表达式创建变量：用小()

```
server {
	server_name ~^(www\.)?(.+)$;
	location / {
			root /sites/$2; # $2就是取的domain中的(.+)
	}
}
```

```
server {
		server_name ~^(www\.)?(?<domain>.+)$;  # 匿名变量?<domain>
		location / {
				root /sites/$domain;
		}
}
```

#### server匹配顺序

1. 精确匹配
2. *在前的域名
3. *在后的域名
4. 按文件中出现的顺序匹配正则表达式域名
5. default server
   - server块下第一个就是default server
   - 可以在listen后加上default指定为default server

### 2.5. HTTP请求的11个阶段

当nginx接收完header时，就会按照这11个阶段来处理请求

| Stage          | Module                         |
| -------------- | ------------------------------ |
| POST_READ      | realip                         |
| SERVER_REWRITE | rewrite                        |
| FIND_CONFIG    |                                |
| REWRITE        | rewrite                        |
| POST_REWRITE   |                                |
| PREACCESS      | limit_conn,limit_req           |
| ACCESS         | Auth_basic,access,auth_request |
| POST_ACCESS    |                                |
| PRECONTENT     | Try_files                      |
| CONTENT        | Index,autoindex,concat         |
| LOG            | Access_log                     |

#### 11个阶段的处理顺序

![](/images/posts/nginx_request_11stages.jpg)

同一个阶段的各个模块也并不一定都会执行到，执行顺序和处理流程。

#### 2.5.1 获取用户真实ip的realip模块

如何拿到用户真实IP

1. TCP连接四元组
2. HTTP头部X-Forward-For用于传递IP
3. HTTP头部X-Real-IP用于传递用户IP
4. 网络中存在许多反向代理

X-Forward-For与X-Real-IP的区别是，X-Forward-For会将用户真实ip地址与经过链路代理的IP地址全部记录下来传递给nginx，而X-Real-IP只能保存一个地址就是用户的真实IP地址。

拿到用户真实IP后基于变量来使用，例如binary_remote_addr,remote_addr这样的变量的值就是真实IP。利用这两个变量来做连接限制才有意义，所以limit_conn模块一定要在PRE_ACCESS阶段而不能在POST_ACCESS阶段。

realip模块默认是不会编译到nginx的，需要通过--with-http_realip_module启用功能。它的功能是修改客户端地址。它修改了原来的remote_addr和remote_port变量，因此如果还想使用原来的变量，需要使用realip_remote_addr和realip_remote_port两个变量。

realip模块的指令：

``` 
set_real_ip_from
real_ip_header X-Real-IP|X-Forward-For|proxy_protocol ,默认是X-Real-IP
real_ip_recursive 环回地址，如果real_ip_header是X-Forward-For ，开启它会取X-Forward-For与客户端相同ip的上一个ip
```

#### 2.5.2 rewrite阶段的rewrite指令和return

rewrite模块执行后，后续的其他模块都无法执行

1. return指令

```
Syntax: return code [text] | code URL | URL
Context server,location,if
```

返回状态码：

- nginx自定义 444：关闭连接
- http1.0标准：
  - 301：http1.0永久重定向
  - 302：临时重定向，禁止被缓存
- http1.1标准：
  - 303：临时重定向，允许改变方法，禁止被缓存
  - 307：临时重定向，不允许改变方法，禁止被缓存
  - 308：永久重定向，不允许改变方法

###### return与error_page

```
Syntax: error_page code ...[=[response]] uri;
Default: -
Context: http,server,location,if in location
```

error_page 收到返回码的时候可以重定向到其他返回码，也可以返回个页面

```
error_page 404 /404.html
error_page 404 =200 /empty.gif;
location / {
		error_page 404 =@fallback
}

location @fallback {
		proxy_pass http://backend;
}
error_page 403 http://example.com/forbidden.html
error_page 404 =301 http://example.com/notfounnd.html
```

与return的区别

例子：

```
server {
		server_name return.example.com;
		listen 80;
		
		root html/;
		error_page 404 403.html;
		return 403;
		location / {
				return 404 "not found";
		}
}
```

问题：

- server与location下的return的关系？

  当同时打开server下的return和locatiion下的return，会直接由高级别的server段的return返回，因为return是rewrite阶段的指令，又返回会直接跳出。

- return与error_page指令的关系？

  当同时打开location下的return和error_page，error_page不会执行，虽然error_page比return级别高，但是由于return处于最开始的rewrite阶段，所以还是优先执行return而直接返回。

#### 2.5.3 rewrite重写url

```
Syntax: rewrite regex replicement [flag]
Default: -
Context: server,location,if
```

将regex指定的url替换成replacement的新url，可以使用正则表达式及变量提取。当replacement以http[s]://或$schema开头，则直接返回302重定向。替换后的url根据flag指定的方式进行处理：

- --last：用replacement这个URL进行新的location匹配
- --break：break指令停止当前脚本指令的执行，等价于独立的break指令
- --redirect：返回302重定向
- --permanent：返回301重定向

示例：

```
server {
		.....
		root html/
		location /first {
				rewrite /first(.*) /second$1 last;
				return 200 'first';
		}
		location /second {
				rewrite /second(.*) /third$1 break;
				return 200 'second';
		}
		location /third {
				return 200 'third';
		}
		location /redirect1 {
				rewrite /redirect1(.*) $1 permanent;
		}
		location /redirect2 {
				rewrite /redirect2(.*) $1 redirect;
		}
		location /redirect3 {
				rewrite /redirect3(.*) http://rewrite.example.com/$1;
		}
		location /redirect4 {
				rewrite /redirect4(.*) http://rewrite.example.com/$1 permanent;
		}
}
```

问题：

1. return和rewrite指令的顺序关系？
2. 访问/firxt/3.txt、/second/3.txt、/third/3.txt分别返回的是什么？
3. 如果不携带flag会怎么样？

解答：

1. rewrite指令在后面的标志不是break的时候，会和return指令形成一个集合按照顺序执行。当flag是break的时候，直接返回结果。
2. 访问/firxt/3.txt、/second/3.txt都会返回'second'，因为first执行了last继续匹配到/second
3. permanent返回301，redirect返回302，redirect3会返回302，redirect4会返回301.

#### 2.5.4. 条件判断if

```
Syntax: if (condition) {……}
Default：-
Context：server，location
```

if指令条件表达式

- 检查变量为空或者值是否为0
- 将变量与字符串做匹配，使用=或者!=
- 将变量与正则表达式做匹配
  - 大小写敏感：~或者!~
  - 大小写不敏感：~\*或者!~\*
- 检查文件是否存在 -f 或者 !-f
- 检查目录是否存在，使用-d或者!-d
- 检查文件、目录、软连接是否存在使用-e或者!-e
- 检查是否为可执行文件，使用-x或者!-x

示例：

```
if ($http_user_agent ~ MSIE) {
		rewrite ^(.*)$ /msie/$1 break;
}
if ($http_cookie ~* "id=([^;]+)(?:;|$)") {
		sed $id $1;
}
if ($request_method = POST){
		return 405;
}
if ($slow) {
		limit_rate 10k;
}
if ($invalid_referer) {
		return 403;
}
```

#### 2.5.5. find_config找到处理请求的location

location指令
```
Syntax: location [=|~|~*|^~] uri {……}
				location @name {……}
Default: -
Context: server location

# merge合并uri中连续 的"/" ,比如连续写了两个 / ,会合并成一个
Syntax: merge_slashes on | off;  
Default: merge_slashes on;
Context: http，server
```

 location匹配规则：仅匹配URI忽略参数

- = 精确匹配 ^~匹配上后则不再进行正则匹配
- 正则表达式：~ 大小写敏感 ~* 忽略大小写
- 用于内部调整的命名 @

location匹配顺序

![](/images/posts/nginx_location_order.png)

#### 2.5.6. 对连接数限制的limit_conn

在preaccess阶段有limit_req和limit_conn。限制每个客户端的并发连接数，ngx_http_limit_conn_module模块

- 生效阶段：NGX_HTTP_PREACCESS_PHASE
- 默认编译进nginx，通过--without-http_limit_conn_module禁用
- 生小范围
  - 全部worker进程，基于共享内存
  - 进入preaccess阶段前不生效
  - 限制的有效性取决于key的设计，依赖postread阶段的realip模块取到用户真实IP

定义共享内存（包括大小）以及key关键字

```
Syntax: limit_conn_zone key zone=name:size
Default: - 
Context: http
```

 限制并发连接数

 ```
Syntax: limit_conn zone number;
Default: - 
Context: http,server,location
 ```

限制发生时的日志级别

```
Syntax: limit_conn_log_level info|notice|warn|error;;
Default: limit_conn_log_level error;
Context: http,server,location
```

限制发生时向客户端返回的错误码

```
Syntax: limit_conn_status code;
Default: limit_conn_status 503
Context: http, server,location
```

#### 2.5.7. 对连接数限制的limit_req

limit_req将用户的突发请求进行恒定请求速率，ngx_http_limit_req_module模块

- 生效阶段：NGX_HTTP_PREACCESS_PHASE
- 默认编译进nginx，通过--without-http_limit_req_module禁用
- 生效算法：leaky bucket算法
- 生小范围
  - 全部worker进程，基于共享内存
  - 进入preaccess阶段前不生效

定义共享内存包括大小，以及key关键字和限制速率

```
Syntax: limit_req_zone key zone=name:size rate=rate;
Default: - 
Context: http
# rate单位为r/s或者r/m
```

限制并发连接数

```
Syntax: limit_req zone=name [burst=number] [nodelay];
Default: - 
Context: http,server,location
# burst默认为0
# nodelay，对burst中的请求不再采用延时处理的做法，而是立刻处理
```

同时打开limit_conn和limit_req，会都生效，但是返回是limit_req的返回，因为req在conn之前

#### 2.5.8. 对ip做限制的access模块

access阶段控制请求是否可以继续向下访问

```
Syntax: allow address|CIDE|unix:|all
Default: - 
Context: http,server,location,limit_except

Syntax: deny address|CIDE|unix:|all
Default: - 
Context: http,server,location,limit_except
```

当有多个语句设置的时候，会按照顺序向下匹配，当匹配到时就直接返回，不会继续执行了

#### 2.5.9. precontent阶段的try_files模块

模块ngx_http_try_files_module

```
Syntax: try_files file ..... uri|code;
Default: - 
Context: server,location
```

依次视图访问多个url对应的文件（由root或者alias指令指定），当文件存在时直接返回文件内存，如果所有文件都不存在，则按最后一个url或者code返回。

```
server {
		server_name tryfiles.example.com;
		root html/;
		location /first {
				try_files /a/a.html $uri $uri/index.html $uri.html @lasturl; # 依次寻找文件，$uri表示html/first文件
		}
		location @lasturl {
				return 200 "lasturl!\n"
		}
		location /second {
				try_files $uri $uri/index.htl =404;
		}
}
```

 #### 2.5.10. precontent阶段的mirror模块

模块ngx_http_mirror_module默认编译进了nginx，处理请求时，生成自请求访问其他服务为，对子请求的返回值不做任何处理。对于多个环境处理用户流量很有帮助

```
Syntax: mirror uri|off;
Default: mirror off;
Context: http,server,location

Syntax: mirror_request_body on|off;  # 是否将请求体发送到上游服务器
Default:mirror_request_body on;
Context: http,server,location
```

#### 2.5.11. static模块的三个变量

1. request_filename 待访问文件的完整路径
2. document_root 由URI和root/alias规则生成的文件夹路径
3. realpath_root 将document_root中的软连接等换成真实路径

```
location /realpath/ {
		alias html/realpath;
		return 200 "$request_filename:$document_root:$realpath_root"
}

realpath为first目录的软连接
```

返回结果： /usr/local/openresty/nginx/html/realpath/1.txt:/usr/local/openresty/nginx/html/realpath:/usr/local/openresty/nginx/html/first



#### 2.5.12. static模块url后面斜杆问题

static模块实现了root/alias功能时，发现访问目标是目录，但是URL末尾加上/时，会返回301重定向

#### 2.5.13. access日志详细用法

```
Syntax: access_log path [format [buffer=size] [gzip [level] [flush=time] [if=condition] ];
				access_log off;
Default: access_log logs/access.log combined;
Context: http,server,location,if in location,limit_expect;
```

- Path 路径可以包含变量：不打开cache时每记录一条日志都需要打开、关闭日志文件
- if通过变量值控制请求日志是否记录
- 日志缓存：
  - 功能：批量将内存中的日志写入磁盘
  - 写入磁盘的条件
    - 所有待写入磁盘的日志超出缓存大小
    - 达到flush指定过期时间
    - worker进程执行reopen命令或者正在关闭
- 日志压缩
  - 功能：批量压缩内存中日志再写入磁盘
  - buffer大小默认64KB
  - 压缩级别默认1，最高9



当日志文件名包含变量时，每次写入都会打开关闭文件，可以通过open_log_file_cache进行优化

```
Syntax: open_log_file_cache max=N [inactive=time] [min_uses=N] [valid=time]
Default: open_log_file_cache off;
Context: http,server,location
```

- Max: 缓存内的最大文件句柄数，超出后用LRU算法淘汰
- Inactive：文件访问完后在这段时间内不会关闭。默认10s
- min_uses: 在inactive时间内使用次数超过min_uses才会继续在内存中使用。默认1
- valid：超出valid时间后，将对缓存的日志文件检查是否存在。默认60s
- off：关闭缓存功能

#### 2.5.14. HTTP过滤模块

返回响应-加工响应内容。重点关注的4个模块

- copy_filter：复制包体内容
- Postpone_filter：处理子请求
- header_filter：构造响应头部
- write_filter：发送响应

 ##### 替换响应中的字符串：sub模块

功能：将响应中指定的字符串替换成新的字符串

模块：ngx_http_sub_filter_module模块，默认未编译进nginx，通过--with-http_sub_filter_module启用

```
# 指定字符串string和替换的字符串replacement
Syntax: sub_filter string replacemnt;
Default: -
Context: http,server,location

# 决定是否在响应头部显示last_modified
Syntax: sub_filter_last_modified on|off;
Default: sub_filter_last_modified off
Context: http,server,location

# 是否只替换一次
Syntax: sub_filter_once on|off;
Default: sub_filter_once on
Context: http,server,location

# 指定替换的文本类型
Syntax: sub_filter_types mime-type .......;
Default: sub_filter_types text/html;
Context: http,server,location
```

##### 在响应的前后添加内容：addition模块

功能：在响应前或者响应后增加内容，儿增加的方式是铜鼓哦新增子请求的响应完成

模块：ngx_http_addition_filter_module 默认未编译进nginx，通过--with-http_addition_filter_module启用

```
Syntax: add_before_body uri;
Default: -
Context: http,server,location

Syntax: add_after_body uri;
Default: -
Context: http,server,location

Syntax: addition_types minie-types;
Default: addition_types text/html; # * 对所有文件类型产生效果
Context: http,server,location
```



#### 2.5.15. 使用变量进行防盗链

简单有效的防盗链手段：referer模块

> 盗链：某网站通过URL引用了一个页面，当用户在浏览器上点击了url时，http请求的头部会通过referer头部，将该网站当前页面的url带上来告诉这个页面的服务器请求是由某网站发起的。盗链就是使用非正常的网站请求我们的站点资源

防止盗链：通过referer模块，用invalid_referer变量根据配置判断referer头部是否合法。该模块默认编译进了nginx，通过--without-http_referer_module禁用

```
Syntax: valid_referers none|blocked|server_names|string……;
Default: -
Context: server,location

Syntax: referer_hash_bucket_size size;
Default: referer_hash_bucket_size 64;
Context: server,location

Syntax: referer_hash_max_size size;
Default: referer_hash_max_size 2048;
Context: server,location
```

Valid_referers指令可以同时携带多个参数，表示多个referer头部都生效，参数值：

- none：允许确实referer头部的请求访问
- block：允许referer头部没有对应的值的请求访问
- server_names：若referer中站点域名与server_name中本机域名某个匹配，允许改请求访问
- 表示域名及URL的字符串，对域名可在前缀或者后缀中含有*通配符：若referer头部的值匹配字符串后，则允许访问
- 正则表达式：若referer头部的值匹配正则表达式后，则允许访问

invalid_referer变量

- 允许访问时变量值为空
- 不允许访问时变量值为1

例子：

```
location / {
		valid_regerers none blocked server_names *.example.com www.xxxx.com/api ~\.google\.;
		if ($invalid_referer){
				return 403;
		}
		return 200;
}
```

#### 2.5.16. 对客户端使用keepalive提升效率

对客户端keepalive行为控制指令。多个HTTP通过复用TCP连接实现了以下功能：

- 减少握手次数
- 通过减少并发连接数减少了服务器资源消耗
- 降低TCP拥塞控制的影响

Connection头部取值为close或者keepalive，close表示请求处理完即关闭连接，keepalive表示复用连接处理下一条请求

Keep-Alive头部：其值为timeout=n，后面的数字n单位为秒，告诉客户端连接至少保留n秒

```
Syntax: keepalive_disable none|browser; #对某些浏览器不进行keepalive
Default: keepalive_disable msie6;
Context: http,server,location

Syntax: keepalive_request number; # 每个保持会话的连接最大请求次数
Default: keepalive_request 100;
Context: http,server,location

Syntax: keepalive_timeout timeout [header_timeout]; # 设置用户请求完成以后，最多经过timeout时间还是没有新请求进来就关闭连接。header_timeout表示nginx同过Keep-Alive头部向浏览器表示这个连接应该保留多少秒
Default: keepalive_timeout 75;
Context: http,server,location
```



 ## 3. 反向代理与负载均衡

### 3.1. 加权round-roubin负载均衡算法

在加权轮询的方式访问server指令指定的上游服务，集成在nginx的upstream框架中，指令：

- weight：服务访问的权重，默认是1
- max_conns：server的最大并发连接数，仅作用于单worker进程，默认是0，表示没有限制
- max_fails：在fail_timeout时间段内，最大的失败次数。当达到最大失败时，会在fail_timeout秒内这台server不允许再次被选择
- fail_timeout：单位为秒，默认为10s，指定一段时间内，最大失败次数。到达max_fails后，该server不能访问的时间

### 3.2. 对上游服务使用keepalive长连接

功能：通过复用连接，降低nginx与上游服务器建立、关闭连接的消耗，提升吞吐量的同时降低时延

模块：ngx_http_upstream_keepalive_module默认编译进nginx

对上游连接http头部设定：

```
proxy_http_version: 1.1;  # http 1.0是不支持长连接的
proxy_set_header Connection "";
```

```
Syntax: keepalive connections; # 指定nginx与一组上游服务器最多保持多少空闲连接用于keepalive请求
Default: -;
Context: upstream
```

当上游服务使用域名的时候，指定上游服务域名解析的resolver指令（可以防止上游服务器域名解析变更带来的问题）

```
Syntax: resolver address …… [valid=time][ipv6=on|off] # 指定dns服务地址
Default: -;
Context: http,server,location

Syntax: resolver_timeout time;
Default: resolver_timeout 30s;
Context: http,server,location
```

### 3.3.   upstream变量

upstream模块提供的变量（不含cache）

- upstream_addr：上游服务器的IP地址，格式为可读的字符串
- upstream_connect_time：与上游服务器建立连接消耗的时间，单位为秒，精确到毫秒
- upstream_header_time：接收上游服务发挥响应中http头部所消耗的时间，单位为秒，精确到毫秒
- upstream_response_time：接收完整上游服务响应所消耗的时间，单位为秒，精确到毫秒
- upstream_http_名称：从上游服务返回响应头部的值

### 3.4. 反向代理模块

#### HTTP反向代理流程

![](/images/posts/nginx_proxy_stream.png)

nginx的反向代理过程在11个阶段的content阶段执行，指令为proxy_pass。当一个客户发起请求时，nginx会检查是否存在响应缓存cache，如果cache命中，则直接发送响应头部，如果未命中或者未开启cache，则根据指令生产发往上游头部及包体。当设置参数：

- proxy_request_buffering on时，nginx先读取完整的包体，然后根据负载均衡策略选择上游服务器，根据参数连接上游服务器后发送请求给上游。
- proxy_request_buffering off时，nginx







 