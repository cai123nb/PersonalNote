# MYSQL 语句小结

## SQL 概述

SQL(Structured Query Language)是关系数据库中标准语言, 也是通用的功能极强的关系数据库语言.

SQL 是在 1974 年由 Boyce 和 Chamberlin 提出,最早叫做 Sequel, 并在 IBM 公司研制的关系数据库管理系统原型 System R 上实现.1986 年 10 月, 美国国家标准局(American National Standard Institute)的数据库委员会 X3H2 批准了 SQL 作为关系数据库语言的美国标准. 同年公布了 SQL 标准文本(简称 SQL-86).1987 年, 国际标准化组织(International Organization for Standardization, ISO)也通过了该标准.

## 新建语句(CREATE)

### 新建数据库

```SQL
/*创建数据库*/
CREATE DATABASE test;
//切换数据库
use test;
```

### 新建表

```SQL
/*情景1 学生表*/
CREATE TABLE Student (
   Sno CHAR(9) PRIMARY KEY,   /*列级完整性约束, Sno为主键*/
   Sname CHAR(20) UNIQUE,      /*唯一值*/
   Ssex CHAR(2),
   Sage SMALLINT,
   Sdept CHAR(20)
);
/*情景2 课程表*/
CREATE TABLE Course (
   Cno CHAR(4) PRIMARY KEY,   /*列级完整性约束, Cno为主键*/
   Cname CHAR(20) UNIQUE,      /*取唯一值*/
   Cpno CHAR(2),            /*先修课(必须先完成的课程)*/
   Ccredit SMALLINT,
   FOREIGN KEY (Cpno) REFERENCES Course(Cno) /*表级完整性约束, Cpno外键, 被参照的表是Course, 被参照的列是Cno.*/
);
/*情景3 学生选课表*/
CREATE TABLE SC (
   Sno CHAR(9),
   Cno CHAR(4),
   Grade SMALLINT,
   PRIMARY KEY(Sno,Cno),   /*主键由两个键构成, 必须作为表级完整性进行定义*/
   FOREIGN KEY(Sno) REFERENCES Student(Sno),   /*外键约束*/
   FOREIGN KEY(Cno) REFERENCES Course(Cno)      /*外键约束*/
);
/*情景4 学生选课表添加完整性约束细节*/
CREATE TABLE SC (
   Sno CHAR(9),
   Cno CHAR(4),
   Grade SMALLINT,
   PRIMARY KEY(Sno,Cno),   /*主键由两个键构成, 必须作为表级完整性进行定义*/
   FOREIGN KEY(Sno) REFERENCES Student(Sno)
      ON DELETE CASCADE   /*删除Student表中的元组时, 级联删除SC表中的元组*/
      ON UPDATE CASCADE,   /*更新Student表中的元组时, 级联更新SC表中的元组*/
   FOREIGN KEY(Cno) REFERENCES Course(Cno)
      ON DELETE NO ACTION /*当删除Course表中的元组造成与SC表中元组不一致时, 拒绝删除(不设置, 默认为该选项)*/
      ON UPDATE CASCADE /*当更新Course表中的Cno时, 级联更新SC表中的相应元组*/
);
```

表的常见数据类型见附录 1.

### 新建模式

本质上:一个命名空间,一个域,每个表都属于一个模式.

```SQL
CREATE SCHEMA <模式名> AUTHORIZATION <用户名>
CREATE SCHEMA TEST /*为TEST用户创建一个TEST空间*/
CREATESCHEMA"S-T" AUTHORIZATION WANG /*为用户WANG创建了一个S-T空间*/
   CREATE TABLE TAB1(COL1 SMALLINT, COL2 INT); /*并在该空间创建了一个表TAB1*/
```

### 新建索引

```SQL
CREATE [UNIQUE/CLUSTER] INDEX <索引名> ON  <表名>(<列名>[<次序][,<列名>[<次序]]...)
CREATE UNIQUE INDEXS Stusno ON Student(Sno);
CREATE UNIQUE INDEX SCno ON SC(Sno ASC, Cno DESC);
```

