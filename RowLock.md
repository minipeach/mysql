#mysql行锁

做项目时由于业务逻辑的需要，必须对数据表的一行或多行加入行锁，举个最简单的例子，图书借阅系统。假设 id=1 的这本书库存为 1 ，但是有 2 个人同时来借这本书，此处的逻辑为

```sql
Select   restnum  from  book  where  id =1 ;    
-- 如果 restnum 大于 0 ，执行 update
Update   book  set restnum=restnum-1 where id=1 ;
```

问题就来了，当 2 个人同时来借的时候，有可能第一个人执行 select 语句的时候，第二个人插了进来，在第一个人没来得及更新 book 表的时候，第二个人查到数据了，其实是脏数据，因为第一个人会把 restnum 值减 1 ，因此第二个人本来应该是查到 id=1 的书 restnum 为 0 了，因此不会执行 update ，而会告诉它 id=1 的书没有库存 了，可是数据库哪懂这些，数据库只负责执行一条条 SQL 语句，它才不管中间有没有其他 sql 语句插进来，它也不知道要把一个 session 的 sql 语句执行完再执行另一个 session 的。因此会导致并发的时候 restnum 最后的结果为 -1 ，显然这是不合理的，所以，才出现锁的概念， Mysql 使用 innodb 引擎可以通过索引 对数据行加锁。以上借书的语句变为：

```sql
Begin;
Select   restnum  from  book  where  id =1  for   update ;
-- 给 id=1 的行加上排它锁且 id 有索引
Update   book  set restnum=restnum-1 where id=1 ;
Commit;
```

这样，第二个人执行到 select 语句的时候就会处于等待状态直到第一个人执行 commit 。从而保证了第二个人不会读到第一个人修改前的数据。
那这样是不是万无一失了呢，答案是否定的。看下面的例子。
 
跟我一步一步来，先建立表 

```sql
CREATE TABLE `book` ( 
  `id` int(11) NOT NULL auto_increment, 
  `num` int(11) default NULL, 
  `name` varchar(0) default NULL, 
  PRIMARY KEY  (`id`), 
  KEY `asd` (`num`) 
) ENGINE=InnoDB  DEFAULT CHARSET=gbk 
```



其中 num 字段加了索引 

然后插入数据，运行，

```sql
insert into book(num) values(11),(11),(11),(11),(11); 
insert into book(num) values(22),(22),(22),(22),(22); 
```

然后打开 2 个 mysql 控制台窗口，其实就是建立 2 个 session 做并发操作 

******************************************************************** 
在第一个 session 里运行：
```sql
begin; 
select * from book where num=11 for update; 
出现结果： 
+----+-----+------+ 
| id | num | name | 
+----+-----+------+ 
| 11 |  11 | NULL | 
| 12 |  11 | NULL | 
| 13 |  11 | NULL | 
| 14 |  11 | NULL | 
| 15 |  11 | NULL | 
+----+-----+------+ 
5 rows in set 
```

然后在第二个 session 里运行： 
```sql
begin; 
select * from book where num=22 for update; 
出现结果： 
+----+-----+------+ 
| id | num | name | 
+----+-----+------+ 
| 16 |  22 | NULL | 
| 17 |  22 | NULL | 
| 18 |  22 | NULL | 
| 19 |  22 | NULL | 
| 20 |  22 | NULL | 
+----+-----+------+ 
5 rows in set 
```

好了，到这里什么问题都没有，是吧，可是接下来问题就来了，大家请看： 
回到第一个 session ，运行： 
```sql
update book set name='abc' where num=11; 
```

******************************************************************************************** 
问题来了， session 竟然处于等待状态 ，可是 num=11 的行不是被第一个 session 自己锁住的么，为什么不能更新呢？好了，打这里大家也许有自己的答案，先别急，再请看一下操作。 


把 2 个 session 都关闭，然后运行： 

```sql
delete from book where num=11 limit 3; 
delete from book where num=22 limit 3; 
```

其实就是把 num=11 和 22 的记录各删去 3 行， 
然后重复 “***********************” 之间的操作 
竟然发现，运行 update book set name='abc' where num=11; 后，有结果出现了，说明没有被锁住， 
这是为什么呢，难道 2 行数据和 5 行数据，对 MySQL 来说，会产生锁行和锁表两种情况吗 。经过跟网友讨论和翻阅资料，仔细分析后发现：
在以上实验数据作为测试数据的情况下，由于 num 字段重复率太高，只有 2 个值，分别是 11 和 12. 而数据量相对于这两个值来说却是比较大的，是 10 条， 5 倍的关系。 
那么 mysql 在解释 sql 的时候，会忽略索引，因为它的优化器发现：即使使用了索引，还是要做全表扫描，故而放弃了索引，也就没有使用行锁，却使用了表锁。 简单的讲，就是 MYSQL 无视了你的索引，它觉得与其行锁，还不如直接表锁，毕竟它觉得表锁所花的代价比行锁来的小。以上问题即便你使用了 force index 强制索引，结果还是一样，永远都是表锁。
所以 mysql 的行锁用起来并不是那么随心所欲的，必须要考虑索引。再看下面的例子。

```sql
select id from items where id in (select id from items where id <6) for update;  
--id字段加了索引
select id from items where id in (1,2,3,4,5) for update;
```

大部分会认为结果一样没什么区别，其实差别大了，区别就是第一条 sql 语句会产生表锁，而第二个 sql 语句是行锁，为什么呢？因为第一个 sql 语句用了子查询外围查询故而没使用索引，导致表锁。
 
好了，回到借书的例子，由于 id 是唯一的，所以没什么问题，但是如果有些表出现了索引有重复值，并且 mysql 会强制使用表锁的情况，那怎么办呢？一般来说只有重新设计表结构和用新的 SQL 语句实现业务逻辑，但是其实上面借书的例子还有一种办法。请看下面代码：
   
```sql
Set   sql_mode=
'STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
Begin;
Select  restnum  from  book  where  id =1   ;  -- 取消排它锁 , 设置 restnum 为 unsigned
Update   book  set restnum=restnum-1 where id=1 ;
If(update 执行成功 )  commit;
Else   rollback; 
```


上面是个小技巧，通过把数据库模式临时设置为严格模式，当 restnum 被更新为 -1 的时候，由于 restnum 是 unsigned 类型的，因此 update 会执行失败，无论第二个 session 做了什么数据库操作，都会被回滚，从而确保了数据的正确性，这个目的只是为了防止并发的时候极小概率出现的 2 个 session 的 sql 语句嵌套执行导致数据脏读。当然最好的办法还是修改表结构和 sql 语句，让 MYSQL 通过索引来加行锁, MySQL 测试版本为 5.0.75-log 和 5.1.36-community




