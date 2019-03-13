> **redis通过保存服务器执行的写命令来记录服务器状态** 

## AOF持久化的实现

> 持久化功能分为命令追加、文件写入、文件同步

#### 命令追加

当服务器开启AOF，服务器在执行一个命令之后，会议协议格式将执行的命令追加到aof_buf缓冲区末尾：

```c
struct redisServer{
    //...
    //AOF缓冲区
    sds aof_buf;
}
```

#### AOF文件的写入与同步

服务器定期执行serverCron函数，都会调用flushAppendOnlyFile函数，考虑需不需要将aof_buf的内容写入文件。

```
def eventLoop():
	while true:
		processFileEvents()
		processTimeEvents()
		
		//将aof_buf刷入文件（看配置是否同步）
		flushAppendOnlyFile()
```



flushAppendOnlyFile函数行为由服务器配置的appendfsync选项值来决定：

| appendfsync            | flushAppendOnlyFile                                          |
| ---------------------- | ------------------------------------------------------------ |
|                        | 将aof_buf缓冲区的内容写入并同步到AOF文件                     |
| everysec（**默认值**） | 将aof_buf缓冲区的内容写入                                    |
|                        | 将aof_buf缓冲区的内容写入AOF文件，但不进行同步，什么时候同步由系统决定。 |

> 文件的写入与同步：
>
> 操作系统为提高文件写入效率，用户调用write函数（写入）时，通常操作系统会将内容保存在系统缓冲区，当系统缓冲区被填满或超时才真正写入磁盘。
>
> 为此操作系统提供fsync和fdatasync两个同步函数，他们可以强制让系统缓存刷入磁盘。



> **注意，即使配置always， 也可能丢失一个事件循环（100ms）的数据。**

#### AOF文件的载入与数据还原

AOF文件里包含重建数据库状态的所有写命令，还原步骤如下：

1.  创建一个不带网络连接的伪客户端，因为redis只能在客户端上下文中执行命令。
2. 分析AOF文件，并读出一条命令
3. 伪客户端执行命令
4. 循环2,3 完成数据库状态还原

#### AOF重写

同一个key有可能被更新过多次，使得有好多条AOF记录，但只有最终状态才是有用的（比如push多次）。如果不加以控制AOF文件体积会越来越大，还原所需时间越来越多。

AOF的重写不依赖当前AOF文件，而是依赖当前数据库状态。流程：

```c
def aof_rewrite(new_aof_file_name) {
    f = create_file(new_aof_file_name)
    for db in redisServer.db:
    	if db.is_empty: continue
    	f.write_command("SELECT" + db.id)
    	for key in db:
    		...
}
```

> 实际中，为了防止重写时客户端输入缓冲区溢出，对于列表，哈希表，集合，有序集合四种如果元素数量超过64，会拆分为多个命令。

Redis防止主进程阻塞，使用子进程执行重写。但是也有个问题：子进程重写期间，主进程还能处理请求，导致重写后的AOF和当前数据库状态不一致。

为了解决数据不一致问题，redis设置了重写缓冲区，重写期间，redis服务器执行完一个命令会将写命令发送给AOF缓冲区（保存AOF文件）和AOF重写缓冲区（保存AOF重写文件）

当重些完成，AOF重写缓冲区的内容会追加到AOF重写文件，然后原子的覆盖旧AOF文件。