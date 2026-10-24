# SQL 速查表 · SQL Cheatsheet

[![Stars](https://img.shields.io/github/stars/Ezra-Zhao/sql-cheatsheet?style=social)](https://github.com/Ezra-Zhao/sql-cheatsheet/stargazers) [![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE) ![Languages](https://img.shields.io/badge/languages-3-blue)

Support & more: [GitHub Sponsors](https://github.com/sponsors/Ezra-Zhao) · [Gumroad — templates & workbooks I use](https://ezraflow28.gumroad.com) · [Ko-fi](https://ko-fi.com/ezra777zhao)


> 🌍 [Español](README.es.md) · [Português](README.pt.md)

一份**一页纸**的 SQL 速查表：30 条高频查询与操作，通用标准 SQL 写法，覆盖查询、聚合、JOIN、子查询/CTE、增删改与事务、索引与表结构。每个条目只讲三件事——**它是干什么的、一条能直接复制粘贴的 SQL、适用场景标签**。**全部 30 条已在 SQLite 3.45 上逐条真实执行验证**（连每条的结果都对过），MySQL / PostgreSQL 的方言差异在条目里逐条标注，不编造不存在的函数。

## 面向谁

后端开发者、数据分析师、运维/SRE，以及面试前想快速过一遍 SQL 的你。如果你每次写 JOIN 都要现查 ON 写哪、HAVING 和 WHERE 有什么区别，这张表能帮你把最常用的 30 条一次记牢。

## 怎么用

1. 直接看 **[CHEATSHEET.md](CHEATSHEET.md)**——英文版正文，和本页条目一一对应。
2. 先把下面的测试表建起来（SQLite / MySQL / PostgreSQL 都能跑），每条 SQL 亲手执行一遍，对照注释里的预期结果。
3. 顺序就是学习路径：查询基本功 → 聚合分组 → JOIN → 子查询/CTE → 增删改与事务 → 索引与表结构。

## 先跑起来（测试表，30 秒建好）

三张小表：部门 `departments`、员工 `employees`、订单 `orders`。本页所有示例都基于这组数据，结果可复现。

```sql
CREATE TABLE departments (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE employees (id INTEGER PRIMARY KEY, name TEXT NOT NULL, dept_id INTEGER, salary INTEGER, manager_id INTEGER, hired_on TEXT);
CREATE TABLE orders (id INTEGER PRIMARY KEY, emp_id INTEGER, amount REAL, ordered_on TEXT);

INSERT INTO departments VALUES (1, 'Engineering'), (2, 'Sales'), (3, 'HR');
INSERT INTO employees VALUES
  (1, 'Alice', 1, 120000, NULL, '2019-03-01'),
  (2, 'Bob',   1,  95000, 1,    '2020-07-15'),
  (3, 'Carol', 1,  88000, 1,    '2021-01-10'),
  (4, 'David', 2,  72000, NULL, '2018-05-20'),
  (5, 'Eve',   2,  68000, 4,    '2022-09-01'),
  (6, 'Frank', 3,  54000, NULL, '2021-11-30'),
  (7, 'Grace', NULL, 60000, NULL, '2023-02-14');
INSERT INTO orders VALUES
  (101, 2, 250.00, '2024-01-05'),
  (102, 2, 120.50, '2024-01-20'),
  (103, 4, 980.00, '2024-02-02'),
  (104, 5, 430.75, '2024-02-14'),
  (105, 4, 150.00, '2024-03-01'),
  (106, 1, 2000.00, '2024-03-11'),
  (107, 5, 89.99,  '2024-03-15');
```

注意 Grace 没有部门（`dept_id` 为 NULL）、Carol / Frank / Grace 没有订单——好几条示例就靠这两处数据看出差别。

## 收录了什么

| 分类 | 内容 |
|---|---|
| 🔍 查询基本功 | 指定列 / WHERE / ORDER BY+LIMIT / DISTINCT / LIKE / IN·BETWEEN（6 条） |
| 📊 聚合与分组 | COUNT·SUM·AVG / GROUP BY / HAVING / COUNT(DISTINCT) / CASE 条件聚合（5 条） |
| 🔗 JOIN 家族 | INNER JOIN / LEFT JOIN / 反连接 / 自连接 / 三表 JOIN（5 条） |
| 🧩 子查询与 CTE | 标量子查询 / IN 子查询 / EXISTS / WITH / 窗口函数 RANK（5 条） |
| ✏️ 增删改与事务 | INSERT / UPDATE / DELETE / 事务 / UPSERT（5 条） |
| 🗂️ 索引与表结构 | CREATE TABLE / ALTER TABLE / CREATE INDEX / DROP（4 条） |

---

## 🔍 查询基本功

### `SELECT` — 取指定列
```sql
SELECT name, salary FROM employees;
-- 7 行：全体员工的姓名与工资。别习惯性写 SELECT *，取需要的列更清楚也更快
```
🏷️ `#查询` `#基础`

### `WHERE` — 按条件过滤行
```sql
SELECT name, salary FROM employees WHERE salary > 80000 ORDER BY salary DESC;
-- Alice 120000 / Bob 95000 / Carol 88000
```
🏷️ `#查询` `#过滤`

### `ORDER BY ... LIMIT` — 排序取前 N
```sql
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 3;
-- 工资最高的三人：Alice / Bob / Carol
-- LIMIT 在 MySQL / PostgreSQL / SQLite 通用；SQL Server 用 TOP 3，Oracle 用 FETCH FIRST 3 ROWS ONLY
```
🏷️ `#查询` `#排序` `#分页`

### `DISTINCT` — 去重
```sql
SELECT DISTINCT dept_id FROM employees;
-- 1 / 2 / 3 / NULL（Grace 无部门，NULL 也算一个值）
```
🏷️ `#查询` `#去重`

### `LIKE` — 模糊匹配
```sql
SELECT name FROM employees WHERE name LIKE 'A%';
-- Alice。% 匹配任意多个字符，_ 匹配单个字符
-- 方言：SQLite / MySQL 的 LIKE 对 ASCII 默认不区分大小写，PostgreSQL 的 LIKE 区分大小写，
--       PostgreSQL 另有 ILIKE 专做忽略大小写匹配（PG 专属，MySQL / SQLite 没有 ILIKE）
```
🏷️ `#查询` `#过滤` `#字符串`

### `IN` / `BETWEEN` — 多值与范围条件
```sql
SELECT name, salary FROM employees
WHERE dept_id IN (1, 2) AND salary BETWEEN 70000 AND 100000
ORDER BY salary DESC;
-- Bob 95000 / Carol 88000 / David 72000（BETWEEN 含两端）
```
🏷️ `#查询` `#过滤`

## 📊 聚合与分组

### `COUNT` / `SUM` / `AVG` — 基础聚合
```sql
SELECT COUNT(*) AS headcount, SUM(salary) AS total_payroll, AVG(salary) AS avg_salary FROM employees;
-- 7 人 / 工资总和 557000 / 平均 79571.43
```
🏷️ `#聚合`

### `GROUP BY` — 分组统计
```sql
SELECT dept_id, COUNT(*) AS n, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id;
-- 部门 1：3 人均薪 101000；部门 2：2 人 70000；部门 3：1 人 54000；NULL 组：1 人 60000
```
🏷️ `#聚合` `#分组`

### `HAVING` — 过滤分组结果
```sql
SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id HAVING AVG(salary) > 60000;
-- 只剩部门 1（101000）和部门 2（70000）
-- WHERE 过滤的是分组前的行，HAVING 过滤的是分组后的组，别写混
```
🏷️ `#聚合` `#分组` `#过滤`

### `COUNT(DISTINCT ...)` — 去重计数
```sql
SELECT COUNT(DISTINCT dept_id) AS dept_count FROM employees;
-- 3。COUNT 会跳过 NULL，所以 Grace 的空部门不计入
```
🏷️ `#聚合` `#去重`

### `CASE WHEN` — 条件聚合
```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN salary >= 90000 THEN 1 ELSE 0 END) AS high_earners
FROM employees;
-- 总人数 7 / 高薪人数 2（Alice、Bob）。一个查询同时出多个口径的统计就靠它
```
🏷️ `#聚合` `#条件`

## 🔗 JOIN 家族

### `INNER JOIN` — 两表内连接
```sql
SELECT e.name, d.name AS dept FROM employees e JOIN departments d ON e.dept_id = d.id;
-- 6 行：Grace 没有部门，被排除在外
```
🏷️ `#JOIN`

### `LEFT JOIN` — 保留左表全部行
```sql
SELECT e.name, d.name AS dept FROM employees e LEFT JOIN departments d ON e.dept_id = d.id;
-- 7 行：Grace 也在，dept 为 NULL。报全员名单用 LEFT JOIN，只报有归属的用 INNER JOIN
```
🏷️ `#JOIN`

### `LEFT JOIN ... IS NULL` — 找没有关联记录的行
```sql
SELECT e.name FROM employees e LEFT JOIN orders o ON o.emp_id = e.id WHERE o.id IS NULL;
-- Carol / Frank / Grace：从没下过单的员工。排查"孤儿数据"的经典写法
```
🏷️ `#JOIN` `#排查`

### 自连接 — 同一张表连自己
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e LEFT JOIN employees m ON e.manager_id = m.id;
-- Bob→Alice、Carol→Alice、Eve→David；没有经理的人 manager 为 NULL
-- 组织架构、上下级、评论回复这类层级数据都是这个套路
```
🏷️ `#JOIN` `#层级`

### 三表 JOIN + 聚合 — 部门订单统计
```sql
SELECT d.name AS dept, COUNT(o.id) AS order_count, SUM(o.amount) AS revenue
FROM departments d
JOIN employees e ON e.dept_id = d.id
JOIN orders o ON o.emp_id = e.id
GROUP BY d.id, d.name;
-- Engineering：3 单 2370.50；Sales：4 单 1650.74；HR 没人下单，不出现（INNER JOIN）
```
🏷️ `#JOIN` `#聚合` `#实战`

## 🧩 子查询与 CTE

### 标量子查询 — 和整体平均比
```sql
SELECT name, salary FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
-- Alice / Bob / Carol：工资高于全公司平均（79571.43）的人
```
🏷️ `#子查询`

### `IN` 子查询 — 按另一张表的结果筛选
```sql
SELECT name FROM employees WHERE id IN (SELECT emp_id FROM orders WHERE amount > 500);
-- Alice / David：下过 500 以上大单的员工
```
🏷️ `#子查询`

### `EXISTS` — 存在性判断
```sql
SELECT d.name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id AND e.salary > 100000);
-- Engineering：有员工工资超过 10 万的部门。EXISTS 只关心"有没有"，大表筛选常用它
```
🏷️ `#子查询`

### `WITH`（CTE）— 先算中间结果再用
```sql
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id
)
SELECT e.name, e.salary FROM employees e
JOIN dept_avg a ON e.dept_id = a.dept_id
WHERE e.salary > a.avg_salary;
-- Alice（高于部门均薪 101000）/ David（高于 70000）：工资超过本部门平均的人
-- 复杂查询拆成 CTE 一步步写，比嵌套子查询好读得多；MySQL 8.0+ / PostgreSQL / SQLite 3.8.8+ 都支持
```
🏷️ `#CTE` `#子查询`

### 窗口函数 `RANK()` — 组内排名
```sql
SELECT name, dept_id, salary,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
FROM employees;
-- 部门 1 内：Alice 1、Bob 2、Carol 3；部门 2 内：David 1、Eve 2
-- GROUP BY 会把行合并，窗口函数不合并行、每行都带排名，这是本质区别
-- 方言：MySQL 8.0+ / PostgreSQL / SQLite 3.25+ 支持，老版本 MySQL 用不了
```
🏷️ `#窗口函数` `#排名`

## ✏️ 增删改与事务

### `INSERT INTO` — 插入一行
```sql
INSERT INTO employees (id, name, dept_id, salary, manager_id, hired_on)
VALUES (8, 'Henry', 2, 71000, 4, '2024-06-01');
-- 明确写出列名，别依赖表的列顺序；多行插入 VALUES 后用逗号连多组括号
```
🏷️ `#写入`

### `UPDATE ... WHERE` — 更新行
```sql
UPDATE employees SET salary = salary + 5000 WHERE name = 'Eve';
-- Eve 工资 68000 → 73000，只影响 1 行
-- ⚠️ 忘了写 WHERE 就是全表更新。手滑前先把同样的条件跑一遍 SELECT 数一下行数
```
🏷️ `#写入` `#高危`

### `DELETE ... WHERE` — 删除行
```sql
DELETE FROM orders WHERE amount < 100;
-- 删掉 1 行（89.99 的小单）。同样：没有 WHERE 就是清空整张表
-- 只想软删的业务表通常加 deleted_at 字段标记，而不是真 DELETE
```
🏷️ `#写入` `#高危`

### 事务 — 要么全成，要么全回滚
```sql
BEGIN;
UPDATE employees SET dept_id = 2 WHERE name = 'Frank';
COMMIT;
-- Frank 从 HR 调到 Sales。中途任何一步出错就 ROLLBACK，改动全部撤销，就像没发生过
-- 转账、扣库存这类"几步必须绑在一起"的操作必须用事务。MySQL 里 START TRANSACTION 与 BEGIN 等价
```
🏷️ `#事务` `#写入`

### `UPSERT` — 有则更新，无则插入
```sql
INSERT INTO departments (id, name) VALUES (3, 'People')
ON CONFLICT(id) DO UPDATE SET name = excluded.name;
-- id=3 已存在 → 部门名 HR 改成 People；不存在就会新插入一行
-- 这是 SQLite / PostgreSQL 写法；MySQL 用 INSERT ... ON DUPLICATE KEY UPDATE name = VALUES(name)
```
🏷️ `#写入` `#方言`

## 🗂️ 索引与表结构

### `CREATE TABLE` — 建表
```sql
CREATE TABLE projects (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  lead_id INTEGER REFERENCES employees(id)
);
-- 主键、非空、外键在建表时就定好，别指望应用层兜底
-- 方言：INTEGER PRIMARY KEY 在 SQLite 是自增行 id；
--       MySQL 写 INT AUTO_INCREMENT PRIMARY KEY；
--       PostgreSQL 写 INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY（或老写法 SERIAL）
```
🏷️ `#DDL` `#建表`

### `ALTER TABLE` — 给表加一列
```sql
ALTER TABLE employees ADD COLUMN email TEXT;
-- 新列默认 NULL。加完先补数据、再考虑加 NOT NULL 约束，分步走不容易锁表出事
```
🏷️ `#DDL`

### `CREATE INDEX` — 给常查的列建索引
```sql
CREATE INDEX idx_employees_dept ON employees(dept_id);
-- 之后 WHERE dept_id = ? 走索引不再全表扫
-- 验证：SQLite 用 EXPLAIN QUERY PLAN 看是否命中索引，MySQL / PostgreSQL 用 EXPLAIN
-- 索引不是越多越好：每个索引都会拖慢写入，只给高频查询条件建
```
🏷️ `#DDL` `#索引` `#性能`

### `DROP` — 删除索引与表
```sql
DROP INDEX idx_employees_dept;
-- MySQL spelling: DROP INDEX idx_employees_dept ON employees;
DROP TABLE projects;
-- ⚠️ 不可恢复。上生产前先确认没有别的表外键引用它，先备份再动手
```
🏷️ `#DDL` `#高危`

---

## 为什么是这 30 条

- **覆盖 90% 日常**：查、算、连、改——每天写的 SQL 翻来覆去就是这些，冷门语法不收。
- **条条带预期结果**：每条都注明跑完应该长什么样，你对不上就是理解错了地方，当场纠正。
- **方言不装糊涂**：LIMIT、ILIKE、UPSERT、自增主键这些各家不一样的，逐条写明谁是谁的写法。

## 欢迎 PR

发现 SQL 写错了、方言标注不对、翻译不到位——欢迎提 Pull Request。新增条目请保持统一格式（一句话说明 / sql 代码块 / 标签），**提交前请用真实数据库跑一遍验证**，结果对不上预期的不要收。

## 📚 更多速查表

同一作者的其他速查表，同样有中/西/葡三语版本：

- [FFmpeg 命令速查表](https://github.com/Ezra-Zhao/ffmpeg-cheatsheet) — 28 条音视频处理命令
- [Git 实战速查表](https://github.com/Ezra-Zhao/git-cheatsheet) — 30 条日常+救命命令
- [Docker 命令速查表](https://github.com/Ezra-Zhao/docker-cheatsheet) — 28 条容器命令
- [curl 命令速查表](https://github.com/Ezra-Zhao/curl-cheatsheet) — 29 条 API 调试命令
- [正则表达式速查表](https://github.com/Ezra-Zhao/regex-cheatsheet) — 30 条实战正则模式
- [jq 命令速查表](https://github.com/Ezra-Zhao/jq-cheatsheet) — 30 条 JSON 处理 filter
- [终端神器速查表](https://github.com/Ezra-Zhao/terminal-superpowers-cheatsheet) — 14 个终端效率工具
- [Adobe 免费替代速查表](https://github.com/Ezra-Zhao/adobe-free-alternatives-cheatsheet) — 16 个 Adobe 免费替代品
- [退订省钱速查表](https://github.com/Ezra-Zhao/desubscribe-cheatsheet) — 10 组退订替代，年省约 $1,260
- [kubectl 命令速查表](https://github.com/Ezra-Zhao/kubectl-cheatsheet) — Kubernetes 高频命令
- [SSH 速查表](https://github.com/Ezra-Zhao/ssh-cheatsheet) — 登录、隧道与文件传输
- [Vim 命令速查表](https://github.com/Ezra-Zhao/vim-cheatsheet) — 30 条编辑、搜索替换与宏命令
- [tmux 命令速查表](https://github.com/Ezra-Zhao/tmux-cheatsheet) — 30 条会话、窗口与窗格命令
- [systemd 命令速查表](https://github.com/Ezra-Zhao/systemd-cheatsheet) — 30 条服务管理与日志命令

## License

MIT © 2026 Guangyi Zhao，详见 [LICENSE](LICENSE)。
