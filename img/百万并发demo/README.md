# 一、reactor是什么？

reactor是一种事件驱动的方式，通常用于处理大量的客户端请求，根据i/o上触发的读写事件，执行不同的回调函数

所以实现了从 select poll epoll 的IO管理变成了转为为事件的管理

## 二、reactor的工作流程

1. 监听事件 : 在当前的版本中就是监听注册事件和监听数据收发事件

2. 事件触发 : 事件准备进行一些操作时候

3. 事件分发 : 将事件分给不同的处理函数进行处理

4. 处理事件 : 事件处理函数进行相应的操作注册 、读、写

## 三、reactor的实现步骤

注 : 其实reactor的实现的底层还是基于epoll来做的 , 只是说从的IO管理变成了对事件的管理 , 不同的事件会执行不同的回调函数

1. 设计连接池 (reactor结构体) , 设计reactor需要的基本信息

2. 设计事件触发和分发机制

3. 设计处理不同事件相应的回调函数

4. 主循环

### 设计连接池

首先我们的场景是设计一个TCP server可以进行连接和读写操作

1. 每一个 TCP 的连接状态对应着内核中的一个 fd (file description)

2. 每一个 fd 的读写操作都是独立的所以需要独立的接受缓冲区和发送缓冲区

3. 每一个 fd 都有自己的回调函数

于是就有

```c
struct reactor {
    int fd;

    char rbuffer[BUFFERLENGTH];
    int rlength;
    char wbuffer[BUFFERLENGTH];
    int wlength;

    RCALLBACK send_callback;
    union {
        RCALLBACK recv_callback;
        RCALLBACK accept_callback;
    }r_action;
};
```

但是这里有一个实现细节 : 为什么后面要用一个联合体?

先建立 model :

```
站在服务端来说
io       --> event    --> callback
listenfd --> EPOLLIN  --> accept_cb
clientfd --> EPOLLIN  --> recv_cb
clientfd --> EPOLLOUT --> send_cb
```

我们站在事件的角度说 , 这里其实只有两种event EPOLLIN EPOLLOUT

此外这两个 EPOLLIN 事件对应的 accept_cb 和 recv_cb 是一个或的关系一个事件只能在注册之后才能开始接受数据 , 于是这里使用一个联合体

### 事件关联函数

事件关联函数在这里是为了方便修改监听事件的监听模式 EPOLLIN | EPOLLOUT

```c
int set_event(int fd, int event, int flag)
{
    if(flag) // 1  add
    {

        struct epoll_event ev;
        ev.events = event;
        ev.data.fd = fd;
        epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

    }else{  //0 修改

        struct epoll_event ev;
        ev.events = event;
        ev.data.fd = fd;
        epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
    }
}
```

### 回调函数

三种事件对应三种不同的处理函数

**accept_cb**

```c
int accept_cb(int fd)
{
    struct sockaddr_in clientaddr;
    socklen_t len = sizeof(clientaddr);
    int clientfd = accept(fd, (struct sockaddr*)&clientaddr, &len);
    if(clientfd < 0)
    {
        printf("accept errno: %d  ----> %s \n", errno, strerror(errno));
        return -1;
    }
    event_register(clientfd, EPOLLIN);  // 或上就是加个共功能， 
      //   默认水平触发
      //   EPOLLET 边沿触发   event_register(clientfd, EPOLLIN | EPOLLET);

    if((clientfd % 1000) == 0){
        struct timeval current;
        gettimeofday(&current, NULL);
        int time_used = TIME_SUB_MS(current, begin);
        memcpy(&begin, &current, sizeof(struct timeval));

        printf("client finished: %d, time_used:%d \n", clientfd, time_used);
    }
    return 0;

}

int event_register(int fd, int event)
{
  conn_list[fd].fd = fd;

  conn_list[fd].r_action.recv_callback = recv_cb;
  conn_list[fd].send_callback = send_cb;

  memset(conn_list[fd].rbuffer, 0, BUFFER_LENGTH);
  conn_list[fd].rlength = 0;

  memset(conn_list[fd].wbuffer, 0, BUFFER_LENGTH);
  conn_list[fd].wlength = 0;

  return set_event(fd, event, 1);
}
```

**recv_cb**

```c
int recv_cb(int fd)
{
    memset(conn_list[fd].rbuffer, 0, BUFFER_LENGTH);
    // 阻塞的fd
    int count = recv(fd, conn_list[fd].rbuffer, BUFFER_LENGTH, 0);
    if(count == 0)
    {
        printf("client disconnect: %d \n", fd);
        close(fd);
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL); 
        return 0;  
        // continue;
    }
    else if(count  < 0) // 返回-1是因为 对方发出rst包（tcp包）正常现象
    {
        printf("count : %d, errno: %d, %s\n", count, errno, strerror(errno));
        close(fd);
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        return 0;
    }
    conn_list[fd].wLength = count;
    // printf("recv:%s", conn_list[fd].rbuffer);
    conn_list[fd].wLength = conn_list[fd].rLength;
    memcpy(conn_list[fd].wbuffer, conn_list[fd].rbuffer, conn_list[fd].wLength);
    set_event(fd, EPOLLOUT, 0);
    return count;
}
```

