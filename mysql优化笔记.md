1、awk简洁入门
Awk是一个简便的直译是的文本处理工具
擅长处理 - 多行多列的数据

处理过程：
while(还有下一行){
1.读取下一行，并吧下一行赋给$0,各列赋给$1,$2......$N变量
2.用指定的命令来处理该行
}

如何处理1行数据？
答：分为两部分 。 pattern（条件）+ action（处理动作）

//awk 查看mysql运行状态过滤显示必要参数
例子：  

所需的数据   Queries   Threads_connected     Threads_running 

通过 mysqladmin  -uroot ext  查看当前mysql运行状态
通过 awk 命令显示当前必要的数据

mysqladmin -uroot ext | awk '/Queries/{printf("%d",$4)}/Threads_connected/{printf("%d",$4)}/Threads_running /{printf("%d",$4)}'

简洁操作 通过赋值后一次性输出所有值
mysqladmin -uroot ext | awk '/Queries/{q=$4)}/Threads_connected/{c=$4}/Threads_running /{r=$4}END{printf("%d %d %d\n",q ,c,r)}'

awk 行数相减
awk '{q=$1-last;last=$1,$}{printf("%d %d %d\n", q,$2,$3)}'

2、观察服务器周期变化

设计实验
总数据3W以上，50个并发，每秒请求500-1000次
请求结果缓存在memcache，生命周期为5分钟
观察mysql连接数，没面请求数的周期变化

实验素材
1.index.php (随机访问3W条热数据，并储存在memcached中)
2.memcache（储存查询结果）
3.ab压力测试工具
----------------------------------------------------------------------------------------------------------------------------
index.php 
/*
模拟随机选取3W条热数据
去除后储存在memcached
生命周期为5分钟
同事，调整ab参数，尽量在1分钟内完成缓存创建
*/
$id = 13300000 + mt_rand(0,30000);
$sql = " select id,name,brief,address from lx_com where id=".$id;
$mem = new memcache();
$mem = pconnect('localhost');
if( ($com = $mem->get($sql)) === false ){
$conn = mysql_connect('127.0.0.1','root','123456');
mysql_query('use big_data',$conn);
mysql_query('set names utf8' , $conn);

$rs = mysql_query($sql,$conn);
$com = mysql_fetch_assoc($rs);
$mem->add($sql,$com,false,30);
} 
print_r($com);
----------------------------------------------------------------------------------------------------------------------------
创建 shell 脚本 监控服务器被请求的变化 
vi tjstatus.sh   

#!/bin/bash
while true 
do 
mysqladmin -uroot ext | awk '/Queries/{q=$4)}/Threads_connected/{c=$4}/Threads_running /{r=$4}END{printf("%d %d %d\n",q ,c,r)}'  >> status.txt
sleep 1
done
>    覆盖文件中的数据
>>  将数据追加到文件中

启动该脚本
sh tjstatus.sh  

查看 文件
more status.txt

----------------------------------------------------------------------------------------------------------------------------
//ab测试命令
/usr/local/httpd/ab -c 50 -n 200000 http://192.168.1.201/mysqltest/index.php
-c 代表并发数
-n 代表请求数

将获取到的数据今天处理
awk '{q=$1-last;last=$1,$}{printf("%d %d %d\n", q,$2,$3)}'
然后生成图表

3、观察mysql进程状态
mysql -uroot -e ' showprocesslist';


创建 shell 脚本 监控服务器被请求的变化 
vi tjprocess.sh   

#!/bin/bash
while true 
do 

mysql -uroot -e 'show processlist\G'|grep Starte:|uniq -c|sort -rn >> process.txt

usleep 100000 
#每秒执行 10 次
done


启动该脚本
sh tjprocess.sh  

查看 文件
more process.txt

需要用sysbench来做测试
./bin/sysbench --test=oltp --mysql-table-engine=innodb -- mysql-user=root --db-driver=mysql --mysql-db=test --oltp-table-name=t1 --oltp-table-size=3000 --mysql-socket=/var/lib/mysql/mysql.sock run

查看结果

more process.txt | uniq -c|sort -rn 

过滤多余的行
more process.text  | sort |uniq -c| sort -rn

