## 消息的增删改查

```php
# * 号表示服务器自动生成 ID，后面顺序跟着一堆 key/value
127.0.0.1:6379> xadd nba * name kobe age 1
"1590457690522-0" # 生成的消息 ID
127.0.0.1:6379> xadd nba * name james age 2
"1590457702886-0"
127.0.0.1:6379> xadd nba * name yao age 3
"1590462473402-0"
127.0.0.1:6379> xlen nba
(integer) 3
# -表示最小值 , + 表示最大值
127.0.0.1:6379> xrange nba - +
1) 1) "1590457690522-0"
   2) 1) "name"
      2) "kobe"
      3) "age"
      4) "1"
2) 1) "1590457702886-0"
   2) 1) "name"
      2) "james"
      3) "age"
      4) "2"
3) 1) "1590462473402-0"
   2) 1) "name"
      2) "yao"
      3) "age"
      4) "3"
# 指定最小消息 ID 的列表
127.0.0.1:6379> xrange nba 1590457702886-0 +
1) 1) "1590457702886-0"
   2) 1) "name"
      2) "james"
      3) "age"
      4) "2"
2) 1) "1590462473402-0"
   2) 1) "name"
      2) "yao"
      3) "age"
      4) "3"
# 指定最大消息 ID 的列表
127.0.0.1:6379> xrange nba - 1590457702886-0
1) 1) "1590457690522-0"
   2) 1) "name"
      2) "kobe"
      3) "age"
      4) "1"
2) 1) "1590457702886-0"
   2) 1) "name"
      2) "james"
      3) "age"
      4) "2"
127.0.0.1:6379> xdel nba 1590462473402-0
(integer) 1
# 长度改变了
127.0.0.1:6379> xlen nba
(integer) 2
# 被删除的消息没了
127.0.0.1:6379> xrange nba - +
1) 1) "1590457690522-0"
   2) 1) "name"
      2) "kobe"
      3) "age"
      4) "1"
2) 1) "1590457702886-0"
   2) 1) "name"
      2) "james"
      3) "age"
      4) "2"
# 删除整个 Stream
127.0.0.1:6379> del nba
(integer) 1
```

## 独立消费

```php
127.0.0.1:6379> xadd nba * name a age 1
"1590463614436-0"
127.0.0.1:6379> xadd nba * name b age 2
"1590463620067-0"
127.0.0.1:6379> xadd nba * name c age 3
"1590463623840-0"
# 从 Stream 头部读取两条消息
127.0.0.1:6379> xread count 2 streams nba 0-0
1) 1) "nba"
   2) 1) 1) "1590463614436-0"
         2) 1) "name"
            2) "a"
            3) "age"
            4) "1"
      2) 1) "1590463620067-0"
         2) 1) "name"
            2) "b"
            3) "age"
            4) "2"
# 从 Stream 尾部读取一条消息，毫无疑问，这里不会返回任何消息
127.0.0.1:6379> xread count 1 streams nba $
(nil)
# 从尾部阻塞等待新消息到来，下面的指令会堵住，直到新消息到来
127.0.0.1:6379> xread block 0 count 1 streams nba $
# 我们从新打开一个窗口，在这个窗口往 Stream 里塞消息
127.0.0.1:6379> xadd nba * name d age 4
"1590463724560-0"
# 再切换到前面的窗口，我们可以看到阻塞解除了，返回了新的消息内容
# 而且还显示了一个等待时间，这里我们等待了 14s
127.0.0.1:6379> xread block 0 count 1 streams nba $
1) 1) "nba"
   2) 1) 1) "1590463724560-0"
         2) 1) "name"
            2) "d"
            3) "age"
            4) "4"
(14.34s)
# block 0 表示永远阻塞，直到消息到来，block 1000 表示阻塞 1s，如果 1s 内没有任何消息到来，就返回 nil。
127.0.0.1:6379> xread block 1000 count 1 streams nba $
(nil)
(1.00s)
```

## 创建消费组

