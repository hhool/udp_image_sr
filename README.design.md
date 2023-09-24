# 设计文档

## 1.项目背景

假设有两路网络摄像头，通过各自的网络UDP协议传输到上位机，上位机有两个网卡，每个网卡对应一个接收通道收取摄像头数据。
每个摄像头的帧率都是60帧/s ，每个摄像头的每一帧具有时间戳（服务器时间）。但是上位机接收两路并不一定同步。

设计一个上位机接收同步机制，如果收到两路帧是同步的，并且数据是正确的（正确的定义是没错丢包，错包，同步的定义是时间戳的差不超过某个固定值），那么将这两帧数据绑定一块。进行下一帧的同步操作。

要求具有较高的容错性
尽可能高的同步速度
尽可能少的扔掉有效帧

## 2.设计思路

## 3.摄像机数据格式

图像格式为RGB，YUV，MJPEG，H264，H265等。
由于网络摄像机概念比较广泛，所以这里暂定摄像机端是可以进行二次开发。
考虑图像数据无损压缩，图像格式可选 RGB，YUV，MJPEG。
考虑通道带宽，图像格式选为MJPEG 压缩率【1:10】到 【1:20】。

## 4.专用词汇

1. 图像数据包：摄像机采集到的图像数据，每个图像数据包都有一个时间戳，时间戳的单位是服务器的NTP时间戳。
   简称：图像数据， 数据包，img_data_t，img_data。
2. 图像数据包拆分包：将图像数据包拆分为多个数据包，每个数据包的大小为1400字节，每个数据包都有一个序号，序号从0开始，每个数据包的序号是连续的。
   简称：数据拆分包， 拆分包， img_data_t，img_data，。
3. 图像数据包队列：用于存储图像数据包的队列，队列的长度为10，队列的长度可以根据实际情况进行调整。
   简称：图像数据队列， 数据队列。
4. 图像数据包拆分包队列：用于存储图像数据包拆分包的队列，队列的长度为10，队列的长度可以根据实际情况进行调整。
5. 图像数据包拆分包组装：将图像数据包拆分包组装为图像数据包。
   简称：数据拆分包组装， 拆分包组装。

## 5.发送端

`图像数据包的格式`:

```cpp
/*
    图像数据拆分包格式，用于网络传输。
    __packed 用于取消结构体字节对齐 紧凑型结构体
*/
struct __packed img_data_t {
    /*
        数据包id，
        针对同一幅可展现的图片完整数据，id唯一。
        id唯一。
        下一副数据包id自增。
        默认值为0。
    */
    uint64_t id = 0;
    /*
        数据包时间戳，取系统NTP时间戳。
        针对同一幅可展现的图片完整数据，时间戳相同。
        默认值为0。
    */
    uint64_t timestamp = 0;
    /*
        数据包拆分后的包序号, 用于排序组完成可展现图片完整数据。
        同时用于判断数据拆分包是否因接收超时或者丢失辅助运算。
        默认值为0，每个数据包的序号是连续的。
    */
    uint16_t segment_id = 0；
    /*
        数据拆分包针对可展现图片数据是否是最后一个数据包。
        end = 0，表示不是最后一个数据包。
        end = 1，表示是最后一个数据包。segment_id 为最后一个数据包的序号。
        检查缓存队列针对当前图片数据id连续性是否已经接收完成。
    */
    uint8_t end = 0；
    /*
        数据拆分包数据实际长度
        data_len <= 1400
    */
    uint16_t data_len = 0；
    /*
        数据拆分包数据 根据数据长度data_len进行数据拷贝
        由于UDP协议的数据包大小限制，所以每个数据包的数据长度不能超过1400字节
    */
    uint8_t data[1400];
};

const int32_t fps = 60;
```

### 5.1 发送策略

   `流程图`：

   ```mermaid
    graph LR
    subgraph 发送线程
        direction TB
        queue1[图像数据包队列] --> bottom1[数据包拆包发送]
    end
    A[摄像机] -- MJPEG--> 发送线程[发送线程]
   ```

`说明`：