尽可能避免红色框中的内容

★ 值得注意的mysql进程状态 (要避免这些状态)
converting HEAP to MyISAM 查询结果太大是，把结果放在磁盘  （tmp_table_size   过小会出现）
create tmp table  创建临时表（如group时储存中间结果）
Copying to tmp table on disk 把内存临时表复制到磁盘
locked 被其他查询锁住
logging slow query 记录慢查询
注：把临时表内存变现，重现前

mysql> show variables like '%size%';
tmp_table_size   临时表的大小
修改方法  set session tmp_table_size = 1024;
mysql> set profiling = 1;
mysql> show profiling ;
empty set (0.00 sec)

mysql> select * from lx_com limit 10000;
mysql> show profiling ;
//生成分析结果
mysql> show profile for query 2;


什么情况下产生临时表？
1.group by 的列和order by的列不同是，2表边查时，取A表的内容，group/order by另外表的列
2.distinct 和 order by 一起使用时
3.开启了 SQL_SMALL_RESULT 选项

4、profiling的用法
查看profiling是否开启
show variables like '%profiling%';

开启profiling
set profiling =on;
查看当前执行的语句
show profiles;

查看具体某一天语句的解释
show profile for query 8;

★ 出现以下三种情况 说明sql语句有问题  可以改进
converting HEAP to MyISAM 查询结果太大是，把结果放在磁盘  （tmp_table_size   过小会出现）
create tmp table  创建临时表（如group时储存中间结果）
Copying to tmp table on disk 把内存临时表复制到磁盘

------------------------------------------------索引优化--------------------------------------------------------

5、表的优化与列类型选择
列选择原则
1、字段类型优先级： 整型 > date，time > char,varchar > blob
原因：整型，time运算快，节省空间
char/varchar 要考虑字符集的转换与排序时的校对集，速度慢
Blob无法使用内存临时表

2.够用就行，不要慷慨（如smalint，varchar(N)）
原因：大的字段浪费内存，影响速度
以varchar(10)，varchar(300)存储的内容相同，但是在表联查是，varchar(300)要花更多内存

3尽量避免用NULL();
原因：NULL不利于索引，要用特殊的直接来标注
在磁盘上占据的空间其实更大

实验：
可以建立2张字段相同的表，一个允许null,一个不允许为null，各加入1万条，查询索引文件的大小，可以发现，为null的索引要大些；
/*
create table t3(
	name char(1) not null default '',
  key(name)
)engine myisam charset utf8;
create table t4(
	name char(1) ,
  key(name)
)engine myisam charset utf8;
insert into t3 values('a'),('');
insert into t4 values('a'),(null);

explain select * from t3 where `name`='a';

explain select * from t4 where `name`='a';

#null 会占用一定的磁盘空间  并且查询值为null的select的写法为
explain select * from t4 where `name` is null;
*/

enum（枚举）列的说明
1、enum列在内部是用整型来存储的
例子：
create table t5(
	gender enum('male','female') not null default 'male'
)engine myisam charset utf8;

insert into t5 values('male'),('female');

select gender from t5;

select gender+0 from t5;

2、enum列与enum列相关联速度最快
3、enum列比char(varchar)的弱势 -- 在碰到与char关联是，要转化，要花时间
4、优势在于，当char非常长时，enum已让是整型固定长度，当查询的数据量越大时，enum的优势就越明显
5、enum与char/varchar关联，因为要转化，速度要比enum->enum，char->char要慢，
但是有时也这样用 ----- 就是在数据量特别大时，可以节省IO。

列<-------->列时间enum <----> enum10.63char <----> char24.65varchar <----> varhcar24.04enum <----> char18.22
6.再做测试   enum->enum,char<---->char,enum<---->char,精准的查询1条，再做比较

总结：enum和enum类型关联速度比较快
6、多列索引生效规则
索引优化策略
1、索引类型
1.1B-tree索引
注：名叫btree索引，大的方面看，都用在平衡树，但具体的实现上，个引擎稍有不同
    比如，严格的说NDB引擎，使用的是T-tree
MyISAM，innodb中，默认用B-tree索引

