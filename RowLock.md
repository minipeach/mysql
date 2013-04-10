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








