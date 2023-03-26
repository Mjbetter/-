# 多路IO转接服务器-SELECT



#### 一、select函数及其相关函数

1. ```c
   int select(int nfds,fd_set *readfs,fd_set *writefds,fd_set *exceptfds,struct timeval *timeout);
   //nfds:监控的文件描述集里最大文件描述符+1，因此此参数会告诉内核检测前多少个文件描述符的状态
   //readfds:监控有读数据到达文件描述符集合，传入传出参数  --读事件
   //writefds:监控写数据到达文件描述符集合，传入传出参数  --写事件
   //exceptfds:监控异常发生到达文件描述符集合，如带外数据到达异常，传入传出参数  --异常事件
   //timeout:定时阻塞监控时间，3种情况
   		//1、NULL,永远等待下去
   		//2、设置timeval，等待固定时间
   		//3、设置timeval里时间均为0，检查描述字后立即返回0，轮询
   
   struct timevale{
   	long tv_sec; //second 秒
   	long tv_usec; //microseconds 微妙
   }
   
   void FD_CLR(int fd,fd_set *set); //把文件描述符集合里的fd清0
   int FD_ISSET(int fd,fd_set *set); //测试文件描述符集合里fd是否置1
   int FD_SET(int fd,fd_set *set); //把文件描述符集合里fd位置1
   void FD_ZERO(fd_set *set); //把文件描述符集合里所有位清0 
   ```

#### 二、思路分析

1. ```c
   lfd = socket(); //创建套接字
   bind(); //绑定地址结构
   listen(); //设置监听上限
   fd_set rset,allset; //创建all监听集合
   FD_ZERO(&allset); //将all监听集合清零
   FD_SET(lfd,&allset); //将lfd添加到r集合中
   while(1){
       rset = allset; //保存监听集合
   	ret = select(lfd+1,&rset,NULL,NULL,NULL); //监听文件描述符集合对应事件
       if(ret > 0 ){ //有监听描述符满足对应事件
           if(FD_ISSET(lfd,&rset)){ //1表明在集合，有读事件，有客户端来连接，0表明不在集合中
               cfd = accpet();
               FD_SET(cfd,&allset); //添加到监听通信描述符集合中
           }
           for(int i=lfd+1;i<=最大文件描述符;i++){
               FD_ISSET(i,&rset); //有读写事件发生
               read();
               小->大;
               write();
           }
       }
   }
   ```


#### 三、服务器代码实现

1. ```c
   #include "warp.h"
   
   #define SERVER_PORT 9000
   
   int main()
   {
   	int listenfd,connfd;
   	char buf[BUFSIZ];
   	struct sockaddr_in server_addr,client_addr;
   	socklen_t client_addr_len;
   	
   	//创建套接字，服务器
   	listenfd = Socket(AF_INET,SOCK_STREAM,0);
   	//端口复用实现
   	int opt = 1;
   	setsockopt(listenfd,SOL_SOCKET,SO_REUSEADDR,&opt,sizeof(opt));
   	//将地址结构清零
   	bzero(&server_addr,sizeof(server_addr));
   	server_addr.sin_family = AF_INET;
   	server_addr.sin_port = htons(SERVER_PORT);
   	server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
   	//绑定套接字
   	Bind(listenfd,(struct sockaddr *)&server_addr,sizeof(server_addr));
   	//监听上限
   	Listen(listenfd,128);
   
   	//开始定义集合
   	fd_set rset,allset; //定义读集合，备份集合allset
   	int ret,max_fd = 0,i,j,n;
   
   	FD_ZERO(&allset); //清空监听集合
   	FD_SET(listenfd,&allset); //将待监听fd添加到监听结合中
   	max_fd = listenfd; //最大文件描述符
   
   	while(1)
   	{
   		rset = allset; //备份
   		ret = select(max_fd+1,&rset,NULL,NULL,NULL); //使用select监听
   		if(ret < 0)
   		{
   			sys_err("select error!");
   		}
   		if(FD_ISSET(listenfd,&rset)) //listenfd满足监听的读事件
   		{
   			client_addr_len = sizeof(client_addr);
   			connfd = Accept(listenfd,(struct sockaddr *)&client_addr,&client_addr_len); //建立链接
   			FD_SET(connfd,&allset); //将新产生的fd添加到监听集合中
   			if(max_fd < connfd)
   			{
   				max_fd = connfd; //修改max_fd
   			}
   			if(ret == 1) //说明select只返回一个，并且是listenfd，那么后续代码无需执行，以提高执行速率
   			{
   				continue;
   			}
   		}
   		for(i=listenfd+1;i<=max_fd;i++) //处理满足读事件的fd
   		{	
   			if(FD_ISSET(i,&rset))//找到满足读事件的fd
   			{
   				n = read(i,buf,sizeof(buf));
   				if(n==0) //检测到客户端已经关闭连接
   				{
   					Close(i);
   					FD_CLR(i,&allset);
   				}	
   				else if(n==-1)
   				{
   					sys_err("read error!");
   				}
   
   				for(j = 0;j<n;j++)
   				{
   					buf[j] = toupper(buf[j]);
   				}
   				write(i,buf,n);
   				write(STDOUT_FILENO,buf,n);
   			}
   		}
   	}
   	Close(listenfd);
   	return 0;
   }
   
   ```

#### 四、select的优缺点

1. 缺点：
   - 监听上限受文件描述符限制，最大为1024个。
   - 检测满足条件的fd，自己添加业务逻辑提高小，提高了编码难度。 
2. 优点：
   - 跨平台，win，linux，macOS，Unix，类Unix，

​	