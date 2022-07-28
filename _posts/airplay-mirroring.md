---
title: Airplay镜像传输报文
date: 2020-05-30 11:55:50
categories: Airplay
tags:
    - Airplay
description: Windows平台下Airplay接收端在与IOS设备进行镜像连接时，传输流程以及报文
---
# POST /feedback
发送端每**2**秒向接收端发送心跳包
```
POST /feedback RTSP/1.0
CSeq: 20
DACP-ID: 1A6F2C4DA766ED24
Active-Remote: 2869679872
User-Agent: AirPlay/420.45


```
接收端回应
```
RTSP/1.0 200 OK
CSeq: 20
Server: AirTunes/220.68


```
# NTP
airplay的NTP协议基于UDP协议

**NTP报文 = NTP头部（16字节）+ 4个时间戳（8字节） = 48 字节**

由接收方发出第一份NTP报文
```
0000   10 30 25 a6 e6 b8 fe 5c 68 df 45 d7 08 00 45 00
0010   00 4c 58 10 00 00 80 11 4e 57 c0 a8 89 01 c0 a8
0020   89 e7 e7 98 f4 73 00 38 57 0e 23 00 00 00 00 00
0030   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0040   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0050   00 00 07 55 86 3a 55 73 32 26

// data
xxxx   xx xx xx xx xx xx xx xx xx xx 23 00 00 00 00 00
0030   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0040   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0050   00 00 07 55 86 3a 55 73 32 26
```
NTP头部为**0x23**表示：
- `LI` Leap Indicator，0-2 bits
- `VN` Version Number，2-5 bits，NTP的版本号。这里是4，最新版
- `Mode` Mode，5-8 bits，NTP的工作模式。这里是3，表示client，客户模式

时间精度保留到**毫秒**，在最后8个字节表示。其中前4字节以大端序表示**自1900年以来的秒数**，后4字节同样以大端序表示毫秒

发送方回应NTP报文
```
0000   fe 5c 68 df 45 d7 10 30 25 a6 e6 b8 08 00 45 c0
0010   00 4c 43 77 00 00 40 11 a2 30 c0 a8 89 e7 c0 a8
0020   89 01 f4 73 e7 98 00 38 9a fd 24 01 00 e8 00 00
0030   00 00 00 00 00 00 58 4e 55 31 00 00 00 00 00 00
0040   00 00 07 55 86 3a 55 73 32 26 83 aa e5 4d 16 9d
0050   a8 63 83 aa e5 4d 16 a8 65 0e

// data
xxxx   xx xx xx xx xx xx xx xx xx xx 24 01 00 e8 00 00
0030   00 00 00 00 00 00 58 4e 55 31 00 00 00 00 00 00
0040   00 00 07 55 86 3a 55 73 32 26 83 aa e5 4d 16 9d
0050   a8 63 83 aa e5 4d 16 a8 65 0e
```
关键的信息有：
- `Mode` 5-8 bits，NTP的工作模式。这里是4，表示server，服务器模式
- `Reference Timestamp` 16-24 byte，本地时钟最后一次被设定或者更新的时间
- `Originate Timestamp` 24-32 byte，NTP请求报文离开发送端时发送端的本地时间
- `Receive Timestamp` 32-40 byte, NTP请求报文到达接收端时接收端的本地时间
- `Transmit Timestamp` 40-48 byte，应答报文离开应答者时应答者的本地时间

可以看到，`Originate Timestamp`与接收方发出的报文所带的时间戳一致

## NTP流程
从报文可以看到，NTP协议无加密，使用客户端/服务器模式，而且恰好相反，由**发送端（IOS设备）作为服务器，接收端（TV）作为客户端**

TV先发出NTP报文，内容是`Originate Timestamp`（T1）

IOS设备收到报文后，记录收到的时间`Receive Timestamp`（T2），发送应答报文时带上离开IOS设备的时间`Transmit Timestamp`（T3）

TV收到应答报文后，记录收到的时间T4。此时TV知道了T1、T2、T3、T4，就能计算出：
- 网络延时d = ( T4 - T1 ) - ( T3 - T2 )
- 与IOS设备的时差t = [ ( T2 - T1 ) + ( T3 - T4 ) ] / 2

以上为一轮时间同步

**实际上在第一轮后，TV就以IOS设备的时区为基准传输时间了**
**除了第一轮完成后立即进行下一轮外，TV大约每3秒进行一轮时间同步**

关于NTP协议的更多内容，可以在这里[查询](http://www.023wg.com/message/message/cd_feature_ntp_message.html)

# 视频传输
airplay的**RTP协议**基于**TCP协议**，格式是**H264**，**部分数据加密**

在标准的RTP报文前，还有一段**128**字节的报文头，附带了其他信息
- **0 - 4** RTP Payload块大小`payloadsize`
- **4 - 6** RTP Payload块类型`payloadtype`

`payloadtype`有两种类型：0和1
## `payloadtype` = 1 的数据包
```
xxxx   xx xx 25 00 00 00 01 00 16 01 51 b4 05 4c fd 98
0050   00 00 00 00 b4 44 00 00 87 44 00 00 00 00 00 00
0060   00 00 00 00 00 00 00 00 00 00 00 00 b4 44 00 00
0070   87 44 00 00 70 43 00 00 00 00 00 00 b4 44 00 00
0080   87 44 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0090   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00a0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00b0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00c0   00 00 

// H264
xxxx   xx xx 01 64 00 2a ff e1 00 12 27 64 00 2a ac 13
00d0   14 50 16 80 89 f9 66 e0 20 20 20 40 01 00 04 28
00e0   ee 3c b0 02 00 00 00 ce // payload 块，下面是数据
```
在上方128字节的报文头中
- **40 - 44**字节 源宽，例子是1440
- **44 - 48**字节 源高，例子是1080
- **56 - 60**字节 宽，例子是1440
- **60 - 64**字节 高，例子是1080

其他数据还没懂什么意思

H264数据格式似乎不是遵循标准的RTP协议[RFC 6184](https://tools.ietf.org/html/rfc6184#page-10)
- **1字节** 版本号。第一版
- **2字节** [Profile](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Profiles)。控制编码效率以及压缩率，0x64即100属于**High Profile**
- **3字节** Compatibility 兼容性
- **4字节** [Level](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Levels)。控制解码器的处理能力以及内存容量，0x2a即42属于4.2的级别
- **7 ~ 8字节** SPS Length。SPS数据长度，上图是0x12即18字节
- **后SPS Length字节**SPS数据。
- **后1字节** PPS Number。PPS串数量
- **后2字节** PPS Length。这个数似乎不能直接得出。需要运算`payload[h264.lengthofSPS + 9] & 2040) + payload[h264.lengthofSPS + 10]) & 255`
- **后PPS Length字节**PPS数据

 **`payloadtype` = 1 的数据包整合了H264中的PPS以及SPS，并没有使用加密**

 ## `playloadtype` = 0 的数据包
 ```
xxxx   xx xx f3 16 00 00 00 00 00 00 71 2c 02 f1 69 98
0050   02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0060   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0070   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0080   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0090   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00a0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00b0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00c0   00 00 

// H264
00c0   xx xx d2 92 89 1c f5 8f 6d 1f d7 f9 c9 5e b4 51
00d0   ff 59 b2 a8 68 75 0f f6 f9 08 79 38 a9 0a 79 99
00e0   fb 44 36 05 fe b6 fb 9b 32 0d 10 81 61 4c 52 23
...
 ```
 上方128字节的报文头还不清楚其具体意思

 下方是加密的H264帧，[解密](airplay_handshake_3.md#raop初始化)后即可解码使用