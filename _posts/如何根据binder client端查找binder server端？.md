发生SWT时，backtrace经常会卡在binder client端等待binder server端返回：

```java
IPCThreadState::waitForResponse-->IPCThreadState::talkWithDriver
```
需要找到server端的pid才能进行下一步分析


##### 方法一：常规方法

根据binder client thread的sysTid在SYS_BINDER_INFO/SWT_JBT_TRACES中查找binder通信对端，关键字“**outgoing transaction**”:
>SYS_BINDER_INFO / SWT_JBT_TRACES中可以查看binder信息

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210516185335441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70)
Outgoing: Current thread is performing binder request to other process
Incoming: Current thread is performing binder service for other process

##### 方法二：如果在SYS_BINDER_INFO中无法找到binder对端信息怎么办？

可以尝试在kernel_log中查找binder release的log（binder所在进程结束时会调用），找到
debug_id再进行下一步定位，例如：
kernel_log
[ 543.692215] .(6)[6750:kworker/6:1]binder: release 1035:1035 transaction 257798 out, still active
debug_id是257798，之后根据debug_id对应查找SYS_BINDER_INFO可以找到对端信息：

```java
//SYS_BINDER_INFO：
//此处可以看到binder server端是670:0, 与binder server端通信的是1035进程
proc 670
context binder
thread 670: l 12 need_return 0 tr 0
outgoing transaction 261951: 0000000000000000 from 670:670 to 1035:0 code 1 flags 10 pri 0:120 r1 start
467.941511 android 2019-08-07 13:52:51.231
incoming transaction 256020: 0000000000000000 from 8297:8404 to 670:670 code 17 flags 10 pri 0:120 r1
start 462.911897 android 2019-08-07 13:52:46.201 node 1053 size 496:0 data 0000000000000000
……
pending transaction 257798: 0000000000000000 from 1035:1035 to 670:0 code 5 flags 10 pri 0:118 r1 start
463.595489 android 2019-08-07 13:52:46.885 node 1053 size 112:0 data 0000000000000000
```
如果SYS_BINDER_INFO中没有有效信息，也可以在KERNEL_LOG继续搜索看看是否有有效信息，例如：

```java
[340361.842572] .(0)[10434:kworker/0:2]binder: release 1170:1189 transaction 484462897 out, still active
//debug_id是484462897，找到对端信息：
[340361.959179] .(4)[1011:Binder:567_4]binder: 567:1011 transaction failed 29189/0, size 28-0 line 4487
[340361.960610] .(4)[1011:Binder:567_4]binder: send failed reply for transaction 484462897, target dead
//那么567:1011就是对端
```
>注意：
如果binder server为670:0，表示binder server没有空闲binder，需要查看binder server端的
binder都在做什么事情，是不是有卡住的情况；
如果binder server为0:0，表示binder server process已经挂掉，需要从log中查看是否有其它相
关信息。
如果按照上述方法依然找不到对端，只能按照code逻辑关系请相关module owner帮忙排查

