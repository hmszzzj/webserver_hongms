# 套接字TCP通信实现
## 头文件
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <arpa/inet.h>
## 用到的函数
### socket()函数
    int socket(int domain, int type, int protocol);

    - 功能 ： 创建一个套接字
    - 参数 ： 
      - domain ： 协议族
        -  AF_INET ：ipv4
        -  AF_INT6 ：ipv6
        -  AF_UNIX, AF_LOCAL ： 本地套接字通信（进程间通信）
      - type
        - SOCK_STREAM : 流式协议
        - SOCK_DGRAM : 报式协议
      - protocol : 具体的一个协议。一般写 0
        - OCK_STREAM : 流式协议默认使用 tcp
        - SOCK_DGRAM : 报式协议默认使用 udp
    - 返回值 ： 文件描述符
      - fd ： 成功
      - -1 ： 失败

### bind()函数
    int bind(int sockfd, const struct sockaddr * addr, socklen_t addrlen);
    - 功能 ： 绑定，将 fd 和本地 IP + PORT 进行绑定
    - 参数 ：
      - sockfd : 通过socket()函数得到的文件描述符
      - addr : 需要绑定的服务器端的socket地址，封装了IP地址和PORT信息
      - addrlen : 参数结构体addr所占的内存大小
    - 返回值 ：
      - 0 ： 成功
      - -1 ： 失败
### listen()函数
    int listen(int sockfd, int backlog);
    - 参数 ：
      - sockfd ： 通过socket()函数得到的文件描述符
      - backlog : 未连接和已连接的连接数的最大值（listen()函数会创建两个队列：未连接、已连接）
    - 返回值 ：
      - 0 ： 成功
      - -1 ： 失败
### accept()函数
    服务器端调用的函数
    int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
    - 功能 ： 接收客户端连接，默认是阻塞函数
    - 参数 ：
      - sockfd ： 通过socket()函数得到的文件描述符
      - addr ： 传出参数，记录连接成功后客户端的地址信息，封装了IP地址和PORT信息
      - addrlen ： 传出参数，指定第 2 个参数的结构体的内存大小
    - 返回值 ：
      - fd ： 成功，用于通信的文件描述符
      - 0 ： 失败
### connect()函数
    客户端调用的函数
    int connect(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
    - 功能 ： 客户端连接服务器
    - 参数 ： 
      - sockfd : 用于通信的文件描述符
      - addr ： 客户端要连接服务器的地址信息
      - addrlen ： 第二个参数的内存大小
    - 返回值 ： 
      - 0 ： 成功
      - -1 ： 失败
### write()函数
    int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
    - 功能 ：写数据
    - 参数 ：
      - sockfd ： 写数据的目标文件描述符
      - addr : 要写入的数据的地址
      - addrlen ： 要写入数据的长度
    - 返回值 ：
      - 0 ： 没有数据写入
      - > 0 : 写入数据的长度
      - -1 ： 失败
### read()函数
    ssize_t read(int fd, void *buf, size_t count);
    - 功能 ： 读取数据
    - 参数 ：
      - fd ： 读取数据的目标文件描述符
      - buf ： 数据被读取后存入的地址
      - count ： 要读取数据的长度
    - 返回值 ：
      - > 0 : 成功
      - -1 ： 失败

## 整体代码
### server
~~~c
int main() {

    // 1.创建socket(用于监听的套接字)
    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    if(lfd == -1) {
        perror("socket");
        exit(-1);
    }

    // 2.绑定
    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    // inet_pton(AF_INET, "192.168.193.128", saddr.sin_addr.s_addr);
    saddr.sin_addr.s_addr = INADDR_ANY;  // 0.0.0.0
    saddr.sin_port = htons(9999);
    int ret = bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));

    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 3.监听
    ret = listen(lfd, 8);
    if(ret == -1) {
        perror("listen");
        exit(-1);
    }

    // 4.接收客户端连接
    struct sockaddr_in clientaddr;
    int len = sizeof(clientaddr);
    int cfd = accept(lfd, (struct sockaddr *)&clientaddr, &len);
    
    if(cfd == -1) {
        perror("accept");
        exit(-1);
    }

    // 输出客户端的信息
    char clientIP[16];
    inet_ntop(AF_INET, &clientaddr.sin_addr.s_addr, clientIP, sizeof(clientIP));
    unsigned short clientPort = ntohs(clientaddr.sin_port);
    printf("client ip is %s, port is %d\n", clientIP, clientPort);

    // 5.通信
    char recvBuf[1024] = {0};
    while(1) {
        
        // 获取客户端的数据
        int num = read(cfd, recvBuf, sizeof(recvBuf));
        if(num == -1) {
            perror("read");
            exit(-1);
        } else if(num > 0) {
            printf("recv client data : %s\n", recvBuf);
        } else if(num == 0) {
            // 表示客户端断开连接
            printf("clinet closed...");
            break;
        }

        char * data = "hello,i am server";
        // 给客户端发送数据
        write(cfd, data, strlen(data));
    }
   
    // 关闭文件描述符
    close(cfd);
    close(lfd);

    return 0;
}
~~~
### client
~~~c
int main() {

    // 1.创建套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }

    // 2.连接服务器端
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    inet_pton(AF_INET, "192.168.193.128", &serveraddr.sin_addr.s_addr);
    serveraddr.sin_port = htons(9999);
    int ret = connect(fd, (struct sockaddr *)&serveraddr, sizeof(serveraddr));

    if(ret == -1) {
        perror("connect");
        exit(-1);
    }

    
    // 3. 通信
    char recvBuf[1024] = {0};
    while(1) {

        char * data = "hello,i am client";
        // 给客户端发送数据
        write(fd, data , strlen(data));

        sleep(1);
        
        int len = read(fd, recvBuf, sizeof(recvBuf));
        if(len == -1) {
            perror("read");
            exit(-1);
        } else if(len > 0) {
            printf("recv server data : %s\n", recvBuf);
        } else if(len == 0) {
            // 表示服务器端断开连接
            printf("server closed...");
            break;
        }

    }

    // 关闭连接
    close(fd);

    return 0;
}
~~~
