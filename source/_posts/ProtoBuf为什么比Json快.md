---
title: ProtoBuf为什么比Json快
date: 2021-12-01 22:52:45
tags: Protobuf
---
> Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. 
<!--more-->
Protocol buffers是Google为了序列结构化数据开发的语言无关、平台无关、可拓展的机制。类似XML，但是更小，更快，更简单。
> JSON (JavaScript Object Notation) is a lightweight data-interchange format. It is easy for humans to read and write.

JSON是一个易于读写的轻量数据交换格式。

## 从一个简单的例子开始
一份Json数据以及对应的二进制数据
```
{
    "name":"Mike",
    "age":29,
    "sex":true,
    "phone":"A123456"
}

bytes size: 60
7b 22 6e 61 6d 65 22 3a
20 22 4d 69 6b 65 22 2c
20 22 61 67 65 22 3a 20
32 39 2c 20 22 73 65 78
22 3a 20 74 72 75 65 2c
20 22 70 68 6f 6e 65 22
3a 20 22 41 31 32 33 34
35 36 22 7d
```
使用类似的.proto定义生成对应的二进制数据
```
//user.proto
syntax = "proto3";
message UserInfo {
  string name = 1;
  int32 age = 2;
  bool sex = 3;
  string phone = 4;
}

//python code snippet
def print_protobuf():
    user = user_pb2.UserInfo()
    user.name = "Mike"
    user.age = 29
    user.sex = True
    user.phone = "A123456"
    print_hex(user.SerializeToString())

bytes size: 19
0a 04 4d 69 6b 65 10 1d
18 01 22 07 41 31 32 33
34 35 36
```

可以看到，在这个例子中，为了传输同样的数据，使用Protobuf生成的数据量是使用Json时的三分之一。Json对应的二进制数据非常简单，每个字符由ASCII码表示，总计60个字符，每个字符一个字节，二进制数据总计60字节。而由protobuf生成的二进制数据就比较难懂了，为什么只用19个字节便可以表示相同的数据了呢？

## Protobuf 编码方式

