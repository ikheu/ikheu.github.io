---
layout: post
title: MySQL Crash Course 命令总结
categories: 数据库
tags: MySQL
---
{:toc}

经常不写 SQL 语句容易生疏，所以这里总结了 MySQL 的命令，用于快速回顾。所有示例摘录自 [MySQL Crash Course](https://forta.com/books/0672327120/) 一书。

## 数据表

使用网站提供的 SQL 文件，建立数据库 crashcourse，它包含以下表：

```
mysql> show tables;
+-----------------------+
| Tables_in_crashcourse |
+-----------------------+
| customers             |
| orderitems            |
| orders                |
| productnotes          |
| products              |
| vendors               |
+-----------------------+
```

使用 Workbench 生成该数据库的 EER 图：

![](/assets/img/eer.svg){:width="100%"}


## 检索数据

`select` 命令的通用形式如下：

``` sql
select 选择字段
from 从某个表
where 行过滤
group by 分组
having 分组过滤
order by 结果排序
limit 限制集合条数
```

简单的检索命令：

```sql
# 选择一列
select prod_name
from products;

# 选择多列
select prod_id, prod_name, prod_price 
from products;

# 选择所有列
select * 
from prodcucts;

# 去重
select distinct vend_id 
from products;

# 限制条数
select prod_name 
from products 
limit 5;

select prod_name 
from products 
limit 3, 5;

# 使用完整名称
select products.prod_name 
from crashcourse.products;

# 排序
select prod_name
from products
order by prod_name;

# 排序多个列
select prod_id, prod_price, prod_name
from products
order by prod_price, prod_name;

# 指定顺序
select prod_id, prod_price, prod_name
from products
order by prod_price desc, prod_name asc;
```

## 过滤数据

```sql
# 等于
select prod_name, prod_price
from products
where prod_price = 2.5;

# 等于
select prod_name, prod_price
from products
where prod_name = 'fuses';

# 小于
select prod_name, prod_price
from products
where prod_price < 10;

# 不等于
select prod_name, prod_price
from products
where vend_id <> 1003;

# 范围
select prod_name, prod_price
from products
where prod_price between 5 and 10;

# 空值检查
select prod_name, prod_price
from products
where  prod_price is null;

# 组合 where 子句
select prod_id, prod_price, prod_name
from products
where vend_id = 1003 and prod_price <= 10;

# 组合 and 和 or
select prod_id, prod_price, prod_name
from products
where vend_id = 1002 or vend_id = 1002 and prod_price >= 10;

# in 操作
select prod_name, prod_price
from products
where vend_id in (1002, 1003);

# not 操作
select prod_name, prod_price
from products
where vend_id not in (1002, 1003);
```

## 通配符过滤

```sql
# 以某子串开头
select prod_name, prod_price
from products 
where prod_name like 'jet%';

# 包含某子串
select prod_name, prod_price
from products 
where prod_name like '%anvil%';

# 匹配单个字符 _
select prod_name, prod_price
from products 
where prod_name like '_ ton anvil'; 

```

## 正则表达式

```sql
# 包含
select prod_name  
from products  
where prod_name regexp '1000';

# . 匹配单个字符
select prod_name  
from products  
where prod_name regexp '.000';

# or 匹配
select prod_name  
from products  
where prod_name regexp '1000|2000';

# 匹配几个字符之一
select prod_name  
from products  
where prod_name regexp '[1-5] Ton';

# 匹配特殊字符
select prod_name  
from products  
where prod_name regexp '\\.[1-5] ton';

# 综合
select prod_name  
from products  
where prod_name regexp '\\([0-9] sticks?\\)';

select prod_name  
from products  
where prod_name regexp '[[:digit:]]{4}';

# 定位符
select prod_name  
from products  
where prod_name regexp '^[0-9\\.]';
```

## 计算字段

```sql
# 拼接字段
select concat(vend_name, '(', vend_country, ')')
from vendors
order by vend_name;

# 去除空格
select concat(vend_name, '(', rtrim(vend_country), ')')
from vendors
order by vend_name;

# 使用别名
select concat(vend_name, '(', rtrim(vend_country), ')') as vend_title
from vendors
order by vend_name;

# 算术运算
select prod_id, quantity, item_price, quantity*item_price as expanded_price
from orderitems
where order_num = 20005;

# 文本处理
select vend_name, upper(vend_name) as vend_name_upcase
from vendors
order by vend_name;

# 日期处理
select cust_id, order_num, order_date
from orders
where date(order_date) = '2005-09-01';

# 日期范围
select cust_id, order_num, order_date
from orders
where year(order_date) = 2005 and month(order_date) = 9;
```

## 数据汇总

```sql
# avg, count, max, min, sum
# avg
select avg(prod_price) 
from products; 

# count，不忽略 null
select count(*)
from orders;

# count，忽略 null
select count(cust_email)
from customers;

# max
select max(prod_price) as max_price
from products;

# 聚集不同的值
select avg(distinct prod_price) as avg_price
from products
where vend_id = 1003;

# 组合聚集
select count(*) as total_num,
min(prod_price) as price_min,
max(prod_price) as price_max,
avg(prod_price) as price_avg
from products;
```

## 分组数据

```sql
# 创建分组
select vend_id, count(*) 
from products 
group by vend_id;

# 过滤分组，注意 where处理行，having处理分组
select cust_id, count(*) as orders
from orders
where cust_id in (10001, 10003)
group by cust_id
having count(*) >= 2;

# 综合
select order_num, sum(quantity * item_price) as ordertotal
from orderitems
where order_num in (20006, 20008)
group by order_num
having ordertotal >= 50
order by ordertotal;
```

## 子查询

```sql
# 从子查询中过滤
select *
from orders
where order_num in (
    select order_num
    from orderitems
    where prod_id = 'tnt2'
);

# 自查询作为字段
select *,
    (
        select count(*)
        from orders
        where orders.cust_id = customers.cust_id
    ) as orders
from customers;
```

## 联结查询

```sql
# 使用 where 联结
select vend_name, prod_name, prod_price
from vendors, products
where vendors.vend_id = products.vend_id
order by vend_name, prod_name;

# 使用内部联结
select vend_name, prod_name, prod_price 
from vendors inner join products 
    on vendors.vend_id = products.vend_id

# 更多表的联结
select cust_name, cust_contact
from customers, orders, orderitems
where customers.cust_id = orders.cust_id
and orderitems.order_num = orders.order_num
and prod_id = 'tnt2' ;

# 自联结
select p1.prod_id, p1.prod_name 
from products as p1, products as p2
where p1.vend_id = p2.vend_id
and p2.prod_id = 'DTNTR';

# 外部联结
select customers.cust_id, orders.order_num 
from customers left outer join orders 
on customers.cust_id = orders.cust_id;

# 带有聚合的联结
select customers.cust_name, customers.cust_id, count(orders.order_num) as num_ord
from customers inner join orders 
on customers.cust_id = orders.cust_id
group by customers.cust_id;
```

## 组合查询

```sql
# 组合
select vend_id, prod_id, prod_price
from products
where prod_price <= 5
union
select vend_id, prod_id, prod_price
from products
where vend_id in (1001, 1002);

# 不去重的组合，where 针对单个 select， order by 针对整个数据集
select vend_id, prod_id, prod_price
from products
where prod_price <= 5
union all
select vend_id, prod_id, prod_price
from products
where vend_id in (1001, 1002)
order by prod_price;
```

## 全文本查询

```sql
select *
from productnotes
where match(note_text) against('rabbit');
```

## 插入/更新/删除

```sql
# 插入完整的行
insert into customers 
values(null, 
    'Pep E. Lapew', 
    '100 Main Street', 
    'Los Angeles',
    'CA', 
    '90046', 
    'USA', 
    null, 
    null);

# 不完整，缺少的字段默认为 null，需要允许 default
insert into customers(
    cust_name,
    cust_address,
    cust_city,
    cust_state,
    cust_zip,
    cust_country
)
values(
    'Pep E. Lapew', 
    '100 Main Street', 
    'Los Angeles',
    'CA', 
    '90046', 
    'USA'),(
    'M. Martian', 
    '42 Galaxy Way', 
    'New York',
    'NY', 
    '11213', 
    'USA');

# 更新
update customers 
set cust_email='elmer@foo.email', cust_contact='T TL'
where cust_id = 10006;

delete from customers 
where cust_id = 10007;
```

## 操作表

```sql
# 查看表
show tables;

# 检查表的字段属性
show columns from customers2;

# 查看创建
show create table orders;

# 创建
create table customers (
    cust_id         int         not null auto_increment,
    cust_name       char(50)    not null,
    cust_address    char(50)    null,
    cust_city       char(50)    null,
    cust_state      char(50)    null,
    cust_zip        char(50)    null,
    cust_country    char(50)    null,
    cust_contact    char(50)    null,
    cust_email      char(50)    null,
    primary key (cust_id)
) engine=Innodb;

# 指定默认值
create table orderitems (
    order_num       int             not null,
    order_item      int             not null,
    prod_id         char(10)        not null,
    quantity        int             not null default 1,
    item_price      decimal(8, 2)   not null,
    primary key(order_num, order_item)
) engine=Innodb;

# 重命名
rename table customers1 to customers2;

# 定义外键
alter table orders
add constraint fk_orders_customers foreign key (cust_id)
references customers (cust_id);

# 删除表
drop table customers2;
```

## 使用视图

```sql
# 创建视图(虚拟的表，使用时动态查找获得数据。show tables 中也能查到)
create view productcustomers as
select cust_name, cust_contact, prod_id
from customers, orders, orderitems
where customers.cust_id = orders.cust_id
    and orderitems.order_num = orders.order_num;

# 使用视图
select * 
from productcustomers;

# 查看视图
desc productcustomers;
show create view productcustomers;
```

## 存储过程

```sql
# 创建存储过程
delimiter //
create procedure productpricing()
begin
select avg(prod_price) as priceaverage
from products;
end //

delimiter ;

# 调用存储过程
call productpricing();

# 删除
drop procedure productpricing;
```

# 使用游标

```sql
delimiter //
create procedure processorders()
begin
    declare done boolean default 0;
    declare o int;
    declare ordernumbers cursor
    for
    select order_num from orders;

    open ordernumbers;
    repeat
        fetch ordernumbers into o;
    until done end repeat;
    close ordernumbers;
end //
delimiter ;

```

## 触发器

```sql
create trigger newproduct after insert on products
for each row select 'Product added' into @res;
```

## 事务

```sql
# 回滚
start transaction;
delete from customers where cust_id = 10008;
rollback;

# 提交
start transaction;
delete from customers where cust_id = 10008;
commit;
```
