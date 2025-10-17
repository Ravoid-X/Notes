# 消息队列
1. 定义
先进先出的队列型数据结构，是系统内核中的一个内部链表，尾添加头读取
2. 一般结构
   ````
    struct msgbuf{
	   long    mtype;
	   char	   mtext[1];
    }
   ````
   mtype指定了消息类型，为正整数\
   引入消息类型后，在逻辑上由一个消息链表转化为多个消息链表。消息依旧写入尾部，但接收时却可以有选择地读取某个特定类型的消息中最接近队列头的一个
3. 创建
   ````
   msgget(key_t key, int msgflg);
   ````
   key是关键字,参数key取值IPC_PRIVATE时，函数创建关键字为0的消息队列\
   UNIX内核要求关键字唯一，但也可以创建多个关键字为0的消息队列\
   msgflg低9位指定队列属主、属组和其他用户的访问权限，其它位指定消息队列的创建方式\
   IPC_CREAT：创建，如存在则打开，队列已存在返回其标识号
   ````
   msgid = msgget(0x1234, 0666|IPC_CREAT);
   ````
   IPC_EXCL：与IPC_CREAT使用，单独使用无意义,队列已存在则报错
   ````
   msgid = msgget(0x1234, 0666|IPC_CREAT|IPC_EXCL);
   ````
4. 发送
   ````
   int msgsnd(int msqid, void *msgp, int msgsz, int msgflg);
   ````
   msqid：消息队列标识符\
   msgp：msgp可以是任何类型的结构体，第一个字段必须为long,表明消息的类型\
   msgsz：要发送消息的大小，不含消息类型占用的4个字节\
   msgflg：0：当消息队列满时，msgsnd将会阻塞，直到消息能写进消息队列\
&emsp;&emsp;&emsp;&emsp;IPC_NOWAIT：队列已满时，msgsnd函数不等待立即返回\
&emsp;&emsp;&emsp;&emsp;IPC_NOERROR：若消息大于size字节，则截断超出消息不通知发送进程。
5. 读取
   ````
   int msgrcv(int msgid, void *msgp, int msgsz, long msgtyp, int msgflg);
   ````
   msgtyp：0：接收第一个消息\
&emsp;&emsp;&emsp;&emsp;>0：接收类型等于msgtyp的第一个消息\
&emsp;&emsp;&emsp;&emsp;<0：接收类型等于或者小于msgtyp绝对值的第一个消息\
   msgflg：0: 阻塞式接收消息，没有该类型的消息msgrcv函数一直阻塞等待\
&emsp;&emsp;&emsp;&emsp;IPC_NOWAIT：如果没有返回条件的消息调用立即返回，此时错误码为ENOMSG\
&emsp;&emsp;&emsp;&emsp;IPC_EXCEPT：与msgtype配合使用返回队列中第一个类型不为msgtype的消息\
&emsp;&emsp;&emsp;&emsp;IPC_NOERROR：若队列中满足条件的消息内容大于size字节，则截断超出消息
6. 获取属性
   ````
   int msgctl(int msqid, int cmd, struct msqid_ds *buf)
   ````
   cmd：PC_STAT:获得msgid的消息队列头数据到buf中\
&emsp;&emsp;&emsp;IPC_SET：设置队列属性，可设置msg_perm.uid、msg_perm.gid、msg_perm.mode、msg_qbytes\
buf：消息队列管理结构体，存储要设置的属性\