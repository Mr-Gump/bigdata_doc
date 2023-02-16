# 第11章 文件格式和压缩

## 11.1 Hadoop压缩概述

| 压缩格式 | 算法    | 文件扩展名 | 是否可切分 |
| -------- | ------- | ---------- | ---------- |
| DEFLATE  | DEFLATE | .deflate   | 否         |
| Gzip     | DEFLATE | .gz        | 否         |
| bzip2    | bzip2   | .bz2       | 是         |
| LZO      | LZO     | .lzo       | 是         |
| Snappy   | Snappy  | .snappy    | 否         |

为了支持多种压缩/解压缩算法，Hadoop 引入了编码/解码器，如下表所示：

Hadoop 查看支持压缩的方式 hadoop checknative。

Hadoop 在 driver 端设置压缩。

| 压缩格式 | 对应的编码/解码器                          |
| -------- | ------------------------------------------ |
| DEFLATE  | org.apache.hadoop.io.compress.DefaultCodec |
| gzip     | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2    | org.apache.hadoop.io.compress.BZip2Codec   |
| LZO      | com.hadoop.compression.lzo.LzopCodec       |
| Snappy   | org.apache.hadoop.io.compress.SnappyCodec  |

压缩性能的比较：

| 压缩算法 | 原始文件大小 | 压缩文件大小 | 压缩速度 | 解压速度 |
| -------- | ------------ | ------------ | -------- | -------- |
| gzip     | 8.3GB        | 1.8GB        | 17.5MB/s | 58MB/s   |
| bzip2    | 8.3GB        | 1.1GB        | 2.4MB/s  | 9.5MB/s  |
| LZO      | 8.3GB        | 2.9GB        | 49.3MB/s | 74.6MB/s |

`On a single core of a Core i7 processor in 64-bit mode, Snappy compresses at about 250 MB/sec or more and decompresses at about 500 MB/sec or more.`

## 11.2 Hive文件格式

为 Hive 表中的数据选择一个合适的文件格式，对提高查询性能的提高是十分有益的。Hive 表数据的存储格式，可以选择 text file、orc、parquet、sequence file 等。

### 11.2.1 Text File

文本文件是 Hive 默认使用的文件格式，文本文件中的一行内容，就对应 Hive 表中的一行记录。

可通过以下建表语句指定文件格式为文本文件:

```sql
create table textfile_table
(column_specs)
stored as textfile;
```

### 11.2.2 ORC

1）文件格式

ORC（Optimized Row Columnar）file format 是 Hive 0.11 版里引入的一种列式存储的文件格式。ORC 文件能够提高 Hive 读写数据和处理数据的性能。

与列式存储相对的是行式存储，下图是两者的对比：

