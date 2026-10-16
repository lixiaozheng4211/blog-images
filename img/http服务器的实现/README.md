# 基于reactor模型的http服务器

## 前言

基于reactor模型的实现一个简单的http demo服务器，主要是为了学习**基于reactor，http协议该如何封装，如何解耦业务**，所以HTTP的内容只做简单介绍

## HTTP

这里只讨论 HTTP/1.1：HTTP 规定请求和响应怎么写，TCP 负责传输字节。

1. 解析HTTP 的请求消息

2. 根据业务对数据进行处理

3. 封装HTTP响应消息发回去

## HTTP请求消息格式

请求格式：请求行 → 请求头 → 空行 → 请求体（可以没有）。头部以 `\r\n\r\n` 结束。

![](https://camo.githubusercontent.com/9e0e1a8a4baebf249559b034e7e05012d93f490a925dbb2cc2a4b2787dea0226/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e6832392e706e67)

![](https://camo.githubusercontent.com/b2e2cc40e69a27caca2a484832d578a032b3a3219f95b873bb16095ebbf8c210/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e696a6c2e706e67)

## HTTP响应格式

响应格式：状态行 → 响应头 → 空行 → 响应体（可以没有）。

![](https://camo.githubusercontent.com/9490a9cc1caed3a5fc0ccfda27e8ea58c8d45e5a5baa1f0c2869726b2bcb6921/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e6b35682e706e67)

![](https://camo.githubusercontent.com/126c8502396c00289fe65a2e9e54126453b66a139c0cfc2bc00d38a4edc8dc30/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e6d386a2e706e67) 

## HTTP的业务流程

这个暂时不在这里展开 , 但是我们现在只要清楚如果客户端通过 HTTP 来获取资源例如下面的 index.html 那么服务端就会通过 TCP 把这个文件发送出去

![](https://camo.githubusercontent.com/2197c15651fc3d431e54ddbbb590a38fa01af7ae05697259941a254fe60485b2/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e6e6a642e706e67)

![](https://camo.githubusercontent.com/38e2ed1d52f4bff478a40aea7dd1de73d5dc8f00e2471fa5b9c24d6dcf765fbf/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e7037792e706e67)

![](https://camo.githubusercontent.com/00937dc868d03ccb48966b8798c7001dc9240afa8274f80f98972ef796a642ba/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e7168352e706e67)

这就是大致的流程

## reactor

### reactor 是什么 :

Reactor 就是：等事件发生，再调用对应的处理函数。比如连接可读时去接收，可写时继续发送。

这里用 epoll 等事件，用非阻塞 socket 避免卡在网络读写上。

### reactor 怎么用 :

1. webserver

2. websocket

在这个 demo 的接口设计中，可以分别提供请求处理和响应处理函数，例如：

```c
int http_response(struct conn *c);
int http_request(struct conn *c);

int ws_response(struct conn *c);
int ws_request(struct conn *c);
```

这里以 Webserver 举例 : 

重点看 `int http_response(struct conn *c);` 如何准备不同内容。这些函数名是这个 demo 自己定义的。

从发送不同类型的数据来说明这个解耦 : 

> 下面是学习片段，还不是完整可用的服务器。先看流程

#### 字符串

```c
int http_response(struct conn *c)
{
  c->wlength = sprintf(c->wbuffer,
                       "HTTP/1.1 200 OK\r\n"
                       "Content-Type: text/html\r\n"
                       "Accept-Ranges: bytes\r\n"
                       "Content-Length: 84\r\n"
                       "Date: Tue, 30 Apr 2024 13:16:46 GMT\r\n\r\n"
                       "<html><head><title>QC.yes</title></head><body><h1>hello world</h1></body></html>\r\n\r\n");
return 0;
}
```

![](https://camo.githubusercontent.com/56d9ce1624bf8f143cdf8ca002531eaa5a16f82ce2272707d5c15cd24d8ea96d/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e7837312e706e67)

`Content-Length` 只算正文的字节数，不算响应头和字符串末尾的 `\0`。这里连同正文末尾的换行，共 84 字节。

#### HTML文件

一次读入缓冲区的简化实现：

> 问题：一次 `read()` 不保证读完整个文件，缓冲区也可能装不下。

```c
int http_response(struct conn *c)
{
  int filefd = open("index.html", O_RDONLY);
  if (filefd < 0)
  {
    log("file not exists \n");
    close(filefd);
    return -2;
  }
  // 获取文件大小
  struct stat stat_buf;
  fstat(filefd, &stat_buf);

  c->wlength = sprintf(c->wbuffer,
                       "HTTP/1.1 200 OK\r\n"
                       "Content-Type: text/html\r\n"
                       "Accept-Ranges: bytes\r\n"
                       "Content-Length: %ld\r\n"
                       "Date: Tue, 30 Apr 2024 13:16:46 GMT\r\n\r\n",
                       stat_buf.st_size);
  int count = read(filefd, c->wbuffer + c->wlength, BUFFER_LENGTH - c->wlength);
  c->wlength += count;
  close(filefd);
  return 0;
}
```

状态机的初步实现：

> 问题：这段代码调用一次 `sendfile()` 就标记完成，实际上可能还没发完。图片版本也一样。

```c
typedef enum
{
  start = 0,
  continue_send,
  finished_send,
} SM_status;

int http_response(struct conn *c)
{
  int filefd = open("index.html", O_RDONLY);
  // 获取文件大小
  struct stat stat_buf;
  fstat(filefd, &stat_buf);
  if (c->status == start)
  {
    c->wlength = sprintf(c->wbuffer,
                         "HTTP/1.1 200 OK\r\n"
                         "Content-Type: text/html\r\n"
                         "Accept-Ranges: bytes\r\n"
                         "Content-Length: %ld\r\n"
                         "Date: Tue, 30 Apr 2024 13:16:46 GMT\r\n\r\n",
                         stat_buf.st_size);
    c->status = continue_send;
  }
  else if (c->status == continue_send)
  {
    int ret = sendfile(c->fd, filefd, NULL, stat_buf.st_size);
    if (ret == -1)
    {
      log("errno : %d\n", errno);
    }
    c->wlength = 0;
    memset(c->wbuffer, 0, BUFFER_LENGTH);
    c->status = finished_send;
  }
  else if (c->status == finished_send)
  {
    c->wlength = 0;
    memset(c->wbuffer, 0, BUFFER_LENGTH);
    c->status = start;
  }

  close(filefd);
  return 0;
}
```

这里有一个细节 : 为什么在这里设计的时候要用一个状态机 , 有什么好处? 

因为一次可能发不完，需要记住两件事：**现在在发什么，以及已经发了多少。**

状态记录阶段，偏移量记录进度。暂时发不了，就先处理其他连接，等可写后接着发。

然后我们来看看reactor的在这里的实现 : 

```c
int send_cb(int fd) {
  http_response(&conn_list[fd]);
  int count;
  int clstatus = conn_list[fd].status;
  if (clstatus == continue_send)
  {
    count = send(fd, conn_list[fd].wbuffer, conn_list[fd].wlength, 0);
    set_event(fd, EPOLLOUT, 0);
  }
  else if (clstatus == finished_send)
  {
    set_event(fd, EPOLLOUT, 0);
  }
  else if (clstatus == start)
  {
    if (conn_list[fd].wlength != 0)
    {
      int count = send(fd, conn_list[fd].wbuffer, conn_list[fd].wlength, 0);
    }
    set_event(fd, EPOLLIN, 0);
  }
  return count;
}
```

但是从表面上看好像没有说可以循环发送啊? 

这里确实只尝试发送一次，没有保存进度。业务简单也不能保证一次发完。

改进方向：在连接里保存响应头进度 `wpos`、文件进度 `woffset`，以及要继续发送的文件 `filefd`。

```c
int send_cb(int fd)
{
    struct conn *c = &conn_list[fd];

    // 要求使用新的配套版本：只在 start 时准备响应，其他状态不推进
    http_response(c);

    if (c->status == continue_send)
    {
        // ---- 阶段 1：发响应头 ----
        if (c->wpos < c->wlength)
        {
            ssize_t n = send(fd, c->wbuffer + c->wpos,
                             c->wlength - c->wpos, MSG_NOSIGNAL);
            if (n < 0)
            {
                if (errno == EINTR || errno == EAGAIN || errno == EWOULDBLOCK)
                {
                    set_event(fd, EPOLLOUT, 0);  // 头没发完，继续关注写
                    return 0;
                }
                return close_connection(fd);
            }
            c->wpos += n;
            if (c->wpos < c->wlength)
            {
                set_event(fd, EPOLLOUT, 0);  // 头还没发完
                return 0;
            }
            // 头发完了，继续往下发文件
        }

        // ---- 阶段 2：发文件内容（分块） ----
        while (c->woffset < c->fsize)
        {
            off_t offset = c->woffset;
            ssize_t n = sendfile(fd, c->filefd, &offset,
                                 c->fsize - c->woffset);
            if (n < 0)
            {
                if (errno == EINTR || errno == EAGAIN || errno == EWOULDBLOCK)
                {
                    set_event(fd, EPOLLOUT, 0);  // 缓冲区满，下次继续
                    return 0;
                }
                return close_connection(fd);
            }
            if (n == 0) break;  // 文件读完
            c->woffset = offset;
        }

        if (c->woffset >= c->fsize)
        {
            // 文件发完了，进入收尾
            c->status = finished_send;
            set_event(fd, EPOLLOUT, 0);  // 再进来一次做收尾
        }
        else
        {
            // 还没发完，保持 continue_send
            set_event(fd, EPOLLOUT, 0);
        }
    }
    else if (c->status == finished_send)
    {
        // ---- 阶段 3：收尾 ----
        if (c->filefd >= 0) {
            close(c->filefd);
            c->filefd = -1;
        }
        c->wlength = 0;
        c->wpos = 0;
        c->woffset = 0;
        c->status = start;
        set_event(fd, EPOLLIN, 0);  // 写循环结束，切回读
    }

    return 0;
}
```

继续发送的规则：**能发就发，发不了就记住位置、等事件，发完就取消写事件关注。** 不是只监听、不调用发送函数，也不是遇到 EAGAIN 还在原地重试

![](https://camo.githubusercontent.com/663e0f795c1bba24221b0fca2e512f70621e2b14a0a41114d51b9048c43f132e/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d61676573406d61696e2f696d672f313739323033353938323532342e706e67)

![](https://camo.githubusercontent.com/5bf20c529caa1d2c73183e7d930696b63b98810af250f21544ec21ec539b4765/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686e7a34732e706e67)

#### 图片数据

```c
int http_response(struct conn *c)
{
  int filefd = open("vip.png", O_RDONLY);
  // 获取文件大小
  struct stat stat_buf;
  fstat(filefd, &stat_buf);
  if (c->status == start)
  {
    c->wlength = sprintf(c->wbuffer,
                         "HTTP/1.1 200 OK\r\n"
                         "Content-Type: image/png\r\n"
                         "Accept-Ranges: bytes\r\n"
                         "Content-Length: %ld\r\n"
                         "Date: Tue, 30 Apr 2024 13:16:46 GMT\r\n\r\n",
                         stat_buf.st_size);
    c->status = continue_send;
  }
  else if (c->status == continue_send)
  {
    int ret = sendfile(c->fd, filefd, NULL, stat_buf.st_size);
    if (ret == -1)
    {
      log("errno : %d\n", errno);
    }
    c->wlength = 0;
    memset(c->wbuffer, 0, BUFFER_LENGTH);
    c->status = finished_send;
  }
  else if (c->status == finished_send)
  {
    c->wlength = 0;
    memset(c->wbuffer, 0, BUFFER_LENGTH);
    c->status = start;
  }

  close(filefd);
  return 0;
}
```

HTML 换成 PNG，主要变的是文件和 Content-Type，连接管理和发送逻辑可以共用，这就是解耦的方向。

不过这里的 http_response() 还直接调用 sendfile()，业务准备和网络发送还混在一起。

在这里还要注意一个状态机的小细节 : 

其实这个 `http_response()`函数其实在开始的时候就走了一步状态机 : 从Start走到了continue_send , 所以在返回之后在 reactor 的 send_cb 中是

```c
  int count;
  int clstatus = conn_list[fd].status;
  if (clstatus == continue_send)
  {
    count = send(fd, conn_list[fd].wbuffer, conn_list[fd].wlength, 0);
    set_event(fd, EPOLLOUT, 0);
  }
```

第一轮循环直接是从continue_send开始 , 不是从Start开始 , 因为当前的状态不是Start了

![](https://camo.githubusercontent.com/c1bd507de22ed47e6832cbcf00b51ff055b8e0d7295aa5cb02aca0e979f44abd/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686f3067312e706e67)

![](https://camo.githubusercontent.com/626db48cba8f485e1cc46bb34041aa4a3a7eb6a05cca6a8c20aca8f99273c1ae/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686f3338702e706e67)

## 性能测试

httpserver的qps测试 : 

![](https://camo.githubusercontent.com/a63d6657bac4f6b9443aafe240d5ec45eb85906ffc856a7fae50193010c4d88f/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686f346e792e706e67)



这里重点看一下 websocket 的请求信息

![](https://camo.githubusercontent.com/a115a435809aa01d2cb8dcecf071346b5f8ea4caf4703a60b17e6fd2dca54585/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6c697869616f7a68656e67343231312f626c6f672d696d6167657340303763393837336637336166306539373832396235323238643164373536383264323435306563322f696d672f687474702545362539432538442545352538412541312545352539392541382545372539412538342545352541452539452545372538452542302f6d7661686f3639302e706e67)

这是一段 WebSocket 握手请求：先通过 HTTP 请求升级，成功后改用 WebSocket 帧通信。socket 是通信接口，WebSocket 是协议，不是一回事

## 

## 总结

做的事情是：收到请求，找到内容，包成 HTTP 响应，再发回去。

Reactor 负责等事件、调用处理函数；业务决定返回什么。换 HTML 或图片时，尽量复用连接管理和发送逻辑。

一次可能发不完，所以要记住“现在发到哪个阶段、已经发了多少”。能发就继续，暂时发不了就等事件，发完再收尾。



希望大家能指出问题 , 谢谢!
