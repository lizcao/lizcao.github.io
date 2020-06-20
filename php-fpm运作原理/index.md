# php-fpm运作原理

### PHP-FPM 的运作模式

PHP-FPM是一个多进程的 FastCGI 管理程序，是绝大多数 PHP应用所使用的运行模式。

假设我们使用 `Nginx` 提供 `HTTP` 服务（`Apache` 同理），所有客户端发起的请求最先抵达的都是 `Nginx`，然后 `Nginx` 通过 `FastCGI` 协议将请求转发给 `PHP-FPM` 处理，`PHP-FPM` 的 `Worker 进程` 会抢占式的获得 CGI 请求进行处理，这个处理指的就是，等待 `PHP` 脚本的解析，等待业务处理的结果返回，完成后回收子进程，这整个的过程是阻塞等待的，也就意味着 `PHP-FPM` 的进程数有多少能处理的请求也就是多少，假设 `PHP-FPM` 有 `200` 个 `Worker 进程`，一个请求将耗费 `1` 秒的时间，那么简单的来说整个服务器理论上最多可以处理的请求也就是 `200` 个，`QPS` 即为 `200/s`，在高并发的场景下，这样的性能往往是不够的，尽管可以利用 `Nginx` 作为负载均衡配合多台 `PHP-FPM` 服务器来提供服务，但由于 `PHP-FPM` 的阻塞等待的工作模型，一个请求会占用至少一个 `MySQL` 连接，多节点高并发下会产生大量的 `MySQL` 连接，而 `MySQL` 的最大连接数默认值为 `100`，尽管可以修改，但显而易见该模式没法很好的应对高并发的场景。

PHP-FPM 即 PHP FastCGI 进程管理器，要了解 PHP-FPM ，首先要看看 CGI 与 FastCGI 的关系。

### CGI

CGI 的英文全名是 Common Gateway Interface，即通用网关接口，是 Web 服务器调用外部程序时所使用的一种服务端应用的规范。

早期的 Web 通信只是按照客户端请求将保存在 Web 服务器硬盘中的数据转发过去而已，这种情况下客户端每次获取的信息也是同样的内容（即静态请求，比如图片、样式文件、HTML文档），而随着 Web 的发展，Web 所能呈现的内容更加丰富，与用户的交互日益频繁，比如博客、论坛、电商网站、社交网络等。

这个时候仅仅通过静态资源已经无法满足 Web 通信的需求，所以引入 CGI 以便客户端请求能够触发 Web 服务器运行另一个外部程序，客户端所输入的数据也会传给这个外部程序，该程序运行结束后会将生成的 HTML 和其他数据通过 Web 服务器再返回给客户端（即动态请求，比如基于 PHP、Python、Java 实现的应用）。利用 CGI 可以针对用户请求动态返回给客户端各种各样动态变化的信息。

### FastCGI

FastCGI 顾名思义，是 CGI 的升级版本，为了提升 CGI 的性能而生，CGI 针对每个 HTTP 请求都会 fork 一个新进程来进行处理（解析配置文件、初始化执行环境、处理请求），然后把这个进程处理完的结果通过 Web 服务器转发给用户，刚刚 fork 的新进程也随之退出，如果下次用户再请求动态资源，那么 Web 服务器又再次 fork 一个新进程，如此周而复始循环往复。

而 FastCGI 则会先 fork 一个 master 进程，解析配置文件，初始化执行环境，然后再 fork 多个 worker 进程（与 Nginx 有点像），当 HTTP 请求过来时，master 进程将其会传递给一个 worker 进程，然后立即可以接受下一个请求，这样就避免了重复的初始化操作，效率自然也就提高了。

而且当 worker 进程不够用时，master 进程还可以根据配置预先启动几个 worker 进程等着；

当空闲 worker 进程太多时，也会关掉一些，这样不仅提高了性能，还节约了系统资源。

### PHP-FPM

FastCGI 只是一个协议规范，需要每个语言具体去实现，PHP-FPM 就是 PHP 版本的 FastCGI 协议实现。

master进程只有一个，负责监听端口，接收来自服务器的请求，而worker进程则一般有多个（具体数量根据实际需要配置），每个进程内部都嵌入了一个PHP解释器，
是PHP代码真正执行的地方，比如：1个master进程，2个worker进程。
从FPM接收到请求，到处理完毕，其具体的流程如下

1. FPM的master进程接收到请求。
2. master进程根据配置指派特定的worker进程进行请求处理，如果没有可用进程，返回错误，这也是我们配合Nginx遇到502错误比较多的原因。
3. worker进程处理请求，如果超时，返回504错误。
4. 请求处理结束，返回结果。

### PHP-FPM配置文件

