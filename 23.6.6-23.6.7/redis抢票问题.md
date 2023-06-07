有一个演唱会抢票，我有3个演唱会抢票场次
每场100张票，mysql表该怎样设计？redis怎么设计?
* 3个抢票场次是同一时间进行
* 怎样确定是哪个场次的呢

id conut    id  tid 
1   100     1    2
2   100     2    3 
3   100     3    1

redis ticket:1 2  redis ticket:1 100
lua脚本：保证redis原子性操作 保证不超卖.
redisson 里面内嵌有lua
redis宕机、mysql数据不一致：支付接口回调消费者使用mq减库存。
mq延时消息，超时付款。
各种组件连起来。
