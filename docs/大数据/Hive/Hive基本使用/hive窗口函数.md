# hive之窗口函数

### **窗口函数**

1．相关函数说明

```
COVER()：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变而变化
CURRENT ROW：当前行
n PRECEDING：往前n行数据
n FOLLOWING：往后n行数据
UNBOUNDED：起点，UNBOUNDED PRECEDING 表示从前面的起点， UNBOUNDED FOLLOWING表示到后面的终点
LAG(col,n)：往前第n行数据
LEAD(col,n)：往后第n行数据
```

NTILE(n)：把有序分区中的行分发到指定数据的组中，各个组有编号，编号从1开始，对于每一行，NTILE返回此行所属的组的编号。注意：n必须为int类型。 

2．数据准备：name，orderdate，cost 

```
jack,2017-01-01,10
tony,2017-01-02,15
jack,2017-02-03,23
tony,2017-01-04,29
jack,2017-01-05,46
jack,2017-04-06,42
tony,2017-01-07,50
jack,2017-01-08,55
mart,2017-04-08,62
mart,2017-04-09,68
neil,2017-05-10,12
mart,2017-04-11,75
neil,2017-06-12,80
mart,2017-04-13,94
```

3．需求

（1）查询在2017年4月份购买过的顾客及总人数

（2）查询顾客的购买明细及月购买总额

（3）上述的场景,要将cost按照日期进行累加

（4）查询顾客上次的购买时间

（5）查询前20%时间的订单信息

4．创建本地business.txt，导入数据

 vi business.txt

5．创建hive表并导入数据 

```
create table business(
name string,
orderdate string,
cost int
) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';
load data local inpath "/opt/module/datas/business.txt" into table business;
```

6．按需求查询数据

（1）查询在2017年4月份购买过的顾客及总人数

```
select name,count(*) over ()
from business 
where substring(orderdate,1,7) = '2017-04'
group by name;
```

（2）查询顾客的购买明细及月购买总额

```
select name,orderdate,cost,sum(cost) over(partition by month(orderdate)) from business;
```

（3）上述的场景,要将cost按照日期进行累加

```
select name,orderdate,cost,

sum(cost) over() as sample1,--所有行相加

sum(cost) over(partition by name) as sample2,--按name分组，组内数据相加

sum(cost) over(partition by name order by orderdate) as sample3,--按name分组，组内数据累加

sum(cost) over(partition by name order by orderdate rows between UNBOUNDED PRECEDING and current row ) as sample4 ,--和sample3一样,由起点到当前行的聚合

sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING and current row) as sample5, --当前行和前面一行做聚合

sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING AND 1 FOLLOWING ) as sample6,--当前行和前边一行及后面一行

sum(cost) over(partition by name order by orderdate rows between current row and UNBOUNDED FOLLOWING ) as sample7 --当前行及后面所有行

from business;
```

（4）查看顾客上次的购买时间 

```
select name,orderdate,cost,
lag(orderdate,1,'1900-01-01') over(partition by name order by orderdate ) as time1, lag(orderdate,2) over (partition by name order by orderdate) as time2
from business;
```

（5）查询前20%时间的订单信息 

```
select * from (
    select name,orderdate,cost, ntile(5) over(order by orderdate) sorted
    from business
) t
where sorted = 1;
```





数据：

```
id,月份，
A,01,02,18
A,01,03,20
A,01,04,50

A,02,09,10
A,02,13,20

A,03,05,20
A,03,08,30

B,01,02,20
B,02,05,30
B,02,08,40
```

累计

```
A,01,88,88
A,02,30,118
A,03,50,168
B,01,20,20
B,01,70,90


```

## 二.unbounded 

1.order表结构

```
MONTH       NUMBER ( 2 )
TOT_SALES   NUMBER 
```

2.数据如下

```
MONTH   TOT_SALES
-- ------------------- 
1       610697 
2       428676 
3       637031 
4       541146 
5       592935 
6       501485 
7       606914 
8       460520 
9       392898 
10       510117 
11       532889 
12       492458 
```

3.测试sql语句

```sql
select month,sum (tot_sales) month_sales,sum ( sum (tot_sales))  over  ( order   by   month  rows  between   1  preceding  and  unbounded  following) all_sales from  orders group   by   month ;
```

查询到的结果集：

```
MONTH    MONTH_SALES   ALL_SALES
1         610697        6307766 
2         428676        6307766 
3         637031        5697069 
4         541146        5268393 
5         592935        4631362 
6         501485        4090216 
7         606914        3497281 
8         460520        2995796 
9         392898        2388882 
10        510117        1928362 
11        532889        1535464 
12        492458        1025347 
```

需求2：列出每月的订单总额以及截至到当前月的订单总额。也就是说2月份的记录要显示当月的订单总额和1,2月份订单总额的和。3月份要显示当月的订单总额和1,2,3月份订单总额的和，依此类推

```sql
select month,sum(tot_sales) month_sales,sum(sum (tot_sales)) over (order by month rows between unbounded preceding  and current row) current_total_sales from orders group by month;
```

查询到的结果集：

```
MONTH   MONTH_SALES   CURRENT_TOTAL_SALES
1        610697         610697 
2        428676         1039373 
3        637031         1676404 
4        541146         2217550 
5        592935         2810485 
6        501485         3311970 
7        606914         3918884 
8        460520         4379404 
9        392898         4772302 
10       510117         5282419 
11       532889         5815308 
12       492458         6307766 
```

