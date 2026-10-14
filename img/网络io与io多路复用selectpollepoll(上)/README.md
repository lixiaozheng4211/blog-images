# 网络IO与IO多路复用基础



![](https://cdn.jsdelivr.net/gh/lixiaozheng4211/blog-images@c74a83eb304b002b8a8c57861182ebd2d106a9f4/img/%E7%BD%91%E7%BB%9Cio%E4%B8%8Eio%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8selectpollepoll%28%E4%B8%8A%29/mv7t7ium.png)

网络IO 

作用 : 解决数据的发送与接受

### 总目标 : 实现一个TCPserver有多个client连接并发的传输信息

## phase 1 一个server连接一个client

用到的工具 : 

![](https://cdn.jsdelivr.net/gh/lixiaozheng4211/blog-images@c74a83eb304b002b8a8c57861182ebd2d106a9f4/img/%E7%BD%91%E7%BB%9Cio%E4%B8%8Eio%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8selectpollepoll%28%E4%B8%8A%29/mv7t7lyb.png)

基础语法 

实现一个简单server

```c
  // 创建 net IO
  int sockfd = socket(AF_INET, SOCK_STREAM, 0);

  // 服务器地址
  struct sockaddr_in server_addr = {0};
  server_addr.sin_family = AF_INET;
  server_addr.sin_port = htons(2000);
  server_addr.sin_addr.s_addr = htonl(INADDR_ANY); // 监听所有网卡
  socklen_t serv_addr_len = sizeof(struct sockaddr_in);
  if (bind(sockfd, (struct sockaddr *)&server_addr, serv_addr_len) < 0)
  {
    return -1;
  }

  if (listen(sockfd, 5) < 0)
  {
    return -2;
  }
```

这里如果爆出错误这段端口被绑定了 , 换一个端口就可以 , 从 0 ~ 1023 是系统默认端口 , 所以选择一个1024及以上的就可以

实现一个简单clientfd 也就是 client端IO

```c
  struct sockaddr_in client_addr;
  socklen_t client_addr_len = sizeof(struct sockaddr_in);
  printf("wait accept \n");
  int client_fd = accept(sockfd, (struct sockaddr *)&client_addr, &client_addr_len);

  printf("accept finish\n");
  char buffer[BUFFER_SIZE];
  recv(client_fd, buffer, BUFFER_SIZE, 0);
  printf("recv : %s",buffer);
```

我们在这里可以使用 sudo netstat -antp | grep 2000 查看

![](https://cdn.jsdelivr.net/gh/lixiaozheng4211/blog-images@c74a83eb304b002b8a8c57861182ebd2d106a9f4/img/%E7%BD%91%E7%BB%9Cio%E4%B8%8Eio%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8selectpollepoll%28%E4%B8%8A%29/mv7t7nhz.png)

多加几个连接

![](https://cdn.jsdelivr.net/gh/lixiaozheng4211/blog-images@c74a83eb304b002b8a8c57861182ebd2d106a9f4/img/%E7%BD%91%E7%BB%9Cio%E4%B8%8Eio%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8selectpollepoll%28%E4%B8%8A%29/mv7t7qch.png)

从这里的  LISTEN CLOSE_WAIT ESTABLISHED 可以看出 ,：一个 listenfd 可以持续 accept，生成很多个独立 connfd，每个 connfd 对应一个客户端 TCP 连接 , 但是发送接收消息只能与建立连接的来做

如何解决? 

我们每来一个新的accept都可以创建一个新的线程来处理

基础语法 

```c
  while (1)
  {
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(struct sockaddr_in);
    printf("wait accept \n");
    int *p_connfd = malloc(sizeof(int)); // 分配内存保存fd，解决竞态
    *p_connfd = accept(sockfd, (struct sockaddr *)&client_addr, &client_addr_len);
    printf("accept finish \n");
    pthread_t *threadid = (pthread_t *)malloc(sizeof(pthread_t *));
    pthread_create(threadid, NULL, client_recv, p_connfd);
  }
```

还有回调函数 Callback

```c
void *client_recv(void *arg)
{
  int p_connfd = *(int *)arg;
  while (1)
  {
    char buffer[BUFFER_SIZE];
    memset(buffer, 0, BUFFER_SIZE);
    int count = recv(p_connfd, buffer, BUFFER_SIZE, 0);
    printf("recv : %s \n", buffer);
    send(p_connfd, buffer, count, 0);
    printf("send finish \n");
  }
}
```

我们再一次用三个客户端连接于是发现

![](https://cdn.jsdelivr.net/gh/lixiaozheng4211/blog-images@c74a83eb304b002b8a8c57861182ebd2d106a9f4/img/%E7%BD%91%E7%BB%9Cio%E4%B8%8Eio%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8selectpollepoll%28%E4%B8%8A%29/mv7t7rw2.png)

用 netstat命令也可以看到

![](https://cdn.jsdelivr.net/gh/lixiaozheng4211/blog-images@c74a83eb304b002b8a8c57861182ebd2d106a9f4/img/%E7%BD%91%E7%BB%9Cio%E4%B8%8Eio%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8selectpollepoll%28%E4%B8%8A%29/mv7t7tmd.png)

这里都变成了 ESTABLISHED 

到此我们可以总结出这几条结论

也就是 :

1. 同一个端口只能被绑定一次 , 当然还有别的机制可以解决这个问题

2. listenfd只要进入listen就可以产生新的连接状态

3. fd 与 TCP 连接信息一一对应 , 有几个client连接就有几个 fd 句柄

到这里就实现了一请求一线程

优点 : 简单

缺点 : 就算没有数据也需要创建一个持续等待的线程

所以存在一个问题 : 为什么要为每个暂时没有数据的 Socket 分配一个专门等待的线程？



socket 就是一个网络通信接口 , 也就是如果要使用这个网络通信能力就接上这个接口就可以了
