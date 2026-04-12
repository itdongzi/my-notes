# Oracle SQL 与 MySQL 差异 · 学习笔记


## 一、核心差异速查表

| 场景 | MySQL | Oracle（本项目） |
| :--- | :--- | :--- |
| 字符串拼接 | `CONCAT()` | `\|\|` 或 `CONCAT()` |
| 转义字符串 | `'` | `''` 或 `q'[]'` |
| 日期转字符串 | `DATE_FORMAT()` | `TO_CHAR()` |
| 字符串转日期 | `STR_TO_DATE()` | `TO_DATE()` |
| 日期当前值 | `NOW()` / `CURDATE()` | `SYSDATE` |
| 分页行号 | `LIMIT` | `ROWNUM`（嵌套三层） |
| 空值替换 | `IFNULL()` | `NVL()`/`NVL2()` |
| 字符串转大写 | `UPPER()` | `UPPER()`（相同） |
| 字符串转小写 | `LOWER()` | `LOWER()`（相同） |
| 取前N行 | `LIMIT N` | `WHERE ROWNUM <= N` |
| 左连接 | `LEFT JOIN` | `LEFT JOIN`（Oracle 9i+ 也支持，项目大量用`+`） |
| 判空连接 | 用 `IFNULL` 或 `COALESCE` | `(+)`（Oracle 特有的外连接写法） |
| 当前时间 | `NOW()` | `SYSDATE` |
| 区间时间查询 | `BETWEEN AND` | `BETWEEN AND`（相同） |
| 空值判断 | `IS NULL` / `IS NOT NULL` | `IS NULL`/`IS NOT NULL`（相同） |
| 处理字符串转日期 | 避免`STR_TO_DATE` | 避免`TO_DATE(param` |

---

## 二、分页查询（⭐最重要⭐）
MySQL 的 `LIMIT` 很简单，Oracle 则需要三层嵌套子查询 + `ROWNUM`。

**公式：**
```sql
SELECT * FROM (
  SELECT rownum as rowno, 子查询.* FROM (
    -- 最内层：正常查询 + ORDER BY
  ) 子查询别名
  WHERE rownum <= 结束行
) m
WHERE m.rowno >= 起始行
```

**本项目封装写法：**
```sql
SELECT * FROM (
    SELECT rownum as rowno, t.*
    FROM (
        SELECT ... FROM ... WHERE ... ORDER BY ...
    ) t
) m
WHERE m.rowno BETWEEN f_开始行(:page, :rows) AND f_结束行(:page, :rows)
```
**注意：**
+ 起始行 = (:page - 1) * :rows + 1，结束行 = page * rows
+ ROWNUM是伪列，查询时动态生成，不能直接用于 ORDER BY 外层
+ 必须在中间层处理 ROWNUM 截断，最外层再过滤起始位置

---

## 三、空值处理函数
### 3.1 NVL（最常用）
**公式：**`NVL(字段, 默认值)`  
相当于 MySQL 的` IFNULL(表达式, 默认值)`。  
```sql
-- Oracle
NVL(字段, 0)
NVL(字段, '未知')
NVL(数量, 0)

-- 对应 MySQL
IFNULL(字段, 0)
```

### 3.2 NVL2（双分支）
**公式：**`NVL2(表达式, 不为空值, 为空值)`  
```sql
-- 如果 名称 不为空，用 名称；否则用 '未知菜品'
NVL2(名称, 名称, '未知菜品')

-- MySQL 没有直接对应，用 CASE WHEN 代替
CASE WHEN 名称 IS NOT NULL THEN 名称 ELSE '未知菜品' END
```

### 3.3 NVL 与 NVL2 区别  
| 函数          | 当表达式不为NULL | 当表达式为NULL |
|:------------|:----------:|:---------:|
| NVL(a,b)    |     `返回a`    |    `返回b`    |
| NVL2(a,b,c) |    `返回b`     |    `返回c`    |


### 3.4 DECODE（条件分支，Oracle 特有）  
**公式：**`DECODE(字段, 值1, 结果1, 值2, 结果2, ..., 默认值)`
相当于 `MySQL` 的 `ELT ()` 或简化版 `CASE WHEN`：
```sql
-- Oracle
DECODE(t.出入状态, '1','入库','2','出库')
DECODE(m.状态,'1','已处理','0','待处理','删除')

-- MySQL 等价
CASE t.出入状态 WHEN '1' THEN '入库' WHEN '2' THEN '出库' END
CASE m.状态 WHEN '1' THEN '已处理' WHEN '0' THEN '待处理' ELSE '删除' END  
```

---  
## 四、日期时间处理
### 4.1 当前时间  
```sql
-- Oracle
SYSDATE -- 当前日期时间（最常用）
SYSTIMESTAMP -- 带时区的时间戳

-- MySQL
NOW()
```

