

# case表达式

##### 在SQL里表达条件分支：

case表达式3是SQL例非常重要而且使用起非常便利的技术，应该学会使用它来描述条件分支。本节主要介绍行列转换、已有数据重分组（分类）、与约束的结合使用、针对聚合结果的条件分支等使用方法。

## case表达式概述

case表达式有简单case表达式（simple case expression）和搜索case表达式（searched case expression）两种写法，分别如下所示：

```plsql
--简单CASE表达式
CASE sex
  WHEN '1' THEN '男'
  WHEN'2' THEN '女'
ELSE '其他' END

--搜索CASE表达式
CASE WHEN sex = '1' THEN '男'
	 WHEN sex = '2' THEN '女'
ELSE '其他' END
```

两种写法的执行结果是相同的，但是简单case表达式实现的事情是有限的，所以一般采用搜索case表达式的写法。

在编写SQL语句时需要注意，发现真的 **WHEN** 子句时，case表达式的真假判断就会终止，而剩下的 **WHEN** 子句会被忽略。所以为了避免引起不必要的混乱，使用 **WHEN** 子句时要注意条件的排他性。

+ 注意事项：

1. 统一各分支返回的数据类型，不能某个分支返回字符类型，而其他分支返回数值类型；
2. 不要忘了写：END；
3. 养成写ELSE子句的习惯。



## 将已有编号方式转化为新的方式并统计

在进行非定制统计时，经常会遇到将已有编号方式转换为另外一种便于分析的方式进行统计的需求。

例如，现在有一张按照“‘1：北海道’、‘2：青森’、······、‘47：冲绳’”这种编号方式来统计都道县府人口的表，需要以东北、关东、九州等地区为单位来分组，并统计人口数量。具体见下图：

<img src="C:\Users\lenovo\Desktop\Markdown\image\SQL进阶教程\1-01.PNG" alt="1-01" style="zoom: 67%;" />

+ 实现方法一：

```plsql
--把县编号转换为地区编号
SELECT CASE pref_name
			WHEN '德岛' THEN '四国'
			WHEN '香川' THEN '四国'
			WHEN '爱媛' THEN '四国'
			WHEN '高知' THEN '四国'
			WHEN '福冈' THEN '九州'
			WHEN '佐贺' THEN '九州'
			WHEN '长崎' THEN '九州'
		ELSE '其他' END AS district,
		SUM(population)
FROM PopTbl
GROUP BY CASE pref_name
			WHEN '德岛' THEN '四国'
			WHEN '香川' THEN '四国'
			WHEN '爱媛' THEN '四国'
			WHEN '高知' THEN '四国'
			WHEN '福冈' THEN '九州'
			WHEN '佐贺' THEN '九州'
			WHEN '长崎' THEN '九州'
		ELSE '其他' END AS district;
```

+ 实现方法二（进阶）：

  将数值按照适当的等级进行分类统计。比如，按人口数量等级查询都道县个数：

```plsql
--按人口数量等级划分都道县
SELECT CASE WHEN population < 100 THEN '01'
			WHEN population >= 100 AND population < 200 THEN '02'
			WHEN population >= 200 AND population < 300 THEN '03'
			WHEN population >= 300 THEN '04'
	   ELSE NULL END AS pop_class,
       COUNT(*) AS cnt
FROM PopTbl
GROUP BY CASE WHEN population < 100 THEN '01'
			WHEN population >= 100 AND population < 200 THEN '02'
			WHEN population >= 200 AND population < 300 THEN '03'
			WHEN population >= 300 THEN '04'
	   ELSE NULL END;
```

+ 实现方法三（仅在PostgreSQL和MySQL中可以实现）

  方法一必须在 **SELECT** 和 **GROUP BY** 子句这两处写一样的case表达式，后期修改比较麻烦，方法三就方便多了：

