---
layout: post
title: "[Linux] UDP Socket Reuse Address "
description: ""
category: 
tags: []
---
{% include JB/setup %}

最近因為工作需要，在Linux下需要讓多個UDP socket使用同一個port。本文將透過實驗來觀察多個UDP socket使用同一個port時會發生什麼事。


<!--more-->
本文的程式原始碼可在以下網址取得  
<https://github.com/huangtw/UDPReuseAddrTest>

###SO_REUSEADDR

在Linux下如果多個socket要bind同一個port，可以用setsocket設定SO_REUSERADDR來達成，而在TCP socket與UDP socket上各有不同的效果。

###TCP
在尚未設定SO_REUSERADDR的情況下，當一個TCP socket bind了0.0.0.0:PortA後，則其他Tsocket將無法在bind PortA，因為0.0.0.0代表任何一個interface的IP，因此socket不管綁定哪個IP的PortA都會造成衝突。

此時如果一個socket設定了SO_REUSERADDR，則可以成功bind指定IP的PortA，但是還是不能bind 0.0.0.0:PortA，也就是說在設定了SO_REUSERADDR之後，只有IP一模一樣才會視為衝突，0.0.0.0與特定IP不再視為衝突。

不過有一個例外，當一個TCP socket bind了0.0.0.0:PortA，並且開始listen之後，此時其他TCP socket即使設定了SO_REUSERADDR也無法bind PortA，就算指定特定的IP也一樣。

除了共用port之外，另外一個常見使用情境是，當一個bind了某個port的TCP socket close之後，其他socket並無法馬上bind同一個port，因為TCP socket還處於TIME_WAIE的狀態，尚未真的關掉。此時新的socket如果設定SO_REUSERADDR，則可以成功bind同一個port。

###UDP
在UDP與TCP不太一樣，只要每個UDP socket都有設定SO_REUSERADDR，則可以綁定一模一樣的IP跟port。在boardcast或者multicast模式下，如果多個UDP綁定同一個IP與port，當有UDP封包送到該IP:port時，每一個socket都會收到一份相同的封包。

如果是unicast呢？網路上似乎很少提到這個狀況，因此我做了一個實驗來確定這件事。

###實驗
本實驗需要兩個程式，一邊為傳送端(udp_send)，一邊為接收端(udp_receive)，傳送端每隔固定時間會從IP_A:Port_A向IP_B:Port_B發送UDP封包，而接收端會產生多個UDP socket bind同時bind IP_B:Port_B，觀察哪一個socket能收到封包。

除此之外，接收端的部分UDP socket為connected(指定destination為IP_A:Port_A，只會收到來自IP_A:Port_A的封包，封包也只能傳送給IP_A:Port_A)，來看這種情況下一般UDP socket與connected UDP socket的差異。

- **udp_send.c**
{% highlight c %}
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <netdb.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>

