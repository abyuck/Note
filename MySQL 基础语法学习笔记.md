# MySQL 基础语法笔记

---

## 一、SQL 分类

SQL（Structured Query Language）用来和关系型数据库交互。关键字通常大写，其他小写。

| 分类 | 全称 | 说明 | 关键字 |
|------|------|------|--------|
| DDL | Data Definition Language | 数据定义语言 | `CREATE`、`DROP`、`ALTER`、`TRUNCATE` |
| DML | Data Manipulation Language | 数据操作语言 | `INSERT`、`UPDATE`、`DELETE`、`CALL` |
| DQL | Data Query Language | 数据查询语言（DML子集） | `SELECT` |
| DCL | Data Control Language | 数据控制语言 | `GRANT`、`REVOKE` |

---

## 二、常用命令

### 创建数据库
```sql
CREATE DATABASE test_db;
USE test_db;
```

### 创建表
```sql
CREATE TABLE player (
    id INT,
    name VARCHAR(100),
    level INT,
    exp INT,
    gold DECIMAL(10, 2)
);
```

### 查看表结构
```sql
DESC player;
```

### 修改表结构
```sql
ALTER TABLE player MODIFY COLUMN name VARCHAR(200);
ALTER TABLE player RENAME COLUMN name TO nick_name;
ALTER TABLE player ADD COLUMN last_login DATETIME;
ALTER TABLE player DROP COLUMN last_login;
```

### 删除表
```sql
DROP TABLE player;
```

---

## 三、数据的增删改查

### 插入（INSERT）
```sql
-- 指定列名插入
INSERT INTO player (id, name, level, exp, gold) 
VALUES (1, '张三', 1, 1, 1);

-- 一次插入多条
INSERT INTO player (id, name, level, exp, gold) 
VALUES (1, '张三', 1, 1, 1), (2, '李四', 2, 2, 2);

-- 省略列名（顺序和表结构完全一致时可省略）
INSERT INTO player VALUES (1, '张三', 1, 1, 1);
```

### 查询（SELECT）
```sql
SELECT * FROM player;           -- 查所有列
SELECT id, name FROM player;    -- 查指定列
```

### 更新（UPDATE）
```sql
UPDATE player SET level = 2 WHERE name = '张三';
-- ⚠️ 注意：没有 WHERE 会更新全部记录！
-- 可以同时修改多个列，逗号分隔
```

### 删除（DELETE）
```sql
DELETE FROM player WHERE gold = 0;
-- ⚠️ 注意：没有 WHERE 会删除全部记录！
```

### 设置默认值
```sql
ALTER TABLE player MODIFY level INT DEFAULT 1;

-- 插入时省略该列，自动使用默认值（不再为 NULL）
INSERT INTO player (id, name, exp, gold) VALUES (4, '王五', 0, 0);
```

---

## 四、数据导入导出

### 导出（mysqldump）

```bash
# 基本语法（推荐：-p 后不写密码，回车后安全输入）
mysqldump -u [用户名] -p [选项] [数据库名] > [导出文件.sql]

# 示例1：导出整个数据库
mysqldump -u root -p mydatabase > mydatabase_backup.sql

# 示例2：导出特定表
mysqldump -u root -p mydatabase users orders > tables_backup.sql

# 示例3：仅导出结构（不含数据）
mysqldump -u root -p --no-data mydatabase > mydatabase_schema.sql

# 示例4：导出所有数据库
mysqldump -u root -p --all-databases > all_databases_backup.sql

# 示例5：生产环境推荐命令（InnoDB）
mysqldump -u root -p \
  --single-transaction \
  --routines \
  --triggers \
  --master-data=2 \
  mydatabase > mydatabase_prod_backup.sql

# 示例6：导出并压缩
mysqldump -u root -p mydatabase | gzip > mydatabase_backup.sql.gz
```

**参数说明：**
- `--single-transaction`：创建一致性快照，不锁表（InnoDB）
- `--routines`：导出存储过程和函数
- `--triggers`：导出触发器
- `--master-data=2`：记录二进制日志位置（用于主从复制）

### 导入（mysql）

```bash
# 方式一：使用重定向符（最常用）
mysql -u [用户名] -p [目标数据库名] < [要导入的文件.sql]

# 方式二：登录后使用 source 命令
mysql -u root -p
USE [目标数据库名];
source /完整路径/文件名.sql;

# 导入压缩文件（无需先解压）
gunzip < mydatabase_backup.sql.gz | mysql -u root -p new_database
```

**⚠️ 注意：** 导入前目标数据库通常需先手动创建：
```sql
CREATE DATABASE new_database 
  CHARACTER SET utf8mb4 
  COLLATE utf8mb4_unicode_ci;
```

---

## 五、表的常用约束

### 约束类型总览

| 约束名 | 关键字 | 作用 |
|--------|--------|------|
| 主键约束 | `PRIMARY KEY` | 唯一标识一行，非空且唯一 |
| 非空约束 | `NOT NULL` | 该列值不能为 NULL |
| 唯一约束 | `UNIQUE` | 该列值不能重复（允许多个 NULL） |
| 默认值约束 | `DEFAULT` | 插入时未指定值则使用默认值 |
| 检查约束 | `CHECK` | 值必须满足指定条件（MySQL 8.0.16+） |
| 外键约束 | `FOREIGN KEY` | 关联另一张表，保证引用完整性 |