- UNIQUE: 每一个索引值都对应唯一的数据记录.
- CLUSTER: 建立的是聚簇索引.

## 删除语句(DROP)

### 删除数据库(DROP DATABASE)

```SQL
DROP DATABASE test;
```

### 删除表(DROP DATABASE)

```SQL
DROP TABLE <表名> [RESTRICT|CASCADE]
DROP TABLE SC RESTRICT;   //删除表SC
DROP TABLE Student CASCADE;   //删除表Student, SC表也会被级联删除
```

RESTRICT: 删除表非常严格, 如 CHECK, 外键, 视图, 触发器等, 如果存在依赖, 则不能删除.
CASCADE: 删除没有限制, 删除表的时候, 相关依赖的对象都会被一起删除.

### 删除索引(DROP INDEX)

```SQL
DROP INDEX <索引名> ON <表名>
DROP INDEX Stusname ON Student;
```

### 删除模式(DROP SCHEMA)

```SQL
DROP SCHEMA <模式名> [CASCADE|RESTRICT]
DROP SCHEMA ZHANG CASCADE;    //删除模式ZHANG, 同时删除在内部的定义的所有表和视图
```

CASCADE: 彻底删除, 包括内部创建的所有数据库对象(表, 视图等).
RESTRICT: 删除时检查是否存在下属数据库对象, 如果存在拒绝删除.

### 删除表中的数据(DELETE FROM <表名> [WHERE <条件>]

```SQL
DELETE FROM <表名> [WHERE<条件>]
DELETE FROM SC; //清空SC表
DELETE FROM Student WHERE Sno='201215128'; //删除学号为201215128的学生.
DELET FROM SC WHERE Sno IN (SELECT Sno FROM Student WHERE Sdept='CS'); //删除计算机系的所有学生选课记录
```

## 修改语句(ALTER)

### 修改索引(ALTER INDEX)

```SQL
ALTER INDEX <旧的索引名> RENAME TO <新的索引名>
ALTER INDEX SCno RENAME TO SCSno
```

### 修改表(ALTER TABLE)

```SQL
ALTER TABLE <表名>
[ADD [COLUMN] <新列名><数据类型>[完整性约束]]
[ADD <表级完整性约束>]
[DROP [COLUMN]<列名>[CASCADE|RESTRICT]]
[DROP CONSTRAINT <完整性约束名> [CASCADE|RESTRICT]]
[ALTER COLUMN <列名><数据类型>]
ALTER TABLE Student ADD S_entrance DATE;    /* 添加入学时间列 */
ALTER TABLE Student ALTER COLUMN Sage INT; /* 修改年龄为整数 */
ALTER TABLE Course ADD UNIQUE(Cname);   /* 给Cname添加唯一约束 */
```

### 修改表中的数据

```SQL
UPDATE <表名> SET <列名>=<表达式>[,<列名>=<表达式>]...[WHERE<条件>]
UPDATE Student Set Sage=22 WHERE Sno='201215121';/*将学生201215121的年龄设置为22岁*/
UPDATE Student Set Sage = Sage + 1; /*将所有学生的年龄加一*/
UPDATE SC SET Grade=0 WHERE Sno IN (SELECT Sno FROM Student WHERE Sdept = 'CS'); /*将所有计算机系学生的成绩归零*/
/*插入数据*/
INSERT INTO <表名> [(<属性列>[,<属性列2>]...)] VALUES (<常量1>[,<常量2>]...);
INSERT INTO Student (Sno,Sname,Ssex,Sdept,Sage) VALUES('201215128','程东','男','IS',18);
INSERT INTO Student VALUES('201215128','程东','男','IS',18);
INSERT INTO Dept_age(Sdept,Avg_age) SELECT Sdept,AVG(Sage) FROM Student GROUP BY Sdept; /*向Dept_age表中插入每个系的学生的平均年龄*/
```

