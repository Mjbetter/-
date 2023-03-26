# 多路IO转接服务器-POLL



#### 一、poll函数

1. ```c
   #include <poll.h>
   
   int poll(struct pollfd *fds,nfds_t nfds,int timeout);
   //fds:要监听的文件描述符数组
   //nfds:描述监听数组的实际有效监听个数
   //timeout:超时时长，单位是毫秒
   		//-1:阻塞等待
   		//0:立即返回，不阻塞进程
   		//>0:等待指定毫秒数，如当前系统时间精度不够毫秒，向上取值
   //返回值：返回满足对应监听事件的文件描述符总个数
   
   struct pollfd{
       int fd; //要监听的文件描述符
       short events; //待监听的文件描述符所对应的监听事件，POLLIN,POLLOUT,POLLERR
       short revents; //传入时，给0。如果满足对应事件的话，返回非0 --> POLLIN,POLLOUT,POLLERR
   }
   ```

2. 优缺点

   - 优点：自带数组结构，可以将监听事件集合和返回事件集合分离。可以拓展监听上限，超出1024限制。
   - 不能跨平台。无法直接定位满足监听事件的文件描述，编码难度较大。

​	