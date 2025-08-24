# MIDI文件解析

### 一、 MIDI文件

- MIDI，Musical Instrument Digital Interface，音乐数字接口。
- MIDI不发送声音，只发送像音调或音乐强度等数据，因此在不同的设备上，输出的声音也因音源器不同而有差异。

![midi](.image\midi.jfif)

### 二、MIDI文件格式

#### 2.1 大端序转换小端序

- MIDI文件默认使用大端序，对于小端序的处理器，在读取数据之后，需要进行字节转换。
- 在MIDI文件中，常见的大端序数据类型是`uint32_t`和`uint16_t`类型，转换代码如下，可以进行复用。

```c
/************************************************************************
 *@function: be_to_uint16
 *@brief: 将大端序数据转换为小端序(big-endian to little-endian)
 *@param: data，uint16_t数据的起始字节
 *@return: 小端序的uint16_t数据
*************************************************************************/
static uint16_t midi_be_to_uint16(const uint8_t *data)
{
    return ((uint16_t) data[0] << 8) | (uint16_t) data[1];
}
```

```c
/************************************************************************
 *@function: be_to_uint32
 *@brief: 将大端序数据转换为小端序(32位)
 *@param: data，uint32_t数据的起始字节
 *@return: 小端序的uint32_t数据
*************************************************************************/
static uint32_t midi_be_to_uint32(const uint8_t *data)
{
    return ((uint32_t) data[0] << 24) | ((uint32_t) data[1] << 16) | ((uint32_t) data[2] << 8) | (uint32_t) data[3];
}
```

#### 2.2 MIDI结构

![midi_struct](.image\midi_struct.png)

- MIDI文件中包含Header Chunk和Track Chunk两种类型，其中Header Chunk必须存在且仅为1，Track Chunk的数量>=1即可。
- Header Chunk用来存储整个MIDI文件的通用数据；Track Chunk作为音轨，每个Track Chunk表示一个音轨(一条轨道)。

#### 2.3 Header Chunk

![midi_header_struct](.image\midi_header_struct.png)

```c
4D 54 68 64   /* Magic, MThd*/
00 00 00 06   /* Length, 6*/
00 01         /* Format, 1*/
00 02         /* ntrks, 2*/
01 E0         /* Division, 480*/
```

##### 2.3.1 Magic

- Magic字段是MIDI Header的魔数，由于Header Chunk是整个MIDI文件的开端，因此Magic也可以认为是整个MIDI文件的魔数。
- Magic字段固定为"MThd"，在读取完成后，可以使用strcmp进行校验，以确定该文件是否是MIDI文件，该字段是四个字节的拼接，而不是一个uint32_t类型，因此不需要小端序转换。

##### 2.3.2 Length

- Length字段表示Header Chunk中数据段的字节数，需要转换成小端序。
- 对于MIDI文件，Header Chunk的数据段固定是Format+ntrks+Division，因此该值固定是6。在读取完成后，可以使用memcmp进行校验，以确定该MIDI文件是否被破坏。

##### 2.3.3 Format

- Format字段表示MIDI文件的格式，该字段有三种取值，0/1/2分别表示格式0、格式1和格式2，需要转换成小端序。
- **格式0**：MIDI文件中有一个sequence，并且只允许有一条音轨，所有声音都叠加在这一条音轨上，因此音轨中的事件天生顺序执行。
  - 可以类比一首音乐成品，一首音乐就是一个播放序列，即sequence，如果使用过诸如PR等编辑器，将音乐导入后，可以发现这个音乐只有一个轨道，这样的音乐就是格式0。

- **格式1**：MIDI文件中有一个sequence，允许有多条音轨，MIDI规定多个音轨必须从统一的时间0开始并行播放，播放效果是多条音轨叠加的。
  - 可以类比正在创作一首音乐，一首音乐作为sequence，但是这首音乐由不同的声音片段叠加产生，这样的音乐就是格式1。
- **格式2**：MIDI文件中有多个sequence，每个track作为一个sequence，这些track并不会并行，即不会同步播放，每一个sequence都是独立的。
  - 可以类比为一个音乐包，里面包含很多首音乐，这些音乐彼此独立。
  - 格式2在MIDI文件中并不常见，后文不做解析。

##### 2.3.4 ntrks

