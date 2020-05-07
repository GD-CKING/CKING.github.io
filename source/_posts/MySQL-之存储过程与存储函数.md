---
title: MySQL 之存储过程与存储函数
date: 2020-05-06 16:12:47
tags: MySQL
categories: MySQL
---

## 存储过程

### 什么是存储过程

存储过程是一组为了完成某项特定功能的 SQL 语句集，其实质上就是一段存储在数据库中的代码，它可以有声明式的 SQL 语句（如 CREATE，UPDATGE，SELETE 等语句）和过程式 SQL 语句（如 IF...THEN...ELSE... 控制结构语句）组成。存储过程思想上很简单，就是数据库 SQL 语言层面的代码封装和重用。



### 存储过程的优缺点

#### 优点

- **可增强 SQL 语言的功能和灵活性**。存储过程可以用流程控制语言编写，有很强的灵活性，可以完成复杂的判断和较复杂的运算。

- **良好的封装性**。存储过程被创建后，可以在程序中被多次调用，而不必担心重写编写该存储过程的 SQL 语句。

- **高性能**。存储过程执行一次后，其执行规划就驻留在高速缓冲存储器中，以后的操作中只需要从高速缓冲器中调用已编译好的二进制代码执行即可，从而提高了系统性能。



#### 缺点

存储过程，往往定制化与特定的数据库上，因为支持的编程语言不同。当切换到其他厂商的数据库系统时，需要重写原有的存储过程。



### 创建存储过程

#### DELIMITER 定界符

在 SQL 中服务器处理 SQL 语句默认是以分号作为语句的结束标志，然而在创建存储过程时，存储过程体中可能包含多条 SQL 语句，这些 SQL 语句如果仍以分号作为语句结束符，那么服务器在处理时会以第一条 SQL 语句处的分号作为整个程序的结束符，而不再去处理后面的 SQL



为解决这个问题，通常使用 `DELIMITER` 命令，将 SQL 语句的结束符临时修改为其他符合。**DELIMITER 语法格式为：**

```mysql
DELIMITER $$
```



`$$` 是用户定义的结束符，通常这个符合可以是一些特殊的符号。另外应该避免使用反斜杠，因为它是转义字符。若希望换回默认的分号作为结束标记，只需要在命令行输入下面的 SQL 语句即可。

```mysql
DELIMITER ;
```



#### 创建存储过程

在 MySQL 中，使用 `CREATE PROCEDURE` 语句来创建存储过程。

```mysql
CREATE PROCEDURE p_name([proc_parameter[, ...]])
routine_body
```



其中，语法项 `proc_parameter` 的语法格式是：

```mysql
[IN | OUT | INOUT]parame_name type
```

- `p_name` 用于指定存储过程的名称

- `proc_parameter` 用于指定存储过程中的参数列表。其中，语法项 `parame_name` 为参数名，`type` 为参数的类型（类型可以是 MySQL 中任意的有效数据类型）。MySQL 的存储过程支持三种类型的参数，即输入参数 `IN`，输出类型 `OUT`，输入输出参数 `INOUT`。输入参数是使数据可以传递给一个存储过程；输出参数是用于存储过程需要返回的一个操作结果；输入输出参数既可以充当输入参数也可以充当输出结果

- 语法项 `routine_body` 表示存储过程的主体部分，也成为存储过程体，其包含了需要执行的 SQL。过程体以关键字 `BEGIN` 开始，以关键字 `END` 结束。若只有一条 SQL 可以忽略 `BEGIN...END` 标志



#### 局部变量

在存储过程中可以声明局部变量，用来存储过程体中的临时结果。在 MySQL 中使用 `DECLARE` 语句来声明局部变量

```mysql
DECLARE var_name type [DEFAULT VALUE]
```



`var_name` 用于指定局部变量的名称；`type` 用来声明变量的类型；`DEFAULT` 用来指定默认值，如果没有指定则为 NULL



> 注意：局部变量只能在存储过程体的 BEGIN...END 语句块中；局部变量必须在存储过程体的开头处声明；局部变量的作用范围仅限于声明它的 BEGIN...END 语句块，其他语句块中的语句不可以使用它。



#### 用户变量

用户变量一般以 @ 开头

注意：滥用用户变量会导致程序难以理解及管理



#### SET 语句

在 MySQL 中通过 SET 语句对局部变量赋值，其格式是：

```mysql
SET var_name = expr[, var_name2 = expr] ...
```



#### SELECT.....INTO 语句

在 MySQL 中，可以使用 SELECT....INTO 语句把选定的列的值存储到局部变量中，格式是：

```mysql
SELECT col_name[,...] INTO var_name[,...] table_expr
```



