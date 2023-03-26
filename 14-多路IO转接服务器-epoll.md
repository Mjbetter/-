# 多路IO转接服务器-epoll



#### 一、介绍

1. epoll是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下系统CPU的利用率，因为它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合，另一点的原因就是获取事件的时候，它无需遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。
2. epoll除了提供select/poll那种IO事件的电平触发（Level Triggered）外，还提供了边沿触发（Edge Triggered），这就使得用户空间程序有可能缓存IO状态，减少epoll_wait/epoll_pwait的调用，提供应用程序效率。

#### 二、突破文件描述符限制

1. 可以使用cat 命令来查看一个进程可以打开的socket描述符上限。

   ```
   cat /proc/sys/fs/file-max
   ulimit -a 当前用户下的进程，默认打开文件描述符个数
   ```

2. 通过修改配置文件的方式修改该上限值

   ```
   sudo vi /etc/security/limits.conf
   在文件尾部写入以下配置，soft软限制，hard硬限制。
   soft nofile 65536
   hard nofile 100000
   ```

#### 三、epoll_create和epoll_ctl

1. ```c
   int epoll_create(int size);
   //size：要创建的红黑树的监听结点数量。（仅供内核参考）
   //返回值：成功：指向新创建的红黑树的根节点的fd
   //		失败：-1 error
   int epoll_ctl(int epfd,int op,int fd,struct epoll_event *event);
   //epfd：epoll_create函数的返回值
   //op：对该监听红黑树所做的操作
   	//EPOLL_CTL_ADD 添加fd到监听红黑树
   	//EPOLL_CTL_MOD 修改fd在监听红黑树上的监听事件
   	//EPOLL_CTL_DEL 将一个fd从监听红黑树上摘下（取消监听）
   //fd：待监听的fd
   //event：本质是struct epoll_event结构体地址
   //返回值：成功0，失败-1
   typedef union epoll_data{
       void *ptr; 
       int fd; //对应监听事件的fd
       uint32_t u32; //
       uint64_t u64;
   }epoll_data_t;
   
   struct epoll_event{
       uint32_t events; //EPOLLIN EPOLLOUT EPOLLERR EPOLLET
       epoll_data_t data; //联合体（共用体）
   }
   ```

#### 四、epoll_wait

1. ```c
   int epoll_wait(int epfd,struct epoll_event *events,int maxevents,int timeout);
   //epfd：epoll_create函数的返回值
   //events：传出参数，用来存内核得到的事件集合，可简单看作数组
   //maxevents：数组元素的总个数
   //timeout；-1阻塞，0不阻塞，>0超时时间（毫秒）
   //返回值：>0,满足监听的总个数，可以用作循环上限；0没有fd满足监听事件；-1失败，errno
   ```

#### 五、epoll实现多路转接IO的思路

1. ```c
   lfd = socket();
   bind();
   listen();
   
   int epfd = epoll_create(1024);
   struct epoll_event,tep,ep[1024];//tep用来设置单个fd属性，ep是epoll_wait()传出的满足监听事件的数组
   tep.events = EPOLLIN
   tep.data.fd = lfd;
   epoll_ctl(epfd,EPOLL_CTL_ADD,lfd,&tep);//将lfd添加到监听红黑树上
   while(1){
       int ret = epoll_wait(epfd,ep,1024,-1);//实时监听
       for(i = 0; i<ret; i++){
           if(ep[i].data.fd == lfd){ //lfd满足读事件，有新的客户端发起连接请求
               cfd = Accept();
               tep = EPOLLIN;
               temp.data.fd = lfd;
               epoll_ctl(epfd,EPOLL_CTL_ADD,cfd,&tep);
           }
           else{ //cfd满足读事件
               n = read();
               if(n==0){
                   close(ep[i].data.fd);
               	epoll_ctl(epfd,EPOLL_CTL_DEL,cfd,NULL); //将cfd从监听的红黑树摘下
               }
               else if( n > 0){
                   小->大；
                  	write(ep[i].data.fd,buf,n);
               }
           }
       }
   }
   
   ```

#### 六、EPOLL事件有两种模型

1. Edge Triggered(ET)	边缘触发只有数据到来才触发，不管缓冲区中是否还有数据。（0->1,1->0）
2. Level Triggered(LT)    水平触发只要有数据都会触发。（电平持续为1）
   - 客户端第一次向缓冲区中写入了aaaa\nbbbb\n，只读出了aaaa\n，此时缓冲区中还剩下bbbb\n，那么这时候会继续触发函数epoll_wait，输出剩余的数据bbbb\n，如果是ET模式，那么必须等到下一次客户端向缓冲区中写入数据，然后服务器才会从缓冲区中将bbbb\n读取出来。
3. LT和ET的区别
   - LT是确省的工作方式，并且同时支持block和no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不进行任何操作，内核还是会继续通知你的。所以，这种模式编程出错误可能性小一点，传统的socket/poll都是这种模型的代表。
   - ET是高速工作方式，只支持no-block socket，在这种模式下，当描述符从未就绪变成就绪时，内核通过epoll告诉你，然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，请注意，如果一直不对这个fd作IO操作（从而导致它再次变成未就绪），内核不会发送更多的通知（only once）。

#### 七、EPOLL实现代码