## 数据查询

SQL 提供了 SELECT 语句进行数据查询, 该语句具有灵活的使用方式和丰富的功能. 一般格式为:

SELECT [ALL|DISTINCT] <目标列表达式>[,<目标列表达式>...]
FROM <表名或者视图名> [,<表名或者视图名>...] | (`<SELECT 语句>`) [AS] <别名>
[WHERE <条件表达式>]GROUP BY <列名 1>[HAVING <条件表达式>]]
[ORDER BY <列名 2> [ASC|DESC]];

### 单表查询

单表查询: 只涉及到一个表的查询.

```SQL
/*查询指定列*/
SELECT Sno, Sname FROM Student; /* 查找所有全体学生学号和姓名 */
/*查询所有列*/
SELECT * FROM Student; /* 查询所有的学生的详细记录 */
/*查询经过计算的列*/
SELECT Sname,2014-Sage FROM Student; /*查询全体学生姓名和出生年份(第二个属性是计算表达式)*/
SELECT Sname, 'Year of Birth:', 2014-Sage, LOWER(Sdept); /*查询全体学生的姓名, 出生年份和所在的院系*/
SELECT Sname, 2014-Sage BIRTHDAY, LOWER(Sdept); /*查询全体学生的姓名, 出生年份和所在的院系(这里第二列的列标题就变成(BIRTHDAY)*/
/*去除重复组*/
SELECT DISTINCT Sno FROM SC; /*查询所有的学生学号,去重(默认就是ALL不去重)*/
/*添加查询条件*/
/*比较大小: =, >, <, >=, <=, !=, <>, !>, !<. */
SELECT Sname FROM Student WHERE Sdept='CS';   /*查询计算机学院的全体学生名单*/
SELECT Sname, Sage FROM Student WHERE Sage<20;   /*查询所有年龄在20岁以下的学生名单和年龄*/
SELECT DISTINCT Sno FROM SC WHERE Grade<60; /*查询考试成绩不及格的学生学号(这里使用DISTINCT进行去重, 多次挂科也只算一次)*/
/*确定范围: BETWEEN ... AND...*/
SELECT Sname,Sdept,Sage FROM Student WHERE Sage BETWEEN 20 AND 23; /*查询20-23岁之间的学生姓名,系别和年龄*/
SELECT Sname,Sdept,Sage FROM Student WHERE Sage NOT BETWEEN 20 AND 23; /*查询年龄不在20-23岁之间的学生姓名,系别和年龄*/
/*确定集合: IN */
SELECT Sname,Ssex FROM Student WHERE Sdept IN('CS','MA','IS');/*查询计算机科学系(CS),数学系(MA)和信息系(IS)学生的姓名和性别*/
SELECT Sname,Ssex FROM Student WHERE Sdept NOT IN('CS','MA','IS');/*查询既不是计算机科学系(CS),数学系(MA), 也不是信息系(IS)的学生的姓名和性别*/
/*字符串匹配: [NOT] LIKE `<匹配串>` [ESCAPE '<换码字符>'] 通配符: %, 任意长度字符串, _, 任意单个字符 */
SELECT * FROM Student WHERE Sno LIKE '201215121'; /*查询学号为201215121学生的详情情况(由于没有使用通配符, 等价于=)*/
SELECT Sname,Sno,Ssex FROM Student WHERE Sname LIKE '刘%'; /*查询所有姓刘的学生的姓名,学号和性别*/
SELECT Sname,Sno,Ssex FROM Student WHERE Sname LIKE '欧阳_'; /*查询姓"欧阳"且全名为三个汉字的学生的姓名,学号和性别*/
SELECT Sname,Sno FROM Student WHERE Sname LIKE '_阳%'; /*查询名字中第二字为"阳"的学生姓名和学号*/
SELECT Sname,Sno FROM Student WHERE Sname NOT LIKE '刘%'; /*查询所有不姓刘的学生的姓名,学号*/
SELECT Cno,Ccredit FROM Course WHERE Cname LIKE 'DB\_Design' ESCAPE '\\'; /*实际输入时一个斜杠即可,查找DB_Design课程的课程号和学分(使用ESCAPE转换成换码字符,\后的_不在具备通配符的作用)*/
SELECT * FROM Course WHERE Cname LIKE 'DB\_%i__' ESCAPE '\\'; /*实际输入时一个斜杠即可,查询以"DB_"开头,且倒数第三个字符为i的课程的详细信息*/
/*空值查询*/
SELECT Sno, Cno FROM SC WHERE Grade IS NULL; /*查询缺考的学生编号和课程号, 这里IS不能使用=*/
SELECT Sno, Cno FROM SC WHERE Grade IS NOT NULL; /*查询所有有成绩的学生学号和课程号*/
/*多重条件查询: AND和OR连接多个查询条件*/
SELECT Sname FROM Student WHERE Sdept='CS' AND Sage<20; /*查询计算机学院年龄在20岁以下的学生姓名*/
SELECT Sname,Ssex FROM Student WHERE Sdept='CS' OR Sdept='MA' OR Sdept='IS';/*查询计算机科学系(CS),数学系(MA)和信息系(IS)学生的姓名和性别(IN可以转换为OR查询)*/
/*ORDER BY子句,按照一个或多个属性列排序, 升序(ASC)或者降序(DESC), 默认升序*/
SELECT Sno,Grade FROM SC WHERE Cno='3' ORDER BY Grade DESC; /* 查询选修了三号课程学生的学号和成绩,  查询结果按照成绩降序排列*/
SELECT * FROM Student ORDER BY Sdept, Sage DESC; /* 查询全体学生情况, 查询结果按所在系号升序排列, 同一系的学生按照年龄降序排列*/
/* 聚集函数, COUNT(*): 统计元组个数, COUNT([DISTINCT|ALL] <列名>): 统一一列中的值的个数, SUM([DISTINCT|ALL] <列名>): 计算一列值的总和(必须是数值), AVG,MAX,MIN类似*/
SELECT COUNT(*) FROM Student; /* 查询总人数 */
SELECT COUNT(DISTINCT Sno) FROM SC; /* 查询选修了课程的学生人数 (学生可能选修多门, 需要去重)*/
SELECT AVG(Grade) FROM SC WHERE Sco='1'; /*查询选修1号课的学生的平均成绩*/
SELECT MAX(Grade) FROM SC WHERE Sco='1';/*查询选修1号课的学生的最高成绩*/
SELECT SUM(Ccredit) FROM SC,Course WHERE Sno='201215012' AND SC.Cno=Course.Cno; /* 查询学生201215012选修课程的总学分*/
/* 注意聚集函数一般都是跳过空值处理(除了COUNT(*)), 并且只能用于SELECT子句和GROUP BY中的HAVING子句.*/
/* GROUP BY子句: 将查询结果按照某一列或多列的值进行分组, 值相等的为一组.: 主要是为了细化聚集函数的作用对象 */
SELECT Cno,COUNT(Sno) FROM SC GROUP BY Cno;   /* 查询每个课程的选课人数 */
SELECT Sno FROM SC GROUP BY Sno HAVING COUNT(*) > 3;   /* 查询选修了三门以上课程的学生学号. (先使用GROUP BY按照Sno进行分组, 然后使用聚集函数COUNT对每一组进行计数,最后使用HAVING给出选择条件)*/
/* 注意HAVING是作用于组, 而WHERE是作用于一般的表或者视图(不能使用聚集函数). 如查询平均成绩大于90分的学号的学生学号和平均成绩: */
SELECT Sno,AVG(Grade) FROM SC WHERE (AVG(Grade)>= 90) GROUP BY Sno; /* 这是错误的语句 */
SELECT Sno,AVG(Grade) FROM SC GROUP BY Sno HAVING AVG(Grade)>=90; /* 这是正确的语句 */
```

