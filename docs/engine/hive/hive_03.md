# 第3章 Hive数据类型

## 3.1 基本数据类型

| Hive         | MySQL         | Java    | 长度                                                 | 例子                                |
| ------------ | ------------- | ------- | ---------------------------------------------------- | ----------------------------------- |
| tinyint      | tinyint       | byte    | 1byte有符号整数                                      | 2                                   |
| smallint     | smallint      | short   | 2byte有符号整数                                      | 20                                  |
| {==int==}    | int           | int     | 4byte有符号整数                                      | 2000                                |
| {==bigint==} | bigint        | long    | 8byte有符号整数                                      | 20000000                            |
| boolean      | {==无==}      | boolean | 布尔类型，true或者false                              | true  false                         |
| float        | float         | float   | 单精度浮点数                                         | 3.14159                             |
| {==double==} | double        | double  | 双精度浮点数                                         | 3.14159                             |
| {==string==} | {==varchar==} | string  | 字符系列。可以指定字符集。可以使用单引号或者双引号。 | 'now is the time'"for all good men" |
| timestamp    | timestamp     |         | 时间类型                                             |                                     |
| binary       | binary        |         | 字节数组                                             |                                     |

对于 Hive 的 string 类型相当于数据库的 varchar 类型，该类型是一个可变的字符串，不过它不能声明其中最多能存储多少个字符，理论上它可以存储 2GB 的字符数。

**1）案例实操**

（1）创建一张 stu 表并加入数据

```sql
create table stu2(id int, name string, age int, weight double);
```

```sql
insert into stu values
(1,'xiaohaihai',18,75),(2,'xiaosongsong',16,80),
(3,'xiaohuihui',17,60),(4,'xiaoyangyang',16,65);
```

## 3.2 集合数据类型

| 数据类型 | 描述                                                         | 语法示例                                              |
| -------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| struct   | 结构体由一组称为成员的不同数据组成，其中每个成员可以具有不同的类型。通常用来表示类型不同但是又相关的若干数据。例如，如果某个列的数据类型是struct{first string, last string}，那么{==第 1 个元素可以通过字段 .first 来引用==}。 | struct() 例如{==struct<street:string, city:string>==} |
| map      | map 是一组键-值对元组集合，使用数组表示法可以访问数据。例如，如果某个列的数据类型是 map，其中键->值对是 'first' -> 'John' 和 'last' -> 'Doe'，那么可以{==通过字段名['last']获取最后一个元素==} | map() 例如 {==map<string, int>==}                     |
| array    | 数组是一组具有{==相同类型==}和名称的变量的集合。这些变量称为数组的元素，每个数组元素都有一个编号，编号从零开始。例如，数组值为['John', 'Doe']，那么{==第 2 个元素可以通过数组名 [1] 进行引用==}。 | array() 例如{==array\<string>==}                      |

Hive 有三种复杂数据类型 array、map 和 struct。array 和 map 与 Java 中的 array 和 map 类似，而 struct 由一组称为成员的不同数据组成，其中每个成员可以具有不同的类型。

**1）案例实操**

（1）假设某表有如下一行，我们用 Json 格式来表示其数据结构。在 Hive 下访问的格式为。

```json
{
    "name": "dasongsong",
    "friends": ["bingbing" , "lili"],      //列表array 
    "students": {                              //键值map
        "xiaohaihai": 18,
        "xiaoyangyang": 16
    }
    "address": {                               //结构struct
        "street": "hui long guan", 
        "city": "beijing",
        "email":10010
    }
}
```

（2）基于上述数据结构，我们在 Hive 里创建对应的表，并导入数据。

在 /opt/module/hive/datas 路径下，创建本地测试文件 tch.txt。

```txt title="tch.txt"
songsong,bingbing_lili,xiaohaihai:18_xiaoyangyang:16,hui long guan_beijing_10010
yangyang,caicai_susu,xiaosongsong:17_xiaohuihui:16,chao yang_beijing_10011
```

注意：map，struct 和 array 里的元素间关系都可以用同一个字符表示，这里用 "_"。

（3）Hive 上创建测试表 tch

```sql
create table tch(
    name string,                    --姓名
    friends array<string>,        --朋友
    students map<string, int>,   --学生
    address struct<street:string,city:string,email:int> --地址
)
row format delimited fields terminated by ','
collection items terminated by '_'
map keys terminated by ':'
lines terminated by '\n';
```

字段解释：

- row format delimited fields terminated by ','  -- 列分隔符
- collection items terminated by '_'  	-- map、struct 和 array 的数据分隔符
- map keys terminated by ':'			-- map 中的 key 与 value 的分隔符
- lines terminated by '\n';				-- 行分隔符

（4）导入文本数据到测试表

```shell
hadoop fs -put tch.txt /user/hive/warehouse/tch
```

（5）访问三种集合列里的数据，以下分别是 array，map，struct 的访问方式

```sql
select 
    friends[1],
    students['xiaohaihai'],
    address.city 
from tch
where name="songsong";
```

## 3.3 类型转化

Hive 的原子数据类型是可以进行隐式转换的，类似于 Java 的类型转换，例如某表达式使用 int 类型，tinyint 会自动转换为 int 类型，但是 Hive 不会进行反向转化，例如，某表达式使用 tinyint 类型，int 不会自动转换为 tinyint 类型，它会返回错误，除非使用 cast 操作。

**1）隐式类型转换规则如下**

（1）任何整数类型都可以隐式地转换为一个范围更广的类型，如 tinyint 可以转换成 int，int 可以转换成 bigint。

（2）所有整数类型、float 和 {==string 类型==}都可以隐式地转换成 double。

（3）tinyint、smallint、int 都可以转换为 float。

（4）boolean 类型不可以转换为任何其它的类型。

**2）可以使用 cast 操作显示进行数据类型转换**

语法：`cast(expr as <type>) `
返回值：Expected "=" to follow "type"
说明：返回转换后的数据类型
案例实操：

<div class="termy">
```console
$ select '1' + 2, cast('1' as int) + 2;

_c0	   _c1
3.0	    3
```
</div>