```plsql
--把县编号转换成地区编号：将case表达式归纳到一处
SELECT CASE pref_name
			WHEN '德岛' THEN '四国'
			WHEN '香川' THEN '四国'
			WHEN '爱媛' THEN '四国'
			WHEN '高知' THEN '四国'
			WHEN '福冈' THEN '九州'
			WHEN '佐贺' THEN '九州'
			WHEN '长崎' THEN '九州'
		ELSE '其他' END AS district,
		SUM(population)
FROM PopTbl
GROUP BY district;
```



## 用一条SQL语句并行不同条件的统计

进行不同条件的统计是case表达式的常用方法。例如，需要往存储各县人口数量的表PopTbl里添加上“性别”列，然后按性别、县名汇总的人数。具体见下图：

<img src="C:\Users\lenovo\Desktop\Markdown\image\SQL进阶教程\1-02.PNG" style="zoom: 50%;" />

+ 实现方法：

```plsql
SELECT pref_name,
	   --男性人口
	   SUM(CASE WHEN sex = '1' THEN population ELSE 0 END) AS cnt_m,
	   --女性人口
	   SUM(CASE WHEN sex = '2' THEN population ELSE 0 END) AS cnt_f
  FROM PopTbl
 GROUP BY pref_name；
```

上面这段代码实现了分别计算每个县的“男性”（即‘1’）人数和“女性”（即‘2’）人数。也就是说将“行结构”的数据转化成了“列结构”的数据。除了SUM，COUNT，AVG等聚合函数也都可以用于将行结构的数据转化成列结构的数据。



## 用CHECK约束定义多个列的条件关系

check约束和case表达式可以一起使用。假设某公司规定“女性员工的工资必须在20万日元以下”，二在这个公司的人事表中，这条无理的规定是使用CHECK约束来描述的，代码如下：

```plsql
--语法：CONSTRAINT <列命> CHECK （条件表达式）
CONSTRAINT check_salary CHECK
			(CASE WHEN sex = '2'
            	  THEN CASE WHEN salary <= 200000
            				THEN 1 ELSE 0 END
            ELSE 1 END = 1)
```

在这段代码里，CASE表达式被嵌入大CHECK约束里，描述了“如果是女性员工，则工资是20万日元以下”这个命题。在命题逻辑中，该命题是叫蕴含式（conditional）的逻辑表达式，记作$P\to Q$。

蕴含式与逻辑与（logical product）的区别。逻辑与也是一个逻辑表达式，意思是“P且Q”，记作$P\and Q$。用逻辑与改写的CHECK约束不能实现“规定”。因为如果在CHECK约束里使用逻辑与，该公司将不能雇佣男性员工。而是用蕴含式，男性也可以在这里工作。下表是关于逻辑与和蕴含式的真值表，其中U是SQL中的三值逻辑的特有值unknown的缩写：

<img src="C:\Users\lenovo\Desktop\Markdown\image\SQL进阶教程\1-03.PNG" style="zoom: 50%;" />

## 在UPDATE语句里进行条件分支

考虑一下这样一中需求：以某数值型的列当前值为判断对象，将其更新成别的值。这里的问题是，词是UPDATE操作的条件会有多个分支。例如，通过通过下面这样一张公司人事部的员工工资信息表Salaries来看一下这种情况。

<img src="C:\Users\lenovo\Desktop\Markdown\image\SQL进阶教程\1-04.PNG" style="zoom: 67%;" />

假设现在需要根据一下条件对该表的数据进行更新。

1. 对当前工资为30万日元以上的员工，降薪10%
2. 对当前工资为25万日元以上且不满28万日元的员工，加薪20%。

按照这些要求更新完的数据应该如下表所示：

<img src="C:\Users\lenovo\Desktop\Markdown\image\SQL进阶教程\1-05.PNG" style="zoom: 67%;" />

+ 实现方法：

```plsql
--用case表达式的更新操作
UPDATE Salaries
	SET salary = CASE WHEN salary >= 300000
    				  THEN salary * 0.9
    				  WHEN salary >= 250000 AND salary < 280000
    				  THEN salary * 1.2
    				  ELSE salary END;
```

用这种方法还可以实现主键值调换：

