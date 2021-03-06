# swoole进程模型

### 进程结构图

![351590748283](/images/351590748283.jpg)

### Reactor 线程

- Reactor 线程是在 Master 进程中创建的线程
- 负责维护客户端 `TCP` 连接、处理网络 `IO`、处理协议、收发数据
- 不执行任何 PHP 代码
- 将 `TCP` 客户端发来的数据缓冲、拼接、拆分成完整的一个请求数据包

### Worker 进程

- 接受由 `Reactor` 线程投递的请求数据包，并执行 `PHP` 回调函数处理数据
- 生成响应数据并发给 `Reactor` 线程，由 `Reactor` 线程发送给 `TCP` 客户端
- 可以是异步非阻塞模式，也可以是同步阻塞模式
- `Worker` 以多进程的方式运行

### TaskWorker 进程

- 接受由 `Worker` 进程通过 Swoole\Server->task/taskwait/taskCo/taskWaitMulti 方法投递的任务
- 处理任务，并将结果数据返回（使用 Swoole\Server->finish给 `Worker` 进程
- 完全是**同步阻塞**模式
- `TaskWorker` 以多进程的方式运行，[task 完整示例](https://wiki.swoole.com/#/start/start_task)

### Manager 进程

- 负责创建 / 回收 `worker`/`task` 进程

他们之间的关系可以理解为 `Reactor` 就是 `nginx`，`Worker` 就是 `PHP-FPM`。`Reactor` 线程异步并行地处理网络请求，然后再转发给 `Worker` 进程中去处理。`Reactor` 和 `Worker` 间通过 [unixSocket](https://wiki.swoole.com/#/learn?id=什么是ipc) 进行通信。

在 `PHP-FPM` 的应用中，经常会将一个任务异步投递到 `Redis` 等队列中，并在后台启动一些 `PHP` 进程异步地处理这些任务。`Swoole` 提供的 `TaskWorker` 是一套更完整的方案，将任务的投递、队列、`PHP` 任务处理进程管理合为一体。通过底层提供的 `API` 可以非常简单地实现异步任务的处理。另外 `TaskWorker` 还可以在任务执行完成后，再返回一个结果反馈到 `Worker`。

`Swoole` 的 `Reactor`、`Worker`、`TaskWorker` 之间可以紧密的结合起来，提供更高级的使用方式。

一个更通俗的比喻，假设 `Server` 就是一个工厂，那 `Reactor` 就是销售，接受客户订单。而 `Worker` 就是工人，当销售接到订单后，`Worker` 去工作生产出客户要的东西。而 `TaskWorker` 可以理解为行政人员，可以帮助 `Worker` 干些杂事，让 `Worker` 专心工作。

如图：

![process_demo](https://wiki.swoole.com/_images/server/process_demo.png)