1. ```c
   #include "warp.h"
   #include <sys/epoll.h>
   
   #define SERVER_PORT 9000
   #define OPEN_MAX 5000
   #define MAXLINE 8192
   
   int main(int argc,char *argv[])
   {
   	int listenfd,connfd;
   	char buf[MAXLINE];
   	struct sockaddr_in server_addr,client_addr;
   	socklen_t client_addr_len;
   	int i,opt = 1;
   
   	listenfd = Socket(AF_INET,SOCK_STREAM,0);
   	setsockopt(listenfd,SOL_SOCKET,SO_REUSEADDR,&opt,sizeof(opt));
   	bzero(&server_addr,sizeof(server_addr));
   	server_addr.sin_port = htons(SERVER_PORT);
   	server_addr.sin_family = AF_INET;
   	server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
   	
   	Bind(listenfd,(struct sockaddr *)&server_addr,sizeof(server_addr));
   	Listen(listenfd,128);
   
   	int sockfd;
   	int num,n;
   	char str[INET_ADDRSTRLEN];
   	ssize_t nready,efd,res;
   	//tep:parameter in function "epoll_ctl";  ep[]:parameter in function "epoll_wait"
   	struct epoll_event tep,ep[OPEN_MAX];
   	
   	//create epoll model,efd point to the red and black tree root
   	efd = epoll_create(OPEN_MAX);
   	if(efd == -1)
   	{
   		sys_err("epoll_create error!");
   	}
   	//specify lfd listen for an event to read
   	tep.events = EPOLLIN;
   	tep.data.fd = listenfd;
   	//set the lfd and corresponding structure in to a tree
   	res = epoll_ctl(efd,EPOLL_CTL_ADD,listenfd,&tep);
   	if(res == -1)
   	{
   		sys_err("epoll_ctl  error!");
   	}
   	for(;;)
   	{
   		nready = epoll_wait(efd,ep,OPEN_MAX,-1);
   		if(nready == -1)
   		{
   			sys_err("epoll_wait is error!");
   		}
   		for(i = 0;i<nready;i++)
   		{
   			if(!ep[i].events & EPOLLIN)
   			{
   				continue;
   			}
   			if(ep[i].data.fd == listenfd)
   			{
   				client_addr_len = sizeof(client_addr);
   				connfd = Accept(listenfd,(struct sockaddr *)&client_addr,&client_addr_len);
   
   				printf("received from %s at PORT %d\n",
   						inet_ntop(AF_INET,&client_addr.sin_addr,str,sizeof(str)),
   						ntohs(client_addr.sin_port));
   				printf("cfd%d---client %d\n",connfd,++num);
   
   				tep.events = EPOLLIN;
   				tep.data.fd = connfd;
   				res = epoll_ctl(efd,EPOLL_CTL_ADD,connfd,&tep);
   				if(res == -1)
   				{
   					sys_err("epoll_ctl is error!");
   				}
   			}
   			else
   			{
   				sockfd = ep[i].data.fd;
   				n = read(sockfd,buf,MAXLINE);
   				if(n==0)
   				{
   					res = epoll_ctl(efd,EPOLL_CTL_DEL,sockfd,NULL);
   					if(res == -1)
   					{
   						sys_err("epoll_ctl is error!");
   					}
   					Close(sockfd);
   					printf("client[%d] closed connection\n",sockfd);
   				}
   				else if(n < 0)
   				{
   					sys_err("read n < 0 error");
   					res = epoll_ctl(efd,EPOLL_CTL_DEL,sockfd,NULL);
   					Close(sockfd);
   				}
   				else
   				{
   					for(i=0;i<n;i++)
   					{
   						buf[i] = toupper(buf[i]);
   					}
   					write(STDOUT_FILENO,buf,n);
   					write(sockfd,buf,n);
   				}
   			}
   		}
   	}
   	return 0;
   }
   
   ```

2. 非阻塞epoll实现

   ```c
   #include "warp.h"
   #include <sys/epoll.h>
   
   #define MAXLINE 10
   #define SERVER_PORT 9000
   
   int main()
   {
   	struct sockaddr_in server_addr,client_addr;
   	char buf[MAXLINE];
   	char str[INET_ADDRSTRLEN];
   	int lfd,cfd,efd,res;
   	socklen_t client_addr_len;
   	//创建服务器套接字
   	lfd = Socket(AF_INET,SOCK_STREAM,0);
   	//初始化服务器地址结构
   	server_addr.sin_family = AF_INET;
   	server_addr.sin_port = htons(SERVER_PORT);
   	server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
   	//绑定服务器地址结
   	Bind(lfd,(struct sockaddr *)&server_addr,sizeof(server_addr));
   	//上限设置
   	Listen(lfd,128);
   	
   	struct epoll_event event;
   	struct epoll_event resevent[10];
   	int len;
   	efd = epoll_create(10);
   	event.events = EPOLLIN | EPOLLET; //ET
   	//event.events = EPOLLIN;  //LT
   	printf("Accepting connections...\n");
   	
   	client_addr_len = sizeof(client_addr);
   	cfd = Accept(lfd,(struct sockaddr *)&client_addr,&client_addr_len);
   	
   	printf("received from %s at port %d\n",
   			inet_ntop(AF_INET,&client_addr.sin_addr,str,sizeof(str)),
   			ntohs(client_addr.sin_port));
       //修改cfd为非阻塞读
       flag = fcntl(cfd,F_GETFL);
       flag |= O_NONBLACK;
       fcntl(cfd,F_SETFL,flag); 
       
   	event.data.fd = cfd;
   	epoll_ctl(efd,EPOLL_CTL_ADD,cfd,&event);
   
   	while(1)
   	{
   		res = epoll_wait(efd,resevent,10,-1);
   
   		printf("res %d\n",res);
   		if(resevent[0].data.fd == cfd)
   		{
   			len = read(cfd,buf,MAXLINE/2);
   
   			write(STDOUT_FILENO,buf,len);
   		}
   	}
   	return 0;
   }
   
   ```

#### 八、epoll优缺点

1. 优点：高效。突破1024文件描述符限制。
2. 缺点：不能跨平台。