auto commit
--------
MYSQL預設事務立即执行(AUTOCOMMIT=1)
檢查測試機與正式機AUTOCOMMIT皆為ON

SHOW VARIABLES WHERE Variable_name='autocommit';
or
SELECT @@autocommit;

事務隔離級別
---------
check session transaction level (mysql8+)
SELECT @@transaction_ISOLATION;

check global transaction level (mysql8+)
SELECT @@global.transaction_ISOLATION;

MySQL預設的事務隔離級別是 REPEATABLE-READ
檢查測試與正式皆為 `REPEATABLE-READ`

*** 
* 自增長id情況下，彼此 insert 不影響，兩事務可同時insert, lock=>primary key

* MAX會鎖表
ex: brands表
```
INSERT INTO `brands`
(`memo`, `sort`, `created_at`, `updated_at`)
SELECT '', IFNULL(MAX(sort) + 1, 1) , '2023-07-03 14:47:15' , '2023-07-03 14:47:15'
FROM `brands`
```


實例
--------
moveSupplierImportingProductsForRebuilding rebuild時 `product_categories`  會刪除再重建，若剛好是最後一筆，會產生supremum pseudo-record 的 間隙鎖(後來看LOCK_MODE是 X)，
這時若有其他行程執行insert會等待
* 造成原因:
* 非唯一索引會鎖前後gap，詳細請看
解法:
1. innodb_locaks_unsafe_for_binlog 似乎是可以解決的危險方案，mysql8已棄用，不再研究
2. SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
   ISOLATION LEVEL READ COMMITTED
3. 將transaction拆小，或者如果沒有很頻繁的發生，就讓他等吧
   查看資料庫總共rebuild也只有發生7次，計算事務開始到結束在效能較差的local機平均也才0.3秒左右，這在排程上是很可以接受的速度，故決定不理會它。
  

這篇解釋得很棒
* https://juejin.cn/post/7018137095315128328



題外話，
* Laravel 的 insertGetId 並非事務，雖然機率很小，但有可能取到的並不是正確的id
若沒要取得id，用insert就好，可以少進出DB一次
* insertBatch 是使用 insert ignore into , 只要其中有一個成功就會回傳1
	insert ignore into 與 insert into鎖的機制相同


MySQL 可以透過 SQL 指令 > SHOW ENGINE INNODB STATUS 看最後一筆發生 Deadlock 的原因，或是開啟 innodb_print_all_deadlocks 把每一次 Deadlock 原因都輸出到 error log 中
開啟設定的 SQL 為 > SET GLOBAL innodb_print_all_deadlocks=ON;


SELECT * FROM performance_schema.data_locks


* 意象鎖:避免要取得表鎖時，須全表掃描是否有行鎖
* S锁和X锁都是行级别（rwo-level）的行锁，是加在索引（记录）上的，兼容与否是指对同一条记录（row）来说的。
* 意向锁之间是互相兼容的
  https://juejin.cn/post/6844903666332368909
  意向锁不会与行级的共享 / 排他锁互斥

* https://developer.aliyun.com/article/890201
在《MySQL实战45讲》中作者给行锁加锁规则总结了“两个原则”、“两个优化”和“一个bug”（MySQL版本：5.x系列<=5.7.24，8.0系列 <=8.0.13），其中的“一个bug”就是唯一索引上的范围查询会访问到不满足条件的第一个值为止，也就是在上面的例子中虽然扫描到了id=12的索引，还要继续向后扫描，所以还要对supremum pseudo-record加邻键锁

* insert有唯一鍵時，若遇到重複的會產生在 supremum pseudo-record的排他鎖(或是臨鍵鎖)，造成其他的insert堵塞
  ps: 自己跟自己重複也一樣，會產生 supremum pseudo-record的排他鎖
  ps: 複合主見時，沒遇到這狀況，似乎只有單欄位的唯一鍵會有這問題

* 使用`lock tables [table_name] WRITE;`在_performance_schema_看不到
  要下`SHOW OPEN TABLES` 才可以看到鎖住的表

* innodb_lock_wait_timeout：innodb的dml操作的行级锁的等待时间
  lock_wait_timeout：数据结构ddl操作的锁的等待时间