![image-20230216103951434](https://cos.gump.cloud/uPic/image-20230216103951434.png)

如图所示左边为逻辑表，右边第一个为行式存储，第二个为列式存储。

**（1）行存储的特点**

查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列的值，行存储只需要找到其中一个值，其余的值都在相邻地方，所以此时行存储查询的速度更快。

**（2）列存储的特点**

因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法。

前文提到的 text file 和 sequence file 都是基于行存储的，orc 和 parquet 是基于列式存储的。

orc 文件的具体结构如下图所示：

![image-20230216104043563](https://cos.gump.cloud/uPic/image-20230216104043563.png)

每个 ORC 文件由 Header、Body 和 Tail 三部分组成。

其中 Header 内容为 ORC，用于表示文件类型。

Body 由 1 个或多个 stripe 组成，每个 stripe 一般为 HDFS 的块大小，每一个 stripe 包含多条记录，这些记录按照列进行独立存储，每个 stripe 里有三部分组成，分别是 Index Data，Row Data，Stripe Footer。

{==Index Data==}：一个轻量级的 index，默认是为各列每隔 1W 行做一个索引。每个索引会记录第 n 万行的位置，和最近一万行的最大值和最小值等信息。

{==Row Data==}：存的是具体的数据，按列进行存储，并对每个列进行编码，分成多个 Stream 来存储。

{==Stripe Footer==}：存放的是各个 Stream 的位置以及各 column 的编码信息。

Tail 由 File Footer 和 PostScript 组成。File Footer 中保存了各 Stripe 的真实位置、索引长度、数据长度等信息，各 Column 的统计信息等；PostScript 记录了整个文件的压缩类型以及 File Footer 的长度信息等。

在读取 ORC 文件时，会{==先从最后一个字节==}读取 PostScript 长度，进而读取到 PostScript，从里面解析到 File Footer 长度，进而读取 FileFooter，从中解析到各个 Stripe 信息，再读各个 Stripe，即从后往前读。

3）建表语句

```sql
create table orc_table
(column_specs)
stored as orc
tblproperties (property_name=property_value, ...);
```

ORC 文件格式支持的参数如下：

| 参数                 | 默认值     | 说明                                    |
| -------------------- | ---------- | --------------------------------------- |
| orc.compress         | ZLIB       | 压缩格式，可选项：NONE、ZLIB,、SNAPPY   |
| orc.compress.size    | 262,144    | 每个压缩块的大小（ORC文件是分块压缩的） |
| orc.stripe.size      | 67,108,864 | 每个stripe的大小                        |
| orc.row.index.stride | 10,000     | 索引步长（每隔多少行数据建一条索引）    |

### 11.1.3 Parquet

Parquet 文件是 Hadoop 生态中的一个通用的文件格式，它也是一个列式存储的文件格式。

Parquet 文件的格式如下图所示：

![image-20230216104446169](https://cos.gump.cloud/uPic/image-20230216104446169.png)

上图展示了一个 Parquet 文件的基本结构，文件的首尾都是该文件的 Magic Code，用于校验它是否是一个 Parquet 文件。

首尾中间由若干个 Row Group 和一个 Footer（File Meta Data）组成。

每个 Row Group 包含多个 Column Chunk，每个 Column Chunk 包含多个 Page。以下是 Row Group、Column Chunk 和 Page 三个概念的说明：

{==行组（Row Group）==}：一个行组对应逻辑表中的若干行。 

{==列块（Column Chunk）==}：一个行组中的一列保存在一个列块中。 

{==页（Page）==}：一个列块的数据会划分为若干个页。

Footer（File Meta Data）中存储了每个行组（Row Group）中的每个列快（Column Chunk）的元数据信息，元数据信息包含了该列的数据类型、该列的编码方式、该类的 Data Page 位置等信息。

3）建表语句

```sql
Create table parquet_table
(column_specs)
stored as parquet
tblproperties (property_name=property_value, ...);
```

支持的参数如下：

| 参数                | 默认值       | 说明                                                         |
| ------------------- | ------------ | ------------------------------------------------------------ |
| parquet.compression | uncompressed | 压缩格式，可选项：uncompressed，snappy，gzip，lzo，brotli，lz4 |
| parquet.block.size  | 134217728    | 行组大小，通常与HDFS块大小保持一致                           |
| parquet.page.size   | 1048576      | 页大小                                                       |

## 11.3 压缩

在 Hive 表中和计算过程中，保持数据的压缩，对磁盘空间的有效利用和提高查询性能都是十分有益的。

### 11.2.1 Hive表数据进行压缩

在 Hive 中，不同文件类型的表，声明数据压缩的方式是不同的。

**1）TextFile**

若一张表的文件类型为 TextFile，若需要对该表中的数据进行压缩，多数情况下，无需在建表语句做出声明。直接将压缩后的文件导入到该表即可，Hive 在查询表中数据时，可自动识别其压缩格式，进行解压。

需要注意的是，在执行往表中导入数据的 SQL 语句时，用户需设置以下参数，来保证写入表中的数据是被压缩的。

```sql
--SQL语句的最终输出结果是否压缩
set hive.exec.compress.output=true;
--输出结果的压缩格式（以下示例为snappy）
set mapreduce.output.fileoutputformat.compress.codec =org.apache.hadoop.io.compress.SnappyCodec;
```

**2）ORC**

若一张表的文件类型为 ORC，若需要对该表数据进行压缩，需在建表语句中声明压缩格式如下：

```sql
create table orc_table
(column_specs)
stored as orc
tblproperties ("orc.compress"="snappy");
```

**3）Parquet**

若一张表的文件类型为 Parquet，若需要对该表数据进行压缩，需在建表语句中声明压缩格式如下：

```sql
create table orc_table
(column_specs)
stored as parquet
tblproperties ("parquet.compression"="snappy");
```

### 11.2.2 计算过程中使用压缩

**1）单个 MR 的中间结果进行压缩**

单个 MR 的中间结果是指 Mapper 输出的数据，对其进行压缩可降低 shuffle 阶段的网络 IO，可通过以下参数进行配置：

```sql
--开启MapReduce中间数据压缩功能
set mapreduce.map.output.compress=true;
--设置MapReduce中间数据数据的压缩方式（以下示例为snappy）
set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec;
```

2）单条 SQL 语句的中间结果进行压缩

单条 SQL 语句的中间结果是指，两个 MR（一条 SQL 语句可能需要通过MR进行计算）之间的临时数据，可通过以下参数进行配置：

```sql
--是否对两个MR之间的临时数据进行压缩
set hive.exec.compress.intermediate=true;
--压缩格式（以下示例为snappy）
set hive.intermediate.compression.codec= org.apache.hadoop.io.compress.SnappyCodec;
```

