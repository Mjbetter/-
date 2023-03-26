# TCP通信流程分析



1. server:
   - socket()	--创建socket
   - bind()        --绑定服务器地址结构
   - listen()      --设置监听上限
   - accept()    --阻塞，等待客户端连接
   - read()        --读socket获取客户端数据
   - 小->大写   --toupper()
   - write(fd)
   - close()
2. client:
   - socket	--创建socket
   - conncet()     --与服务器建立连接
   - write()          --写数据到socket
   - read()           --读转换后的数据
   - 显示读取结果
   - close()