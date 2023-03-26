# EPOLL反应堆模型



#### 一、反应堆模型

1. epoll ET模式 + 非阻塞、忙轮询 + void *ptr
2. epoll本来流程：
   - socket、bind、listen -- epoll create 创建红黑树监听模型 -- 返回epfd -- epoll_ctl() 向树上添加一个监听fd -- epoll_wait监听 -- 对应监听fd有事件产生 -- 返回 监听满足数组。 -- 判断返回数组元素 -- lfd满足 -- Accept -- cfd 满足 -- read() -- 小到到 -- write回去
3. 反应堆的流程：不但要监听cfd的读事件，还要监听cfd的写事件
   - socket、bind、listen -- epoll create 创建红黑树监听模型 -- 返回epfd -- epoll_ctl() 向树上添加一个监听fd -- epoll_wait监听 -- 对应监听fd有事件产生 -- 返回 监听满足数组。 -- 判断返回数组元素 -- lfd满足 -- Accept -- cfd 满足 -- read() -- 小到到 -- 把cfd从监听红黑树上摘下 -- epoll_ctl监听cfd的写事件 -- EPOLLOUT -- 回调函数 -- epoll_ctl -- EPOLL_CTL_ADD 重新放到红黑树上去监听写事件 -- 等待epoll_wati返回 -- 返回说明cfd可写 -- write回去 -- cfd从红黑树上摘下来 -- EPOLLIN -- epoll_ctl() -- EPOLL_CTL_ADD 重新放到红黑上监听事件 -- epoll_wait 监听

#### 二、代码实现

