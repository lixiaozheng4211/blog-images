# 网络IO与IO多路复用 select poll epoll(下)

首先需要介绍一下在Linux中很重要的一个操作入口 fd : 

fd 是一个 int 类型的值 , 是进程操作内核socket对象的入口

![](https://camo.githubusercontent.com/7892c4aa77473cf025ce29949260173ec5a7ff8e5d982eac2a5707a13218e6f8/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340623939303965326466633961386661626136626238656265656232616639343033613436316266392f696d672f254537254244253931254537254242253943696f254534254238253845696f25453525413425394125453825423725414625453525413425384425453725393425413873656c656374706f6c6c65706f6c6c2532382545342542382538422532392f6d763774377869662e706e67)

从这里可以看出系统 fd 的前面三个都是确定的

分别是 标准输入 , 标准输出 , 标准错误

在Linux下操作一切皆是文件 , 文件描述符 fd 也就是操作入口 

一连接一线程的这种做法有什么优点/缺点 ? 

优点 : 

    1. 代码逻辑简单

缺点

    1.不利于并发

## IO多路复用 : select

select 是一个bit位集合 , 默认是1024个bit位

这是原始定义 : 

![](https://camo.githubusercontent.com/32e0e8ef3cf680b9cbcec6bdb5aa98c21d80f80507a934160189100025c0e3f3/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340623939303965326466633961386661626136626238656265656232616639343033613436316266392f696d672f254537254244253931254537254242253943696f254534254238253845696f25453525413425394125453825423725414625453525413425384425453725393425413873656c656374706f6c6c65706f6c6c2532382545342542382538422532392f6d763774377a78342e706e67)

有几个需要注意的点 : 

1. 每次调用需要把fd_set集合 , 从用户空间copy到内核空间

2. maxfd , 遍历到最大的maxfd 
   
   ```
   for (int i = sockfd + 1 ; i <= maxfd ; i++) 
   ```

3. 最开始我们定义的 sockfd 其实是一个监听fd也就是说如果这个fd收到信息就表示有新的连接请求 调用accept处理
   否则就是clientfd收到的信息需要接受消息 , 调用recv处理

基础语法 : 

```c
 fd_set rfds, rset; // 定义两份 , 一份给应用 , 一份给内核
  FD_ZERO(&rfds);
  FD_SET(sockfd, &rfds);
  int maxfd = sockfd;
  /*
    为什么要定义两个fd_set ? 
    因为select会原地修改集合
    从 : 所有你关心的、想要监听可读事件的 fd
    变成 : 哪些 fd 现在可读
  */
  while (1)
  {
    rset = rfds;
    int nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
    /*
      这三个参数是需要关注的fd的不同特征
      readfds   : 关注 ... 是否可以读
      writefds  : 关注 ... 是否可以写
      exceptfds : 关注 ... 是否出错
      return    : 有多少fd是可读可写出错的

      为什么这里要用 maxfd + 1 ? 
      ans : 因为select维护的位图从0开始 , 如果传入 maxfd 实际只会检查到 maxfd - 1
    */
    /*
      网络IO : 有两种操作 --> accept / recv
      1. 如果是没有创建连接就是走accept
      2. 否则连接之后就是接受数据走recv
    */
    if (nready < 0)
    {
      perror("select");
      continue;
    }
    /*
      关键 API : 
      FD_ISSET(fd , set) 
      表示这个fd在这个集合中是否被置位
    */
    if (FD_ISSET(sockfd, &rset)) // 表示监听fd收到数据需要建立新的连接
    {
      struct sockaddr_in client_addr;
      socklen_t client_len = sizeof(client_addr);
      int clientfd = accept(sockfd, (struct sockaddr *)&client_addr, &client_len);
      if (clientfd < 0)
      {
        perror("accept");
        continue;
      }
      FD_SET(clientfd, &rfds); // 置位 , 可读集合
      if (clientfd > maxfd)
        maxfd = clientfd;
      printf("new client fd:%d\n", clientfd);
    }

    for (int i = sockfd + 1 ; i <= maxfd; i++) // 因为sockfd是最先定义的所以之后的fd肯定在其后面
    {
      if (FD_ISSET(i, &rset))
      {
        char buffer[BUFFER_SIZE];
        memset(buffer, 0, BUFFER_SIZE);
        int count = recv(i, buffer, BUFFER_SIZE, 0);
        if (count <= 0)
        {
          printf("client disconnect : %d \n", i);
          /*
            先 clear 再close
          */
          FD_CLR(i, &rfds);
          close(i);
          continue;
        }
        buffer[count] = '\0';
        printf("recv : %s \n", buffer);
        // echo回发
        send(i, buffer, count, 0);
      }
    }
  }
```

## IO多路复用 : poll

这是原始定义 这里有三个参数 : 

![](https://camo.githubusercontent.com/dfe1a4dac3f168753072081ff40f589996fce6a7de7af60a6ae60aaa2cf0df65/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340623939303965326466633961386661626136626238656265656232616639343033613436316266392f696d672f254537254244253931254537254242253943696f254534254238253845696f25453525413425394125453825423725414625453525413425384425453725393425413873656c656374706f6c6c65706f6c6c2532382545342542382538422532392f6d763774383161322e706e67)

1. fd (file description) 文件描述符

2. event 应用程序希望关注什么事件

3. revent 本次实际上发生了什么事件

其实和上面的select类似但是把集合变成了一个一个单元管理 

eg.     

```c
struct polled fds;
fds.fd = sockfd;
fds.event = POLLIN; // 表示监听这个socket的这个pollin事件

// 判断接收
if (fds.revent & POLLIN) { // 判断要监听的socket是否有这个pollin事件
    ...
} 
```

poll 的事件定义

在 `<poll.h>` 中定义，采用位掩码（十六进制）表示，可以通过按位或（`|`）组合

![](https://camo.githubusercontent.com/ef46fe1ce0d56a23582a95861920afbdf3d83da7473ea34474a32e5f805b58b9/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340623939303965326466633961386661626136626238656265656232616639343033613436316266392f696d672f254537254244253931254537254242253943696f254534254238253845696f25453525413425394125453825423725414625453525413425384425453725393425413873656c656374706f6c6c65706f6c6c2532382545342542382538422532392f6d76377438326d382e706e67)

从上到下分别是 : 

```c
/* These are specified by iBCS2 */
#define POLLIN     0x0001   // 表示关注数据可读（普通数据或优先级带数据）
#define POLLPRI    0x0002   // 表示关注高优先级数据可读（带外数据/紧急数据）
#define POLLOUT    0x0004   // 表示关注数据可写（写操作不会阻塞）
#define POLLERR    0x0008   // 表示发生错误（只能作为返回结果，不能在 events 中设置）
#define POLLHUP    0x0010   // 表示发生挂起，通常对端已关闭连接（只能作为返回结果）
#define POLLNVAL   0x0020   // 表示文件描述符无效，如 fd 未打开（只能作为返回结果）
```

基础语法 

```c
struct pollfd fds[1024];
  fds[sockfd].fd = sockfd;
  fds[sockfd].events = POLLIN;

  int maxfd = sockfd;

  while (1)
  {
    int nready = poll(fds, maxfd + 1, -1); // 这里使用maxfd + 1 同样是因为数组是0-base
    if (fds[sockfd].revents & POLLIN) // 表示监听fd收到数据需要建立新的连接
    {
      struct sockaddr_in client_addr;
      socklen_t client_len = sizeof(client_addr);
      int clientfd = accept(sockfd, (struct sockaddr *)&client_addr, &client_len);
      if (clientfd < 0)
      {
        perror("accept");
        continue;
      }
      fds[clientfd].fd = clientfd;
      fds[clientfd].events = POLLIN;
      if (clientfd > maxfd)
        maxfd = clientfd;
      printf("new client fd:%d\n", clientfd);
    }
    for (int i = sockfd + 1; i <= maxfd; i++) // 因为sockfd是最先定义的所以之后的fd肯定在其后面
    {
      if (fds[i].revents & POLLIN)
      {
        char buffer[BUFFER_SIZE];
        memset(buffer, 0, BUFFER_SIZE);
        int count = recv(i, buffer, BUFFER_SIZE, 0);
        if (count <= 0)
        {
          printf("client disconnect : %d \n", i);
          /*
            先 clear 再close
          */
          fds[i].fd = -1;
          fds[i].events = 0;
          close(i);
          continue;
        }
        buffer[count] = '\0';
        printf("recv : %s \n", buffer);
        // echo回发
        count = send(i, buffer, count, 0);
        printf("send : %d\n", count);
      }
    }
  }
```

从这里可以看出他与select的写法非常相似可以说简直是一模一样 , 就是初始化,判断方式,clear方式有点区别其他都是一样

## IO多路复用 : epoll

原始定义 : 

![](https://camo.githubusercontent.com/2a7a0bd0b0958e6a879683affec9f8e360499d9e2772e6602fb1119475808f78/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340623939303965326466633961386661626136626238656265656232616639343033613436316266392f696d672f254537254244253931254537254242253943696f254534254238253845696f25453525413425394125453825423725414625453525413425384425453725393425413873656c656374706f6c6c65706f6c6c2532382545342542382538422532392f6d763774386732722e706e67)

基础语法 

```c
  int epfd = epoll_create(1); 
  // 注意这里传入的是一个无效参数 , 为了兼容老版本 , 传入参数大于0即可

  struct epoll_event ev;
  ev.events = EPOLLIN;
  ev.data.fd = sockfd;

  epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);
  /*
    解释一下epoll_ctl : 将sockfd 添加到 epfd 的监听集合中，并绑定事件 ev
    四个参数 :
    epfd : 内核实例对象的资源标记
    EPOLL_CTL_ADD : 操作类型 , 这里是加入
    sockfd : 需要操作的文件描述符
    ev : 需要监听的事件
  */
  while (1)
  {
    struct epoll_event events[1024]; // 
    int nready = epoll_wait(epfd, events, 1024, -1);
    /*
      epoll_wait: 等待 epoll 实例上的 I/O 事件
      epfd   : epoll 句柄
      events : 输出数组，用于接收就绪的事件
      1024   : events 数组容量，本次最多返回的事件数
      -1     : 超时时间，-1 表示永久阻塞，直到有事件发生
      return : 就绪事件个数；0 超时；-1 出错
     */
    if (nready < 0)
    {
      perror("epoll_wait");
      continue; // 或者 break
    }

    for (int i = 0; i < nready; i++) 
    {
      int connfd = events[i].data.fd;

      if (connfd == sockfd) // 表示有新的连接
      {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        int clientfd = accept(sockfd, (struct sockaddr *)&client_addr, &client_len);

        if (clientfd < 0)
        {
          perror("accept");
          continue;
        }

        ev.events = EPOLLIN;
        ev.data.fd = clientfd;

        // 注意 : 添加的是 clientfd，不是 connfd , 添加的是新的fd
        epoll_ctl(epfd, EPOLL_CTL_ADD, clientfd, &ev);
        printf("new client fd:%d\n", clientfd);
      }
      else if (events[i].events & EPOLLIN)
      {
        char buffer[BUFFER_SIZE];
        memset(buffer, 0, BUFFER_SIZE);

        int count = recv(connfd, buffer, BUFFER_SIZE, 0);

        if (count <= 0)
        {
          printf("client disconnect : %d \n", connfd);
          /*
            先 clear 再close
          */
          epoll_ctl(epfd, EPOLL_CTL_DEL, connfd, NULL);
          close(connfd); 
          continue;
        }

        buffer[count] = '\0';
        printf("recv : %s \n", buffer);

        count = send(connfd, buffer, count, 0);
        printf("send : %d\n", count);
      }
    }
  }
```

![](https://camo.githubusercontent.com/d6f14df2195b6ef42242c7d230e89ca27ab33c3373e1d725a66d7f71b1ecefea/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340623939303965326466633961386661626136626238656265656232616639343033613436316266392f696d672f254537254244253931254537254242253943696f254534254238253845696f25453525413425394125453825423725414625453525413425384425453725393425413873656c656374706f6c6c65706f6c6c2532382545342542382538422532392f6d763774387465322e706e67)
epoll_ctl中的 op 定义

#### 思考 : 为什么不用遍历所有节点?

epoll 不需要每次扫描全部监控项来寻找就绪项，关键是内核能够维护就绪列表

红黑树则主要负责管理关系



#### 我们现在的解决的是什么?

ans :  IO事件管理 / IO事件触发

关键概念 : 在IO的生命周期是由无数个事件组成的 , 不同的事件 ==> 不同的回调函数(Callback)

#### 总结一下四种实现方式 : 一请求一线程 , select , poll , epoll 的区别是什么?

对于 **<u>pthread</u>** 

关键函数

```c
pthread_create(threadid, NULL, client_recv, p_connfd); 
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                          void *(*start_routine) (void *), void *arg);
```

主要是通过创建一个线程 , 然后注册回调函数client_recv 和 函数参数 p_connfd

实现简单 , 但是并发量有限

对于 **<u>select</u>** 

就是维护三个集合 : rfds , wfds , efds

每一次都把这个三个集合先copy到内核中 , 然后遍历这三个集合的全部元素挨个处理

关键函数和结构体

```c
  fd_set rfds, rset;
  FD_ZERO();
  FD_SET();
  FD_ISSET();
  int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```

实现了一个线程管理多个fd 但是每一次都要遍历所有的集合 , 相当于一直在询问

对于 **<u>poll</u>** 

从写法上来说和 select 没有多大区别

关键函数和结构体

```c
struct pollfd fds[1024]; // 用来存储事件
fds[i].fd; // file description
fds[i].event; // 监听事件 POLLIN ... 用bitmask管理
fds[i].revent; // 接收到的事件
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
/*
 * fds     : pollfd 数组，存放要监听的 fd 和关注的事件
 * nfds    : 数组长度，即监听多少个 fd
 * timeout : 超时时间（毫秒）
 *           -1 永久阻塞；0 立即返回；>0 等待指定毫秒
 * 返回    : >0 就绪 fd 个数；0 超时；-1 出错
 */
```

从集合管理变成了节点管理 , 但是依旧没有解决一直询问的问题 , 还是浪费时间和资源

对于 **<u>epoll</u>** 

关键函数与结构体 

```c
struct epoll_event ev;
int epoll_create(int size); // size 没有用到
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
int epoll_ctl(int epfd, int op, 
              int fd, struct epoll_event *event);                      
```

1. 节点是内核长期维护的 , 不需要每一次都反复copy

2. 在底层加了缓冲区 , 所有信息先放到缓冲区中然后统一拿取 , 提高了效率 , 节约了时间

#### 思考一个场景 :

现在我们有很多不同的IO事件 , 每一个不同事件有不同的回调处理函数 , 那我们要怎么管理这些回调函数呢? 

形象理解 : 事件 ==> 回调 ==>  相当于给了一个名片 ==> 有事情就call我

所以事件才是应用层更加关注的事情 :

引出 reactor : 从 IO 管理变成事件管理 

为什么用reactor? 

更好的关注事件处理
