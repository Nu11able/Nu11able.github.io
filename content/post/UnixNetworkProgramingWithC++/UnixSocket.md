---
title: Unix网络编程基础篇(例子)
date: 2022-02-23
tags:
 - socket
 - c++
 - linux
categories:
 - UnixNetworkProgramingWithC++
---


# Unix网络编程基础篇(例子)

## 一个简单的echo client/server
```cpp
// server.cpp

// 单进程
#include<iostream>
#include<unistd.h>
#include<sys/socket.h>
#include<arpa/inet.h>
#include<strings.h>
#include<string>
using namespace std;

void output(const string &str, int size) {
    for (int i = 0; i < size; ++i)
        cout << str[i];
}

int main() {
    int sock_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in ser_addr, clnt_addr;
    unsigned addr_len = sizeof(struct sockaddr_in), clnt_addr_len;

    bzero(&ser_addr, sizeof(ser_addr));
    ser_addr.sin_family = AF_INET;
    ser_addr.sin_port = htons(65535);
    ser_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    bind(sock_fd, (struct sockaddr*)&ser_addr, sizeof(ser_addr));

    listen(sock_fd, 10);

    for (;;) {
        int client_fd = accept(sock_fd, (struct sockaddr*)&clnt_addr, &addr_len);
        string client_tag(100, 0);
        inet_ntop(AF_INET, &clnt_addr.sin_addr.s_addr, &client_tag[0], client_tag.capacity());
        client_tag += to_string(ntohs(clnt_addr.sin_port));
        cout << "connected from " << client_tag << endl;

        client_tag.push_back('>');
        int n;
        string recv_buf(1024, 0);
        while ((n = read(client_fd, &recv_buf[0], recv_buf.capacity()))) {
            cout << client_tag;
            output(recv_buf, n);
            cout << endl;

            n = write(client_fd, &recv_buf[0], n);
        }
        close(client_fd);
    }

    close(sock_fd);
    return 0;
}

```

`若没有特殊说明后续实现的server均可用此SimpleClient作为客户端连接`
```cpp
// SimpleClient.cpp
#include<iostream>
#include<unistd.h>
#include<arpa/inet.h>
#include<sys/socket.h>
#include<string>
#include<strings.h>

using namespace std;

void output(const string &str, int size) {
    for (int i = 0; i < size; ++i)
        cout << str[i];
}

int main() {
    int sock_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv, clnt;
    bzero(&serv, sizeof(serv));
    serv.sin_family = AF_INET,
    serv.sin_port = htons(65535);
    inet_pton(AF_INET, "127.0.0.1", &serv.sin_addr);

    connect(sock_fd, (struct sockaddr*)&serv, sizeof(serv));

    string buffer(1024, 0);
    cin >> buffer;
    int n;
    while (buffer != "exit") {
        write(sock_fd, &buffer[0], buffer.length());
        n = read(sock_fd, &buffer[0], buffer.capacity());
        cout << "server>";
        output(buffer, n);
        cout << endl;
        cin >> buffer;
    }
    close(sock_fd);

    return 0;
}
```

## 一个简单的多进程echo服务器
`注意点：使用fork创建子进程后要在主进程中要记得调用wait等函来等待子进程`
> 当子进程走完了自己的生命周期后,它会执行exit()系统调用,内核释放该进程所有的资源,包括打开的文件,
> 占用的内存等。但是仍然为其保留一定的信息(包括进程号the process ID,退出码exit code,退出状态the terminationstatus of the process,
> 运行时间the amount of CPU time taken by the process等),这些数据会一直保留到系统将它传递给它的父进程为止,
> 直到父进程通过wait / waitpid来取时才释放。