- ntrks字段表示MIDI文件中的track chunk数量，需要转换成小端序。
- 大多数简单MIDI文件的track chunk数量为2，其中第一个track chunk描述一些通用信息，第二个track chunk描述声音。

##### 2.3.5 Division

- Division字段表示MIDI文件的时间分辨率，需要转换成小端序。
- Division有两种表示方式，metrical time(bit 15 == 0)和time-code-based time(bit 15 == 1)，metrical time格式最常见，下文只讨论该类型。
- 在音乐领域，通常用一个四分音符quarter note表示一个节拍，Division表示一个四分音符的时间，即一拍的时间，单位是tick。
- MIDI文件为了便于存储时间，使用tick表示时间，tick都是整数。
- 在MIDI文件中，有一个参数是节拍速度tempo，通过这个参数以及节拍tick数就可以计算一个节拍的实际时间。

#### 2.4 Track Chunk

![midi_track_struct](.image\midi_track_struct.png)

```c
4D 54 72 6B                                     /* Magic,MTrk*/
00 00 00 0F                                     /* Length,15*/
00 FF 03 00 00 FF 51 03 07 A1 20 00 FF 2F 00    /* EventStream*/
```

##### 2.4.1 Magic

- Magic字段是MIDI Track的魔数，该字段是每个Track Chunk的开端。
- Magic字段固定为"MTrk"，在读取完成后，可以使用strcmp进行校验，以确定该MIDI文件是否被破坏，该字段是四个字节的拼接，而不是一个uint32_t类型，因此不需要小端序转换。

##### 2.4.2 Length

- Length字段表示Header Track事件流中的字节数，需要转换成小端序。

##### 2.4.3 EventStream

- EventStream是字节流，由多个事件组成，解析MIDI文件的重点是解析事件流，需要使用循环逐字节解析。

#### 2.5 Meta Event

- Meat Event提供乐曲的附加信息，大多数位于Track #1中，该类事件不会发送给播放设备，而是被播放器或解析器使用。

```c
/**
 * @brief meta event
 */
typedef struct meta_event
{
    uint8_t magic;    /* 魔数，固定0xFF*/
    uint8_t type;     /* meta事件类型*/
    vlq length;       /* 数据段字节数，该字段是可变长度数值，最小为1字节，最大为4字节*/
    uint8_t *data;    /* 数据段*/
} meta_event_t;
```

##### 2.5.1 Meta 0x00

```c
FF 00 02 00 00  /* 格式0或格式1*/
FF 00 02 00 01  /* 格式2*/
```

- 表示序列号，即sequence num，该事件的length固定为0x02，即数据段只有2字节。
- 若该事件出现在格式0或格式1文件中，必须位于Track #1事件流的最开端，因为这类格式的MIDI文件只包含一个sequence。
- 若该类事件出现在格式2文件中，需要位于每个Track事件流的最开端，表示每个sequence的编号。

##### 2.5.2 Meta 0x01

```c
FF 01 length text
```

- 表示任意文本，可以包含音轨的名称、配置器说明以及用户希望在此位置放置的任何其他信息。
- 该事件的文本应为可以打印的ASCII字符。

##### 2.5.3 Meta 0x02

```c
FF 02 length text
```

- 表示版权声明，应包含字符C、版权年份和版权所有者。
- 该事件的文本应为可以打印的ASCII字符。

##### 2.5.4 Meta 0x03

```c
FF 03 length text
```

- 表示序列名称/轨道名称。
- 对于格式0/格式1的MIDI文件，表示序列名称；对于格式2的MIDI文件，表示轨道名称。
- 该事件的文本应为可以打印的ASCII字符。

##### 2.5.5 Meta 0x04

```c
FF 04 length text
```

- 表示当前音轨所用乐器的名称。
- 该事件的文本应为可以打印的ASCII字符。

##### 2.5.6 Meta 0x05

```c
FF 05 length text
```

- 表示演奏中的歌词。
- 该事件的文本应为可以打印的ASCII字符。

##### 2.5.7 Meta 0x06

```c
FF 06 length text
```

- 表示演奏中的锚点名称。
- 该事件的文本应为可以打印的ASCII字符。

##### 2.5.8 Meta 0x07

```c
FF 07 length text
```