抽象一下 --- B-tree系统，可理解为"排好序的快速查找结构"，

1.2hash索引
在memory表中，默认是hash索引，hash的理论查询时间复杂度为O(1);
疑问：既然hash的查找如此快，为什么不都用hash索引？
答：
1.hash函数计算后的结果，是随机的，如果是磁盘上放置数据，比主键为ID为例，那么随着id的增长，id对应的行在磁盘上随机放置。
2.无法对范围查询进行优化
3.无法利用前缀索引，比如在btree中，field列的值"helloworld",并加上索引，查询xx=helloworld，自然可以利用索引，xx=hello，也可以利用索引.(左前缀索引)
应为hash("helloworld")和hash("hello")两者的关系仍为水晶
4.排序也无法优化
5.必须回行，就是说，通过索引拿到数据位置，必须回到表中取数据

2、btree索引的常见误区
2.1在where条件常用的列上都加上索引
例：where cat_id-3 and price>100 ;//查询第3个栏目，100元以上的商品
误：cat_id上和price上都加上索引
错：只能用上cat_id或price索引，因为是独立的索引，同时只能用上1个

注：在mysql5.6以上做了改进，允许两个索引在一定程度上合并

2.2在多列上建立索引后，查询那个列，索引都将发挥作用
误：多列索引上，索引发挥左右，需要满足左前缀要求
左前缀要求:必须从左到右层层使用否则索引就不发挥作用
以index(a,b,c)为例
语句索引是否发挥作用where a=3是where a=3 and b=5是where a=3 and b=5 and c=4是where b=3 | where c=4否where a=3 and c=4a列能发挥索引，c不能where a=3 and b>10 and c=7a能利用，b能利用，c不能利用同上，where a=3 and b like “xxxx%” and c=7a能利用，b能利用，c不能利用

7、多列索引实验

假设某个表有一个联合索引（c1,c2,c3,c4）一下——只能使用该联合索引的c1,c2,c3部分
A where c1=x and c2=x and c4>x and c3=x
B where c1=x and c2=x and c4=x order by c3
C where c1=x and c4= x group by c3,c2
D where c1=? and c5=? order by c2,c3 
E where c1=? and c2=? and c5=? order by c2,c3

有谁知道下面A-E能否可以使用索引！！为什么？
A   c1,c2,c3,c4能利用索引
B   c1,c2,c3,c4能利用 (c3的索引用于排序)
D 

若将语句修改成
sql : explain select * from t6 where c1='a' and c5='a' order by c3,c2
该语句将会使用到Using filesort

E  c1,c2c3都利用,c5不能利用

sql例子：select *　from t6 where c1='a' and c2='b' and c4='a' order by c5 asc;
Using filesort：二次排序  这样的话就有些索引不被使用
Using where： 没有使用到排序

  select_type       搜索类型  simple 简单查询 
  table                 表
  range                范围查询
  possible_keys   可能使用到的索引
  key                    键
  key_len  	   索引键的长度 （UTF8字符的长度为3个）
  ref      
  rows     		    行数  （扫描的行数）
  Extra                  外键     Using index condition  使用索引条件

create table t6(
	c1 char(1) not null default '',
	c2 char(1) not null default '',
	c3 char(1) not null default '',
	c4 char(1) not null default '',
	c5 char(1) not null default '',
	key(c1,c2,c3,c4)
) ENGINE MyISAM charset utf8;

insert into t6 values('a','b','c','d','e'),('A','b','c','d','e'),('a','B','c','d','e'),('a','b','C','d','e');
8、复合索引（联合索引）和聚簇索引和索引覆盖
问题：
 create table A(
id varchar(64) primary key,
ver int,
......
)
在id、ver上有联合索引
10000条数据 
为什么  select id from A order by  id 特别慢？
而select id from A order by id ,ver 非常快
我的表有几个很长的字段 varbinary(3000)

复合索引（联合索引）：索引可以包含一个、两个或更多个列。两个或更多个列上的索引
聚簇索引
索引覆盖

innodb和MyISAM索引的区别：
1、innodb的次索引指向对主键的引用
2、MyISAM的主索引和次索引 都指向物理行