```plsql
--用CASE表达式调换主键值
UPDATE SomeTable
	SET p_key = CASE WHEN p_key = 'a'
					 THEN 'b'
					 WHEN p_key = 'b'
					 THEN 'a'
					 ELSE p_key END
  WHERE p_key IN ('a','b');
```



## 表之间的数据匹配

在case表达式里，可以使用BETWEEN、LIKE和<、> 等便利的谓词组合，以及能嵌套子查询的IN和EXISTS谓词。如下所示，假设有一张资格培训学校的课程一览表和一张管理每个月所设课程的表。

<img src="C:\Users\lenovo\Desktop\Markdown\image\SQL进阶教程\1-06.PNG" style="zoom: 67%;" />

用这两张表来生成下面这样的交叉表，以便一目了然地知道每个月开设的课程。

<img src="C:\Users\lenovo\Desktop\Markdown\image\SQL进阶教程\1-07.PNG" style="zoom: 67%;" />

+ 实现方法：

```plsql
--表的匹配：使用IN谓词
SELECT course_name,
	   CASE WHEN course_id IN (SELECT course_id FROM OpenCourse 
                                 WHERE month = 200706) THEN 'O'
            ELSE 'X' END AS "6月",
       CASE WHEN course_id IN (SELECT course_id FROM OpenCourse 
                                 WHERE month = 200707) THEN 'O'
            ELSE 'X' END AS "7月",
       CASE WHEN course_id IN (SELECT course_id FROM OpenCourse 
                                 WHERE month = 200708) THEN 'O'
            ELSE 'X' END AS "8月"
  FROM CourseMaster;
  
  --表的匹配：使用EXISTS谓词
  SELECT CM.course_name,
	   CASE WHEN course_id IN (SELECT course_id FROM OpenCourse OC
                                 WHERE month = 200706 AND OC.course_id = CM.course_id) THEN 'O'
            ELSE 'X' END AS "6月",
       CASE WHEN course_id IN (SELECT course_id FROM OpenCourse 
                                 WHERE month = 200707 AND OC.course_id = CM.course_id) THEN 'O'
            ELSE 'X' END AS "7月",
       CASE WHEN course_id IN (SELECT course_id FROM OpenCourse 
                                 WHERE month = 200708 AND OC.course_id = CM.course_id) THEN 'O'
            ELSE 'X' END AS "8月"
  FROM CourseMaster CM;
```

这样的查询没有进行聚合，因此不需要排序，月份增加的时候仅需要修改SELECT子句就可以了，扩展性比较好。使用IN或EXISTS得到的查询结果是一样的，但是从性能上看，EXISTS更好。



## 在CASE表达式中使用聚合函数

假设有一张显示了学生及其加入的社团的一览表。如表SrudentClub所示，这张表的主键是“学号、社团ID”，存储了学生和社团之间多对多的关系。有的同学之加入了一个社团，而有的同学加入了多个社团。对与加入多个社团的学生，通过将其“主社团标志”列设置为Y或N来表明哪一个社团是他的主社团：对于只加入了一个社团的学生，将其“主社团标志”列设置为N。现在想要知道每个学生的主社团ID。

+ 实现代码：

```plsql
--查询学生的主社团
SELECT std_id,
	   CASE WHEN COUNT(*) = 1 --只加入了一个社团
	   		THEN MAX(club_id)
	   		ELSE MAX(CASE WHEN main_club_flag = 'Y'
                    	  THEN club_id 
                    	  ELSE NULL END)
       END AS main_club
  FROM StudentClub
 GROUP BY std_id;
```

这条SQL语句在CASE表达式里使用了聚合函数，又在聚合函数里使用了CASE表达式。所以CASE表达式用在SELECT子句里时，既可以写在聚合函数内部，也可以写在聚合函数外部。

## 小结

作为表达式，CASE表达式在执行时会被判定为一个固定值，因此它可以写在聚合函数内部；也正因为它是表达式，所以还可以写在SELECT子句、GROUP BY子句、WHERE子句、ORDER BY子句里。简单点说，在能写列名和常量的地方，通常都可以写CASE表达式。