- 表示提示点，即演奏过程中对电影、视频或舞台上发声的事情的描述。
- 事件的文本应为可以打印的ASCII字符。

##### 2.5.11 Meta 0x20

```c
FF 20 01 CC
```

- 该事件用于指明后续事件属于哪个通道，该通道有效到下一个Channel事件或Meta 0x20事件。
- 该事件的length固定为0x01，即数据段只有1字节。
- 该事件主要用于同步歌词。

##### 2.5.13 Meta 0x2F

```c
FF 2F 00 00
```

- 该事件作为事件流的结束点，每一个事件流必须以该事件结尾。

##### 2.5.14 Meta 0x51

```c
FF 51 03 TT[23:16] TT[15:8] TT[7:0]
```

- 该事件表示节拍速度，即每个四分音符的时间，单位是微秒。
- 该事件的length固定是0x03，即数据段有3个字节。
- 数据段的3字节拼接成一个uint32_t类型的数据，该数据即为节拍速度。

##### 2.5.15 Meta 0x54

- 该事件表示音轨块开始的SMPTE时间，只有当Division采用time-code-based time格式的时候存在该事件。

##### 2.5.16 Meta 0x58

```c
FF 58 04 nn dd cc bb
FF 58 04 04 02 18 08  /* 表示4/4拍*/
```

- 该事件表示Time Signature，即乐谱的时间记号。
- 该事件的length固定为0x04，即数据段有4个字节。

- **nn**表示分子。
- **dd**表示分母(2的次方数)。
- **cc**表示每拍的MIDI时钟数(24是四分音符1拍的时钟数)。
- **bb**表示每32分音符的个数(通常为8，即四分音符)。

##### 2.5.17 Meta 0x59

```c
FF 59 02 sf mi
FF 59 02 00 00   /* 表示C大调*/    
```

- 该事件表示调号。
- 该事件的length固定为0x02，即数据段有2个字节。
- **sf**表示升/降号个数，是有符号字节，正数表示升号个数，负数表示降号个数。
- **mi**表示大调或小调，0表示大调，1表示小调。

#### 2.6 Channel Event

- Channel Event是MIDI文件中真正决定“发声”的事件，这类事件代表某个通道上的演奏动作，比如弹下音符、放开音符、换音色、弯音等。

```c
/**
 * @brief channel event
 */
typedef struct channel_event
{
    uint8_t status;     /* channel事件的状态字节，该字节决定了channel事件类型和通道号*/
    uint8_t data_1;     /* 参数1*/
    uint8_t data_2;     /* 参数2*/
} channel_event_t;
```

##### 2.6.1 Note Off

- `status & 0xF0 = 0x80`时，表示关闭通道n的某个音符。
- `status & 0x0F`表示通道号。
- data_1表示note，即音符编号。
- data_2表示velocity，即音符力度(对于Note Off来讲，velocity通常为0)。

##### 2.6.2 Note On

- `status & 0xF0 = 0x90`时，表示开启通道n的某个音符。
- `status & 0x0F`表示通道号。
- data_1表示note，即音符编号。
- data_2表示velocity，即音符力度，当velocity为0时，等价于Note Off。

##### 2.6.3 Polyphonic Key Pressure

- `status & 0xF0 = 0xA0`时，表示对某个音符施加压力。
- `status & 0x0F`表示通道号。
- data_1表示note，即音符编号。
- data-2表示pressure，即压力值。

##### 2.6.4 Control Change

- `status & 0xF0 = 0xB0`时，表示改变某个控制器的值。
- `status & 0x0F`表示通道号。
- data_1表示控制器编号。
- data_2表示控制器的新值。

##### 2.6.5 Program Change

- `status & 0xF0 = 0xC0`时，表示切换音色。
- `status & 0x0F`表示通道号。
- 该类事件只有一个数据data_1，表示音色。

##### 2.6.6 Channel Pressure

- `status & 0xF0 = 0xD0`时，表示对整个通道施加压力。
- `status & 0x0F`表示通道号。
- 该类事件只有一个数据data_1，表示压力值。

##### 2.6.7 Pitch Bend

- `status & 0xF0 = 0xE0`时，表示对整个通道进行音高弯曲。
- `status & 0x0F`表示通道号。
- data_1：高7位。
- data_2：低7位。
- data_1+data_2：14位无符号值，范围是0 ~ 16383，中心是0x2000。

