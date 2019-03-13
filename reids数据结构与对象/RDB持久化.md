> #### 为了数据持久化-RDB文件是压缩后的二进制文件 SAVE or BGSAVE

## RDB的创建载入

> save （阻塞命令）和 bgsave（不阻塞） 都可以创建rdb文件，

#### RDB文件创建

save会阻塞当前服务器，bgsave派生子进程来执行操作。

```c
def SAVE():
	rdbSave();

def BGSAVE()
    pid = fork();
	if pid == 0:
		//子进程负责创建RDB文件
		rdbSave();
		//完成后通知父进程
		siginal_parent();
	elif pid > 0:
		handle_request_and _wait_signil();
	else:
		handle_fork_err();
```

> BGSAVE执行时save会被拒绝，BGREWRITEAOF会被推迟到BGSAVE之后。BGREWRITEAOF执行时，BGSAVE会被拒绝，**优先级： SAVE < BGSAVE < BGREWRITEAOF**

#### RDB文件载入

redis服务器启动时自动检测，如果AOF文件存在，载入AOF文件，不存在，就会载入RDB文件（前提开启了RDB）



## 自动间隔性保存

#### 以下配置：触发保存含义

```c
struct saveparam{
    //秒数
    time_t seconds;
    //修改数
    int changes;
}
```



```
save 900 1
save 300 10
save 60 10000
```

- 服务器在900s内至少执行一次修改
- 服务器在300s内至少10次修改
- 服务器60s内至少执行10000次修改

#### dirty计数器和lastsave属性

```c
//这个属性是和整个数据库同级的，因为他针对所有key统计
struct redisServer{
    //...
    //一个数组，保存着服务器中的所有数据库
    redisDB **db;
    //修改计数器
    long long dirty;
    //上一次保存时间
    time_t lastsave;
    //条件数组
    saveparam **saveparam
}
```

#### 检查条件是否满足

redis服务器会周期运行serverCron，默认100毫秒执行一次。遍历saveparam数组，检查条件是否满足。



## RDB文件结构

![](https://github.com/neorof/picture/blob/master/redis/RDB/RDB%E7%BB%93%E6%9E%84.jpg?raw=true)

1. REDIS 5个字节开始，作为rdb文件开始标记
2. 4字节db_version，rdb文件版本
3. database 包含0个或多个数据库**以及键值对** 。
4. EOF 1字节，代表文件结束
5. check_sum8字节校验和

#### database部分

![](https://github.com/neorof/picture/blob/master/redis/RDB/database.jpg?raw=true)

SELECTDB长度1字节，db_number 1,2,5字节，key_value_pairs键值对。如果有过期时间，过期时间会和键值对保存在一起。

![](https://github.com/neorof/picture/blob/master/redis/RDB/RDB%E6%96%87%E4%BB%B6%E5%85%A8%E8%B2%8C.png?raw=true)

#### key_value_pairs部分

![](https://github.com/neorof/picture/blob/master/redis/RDB/k-v%E8%BF%87%E6%9C%9F%E6%97%B6%E9%97%B4%E4%BF%9D%E5%AD%98%E6%96%B9%E5%BC%8F.png?raw=true)

>  **在转为RDB文件时**  如果字符串对象是**raw编码**，如果字符串大于20字节，用压缩方式保存，否则原样保存。



## 分析RDB文件

#### 空RDB文件

```shell
FLUSHDB
SAVE
od -c dump.rdb
```

可以看到获取的文件如下：

![](https://github.com/neorof/picture/blob/master/redis/RDB/RDB%E6%BA%90%E6%96%87%E4%BB%B6.jpg?raw=true)

- 5个字节：R E D I S
- 四个字节RDB文件版本0 0 0 6
- 一个字节AOF 337
- 8个字节校验和

