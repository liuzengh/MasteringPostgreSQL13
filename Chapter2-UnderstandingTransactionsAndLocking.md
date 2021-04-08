

- 使用PostgreSQL事务
- 理解基本锁定
- 使用`FOR SHARE`和`FOR UPDATE`
- 理解事务隔离级别
- 观察死锁和类似的问题
- 利用咨询锁
- 优化存储和管理清理

### 使用PostgreSQL事务

PostgreSQL为你提供了非常先进的事务机制，为开发人员和管理员提供了无数的特性。在本节中，我们将研究事务的基本概念。

首先要知道的是，在PostgreSQL中，一切都是事务。如果你向服务器发送一个简单的查询，那么它已经是一个事务了。下面是一个例子:

```sql
postgres=# select now(), now();
              now              |              now              
-------------------------------+-------------------------------
 2021-04-08 21:33:28.727962+08 | 2021-04-08 21:33:28.727962+08
(1 row)
```

在本例中，`SELECT`语句将是一个单独的事务。如果再次执行相同的命令，将返回不同的时间戳。

提示： 请记住，`now()`函数将返回事务时间。因此，`SELECT`语句总是返回两个相同的时间戳。如果你想要获取实时时间，请考虑使用`clock_timestamp()`而不是`now()`。

同一个事务包含多条语句，则必须使用BEGIN语句，如下所示:

```sql
postgres=# \h begin
Command:     BEGIN
Description: start a transaction block
Syntax:
BEGIN [ WORK | TRANSACTION ] [ transaction_mode [, ...] ]

where transaction_mode is one of:

    ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }
    READ WRITE | READ ONLY
    [ NOT ] DEFERRABLE

URL: https://www.postgresql.org/docs/13/sql-begin.html
```

`BEGIN`语句将确保多个命令可以放进同一个事务:

```sql
postgres=# begin;
BEGIN
postgres=*# select now();
              now              
-------------------------------
 2021-04-08 21:46:52.280983+08
(1 row)

postgres=*# select now();
              now              
-------------------------------
 2021-04-08 21:46:52.280983+08
(1 row)

postgres=*# commit;
COMMIT
```

这里的重点是两个时间戳是相同的。正如我们前面提到的，我们讨论的是事务时间。

要结束事务，可以使用`COMMIT`:

```sql
postgres=# \h commit;
Command:     COMMIT
Description: commit the current transaction
Syntax:
COMMIT [ WORK | TRANSACTION ] [ AND [ NO ] CHAIN ]

URL: https://www.postgresql.org/docs/13/sql-commit.html
```

这里有一些语法元素。您可以只使用`commit`、`commit work`或`commit transaction`。这三个命令都有相同的含义。如果这还不够，还有更多:

```sql
postgres=# \h end
Command:     END
Description: commit the current transaction
Syntax:
END [ WORK | TRANSACTION ] [ AND [ NO ] CHAIN ]

URL: https://www.postgresql.org/docs/13/sql-end.html
```

`END`子句与`COMMIT`子句含义相同。

`ROLLBACK`是`COMMIT`相反的操作。它不会成功地结束一个事务，而是简单地停止事务，让当前事务对其他事务不可见，如下面的代码所示:

```sql
postgres=# \h rollback
Command:     ROLLBACK
Description: abort the current transaction
Syntax:
ROLLBACK [ WORK | TRANSACTION ] [ AND [ NO ] CHAIN ]

URL: https://www.postgresql.org/docs/13/sql-rollback.html
```

一些应用程序使用`ABORT`而不是`ROLLBACK`。意思是一样的。`PostgreSQL 12`的新颖之处在于链式事务的概念。这一切的意义是什么?下面的清单显示了一个示例:

```sql
postgres=# show transaction_read_only;
 transaction_read_only 
-----------------------
 off
(1 row)
postgres=# begin transaction read only;
BEGIN
postgres=*# select 1;
 ?column? 
----------
        1
(1 row)

postgres=*# commit and chain;
COMMIT
postgres=*# show transaction_read_only;
 transaction_read_only 
-----------------------
 on
(1 row)

postgres=*# select 1;
 ?column? 
----------
        1
(1 row)

postgres=*# commit and no chain;
COMMIT
postgres=# show transaction_read_only;
 transaction_read_only 
-----------------------
 off
(1 row)

postgres=# commit;
WARNING:  there is no transaction in progress
COMMIT
```

让我们一步一步地看看这个例子:

1. 显示`transaction_read_only`是否打开。它是关闭的，因为默认情况下，我们处于`read/write`模式。
2. 使用`BEGIN`启动只读事务。这将自动调整`transaction_read_only`变量。
3. 使用`AND CHAIN`提交事务，然后`PostgreSQL`将自动启动一个新的事务，该事务具有与前一个事务相同的属性。

在我们的示例中，我们也将处于`read-only`模式，就像之前的事务一样。不需要显式地打开一个新事务并再次设置任何值，这可以显著减少应用程序和服务器之间的往返次数。如果事务正常提交(=没有链)，事务的只读属性将消失。