### 完整建表示例

```sql
-- 第1步：创建被引用的主表 departments
CREATE TABLE departments (
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL UNIQUE
);

-- 第2步：创建带所有常用约束的 employees 表
CREATE TABLE employees (
    -- 1. 主键 + 自增
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT 
        COMMENT '员工ID, 主键, 自动递增',

    -- 2. 非空 + 唯一
    employee_number VARCHAR(10) NOT NULL UNIQUE 
        COMMENT '员工编号, 必须提供且不能重复',

    -- 3. 非空
    first_name VARCHAR(50) NOT NULL COMMENT '名字, 不能为空',
    last_name VARCHAR(50) NOT NULL COMMENT '姓氏, 不能为空',

    -- 非空 + 唯一
    email VARCHAR(100) NOT NULL UNIQUE COMMENT '邮箱, 不能为空且不能重复',

    phone_number VARCHAR(20) COMMENT '联系电话, 可以为空',

    -- 4. 默认值
    hire_date DATE NOT NULL DEFAULT (CURRENT_DATE) 
        COMMENT '入职日期, 不能为空, 默认为当前日期',

    -- 5. 检查约束（MySQL 8.0.16+）
    salary DECIMAL(10, 2) NOT NULL CHECK (salary >= 3000.00) 
        COMMENT '底薪, 不能为空, 且必须大于等于3000',

    -- 6. 外键约束
    department_id INT UNSIGNED COMMENT '所属部门ID, 外键',

    -- 检查约束示例
    status VARCHAR(10) NOT NULL DEFAULT 'active' 
        CHECK (status IN ('active', 'on_leave', 'terminated')) 
        COMMENT '员工状态',

    -- 表级外键定义
    CONSTRAINT fk_department
        FOREIGN KEY (department_id)
        REFERENCES departments(id)
        ON DELETE SET NULL
        ON UPDATE CASCADE
);
```

**外键行为说明：**
- `ON DELETE SET NULL`：主表记录删除时，从表关联字段设为 NULL（要求该列允许 NULL）
- `ON UPDATE CASCADE`：主表 ID 更新时，从表关联字段自动级联更新

---

## 六、常用查询语句

### WHERE 子句

```sql
-- 比较运算符
SELECT * FROM player WHERE level = 1;
SELECT * FROM player WHERE level > 1 AND level < 5 OR level = 7;

-- IN：属于某个集合
SELECT * FROM player WHERE level IN (1, 3, 5);

-- BETWEEN...AND：范围内（闭区间）
SELECT * FROM player WHERE level BETWEEN 1 AND 10;
```

### 模糊查询（LIKE / REGEXP）

```sql
-- LIKE：% 匹配任意长度字符串，_ 匹配单个字符
SELECT * FROM player WHERE name LIKE '王%';   -- 王开头，如"王五""王大锤"
SELECT * FROM player WHERE name LIKE '王_';   -- 王+单字，如"王五"

-- REGEXP：正则表达式
SELECT * FROM player WHERE name REGEXP '王';     -- 包含"王"
SELECT * FROM player WHERE name REGEXP '^宫本';  -- "宫本"开头
SELECT * FROM player WHERE name REGEXP '李$';    -- "李"结尾
```

**正则通配符：**
| 符号 | 含义 |
|------|------|
| `.` | 任意单个字符 |
| `^` | 开头 |
| `$` | 结尾 |
| `[abc]` | 其中任意一个字符 |
| `[a-z]` | 范围内任意一个字符 |
| `A\|B` | A 或 B |

### NULL 值处理

```sql
-- ❌ 错误：NULL 不能用 = 比较
SELECT * FROM player WHERE level = NULL;

-- ✅ 正确
SELECT * FROM player WHERE level IS NULL;
SELECT * FROM player WHERE level IS NOT NULL;
-- 也可以用 <=> 操作符
SELECT * FROM player WHERE level <=> NULL;
```

> **注意：** 空字符串 `''` 不等于 `NULL`！

### ORDER BY 排序

```sql
SELECT * FROM player ORDER BY level;            -- 升序（默认 ASC，可省略）
SELECT * FROM player ORDER BY level DESC;       -- 降序
SELECT * FROM player ORDER BY level DESC, exp ASC, name DESC;  -- 多列排序
-- 也可以用列序号代替列名（level 是第5列）
SELECT * FROM player ORDER BY 5 DESC;
```

### 聚合函数

| 函数 | 作用 |
|------|------|
| `COUNT(*)` | 统计行数 |
| `AVG(col)` | 平均值 |
| `MAX(col)` | 最大值 |
| `MIN(col)` | 最小值 |
| `SUM(col)` | 求和 |

```sql
SELECT COUNT(*) FROM player;
SELECT AVG(level) FROM player;
```

### GROUP BY 分组 + HAVING 过滤

