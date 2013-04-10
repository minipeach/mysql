mysql行锁
===================

做项目时由于业务逻辑的需要，必须对数据表的一行或多行加入行锁，举个最简单的例子，图书借阅系统。假设 id=1 的这本书库存为 1 ，但是有 2 个人同时来借这本书，此处的逻辑为

    Select   restnum  from  book  where  id =1 ;    
    -- 如果 restnum 大于 0 ，执行 update
    Update   book  set restnum=restnum-1 where id=1 ;