```php
# 0-0 表示从头开始消费
127.0.0.1:6379> xgroup create nba cg1 0-0
OK
# $ 表示从尾部开始消费，只接受新消息，当前 Stream 消息会全部忽略
127.0.0.1:6379> xgroup create nba cg2 $
OK
# 获取 Stream 信息
127.0.0.1:6379> xinfo stream nba
 1) "length"
 2) (integer) 4 # 共 4 个消息
 3) "radix-tree-keys"
 4) (integer) 1
 5) "radix-tree-nodes"
 6) (integer) 2
 7) "groups" # 两个消费组
 8) (integer) 2
 9) "last-generated-id"
10) "1590463724560-0"
11) "first-entry" # 第一个消息
12) 1) "1590463614436-0"
    2) 1) "name"
       2) "a"
       3) "age"
       4) "1"
13) "last-entry" # 最后一个消息
14) 1) "1590463724560-0"
    2) 1) "name"
       2) "d"
       3) "age"
       4) "4"
# 获取 Stream 的消费组信息
127.0.0.1:6379> xinfo groups nba
1) 1) "name"
   2) "cg1"
   3) "consumers"
   4) (integer) 0 # 该消费组还没有消费者
   5) "pending"
   6) (integer) 0 # 该消费组没有正在处理的消息
   7) "last-delivered-id"
   8) "0-0"
2) 1) "name"
   2) "cg2"
   3) "consumers"
   4) (integer) 0 # 该消费组还没有消费者
   5) "pending"
   6) (integer) 0 # 该消费组没有正在处理的消息
   7) "last-delivered-id"
   8) "1590463724560-0"
```

## 消费

```php
# > 号表示从当前消费组的 last_delivered_id 后面开始读
# 每当消费者读取一条消息，last_delivered_id 变量就会前进
127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 1 streams nba >
1) 1) "nba"
   2) 1) 1) "1590463614436-0"
         2) 1) "name"
            2) "a"
            3) "age"
            4) "1"
127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 1 streams nba >
1) 1) "nba"
   2) 1) 1) "1590463620067-0"
         2) 1) "name"
            2) "b"
            3) "age"
            4) "2"
127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 2 streams nba >
1) 1) "nba"
   2) 1) 1) "1590463623840-0"
         2) 1) "name"
            2) "c"
            3) "age"
            4) "3"
      2) 1) "1590463724560-0"
         2) 1) "name"
            2) "d"
            3) "age"
            4) "4"
# 再继续读取，就没有新消息了
127.0.0.1:6379> xreadgroup GROUP cg1 c1 count 1 streams nba >
(nil)
# 阻塞等待
127.0.0.1:6379> xreadgroup GROUP cg1 c1 block 0 count 1 streams nba >
# 开启另一个窗口，往里塞消息
127.0.0.1:6379> xadd nba * name e age 5
"1590470117509-0"
# 回到前一个窗口，发现阻塞解除，收到新消息了
127.0.0.1:6379> xreadgroup GROUP cg1 c1 block 0 count 1 streams nba >
1) 1) "nba"
   2) 1) 1) "1590470117509-0"
         2) 1) "name"
            2) "e"
            3) "age"
            4) "5"
(9.95s)
# 观察消费组信息
127.0.0.1:6379> xinfo groups nba
1) 1) "name"
   2) "cg1"
   3) "consumers"
   4) (integer) 1 # 一个消费者
   5) "pending"
   6) (integer) 5 # 共 5 条正在处理的信息还有没有 ack
   7) "last-delivered-id"
   8) "1590470117509-0"
2) 1) "name"
   2) "cg2"
   3) "consumers"
   4) (integer) 0 # 消费组 cg2 没有任何变化，因为一直在操纵 cg1
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "1590463724560-0"
# 如果同一个消费组有多个消费者，我们可以通过 xinfo consumers 指令观察每个消费者的状态
127.0.0.1:6379> xinfo consumers nba cg1
1) 1) "name"
   2) "c1"
   3) "pending"
   4) (integer) 5 # 共 5 条待处理消息
   5) "idle"
   6) (integer) 58005 # 空闲了多长时间 ms 没有读取消息了
# ack 一条消息
127.0.0.1:6379> xack nba cg1 1590457690522-0
(integer) 0
127.0.0.1:6379> xack nba cg1 1590463614436-0
(integer) 1
127.0.0.1:6379> xinfo consumers nba cg1
1) 1) "name"
   2) "c1"
   3) "pending"
   4) (integer) 4 # 变成了 5 条
   5) "idle"
   6) (integer) 142207
# ack 所有消息
127.0.0.1:6379> xack nba cg1 1590463620067-0 1590463623840-0 1590463724560-0 1590470117509-0
(integer) 4
127.0.0.1:6379> xinfo consumers nba cg1
1) 1) "name"
   2) "c1"
   3) "pending"
   4) (integer) 0 # pel 空了
   5) "idle"
   6) (integer) 191720
```

