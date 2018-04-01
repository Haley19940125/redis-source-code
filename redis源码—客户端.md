## redis源码--客户端
主要就是分为两部分： 客户端的创建与释放
### 客户端的创建
> Redis 服务器是一个同时与多个客户端建立连接的程序。每个客户端可以向服务器发送命令请求，而服务器则接收并处理客户端发送的命令，并向客户端返回命令回复。</br>

当客户端连接上服务器时，服务器会建立一个server.h/client结构来保存客户端的状态信息。
```c
typedef struct client {
uint64_t id;            /* Client incremental unique ID. */
int fd;                 /* Client socket. */
redisDb *db;            /* Pointer to currently SELECTed DB. */
robj *name;             /* As set by CLIENT SETNAME. */
sds querybuf;           /* Buffer we use to accumulate client queries. */
sds pending_querybuf;   /* If this is a master, this buffer represents the
yet not applied replication stream that we
are receiving from the master. */
size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size. */
int argc;               /* Num of arguments of current command. */
robj **argv;            /* Arguments of current command. */
struct redisCommand *cmd, *lastcmd;  /* Last command executed. */
int reqtype;            /* Request protocol type: PROTO_REQ_* */
int multibulklen;       /* Number of multi bulk arguments left to read. */
long bulklen;           /* Length of bulk argument in multi bulk request. */
list *reply;            /* List of reply objects to send to the client. */
unsigned long long reply_bytes; /* Tot bytes of objects in reply list. */
size_t sentlen;         /* Amount of bytes already sent in the current
buffer or object being sent. */
time_t ctime;           /* Client creation time. */
time_t lastinteraction; /* Time of the last interaction, used for timeout */
time_t obuf_soft_limit_reached_time;
int flags;              /* Client flags: CLIENT_* macros. */
int authenticated;      /* When requirepass is non-NULL. */
int replstate;          /* Replication state if this is a slave. */
int repl_put_online_on_ack; /* Install slave write handler on ACK. */
int repldbfd;           /* Replication DB file descriptor. */
off_t repldboff;        /* Replication DB file offset. */
off_t repldbsize;       /* Replication DB file size. */
sds replpreamble;       /* Replication DB preamble. */
long long read_reploff; /* Read replication offset if this is a master. */
long long reploff;      /* Applied replication offset if this is a master. */
long long repl_ack_off; /* Replication ack offset, if this is a slave. */
long long repl_ack_time;/* Replication ack time, if this is a slave. */
long long psync_initial_offset; /* FULLRESYNC reply offset other slaves
copying this slave output buffer
should use. */
char replid[CONFIG_RUN_ID_SIZE+1]; /* Master replication ID (if master). */
int slave_listening_port; /* As configured with: SLAVECONF listening-port */
char slave_ip[NET_IP_STR_LEN]; /* Optionally given by REPLCONF ip-address */
int slave_capa;         /* Slave capabilities: SLAVE_CAPA_* bitwise OR. */
multiState mstate;      /* MULTI/EXEC state */
int btype;              /* Type of blocking op if CLIENT_BLOCKED. */
blockingState bpop;     /* blocking state */
long long woff;         /* Last write global replication offset. */
list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */
list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
sds peerid;             /* Cached peer ID. */

/* Response buffer */
int bufpos;
char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```
所以在客户端创建时，就会初始化这样一个结构。客户端创建的源码如下：
```c
client *createClient(int fd) {
client *c = zmalloc(sizeof(client));//分配空间

/* passing -1 as fd it is possible to create a non connected client.
* This is useful since all the commands needs to be executed
* in the context of a client. When commands are executed in other
* contexts (for instance a Lua script) we need a non connected client. */
if (fd != -1) {
anetNonBlock(NULL,fd);
anetEnableTcpNoDelay(NULL,fd);
if (server.tcpkeepalive)
anetKeepAlive(NULL,fd,server.tcpkeepalive);
if (aeCreateFileEvent(server.el,fd,AE_READABLE,
readQueryFromClient, c) == AE_ERR)
{
close(fd);
zfree(c);
return NULL;
}
}

selectDb(c,0);
uint64_t client_id;
atomicGetIncr(server.next_client_id,client_id,1);
c->id = client_id;
c->fd = fd;
c->name = NULL;
c->bufpos = 0;
c->querybuf = sdsempty();
c->pending_querybuf = sdsempty();
c->querybuf_peak = 0;
c->reqtype = 0;
c->argc = 0;
c->argv = NULL;
c->cmd = c->lastcmd = NULL;
c->multibulklen = 0;
c->bulklen = -1;
c->sentlen = 0;
c->flags = 0;
c->ctime = c->lastinteraction = server.unixtime;
c->authenticated = 0;
c->replstate = REPL_STATE_NONE;
c->repl_put_online_on_ack = 0;
c->reploff = 0;
c->read_reploff = 0;
c->repl_ack_off = 0;
c->repl_ack_time = 0;
c->slave_listening_port = 0;
c->slave_ip[0] = '\0';
c->slave_capa = SLAVE_CAPA_NONE;
c->reply = listCreate();
c->reply_bytes = 0;
c->obuf_soft_limit_reached_time = 0;
listSetFreeMethod(c->reply,freeClientReplyValue);
listSetDupMethod(c->reply,dupClientReplyValue);
c->btype = BLOCKED_NONE;
c->bpop.timeout = 0;
c->bpop.keys = dictCreate(&objectKeyPointerValueDictType,NULL);
c->bpop.target = NULL;
c->bpop.numreplicas = 0;
c->bpop.reploffset = 0;
c->woff = 0;
c->watched_keys = listCreate();
c->pubsub_channels = dictCreate(&objectKeyPointerValueDictType,NULL);
c->pubsub_patterns = listCreate();
c->peerid = NULL;
listSetFreeMethod(c->pubsub_patterns,decrRefCountVoid);
listSetMatchMethod(c->pubsub_patterns,listMatchObjects);
if (fd != -1) listAddNodeTail(server.clients,c);
initClientMultiState(c);
return c;
}
```
源码剖析：
1. fd： 是客户端正在使用的套接字描述符。可以用于创建不同情景下的client。
- fd == -1:表示创建一个无网络连接的客户端。主要用于执行 lua 脚本中包含的redis命令或载入AOF文件并还原数据库状态，而不是来源于网络。所以这种客户端并不需要套接字连接，也不需要记录套接字描述符。就是一种伪客户端。
- d != -1。表示接收到一个正常的客户端连接，则会创建一个有网络连接的客户端，也就是创建一个文件事件，来监听这个fd是否可读，当客户端发送数据，则事件被触发。创建客户端时，还会禁用Nagle算法。
> Nagle算法能自动连接许多的小缓冲器消息，这一过程（称为nagling）通过减少必须发送包的个数来增加网络软件系统的效率。但是服务器和客户端的对即时通信性有很高的要求，因此禁止使用 Nagle 算法，客户端向内核递交的每个数据包都会立即发送给服务器。
```c
if (fd != -1) {
// 设置fd为非阻塞模式
anetNonBlock(NULL,fd);
// 禁止使用 Nagle 算法
anetEnableTcpNoDelay(NULL,fd);
// 如果开启了tcpkeepalive
if (server.tcpkeepalive)
// 设置tcp连接的keep alive选项，保持连接
anetKeepAlive(NULL,fd,server.tcpkeepalive);
// 创建一个文件事件状态el，监听读事件，开始接受命令的输入
if (aeCreateFileEvent(server.el,fd,AE_READABLE,
readQueryFromClient, c) == AE_ERR)
{
close(fd);
zfree(c);
return NULL;
}
}
```
2. 创建客户端的过程，会将server.h/client结构的所有成员初始化。（按照《redis设计与实现》中依次介绍部分重要的成员）
- robj *name：默认创建的客户端是没有名字的，可以通过CLIENT SETNAME命令设置名字。</br>
CLIENT SETNAME:</br>
>关于CLIENT命令，redis中有6条:</br>
CLIENT LIST</br>
CLIENT REPLY  ON|OFF|SKIP</br>
CLIENT KILL <ip:port> <option> [value] ... <option> [value]</br>
CLIENT SETNAME connection-name</br>
CLIENT GETNAME</br>
CLIENT PAUSE  </br>