内容参考Google的[官方文档](https://developers.google.com/protocol-buffers/docs/encoding)

### Varint
一个unsigned int32类型的变量占用4个字节，可以表示的数字范围是 0 到 2^32 (4,294,967,295)。在实际的场景中，小数字的使用频率是远大于大数字的使用频率的。这样，在存储中就会浪费掉一部分空间。 例如数字255的二进制小端表示中，后三个字节被浪费。
```
01111111 00000000 00000000 00000000
         --------------------------
```
针对这种情况，提出了一种优化的方案 即varints, varints 是一种用一个或多个字节序列化整型数的方法。可以根据存储数据的大小动态的决定一个整型数占用的空间。 <br/>
在varint的每一个字节中的最高位被设置为msg(most significant bit)，当msb为1表示下一字节还是属于当前数据，为0则表示当前字节为此数据的最后一个字节。每个字节的低7位用7位一组存储数字的二进制补码， 最低有效组在前，即数据字节为小端序。一些例子
```
1 -> 0 0000001

255 -> 0 1111111

256 -> 1 0000000  0 0000001 -> 0000001 0000000 -> 256

300 -> 1 0101100  0 0000010 -> 0000010 0101100 -> 256 + 32 + 8 + 4 = 300
```
这样，当使用一个varint来表示unsigned int32类型整形数时，所占字节数会根据数字大小动态的改变。
```
0 <= x < (2^8 - 1)       1 bit
2^8 <= x < (2^15 - 1)    2 bit
2^15 <= x < (2^22 - 1)   3 bit
2^22 <= x < (2^29 - 1)   4 bit
2^29 <= x <= (2^32)      5 bit
```
### 消息结构
一条protocol buffer消息是一系列的键值对。与Json格式数据的区别是，二进制版protocol buffer数据中使用field's number来作为key值，而最初定义的key值如需要在解码完成后通过定义文件来确定。<br/>
当一条消息被编码时，key值和value被整合进字节流中。在protobuf中，key值是一个varint数，其中包含了两部分内容，第一部分是.proto文件中定义的每个key值对应的field number，第二部分是类型信息，用来确定value值的长度。可用的类型有以下几种
![](wire_types.png)
key值的组织方式为:
```
(field_number << 3) | wire_type

//即最后3bit数代表类型 例如上述例子中protobuf二进制数据的第一个字节0a代表的含义:
0   0001              010
-   ----              ---
msb fieldnumber = 1   type = 2
```
### Strings
当一个k-v对的类型字段为2，即(length-delimited)时，在key值之后会紧跟一个varint值，来表示value的长度。

## 解析数据
现在我们回头来看例子中产生的二进制数据
```
0a 04 4d 69 6b 65
0a -> 0 0001 010 -> field = 1, type = 2
04 -> value length = 4
4d -> M
69 -> i
6b -> k
65 -> e

10 1d 
0 0010 000 -> field = 2, type = 0
00011101  -> 29

18 01
0 0011 000 -> field = 3, type = 0
01 -> true

22 07 41 31 32 33 34 35 36
0 0100 010 -> field = 4, type = 2
07 -> value length = 7
41 31 32 33 34 35 36 -> "A123456"
```
到此，便可以理解这个简单例子中protobuf的二进制数据是如何组织的了。对于题目提出的问题，也有了一个初步的回答。
- 使用varint类型，在使用较小数字更频繁的场景中，更加节省空间。Json中数字按字符串存储，有多少位数就要占用多少bit空间，如例子中 29 被存储为 32 39。示例中节省1字节。
- bool类型值占用1字节。示例中节省3字节。
- 不需要存储类似 {、"、 }、 space 等冗余的字符。例子中Json二进制数据总计有28个冗余字符。
- key值为一个varint类型数，通常只占1个字节。例子中Protobuf二进制数据 key值总计6字节，json中15字节。
- 60(json bytes size) - 19(protobuf bytes size) = 1 + 3 + 28 + (15 - 6)

## 初探protobuf源码
如果有用过protobuf的话，一定对protoc不陌生。他是protocol编译器，用来编译含有消息定义的.proto文件。这个编译器提供一个很有意思的功能
```
$ protoc | grep decode_raw -A4
  --decode_raw                Read an arbitrary protocol message from
                              standard input and write the raw tag/value
                              pairs in text format to standard output.  No
                              PROTO_FILES should be given when using this
                              flag.
```
即在不需要.proto定义文件的情况下，从标准输入读取任意的protocol消息，然后把他们以 tag/value 的文字格式打印到标准输出。后续内容都围绕这个功能来展开。<br/>
先拿个稍微复杂一点的例子尝试一下
```
// .proto定义文件
$ cat user.proto                      
syntax = "proto3";                    
                                      
message UserInfo {                    
  string name = 1;                    
  int32 age = 2;                      
  bool sex = 3;                       
  string phone = 4;                   
}                                     
                                      
message AddressBook {                 
  string email = 1;                   
  string phone = 2;                   
  string twitter = 3;                 
}                                     
message Location {                    
  string state = 1;                   
  int32 longitude = 2;                
  int32 latitude = 3;                 
  AddressBook contact = 4;            
}                                     
                                      
message Company {                     
  string name = 1;                    
  repeated UserInfo legal_person = 2; 
  fixed32 tel = 3;                    
  fixed64 fund = 4;                   
  Location location = 5;              
  bytes checksum = 6;                 
  repeated int32 int_array = 7;       
}
```
```
// python 生成数据的代码段
user = user_pb2.UserInfo()
user.name = "Mike"
user.age = 29
user.sex = True
user.phone = "A123456"
user1 = user_pb2.UserInfo()
user1.name = "Amy"
user1.age = 25
user1.sex = False
user1.phone = "A654321"
addressbook = user_pb2.AddressBook()
addressbook.email = "haha@qq.com"
addressbook.phone = "A123456"
addressbook.twitter = "dalala"
location = user_pb2.Location()
location.state = "China"
location.longitude = 123
location.latitude = 456
location.contact.CopyFrom(addressbook)
company = user_pb2.Company()
company.name = "Baidu"
company.tel = 123456789
company.legal_person.extend([user, user1])
company.fund = 100000000000000
company.location.CopyFrom(location)
company.checksum = bytes([0xff, 0xf2, 0x12, 0xf4, 0x34])
company.int_array.extend([1, 2, 3, 4, 5, 6])
print("protobuf:", company.SerializeToString())
file = open("protobuf.bin", "wb")
file.write(bytearray(company.SerializeToString()))
file.close()
```
```
// 试一下~
$ protoc --decode_raw < protobuf.bin
1: "Baidu"
2 {
  1: "Mike"
  2: 29
  3: 1
  4: "A123456"
}
2 {
  1: "Amy"
  2: 25
  4: "A654321"
}
3: 0x075bcd15
4: 0x00005af3107a4000
5 {
  1: "China"
  2: 123
  3: 456
  4 {
    1: "haha@qq.com"
    2: "A123456"
    3: "dalala"
  }
}
6: "\377\362\022\3644"
7: "\001\002\003\004\005\006"
```

恰好，protoc这个程序的代码是开源的，所以为了弄懂这个功能是如何实现的，按照[构建文档](https://github.com/protocolbuffers/protobuf/tree/master/cmake)来,得到一个有源码,可调试的protoc程序。
![](debug_protoc.png)
然后修改程序的运行时参数，添加decode_raw参数、并将一个样例二进制文件作为标准输入。 这样，就可以愉快的单步调试，找到对应部分的代码了。 解析二进制数据核心代码的起始部分调用栈如下:

                                                                                                                                          
```                                                                                                                                       
1   google::protobuf::internal::WireFormat::_InternalParse                                  wire_format.cc            792  0x7ff77988e084 
2   google::protobuf::Message::_InternalParse                                               message.cc                167  0x7ff779841a8c 
3   google::protobuf::internal::MergeFromImpl<0>                                            message_lite.cc           159  0x7ff7796b9346 
4   google::protobuf::MessageLite::ParseFrom<3,google::protobuf::io::ZeroCopyInputStream *> message_lite.h            552  0x7ff7796b99e7 
5   google::protobuf::MessageLite::ParsePartialFromZeroCopyStream                           message_lite.cc           267  0x7ff7796b59d2 
6   google::protobuf::compiler::CommandLineInterface::EncodeOrDecode                        command_line_interface.cc 2384 0x7ff7794a0ecd 
7   google::protobuf::compiler::CommandLineInterface::Run                                   command_line_interface.cc 1121 0x7ff77949b251 
8   google::protobuf::compiler::ProtobufMain                                                main.cc                   104  0x7ff7794349b8 
9   main                                                                                    main.cc                   113  0x7ff779434aaf 
10  invoke_main                                                                             exe_common.inl            79   0x7ff7799723e9 
```

### 一些代码片段
```
//获取包含field_number和 wire_type的key值

// Same as ParseVarint but only accept 5 bytes at most.
inline const char* ReadTag(const char* p, uint32_t* out,
                           uint32_t /*max_tag*/ = 0) {
  uint32_t res = static_cast<uint8_t>(p[0]);
  if (res < 128) {
    *out = res;
    return p + 1;
  }
  uint32_t second = static_cast<uint8_t>(p[1]);
  res += (second - 1) << 7;         
  if (second < 128) {
    *out = res;
    return p + 2;
  }
  auto tmp = ReadTagFallback(p, res);
  *out = tmp.second;
  return tmp.first;
}

std::pair<const char*, uint32_t> ReadTagFallback(const char* p, uint32_t res) {
  for (std::uint32_t i = 2; i < 5; i++) {
    uint32_t byte = static_cast<uint8_t>(p[i]);
    res += (byte - 1) << (7 * i);
    if (PROTOBUF_PREDICT_TRUE(byte < 128)) {
      return {p + i + 1, res};
    }
  }
  return {nullptr, 0};
}
```

这里是protobuf源码中读取varint的方法，因为varint的字节序是小端序，按索引顺序读取字节，若当前字节小于128，即最高位为0，则完成读取，返回结果。否则最高位为1，表示下一字节仍然属于当前值，则读取下一字节。<br/>
当第一个字节大于128时，读取第二个字节， 结果值的计算方式为: res += (second - 1) << 7;  这里减1减掉的其实是第一个字节中的msg(most significant bit)，等同于res = (res - 128) + (second << 7)。
当varint长度大于2时，计算方法一般化为 res += (byte - 1) << (7 * i)。 


``` 
// 获取field字段

static constexpr int kTagTypeBits = 3;

inline int WireFormatLite::GetTagFieldNumber(uint32_t tag) {
  return static_cast<int>(tag >> kTagTypeBits);
}
```


```
//  读取Varint值
template <typename T>
const char* VarintParse(const char* p, T* out) {
  auto ptr = reinterpret_cast<const uint8_t*>(p);
  uint32_t res = ptr[0];
  if (!(res & 0x80)) {
    *out = res;
    return p + 1;
  }
  uint32_t byte = ptr[1];
  res += (byte - 1) << 7;
  if (!(byte & 0x80)) {
    *out = res;
    return p + 2;
  }
  return VarintParseSlow(p, res, out);
}

inline const char* VarintParseSlow(const char* p, uint32_t res, uint64_t* out) {
  auto tmp = VarintParseSlow64(p, res);
  *out = tmp.second;
  return tmp.first;
}

std::pair<const char*, uint64_t> VarintParseSlow64(const char* p,
                                                   uint32_t res32) {
  uint64_t res = res32;
  for (std::uint32_t i = 2; i < 10; i++) {
    uint64_t byte = static_cast<uint8_t>(p[i]);
    res += (byte - 1) << (7 * i);
    if (PROTOBUF_PREDICT_TRUE(byte < 128)) {
      return {p + i + 1, res};
    }
  }
  return {nullptr, 0};
}

```

### 小练习
这个功能的实现中，大部分类型数据的解析只需要按照常规的方式顺序解释二进制数据就好了(例如 Varint、64-bit、32-bit)。 主要的问题在于对Length-delimited类型数据的解析，在没有.proto定义的情况下，是无法确定一段二进制数据究竟是 embedded messages类型 还是 string\bytes类型的。 源码中大概的步骤是:
- 第一步顺序读取字节流，按对应规则解释二进制数据，存入一个UnknownFieldSet的结构中。在第一步中对于Length-delimited数据具体是什么类型不做判断和区分。
- 第二步打印UnknownFieldSet数据。打印的过程中，对于Length-delimited类型的数据会首先尝试以embedded messages的方式解析这段二进制数据，如果解析成功,则按照embedded messages来解析和打印，否则则按照utf-8编码的string类型来打印。

为了验证一下对这部分代码的理解，顺带练习一下python, 用python重新实现了一遍这个小功能。基本实现了 protoc --decode_raw 这个命令的功能(可能会有bug)，另外对string/bytes类型数据做了区分、把普通的repeated elements放在了一个数组里。代码总共不到三百行，对于这个功能具体实现细节感兴趣的同学可以拿来玩一玩。代码放在github上: [protobuf_decoder](https://github.com/liujianfengv/protobuf_decoder)
```
// 拿同样的数据试下
$ python protobuf_decoder.py example/protobuf.bin
{
    "1": "Baidu",
    "2": [
        {
            "1": "Mike",
            "2": 29,
            "3": 1,
            "4": "A123456"
        },
        {
            "1": "Amy",
            "2": 25,
            "4": "A654321"
        }
    ],
    "3": 123456789,
    "4": 100000000000000,
    "5": {
        "1": "China",
        "2": 123,
        "3": 456,
        "4": {
            "1": "haha@qq.com",
            "2": "A123456",
            "3": "dalala"
        }
    },
    "6": "ff f2 12 f4 34",
    "7": "\u0001\u0002\u0003\u0004\u0005\u0006"
}
```

## 相关源码
- [protobuf_decoder](https://github.com/liujianfengv/protobuf_decoder)

## 参考链接
- [Encoding | Protocol Buffers | Google Developers](https://developers.google.com/protocol-buffers/docs/encoding)
- [protocolbuffers / protobuf](https://github.com/protocolbuffers/protobuf)