其中 `col_name` 用于指定列名；`var_name` 用于指定要赋值的变量名；`table_expr` 表示 SELECT 语句中 FROM 后面的部分



> 注意：SELECT...INTO 语句返回的结果集只能有一行数据



#### 流程控制语句

##### 条件判断语句 



**if-then-else 语句**

```mysql
mysql > DELIMITER &&  
mysql > CREATE PROCEDURE proc2(IN parameter int)  
     -> begin 
     -> declare var int;  
     -> set var=parameter+1;  
     -> if var=0 then 
     -> insert into t values(17);  
     -> end if;  
     -> if parameter=0 then 
     -> update t set s1=s1+1;  
     -> else 
     -> update t set s1=s1+2;  
     -> end if;  
     -> end;  
     -> &&  
mysql > DELIMITER ;
```



**case 语句**

```mysql
mysql > DELIMITER &&  
mysql > CREATE PROCEDURE proc3 (in parameter int)  
     -> begin 
     -> declare var int;  
     -> set var=parameter+1;  
     -> case var  
     -> when 0 then   
     -> insert into t values(17);  
     -> when 1 then   
     -> insert into t values(18);  
     -> else   
     -> insert into t values(19);  
     -> end case;  
     -> end;  
     -> &&  
mysql > DELIMITER ; 
```



#### 循环语句



**while .... end while**

```mysql
mysql > DELIMITER &&  
mysql > CREATE PROCEDURE proc4()  
     -> begin 
     -> declare var int;  
     -> set var=0;  
     -> while var<6 do  
     -> insert into t values(var);  
     -> set var=var+1;  
     -> end while;  
     -> end;  
     -> &&  
mysql > DELIMITER ;
```



**repeat .... end repeat**



它在执行操作后检查结果，而 while 则是执行前进行检查

```mysql
mysql > DELIMITER &&  
mysql > CREATE PROCEDURE proc5 ()  
     -> begin   
     -> declare v int;  
     -> set v=0;  
     -> repeat  
     -> insert into t values(v);  
     -> set v=v+1;  
     -> until v>=5  
     -> end repeat;  
     -> end;  
     -> &&  
mysql > DELIMITER ;
```



```
repeat
    --循环体
    until 循环条件  
end repeat;
```



**loop ... end loop**

loop 循环不需要初始条件，这点和 while 循环相似，同时和 repeat 循环一样不需要结束条件。`leave` 语句的意义是离开循环

```mysql
mysql > DELIMITER &&  
mysql > CREATE PROCEDURE proc6 ()  
     -> begin 
     -> declare v int;  
     -> set v=0;  
     -> LOOP_LABLE:loop  
     -> insert into t values(v);  
     -> set v=v+1;  
     -> if v >=5 then 
     -> leave LOOP_LABLE;  
     -> end if;  
     -> end loop;  
     -> end;  
     -> &&  
mysql > DELIMITER ;
```



**ITERATE 迭代**

```mysql
mysql > DELIMITER &&  
mysql > CREATE PROCEDURE proc10 ()  
     -> begin 
     -> declare v int;  
     -> set v=0;  
     -> LOOP_LABLE:loop  
     -> if v=3 then   
     -> set v=v+1;  
     -> ITERATE LOOP_LABLE;  
     -> end if;  
     -> insert into t values(v);  
     -> set v=v+1;  
     -> if v>=5 then 
     -> leave LOOP_LABLE;  
     -> end if;  
     -> end loop;  
     -> end;  
     -> &&  
mysql > DELIMITER ;
```



#### 游标

MySQL 中的游标可以理解为一个可迭代对象（类比 Python 中的列表、字典等可迭代对象），它可以用来存储 select 语句查询到的结果集，这个结果集可以包含多行数据，从而使我们可以使用迭代的方法从游标中依次取出每行数据。



> MySQL 游标的特点：
>
> - 只读：无法通过游标更新基础表中的数据
>
> - 不可滚动：只能按照 select 语句确定的顺序获取行。不能以相反的顺序获取行。此外。不能跳过行或跳转到结果集中的特定行
>
> - 敏感。游标分为敏感游标和不敏感游标。敏感游标指向实际数据，不敏感游标使用数据的临时副本。敏感游标比一个不敏感游标执行得更快，因为它不需要临时拷贝数据。MySQL 游标是敏感的。



##### 声明游标

游标声明必须在变量声明之后。如果在变量声明之前声明游标，MySQL 将会发出一个错误。游标必须始终与 select 语句相关联。

```mysql
DECLARE cursor_name CURSOR FOR select_statement; 
```



##### 打开游标

使用 `open` 语句打开游标，只有先打开游标才能读取数据

```mysql
open cursor_name;
```



##### 读取游标