```sql
-- 按 level 分组，统计每组数量
SELECT level, COUNT(level) FROM player GROUP BY level;

-- HAVING：对分组结果过滤（类似 WHERE，但用于聚合后）
SELECT level, COUNT(level) FROM player 
GROUP BY level 
HAVING COUNT(level) > 4 
ORDER BY COUNT(level);
```

### LIMIT 限制结果数量

```sql
-- 取前3名
SELECT * FROM player ORDER BY level DESC LIMIT 3;

-- 两个参数：LIMIT 偏移量, 数量（从第4名开始，取3个）
SELECT * FROM player ORDER BY level DESC LIMIT 3, 3;
```

### DISTINCT 去重

```sql
SELECT DISTINCT name FROM player;
```

---

## 七、集合运算

| 运算 | 关键字 | 说明 |
|------|--------|------|
| 并集（去重） | `UNION` | 合并结果，自动去重 |
| 并集（不去重） | `UNION ALL` | 合并结果，保留重复 |
| 交集 | `INTERSECT` | 两个结果集的交集 |
| 差集 | `EXCEPT` | 第一个结果集减去第二个 |

```sql
SELECT * FROM player WHERE level BETWEEN 1 AND 3
UNION
SELECT * FROM player WHERE exp BETWEEN 1 AND 3;

SELECT * FROM player WHERE level BETWEEN 1 AND 3
UNION ALL
SELECT * FROM player WHERE exp BETWEEN 1 AND 3;
```

---

## 八、子查询

```sql
-- 标量子查询（返回单个值）
SELECT * FROM player WHERE level > (SELECT AVG(level) FROM player);

-- 子查询用作列
SELECT 
    level, 
    ROUND((SELECT AVG(level) FROM player)) AS average,
    level - ROUND((SELECT AVG(level) FROM player)) AS diff
FROM player;

-- 子查询用于 INSERT（插入查询结果）
INSERT INTO new_player 
SELECT * FROM player WHERE level BETWEEN 6 AND 10;

-- EXISTS：判断子查询结果是否存在（返回 0/1）
SELECT EXISTS(SELECT * FROM player WHERE level > 100);
```

---

## 九、表关联（JOIN）

### 内连接（INNER JOIN）
返回两表都匹配的数据：

```sql
-- 标准写法
SELECT * FROM player
INNER JOIN equip ON player.id = equip.player_id;

-- 使用 WHERE 的旧写法（效果相同）
SELECT * FROM player, equip
WHERE player.id = equip.player_id;

-- 使用别名
SELECT * FROM player p, equip e
WHERE p.id = e.player_id;
```

### 左连接（LEFT JOIN）
返回左表全部数据 + 右表匹配数据（不匹配的填 NULL）：

```sql
SELECT * FROM player
LEFT JOIN equip ON player.id = equip.player_id;
```

### 右连接（RIGHT JOIN）
返回右表全部数据 + 左表匹配数据：

```sql
SELECT * FROM player
RIGHT JOIN equip ON player.id = equip.player_id;
```

### 笛卡尔积
两表所有行的组合（去掉连接条件就会出现）：

```sql
SELECT * FROM player, equip;  -- 笛卡尔积
```

> **本质：** 表连接 = 笛卡尔积 + 条件过滤

---

## 十、索引（INDEX）

索引是一种提高查询效率的数据结构，相当于书的目录。

```sql
-- 创建索引
CREATE INDEX index_name ON tbl_name (col_name, ...);
-- 或
ALTER TABLE tbl_name ADD INDEX index_name (col_name);

-- 创建唯一索引
CREATE UNIQUE INDEX index_name ON tbl_name (col_name);

-- 删除索引
DROP INDEX index_name ON tbl_name;
```

**最佳实践：**
- 对主键自动创建索引（主键索引）
- 对经常出现在 `WHERE`、`JOIN` 后的列创建索引
- 不要过度创建索引（影响插入/更新性能）

---

## 十一、视图（VIEW）

视图是一种虚拟表，本身不包含数据，只是保存了一条查询语句。

```sql
-- 创建视图
CREATE VIEW view_name AS
[查询语句];

-- 示例：创建等级前10名视图
CREATE VIEW top10 AS
SELECT * FROM player ORDER BY level DESC LIMIT 10;

-- 查询视图（像普通表一样使用）
SELECT * FROM top10;

-- 修改视图
ALTER VIEW view_name AS
[新查询语句];

-- 删除视图
DROP VIEW view_name;
```

---

## 附录：练习题

1. 查询名字以"宫本"开头的玩家
2. 查询名字以"李"结尾的玩家
3. 统计各个姓氏的玩家数量，按数量降序排列，只显示数量 ≥ 5 的姓氏
   ```sql
   SELECT SUBSTR(name, 1, 1), COUNT(SUBSTR(name, 1, 1)) 
   FROM player 
   GROUP BY SUBSTR(name, 1, 1)
   HAVING COUNT(SUBSTR(name, 1, 1)) >= 5
   ORDER BY COUNT(SUBSTR(name, 1, 1)) DESC;
   ```

---

*笔记整理自《两个MC开发者的浅显思考》系列教程，原文更新于 2025-11-08*
