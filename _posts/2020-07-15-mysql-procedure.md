---
layout:     post
title:      MySQL PROCEDURE
subtitle:   存储过程
date:       2020-07-15
author:     Static
header-img: 
catalog: true
tags:
    - mysql
    
---

## 1. What?

> MySQL 5.0 版本开始支持存储过程

**存储过程**（Stored Procedure）是一种在数据库中存储复杂程序，以便外部程序调用的一种数据库对象。

存储过程是为了完成特定功能的SQL语句集，经编译创建并保存在数据库中，用户可通过指定存储过程的名字并给定参数(需要时)来调用执行。

存储过程思想上很简单，就是数据库 SQL 语言层面的代码封装与重用。

> 类似于Java中的方法。


## 2. Why?

1. 封装性

> 减少重复性代码

2. 可增强 SQL 语句的功能和灵活性

> 支持复杂的运算，多个SQL语句并行

3. 可减少网络流量

> 存储过程是在服务器端运行的，执行快，网络传输也只是传送命令。

4. 高性能

> 存储过程执行一次后，产生的二进制代码就驻留在缓冲区，在以后的调用中，只需要从缓冲区中执行二进制代码即可，从而提高了系统的效率和性能。

5. 提高数据库的安全性和数据的完整性

> 使用存储过程可以完成所有数据库操作，并且可以通过编程的方式控制数据库信息访问的权限。

## 3. How?  

#### 1. 基本用法

```
CREATE PROCEDURE <过程名> ( [过程参数[,…] ] ) <过程体>
[过程参数[,…] ] 格式
[ IN | OUT | INOUT ] <参数名> <类型>
```

> `过程名`类似 `Java中的方法名`，`过程参数` 类似 `入参`，`过程体` 类似 `方法体`

### 2. 案例

> 写一个程序循环建表

```sql
-- 控制台中需要用 && 判断是否为一整句，结束后 设为默认 ;
DELIMITER &&

-- 循环建表
DROP PROCEDURE IF EXISTS create_test_table&&

CREATE PROCEDURE create_test_table(IN begin_index INT, IN end_index INT)
  BEGIN
    DECLARE i INT;
    SET i = begin_index;
    WHILE i < end_index + 1 DO
      SET @sql_create_table = concat(
          'CREATE TABLE IF NOT EXISTS test_db_log_', i,
          "(
            `id` int(11) NOT NULL AUTO_INCREMENT,
             `createTime` datetime DEFAULT NULL COMMENT '创建时间',
              PRIMARY KEY (`id`),
          INDEX `createTime` (`createTime`) USING BTREE
          ) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='pattern'
          ");
      PREPARE sql_create_table FROM @sql_create_table;
      EXECUTE sql_create_table;
      SET i = i + 1;
    END WHILE;
  END&&

DELIMITER ;

-- 执行程序
CALL create_test_table(0, 10);
```
> 修改程序，用 `ALTER`，删除程序用 `DROP`

<html>
    <img src="/img/mysql/MySQL-PROCEDURE.png" width="500" height="600" /> 
</html>