```cpp
#include<iostream>
#include<unistd.h> 
#include<sys/socket.h>
#include<arpa/inet.h>
#include<strings.h>
#include<sys/wait.h>
#include<string>
#include<algorithm>
#include<vector>
using namespace std;

void output(const string &str, int size) {
    for (int i = 0; i < size; ++i)
        cout << str[i];
}

int main() {
    int sock_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in ser_addr, clnt_addr;
    unsigned addr_len = sizeof(struct sockaddr_in), clnt_addr_len;

    bzero(&ser_addr, sizeof(ser_addr));
    ser_addr.sin_family = AF_INET;
    ser_addr.sin_port = htons(65535);
    ser_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    bind(sock_fd, (struct sockaddr*)&ser_addr, sizeof(ser_addr));

    listen(sock_fd, 10);

    vector<int> children;
    for (;;) {
        int client_fd = accept(sock_fd, (struct sockaddr*)&clnt_addr, &addr_len);
        string client_tag(100, 0);
        inet_ntop(AF_INET, &clnt_addr.sin_addr.s_addr, &client_tag[0], client_tag.capacity());
        client_tag += to_string(ntohs(clnt_addr.sin_port));
        cout << "connected from " << client_tag << endl;

        int pid = fork();
        if (pid == 0) {
            client_tag.push_back('>');
            int n;
            string recv_buf(1024, 0);
            while ((n = read(client_fd, &recv_buf[0], recv_buf.capacity()))) {
                cout << client_tag;
                output(recv_buf, n);
                cout << endl;

                n = write(client_fd, &recv_buf[0], n);
            }
            close(client_fd);
            exit(0);
        }
        children.push_back(pid);

        cout << client_tag << " is working on child process " << pid << endl;
        close(client_fd);
    }
    close(sock_fd);

    while (!children.empty()) {
        int status;
        int pid = wait(&status);
        
        children.erase(find(children.begin(), children.end(), pid));
        if (WIFEXITED(status)) {
            cout << "child " << pid << " terminated normally" << endl;
        } else {
            cout << "child " << pid << " terminated with " << WEXITSTATUS(status) << endl;
        }
    }

    cout << "server exit" << endl;
    return 0;
}
```

## 基于select的非阻塞echo服务器
上述的accept函数以及read函数都将阻塞我们的进程，为了同时处理多个请求我们不得不创建多个进程。我们无法知道套接字上的读写事件何时发生，而**select**可以像一个像一个闹钟一样通知我们有我们关注的事件发生

**int select(int nfds, fd_set \*readfds, fd_set \*writefds,fd_set \*exceptfds, struct timeval \*timeout);**
- void FD_CLR(int fd, fd_set *set);
- int  FD_ISSET(int fd, fd_set *set);
- void FD_SET(int fd, fd_set *set);
- void FD_ZERO(fd_set *set);

    `select限制：`select使用fd_set数据类型作为描述符集，通常是一个数组，它的大小由FD_SETSIZE常值定义，通常为1024(即select最多能够同时监听的同一类型的最大数量为FD_SETSIZE)

**ssize_t recv(int sockfd, void \*buf, size_t len, int flags);**
- flags取值为MSG_DONTWAIT时可以实现非阻塞通信
```cpp
#include<iostream>
#include<unistd.h>
#include<sys/socket.h>
#include<sys/wait.h>
#include<sys/select.h>
#include<strings.h>
#include<string.h>
#include<arpa/inet.h>
#include<string>
#include<algorithm>
#include<map>
#include<vector>
using namespace std;

void output(const string &str, int size) {
    for (int i = 0; i < size; ++i)
        cout << str[i];
}

int main() {
    int sock_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in ser_addr, clnt_addr;
    unsigned addr_len = sizeof(struct sockaddr_in), clnt_addr_len;

    bzero(&ser_addr, sizeof(ser_addr));
    ser_addr.sin_family = AF_INET;
    ser_addr.sin_port = htons(65535);
    ser_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    bind(sock_fd, (struct sockaddr*)&ser_addr, sizeof(ser_addr));

    listen(sock_fd, 10);

    fd_set allset, rset;
    map<int, string> client_names;
    int allfd[FD_SETSIZE];
    for (int i = 0; i < FD_SETSIZE; i++) 
        allfd[i] = -1;
    FD_ZERO(&allset);
    FD_SET(sock_fd, &allset);
    int maxfd = sock_fd;

    for (;;) {
        rset = allset;
        int ready = select(maxfd + 1, &rset, NULL, NULL, NULL);

        if (FD_ISSET(sock_fd, &rset)) {
            int client_fd = accept(sock_fd, (struct sockaddr*)&clnt_addr, &addr_len);
            string client_tag(100, 0);
            inet_ntop(AF_INET, &clnt_addr.sin_addr.s_addr, &client_tag[0], client_tag.capacity());
            client_tag.push_back(':');
            client_tag += to_string(ntohs(clnt_addr.sin_port));
            cout << "connected from " << client_tag << endl;

            bool unset = true;
            for (int i = 0; i < FD_SETSIZE; ++i) {
                if (allfd[i] > 0) {
                    cout << allfd[i] << " ";
                    continue;
                };
                allfd[i] = client_fd;
                maxfd = max(maxfd, client_fd);
                unset = false;
                client_names[client_fd] = client_tag;
                FD_SET(client_fd, &allset);
                break;
            }
            if (unset) {
                cout << "too many clients" << endl;
                close(client_fd);
            }
            ready--;
        }

        int n;
        string recv_buf(1024, 0);
        for (int i = 0; i < FD_SETSIZE && ready; ++i) {
            if (allfd[i] < 0 || !FD_ISSET(allfd[i], &rset)) continue;

            while ((n = recv(allfd[i], &recv_buf[0], recv_buf.capacity(), MSG_DONTWAIT)) > 0) {
                cout << client_names[allfd[i]] << ">";
                output(recv_buf, n);
                cout << endl;
                n = write(allfd[i], &recv_buf[0], n);  
            }
            if (n == 0 || errno != EAGAIN) {
                if (n == 0) 
                    cout << client_names[allfd[i]] << " closed" << endl;
                else
                    cout << "error: " << strerror(errno) << endl;
                FD_CLR(allfd[i], &allset);
                close(allfd[i]);
                allfd[i] = -1;
            }
            ready--;
        }
    }
    close(sock_fd);

    cout << "server exit" << endl;
    return 0;
}
```