### 4.2 日期转字符串
**公式：**`TO_CHAR(日期, '格式')`
```sql
TO_CHAR(t.体检日期,'yyyy-MM-dd') -- 2024-01-15
TO_CHAR(t.创建时间,'yyyy-mm-dd hh24:mi') -- 2024-01-15 14:30
TO_CHAR(SYSDATE,'yyyy-MM-dd HH24:mi:ss')
```
**常用格式：**

| 元素       | 含义                     | 示例     |
| :--------- | :----------------------- | :------- |
| yyyy       | `4 位年     `              |` 2024 `    |
| mm         | `2 位月    `               | `01-12 `   |
| dd         | `2 位日       `            | `01-31 `   |
| hh24       | `24 小时制   `             | `00-23 `   |
| hh 或 hh12 | `12 小时制  `              | `01-12 `   |
| mi         | `分钟                    ` | `00-59 `   |
| ss         | `秒                     `  | `00-59`    |
| day        | `星期几（全名）          ` | `Monday`   |
| d          | `星期几（数字，1 = 周日）` |          |

### 4.3 字符串转日期  
**公式：**`TO_DATE(字符串, '格式')`
```sql
TO_DATE(':体检日期_begin','yyyy-mm-dd')
TO_DATE(':结束时间 23:59','yyyy-mm-dd hh24:mi')
TO_DATE(':开始时间','yyyy-mm-dd hh24:mi:ss')
```
  
### 4.4 日期加法/减法
```sql
-- Oracle 日期加减，直接 + 或 - 数字（代表天数）
TO_DATE(':结束时间 23:59:59','yyyy-mm-dd HH24:mi:ss') >= t.体检日期
SYSDATE - 7 -- 7天前
ADD_MONTHS(SYSDATE, 1) -- 1个月后
TRUNC(SYSDATE) -- 当天零点（TRUNC 截断时分秒）

-- MySQL 等价
DATE_SUB(NOW(), INTERVAL 7 DAY)
DATE_ADD(NOW(), INTERVAL 1 MONTH)
DATE(NOW()) -- 当天零点
```

### 4.5 计算年龄（精确到岁）
**公式：**`TRUNC(MONTHS_BETWEEN(SYSDATE, 出生日期) / 12)`  
```sql
-- 本项目写法（从出生日期计算年龄）
TRUNC(MONTHS_BETWEEN(SYSDATE, 出生日期) / 12)

-- MySQL 等价
TIMESTAMPDIFF(YEAR, 出生日期, CURDATE())
```

### 4.6 计算两个日期之间的天数  
```sql
TRUNC(SYSDATE - 计划日期) -- 距今天数
TRUNC(到期时间) - TRUNC(SYSDATE) -- 距到期天数
```

---

## 五、字符串操作
### 5.1 拼接
```sql
-- Oracle 三种方式（项目中最常用前两种）
名称 || 'x' || 数量 -- 用 || 拼接（最常见）
CONCAT(名称, 数量) -- 最多两个参数
名称 || 数量 || '元' -- 链式拼接

-- MySQL
CONCAT(名称, 'x', 数量) -- 可以多个参数
```

### 5.2 模糊查询
```sql
-- 前后都模糊（最常见）
字段 LIKE '%:关键字%'
-- 前缀匹配
字段 LIKE ':关键字%'

-- MySQL 相同
字段 LIKE CONCAT('%', 关键字, '%')
```

### 5.3 大小写转换
```sql
UPPER('%:关键字%') -- 转大写后匹配
LOWER(字段) -- 转小写

-- MySQL 等价
UPPER()
LOWER()
```

### 5.4 字符串截断时间部分（Oracle 特有）
```sql
-- TRUNC 用于日期时，截断时分秒部分，保留日期
TRUNC(TO_date(':date_s','yyyy-mm-dd')) -- 转为当天 00:00:00
TRUNC(TO_date(':date','yyyy-mm-dd') - 1, 'd') + 1 -- 本周一
TRUNC(TO_date(':date','yyyy-mm-dd') - 1, 'd') + 7 -- 本周日
```

---

## 六、外连接写法（Oracle vs MySQL）
### 6.1 Oracle (+) 写法
**公式：** `左表.字段 = 右表.字段(+)`
```sql
-- Oracle 左外连接（(+) 在右表那边）
FROM 员工合同 h, 用户信息 g, 员工合同类型 l
WHERE h.用户id = g.Id(+) -- 如果 g 没有匹配，g 的字段返回 NULL
AND h.合同类型 = l.编码(+) -- 左外连接
AND h.状态 != '9'

-- MySQL 等价写法
FROM 员工合同 h
LEFT JOIN 用户信息 g ON h.用户id = g.Id
LEFT JOIN 员工合同类型 l ON h.合同类型 = l.编码
WHERE h.状态 != '9'
```

