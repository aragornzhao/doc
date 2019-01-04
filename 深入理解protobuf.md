# 深入理解protobuf

@(开发)[网络]
>protobuf是google开源的一套数据序列化/反序列化库，其思想类似于xml、json，但是为二进制结构，接口使用方便、性能好、跨平台，广泛应用于数据通信模块中，比如微信后台的svrkit、google的grpc等都是基于protobuf

## 前言
protobuf是客户端和后台开发都很常见的一套开源库，本文通过分析protobuf序列化后的二进制流，更深入的理解protobuf，解答开发中常见的问题。

>在正式开始前，先提几个问题：
1. required和optional的区别，optional字段如果不填会占用空间吗？
2. int32、fixed32、sint32有什么区别？uint32和enum在序列化后是否有区别？
3. 如果有一方把某字段类型从uint32升级成了uint64会怎样？即双方不一致
4. string和bytes有什么区别？
5. 生成的字节流是否有压缩？是否有加密？
6. 不同版本的protobuf生成的字节流是否有差异？
7. 如果这些问题你都很清楚，恭喜你，本文你不用看了:)

##protobuf基本用法
为了照顾不熟的同学，及方便后文叙述，简单介绍下protobuf的基本用法。protobuf支持多种语言，如C++、Java等，下面以C++为例：

```protobuf
test.proto
message Request
{
    optional uint32  cmd     = 1;
    optional string  name    = 2;
}
```
运行： protoc test.proto --cpp_out=.

生成test.pb.h和test.pb.cc后，就可以在代码中使用了

发送方赋值，并序列化：

```cpp
Request Req;
Req.set_cmd(1234);
Req.set_name("hello");

std::string content;
Req.SerializeToString(&content);
```
对方收到后，反序列化：
```cpp
Request Req;
Req.ParseFromString(content);
```
就可以得到每一个字段值了。
要想深入了解其具体是怎么实现的，就需要分析其序列化后的二进制字节流。

##protobuf的字节流
###3.1基本结构
protobuf序列化后的字节流并不复杂，其基本结构就是多个字段顺序拼接而成。
对于每一个字段，包括3个部分，即：id type value

如：

    optional uint32  cmd     = 1;
其id即1，type为0（表示变长整型，见后详述），value即设置的值如1234

整个字节流拼接后就是如下结构（示意）：



注意没有包头、包尾之类，直接就是包体内容，这点不像其它很多协议如tcp、udp等。
实际上，由于id和type这两个数字一般都不大，存储时是合并成一个数字的，低3位是type，其它高位是id，即：
(id << 3) | type，我们暂且称之为tag，那么真实的结构就是这样的：



google设计时，为type只留了3个位，即最多只支持8个type，包括int、string等，完整列表见后。
###3.2 数据类型
上述结构图里的type，最多支持8个，实际目前有效的只有4个，protobuf序列化时把各种类型都归为这4种：变长整数、定长32位整数及浮点数、定长64位整数及浮点数、变长Buffer。
其中最常用的就是变长整数，比如int32、uint32、int64、uint64、enum等，实际都是用变长整数存储的，基本结构里的tag（即id+type）也是用变长整数存储的。

####3.2.1 变长整数
C++里的数字类型都是定长的，比如int32占4字节，int64占8字节，而大多数情况下，我们存储的值可能很小，比如3，5等，如果存储时也占用4或8字节，就太浪费了，所以google采用了一种简单的压缩存储方法，即varint，具体如下：

​ a. 变长存储，可能为1个字节，也可能为2或3个字节，最多为10个字节

​ b. 那怎么知道是否有后续字节呢，通过每个字节的最高位来描述，如果为1，表示还有后续字节，否则就是最后一个字节

​ c. 用小端字节序

让我们来看一个具体的例子

十进制365编码后为ED 02，具体过程：

    365的二进制为： 0000 0001  0110 1101
    用7位表示为：    000 0010   110 1101
    调整字节序：     110 1101   000 0010
    加高位标志：    1110 1101  0000 0010
    即结果为：      ED         02
再来看一个解码的例子，十进制131415，序列化后为 D7 82 08，解码过程如下：


####3.2.2 其它类型简介
定长类型比较简单，直接存储即可，小端字节序。
说说变长Buffer，实际用在多个地方，典型的是string，另外复合类型、packed的repeated类型，也都是用变长Buffer存储，其值具体包括2部分：长度+内容，长度也是用前面介绍的varint存储的。

###3.3 完整的例子
基本概念终于介绍完了，下面我们来看一个完整的pb序列化后的例子：

message Request
{
    optional uint32  cmd     = 1;
    optional string  name    = 2;
}
设置cmd值为365，name值为”hello”，序列化后的结果为：

​ 08 ED 02 12 05 68 65 6C 6C 6F

解释如下：