### 多表查询

多表查询: 亦称连接查询, 连接两个及以上的表进行查询, 连接方式: 等值连接, 非等值连接, 自身连接, 外连接查询, 复合条件连接查询.

```SQL
/*等值连接和非等值连接: 使用Where子句来完成两个表的连接, 常用:=,>,<,>=,<=,!=,<>等等*/
SELECT Student.*, SC.* FROM Student,SC WHERE Student.Sno=SC.Sno; /*查询每个学生的选修课情况*/
SELECT Student.Sno, Sname FROM Student,SC WHERE Student.Sno=SC.Sno AND SC.Cno='2' AND SC.Grade>90; /*查询选修了2号课程且成绩在90分以上的学生学号和姓名*/
/*自身连接: 连接操作不仅可以在两个表之间进行, 也可以是一个表和自己进行连接.*/
SELECT FIRST.Cno, SECOND.Cpno FROM Course FIRST, Course SECOND WHERE FIRST.Cpno=Second.Cno; /*查询每一门课的先修课程*/
/*外连接: 通常的连接操作中只有满足连接条件的元组才能作为结果输出. 如查询每个学生的选修情况, 如果有的学生没有选修课程(SC中没有记录),就不会出现在结果里面, 但是如果我们想要显示出来(全部学生信息),但是选修课属性为NULL*/
SELECT Student.Sno,Sname,Ssex,Sage,Sdept,Cno,Grade FROM Student LEFT OUTER JOIN SC (Student.Sno=SC.Sno);/*左连接(T1 LEFT OUTER JOIN T2),右连接(T1 RIGHT OUTER JOIN T2)*/
/*多表连接: 连接2个以上的表进行查询操作*/
SELECT Student.Sno,Sname,Cname,Grade FROM Student,SC,Course WHERE Student.Sno=SC.Sno AND SC.Cno=Course.Cno; /*查询每个学生的学号,姓名,选修的课程名称和成绩*/
/*嵌套查询: 嵌套多层的SELECT-FROM-WHERE语句*/
/*嵌套1: 带有IN谓词的子查询*/
/*查询和"刘晨"在一个系的学生*/
SELECT Sno,Sname,Sdept FROM Student WHERE Sdept IN (SELECT Sdept FROM Student WHERE Sname='刘晨'); /*第一步: 查找刘晨所在系CS, 第二步查找所有在CS系的学生信息*/
/*查询选修了课程名为"信息系统"的学生学号和姓名*/
SELECT Sno,Sname FROM Student WHERE Sno IN (SELECT Sno FROM SC WHERE Cno IN (SELECT Cno FROM Course WHERE Cname='信息系统')); /*第一步: 从Course中找出"信息系统"的课程号3号, 第二步: 从SC表中查找所有选修了3号课程的学生学号, 第三步:从学生信息表中取出Sno和Sname.*/
/*嵌套2: 带有比较运算符的子查询: 如果知道子查询返回的是一个单值, 可以使用<,>,<=,>=,!=,<>等比较运算符.*/
/*找出每个学生超过他自己选修课程平均成绩的课程号*/
SELECT Sno,Cno FROM SC x WHERE Grade>= (SELECT AVG(Grade) FROM SC y WHERE y.Sno=x.Sno); /*第一步: 从父查询中取出一个学号Sno传递给子查询, 第二步: 子查询查找所有该学号的成绩并计算平均值, 第三步: 父查询进行比较取出满足条件的元组*/
/*嵌套3: 带有ANY或ALL谓词的子查询, >ANY: 大于查询中的某个值, >ALL:大于查询中的所有值, !=ANY:不等于子查询中的所有值,=ANY:等于查询中的某个值. 其它类似,可以自行推断.*/
/*查找非计算机系比所有计算机系所有学生年龄都小的学生姓名和年龄*/
SELECT Sname,Sage FROM Student WHERE Sage < ALL(SELECT Sage FROM Student WHERE Sdept='CS') AND Sdept <> 'CS';
/*嵌套4: 带有EXISTS谓词的子查询, 带有EXISTS谓词的子查询不会返回任何数据, 只产生TRUE和FALSE*/
/*查询所有选修了1号课程的学生姓名*/
SELECT Sname FROM Student WHERE EXISTS (SELECT * FROM SC WHERE Sno=Student.Sno AND Cno='1'); /*第一步: 从Student表中依次取出学号送到子查询, 子查询中查找SC中是否存在选课记录为该学号且课程号为1的记录,返回真假*/
/*查询没有选修了1号课程的学生姓名*/
SELECT Sname FROM Student WHERE NOT EXISTS (SELECT * FROM SC WHERE Sno=Student.Sno AND Cno='1'); /*逻辑类似*/
/*查询选修了全部课程的学生姓名*/
SELECT Sname FROM Student WHERE NOT EXISTS (SELECT * FROM Course WHERE NOT EXISTS (SELECT * FROM SC WHERE Sno=Student.Sno AND Cno=Course.Cno));/*逻辑思路: 没有一门课是他没选的. 第一步: Student中依次取出学号送到子查询1, 第二步子查询从Course表中依次取出课程号送到子查询2, 子查询3查询SC是否存在该课程的选课记录*/
/*查询至少选修了学生201215122选修的全部课程的学生号码*/
SELECT DISTINCT Sno FROM SC SCX WHERE NOT EXISTS (SELECT * FROM SC SCY WHERE SCY.Sno='201215122' AND NOT EXISTS (SELECT * FROM SC SCZ WHERE SCZ.Sno=SCX.Sno AND SCZ.Cno=SCY.Cno));
/*逻辑思路: 不存在这样的课程y, 学生201215122选修了y, 但是学生x没有选修. 第一步: 依次从SC表中取出Sno送到子查询1, 子查询1: 选择201215122学生所选的所有课程号依次送到子查询3, 子查询3: 查找SC中是否存在这样的记录*/
/*集合查询: 包括并操作(UNION), 交操作(INERSECT), 差操作(EXCEPT), 但是可以使用多重条件查询替代. 略过*/
/*派生表查询: FROM中嵌套SELECT语句生成的子表.*/
```