int main(int argc, char* argv[])
{
    int fd;
    int localPort, peerPort;
    char *peerIP;
    struct sockaddr_in local_addr, peer_addr;
    char buffer[1024];

    if (4 != argc)
    {
        exit(1);
    }

    localPort = atoi(argv[1]);
    peerIP = argv[2];
    peerPort = atoi(argv[3]);

    if ((fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
    {
        printf("socket open failed!\n");
        exit(1);
    }

    bzero((char *) &local_addr, sizeof(local_addr));
    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = ntohl(INADDR_ANY);
    local_addr.sin_port = htons(localPort);

    if (bind(fd, (struct sockaddr *)&local_addr, sizeof(local_addr)) < 0)
    {
        printf("bind failed\n");
    }

    bzero((char *) &peer_addr, sizeof(peer_addr));
    peer_addr.sin_family = AF_INET;
    peer_addr.sin_addr.s_addr = inet_addr(peerIP);
    peer_addr.sin_port = htons(peerPort);

    if (connect(fd , (struct sockaddr *)&peer_addr , sizeof(peer_addr)) < 0)
    {
        printf("udp connect failed!\n");
        exit(1);
    }

    int i = 0;
    while(1)
    {
        int size,ss;
        int ssize = 0;

        size = sprintf(buffer, "%d\n", i++);
        sleep(2);
        while (ssize < size)
        {
            if ((ss = send(fd, buffer + ssize, size - ssize, 0)) < 0)
            {
                //cout << "send error" << endl;
                printf("send error\n");
                exit(1);
            }

            ssize += ss;
        }
    }

    return 0;
}
{% endhighlight %}

- **udp_receive.c**
{% highlight c %}
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <netdb.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int createSocket(int port)
{
    int fd;
    struct sockaddr_in local_addr;

    if ((fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        printf("socket open failed!\n");
        exit(1);
    }

    int sock_opt = 1;
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (void*)&sock_opt, sizeof(sock_opt)) < 0) {
        printf("setsockopt failed!\n");
        exit(1);
    }

    bzero((char *) &local_addr, sizeof(local_addr));
    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = ntohl(INADDR_ANY);
    local_addr.sin_port = htons(port);

    if (bind(fd, (struct sockaddr *)&local_addr, sizeof(local_addr)) < 0)
    {
        printf("bind failed\n");
    }

    return fd;
}

void* worker(void* pfd)
{
    int fd = *(int*)pfd;
    char buffer[1024];

    while (1)
    {
        if (recv(fd, buffer, sizeof(buffer) - 1, 0) < 0)
        {
            printf("udp recv failed!\n");
            exit(1);
        }
        printf("socket %d received: %s\n", fd, buffer);
        //cout << "socket " << fd << " received:" << buffer << endl;
    }
}

int main(int argc, char* argv[])
{
    int *pfd, localPort, peerPort;
    char *peerIP;
    char buffer[1024];
    struct sockaddr_in local_addr, peer_addr;

    if (4 != argc)
    {
        exit(1);
    }

    localPort = atoi(argv[1]);
    peerIP = argv[2];
    peerPort = atoi(argv[3]);

    int i = 0;
    while (1)
    {
        pfd = malloc(sizeof(int));
        *pfd = createSocket(localPort);
        printf("socket %d opened\n", *pfd);

        if ((i >= 3) && (i < 6))
        {
            bzero((char *) &peer_addr, sizeof(peer_addr));
            peer_addr.sin_family = AF_INET;
            peer_addr.sin_addr.s_addr = inet_addr(peerIP);
            peer_addr.sin_port = htons(peerPort);

            if (connect(*pfd , (struct sockaddr *)&peer_addr , sizeof(peer_addr)) < 0)
            {
                printf("udp connect failed!\n");
                exit(1);
            }
        }

        pthread_t workerThread;
        pthread_create(&workerThread, NULL, worker, pfd);
        pthread_detach(workerThread);
        getchar();
        i++;
    }

    return 0;
}
{% endhighlight%}

在udp_receive中，每按一次按鍵，就會產生bind同一組IP與port的socket，而第4~6個socket還會connect到傳送端的IP跟port，引此能比較出一般UDP socket與connetec UDP的差別

- **執行結果**   
<img src="/images/linux_udp_socket_reuse_address/screenshot.png"/> 

上圖為接收端的執行畫面，接收端的UDP socket會bind 127.0.0.1:5555，傳送端則綁定127.0.0.1:6666。

可以看到前六個socket依序產生的過程中，只有最後一個bind的socket才會收到封包，但是第七個socket(socket 9)，最後產生
卻收不到封包，仍然是第六個socket(socket 8)收到封包。

從這個結果可以知道，如果多個socket綁定同樣的IP跟port，connect到封包來源IP:port的connected UDP socket可以優先接收，如果有多個socket滿足這個條件，則最晚bind的connected UDP socket可以收到封包。如果沒有connected UDP socket，則最晚bind的一般UDP socket可以收到封包。