##### 2.6.8 Running Status

- 如果连续多个事件的status(包括类型和通道)相同，那么可以省略后续事件的status，只写数据字节。

```c
00 90 3C 64   /* delta=0, Note On ch0, note=60, vel=100*/
20 3E 64      /* delta=32,Note On ch0, note=62, vel=100*/
```

#### 2.7 Sysex Event

- Sysex Event是系统事件，为设备或标准扩展保留的自定义消息，通常由厂商定义。

```c
/**
 * @brief sysex event
 */
typedef struct sysex_event
{
    uint8_t magic;     /* sysex事件的魔数，有0xF0和0xF7两类*/
    vlq length;        /* 数据段长度，可变长度数值*/
    uint8_t *date;     /* 数据段*/
} sysex_event_t;
```

##### 2.7.1 Sysex 0xF0

- 0xF0表示这是系统消息的**首包**，若这是一个完整的系统消息，数据段末尾必须是0xF7，否则说明有消息**续包**。

```c
F0 05 43 12 00 07 F7  /* 这是一条完整的系统消息*/
```

##### 2.7.2 Sysex 0xF7

- 0xF7表示这是系统消息的**续包**，说明系统消息被分成了多个包，最后一个续包的数据段末尾必须是0xF7，说明系统消息结束。

#### 2.8 delta-time

- delta-time是可变长度数值，每个事件之前都必须携带一个delta-time，用于表示这个事件的发生时间，单位是tick。

```c
00 90 41 3B    /* 0x41对应的音符在0tick处按下，力度是0x3B，播放通道是0*/
```

### 三、VLQ解析 

- MIDI文件中存在可变长度数值，这类数据的最小字节数是1，最大字节数是4。
- 对于可变长度数值，每个字节只使用bit 0 ~ bit 6表示，bit 7用来表示数据状态，1表示后续还有数据，0表示后续没有数据。
- 如果数据大于127，那么就需要使用多个字节进行表示，这里以0x2000为例：
  - 0x2000的二进制表示为10 0000 0000 0000，将二进制数据从bit 0开始以7位一组进行切分。
  - 低字节为000 0000，这里应该把bit 7设置为0，因为最低字节表示数据完结，即0x00。
  - 高字节为100 0000，这里应该把bit 7设置为1，因为后面需要拼接一个字节，即0xC0。
  - 因此0x2000用可变长度数值表示为两个字节0xC0、0x00。

```c
/************************************************************************
 *@function: midi_vql_to_uint32
 *@brief: 将midi事件流中的可变长数据VQL转换成uint32位类型
 *@param: eventstream：事件流指针
 *@param: eventstream_len：事件流字节数
 *@param: idx：当前正在处理的字节索引
 *@param: vql：存储转换后的值
 *@return:
*************************************************************************/
static int midi_vql_to_uint32(uint8_t *eventstream, uint32_t eventstream_len, uint32_t *idx, uint32_t *vql)
{
    /* 定义返回值*/
    uint32_t retval = 0;
    /* 定义当前操作字节*/
    uint8_t current = 0;
    for (int i = 0; i < MIDI_VLQ_BYTE_MAX_LEN; i++)
    {
        /* 判断是否合法*/
        if (*idx >= eventstream_len)
        {
            return -1;
        }
        /* 获取当前字节并更新事件流指针*/
        current = *(eventstream + *idx);
        /* 拼接当前字节*/
        retval = (retval << 7) | (current & 127);
        /* 更新事件流索引*/
        (*idx)++;
        if (!(current & 128))
        {
            /* 当前字节的最高位为0，没有后续字节*/
            break;
        }
    }
    /* 返回结果*/
    *vql = retval;

    return 0;
}
```

### 四、MIDI音符编号表

![midi_note_num](.image\midi_note_num.png)

### 五、MIDI音符与频率

- MIDI音符编号和实际的物理频率之间有一一对应的数学关系。
- MIDI标准规定，A4的频率是440Hz，所有其他音符都以A4为基准，按照十二平均律等比排列。
- 换算公式：$f = 440 \times 2^{\tfrac{n - 69}{12}}$
  - **f**表示音符对应的频率。
  - **n**表示音符的MIDI编号。
  - **12**表示一个八度有12个半音。