innodb的主索引文件上 直接存放该行数据，称为聚簇索引，次索引指向对主键的引用
myisam中，主索引和次索引都指向物理行

注意：innodb来说
1.主键索引，既储存索引值，又在叶子中储存了行数据
2.如果没有主键，则会Unique key 做主键
3.如果没有unique,则系统会生成一个内部的rowid做主键
4.想innodb中，主键的索引结构中，既储存了主键值，有储存了行数据，这种结构称为“聚簇索引”

聚簇索引随机主键值的效率
 
高性能索引策略
对于innodb而言，因为节点下有数据文件因此节点的分裂将会比较慢
对于innodb的主键，尽量用整型，而且是递增的整型
如果是无规律的数据，将会产生页的分裂。影响速度


索引覆盖
索引覆盖式指 如果查询 的列签好是索引的一部分，那么查询只需要在索引文件上进行，不需要回行到磁盘再找数据
这种查询非常快，称为“索引覆盖"（Using index）

注：查索引是快的而回行是慢的   有索引->磁盘取数据
                                                               (回行)           

9、索引长度和区分索引


高性能索引策略
对于innodb而言，因为节点下有数据文件因此节点的分裂将会比较慢
对于innodb的主键，尽量用整型，而且是递增的整型
如果是无规律的数据，将会产生页的分裂。影响速度

理想的索引
1.查询频繁  2.区分度高  3.长度小 4.尽量能覆盖常用的查询字段


1.索引长度直接影响索引文件的大小，影响增删改的速度，并间接影响查询速度（占用内存多）

针对烈火中的值，从左往右截取部分，来建索引
1.截的越短，重复度越高，区分度越小，索引效果越不好
2.截的越长，重复度越低，区分度越高，索引效果越好，但带来的影响越大 -- 增删改变慢，并影响查询速度
所以我们要在  区分度 + 长度  两者间取一个平衡

惯用手法：截取不同长度，并测试其区分度


 10、为哈希函数降低索引长度 -- 索引长度越小索引速度越快
如 url列
http://www.baidu.com
http://www.zixue.it
列的前11个支付都是一样的，不易区分，可以用如下2个方法来解决
1：把列表内容倒过来存储，并建立索引
moc.udiad.www//:ptth
ti.euxiz.www//:ptth
这样左前缀区分度大

2.伪hash索引效果
同时存url hash列
例子：
为了让索引达到最优，我们可以通过为该数据行建立一个辅助列，来提高所以效率，并通过crc32进行转换保存到辅助列
比如存在一张url表表结构为
idurlcrcurl（辅助列）1http://www.baidu.com35002658942http://www.zixue.it38842768983http://www.51job.com442216211如果查找单独的一行数据，若不适用伪哈希来查找数据，并为该crcurl列建立索引.
但数据量越大，查询的效果越明显


create table t8(
 id int PRIMARY key auto_increment,
  url varchar(30) not null default ''
)engine myisam charset utf8;

insert into t8 (url) values('http://www.51job.com');

alter table t8 add crcurl int unsigned not null default 0;

select * from t8 ;

update t8 set crcurl=crc32(url);

alter table t8 add index url(url(16));

alter table t8 add index crcurl(crcurl);

EXPLAIN select * from t8 where url="http://www.baidu.com";

select CRC32('http://www.51job.com');

EXPLAIN select * from t8 where crcurl=CRC32('http://www.51job.com');
未使用辅助列直接在url字段建立索引搜索出来的分析结果
EXPLAIN select * from t8 where url="http://www.baidu.com";


建立辅助列（伪哈希）分析的结果
EXPLAIN select * from t8 where crcurl=CRC32('http://www.51job.com');



11、大数据量分页优化 （延迟关联）

业务场景：大数据下的分页效果

做法：
select field from table limit 30,10
 比如 每页显示perpage条，当前是第N页
 
 （N-1）*perpage
  ... limit (N-1)*perpage,N

优化方案
1.从业务上去解决
办法：不允许翻过100页
以百度和google为例
百度一般翻页到70页左右，google一般为40页左右

