# Cookie/Session/Token

  [https://huangwang.github.io/2018/07/08/Cookie-Session%E5%92%8CToken%E4%BC%9A%E8%AF%9D%E7%9F%A5%E8%AF%86%E6%95%B4%E7%90%86/](https://huangwang.github.io/2018/07/08/Cookie-Session和Token会话知识整理/) 

## 区别

|        | cookie       | session            | token                     |
| ------ | ------------ | ------------------ | ------------------------- |
| 保存地 | 保存在客户端 | 保存在服务器       | 保存在服务器              |
| 安全度 | 低，可以伪造 | 中，存在被胁持风险 | 高，uid与签名哈希加密匹配 |
| 常用   | 不重要数据   | 重要数据           | 登录状态                  |

简单区别session和token：

都是用于身份验证，但是session是拿到数据再检测，服务端需要保存数据；token是每次请求带上加密数据传输，拿到就解密辨明身份找到对应账号数据返回，服务端不保存数据。

##### 面试问题

项目中session创建的时间点，不同电脑登录同一账号sessionid是同一个吗，sessionid过多怎么办，会出现账号踢出吗，怎么实现？

答：在验证完账号密码登录成功后，不是同一个，因为我redis用的是以随机的sessionid作为key，账号作为Value，数据结构用的是字符串；有LRU处理；所以没有实现踢出，只能自己动手注销或时间到；可以增加一个检测，如果value相同则删除已经保存的sessionid，用新的sessionid取代。

## 分布式session机制

#### 粘性session

锁定session到某一个服务器上，例如第一次请求转发到了A，后面的请求都会转发到A

#### session复制

一个服务器的session发生修改，序列化内容并广播到其他服务器，不管其他服务器要不要session

#### session集中

用redis作为存储容器，作为中间键处理，备份可以用多个redis作为主从复制

#### session持久化

把session保存到数据库
