# Redis RDB 文件格式解析

redis rdb 文件是 redis 在内存中所存储全部数据的二进制表示，结构非常紧凑。在 redis 初始化或者迁移时，加载该文件，可以快速恢复原存储数据到 redis。
对于作者来说了解 redis rdb 文件编码算法，对于学习二进制文件处理、压缩编码算法甚至合并或恢复损坏的 rdb 文件都很有帮助。
本文主要翻译自 redis-rdb-tools 的作者关于 rdb 文件格式的经典剖析，并在此基础上增加部分细节和示例。

## 文件整体结构

rdb 文件整体上可分为三个区域：头信息区、数据区、尾信息区。

### 描述符号约定

<pre><code>
-/./空格/换行/行首数字      # 只为增加展示效果，实际编码中没有
52 45 44 49 53           # 示例数据为16进制
FE/FD/FA...              # 示例数据中的此类十六进制值代表 redis 的特殊操作符，参考「redis 操作符列表」
$xxxx                    # 代表一种数据类型或者编码格式，会在第二节进一步讲解
[]                       # 代表括号所选的范围是可选的，非必须存在
</pre></code>



### 操作符列表
redis 支持的操作符列表
| Byte | Name         | Description                                                  |
| ---- | ------------ | ------------------------------------------------------------ |
| 0xFF | EOF          | rdb 文件结束符                                               |
| 0xFE | SELECTDB     | redis [数据库编号](#数据库编号)                          |
| 0xFD | EXPIRETIME   | redis [过期时间](#1-key-过期时间)，使用秒表示。        |
| 0xFC | EXPIRETIMEMS | redis [过期时间](#1-key-过期时间)，使用毫秒表示。    |
| 0xFB | RESIZEDB     | redis [dbsize](#RESIZEDB)，描述 key 数目和设置了过期时间 key 数目 |
| 0xFA | AUX          | redis [元属性](#AUX metadata)，可以存储任意的的 key-value 对 |

### 头信息区

头信息区用于描述 rdb 文件，如 rdb 版本、redis 版本、rdb 创建时间等信息。

该头信息在 redis 启动加载时会做校验，校验失败则不会加载数据区。

<pre><code>
----------------------------  # rdb 是二进制格式，示例数据增加换行、空格、"-" 只是为解释方，实际编码中没有。示例数据均为16进制
52 45 44 49 53                # ascii值对应 "REDIS" 魔数字符
30 30 30 37                   # rdb 格式版本，示例值对应 ascii 码 0007 ，描述 rdb 格式为第7版
FA $字符串编码的key $字符串编码的value   # AUX 指令，描述 redis 属性，可存在多个 key-value 组合
</pre></code>

### 数据区

rdb 文件实际数据存放区。也是编码最复杂的位置。采用数十种不同编码方法。

<pre><code>
----------------------------                # 开始存储数据
FE 00                                       # FE 指定数据库编号,示例值 "00" 表示接下来是第 0 个库的数据。
FB $整数编码的数1 $整数编码的数2     # RESIZEDB 指令，描述总的 key 个数以及，有过期时间的 key 个数
----------------------------          
1. [FD $无符号整型数 or FC $无符号长整型数]  # 存储 key 的过期时间，以 FD 开始，后续 4byte 为过期时间，单位为秒。以 FC 使用 8 byte表示，单位为毫秒。
2. $值类型编码的整数                              # key 所存储 value 的存储类型
3. $字符串编码的key                      # 编码 key，以字符串方式编码 key
4. $特殊编码的value                           # 编码 value，视数据类型和大小不同，存在多种编码格式
...                                         # 重复1-4，直到当前库的全部数据都存储完毕
----------------------------     
FE $整数编码的数字                         # 指定下一个数据库编号
----------------------------          
</pre></code>


### 尾信息区

尾信息区用来表示 RDB 结束，以及存储 crc 校验码。用于校验整体数据是否正确。

尾信息区一共 9 byte。

<pre><code>
----------------------------     # 数据块都已存储完毕，开始存储末尾信息
FF                               # RDB 文件结束符
23 A5  EC C5 D8 99  90 D8        # 整个 RDB 文件的 CRC 64 校验码，使用 8 byte 表示
</code></pre>

## 文件编码算法
上一节中，介绍了 rdb 文件的整体结构，以及其中的三个编码区域。
接下来，对每个编码区域的细节一一解释。

### 各区域编码方法
#### 头部区域编码
##### 魔数

以 ascii 码 "REDIS" 开头。用来验证当前文件的格式。类似 java 程序编译后的 class 文件，以 "CAFEBABE" 魔数开头，寓意 java 和 cafe 的千丝万缕的关系。

##### rdb version
rdb 文件截止目前，已经有 9 个版本。
高版本 redis 可以  100%  向后兼容加载旧版 rdb 文件。
版本信息使用 4 个字节表示。
如：`30 30 30 37` 转换为 ascii 码 `00 00 00 07`，代表 rdb version 文件为第 7 版。

##### AUX metadata
第 7 版中新增加的操作符，以 0xFA 作为标志。
可以存放多组 key-value，用来表示对应元信息。
每一对 key-value，都以0xFA 开头。key 和 value 均采用 rdb **[字符串编码](#字符串编码)**方法。 
默认元信息列表：

- redis-ver：redis 版本信息。
- redis-bits：输出 rdb 文件机器的位数，64bit 或 32 bit。
- ctime：rdb 文件创建时间。
- used-mem：rdb 加载到内存中的内存使用量。

#### 数据区域编码
##### 数据库编号
以 0xFE 开头。接下来的几个 byte 采用**[整数编码](#整数编码)**算法，用于表示当前数据区的所在的 redis 数据库的编号。默认从0开始，直到16。
##### RESIZEDB
第 7 版中新增加的操作符， 0xFB 作为标志。存放 redis 的 key 数目和设定了过期时间的 key 数目。
该设定用于从 rdb 文件中恢复 redis 时，直接创建对应数量的 hash 槽，减少 rehash 需要的额外时间消耗。
官方测试表明，该优化可提升约 20% 以上的加载速度。

##### key-value 对
数据库编号之后，紧跟着就是该数据库中存放的全部数据。
数据以 key-value 链的形式，一个接着一个存放。
每一个 key-value 由 4 部分组成：

######  1. key 过期时间
   如果有设置过期时间，则会存放具体过期时间的 timestamp。否则这部分不存在。采用小端字节序编码。
   以 0xFD 开头，代表过期时间为秒，之后的 4 byte 表示该key  的过期时间。
   以 0xFC 开头，代表过期时间为毫秒，之后的 8 byte 表示该key  的过期时间。

######  2. value 存储类型
   该部分使用 1 byte 表示。具体的存储类型如下：

   ```bash
   # 0 =  "String Encoding"
   # 1 =  "List Encoding"
   # 2 =  "Set Encoding"
   # 3 =  "Sorted Set Encoding"
   # 4 =  "Hash Encoding"
   # 9 =  "Zipmap Encoding"
   # 10 = "Ziplist Encoding"
   # 11 = "Intset Encoding"
   # 12 = "Sorted Set in Ziplist Encoding"
   # 13 = "Hashmap in Ziplist Encoding" 
   ```

######  3.key
   使用**[字符串编码](#字符串编码)**方法编码 key。 

######  4.value 
   依据 value 存储类型的不同，使用对应的**[值编码](#2. value 存储类型)**方法编码 value。
   如当 value 存储为 0 时，value 使用**[字符串编码](#字符串编码)**方法编码。
   当 value 为 10 时，value 使用**[ziplist 编码](#ziplist 编码)**方法编码。

#### 尾部区域编码
该区域最为简单。固定使用 9 byte。
第一 byte 为 0xFF ，之后固定跟着 8 byte 用于 crc64 校验。
该校验码采用`crc-64-jones`算法生成，用于校验 rdb 文件的合法性。
可以在 redis 配置文件设置 `rdbchecksum no` 关闭校验。之后 dump rdb 文件时将以
`00 00 00 00 00 00 00 00` 结尾。加载 rdb 文件时也会跳过验证 checksum。
### 编码算法细节
至此 rdb 文件各位置的编码方法概要已经介绍完毕。接下来展开解释具体的编码算法。
首先介绍两个 rdb 文件中基础编码算法：整数编码和字符串编码，之后进一步解析稍复杂的值编码算法。
#### 整数编码
该部分为了尽量缩短字节数，采用可变字节编码方法。rdb 文件中频繁使用该算法。
主要用于在二进制文件中存储下一个对象的长度，如在编码一个 key 时，使用该方法在 key 的前几个 byte 存储该 key 占用字节数。

具体算法：
1. 从高位开始，读取第一个 byte 的前 2 bit。
2. 如果高位以 00 开始：当前 byte 剩余 6 bit 表示一个整数。
3. 如果高位以 01 开始：当前 byte 剩余 6 bit，加上接下来的 8 bit 表示一个整数。
4. 如果高位以 10 开始：忽略当前 byte 剩余的 6 bit，接下来的 4 byte 表示一个整数。
1. 如果高位以 11 开始：特殊编码格式，剩余 6 bit 用于表示该格式。

该算法在整数较小时可以缩短编码，如 0-63 只需 1 byte 表示，64-16383 只需 2 byte。

#### 字符串编码
rdb 文件的字符串是二进制安全的。不需要像 c 语言的字符串那样，以 '\0' 为结束符。
rdb 文件的字符串主要有三种编码方法：简单的长度前缀编码字符，使用字符串编码整型以及压缩字符串。
均是以使用**[整数编码](#整数编码)**编码表示字符串长度，之后存储具体字符串编码。
当整数编码为：
- 最高 2 bit 为 00、01、10 时：

   简单字符串方法。
   长度编码后是字符串具体的编码。如字符 'a' 使用 '0x61' 表示。
- 最高 2 bit 为 11 ，剩余 6 bit 为数 0、1、2 时：

   字符串整型编码方法。
   - 为 0 时：之后 8 bit 用于存储该整型。
   - 为 1 时：之后 16 bit 用于存储该整型。
   - 为 2 时：之后 32 bit 用于存储该整型。
- 最高 2 bit 为 11 ，剩余 6 bit 为数 3 时：

   压缩字符串编码方法。
   该类型的解码具体方法：
   1. 使用**[整数编码](#整数编码)**方法读取压缩后字符串长度，如表示为 clen。
   2. 使用**[整数编码](#整数编码)**方法读取未压缩字符串长度，如表示为 len。
   3. 读取 clen 个字节。
   4. 最后使用 lzf 算法这 clen 个字节，解析后还原字符串。
#### hash 编码
当某个 key 存储的 hash 数据的大小超过 hash-max-ziplist-entries 或者 hash-max-ziplist-values 的值时。使用 hash table 编码值为 hash 类型的数据。

编码过程：
1. 使用**[整数编码](#整数编码)**方法 hash key 数，如表示 size。
1. 使用**[字符串编码](#整数编码)**方法读取 2*size 个字符串。该字符串由 size个key-value 对组成。

例如："f1 v1 f2 2" 用来表示 hash 表，{"f1"->"v1"，"f2"->"2"}。最后章节会详细介绍表示该 hash 数据的实际的二进制串。

#### Hashmap in Ziplist 编码
当某个 key 存储的 hash 数据大小都小于 hash-max-ziplist-entries 或者 hash-max-ziplist-values 的值时。hash 表示为连续的 entry 链，并使用 ziplist 编码算法表示 hash 数据。

例如：hash 数据 {"f1"->"v1"，"f2"->"2"} ，使用**[ziplist 编码](#ziplist 编码)**方法编码字符串列表 ["f1","v1","f","22"]。

#### ziplist 编码
ziplist 编码在 redis 各类型的数据，hash、list 等中普遍使用。

ziplist 运行过程可以理解为是将一个字符串 list 序列化，同时为了方便从两端快速检索，增加了额外的 offset 等信息。

ziplist 整体结构：

```
<zlhead><zlbytes><zltail><zllen><entry>...<entry><zlend>
```

1. zlhead

   使用**[字符串编码](#字符串编码)**解码，存储当前 key 所属 value 的 bytes 数目以及是否启用了 lzf 等信息。如 ziplist 以 `1B` 开头，对应 2 进制为 `0b00011011` ，后  6 bit 表示为 十进制 `27` ，表示当前 ziplist 共有 27 bytes。从 <zlbytes> 开始读取直到 <zlend>。

2. zlbytes

   4 byte 无符号整数，采用小端字节序编码。表示当前 ziplist 总占用字节数。

3. zltail

   4 byte 无符号整数，采用小端字节序编码。代表到达最后一个 entry 需要跳过的字节数。

4. zllen

   2 byte 无符号整数，采用小端字节序编码。ziplist entry 数目。当用于存储 hash 数据时，entry 数为 key 数 + value数。

5. entrys

   存储 entry 列表。每个 entry 按如下方法存储。

   entry 结构：

   ```
   <length-prev-entry><special-flag><raw-bytes-of-entry>
   ```

   - <length-prev-entry>

     可变长编码，存储前一个 entry 的占用的字节数。使用**[整数编码](# 整数编码)**编码。

   - <special-flag>

     同样是可变长编码，存储当前 entry 的类型以及长度。

     这里是理解 ziplist 的关键。

     用高位的前若干 bit 分别表示两种类型以及 9 种情况：

     

     | 字节码              | 类型    | 涵义                                                         |
     | ------------------- | ------- | ------------------------------------------------------------ |
     | `00pppppp`          | String  | `00` 紧接的 6 bit 表示字符串长度，代表不超过 64 byte 的字符串 |
     | `01pppppp|qqqqqqqq` | String  | `00` 紧接的14 bit 表示字符串长度，代表不超过 16384 byte 的字符串 |
     | `10______|<4 byte>` | String  | 当前 byte 紧接的 4 bytes 表示字符串长度，代表不小于 16384 byte 的字符串 |
     | `1100____`          | Integer | 当前 byte 紧接的 2 bytes 表示字符串长度，代表一个 16 bit 有符号整型 |
     | `1101____`          | Integer | 当前 byte 紧接的 4 bytes 表示字符串长度，代表一个 32 bit 有符号整型 |
     | `1110____`          | Integer | 当前 byte 紧接的 8 bytes 表示字符串长度，代表一个 64 bit 有符号整型 |
     | `11110000`          | Integer | 当前 byte 紧接的 3 bytes 表示字符串长度，代表一个 24 bit 有符号整型 |
     | `11111110`          | Integer | 当前 byte 紧接的 1 bytes 表示字符串长度，代表一个 8 bit 有符号整型 |
     | `1111[0001-1101]`   | Integer | 特殊编码。`0001-1101`  对应十进制值为 1-13 。实际由于`11110000` 已经被占用，该编码仍需要饱含 `0`。所以需在 `00` 紧接的 4 bit 转换为数值后减掉1。所以该编码可以用来表示 `0-12` 的数值。 |

   - <raw-bytes-of-entry>

        存储实际数据。依据 <special-flag> 指定的类型和长度。采用**[整数编码](# 整数编码)**或**[字符串编码](# 整数编码)**。

     

6. zlend

   固定以 `0xFF` 结尾。

## 示例数据

1. 启动 redis 执行数据插入（注意替换 ip、port）

   ```bash
    echo "hmset h1 f1 v1 f2 100"|redis-cli -h 127.0.0.1 -p 6379 -n 0
    echo "set k1 v1"|redis-cli -h 127.0.0.1 -p 6379 -n 0
    echo "hmset h2 f3 v3"|redis-cli -h 127.0.0.1 -p 6379 -n 1
   ```

1. 查看 dump 文件
   - 找到 redis config
     ```bash
     echo "info server"|redis-cli|grep "config_file"
     ```
   
   - 输出
     ```bash
     config_file:/opt/redis/conf/redis.conf
     ```
   
   - 找到 dump 文件
     ```bash
     cat /opt/redis/conf/redis.conf|grep -E "^dir|^dbfile"
     ```
   
   - 输出
     ```
     dbfilename "dump1.rdb"
     dir /opt/redis/data
     ```
   - 查看 dump（也可以使用 vim + xxd )
    ``` 
    hexedit  /opt/redis/data/dump1.rdb
    ```
   - 输出
   <pre><code>
   00000000   52 45 44 49  53 30 30 30  37 FA 09 72  REDIS0007..r
   0000000C   65 64 69 73  2D 76 65 72  06 33 2E 32  edis-ver.3.2
   00000018   2E 31 33 FA  0A 72 65 64  69 73 2D 62  .13..redis-b
   00000024   69 74 73 C0  40 FA 05 63  74 69 6D 65  its.@..ctime
   00000030   C2 2F C9 BC  5F FA 08 75  73 65 64 2D  ./.._..used-
   0000003C   6D 65 6D C2  E0 6E 09 00  FE 00 FB 02  mem..n......
   00000048   00 0D 02 68  6B 1A 1A 00  00 00 16 00  ...hk.......
   00000054   00 00 04 00  00 02 66 31  04 02 76 31  ......f1..v1
   00000060   04 02 66 32  04 FE 64 FF  00 04 6B 65  ..f2..d...ke
   0000006C   79 31 06 76  61 6C 75 65  31 FE 01 FB  y1.value1...
   00000078   01 00 0D 03  68 6B 32 13  13 00 00 00  ....hk2.....
   00000084   0E 00 00 00  02 00 00 02  66 33 04 02  ........f3..
   00000090   76 33 FF FF  A6 10 58 3C  59 B4 AB 4E  v3....X<Y..N
   </code></pre>
1. 解析文件
  ![](images/redis-rdb-format-example.png?raw=true)
## 参考文章

1. [redis-rdb-tools 作者关于 rdb 文件格式的阐释](https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-RDB-Dump-File-Format)
2. [redis-rdb-tools 作者关于 rdb 文件变更历史的说明](https://github.com/sripathikrishnan/redis-rdb-tools/blob/master/docs/RDB_Version_History.textile)
3. [另一个 rdb 文件格式说明](https://rdb.fnordig.de/file_format.html)
4. [另一个 rdb 文件历史说明](https://rdb.fnordig.de/version_history.html)
5. [ziplist 格式说明](http://zhangtielei.com/posts/blog-redis-ziplist.html)