```
pid = run/php-fpm.pid
#pid设置，默认在安装目录中的var/run/php-fpm.pid，建议开启
 
error_log = log/php-fpm.log
#错误日志，默认在安装目录中的var/log/php-fpm.log
 
log_level = notice
#错误级别. 可用级别为: alert（必须立即处理）, error（错误情况）, warning（警告情况）, notice（一般重要信息）, debug（调试信息）. 默认: notice.
 
emergency_restart_threshold = 60
emergency_restart_interval = 60s
#表示在emergency_restart_interval所设值内出现SIGSEGV或者SIGBUS错误的php-cgi进程数如果超过 emergency_restart_threshold个，php-fpm就会优雅重启。这两个选项一般保持默认值。
 
process_control_timeout = 0
#设置子进程接受主进程复用信号的超时时间. 可用单位: s(秒), m(分), h(小时), 或者 d(天) 默认单位: s(秒). 默认值: 0.
 
daemonize = yes
#后台执行fpm,默认值为yes，如果为了调试可以改为no。在FPM中，可以使用不同的设置来运行多个进程池。 这些设置可以针对每个进程池单独设置。
 
listen = 127.0.0.1:9000
#fpm监听端口，即nginx中php处理的地址，一般默认值即可。可用格式为: ‘ip:port’, ‘port’, ‘/path/to/unix/socket’. 每个进程池都需要设置.
 
listen.backlog = -1
#backlog数，-1表示无限制，由操作系统决定，此行注释掉就行。

listen.allowed_clients = 127.0.0.1
#允许访问FastCGI进程的IP，设置any为不限制IP，如果要设置其他主机的nginx也能访问这台FPM进程，listen处要设置成本地可被访问的IP。默认值是any。每个地址是用逗号分隔. 如果没有设置或者为空，则允许任何服务器请求连接
 
listen.owner = www
listen.group = www
listen.mode = 0666
#unix socket设置选项，如果使用tcp方式访问，这里注释即可。
 
user = www
group = www
#启动进程的帐户和组
 
pm = dynamic #对于专用服务器，pm可以设置为static。
#如何控制子进程，选项有static和dynamic。如果选择static，则由pm.max_children指定固定的子进程数。如果选择dynamic，则由下开参数决定：
pm.max_children #，子进程最大数
pm.start_servers #，启动时的进程数
pm.min_spare_servers #，保证空闲进程数最小值，如果空闲进程小于此值，则创建新的子进程
pm.max_spare_servers #，保证空闲进程数最大值，如果空闲进程大于此值，此进行清理
 
pm.max_requests = 1000
#设置每个子进程重生之前服务的请求数. 对于可能存在内存泄漏的第三方模块来说是非常有用的. 如果设置为 ’0′ 则一直接受请求. 等同于 PHP_FCGI_MAX_REQUESTS 环境变量. 默认值: 0.
 
pm.status_path = /status
#FPM状态页面的网址. 如果没有设置, 则无法访问状态页面. 默认值: none. munin监控会使用到
 
ping.path = /ping
#FPM监控页面的ping网址. 如果没有设置, 则无法访问ping页面. 该页面用于外部检测FPM是否存活并且可以响应请求. 请注意必须以斜线开头 (/)。
 
ping.response = pong
#用于定义ping请求的返回相应. 返回为 HTTP 200 的 text/plain 格式文本. 默认值: pong.
 
request_terminate_timeout = 0
#设置单个请求的超时中止时间. 该选项可能会对php.ini设置中的’max_execution_time’因为某些特殊原因没有中止运行的脚本有用. 设置为 ’0′ 表示 ‘Off’.当经常出现502错误时可以尝试更改此选项。
 
request_slowlog_timeout = 10s
#当一个请求该设置的超时时间后，就会将对应的PHP调用堆栈信息完整写入到慢日志中. 设置为 ’0′ 表示 ‘Off’
 
slowlog = log/$pool.log.slow
#慢请求的记录日志,配合request_slowlog_timeout使用
 
rlimit_files = 1024
#设置文件打开描述符的rlimit限制. 默认值: 系统定义值默认可打开句柄是1024，可使用 ulimit -n查看，ulimit -n 2048修改。
 
rlimit_core = 0
#设置核心rlimit最大限制值. 可用值: ‘unlimited’ 、0或者正整数. 默认值: 系统定义值.
 
chroot =
#启动时的Chroot目录. 所定义的目录需要是绝对路径. 如果没有设置, 则chroot不被使用.
 
chdir =
#设置启动目录，启动时会自动Chdir到该目录. 所定义的目录需要是绝对路径. 默认值: 当前目录，或者/目录（chroot时）
 
catch_workers_output = yes
#重定向运行过程中的stdout和stderr到主要的错误日志文件中. 如果没有设置, stdout 和 stderr 将会根据FastCGI的规则被重定向到 /dev/null . 默认值: 空.
```