1. 摄像机采集到的MJPEG图像数据，记录当前时间戳，时间戳的单位是服务器的NTP时间戳，id自增，追加到发送线程队列中图像数据的队列。
2. 发送线程从图像数据的队列头中取出图像数据，进行拆包为img_data_t格式，填充数据包的时间戳，id，拆分包序号，拆分包数据长度，拆分包数据，然后发送数据拆分包。
3. 发送线程考虑图片数据包的发送速度均匀进行，考虑img_data_t的发送间隔时间设置，图像数据的队列大小设置。
4. img_data_t的发送间隔时间设置：根据实时性要求，默认不设置或者设置为sleep(0)，这样会导致图像数据的有一定延迟。
5. 设置图像数据的队列大小设置：10 主要防止系统异常强况下，导致图像数据队列溢出，从而导致图像数据的丢失。正常状态下，图像数据的队列大小应该是【0，1】。

`备注`：

1. 端上具体一张MJPEG图像数据延时时间计算：第3步发送MJPEG图像数据拆分包的时间戳完毕时间点 - 第1步摄像机采集到MJPEG图像数据的时间戳。
   进而可以计算出端上的MJPEG图像数据延时时间的最大值，最小值，平均值 和 某一段时间内的MJPEG图像数据延时时间的最大值，最小值，平均值。

### 5.2 接收策略-上位机

 `图像数据包的格式`：

```cpp
/*
    图像数据包接收格式，用于接收维护img_data_t拼凑一个完整可解的。
    __packed 用于取消结构体字节对齐 紧凑型结构体
*/
struct __packed img_recv_t {
    /*
        socket fd，用于epoll或者完成端口模式，接收数据包。
    */
    long fd；

    /*
        数据包id，用于判断数据包是否重复，超时，丢失判定。
        针对同一幅可展现的图片完整数据，id唯一。 数据包来自发送端的img_data_t::id。
    */
    uint64_t id;
    /*
        数据包时间戳，取系统NTP时间戳。
        针对同一幅可展现的图片完整数据，时间戳相同。 数据包来自发送端的img_data_t::timestamp。
    */
    uint64_t timestamp;
    /*
        数据包拆分后的包序号, 用于排序组完成可展现图片完整数据。
        同时用于判断数据拆分包是否因接收超时或者丢失辅助运算。
    */
    LinkedList<img_data_t> segment_list;

    /*
        记录当前id下的最后一个数据分拆包的序号。用于判断当前id下的数据包是否接收完成。
        默认值为0。
        付值条件：当接收到数据包的img_data_t::end = 1，那么将当前数据包的segment_id付值给end_segment_id。
    */
   uint16_t end_segment_id = 0;

    /*
        数据包数据完整性标志位，用于判断数据包是否接收完成。
        0：未接收完成。
        1：接收完成。
        默认值为0。
        付值条件：当end_segment_id = 1，且 segment_list.size() == end_segment_id + 1。
    */
   uint8_t recv_finish = 0;
    //////////////////////////////////////
    /// 以下为拆分包组装流程辅助运算, 用于分析数据包是否超时，
    /// 统计完整图片接收完毕所用时间消耗。
    /*
        第一次接收到数据拆分包的时间戳，取系统本地NTP时间戳。
        用于判断数据包是否超时。
        单位：ms 毫秒
    */
    uint64_t first_timestamp_id_seen;

    /*
        最后一次接收到数据拆分包的时间戳，取系统本地NTP时间戳。
        用于计算数据包的接收耗时。
        单位：ms 毫秒
        默认值为0。
    */
    uint64_t last_timestamp_id_seen = 0；
};

/*
    接收数据包集合Map表，用于接收维护img_data_t拼凑一个完整可解的。
    long 为socket fd
*/
static std::map<long, LinkList<img_recv_t>> img_recv_list_map;

/*
    通道接收数据包列表最大长度，及每个fd对应的img_recv_list_map最大长度。
*/
static int32_t img_list_max_size = 10;


//////////////////////////////////////
/* 以下为默认时间同步的图像数据包的结构体信息 */
struct __packed sync_baseline {
/*
   默认时间同步的图像数据包的fd
*/
long fd= -1;

/*
    默认时间同步的图像数据包的id，取自img_recv_t。
*/
 uint64_t id = 0;

/*
    默认时间同步的图像数据包的时间戳， NTP时间戳，取自img_recv_t。
*/
 uint64_t timestamp= 0;
}

/*
    渲染同步基准结构体信息
*/
static std::map<long，sync_baseline> render_sync_baseline;

/*
    渲染同步基准时间差值，
    主要是根据接收端 在2路渲染队列LinkList<img_recv_t>都存数据包img_recv_t的情况下, 且含有img_recv_t.recv_finish完毕的数据包的情况下,
    计算出系统NTP时间戳和2个通道接收完毕的数据包的时间差，然后保存到render_diff_time中。
    目前只用于统计时间差值输出到日志中。
*/
static std::map<long, uint64_t> render_diff_time;
```

