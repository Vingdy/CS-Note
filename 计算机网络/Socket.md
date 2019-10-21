# Socket

socket=addr+port

例子：192.166.44.23:80

## 建立过程

C：socket()->connect()->TCP三步->send()/recv()

S：socket()->bind()->listen()->TCP三步->accept()->send/recv()



| 函数          | 含义                         |
| ------------- | ---------------------------- |
| socket()      | 创建套接字                   |
| connect()     | 初始化套接字（主动）         |
| bind()        | 绑定套接字（解析锁定）       |
| listen()      | 保持监控监听套接字           |
| accept()      | TCP三步后生成连接套接字      |
| send()/recv() | 连接套接字（buffer收发数据） |

正常情况下一个addr+port只能被一个套接字绑定