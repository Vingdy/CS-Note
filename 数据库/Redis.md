# Redis

最大连接数：info clients查看

## 常用数据结构

数组string

列表list

哈希hash

集合set

有序集合zset

## 为什么快

①内存数据库

②单线程模式

③非阻塞IO

## 持久化

RDB（默认）：  当前进程数据生成快照保存到硬盘的过程。

fork一个进程，遍历hashtable，利用copy on write，把整个db dump保存下来。
save, shutdown, slave 命令会触发这个操作。粒度比较大，如果save, shutdown, slave 之前crash了，则中间的操作没办法恢复

AOF（主流）：  以独立日志的方式记录每次写命令，重启时再重新执行AOF文件 

把写操作指令，持续的写到一个类似日志文件里。（类似于从postgresql等数据库导出sql一样，只记录写操作）粒度较小，crash之后，只有crash之前没有来得及做日志的操作没办法恢复。 

 https://www.cnblogs.com/yhq-qhh/p/10140586.html 