**注意：**
+ (+) 写在哪个表的字段后，哪个表就是 "可选的"
+ 多表查询中，(+) 可以跨多表使用
+ 但注意：不能用 (+) 同时连接同一个表的两个外连接

### 6.2 本项目中大量混合写法
```sql
-- 同时使用标准 JOIN 和 (+) 写法
FROM 个人文档记录 a
LEFT JOIN TD个人信息 t2 ON a.个人ID = t2.ID(+)
LEFT JOIN 治疗区 t3 ON a.治疗区ID = t3.ID(+)
LEFT JOIN 用户信息 t4 ON a.创建人ID = t4.ID(+)
```

---
## 七、分组与聚合
### 7.1 WM_CONCAT（行转逗号分隔字符串）
**公式：**`WM_CONCAT(字段) WITHIN GROUP (ORDER BY 字段)`  
这是 Oracle 将多行合并为一行的最常用方法（类似 MySQL 的 GROUP_CONCAT）。
```sql
-- 简单用法（默认逗号分隔）
SELECT WM_CONCAT(名称) FROM 菜品信息
-- 带排序
LISTAGG(名称, ',') WITHIN GROUP (ORDER BY 名称)

-- 对比 MySQL
GROUP_CONCAT(名称 ORDER BY 名称 SEPARATOR ',')
```

### 7.2 LISTAGG（Oracle 11gR2+，推荐）
**公式：**`LISTAGG(字段, 分隔符) WITHIN GROUP (ORDER BY 排序字段)`
```sql
-- 本项目实例
LISTAGG(rq.名称, ',') WITHIN GROUP (ORDER BY rq.名称)
```

---

## 八、层级查询（树形结构）
### 8.1 CONNECT BY（Oracle 特有）
**公式：**
```sql
SELECT ...
FROM 机构信息
START WITH ID = ':机构ID' -- 起始节点
CONNECT BY PRIOR ID = 上级ID -- 向下查子节点
-- 或 CONNECT BY PRIOR 上级ID = ID -- 向上查父节点
ORDER SIBLINGS BY ID -- 同级内排序
```

PRIOR 关键字：表示 "上一层"。

```sql
-- 从当前机构向下查所有子机构
SELECT g.ID, g.上级ID, g.名称, LEVEL AS 层级
FROM 机构信息 g
START WITH ID = ':机构ID'
CONNECT BY PRIOR ID = 上级ID

-- 从当前机构向上查所有上级
START WITH ID = ':机构ID'
CONNECT BY PRIOR 上级ID = ID
```

**本项目实际应用（健康体检下级机构）：**
```sql
WITH subordinate_institutions AS (
SELECT g.ID, g.上级ID, g.名称, LEVEL AS 层级,
(SELECT COUNT(1) FROM 健康体检 WHERE 机构ID = g.ID AND 状态 = '1') 数量
FROM 机构信息 g
START WITH ID = ':机构ID'
CONNECT BY PRIOR ID = 上级ID
ORDER SIBLINGS BY ID
)
SELECT * FROM subordinate_institutions
```

---
## 九、动态条件查询（可选参数）
**公式：** `(参数 IS NULL OR 字段 = 参数)`  
这是本项目中最核心的查询模式，用于实现 "参数为空时不过滤" 的条件：  
```sql
WHERE (':个人ID' Is Null Or t.个人ID = ':个人ID')
AND (':体检日期_begin' Is Null Or t.体检日期 >= to_date(':体检日期_begin','yyyy-mm-dd'))
AND (':体检日期_end' Is Null Or to_date(':体检日期_end'||' 23:59:59','yyyy-mm-dd HH24miss') >= t.体检日期)
AND (':关键字' Is Null Or 个人信息.姓名 Like '%:关键字%' Or 个人信息.简码 Like Upper('%:关键字%'))  
```
**原理：**
+ 参数传入时：参数 IS NULL 为 FALSE，走后面的条件过滤
+ 参数为空时：参数 IS NULL 为 TRUE，整个 OR 短路为 TRUE，返回所有记录


---

## 十、子查询与 WITH 子句
### 10.1 WITH（公用表表达式）
**公式：** `WITH 别名 AS (子查询) SELECT ... FROM 别名`
```sql
WITH Grid AS (
Select Distinct 个人ID From (
Select G2.Id 个人id
From 个人信息 G1, 个人信息 G2
Where G1.Unionid = G2.Unionid And G1.Id = ':个人Id'
Union All Select ':个人Id' 个人ID From Dual
)
)
SELECT m.*
FROM Grid, TD报告文件 f
WHERE f."个人ID" = g.ID(+)
AND (g.ID = Grid.个人ID Or g.证件号码 = Grid.个人ID)
```

