---
layout:     post
title:      "（转）应用层/安全层/传输层如何进行协议选型？"
subtitle:   "how-to-choose-protocal"
date:       2017-03-20 16:54:00
author:     "沈歌"
header-img: "img/home-bg-o.jpg"
tags:
- 协议选型
- im
---

### 系统设计，协议先行

大部分技术人没有接触协议的设计细节，更多的是使用已有协议进行应用层的编码，例如：

- 使用http作为载体，设计get/post/cookie参数
- 使用dubbo框架，而不用去深究内部的二进制包头包体，以及序列化反序列化的细节

无论如何，了解协议设计的原则，对深入理解系统通信非常有帮助。今天就以即时通讯（后称im）为例，讲讲应用层的协议选型。

### 一、im协议的分层设计

所谓“协议”是双方共同遵守的规则，例如：离婚协议，停战协议。协议有语法、语义、时序三要素。

- 语法：即数据与控制信息的结构或格式
- 语义：即需要发出何种控制信息，完成何种动作以及做成何种响应
- 时序：即事件实现顺序的详细说明

im协议设计分为三层：应用层、安全层、传输层

| 分层 |
|----|
| 应用层 |
| 安全层 |
| 传输层 |

### 二、 im应用层协议设计

应用层协议选型，常见的有三种：文本协议、二进制协议、流式XML协议。

#### 1. 文本协议

文本协议是指“贴近人类书面语言表达”的通讯传输协议，典型的协议是http协议，一个http协议大致长成这样：

```
GET /HTTP/1.1
User-Agent:curl
Host:musicmi.net
Accept:*/*
```

文本协议的特点是：

- 可读性好，便于调试
- 扩展性好（通过key：value扩展）
- 解析效率一般（一行一行读入，按照冒号分割，解析key和value）
- 对二进制的支持不好，比如语言、视频

#### 2. 二进制协议

二进制协议是指binary协议，典型是IP协议，以下是IP协议的一个图示：

![](https://shenpengyan.github.io/img/in-post/how-to-choose-protocal/tcp.jpeg)

二进制协议一般定长包头和可扩展变长包体，每个字段固定了含义，例如IP协议的前4个bit表示协议版本号（version）

二进制协议有这样一些特点：

- 可读性差，难于调试
- 扩展性不好，如果要扩展字段，旧版协议就不兼容了，所以一般设计时会有一个version字段
- 解析效率超高（几乎没有解析代价）

对二进制的支持不好，比如语音、视频

im中，QQ使用的是二进制协议。

#### 3. 流式XML协议

im的准标准协议xmpp就是使用流式XML，像gtalk，校内通这些im都是基于xmpp的，让我们来看一个xmpp协议的例子：

``` xml
<message to='romeo@example.net' from='juliet@example.com' type='chat' xml:lang='en'>
<body>How are you?</body>
</message>
```

从xml标签中大致可以判断这是一个romeo发给juliet的聊天消息。

xmpp协议可以实现跨域的互通。例如gtalk和校内通用户聊天。只要服务端实现了s2s服务（server to server），不过现在的im基本没有互通需求，所以这个服务基本没有人实现。

xmpp协议有几个特点：

- 它是准标准协议，可以跨域互通
- XML的优点，可读性好，扩展性好
- 解析代价超高（dom解析）
- 有效数据传输率超低（大量的标签）

个人旗帜鲜明的强烈不建议使用xmpp，特别是无线端im，如果要用，一定要自己做压缩，减少网络流量（用过xmpp的同学都清楚，发一个登录包需要多少交互，要浪费多少流量）

#### 实际的例子

下面来看一个im协议的实际例子，一般常见的做法是：定长二进制包头，可扩展变长包体。

包体可以使用文本、XML等扩展性好的协议。
包头负责传输和解析效率，与业务无关。包体保证扩展性，与业务相关。

这是一个实际的16字节im二进制定长包头

```C++
//sizeof(cs_header)=16
struct cs_header
{
uint32_t version;
uint32_t magic_num;
uint32_t cmd;
uint32_t len;
uint8_t data[];
}__attibute__((packed));
```

- 前4个字节是version
- 接下来的4个字节是“魔数”,用来保证数据错位或丢包问题，常见的做法是，包头放几个约定好的特殊字符，包尾放几个约定好的嗯特殊字符、约定好，发给你的协议，某几个字节位置，是0x01020304，才是正常报文
- 接下来是command（命令号），用来区分是keepalive报文、业务报文、密钥交换报文等；
- len（包体长度），告知服务端要接收多长的包体

这是一个实际的可扩展im变长包体

```protobuf
message CUserLoginReq
{
	optional string username = 1;
	optional string passwd = 2;
}

message CUserLoginResp
{
	optional uint64 uid = 1;
}
```

使用的是google的Protobuf协议，可以看到，登录请求包传入的是用户名和密码，登录响应包返回的是用户的uid。

当然，除了Protobuf，可选择的可扩展包协议还有xml、json、mcpack等。

个人旗帜鲜明的推荐使用Protobuf，主要有几个原因：

- 现成的解析库种类多，可以生成C++、java、php等代码
- 自带压缩功能
- 在工业界已广泛使用
- google制造

### 三、im安全层协议设计

im协议，消息的保密性非常重要，谁都不希望自己聊天内容被看到，所以安全层是必不可少的。

1. SSL

证书管理微微复杂，代价有点高

2. 自行加解密

自己来搞加解密，核心在于密钥的生成和管理，密钥管理方式有多种，主要有这么三种：

- 固定密钥

	服务端和客户端约定好一个密钥，同时约定好一个加密算法（eg：AES），每次客户端im在发送前，就用约定好的算法，以及约定好的密钥加密再传输，服务端收到报文后，用约定好的密钥再解密。这种方式，密钥和算法对程序员都是透明的。
	
- 一人一密钥

	简单来说就是每个人的密钥是固定的，但是每个人之间又不同，其实就是在固定密钥的算法中包含用户的某一特殊属性，比如用户uid、手机号、qq号等。
	
- 动态密钥（一session一密钥）

	动态密钥，一session一密钥的安全性更高，每次会话前协商密钥。
	
	密钥协商的过程要经过2此非对称密钥的随机生成，1次对称加密密钥的随机生成，具体详情这里不展开，有兴趣的同学可以看下SSL密钥协商过程。
	
### 四、im传输层协议设计

可选的协议有TCP和UDP
现在的im传输层基本都是使用TCP，有了epoll等技术后，多连接就不是瓶颈了，单机几十万链接没什么问题。

	
	