CLIENT SETNAME的源码部分：
```c
else if (!strcasecmp(c->argv[1]->ptr,"setname") && c->argc == 3) {
int j, len = sdslen(c->argv[2]->ptr);
char *p = c->argv[2]->ptr;

/* Setting the client name to an empty string actually removes
* the current name. */
// 设置名字为空
if (len == 0) {
// 先释放掉原来的名字
if (c->name) decrRefCount(c->name);
c->name = NULL;
addReply(c,shared.ok);
return;
}
/* Otherwise check if the charset is ok. We need to do this otherwise
* CLIENT LIST format will break. You should always be able to
* split by space to get the different fields. */
// 检查名字格式是否正确
for (j = 0; j < len; j++) {
if (p[j] < '!' || p[j] > '~') { /* ASCII is assumed. */
addReplyError(c,
"Client names cannot contain spaces, "
"newlines or special characters.");
return;
}
}
// 释放原来的名字
if (c->name) decrRefCount(c->name);
// 设置新名字
c->name = c->argv[2];
incrRefCount(c->name);
addReply(c,shared.ok);
}
```
如果客户端自己没有设置名字，那么相应客户端的name属性指向NULL，如果设置了名字，那么name属性指向的字符串对象，该对象保存客户端的名字。
- int flags：客户端状态的标志。在server.h中定义了29种状态。
```c
/* Client flags */
#define CLIENT_SLAVE (1<<0)   /* This client is a slave server */
#define CLIENT_MASTER (1<<1)  /* This client is a master server */
#define CLIENT_MONITOR (1<<2) /* This client is a slave monitor, see MONITOR */
#define CLIENT_MULTI (1<<3)   /* This client is in a MULTI context (表示正在执行事务)*/
#define CLIENT_BLOCKED (1<<4) /* The client is waiting in a blocking operation （被阻塞）*/
#define CLIENT_DIRTY_CAS (1<<5) /* Watched keys modified. EXEC will fail. （事务使用的watchm命令监视的数据库键被修改）*/
#define CLIENT_CLOSE_AFTER_REPLY (1<<6) /* Close after writing entire reply. （表示有用户对这个客户端执行了client kill命令，或者客户端向服务器发送的命令请求中包含了错误的协议内容，服务器会将客户端积存在输出缓存区的所有内容发送给客户端，然后关闭客户端）*/
#define CLIENT_UNBLOCKED (1<<7) /* This client was unblocked and is stored in server.unblocked_clients （不再阻塞）*/
#define CLIENT_LUA (1<<8) /* This is a non connected client used by Lua （处理LUA中包含的redis命令的伪客户端）*/
#define CLIENT_ASKING (1<<9)     /* Client issued the ASKING command */
#define CLIENT_CLOSE_ASAP (1<<10)/* Close this client ASAP （客户端的输出缓冲区大小超出了服务器允许的范围，服务器会在下一次执行serverCronha函数时关闭这个客户端。积存在输出缓冲区的所有内容会被直接释放，不会返回给客户端）*/
#define CLIENT_UNIX_SOCKET (1<<11) /* Client connected via Unix domain socket */
#define CLIENT_DIRTY_EXEC (1<<12)  /* EXEC will fail for errors while queueing（事务在命令入队时出现错误，exec命令执行失败） */
#define CLIENT_MASTER_FORCE_REPLY (1<<13)  /* Queue replies even if is master */
#define CLIENT_FORCE_AOF (1<<14)   /* Force AOF propagation of current cmd. */
#define CLIENT_FORCE_REPL (1<<15)  /* Force replication of current cmd. （强制主服务器将当前执行的命令复制给所有从服务器）*/
#define CLIENT_PRE_PSYNC (1<<16)   /* Instance don't understand PSYNC. */
#define CLIENT_READONLY (1<<17)    /* Cluster client is in read-only state. */
#define CLIENT_PUBSUB (1<<18)      /* Client is in Pub/Sub mode. */
#define CLIENT_PREVENT_AOF_PROP (1<<19)  /* Don't propagate to AOF. */
#define CLIENT_PREVENT_REPL_PROP (1<<20)  /* Don't propagate to slaves. */
#define CLIENT_PREVENT_PROP (CLIENT_PREVENT_AOF_PROP|CLIENT_PREVENT_REPL_PROP)
#define CLIENT_PENDING_WRITE (1<<21) /* Client has output to send but a write
handler is yet not installed. */
#define CLIENT_REPLY_OFF (1<<22)   /* Don't send replies to client. */
#define CLIENT_REPLY_SKIP_NEXT (1<<23)  /* Set CLIENT_REPLY_SKIP for next cmd */
#define CLIENT_REPLY_SKIP (1<<24)  /* Don't send just this reply. */
#define CLIENT_LUA_DEBUG (1<<25)  /* Run EVAL in debug mode. */
#define CLIENT_LUA_DEBUG_SYNC (1<<26)  /* EVAL debugging without fork() */
#define CLIENT_MODULE (1<<27) /* Non connected client used by some module. */
```
- sds querybuf：保存客户端发来命令请求的输入缓冲区。以Redis通信协议的方式保存。</br>
  ###### example：
  set key value
  存在客户端的querbuf属性是一个SDS值

  结构图如下：</br>
  ![image](https://github.com/Haley19940125/redis-source-code/blob/master/querybuf.png?raw=true)
- int argc：命令参数个数。</br>
  robj *argv：命令参数列表。是一个数组，数组的每一项都是一个字符串对象。其中argv[0]是要执行的命令，其他项是传给命令的参数。

  结构图如下：</br>
  ![image](https://github.com/Haley19940125/redis-source-code/blob/master/argv%E5%92%8Cargc.png?raw=true)
- struct redisCommand *cmd, *lastcmd：当服务器得到argv和argc属性的值后，服务器将根据项rgv[0]的值，查找到命令实现函数。
命令表是一个字典，字典的键是一个SDS的结构，保存了命令的名字，字典的值是命令所对应的redisCommand结构（它保存了命令的实现函数、命令的标志、命令应给定的参数、命令的总执行次数和消耗时长等信息）。

  查找命令表，并将客户端状态的cmd指向目标redisCommand结构的整个过程如下：
  ![image](https://github.com/Haley19940125/redis-source-code/blob/master/%E5%91%BD%E4%BB%A4%E5%AE%9E%E7%8E%B0.png?raw=true)
- 输出缓冲区</br>
  char buf[16*1024]：保存执行完命令所得命令回复信息的静态缓冲区，它的大小是固定的，所以主要保存的是一些比较短的回复。分配client结构空间时，就会分配一个16K的大小。</br>
  int bufpos：记录静态缓冲区的偏移量，也就是buf数组已经使用的字节数。</br>
  以上是固定大小的缓冲区属性。</br>

  结构图如下：</br>
  ![image](https://github.com/Haley19940125/redis-source-code/blob/master/%E5%9B%BA%E5%AE%9A%E7%BC%93%E5%86%B2%E5%8C%BA.png?raw=true)

  可变大小缓冲区属性：</br>
  list *reply：保存命令回复的链表。因为静态缓冲区大小固定，主要保存固定长度的命令回复，当处理一些返回大量回复的命令，则会将命令回复以链表的形式连接起来。

  结构图如下：</br>
  ![image](https://github.com/Haley19940125/redis-source-code/blob/master/%E5%8F%AF%E5%8F%98%E5%A4%A7%E5%B0%8F%E7%BC%93%E5%86%B2%E5%8C%BA.png?raw=true)
- int authenticated:身份验证</br>
0表示客户端未通过验证。（除AUTH命令外，其他命令会被服务器拒绝执行）1表示客户端通过验证。
- int id：服务器对于每一个连接进来的都会创建一个ID，客户端的ID从1开始。每次重启服务器会刷新。
- 和时间相关：</br>
  time_t ctime;           /* Client creation time. */</br>
  time_t lastinteraction; /* Time of the last interaction, used for timeout */</br>
  time_t obuf_soft_limit_reached_time;/* 记录了输出缓冲区第一次到达软性限制的时间 */
3. 客户端的创建</br>
  为客户端创建相应的客户端状态，并将这个状态添加到clients链表的结尾。

  结构图如下：
  ![image](https://github.com/Haley19940125/redis-source-code/blob/master/clients%E9%93%BE%E8%A1%A8.png?raw=true)
### 客户端的释放
源码如下：
```c
void freeClient(client *c) {
listNode *ln;

/* If it is our master that's beging disconnected we should make sure
* to cache the state to try a partial resynchronization later.
*
* Note that before doing this we make sure that the client is not in
* some unexpected state, by checking its flags. */
if (server.master && c->flags & CLIENT_MASTER) {
serverLog(LL_WARNING,"Connection with master lost.");
if (!(c->flags & (CLIENT_CLOSE_AFTER_REPLY|
CLIENT_CLOSE_ASAP|
CLIENT_BLOCKED|
CLIENT_UNBLOCKED)))
{
replicationCacheMaster(c);
return;
}
}

/* Log link disconnection with slave */
if ((c->flags & CLIENT_SLAVE) && !(c->flags & CLIENT_MONITOR)) {
serverLog(LL_WARNING,"Connection with slave %s lost.",
replicationGetSlaveName(c));
}

/* Free the query buffer */
sdsfree(c->querybuf);
sdsfree(c->pending_querybuf);
c->querybuf = NULL;

/* Deallocate structures used to block on blocking ops. */
if (c->flags & CLIENT_BLOCKED) unblockClient(c);
dictRelease(c->bpop.keys);

/* UNWATCH all the keys */
unwatchAllKeys(c);
listRelease(c->watched_keys);

/* Unsubscribe from all the pubsub channels */
pubsubUnsubscribeAllChannels(c,0);
pubsubUnsubscribeAllPatterns(c,0);
dictRelease(c->pubsub_channels);
listRelease(c->pubsub_patterns);

/* Free data structures. */
listRelease(c->reply);
freeClientArgv(c);

/* Unlink the client: this will close the socket, remove the I/O
* handlers, and remove references of the client from different
* places where active clients may be referenced. */
unlinkClient(c);

/* Master/slave cleanup Case 1:
* we lost the connection with a slave. */
if (c->flags & CLIENT_SLAVE) {
if (c->replstate == SLAVE_STATE_SEND_BULK) {
if (c->repldbfd != -1) close(c->repldbfd);
if (c->replpreamble) sdsfree(c->replpreamble);
}
list *l = (c->flags & CLIENT_MONITOR) ? server.monitors : server.slaves;
ln = listSearchKey(l,c);
serverAssert(ln != NULL);
listDelNode(l,ln);
/* We need to remember the time when we started to have zero
* attached slaves, as after some time we'll free the replication
* backlog. */
if (c->flags & CLIENT_SLAVE && listLength(server.slaves) == 0)
server.repl_no_slaves_since = server.unixtime;
refreshGoodSlavesCount();
}

/* Master/slave cleanup Case 2:
* we lost the connection with the master. */
if (c->flags & CLIENT_MASTER) replicationHandleMasterDisconnection();

/* If this client was scheduled for async freeing we need to remove it
* from the queue. */
if (c->flags & CLIENT_CLOSE_ASAP) {
ln = listSearchKey(server.clients_to_close,c);
serverAssert(ln != NULL);
listDelNode(server.clients_to_close,ln);
}

/* Release other dynamically allocated client structure fields,
* and finally release the client structure itself. */
if (c->name) decrRefCount(c->name);
zfree(c->argv);
freeClientMultiState(c);
sdsfree(c->peerid);
zfree(c);
}
```
主要是释放各种数据结构和清空一些缓冲区等等操作。</br>
还有一种释放方式：
异步释放客户端
```c
//异步释放客户端
void freeClientAsync(client *c) {
//如果是已经即将关闭或者是lua脚本的伪client，则直接返回
if (c->flags & CLIENT_CLOSE_ASAP || c->flags & CLIENT_LUA) return;
c->flags |= CLIENT_CLOSE_ASAP;
// 将client加入到即将关闭的client链表中
listAddNodeTail(server.clients_to_close,c);
}
```
> server.clients_to_close：是服务器保存所有待关闭的client链表。

设置异步释放客户端的目的主要是：防止底层函数正在向客户端的输出缓冲区写数据的时候，关闭客户端，这样是不安全的。Redis会安排客户端在serverCron()函数的安全时间释放它。

如果想要取消异步释放，那么会调用freeClientsInAsyncFreeQueue()立即释放。</br>
源码如下：
```c
void freeClientsInAsyncFreeQueue(void) {
// 循环遍历所有即将关闭的client
while (listLength(server.clients_to_close)) {
listNode *ln = listFirst(server.clients_to_close);
client *c = listNodeValue(ln);
// flag变为取消立即关闭的标志
c->flags &= ~CLIENT_CLOSE_ASAP;
freeClient(c);
// 从即将关闭的client链表中删除
listDelNode(server.clients_to_close,ln);
}
}
```
