
# EasyIoT SDK 说明

本 SDK 使用 ANSI C 实现了 EasyIoT 终端接口协议，使用简单的 API 抽象，实现了NB-IoT数据上送与下行指令处理、响应等。
核心文件 `easyiot.c` 代码行数仅千行左右，结构精简，易于阅读与修改。

终端侧代码，请包含头文件 `easyiot.h`。

在EasyIoT平台产品定义处，取出导出头文件中的宏定义部分，如本SDK中的example.h

不同平台编译，本SDK代码在gcc、msvc、armcc中编译测试通过

## 基础用法

基础流程

内存碎片处理
终端与EasyIoT平台的通讯，使用了TLV，即Type-length-value数据报，在TLV的处理过程中使用了malloc进行动态内存分配，长时间运行有可能产生内存碎片，导致malloc分配失败。对此，使用SDK的static系列API，即可规避此问题。
static系列api，使用预分配栈空间或全局内存空间进行TLV的管理，程序编译通过后，其运行时内存布局即为确定布局，无内存碎片问题。

AT指令
本SDK代码，未处理任何AT指令集

非BC95模组
重写CoapOutput函数，如AT+ZIPSEND等

## API

### 结构体

```cpp
enum CoapMessageType {
	CMT_USER_UP = 0xF0,
	CMT_USER_UP_ACK = 0xF1,
	CMT_USER_CMD_REQ = 0xF2,
	CMT_USER_CMD_RSP = 0xF3,
	CMT_SYS_CONF_REQ = 0xF4,
	CMT_SYS_CONF_RSP = 0xF5,
	CMT_SYS_QUERY_REQ = 0xF6,
	CMT_SYS_QUERY_RSP = 0xF7
};
```

CoAP数据类型，分别为用户数据上行，平台下发数据收到确认ACK，用户自平台下发到设备的指令，用户对指令的响应，CMT_SYS的则是EasyIoT公有数据格式，可选择性实现，具体定义在文档中心-终端接口协议中有描述。

```cpp
struct TLV {
	uint16_t length;
	uint8_t type;
	uint8_t vformat;
	uint8_t *value;
};
```

TLV结构体，EasyIoT平台与终端通讯格式，使用了TLV，其中vformat字段标注了本TLV中存储了何种格式的数据，其定义在枚举 `enum TlvValueType`中，包括了如 INT、FLOAT、DOUBLE等常见格式。

```cpp
#define MESSAGE_MAX_TLV 32
struct Messages {
	// dtag_mid，在上行数据中使用 dtag 语义，用于在一个较短的时间内，去除重复上行的数据
	// 在下行指令中，取其mid语义，上行的指令响应必须使用同样的mid值，用以关联指令的执行结果。
	uint16_t dtag_mid;

	// msgid，消息ID，即为整个消息的Type，在EasyIoT平台定义消息时，此值会被定义
	// 在导出的头文件中，亦有此值的宏定义，代码中建议使用宏定义。
	uint8_t msgid;

	// 当前Message对象中的TLV个数总和
	uint8_t tlv_count;

	/* 静态内存分配区，在 MessageMalloc函数中，实现了一个简单的静态内存分配机制，即只分配，不释放。 */
	// 内存buffer起始地址，一旦赋值，将不再变动，具体内存使用，由 sbuf_offset 值确定
	uint8_t* sbuf;
	uint16_t sbuf_offset;
	// 原始 sbuf 的最大长度
	uint16_t sbuf_maxlength;
	// 是否使用静态内存分配机制
	uint8_t sbuf_use;
	/* 静态内存分配区 */

	// enum TlvValueType 值，用于区分是何种类型数据
	uint8_t msgType;

	// TLV对象指针数组
	struct TLV* tlvs[MESSAGE_MAX_TLV];
};
```

Message结构体为EasyIoT平台数据格式抽象，各字段释义见注释；

```cpp
enum LoggingLevel {
	LOG_TRACE,
	LOG_DEBUG,
	LOG_INFO,
	LOG_WARNING,
	LOG_ERROR,
	LOG_FATAL
};
```

日志输出等级，可使用SetLogLevel控制日志输出等级。

### 函数

```cpp
void EasyIotInit(const char* imei, const char* imsi);
```

SDK初始化，使用IMEI与IMSI初始化，

