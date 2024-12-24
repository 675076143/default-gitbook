---
description: 2023年6月15日
---

# Mysql 使用 Prepared Statements 导致的性能问题

### 一、起因

项目中发现一个接口很慢(30s) 左右，但SQL采样直接执行又只需要2s就能完成。Mysql版本: 8.0

### 二、背景知识

1. Prepared Statements [https://dev.mysql.com/doc/refman/8.0/en/sql-prepared-statements.html](https://dev.mysql.com/doc/refman/8.0/en/sql-prepared-statements.html)
2. Laravel QueryBuilder Binding 原理
3. PHP PDO 实现 [https://www.php.net/manual/en/book.pdo.php](https://www.php.net/manual/en/book.pdo.php)

### 三、开始

1. 使用 ORM 来执行一段这样查询：

```php
User::query()->whereIn('name', ['Robin', 'John']);
```

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

2. 这段语句会被转变 Prepared Statements 提交给 Mysql

```sql
PREPARE statement FROM 'select count(*) from users where name in (?,?)';
SET @1:='Robin',@2:='John';
EXECUTE statement USING @1,@2;
DEALLOCATE PREPARE statement;
```

3. Mysql 会将这句 Statement 解析后再执行

```sql
select count(*) from users where name in ('Robin','John')
```

这本是一件好事，Prepared Statements 可以帮我们回避掉SQL注入的问题。但当 Prepared Statement 过于复杂的时候，这个 SQL语句的执行耗时会成倍增长。

### 四、如何解决？

花了点时间查阅了下资料，我发现，PHP PDO 其实给我们一种模拟生成 Prepared Statement 的模式:[https://www.php.net/manual/zh/pdo.prepare.php](https://www.php.net/manual/zh/pdo.prepare.php)

```php
PDO::ATTR_EMULATE_PREPARES // true or false
```

通过这种方式，我们就能把解析 Prepared Statements 这个压力从DB层转移到服务层经过测试，原先需要30s才能完成的语句，现在也只要2s了，和直接执行原始sql时间几乎一致

### 五、测试

先决条件Mysql 8.0PHP 8.1Laravel 8.83.27业务表情况: 65W行, count全表 耗时 500ms

* 直接查询 where in 6000 个参数的情况，耗时 2s 左右

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

* 生成 Prepared Statement 后再去查询，耗时 30s 左右

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

* Laravel 不开启 ATTR\_EMULATE\_PREPARES 的情况，平均耗时 29.28s，使用QueryBuilder测试

```php
DB::connection('mysql')->enableQueryLog();
User::query()->whereIn('name', ['Robin','John',...6000]);
dd(DB::connection('mysql')->getQueryLog());
```

* Laravel 开启 ATTR\_EMULATE\_PREPARES 的情况，平均耗时2.1s，使用QueryBuilder测试

附laravel的database.php配置

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

### 六、深入

按理说，Statement 被 Prepared 后，性能应该是增加的，为什么反而降低了这么多呢？ 起初我以为是 Prepared 慢，没想到 真正慢的是 Execute部分。

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

Prepared Statement 不能用常规的 Explain/Analyze 来分析，这边我用了优化器看了下

```sql
-- 注意，由于SQL实在过长，optimizer的内存都不够用了，需要增大到2m，不然trace记录不全
SET optimizer_trace_max_mem_size=2048 * 1000;

SET optimizer_trace="enabled=on";
EXECUTE statement USING @0,@1,@2......;
select * from `information_schema`.`optimizer_trace`;
SET optimizer_trace="enabled=off";
```

左边是使用 PreparedStatement 的情况，右边是直接执行SQL的情况。PreparedStatement 执行时，优化器认为由于开销的原因，最优索引居然和直接执行不一致！

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

到这里，大概就能判断出这个很可能和mysql的优化器有关系了，我跑去翻了下mysql的commit记录，其中不乏一些因为优化器导致的Statement索引失效的问题。比如下面这个就是因为一个Order By导致优化器无法选择最优的索引：

{% embed url="https://github.com/mysql/mysql-server/commit/443384454fdbb365a20bfbf86f552ab914d1ea92#diff-16fd8e507582e428d933bc7ee3984394eab544410785220b536ab964d4e9e084L3671" %}

但找了一圈其实没有搜到能完美解释我这个问题的issue我的猜想, 不一定对：Prepared Statements 后，Execute Statement 其实输入的是变量了, 对比原始SQL，其实输入的字符串是确定的常量。

```sql
where in (?, ?) -- <---这是变量

where in ('Robin', 'John') -- <---这是常量
```

而 in 的本质是要用确定的常量命中索引，当我们传递过多的变量，优化器内部认为开销过大，就不选择这个索引了。\


### 七、Q\&A

* 问：ORM转变为 Prepared Statement 是 laravel 还是 pdo 干的？

答：是PDO干的，可以参考这边。https://www.php.net/manual/en/pdo.prepare.php

```php
// 你如果直接使用PDO来查询数据库的话，代码看起来就是这样的
$pdo = new PDO('host','user','password'); 
$stmt = $pdo->prepare("select * from users where name in (?,?)");
$stmt->execute(['Robin','John']);
```

* 问：所有的 ORM 框架都会转成 Prepared Statement 吗？

答：几乎是的，不过这些框架大多也提供类似laravel DB::row这样的写法，如 nodejs 就有一个 knex.raw()。这可以让你直接提交SQL原文本，而不是提交 Prepared Statement。\


### 八、参考链接

{% embed url="https://bugs.mysql.com/bug.php?id=73056" %}

{% embed url="https://stackoverflow.com/questions/6301598/mysql-prepared-statement-vs-normal-query-gains-losses" %}

{% embed url="https://stackoverflow.com/questions/63431703/is-prepared-statement-significantly-slower-than-single-query-for-large-number-of" %}
这篇讲述了开启 EMULATE 是否会导致安全问题 (不会)
{% endembed %}

{% embed url="https://stackoverflow.com/questions/10113562/pdo-mysql-use-pdoattr-emulate-prepares-or-not" %}

\
\