## 基于poll的非阻塞echo服务器
> 回顾6.3结尾处我们就FD_SETSIZE以及就每个描述符集中最大描述符数目相比每个进
> 程中最大描述符数目展开了讨论。有了poll就不再由那样的问题了，因为分配一个
> pollfd结构的数组并把数组中元素的数目通知内核成了调用者的责任。内核不再需要
> 知道类似fd_set的固定大小的数据类型。

```cpp
#include<sys/socket.h>
#include<arpa/inet.h>
#include<poll.h>
#include<signal.h>
#include<unistd.h>
#include<strings.h>
#include<string.h>
#include<iostream>
#include<string>
#include<vector>
#include<algorithm>


const short PORT = 65535;
const int LISTENQ = 10;
const int BUFFER_SIZE = 1024;
bool exiting = false;

using namespace std;

void output(const string &str, int size) {
    for (int i = 0; i < size; ++i)
        cout << str[i];
}

void exit_handler(int signum) {
    exiting = true;
}

int main() {
    signal(SIGINT, exit_handler);

    int serv_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr, clnt_addr;
    int clnt_fd;
    unsigned addr_len = sizeof(struct sockaddr_in);

    bzero(&serv_addr, addr_len);
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(serv_fd, (struct sockaddr*)&serv_addr, addr_len) < 0) {
        perror("bind");
        exit(0);
    }
    listen(serv_fd, LISTENQ);

    vector<struct pollfd> events;
    vector<string> names;
    struct pollfd poll_temp;
    poll_temp.fd = serv_fd;
    poll_temp.events = POLLRDNORM;
    events.push_back(poll_temp);
    names.push_back("server");

    for (;;) {
        // Specifying a negative value in timeout  means  an  infinite timeout.  Specifying a timeout of zero causes poll() to return 
        // immediately, even if no file descriptors are ready.
        int nready = poll(&events[0], events.size(), -1); 

        if (events[0].revents & POLLRDNORM) { // new client connection
            clnt_fd = accept(serv_fd, (struct sockaddr*)&clnt_addr, &addr_len);
            auto iter = find_if(events.begin(), events.end(), [](auto &iter){ return iter.fd == -1; });
            if (iter != events.end()) {
                iter->fd = clnt_fd;
                iter->events = POLLRDNORM;
            } else {
               events.push_back(poll_temp);
               iter = events.end() - 1;
               names.push_back(string(""));
            }
            iter->fd = clnt_fd;
            iter->events = POLLRDNORM;
            string name(30, 0);
            inet_ntop(AF_INET, &clnt_addr.sin_addr, &name[0], name.capacity());
            name = string(name.c_str());
            name.push_back(':');
            name.append(to_string(ntohs(clnt_addr.sin_port)));
            names[distance(events.begin(), iter)] = name;

            cout << "client " << name << " connect" << endl;
            nready--;
        }

        for (int i = 1; i < events.size() & nready; ++i) {
            if (events[i].fd < 0) continue;

            if (events[i].revents & (POLLRDNORM | POLLERR)) {
                string buffer(BUFFER_SIZE, 0);
                int n;
                while ((n = recv(events[i].fd, &buffer[0], buffer.capacity(), MSG_DONTWAIT)) > 0) {
                    cout << names[i] << ">";
                    output(buffer, n);
                    cout << endl;
                    write(events[i].fd, &buffer[0], n);
                }
                if (n == 0 || errno != EAGAIN) {
                    cout << "connection of " << names[i]; 
                    if (n == 0) 
                        cout << " was closed" << endl;
                    else 
                        cout << " something error: " << strerror(errno) << endl;
                    close(events[i].fd);
                    events[i].fd = -1;
                }
            }
        }

        if(exiting) break;

    }

    cout << "\nserver exiting" << endl;
    for_each(events.begin(), events.end(), [](auto &mem) { if (mem.fd != -1) close(mem.fd); });

    return 0;
}
```