### 5.2 框架流程

   `流程图`：

   ```mermaid
    flowchart LR
    subgraph recv[接收处理]
        direction TB
        loop[等待事件] -->|fd| recv_packet[接收数据包处理流程] --> loop[等待事件]
        loop -->|timer| merge[拼接数据包流程] --> render[渲染数据流程] --> loop[等待事件]
        loop -->|exit| exit[退出]
    end
    init[1.初始化] --> recv[接收处理] --> deinit[6.反初始化]

   ```

`说明`：

1. 初始化: 创建2个socket和1个timer，每个socket对应一个网卡，每个网卡对应一个接收通道,
   render timer 值取 1000 / fps。
   启动一个接收线程，接收线程用epoll或者完成端口模式，

2. 根据接收通道的fd的可读事件，进行相应通道数据拆分包接收处理流程.
3. timer定时器事件处理，用于判断2路数据img_recv_list_map拆分包是否超时和拼接2路数据并渲染处理流程。
4. 根据fd处理数据拆分包到相应img_recv_list_map。
5. 接收线程接收到数据包后，将数据包追加到到相应的图像数据包队列中。

6. 反初始化: 线程资源回收，内存空间释放等，清理资源等。

### 5.3 接收数据包处理流程

`流程图`：

   ```mermaid
    graph LR
    subgraph 接收数据包处理流程
        direction TB
        a{1.拆分包超时检测} -->|无效数据| b[丢弃数据包]
        a -->|有效数据| c[拆分包组装流程]
    end
   ```

`说明`：

1. 拆分包超时检测：判断img_data_t::timestamp是否小于render_sync_baseline[fd].timestamp，如果小于，无效数据， 否则有效数据。
2. 拆分包组装流程：img_data_t::id，img_data_t::timestamp，img_data_t::segment_id，img_data_t::end，img_data_t::data_len，img_data_t::data，组装成img_recv_t
3. 拆分包组装完成后，将img_recv_t追加到相应的图像数据包队列中。

### 5.4 拆分包组装流程

`流程图`：

   ```mermaid
    graph LR
    subgraph 拆分包组装
        direction TB
        a{"1.img_recv_list_map[fd]是否存在img_recv_t LinkList队列"}
        a -->|是| b["获取img_recv_t LinkList队列引用"] --> e["更新img_recv_t LinkList队列"]
        a -->|否| c["创建img_recv_t LinkList队列付值img_recv_list_map[fd]"] --> e["更新img_recv_t LinkList队列"]
    end
   ```

`说明`：

### 5.5 更新img_recv_t LinkList队列

   `流程图`：

   ```mermaid
    graph LR
    subgraph 拆分包组装
        direction TB
        a("1.图像数据接收队列是否含有当前img_data_t::id") -->|是| b["获取img_recv_t对象引用"] --> e["更新img_recv_t对象引用"]
        a -->|否| c["创建img_recv_t对象付值img_recv_t LinkList引用"] --> e["更新img_recv_t对象引用"]
    end
   ```

### 5.6 更新img_recv_t对象引用

