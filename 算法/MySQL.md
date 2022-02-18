### 合并函数

#### union 与 union all

union：会去重并按照字典序排序

union all：不会去重，不会排序，直接合并返回

#### round

四舍五入保留小数：

```mys
select round(1123.26721,2);
1123.27
select round(1123.26721,-2);
1100
```

### Limit

第一个参数是输出记录的初始位置，第二个参数偏移量，偏移多少，输出的条目就是多少。

### 窗口函数

窗口函数原则上只能写在 select 子句中。主要用于处理分组 TOP N 问题。

```sql
<窗口函数> over (partition by <用于分组的列名> order by <用于排序的列名>)
```

>  group by 只保留分组后的第一条数据，所以通常用来求分组和、分组平均值、最大最小值等唯一数据。

#### 求排名的窗口函数

`RANK()` ：排序结果相同时会并列，总数不会变。

`DENSE_RANK()` ：排序排序相同时会并列，总数会减少。

`ROW_NUMBER()`： 排序相同时按照顺序计算，不考虑并列。

```sql
select *,
   rank() over (order by 成绩 desc) as ranking,
   dense_rank() over (order by 成绩 desc) as dese_rank,
   row_number() over (order by 成绩 desc) as row_num
from 班级表
```

<img src="/Users/licheng/Documents/Typora/Picture/image-20200801111847455.png" alt="image-20200801111847455" style="zoom:50%;" />

#### 聚合函数

```sql
select *,
   sum(成绩) over (order by 学号) as current_sum,
   avg(成绩) over (order by 学号) as current_avg,
   count(成绩) over (order by 学号) as current_count,
   max(成绩) over (order by 学号) as current_max,
   min(成绩) over (order by 学号) as current_min
from 班级表
```

<img src="/Users/licheng/Documents/Typora/Picture/image-20200801112219096.png" alt="image-20200801112219096" style="zoom:50%;" />

如上图，聚合函数 sum 在窗口函数中，是对自身记录、及位于自身记录以上的数据进行求和的结果。比如 0004 号，在使用 sum 窗口函数后的结果，是对 0001，0002，0003，0004 号的成绩求和，若是 0005 号，则结果是 0001 号 ~ 0005 号成绩的求和，以此类推。

不仅是 sum 求和，平均、计数、最大最小值，也是同理，都是针对自身记录、以及自身记录之上的所有数据进行计算。

聚合函数作为窗口函数，可以在每一行的数据里直观的看到，截止到本行数据，统计数据是多少（最大值、最小值等）。同时可以看出每一行数据，对整体统计数据的影响。

**如果存在就累加，不存在就插入数据**

```sql
INSERT INTO `weblogs' (
	'name',
	'count'
)
VALUES(
    '四川地震',
    '1'
  ) ON DUPLICATE KEY UPDATE `count`= `count` + VALUES(`count`);
```

on 和 where 的区别

在 inner join 时没有区别

在 left join 时，on 中的条件只对右表生效，where 中的条件对所有表都生效。