1. ```c
   #include "warp.h"
   #include <sys/epoll.h>
   #include <time.h>
   #include <fcntl.h>
   
   #define SERVER_PORT 9000
   #define BUFLEN 4096
   #define MAX_EVENTS 1024 //监听数上限
   
   void recvdata(int fd, int events, void *arg);
   void senddata(int fd, int events, void *arg);
   
   /* 描述就绪文件描述符相关信息 */
   struct myevents_s
   {
   	int fd;		//要监听的文件描述符
   	int events;	//对应的监听事件
   	void *arg;	//泛型参数
   	void (*call_back)(int fd,int events,void *arg);	//回调函数
   	int status;	//是否在监听:1->在红黑树上监听，0->不在(不监听)
   	char buf[BUFLINE];	
   	int len;	
   	long last_active;	//记录每次加入红黑树g_efd的时间的值
   }
   
   int g_efd;	//全局变量，保存epoll_create返回的文件描述符
   struct myevent_s g_events[MAX_EVENTS+1];	//自定义结构体类型数组，+1 --> listen fd
   
   
   void recvdata(int fd, int events, void *arg)
   {
   	struct myevent_s *ev = (struct myevent_s *)arg;
   	int len;
   	len = recv(fd,ev->buf,sizeof(ev->buf),0);//在网络编程中recv==read,send==write
   	eventdel(g_efd,ev);//将该结点从红黑树上摘下    
   	if(len > 0 )
   	{
   		ev->len = len;
   		ev->buf[len] = '\0'; //手动添加字符串标记结束
   		printf("C[%d]:%s\n",fd,ev->buf);
   		
   		eventset(ev,fd,senddata,ev); //设置该fd对应的回调函数为senddata
   		eventadd(g_efd,EPOLLOUT,ev); //将fd加入红黑树g_efd中，监听其写事件
   	}
   	else if(len == 0)
   	{
   		close(ev->fd);
   		printf("[fd=%d] pos[%ld],closed\n",fd,ev->events);
   	}
   	else
   	{
   		close(ev->fd);
   		printf("recv[fd=%d] error[%d]:%s\n",fd,errno,strerror(errno));
   	}
   	return ;
   }
   
   void senddata(int fd,int events,void *arg)
   {
   	struct myevent_s *ev = (struct myevent_s *)arg;
   	int len;
   	len = send(fd,ev->buf,ev->len;0); 
   	eventdel(g_def,ev);
   	if(len>0)
   	{
   		printf("send[fd=%d],[%d]%s\n",fd,len,ev->buf);
   		eventset(ev,fd,recvdata,ev); //将该fd的回调函数改为recvdata
   		eventadd(g_efd,EPOLLIN,ev);	//重新添加到红黑树上，设为监听事件
   	}
   	else
   	{
   		close(ev->fd);
   		printf("send[fd=%d] error %s\n",fd,strerror(errno));
   	}
   	return;
   
   }
   
   //将结构体myevents_s成员变量初始化
   void eventset(struct myevent_s *ev,int fd, void(*call_back)(int, int, void *), void *arg)
   {
   	ev->fd = fd;
   	ev->call_back = call_back;
   	ev->events = 0;
   	ev->arg = arg;
   	ev->status = 0;
   	memset(ev->buf, 0, sizeof(ev->buf));
   	ev->len = 0;
   	ev->last_active = time(NULL);
   	return;
   }
   
   
   void eventadd(int efd,int events,struct myevent_s *ev)
   {
   	struct epoll_event epv = {0, {0}};
   	int op;
   	epv.data.ptr = ev;
   	epv.events = ev->events = events; 
   	if(ev->status == 0)
   	{
   		op = EPOLL_CTL_ADD;
   		ev->status = 1;
   	}
   	if(epoll__ctl(efd,op,ev->fd,&epv)<0)
   	{
   		printf("event add failed [fd=%d], events[%d]\n", ev->fd,events);
   	}
   	else
   	{
   		printf("event add OK [fd=%d], op=%d, events[%0X]\n",ev->fd,op,events);
   	}
   	return ;
   }
   void eventdel(int efd,struct myevent_s *ev)
   {
   	struct epoll_event epv = {0,{0}};
   	if(ev->status !=1)
   	{
   		return;
   	}
   	epv.data.prt = NULL;
   	ev->status = 0;
   	epoll_ctl(efd,EPOLL_CTL_DEL,ev->fd,&epv);
   	return;
   }
   
   //当有文件描述符就绪，epoll返回，调用该函数与客户端建立连接
   void acceptconn(int lfd,int events, void *arg)
   {
   	struct sockaddr_in cin;
   	socklen_t len = sizeof(cin);
   	int cfd,i;
   	if((cfd==accept(lfd,(struct sockaddr *)&cin),&len) == -1)
   	{
   		if(errno != EAGAIN && errno != EINTR)
   		{
   			//暂不处理
   		}
   		printf("%s:accept,%s\n",_func_,strerror(errno));
   		return;
   	}
   	do
   	{
   		for(i = 0; i < MAX_EVENTS; i++)//在全局数组中找出一个空闲元素
   		{
   			if(g_events[i].status == 0)
   			{
   				break;
   			}
   		}
   		if(i == MAX_EVENTS)
   		{
   			printf("%s:max connect limit[%d]\n",_func_,MAX_EVENTS);
   		}
   		int flag = 0;
   		if((flag == fcntl(cfd,F_SETFL,O_NONBLOCK) < 0))
   		{
   			printf("%s: fcntl nonblocking failed, %s\n",_func_,strerror(errno));
   			break;
   		}
   		//给cfd设置一个myevents_s结构体，回调函数设置为recvdata
   		eventset(&g_events[i],cfd,recvdata,&g_events[i]);
   		eventadd(g_efd,EPOLLIN,&g_events[i]);
   	}while(0);
   	printf("new connect [%s:%d][time:%ld],pos[%d]\n",
   			inet_ntoa(cin.sin_addr),ntohs(cin.sin_port),g_events[i].last_active, i);
   	return; 
   }
   
   
   void initlistensocket(int efd, short port)
   {
   	struct sockaddr_in sin;
   	int lfd = socket(AF_INET, SOCK_STREAM, 0);
   	fcntl(lfd,F_SETFL,O_NONBLOCK); //将socket设置为非阻塞
   	memset(&sin,0,sizeof(sin));
   	sin.sin_family = AF_INET;
   	sin.sin_port = htons(port);
   	sin.sin_addr.s_addr = INADDR_ANY;
    
   	bind(lfd,(struct sockaddr *)&sin, sizeof(sin));
   	
   	listen(lfd,20);
   
   	eventset(&g_events[MAX_EVENTS], lfd, acceptconn, &g_events[MAX_EVENTS]);
   
   	eventsadd(efd, EPOLLIN, &g_events[MAX_EVENTS]);
   	return;
   }
   
   
   int main(int argc, char *argv[])
   {	
   	unsigned short port = SERV_PORT;
   	if(argc == 2)
   	{
   		port = atoi(argv[1]); //使用用户指定端口，如果未指定，使用默认端口
   	}
   	g_efd = epoll_create(MAX_EVENTS+1); //创建红黑树，返回给全局g_efd
   	if(g_efd <= 0)
   	{
   		printf("create efd in %s err %s\n",_func_,strerror(errno));
   	}
   	initlistensocket(g_efd,port); //初始化监听socket
   	struct epoll_event events[MAX_EVENTS+1]; //保存已经满足就绪事件的文件描述符数组
   	printf("server running:port[%d]\n",port);
   	int checkpos = 0,i;
   	while(1)
   	{
   		//超时验证，每次测试100个连接，不测试listenfd，当客户端60秒内没有和服务器通信，则关闭此客户端的连接
   		long now = time(NULL);	//当前时间
   		for(i = 0; i < 100; i++, checkpos++)	//一次循环检测100个，使用checkpos控制检测对象
   		{
   			if(checkpos == MAX_EVENTS)
   			{
   				checkpos = 0;
   			}
   			if(g_events[checkpos].status !=1)	//不在红黑树g_efd上
   			{
   				continue;
   			}
   			long duration = now - g_events[checkpos].last_active; //客户端不活跃的时间
   			if(duration >= 60)
   			{
   				close(g_events[checkpos].fd);	//关闭与该客户端的连接
   				printf("[fd=%d] timeout\n",g_events[checkpos].fd);
   				eventdel(g_efd,&g_events[checkpos]); //将该客户端从红黑树上摘除
   			}
   		}
   		//监听红黑树g_efd，将满足的事件的文件描述符加至events数组中，1秒没有事件满足，返回0
   		int nfd = epoll_wait(g_efd,events,MAX_EVENTS+1,1000);
   		if(nfd < 0)
   		{
   			printf("epoll_wait error!\n");
   			break;
   		}
   		for(i = 0 ; i < nfd; i++)
   		{
   			//使用自定义结构体myevent_s类型指针，接受联合data的void *ptr成员
   			struct myevents_s *ev = (struct myevent_s *)events[i].data.ptr;
   
   			if((events[i].events & EPOLLIN) && (ev->events & EPOLLIN)) //读就绪事件
   			{
   				ev->call_back(ev->fd, events[i].events,ev->arg);
   			}
   			if((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT)) //写就绪事件
   			{
   				ev->call_back(ev->fd, events[i].events, ev->arg);
   			}
   		}
   	}		
   	return 0;
   }
   
   ```
   
   