2.不用offset，用条件查询
mysql> select id,name from lx_com limit 5000000,10;
+---------+--------------------------------------------+
| id      | name                             |
+---------+--------------------------------------------+
| 5554609 | 温泉县人民政府供暖中心            |
..................
| 5554618 | 温泉县邮政鸿盛公司              |
+---------+--------------------------------------------+
10 rows in set (5.33 sec)
 
mysql> select id,name from lx_com where id>5000000 limit 10;
+---------+--------------------------------------------------------+
| id      | name                             |
+---------+--------------------------------------------------------+
| 5000001 | 南宁市嘉氏百货有限责任公司                |
.................
| 5000002 | 南宁市友达电线电缆有限公司               |
+---------+--------------------------------------------------------+
10 rows in set (0.00 sec)
问题：2次查询结果不一致
原因：数据被物理删除，有空洞
解决：数据不进行物理删除（可以逻辑删除）
最终在页面上显示数据时，逻辑删除的条目不显示即可
（一般来说，大网站的数据都不是物理删除，只做逻辑删除，比如is_delete=1）

3.非要用物理删除，还要用offset精准查询，还不限制用户分页，怎么办？
分析：优化思路是 不查，少查，查索引，少取
我现在必须要擦，则只查索引，不查数据，得到ID
在用ID去查具体条目，这种技巧就是延迟索引（延迟关联）
mysql > select lx_com.id,name from lx_com inner join (select id from lx_com where id>5000000 limit 10 ) as tmp on lx_com.id = tmp.id；
+---------+-----------------------------------------------+
| id      | name                                          |
+---------+-----------------------------------------------+
| 5050425 | 陇县河北乡大谈湾小学                |
........
| 5050434 | 陇县堎底下镇水管站                   |
+---------+-----------------------------------------------+
10 rows in set (1.35 sec)

12、索引与排序
排序可能发生2种情况：
1.对覆盖索引，直接在索引上查询时，就是有顺序的，using index
2.先出去数据，形成临时表做filesort（文件排序，但文件可能在磁盘上，也可能在内存中）

争取目标 --- 取出来的数据本身就是有序的！利用索引来排序

比如：goods商品表，（cat_id,shop_price）组成联合索引，
where cat_id=N order by shop_price,可以利用索引来排序，
select good_id,cat_id,shop_price from goods order by shop_price;
//using where 按照shop_price索引取出来的结果，本身就是有序的

select good_id,cat_id,shop_price from goods order by click_count;
//using filesort 用到了文件排序，即取出的结果再次排序

13、重复索引与冗余索引
重复索引：是指 在同一个列中（如age），或者顺序相同的几个列（age,school）,建立了多个索引，称为重复索引，重复索引没有任何班组，只会增大索引文件，拖慢更新速度。
冗余索引：是指两个索引所覆盖的列有重叠，称为冗余索引
比如 x,m列，加索引 index x(x) ,index xm(x,m)
x,xm索引，两者的x列重叠了，这种情况称为冗余索引
甚至可以把 index mx(m,x) 索引也建立，mx,xm也不是重复的，因为列的顺序不一样

14、索引碎片与维护
在长期的数据更改过程中，索引文件和数据文件，都会产生空洞，形成碎片。
我们可以通过一个nop操作(不产生对数据实质影响的操作)【该操作应该在夜里执行】，来修改表
比如：表的引擎为innodb，可以
alter table xxx(表名) engine innodb

optimize table 表名 ，也可以修复
optimize table xxx(表名)；

注意：修复表的数据及索引碎片，就会把所有的数据文件重新整理一边，使之对齐。
这个过程，如果表的行数比较大，也非常耗费资源的操作
所以不能频繁的修复

如果表的update操作很频繁，可以按周/月，来修复
如果不频繁，可以更长的周期来修复。

---------------------------------------------------sql语句优化-------------------------------------------------

15、explain 分析sql效果

sql语句慢在哪里  ?
1.等待时间 （连接数，内存 ）
2.执行时间
  a.取出了多少数据
  b.扫描了多少行

重构查询
1.切分查询   --> 操作过多行的分解成多次 （比如三张千万级别的表你要连接查询，还不如分三次查询）
2.分解查询   --> 多表联查拆成单表
----------------------------------------------------------------------------------------------------------------------------

