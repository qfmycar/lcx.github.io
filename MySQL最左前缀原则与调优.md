### MySQL最左前缀原则与调优

#### 一、最左前缀原则

```
	最左前缀原则就是使用复合索引时，只有满足最左边的条件时，才会依次向右进行查询
	
	原因：大数据量情况下，单列索引已经不能满足查询速度的要求，只能使用复合索引。
	意义：对大数据进行多层次筛选，达到在更小的数据量中搜索想要的数据的目的，毕竟在一千万条数据中跟一万条数据中查数据是不一样的效率。
	
	这里还要讲几个注意点：
	1.如果建立多个单列索引，那在查询时会使用其中结果集最少的索引
	2.建立复合索引ABC，那相当于还建立了索引 A, AB
	3.查询时如果只有BC，那会先遍历所有索引，找到复合索引相同的一部分，然后使用它，但它效率不高，在explain 的type上会显示为 index。若结构完全相同则会显示 ref
	4.如果书写顺序为CBA，在执行时MySQL查询优化器会将执行顺序调整为ABC
```



#### 二、MySQL调优

##### 1、合适的索引

```
	建立合适的索引，选择经常查询的字段，并且可以根据业务建立复合索引
```

##### 2、正确的SQL语句

```sql
	书写正确的SQL语句，可能防止无法使用索引的情况，这里举例以下几点：
1.尽量避免在索引列上使用IS NULL 与 IS NOT NULL：
  		不能用null作索引，任何包含null值的列都不会被包含在索引中，即使有复合索引，只要其中有一列含有		null，该列也会从索引中排除
  		任何 Where子句中使用 is null或 is not null的语句，都是不会使用索引的
2.联接列：
		对于有联接的列，是不会执行索引的，比如查询某个人的姓名：
		select * from man where firstName = 'lin' || lastName = 'cx';
		这个语句虽然可以查到结果，但不会执行lastName的索引。
		select * from man where firstName = 'lin' and lastName = 'cx';
		这个语句是能使用索引
3.%的 like语句：
		使用 like模糊查询时，%号在左边会使索引不生效，右边才生效，如果必要可以再设置一个字段，值取原字		段的反转值，然后同时在这个字段的左边加 %进行模糊查询
4.Order by：
		任何在 Order by使用的非索引列或者有计算表达式的，都将降低查询速度，应该避免此情况
5.Not：
		Not可以看做不等于运算符(<>)，使用这个会使索引失效，所以我们可以用大于小于运算符替换，如：
		select * from XXX where num <> 100;
		可以替换成	select * from XXX where num <100 and num >100;这样就能正确的使用索引
6.where子句中的连接顺序
		连接顺序应该符合复合索引的顺序，这样能经过多层筛选，在尽可能小的数据量中寻找需要的数据
7.select 子句中尽量避免用 *
8.使用DECODE函数避免重复扫描相同的记录或连接相同的表
9.整合简单的，无关联的数据库访问
10.定期删除重复记录
		DELETE FROM AAA a WHERE a.rowid > (SELECT MIN(x.rowid) FROM AAA x WHERE a.no = x.no);
11.使用 TRUNCATE代替DELETE
		用delete时，回滚段会存放可被恢复的信息；使用truncate时，回滚段不会存放任何信息，因此占用资源会		更少，速度更快
12.尽可能多的使用COMMIT
13.尽可能用 where代替 having
		where是在计算之前完成的， having是在计算之后才起作用的，所以尽量用 where，可以减少数据量
14.减少对表的查询
		在有子查询的SQL语句中，要尽量减少对表的查询，以及子语句中通过限定条件筛选出更少的数据量
15.尽可能使用内部函数提高SQL效率
16.使用表的别名，减少解析时间
17.使用EXISTS 代替 in， NOT EXISTS代替 NOT IN
		比如：SELECT * FROM X WHERE X.NO>0 AND NO IN(SELECT NO FROM Y WHERE Y.TYPE = 'SHOP')
		可以替换成
		SELECT * FROM X WHERE NO >0 AND EXISTS (SELECT NO FROM Y WHERE Y.NO = X.NO AND Y.TYPE = 	'SHOP')
18.使用EXISTS 代替 DISTINCT
		一般使用多表查询时，用EXISTS代替 DISTINCT会更快，语法如下：
		SELECT DISTINCT NO,NAME FROM A,B WHERE A.NO = B.NO 
		可以替换成
		SELECT NO,NAME FROM A WHERE (SELECT NO FROM B WHERE A.NO = B.NO)
19.SQL语句用大写
20.在Java代码总尽量避免用"+"连接字符串
21.避免在索引列上使用 NOT
22.避免在索引列上使用计算
		SELECT * FROM AAA WHERE POINT * 12 >100
		可以替换成
		SELECT * FROM AAA WHERE POINT > 100/12
23.使用 >= 代替 >
		SELECT * FROM AAA WHERE NO >3
		可以替换成
		SELECT * FROM AAA WHERE NO >=4
		区别在于，前者会先定位到no=3的记录，而后者直接跳到no=4的记录
24.使用 UNION代替 OR
25.使用EXISTS 代替 OR
26.使用复合索引的第一列
```
