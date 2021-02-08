# SpringBoot-Jpa-sharding-jdbc分库分表测试
测试sharding-jdbc分库和分表，只需引入jpa和sharding-jdbc依赖即可，确实对应用侵入很低。

我分别测试了两张表，分两个库，每个库一张表，做分表策略时一定要注意数据均匀分配，否则就会出现某些表中永远没有数据的情况
```yaml
#这种配置就不满足我下面要说的情况，这种配置会导致，每个库中都必须存在冗余表，如果删除会报错
spring.shardingsphere.sharding.tables.user.actual-data-nodes=db$->{0..1}.user_$->{0..1}
```
我的想法： 两个db，db0,db1, db0的库分别存放user0,item0两张表，db1的库分别存放user1,item1两张表，于是有了下面的配置
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200817164814571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)


```yaml
spring.shardingsphere.sharding.tables.user.actual-data-
nodes=db0.user_0,db1.user_1
spring.shardingsphere.sharding.tables.item.actual-data-
nodes=db0.item_0,db1.item_1
```
这样搭配如下的策略就能保证每个表都是均匀存放数据
```yml
# db和table的算法是一样的所以当db = 0 时 表也为0
spring.shardingsphere.sharding.tables.user.table-strategy.inline.algorithm-
expression=user_$->{user_id % 2}
spring.shardingsphere.sharding.tables.item.table-strategy.inline.algorithm-
expression=item_$->{item_id % 2}

spring.shardingsphere.sharding.tables.user.database-strategy.inline.algorithm-
expression=db$->{ user_id % 2}
spring.shardingsphere.sharding.tables.item.database-strategy.inline.algorithm-
expression=db$->{ item_id % 2}
```
然后测试了一下jpa手写sql，leftjoin查询,也是没问题的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200817164905551.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
如上的小demo放在github上提供下载，sql脚本也有，只需要执行sql改下配置文件mysql密码即可使用

[点我跳转到demo下载](https://github.com/NoBugBoy/Jpa-sharding-jdbc)