id(查询的编号)                                                                                                                                   
select_type （查询类型）                                                                                                                    
table （表名|表别名|derived【如from型子查询时】|null【直接计算结果，不用走表】）                                                                                                                     
type (查询的方式非常重要，是分析“查数据”的重要过程)
【
all            代表全盘扫描            
index       代表扫描所有的索引节点相当于 index_all
------------------------------------------------------------------------------------------------------------
rang         范围查询 
ref            通过索引列，介意直接引用到某行数据
eq_ref
const     system  null  这3个分别指查询优化到常量级别，甚至不需要查找时间
一般按照主键来查询时，易出现 const,system
或者直接查询某个表达式，不经过表，出现null
】
possible_keys (可能用到的索引)
注意：系统估计可能用的几个索引，但最终，只能用1个                                                                                                                    
key (最终用的索引)                                                                                                                                                                                                                                                     
key_len(使用的索引的最大长度)                                                                                                            
ref(连接查询时，前表与后表的引用关系)
rows(估计扫描的行数)
Extra【using index | using where | using temporary | using filesort | range checked for each record 】
误区：1.explain 不产生查询   
           2.对limit的解释  
           3.explain即则执行过程
5.6变化：1.确实不产生查询
                2.允许解释非select语句



16、in型子查询陷阱
题：在ecshop商城中，查询6号栏目的商品（注：6号是一个大栏目）
最直观的是 select good_id,cat_id,good_name form ecs_goods where cat_id in(select cat_id form ecs_category where parent_id=6);
误区：给我们的感觉是先擦内层的6号栏目的子栏目，如 7,8,9，11
然后外层，cat_id(7,8,9,11)

事实：如下图，goods表全扫面，并逐行与category表对照，看parent_id=6是否成立


原因：mysql的查询优化器，针对In型做了优化，被改成了exists的执行效果
当goods表越大时，查询速度越慢

改进：用连接查询来代替子查询
explain select g.goods_id,g.cat_id,f.goods_name from ecs_goods as g inner join (select cat_id from ecs_category where parent_id=6) as t  on t.cat_id=g.cat_id

17、exists查询

18、min与max的优化
min/max优化 在表中,一般都是经过优化的. 如下地区表
idareapid1中国02北京1...31153113
 
我们查min(id), id是主键,查Min(id)非常快.
 
但是,pid上没有索引, 现在要求查询3113地区的min(id);
 
select min(id) from it_area where pid=69;  
 
试想 id是有顺序的,(默认索引是升续排列), 因此,如果我们沿着id的索引方向走,
那么  第1个 pid=69的索引结点,他的id就正好是最小的id.
select  id  from it_area use index(primary) where pid=69 limit 1;
 
|       12 | 0.00128100 | select min(id) from it_area where pid=69                         |
|       13 | 0.00017000 | select id from it_area  use index(primary) where pid=69  limit 1 |
 
改进后的速度虽然快,但语义已经非常不清晰,不建议这么做,仅仅是实验目的.


19、count() 优化
误区:
1:myisam的count()非常快
答: 是比较快,.但仅限于查询表的”所有行”比较快, 因为Myisam对行数进行了存储.
一旦有条件的查询, 速度就不再快了.尤其是where条件的列上没有索引.
 
2: 假如,id<100的商家都是我们内部测试的,我们想查查真实的商家有多少?
select count(*) from lx_com where id>=100;  (1000多万行用了6.X秒)
小技巧:
select count(*) from lx_com; 快
select count(*) from lx_com where id<100; 快
select count(*) frol lx_com -select count(*) from lx_com where id<100; 快
select (select count(*) from lx_com) - (select count(*) from lx_com where id<100)

3: group by
注意:
1:分组用于统计,而不用于筛选数据.
比如: 统计平均分,最高分,适合, 但用于筛选重复数据,则不适合.
以及用索引来避免临时表和文件排序
 
2:  以A,B表连接为例 ,主要查询A表的列,
那么 group by ,order by 的列尽量相同,而且列应该显示声明为A的列
 
4: union优化
注意: union all 不过滤 效率提高,如非必须,请用union all
因为 union去重的代价非常高, 放在程序里去重.