`流程图`：

   ```mermaid
    graph LR
    subgraph 拆分包组装
        direction TB
        a["更新img_recv_t对象的id，timestamp，fd"] --> b["更新img_recv_t对象的segment_list"] --> c["更新img_recv_t对象的end_segment_id"] --> d["更新img_recv_t对象的recv_finish"]
    end
   ```

`说明`：

 1. 更新img_recv_t对象的id，timestamp，fd: 将img_data_t::id，img_data_t::timestamp，img_data_t::fd，追加到img_recv_t对象中。
 2. 更新img_recv_t对象的segment_list： 将img_data_t::segment_id，img_data_t::data_len，img_data_t::data，追加到img_recv_t::segment_list中。
 3. 更新img_recv_t对象的end_segment_id： 如果img_data_t::end = 1，那么将img_data_t::segment_id，付值给img_recv_t::end_segment_id。
 4. 更新img_recv_t对象的recv_finish： 如果img_recv_t::end_segment_id = 1，且 img_recv_t::segment_list.size() == img_recv_t::end_segment_id + 1，那么将img_recv_t::recv_finish = 1。

### 5.7 渲染数据流程

`流程图`：

   ```mermaid
    graph LR
    subgraph 渲染数据流程
        direction TB
        a["1.获取可拼接数据包"]
        b["2.拼接数据包处理"]
        d["3.更新接收数据包队列"]
        a -->|成功| b -->|成功| c["渲染数据"] --> d
        a -->|失败| d
        b -->|失败| d
    end
   ```

`说明`：

1. 获取可拼接数据包:
   获取系统NTP时间戳，更新每一通道render_diff_first_time。
   满足条件：接收端在2路渲染队列都存在img_recv_list_map[fd].size() > 0 && LinkList<img_recv_t> 某一图像数据接收完成img_recv_t.recv_finish==1。
   各自计算出2路数据包的接收端系统端与采集时间戳的时间戳差值，然后更新接收端 render_diff_time。
   用于后续计算每一个渲染队列LinkList<img_recv_t>的数据包中选择的可用于拼接渲染的节点数据。
   存在的问题：
    * linklist<img_recv_t>中的数据包的时间戳和采集端的时间戳的时间差值，可能会出现负值，这个时候需要考虑这个问题。
    * 在发送端NTP时间完全一致的情况下，选取的2通道的img_recv_t的数据包的时间戳差值绝对值过大。
      由于网络抖动原因，设备第一次启动，可能会出现这个问题。考虑系统首次前几次拼接的图片不做显示渲染。

2. 拼接数据包处理:
   根据2路渲染队列LinkList<img_recv_t>的数据包中选择的可用于拼接渲染的节点数据，进行拼接处理。

3. 更新接收数据包队列:
   更新接收数据包队列，将已经拼接完成的数据包和以前的数据从接收数据包队列中删除, 更新render_sync_baseline[fd].id，render_sync_baseline[fd].timestamp。

### 总结

1. 考虑到实时性，接收端无状态反馈机制。
2. 网络抖动，丢包，错包，超时等情况直接丢弃数据包。 容错性不高。
3. 只处理了基础链路层的容错处理，没有处理应用层的容错处理，比如图像数据包的id重复，时间戳重复，时间戳小于基准时间戳，时间戳大于基准时间戳，时间戳差值过大等情况。
4. 接收端设置10个图片数据接收缓冲区。用于UDP传输的数据包排序，组装，拼接，渲染。同步性不高。有效真实数据包的丢失率较高。
5. 设计完全依赖基础链路层的质量。

### 迭代优化方向

1. 兼顾实时性基础上，借鉴复用[KCP协议](https://github.com/skywind3000/kcp)的基础上和加入反馈机制，优化接收端的容错性。
2. 优化接收端的同步性。
3. 优化接收端的实时性。

### 参考资料

1. [UDP协议](https://baike.baidu.com/item/UDP/571554?fr=aladdin)
2. [NTP协议](https://info.support.huawei.com/info-finder/encyclopedia/zh/NTP.html)
3. [KCP协议](https://github.com/skywind3000/kcp)