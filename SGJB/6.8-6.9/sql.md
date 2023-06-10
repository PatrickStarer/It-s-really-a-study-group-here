mysql中的limit偏移量过大，导致查询速度降低，如何优化


id age
1、回表 避免回表
select * from table where id in  (select id from table where age =10 )   limit 999999 10;

如果offset大于10000，这个数据查询就会非常的慢。为什么