## 附录

### 附录 1: 常见的数据类型

| 数据类型                        | 含义                                        |
| :------------------------------ | :------------------------------------------ |
| CHAR(n),CHARACTER(n)            | 长度为 N 的定长字符串                       |
| VARCHAR(n), CHARACTERVARYING(n) | 最大长度为 N 的变长字符串                   |
| CLOB                            | 字符串大对象                                |
| BLOB                            | 二进制大对象                                |
| INT,INTEGER                     | 长整数(4 字节)                              |
| SMALLINT                        | 短整数(2 字节)                              |
| BIGINT                          | 大整数(8 字节)                              |
| NUMERIC(p,d)                    | 定点数, 由 p 位小数组成,小数点后有 d 位小数 |
| DECIMAL(p,d), DEC(P,d)          | 同上                                        |
| REAL                            | 取决于机器精度的单精度浮点数                |
| DOUBLE PRECISION                | 取决于机器精度的双精度浮点数                |
| FLOAT(n)                        | 可选精度的浮点数, 精度至少为 n 位数字       |
| BOOLEAN                         | 逻辑布尔量                                  |
| DATE                            | 日期, 包含年月, 格式为 YYYY-MM-DD           |
| TIME                            | 时间, 包含时,分,秒, 格式为 HH:MM:SS         |
| TIMESTAMP                       | 时间戳类型                                  |
| INTERVAL                        | 时间间隔类型                                |

## 参考文献

王, 珊, 萨, 师煊. 数据库系统概论[M]. 高等教育出版社, 2006.