**send_cb**

```c
int send_cb(int fd)
{
    int count  = 0;
    if(conn_list[fd].status == 1)
    {
        count = send(fd, conn_list[fd].wbuffer, 
                    conn_list[fd].wLength, 0);
        // 继续设置该文件描述符为 输出事件
        set_event(fd, EPOLLOUT, 0);
    }else if(conn_list[fd].status == 2){
        set_event(fd, EPOLLOUT, 0);
    }else if(conn_list[fd].status == 0)
    {
        // 如果wlength不等于0， 调用一下send
        if(conn_list[fd].wLength !=0)
        {
            count = send(fd, conn_list[fd].wbuffer, 
                     conn_list[fd].wLength, 0);
        }
        set_event(fd, EPOLLIN, 0);
    }
    return count;
}
```

### 事件分发

```c
  while (1)
  {
    struct epoll_event events[1024] = {0};
    int nready = epoll_wait(epfd, events, 1024, -1);
    for (int i = 0; i < nready; i++)
    {
      int connfd = events[i].data.fd;
      if ((events[i].events & EPOLLIN))
      {
        conn_list[connfd].r_action.accept_callback(connfd);
      }
      if (events[i].events & EPOLLOUT)
      {
        conn_list[connfd].send_callback(connfd);
      }
    }
  }
```

其实这里有一个细节 : 程序是怎么判断当前的 fd 是注册过了还是没有注册?

ans : 其实通过通用体来实现的 , 在这个 event_register 函数中的 :

```c
conn_list[fd].r_action.recv_callback = recv_cb;
```

fd 在注册的时候会把union的函数地址改成 recv_cb 但是从上层来看调用的是accept_cb 但是底层实际的函数地址是 recv_cb

### 事件处理

这里需要讲一下 EPOLL 的两种触发方式 水平触发（LT）边沿触发（ET）

```
LT : 
对于读事件：只要socket上有未读完的数据，EPOLLIN 就会一直触发；
对于写事件：只要socket可写，指的是tcp缓冲区未满，EPOLLOUT就会一直触发
```

```
ET:
对于读事件：只要socket的读缓冲区有数据，从无到有，EPOLLIN才会读
对于写事件：只有在socket的写缓冲区从不可写变为可写，EPOLLOUT 才会触发
```

# 百万并发测试

影响服务器并发的因素有下面几个

1. socket : too many open files 操作系统限制文件打开数量
   通过 ulimit -a 查看数量 通过 ulimit -n 1048576来修改
   ![](https://camo.githubusercontent.com/d591b9f9b5ccdc638d9f94703f4a5c6814867f37c86de2c2fb79d73bdd8794d2/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340633533656235373339393862373134623063393038316461656238386637616335636232306165382f696d672f25453725393925424525453425423825383725453525423925423625453525384625393164656d6f2f6d763774373365712e706e67)

2. 修改/etc/security/limits.conf
   ![](https://camo.githubusercontent.com/37863aecb394e7524969185ca06b68eb4c4bd489ec73b12c17022cc333869bf8/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340633533656235373339393862373134623063393038316461656238386637616335636232306165382f696d672f25453725393925424525453425423825383725453525423925423625453525384625393164656d6f2f6d76377437346d732e706e67)

3. 系统资源的限制
   通常一个tcp 连接由五元组组成 
   服务器ip 服务器端口 客户端端ip 客户端端口 4层网络协议 
   可以在服务器端开放多个端口，也就是创建多个listenfd, 这样可以提高服务器的并发数量

4. net.nf_conntrack_max 修改网络防火墙允许链路建立的个数
   ![](https://camo.githubusercontent.com/d591b9f9b5ccdc638d9f94703f4a5c6814867f37c86de2c2fb79d73bdd8794d2/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340633533656235373339393862373134623063393038316461656238386637616335636232306165382f696d672f25453725393925424525453425423825383725453525423925423625453525384625393164656d6f2f6d763774373365712e706e67)

5. 实测截图
   ![](https://camo.githubusercontent.com/81927622c5a240e0d9c63e049c223730afe3eae255a78a7a4d968e10cd871610/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340633533656235373339393862373134623063393038316461656238386637616335636232306165382f696d672f25453725393925424525453425423825383725453525423925423625453525384625393164656d6f2f6d763774373630302e706e67)

欢迎大家指出小白的错误 , 谢谢!