## 基于epoll的非阻塞echo服务器
```cpp
#include<sys/socket.h>
#include<sys/epoll.h>
#include<arpa/inet.h>
#include<signal.h>
#include<unistd.h>
#include<strings.h>
#include<string.h>
#include<iostream>
#include<string>
#include<vector>
#include<map>
#include<algorithm>


const short PORT = 65535;
const int LISTENQ = 10;
const int BUFFER_SIZE = 1024;
bool exiting = false;

using namespace std;

void output(const string &str, int size) {
    for (int i = 0; i < size; ++i)
        cout << str[i];
}

void exit_handler(int signum) {
    exiting = true;
}

int main() {
    signal(SIGINT, exit_handler);

    int serv_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr, clnt_addr;
    int clnt_fd;
    unsigned addr_len = sizeof(struct sockaddr_in);

    bzero(&serv_addr, addr_len);
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(serv_fd, (struct sockaddr*)&serv_addr, addr_len) < 0) {
        perror("bind");
        exit(0);
    }
    listen(serv_fd, LISTENQ);
    cout << "server running..." << endl;

    int epoll_fd = epoll_create(LISTENQ);
    struct epoll_event ev;
    vector<struct epoll_event> events(LISTENQ);
    vector<int> opened_sockets;
    map<int, string> names;
    ev.data.fd = serv_fd;
    ev.events = EPOLLIN;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, serv_fd, &ev);
    names[serv_fd] = string("server");
    opened_sockets.push_back(serv_fd);


    string buffer(BUFFER_SIZE, 0);
    for (;;) {
        // Specifying a timeout of -1 causes epoll_wait() to block indefinitely, while specifying a timeout equal to zero cause epoll_wait() to return  imme‐     │kevin@kevin-virtual-machine:~/MINE/tingx/build$ ./client 
        // diately, even if no events are available.
        int nready = epoll_wait(epoll_fd, &events[0], LISTENQ, 100); // timeout = 100ms 

        for (int i = 0; i < nready; ++i) {
            // new client connection
            if (events[i].data.fd == serv_fd) {
                clnt_fd = accept(serv_fd, (struct sockaddr*)&clnt_addr, &addr_len);
                ev.data.fd = clnt_fd;
                ev.events = EPOLLIN;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, clnt_fd, &ev);
                string name(30, 0);
                inet_ntop(AF_INET, &clnt_addr.sin_addr, &name[0], name.capacity());
                name = string(name.c_str());
                name.push_back(':');
                name.append(to_string(ntohs(clnt_addr.sin_port)));
                names[clnt_fd] = name;
                opened_sockets.push_back(clnt_fd);
                cout << "client " << name << " connected" << endl;
            }
            else {
                int n;
                while ((n = recv(events[i].data.fd, &buffer[0], buffer.capacity(), MSG_DONTWAIT)) > 0) {
                    cout << names[events[i].data.fd] << ">";
                    output(buffer, n);
                    cout << endl;
                    write(events[i].data.fd, &buffer[0], n);
                }
                if (n == 0 || errno != EAGAIN) {
                    cout << "connection of " << names[events[i].data.fd]; 
                    if (n == 0) 
                        cout << " was closed" << endl;
                    else 
                        cout << " something error: " << strerror(errno) << endl;
                    close(events[i].data.fd);
                    opened_sockets.erase(find(opened_sockets.begin(), opened_sockets.end(), events[i].data.fd));
                    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                }
            }
        }

        if(exiting) break;

    }

    cout << "\nserver exiting" << endl;
    for_each(opened_sockets.begin(), opened_sockets.end(), [](auto &mem) { close(mem); });

    return 0;
}
```

## 参考资料
[Linux中的defunct进程(僵尸进程)](https://blog.csdn.net/weixin_30919571/article/details/95053976)

[Epoll原理解析](https://blog.csdn.net/armlinuxww/article/details/92803381)

《UNIX网络编程 卷1：套接字联网API》