作用：把重复使用的复杂子查询提取出来，提高可读性和复用性。

### 10.2 EXISTS vs IN
```sql
-- EXISTS（效率通常更高）
WHERE EXISTS (SELECT '' FROM 入住记录 r
WHERE r.个人ID = g.ID
AND (':入住状态' Is Null Or r.入住状态 = ':入住状态'))

-- IN
WHERE g.ID IN (SELECT r.个人ID FROM 入住记录 r WHERE ...)
```

---

## 十一、UNION 与 UNION ALL
```sql
-- UNION：合并 + 去重（自动按行比较去重）
SELECT ... FROM 门诊记录
UNION
SELECT ... FROM 住院记录

-- UNION ALL：合并但保留所有行（包括重复）
SELECT ... FROM 门诊记录
UNION ALL
SELECT ... FROM 住院记录
```
本项目大量使用 UNION ALL 做多表合并查询（门诊 + 住院），因为已知数据不会重复，不需要去重开销。

---

## 十二、CASE WHEN 两种写法
### 12.1 搜索型（推荐）
```sql
CASE
WHEN t.状态 = '0' THEN '未签订'
WHEN t.状态 = '1' AND t.结束日期 > SYSDATE THEN '有效'
WHEN t.状态 = '1' AND t.结束日期 <= SYSDATE THEN '已过期'
ELSE '未知'
END
```

### 12.2 简单型
```sql
CASE t.健康状态
WHEN '5' THEN '不满意'
WHEN '4' THEN '不太满意'
WHEN '3' THEN '说不清楚'
WHEN '2' THEN '基本满意'
WHEN '1' THEN '满意'
END
```

 ---

## 十三、XML 函数（本项目治疗文书）
Oracle 有强大的 XML 处理能力：  
```sql
-- 提取 XML 节点值（Oracle 11g 及之前）
EXTRACTVALUE(VALUE(i), '/root/Content[@Name="治疗师签名"]')

-- 提取 XML 片段（Oracle 12c+ 推荐）
EXTRACT(内容信息, '/root/Content[@ElementID=1169]').GETSTRINGVAL()

-- XML 行转列（将 XML 节点展开为行）
TABLE(XMLSequence(extract(文档,'/root/DocID'))) i

-- XMLTYPE 提取
EXTRACTVALUE(t1.内容信息, 'root/Content[@ElementID=1169]')
```

 ---
## 十四、字符串转义
MySQL 用单引号包单引号` ''`，Oracle 同样：
 ```sql
-- Oracle 和 MySQL 相同
WHERE 名称 = '张三''的物品'
```

---
## 十五、本项目命名规范对照

| 含义               | 本项目示例                             |
| :----------------- | :------------------------------------- |
| 主键 ID            | `ID`                                     |
| 外键关联           | `xxxID（如个人 ID、机构 ID）`            |
| 名称               | `名称           `                        |
| 状态（正常 / 删除）| `状态 = '1'/ 状态 != '9' `                |
| 逻辑删除           | `状态 = '9' 表示删除     `               |
| 创建时间           | `创建时间（默认 SYSDATE）  `             |
| 分页参数           | `:page（当前页）、:rows（每页行数） `    |
| 条件参数           | `':字段名'（字符串）、:字段名（数字或 ID）`|

 ---
## 十六、高频 SQL 模式汇总
**模式 1：标准列表查询（带分页 + 条件 + COUNT）**
```sql
-- List
SELECT * FROM (
SELECT rownum as rowno, t.* FROM (
SELECT ... FROM ... WHERE ... ORDER BY ...
) t
) m
WHERE m.rowno Between f_开始行(:page,:rows) And f_结束行(:page,:rows)

-- Count（去除外层分页即可）
SELECT Count(1) FROM ... WHERE ...
```
**模式 2：多表 LEFT JOIN + 可选条件**
```sql
SELECT t.ID, 个人信息.姓名 AS 姓名, 用户信息.姓名 AS 医生
FROM 表1 t
LEFT JOIN 个人信息 ON t.个人ID = 个人信息.ID(+)
LEFT JOIN 用户信息 ON t.医生ID = 用户信息.ID(+)
WHERE (':个人ID' Is Null Or t.个人ID = ':个人ID')
AND t.状态 = '1'
ORDER BY t.创建时间 DESC
```
**模式 3：级联下拉 / 字典查询**
```sql
SELECT 编码, 名称 FROM 字典表 WHERE 状态 = '1'
```

**模式 4：时间范围查询**
```sql
AND (':开始时间' IS NULL OR t.创建时间 >= TO_DATE(':开始时间','yyyy-mm-dd'))
AND (':结束时间' IS NULL OR TO_DATE(':结束时间 23:59:59','yyyy-mm-dd hh24:mi:ss') >= t.创建时间)
```