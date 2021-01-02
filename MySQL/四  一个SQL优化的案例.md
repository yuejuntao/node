[TOC]

#### 一  当前情况

在10.38.161.168当前数据库上执行SQL：

````sql
select SQL_NO_CACHE *
from new_app
where level1_category_id = 15
  and level2_category_id = 19
order by app_id desc
limit 1000,100;
````

执行结果如下：

![image-20201208173106014](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208173106014.png)

用了3秒多，查看执行计划如下：

![image-20201208173159474](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208173159474.png)

![image-20201208173307678](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208173307678.png)

![image-20201208173339016](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208173339016.png)

发现，虽然查询的SQL中level1_category_id是第一个条件，但是这里用到的是pk_level2_category_toplist_test这个索引，即根据level2_category_id的索引树进行二分查找，使用了磁盘排序，并根据where条件返回需要的数据，可能需要遍历一万多条数据。

如果强制使用level1_category_id的索引，执行结果如下：

![image-20201208173814421](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208173814421.png)

发现用了十几秒才返回结果，查看执行计划如下：

![image-20201208173907434](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208173907434.png)

![image-20201208173949597](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208173949597.png)

![image-20201208174005440](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201208174005440.png)

可以看出使用了level1_category_id相关的索引，也需要进行磁盘排序，但是可能需要遍历接近29万条数据，因此在优化器生成执行计划时，判断出使用level2_category_id相关的索引的执行效率更高，因此优化为使用了level2_category_id相关的索引。

MySQL负责生成执行计划的查询优化器，一般会选则在索引里扫描行数比较少的那个条件。

但是，就算使用level2_category_id相关的索引，仍然需要几秒才能返回，怎么优化呢？

#### 二  修改表结构

````sql
SHOW CREATE TABLE new_app;
````

````sql
CREATE TABLE `new_app` (
  `app_id` bigint(20) NOT NULL,
  `package_name_hash` char(32) NOT NULL,
  `level1_category_id` int(11) NOT NULL,
  `level2_category_id` int(11) NOT NULL,
  `developer_type` tinyint(4) NOT NULL,
  `developer_id` bigint(20) NOT NULL,
  `update_time` bigint(20) NOT NULL,
  `create_time` bigint(20) NOT NULL,
  `status` int(11) DEFAULT '0',
  `package_name` varchar(200) NOT NULL,
  `display_name` mediumtext,
  `version_name` varchar(100) DEFAULT '',
  `locale` varchar(300) DEFAULT '["CN"]',
  `extras` mediumtext,
  PRIMARY KEY (`app_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
````

可以看出，存储引擎为innodb，app_id列为主键，因此会基于app_id列建立聚簇索引，并删除了所有其他索引。

#### 三 查询情况1

````sql
select SQL_NO_CACHE  * from new_app  where  level1_category_id = 15 and level2_category_id = 19 order by app_id desc limit 1000,100;
````

这条SQL说明，从new_app表里查询level1_category_id和level2_category_id为固定值，且以主键app_id降序的数据，从第1000位取100个数据。

执行结果如下：

![image-20201207173458058](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207173458058.png)

发现竟然用了13秒多，明显是一个慢SQL。查看执行计划如下：

````sql
EXPLAIN select SQL_NO_CACHE  * from new_app  where  level1_category_id = 15 and level2_category_id = 19 order by app_id desc limit 1000,100;
````

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207173654924.png" alt="image-20201207173654924" style="zoom:50%;" />

![image-20201207173740595](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207173740595.png)

从执行计划可以看出，type为index，即执行了索引的全部扫描，用到的索引的primary，即聚簇索引，Extra是Using where，也就是这时，会扫描聚簇索引，然后根据where条件筛选出符合的数据。

#### 四 优化1 - 建立单列索引

````sql
# 创建l1索引
CREATE INDEX L1_index ON new_app (level1_category_id);
````

此时上面的查询SQL执行结果如下：

![image-20201207174453844](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207174453844.png)

可以发现用了三秒多，再看下执行计划如下：

![image-20201207174615067](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207174615067.png)

![image-20201207174718718](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207174718718.png)

从执行计划可以看出，type是ref即使用了唯一的索引扫描，possible_keys和key都是L1_index，Extra为Using where，也就是这时会根据level1_category_id的辅助索引进行全盘扫描，需要扫描二十多万条数据，然后根据where条件筛选出需要的数据。

这时要说明的一点是，为什么全盘扫描辅助索引比聚簇索引要快，因为辅助索引是比较小，而聚簇索引的叶子节点存的是完整的数据，因此加载的磁盘IO比较大。

#### 五  优化2 - 建立联合索引

创建联合索引：

````sql
## 创建联合索引
CREATE INDEX l1_l2_index ON new_app (level1_category_id, level2_category_id);
````

执行结果如下：

![image-20201207180522493](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207180522493.png)

发现只用了500多毫秒，查看执行计划如下：

![image-20201207180616399](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207180616399.png)

![image-20201207180657385](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207180657385.png)

从执行计划可以看出，type是ref即使用了唯一的索引扫描，possible_keys是L1_index和l1_l2_index，但是key是l1_l2_index，Extra为Using where，也就是这时会根据l1_l2_index联合索引进行扫描，需要扫描1万多条数据，然后根据where条件筛选出需要的数据。

#### 六 查询情况2

从上面看，SQL执行速度在五百毫秒内，其实已经差不多了，这时如果有另一个SQL需要根据level2_category_id进行查询，因此建立了一个level2_category_id的单列索引：

````sql
# 创建l2索引
CREATE INDEX L2_index ON new_app (level2_category_id);
````

执行结果如下：

![image-20201207181620754](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207181620754.png)

发现加上l2索引后，执行速度竟然又涨到了三秒多，执行计划如下：

![image-20201207181831674](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207181831674.png)

![image-20201207181902544](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207181902544.png)

从执行计划可以看出，type是index_merge，possible_keys是L1_index、L2_index和l1_l2_index，但是key是L1_index和l1_l2_index，Extra为Using intersect(L2_index,l1_l2_index); Using where; Using filesort，也就是，这时会扫描L2_index和l1_l2_index两个索引，然后会取交集，然后根据磁盘进行排序，根据where条件筛选出需要的数据，扫描行数为九千多条。

所以如果同时查两个索引树取一个交集后，数据量很小，然后再回表到聚簇索引去查，此时理论上会提升性能。

#### 七 优化3 - 使用强制索引

使用强制索引：

````sql
# 这个sql会直接用l1_l2_index 执行比较快
select SQL_NO_CACHE  * from new_app force index (l1_l2_index)   where  level1_category_id = 15 and level2_category_id = 19 order by app_id desc limit 1000,100;
````

执行结果如下：

![image-20201207182531964](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207182531964.png)

查看执行计划如下：

![image-20201207182601419](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207182601419.png)

![image-20201207182624523](https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201207182624523.png)