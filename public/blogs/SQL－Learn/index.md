Mysql数据库语言学习记录：

# 聚合函数

在SQL中，聚合函数（Aggregate Functions）是用来对一组值执行计算，并返回单个值的函数。它们常用于数据分析，比如在数据库中对一组数据求和、平均值、最大值、最小值等。包括Count、Max、Min、Avg、Sum等。

# 非窗口聚合函数

非聚合窗口函数是相对于聚合窗口函数来说的。聚合函数是对一组数据计算后返回单个值（即分组），非聚合函数一次只会处理一行数据。窗口聚合函数在行记录上计算某个字段的结果时，可将窗口范围内的数据输入到聚合函数中，并不改变行数。

MySQL 支持的非聚合窗口函数如下：

| 名称           | 描述                               |
| -------------- | ---------------------------------- |
| CUME_DIST()    | 累积分配值                         |
| DENSE_RANK()   | 当前行在其分区中的排名，稠密排序   |
| FIRST_VALUE()  | 指定区间范围内的第一行的值         |
| LAG()          | 取排在当前行之前的值               |
| LAST_VALUE()   | 指定区间范围内的最后一行的值       |
| LEAD()         | 取排在当前行之后的值               |
| NTH_VALUE()    | 指定区间范围内第N行的值           |
| NTILE()        | 将数据分到N个桶，当前行所在的桶号 |
| PERCENT_RANK() | 排名值的百分比                     |
| RANK()         | 当前行在其分区中的排名，稀疏排序   |
| ROW_NUMBER()   | 分区内当前行的行号                 |

# Group By

基本用法：

```
select 列名1，... , 列名n 
from 表
where 条件
group by 列名1，... , 列名n
```

“Group By”从字面意义上理解就是根据“By”指定的规则对数据进行分组，所谓的分组就是将一个“数据集”划分成若干个“小区域”，然后针对若干个“小区域”进行数据处理。

Group By的字段顺序决定分组层次，当使用GROUP BY col1, col2时，MySQL会先按col1分组，再在每个col1分组内按col2细分。例如统计部门与岗位的最高薪资：

```
SELECT deptno, job, MAX(sal) 
FROM emp 
GROUP BY deptno, job;
```

# Distinct

基本用法：

```
select distinct 列名1，…，列名n
 from 表
where 条件
```

DISTINCT 是 SQL 中用来返回唯一不重复结果集的关键字。它通常用于 SELECT 语句中，可以指定一个或多个列进行去重，并返回唯一的结果。当使用 SELECT 查询数据时，可能会得到包含重复行的结果集。为了去除这些重复行，可以使用 DISTINCT 关键字来获取唯一的记录。

除了进行单列、多列去重以外，Distinct还可用于对表达式进行去重，这允许你根据某些计算得到的结果进行去重。

```
SELECT DISTINCT ( `name` + age) result FROM students

```

此外搭配Count函数还可统计字段中不重复的值的个数。

```
SELECT COUNT(DISTINCT NAME) num FROM students

```

**注意事项：**

1. distinct 必须放在字段的开头，即放在第一个参数的位置。
2. 只能在select语句中使用，不能在insert、delete、update中使用。
3. distinct表示对后面的所有参数的拼接 取 不重复的记录。

4. distinct 忽略 NULL 值：distinct 关键字默认会忽略 NULL 值，即将 NULL 视为相同的值。如果你希望包括 NULL 值在去重结果中，可以使用 IS NULL 或 IS NOT NULL 进行过滤。

5. DISTINCT 基于所有选择的列：DISTINCT 关键字基于选择的所有列来进行去重。如果你只想根据部分列进行去重，可以使用子查询或者窗口函数等技术来实现。

6. DISTINCT 的性能消耗：DISTINCT 操作可能会对查询的性能产生一定的影响，特别是在处理大量数据时。因为它需要对结果集进行排序和比较以去除重复行。如果性能是一个关键问题，可以考虑其他优化方法，例如使用索引或者合理设计查询。

7. 结果集顺序不保证：使用 DISTINCT 关键字后，结果集的顺序可能会发生变化，因为数据库系统通常会对结果进行重新排列以去除重复行。如果需要特定的结果排序，可以使用 ORDER BY 子句进行排序。

# Having

基本用法

```
select 列名1，... , 列名n 
from 表
group by 列名1，... , 列名n
having  筛选规则

```

其实having很好理解 他的功能与where是一样的，都是为了写条件语句进行筛选数据。但是SQL语法规定，对于group by之后的组结果，想要对其结果进行筛选，必须使用having关键字，不能使用where。所以我们可以把having看成group by的搭档就行了，见了group by 想要对其结果筛选，后面就使用having关键字。就像我们吃饭要用筷子，喝汤要用勺子，筷子和勺子都是吃饭的工具。having与where都是筛选的关键词，只是应用的场景不同而已。

具体细分而言，where不可以搭配聚合函数使用，比如Min、Max、Sum、Avg、Count等等，一般搭配非聚合函数或窗口函数使用，筛选满足条件的行。而Having就是专门用于搭配聚合函数使用的，且一般都是搭配Group By一起使用，筛选满足条件的组。