```cpp
typedef uint64_t(*TimestampCbFuncPtr)(void);
typedef uint8_t(*BatteryCbFuncPtr)(void);
typedef int32_t(*SignalCbFuncPtr)(void);
typedef void(*OutputFuncPtr)(const uint8_t* buf, uint16_t length);
typedef void(*CmdHandlerFuncPtr)(struct Messages* req);

void setsTimestampCb(TimestampCbFuncPtr func);
void setSignalCb(SignalCbFuncPtr func);
void setBatteryCb(BatteryCbFuncPtr func);
void setNbSerialOutputCb(OutputFuncPtr func);
void setAckHandler(CmdHandlerFuncPtr func);
int setCmdHandler(int cmdid, CmdHandlerFuncPtr func);
```
- setsTimestampCb，设置时间戳回调函数，请返回当前系统毫秒数，若系统只支持到秒数，返回值乘以1000转换到毫秒数即可。
- setSignalCb，设置信号强度回调函数，请返回RSRP值
- setBatteryCb，设置电池电量回调函数，返回0~100之间的数值，代表电量百分比
- setNbSerialOutputCb，设置与NB通信的输出函数，设置日志输出回调函数
- setAckHandler，设置ACK处理回调函数
- setCmdHandler，设置指令处理回调函数

NewMessage/NewMessageStatic
新建Message对象，Message对象是EasyIoT中数据的封装，上行数据、下行指令都使用此结构体；后者在指定的内存空间内初始化，并返回内存空间起始值；一个Message对象最多容纳32各TLV对象，在头文件中修改宏定义即可去除此限制；但嵌入式环境中请留意内存的使用。

setMessages
设置Message对象类型，使用枚举值CoapMessageType，具体类型释义请参考文档中心的终端接口协议。
设置Message对象ID，请使用平台导出文件中的指定值。

```cpp
int AddInt8(struct Messages* msg, uint8_t type, int8_t v);
int AddInt16(struct Messages* msg, uint8_t type, int16_t v);
int AddInt32(struct Messages* msg, uint8_t type, int32_t v);
int AddFloat(struct Messages* msg, uint8_t type, float v);
int AddDouble(struct Messages* msg, uint8_t type, double v);
int AddString(struct Messages* msg, uint8_t type, const char* v);
int AddBinary(struct Messages* msg, uint8_t type, const char* v, uint16_t length);
```
将一个传感器值，添加到指定的Message对象中

```cpp
int GetInt8(const struct Messages* msg, uint8_t type, int8_t* v);
int GetInt16(const struct Messages* msg, uint8_t type, int16_t* v);
int GetInt32(const struct Messages* msg, uint8_t type, int32_t* v);
int GetLong64(const struct Messages* msg, uint8_t type, int64_t* v);
int GetFloat(const struct Messages* msg, uint8_t type, float* v);
int GetDouble(const struct Messages* msg, uint8_t type, double* v);
int GetString(const struct Messages* msg, uint8_t type, char** v);
int GetBinary(const struct Messages* msg, uint8_t type, uint8_t** v);
```

从一个Message对象中，获取指定传感器的值，以指定的类型。

pushMessages
将数据发送/推送到EasyIoT平台

FreeMessage
释放Message对象所占用的内存空间，但若对应的Message对象使用静态内存分配，即使用NewMessageStatic构造而得，则本函数仅将初始内存段置0。

```cpp
int a2b_hex(const char* s, char* out, int inMaxLength);
```
a2b_hex，将每两字节ASCII表示的HEX数据，转换到1字节的二进制内存byte数据，如2字节的字符串"AA"，将转换到1字节的byte数据 0xAA；

```cpp
int MessagesSerialize(const struct Messages* msg, char* inBuf, uint16_t inMaxLength);
int MessagesDeserialize(const char* inBuf, uint16_t inLength, struct Messages* out);
```
MessagesSerialize，数据序列化，将Messages对象序列化到内存中，使之可被网络传输；MessagesDeserialize，数据反序列化，将从平台得到的数据，使用a2b_hex转换为二进制内存数据后，使用此函数反序列化为Message对象。

```cpp
int CoapInput(struct Messages* msg, uint8_t *data, uint16_t inLength);
int CoapHexInput(const char* data);
int CoapHexInputStatic(const char* data, uint8_t* inBuf, uint16_t inMaxLength);
```
CoapInput，CoapHexInput，CoapHexInputStatic，将平台下行的CoAP数据，输入到EasyIoT SDK中，本函数首先对数据进行反序列化操作，得到对应的Message对象，再根据Message对象的类型，执行具体的操作，如调用ACK回调函数，调用下行指令回调函数。

```cpp
void SetLogLevel(enum LoggingLevel level);
int Logging(enum LoggingLevel level, const char* fmt, ...);
void setLogSerialOutputCb(OutputFuncPtr func);
```
日志处理，使用 SetLogLevel 函数控制日志输出级别，使用 Logging 函数输出日志。

```cpp
int CoapOutput(uint8_t *inBuf, uint16_t inLength);
```
底层接口 输出CoAP 数据流，不同的模组，需要使用不同的PORTING