使用 `fetch` 语句来检索游标指向的一行数据，并将游标移动到结果集中的下一行

```mysql
FETCH cursor_name INTO var_name;
```



##### 关闭游标

使用 `close` 关闭游标

```mysql
CLOSE cursor_name;
```



当游标不再使用时，应该关闭它。当使用 MySQL 游标时，还必须声明一个 `notfound` 处理程序来处理游标找不到任何行时的情况。因为每次调用 `fetch` 语句时，游标会尝试依次读取结果集中的每一行数据。当游标到达结果集的末尾时，它将无法获得数据，并且会产生一个条件。处理程序用于处理这种情况

```mysql
declare continue handler for not found set type = 1;
```



type 是一个变量，表示游标到达结果集的结尾。

```mysql
DELIMITER $$
CREATE PROCEDURE phoneDeal()
BEGIN
    DECLARE  id varchar(64);   -- id
    DECLARE  phone1  varchar(16); -- phone
    DECLARE  password1  varchar(32); -- 密码
    DECLARE  name1 varchar(64);   -- id
    -- 遍历数据结束标志
    DECLARE done INT DEFAULT FALSE;
    -- 游标
    DECLARE cur_account CURSOR FOR select phone,password,name from account_temp;
    -- 将结束标志绑定到游标
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    -- 打开游标
    OPEN  cur_account;     
    -- 遍历
    read_loop: LOOP
            -- 取值 取多个字段
            FETCH  NEXT from cur_account INTO phone1,password1,name1;
            IF done THEN
                LEAVE read_loop;
             END IF;
 
        -- 你自己想做的操作
        insert into account(id,phone,password,name) value(UUID(),phone1,password1,CONCAT(name1,'的家长'));
    END LOOP;
 
    -- 关闭游标
    CLOSE cur_account;
END;
$$
```



#### 调用存储过程

使用 `call` 语句调用存储过程

```mysql
CALL sp_name[(传参)];
```



#### 删除存储过程

使用 `drop` 语句删除存储过程

```mysql
DROP PROCEDURE sp_name
```



## 存储函数

### 什么是存储函数

存储函数和存储过程一样，都是 SQL 语句组成的代码块。



存储函数不能有输入参数，并且可以直接调用，不需要 call 语句，且必须有一条包含 RETURN 语句。



### 创建存储函数

在 MySQL 中使用 `CREATE FUNCTION`  语句创建：

```mysql
CREATE FUNCTION fun_name(par_name(par_name type[,...]))
RETURNS type
[characteristics]
fun_body
```



其中，`fun_name` 为函数名，并且名字唯一，不能与存储过程重名。`par_name` 是指定的参数，`type` 为参数类型；`RETURNS` 字句用来声明返回值和返回值类型；`fun_body` 是函数体，所有存储过程中的 SQL 在村塾函数中同样可以使用。但是存储函数体中必须包含一个 `RETURN` 语句。



characteristics 指定存储过程的特性，有以下取值：

- **LANGUAGE SQL**：说明 routine_body 部分是有 SQL 语句组成的，当前系统支持的语言为 SQL ，SQL 是 LANGUAGE 特性的唯一值。

- **[NOT] DETERMINISTIC**：指明存储过程执行的结果是否确定。`DETERMINISTIC` 表示结果是确定的，每次执行存储过程时，相同的输入会得到相同的输出；`NOT DETERMINISTIC` 表示结果是不确定的，相同的输入可能得到不同的输出。如果没有指定任意一个值，默认为 NOT

- **[CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA]**：指明子程序使用 SQL 语句的限制。`CONTAINS SQL` 表明子程序包含 SQL 语句，但不包含读写数据语句；`NO SQL` 表明子程序不包含 SQL 语句；`READS SQL DATA` 说明子程序包含读数据库的语句；`MODIFIES SQL DATA` 表明子程序包含写数据的语句。默认情况下，系统会指定为 CONTAINS SQL

- **SQL SECURITY [DEFINER | INVOKER]**：指明谁有权限来执行。`DEFINER` 表明只有定义者才能执行。`INVOKER` 表示拥有权限的调用者可以执行，默认情况下，系统指定我 DEFINER

- **COMMENT 'string'**：注释信息，用于描述存储过程或函数



```mysql
DELIMITER $$
CREATE FUNCTION getAnimalName(animalId int) RETURNS VARCHAR(50)
DETERMINISTIC
begin
   declare name VARCHAR(50);
   set name=(select name from animal where id=animalId);
   return (name);
end$$
delimiter;

-- 调用
select getAnimalName(4)
```



## 参考资料

[Mysql之存储过程与存储函数](https://juejin.im/post/5cdc20ede51d453ce71f6113)

