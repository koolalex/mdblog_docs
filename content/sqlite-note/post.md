# SQLite学习笔记
## 特性
- 没有类似mysql、sql server 等数据库服务器的概念，因此无法像mysql等那样比较好的支持更高的并发度，且无法支持多进程同时修改数据库
- 由于没有服务器协调，因此sqlite的锁机制依赖于系统OS，同时也无法很好支持网络共享访问
- 支持ACID Atomic, Consistent, Isolated, and Durable
- 线程安全，Windows版本默认是线程安全的，Linux版本需要重新编译（编译时需要把预处理宏THREADSAFE设置为1）
- 没有账户概念，没办法进行权限控制
- 支持部分SQL标准，不受支持的功能包括完全触发器和可写视图
- 支持数据库大小至140TB，Max row size:1GB
- 比很多数据在大部分普通的数据库操作要快
- 支持事务及索引等，不支持类似SqlServer聚集索引（即那种按索引排序）

## 数据类型
数据类型 | 描述
-|-
NULL | 值是一个NULL值 | 
INTERGER | 值是一个带符号的整数，根据值的大小存储在1、2、3、4、6或8个字节中 | 
REAL | 值是一个浮点值，存储为8字节的IEEE浮点数字 | 
TEXT | 值是一个文本字符串，使用数据库编码（UTF-8、UTF-16BE或者UTF-16LE）存储。
BLOB | 值是一个blob数据，完全根据它的输入存储（二进制大对象如图片、音频等）

## 并发
### 锁机制
- sqlite提供库级锁（数据库文件级别锁，进程级别锁）
- 支持多进程访问数据库（读取），但是同时只有一个进程可以修改数据库
- NFS文件系统，并发读锁可能存在问题，因为NFS的fcntl()文件锁定时可能会出问题
- 在FAT文件系统中，sqlite依赖于Share.exe后台程序，如果该程序没有运行，则锁机制可能不会工作

### 优化
> sqlite提供库级锁（数据库文件级别锁，进程级别锁）
支持多进程访问数据库（读取），但是同时只有一个进程可以修改数据库
NFS文件系统，并发读锁可能存在问题，因为NFS的fcntl()文件锁定时可能会出问题
在FAT文件系统中，sqlite依赖于Share.exe后台程序，如果该程序没有运行，则锁机制可能不会工作

```c++
表准备(sql)：
//一行大概是
create table t1 (id integer , x integer , y integer， weight real)  

//【慢速】 插入10000000（100W）条,基本上是7.826条/s
sqlite3_exec(db,"begin;",0,0,0);  
for(int i=0;i<nCount;++i)  
{  
    std::stringstream ssm;  
    ssm<<"insert into t1 values("<<i<<","<<i*2<<","<<i/2<<","<<i*i<<")";  
    sqlite3_exec(db,ssm.str().c_str(),0,0,0);  
}  
sqlite3_exec(db,"commit;",0,0,0); 
```
**1. 开启事务**

34095条/s

- 由于sqlite的数据操作实质是文件的IO操作，频繁的插入数据会导致IO经常开闭，因此推荐开启事务。事务的作用是使数据线缓存在系统中，提交事务时再将所有更改写入数据库文件中。

**2. 写同步(synchronous)**

41851条/s
```c++
// 在调用sqlite3_open函数后添加下面一行代码：
sqlite3_exec(db, "PRAGMA synchronous = OFF; ", 0,0,0);
```
- PRAGMA synchronous = FULL; (2) 当synchronous设置为FULL (2), SQLite数据库引擎在紧急时刻会暂停以确定数据已经写入磁盘。这使系统崩溃或电源出问题时能确保数据库在重起后不会损坏。FULL synchronous很安全但很慢。
- PRAGMA synchronous = NORMAL; (1) 当synchronous设置为NORMAL, SQLite数据库引擎在大部分紧急时刻会暂停，但不像FULL模式下那么频繁。 NORMAL模式下有很小的几率(但不是不存在)发生电源故障导致数据库损坏的情况。但实际上，在这种情况 下很可能你的硬盘已经不能使用，或者发生了其他的不可恢复的硬件错误。
- PRAGMA synchronous = OFF; (0) 设置为synchronous OFF (0)时，SQLite在传递数据给系统以后直接继续而不暂停。若运行SQLite的应用程序崩溃， 数据不会损伤，但在系统崩溃或写入数据时意外断电的情况下数据库可能会损坏。另一方面，在synchronous OFF时 一些操作可能会快50倍甚至更多。在SQLite 2中，缺省值为NORMAL.而在SQLite 3中修改为FULL。
- **建议** 如果有定期备份的机制，而且少量数据丢失可接受，用OFF。

**3. 开启预处理**

265816条/s

- 预处理的原理就是将一条语句先预编译到数据库，下次再次执行相同的语句时，就不用再次编译，节省了大量的时间。
- 由于批量插入的数据不尽相同，所以数据库会多次编译插入语句，性能会损耗非常多，也就造成插入需要的时间会比较多。如此一来使用参数化传值，就能使每一次的插入的sql语句都是相同的

```c++
StringBuilder sql = new StringBuilder();
SQLiteParameter[] sp = new SQLiteParameter[2];
foreach (DataRow dr in dt.Rows)
{
    sql.Clear();
    sql.Append("INSERT INTO Info (Name,Code) VALUES(@Name,@Code) \r\n");  
    sps[0] = new SQLiteParameter("@p1", dr["Name"]);
    sps[1] = new SQLiteParameter("@p2", dr["Code"]);
    sqlHelper.SqliteHelper.ExecuteNonQuery(sql.ToString(),sp, CommandType.Text);  
}
```

**4. 综上，急速就是：**

SQLite插入数据效率最快的方式就是：***事务+关闭写同步+执行准备（开启预处理）***，如果对数据库安全性有要求的话，就开启写同步

## 引用&参考
> https://www.cnblogs.com/mofei004/p/9050091.html