08：      //  即 (1<<3)|0，id为1，type为0（变长整型）
ED 02：   // 上面解析过，即365
12：      // 即(2<<3)|2，id为2，type为2（buffer类型，即string）
05 68 65 6C 6C 6F： // 05为长度，后续即为”hello”
###3.4 一点进阶
protobuf的基本结构前面已经介绍的差不多了，看到这里如果你还有兴趣，就继续，不想深入的话本节就可以跳过了。

type类型定义的完整表格：

类型	含义	具体类型
0	变长整数	int32, int64, uint32, uint64, sint32, sint64, bool, enum
1	定长64位	fixed64, sfixed64, double
2	变长Buffer	string, bytes, embedded messages, packed repeated fields
3	已废弃	
4	已废弃	
5	定长32位	fixed32, sfixed32, float
fixed32和fixed64：

和uint32、uint64类似，但是为定长类型。为什么要设计这2个类型呢？

变长类型对于数值比较小时，是比较省空间的，当数值很大时，就反而比定长更占空间了，因为变长整型只有7个有效位。如果业务上某字段值有效位基本都在4字节，如0xF4567890，变长存储需要5字节，而定长存储只需要4字节。

sint32和sint64：

可能很多人没用过这2个类型，它们分别和int32、int64类似，是变长存储的，为什么要设计这2个新的类型呢？

回顾一下上面讲的变长整型，当值很小，比如1、2时，只占1个字节，但如果是负数呢，比如-1，64位表示是0xFFFFFFFFFFFFFFFF，就会占10个字节（64/7=9.14），如果业务上频繁出现负数时，用int32、int64存储就不够节省了。

所以google设计了一种算法，把小的负数映射成小的正数，再用varint编码，就解决了这个问题。具体映射算法为：

Zigzag(n) = (n << 1) ^ (n >> 31),  n为sint32时
Zigzag(n) = (n << 1) ^ (n >> 63),  n为sint64时
映射前后的直观的例子：

映射前	映射后
0	0
-1	1
1	2
-2	3
…	…
2147483647	4294967294
-2147483648	4294967295
复合类型

即type是另一个message类型，比如

message BaseReq
{
    optional uint32  cmd     = 1;
    optional string  name    = 2;
}
message Request
{
    optional BaseReq base    = 1;
    optional uint32  roomid  = 2;
}
对于Request里的base字段，存储方式和string类似，即id为1，type为2（表示变长buffer），value分为长度和内容，其中内容为BaseReq序列化后的字节流。

repeated类型：

message Request
{
    repeated uint32  data = 1;
}
代码中对data添加3个元素，分别为1、2、3，序列化后为：

​ 08 01 08 02 08 03

解释如下：

08:  // (1<<3)|0，即id=1，type=0
01:  // value=1
08:  // (1<<3)|0，即id=1，type=0
02:  // value=2
08:  // (1<<3)|0，即id=1，type=0
03:  // value=3
可以看出，对于repeated类型，只是简单重复，把id和type也重复了很多次。

对于repeated的简单类型可以有另外一种写法，即加[packed=true]，如：

message Request
{
    repeated uint32  data = 1[packed=true];
}
同样对data添加3个元素，分别为1、2、3，序列化后为：

​ 0A 03 01 02 03

解释为：

    0A：          // (1<<3)|2，即id=1，type=2(变长buffer)
    03 01 02 03： // 03为长度，后续3个字节为内容
可以看出，加了[packed=true]后，会对value做集中存储了。
protobuf从2.1版开始支持这个特性，从3.0版开始默认使用这个特性。

## 总结和常见问题
总结一下，从上述结构看：

protobuf序列化后的字节流还是比较简单、紧凑和高效的，兼顾了空间和性能的平衡。序列化后的字节流一定程度上可以自我描述，但不完全，因为序列化后只有4种类型，需要配合proto文件才能完全解析。

最后解答一下文初提出的几个问题

optional字段如果不填会占用空间吗？

答：不会。

int32、fixed32、sint32有什么区别？uint32和enum在序列化后是否有什么区别？

答：int32和sint32类似，都是变长存储，适合大多数情况下数值较小的场景，sint32更适合用在负数较多的场景。fixed32是定长存储，适合数值较大的场景（有效位在4字节）。所谓适合，是指序列化后更省空间。

enum和uint32在序列化后没有区别，反序列化解析时，对于enum会判断有效性，若无效（超出定义范围），则忽略。

如果有一方把某字段类型从uint32升级成了uint64会怎样？即双方不一致

答：uint32和uint64在序列化后是同一种类型，即varint（变长整数）。如果是接收方升级成uint64，正常工作；如果发送方升级成uint64，接收方对于超出uint32的值会截断，只保留低32位。

string和bytes有什么区别？

答：序列化后没有区别；代码实现看，调试版string比bytes多了utf-8的检查，非调试版没有区别。

生成的字节流是否有压缩？是否有加密？

答：对于整数类型，有简单压缩，即varint编码。没有加密，纯明文。

不同版本的protobuf生成的字节流是否有差异？

答：没有差异。也没有包头字段表示字节流是哪个版本的protobuf生成的