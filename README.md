
目录* [1 正常的消息接收流程](https://github.com)
	+ [1\.1 SubscriberImpl::take](https://github.com)
	+ [1\.2 BaseSubscriber::takeChunk](https://github.com)
	+ [1\.3 SubscriberPortUser::tryGetChunk](https://github.com)
	+ [1\.4 ChunkReceiver::tryGet](https://github.com)
	+ [1\.5 ChunkQueuePopper::tryPop](https://github.com)
* [2 回调函数接收机制](https://github.com)
* [3 小结](https://github.com)

0 导引


* [iceoryx源码阅读（一）——全局概览](https://github.com "iceoryx源码阅读（一）——全局概览")
* [iceoryx源码阅读（二）——共享内存管理](https://github.com "iceoryx源码阅读（二）——共享内存管理")
* [iceoryx源码阅读（三）——共享内存管理（一）](https://github.com "- iceoryx源码阅读（三）——共享内存管理（一）"):[FlowerCloud机场](https://hanlianfangzhi.com)
* [iceoryx源码阅读（四）——共享内存通信（二）](https://github.com "- iceoryx源码阅读（四）——共享内存通信（二）")
* [iceoryx源码阅读（五）——共享内存通信（三）](https://github.com "iceoryx源码阅读（五）——共享内存通信（三）")
* iceoryx源码阅读（六）——共享内存创建
* iceoryx源码阅读（七）——服务发现机制
* [iceoryx源码阅读（八）——IPC通信机制](https://github.com "iceoryx源码阅读（八）——IPC通信机制")
* iceoryx源码阅读（九）——等待与通知机制


最近有几个网友希望我续写《iceoryx源码阅读》的剩余部分，所以还是继续写，供大家参考。


本文主要介绍正常的消息接收流程，从流程本身而言，这部分代码逻辑比较简单。为了内容充实，我们对代码做了更深入地解读，结合对前面几篇文章的内容，从代码角度进行分析。


## 1 正常的消息接收流程


正常的消息接收流程是指接收端调用接收函数，从共享消息队列中获取消息所在位置的描述信息，然后根据这一描述信息，从共享内存读取数据的过程。下面，我们iceoryx向应用层提供的接收消息的函数开始介绍，逐步深入。


### 1\.1 SubscriberImpl::take


**职责：**
应用层通过SubscriberImpl::take函数接收数据。从逻辑上来说，这个函数通过更底层的方法获取到`ChunkHeader`指针（我们在iceoryx源码阅读（二）中已经介绍这一数据结构），进而获取到用户负载数据，据此得到用户描述消息所用的结构体。


**模板参数：**


* **T：** 消息体类型。
* **H：** 消息头类型。
* **BaseSubscriberType：** 基类类型，不明白为什么iceoryx常常把基类也定义为范型，范型用的太范了。


**入参：**
无


**返回值：**


返回值的类型为`cxx::expected, ChunkReceiveResult>`，包含两部分：


* 正常情况的返回值类型，即：`Sample`，这个类型是对用户负载数据的封装类，以便于更方便地读取其中的数据。
* 错误情况的返回值类型，即：ChunkReceiveResult，其类型定义如下：



```


|  | enum class ChunkReceiveResult |
| --- | --- |
|  | { |
|  | TOO_MANY_CHUNKS_HELD_IN_PARALLEL, |
|  | NO_CHUNK_AVAILABLE |
|  | }; |


```

说明了错误原因。


以下是这个函数的代码清单：



```


|  | template <typename T, typename H, typename BaseSubscriberType> |
| --- | --- |
|  | inline cxx::expectedconst T, const H>, ChunkReceiveResult> |
|  | SubscriberImpl::take() noexcept |
|  | { |
|  | auto result = BaseSubscriberType::takeChunk(); |
|  | if (result.has_error()) |
|  | { |
|  | return cxx::error(result.get_error()); |
|  | } |
|  | auto userPayloadPtr = static_cast<const T*>(result.value()->userPayload()); |
|  | auto samplePtr = iox::unique_ptr<const T>(userPayloadPtr, [this](const T* userPayload) { |
|  | auto* chunkHeader = iox::mepoo::ChunkHeader::fromUserPayload(userPayload); |
|  | this->port().releaseChunk(chunkHeader); |
|  | }); |
|  | return cxx::successconst T, const H>>(std::move(samplePtr)); |
|  | } |


```

**逐段代码分析：**


* **LINE 05 ～ LINE 09：** 调用基类BaseSubscriberType的takeChunk，即：BaseSubscriber的takeChunk方法，获取指向ChunkHeader的指针，关于ChunkHeader，请参考[iceoryx源码阅读（二）第4节](https://github.com "在iceoryx源码阅读（二）第4节有图示")。
* **LINE 10 ～ LINE 15：** 根据指向ChunkHeader的指针，获取指向用户数据的指针，并据此构造便于应用层使用的`Sample`实例并返回。


### 1\.2 BaseSubscriber::takeChunk


**职责：**


模板类`BaseSubscriber`中，包含了类型为`port_t`的成员`m_port`，这就是端口数据结构，iceoryx的端口中封装了队列数据结构，具体见[iceoryx源码阅读（四）第1节](https://github.com "iceoryx源码阅读（四）第1节")。对于订阅者类，`port_t`实际上就是`SubscriberPortUser`。


**入参：**


无


**返回值：** `cxx::expected`


* 正常情况下返回值类型为`const mepoo::ChunkHeader*`，即指向`ChunkHeader`的指针。
* 错误情况下返回值类型为`ChunkReceiveResult`，用以说明获取失败的原因。


**整体代码分析：**



```


|  | template <typename port_t> |
| --- | --- |
|  | inline cxx::expected<const mepoo::ChunkHeader*, ChunkReceiveResult> BaseSubscriber<port_t>::takeChunk() noexcept |
|  | { |
|  | return m_port.tryGetChunk(); |
|  | } |


```

这个函数只是调用`m_port`（这里是枚举类型`port_t`，特化类型为`SubscriberPortUser`）的`SubscriberPortUser::tryGetChunk`方法。


### 1\.3 SubscriberPortUser::tryGetChunk


这个函数的实现代码如下：



```


|  | cxx::expected<const mepoo::ChunkHeader*, ChunkReceiveResult> SubscriberPortUser::tryGetChunk() noexcept |
| --- | --- |
|  | { |
|  | return m_chunkReceiver.tryGet(); |
|  | } |


```

同样是直接调用了`m_chunkReceiver`，其类型为`ChunkReceiver`的`tryGet`方法，这里的范型参数`SubscriberPortData::ChunkReceiverData_t`通过追溯至`SubscriberPortData`，位于iceoryx\_posh/include/iceoryx\_posh/internal/popo/ports/subscriber\_port\_data.hpp:64，发现其为`iox::popo::SubscriberChunkReceiverData_t`类型：



```


|  | using ChunkReceiverData_t = iox::popo::SubscriberChunkReceiverData_t; |
| --- | --- |


```

而`SubscriberChunkReceiverData_t`又是一个类型别名，定义于iceoryx\_posh/include/iceoryx\_posh/internal/popo/ports/pub\_sub\_port\_types.hpp:：



```


|  | using SubscriberChunkReceiverData_t = |
| --- | --- |
|  | ChunkReceiverData; |


```

`ChunkReceiverData`类也是一个范型类，它的基类是第二个范型参数，在上述代码中就是`SubscriberChunkQueueData_t`，为类型别名，实际类型如下：



```


|  | using SubscriberChunkQueueData_t = ChunkQueueData; |
| --- | --- |


```

这说明`ChunkReceiverData`范型类继承自`ChunkQueueData`范型类，这个类是通信队列数据机构，我们在[《iceoryx源码阅读（四）——共享内存通信（二）》](https://github.com "《iceoryx源码阅读（四）——共享内存通信（二）》")介绍过。


`ChunkReceiverData`类中（不包括基类）定义了唯一一个成员`m_chunksInUse`，具体如下：



```


|  | static constexpr uint32_t MAX_CHUNKS_IN_USE = MaxChunksHeldSimultaneously + 1U; |
| --- | --- |
|  | UsedChunkList m_chunksInUse; |


```

这一成员用于缓存接收到的`SharedChunk`，原因下一小节将会说明。需要指出的是，`UsedChunkList`虽然类型名为`List`，但其本质上是一个数组，具体可以参考iceoryx\_posh/include/iceoryx\_posh/internal/popo/used\_chunk\_list.hpp:79\-82中的数据成员：



```


|  | uint32_t m_usedListHead{INVALID_INDEX}; |
| --- | --- |
|  | uint32_t m_freeListHead{0u}; |
|  | uint32_t m_listIndices[Capacity]; |
|  | DataElement_t m_listData[Capacity]; |


```

下面来简单分析一下`ChunkReceiver`的继承结构，`ChunkReceiver`继承自`ChunkQueuePopper`，顾名思义，这个类是从通信队列获取元素的，是对`ChunkQueueData`的封装。原因如下：这里范型类型`ChunkReceiverData_t`，即：`ChunkReceiverData`中定义了一个类型别名：



```


|  | using ChunkQueueData_t = ChunkQueueDataType; |
| --- | --- |


```

这里的`ChunkQueueDataType`其实就是上文中的`ChunkQueueData`，通过它可以访问消息队列`m_queue`。下面，我们介绍`ChunkReceiver::tryGet`的具体实现，进一步加深对上述数据结构的理解。


### 1\.4 ChunkReceiver::tryGet


`ChunkReceiver`为范型类，范型参数


**职责：**


调用基类的成员方法，取出接收队列中的一个元素，函数返回类型为`SharedChunk`，我们在[《iceoryx源码阅读（二）——共享内存管理》](https://github.com "《iceoryx源码阅读（二）——共享内存管理》")中已经介绍过了（现在看来介绍得有点粗糙 :\-）。


**入参：**


无


**返回值：** `cxx::expected`


* 正常情况下返回值类型为`const mepoo::ChunkHeader*`，即指向`ChunkHeader`的指针。
* 错误情况下返回值类型为`ChunkReceiveResult`，用以说明获取失败的原因。


**逐段代码分析：**



```


|  | template <typename ChunkReceiverDataType> |
| --- | --- |
|  | inline cxx::expected<const mepoo::ChunkHeader*, ChunkReceiveResult> |
|  | ChunkReceiver::tryGet() noexcept |
|  | { |
|  | auto popRet = this->tryPop(); |
|  |  |
|  | if (popRet.has_value()) |
|  | { |
|  | auto sharedChunk = *popRet; |
|  |  |
|  | // if the application holds too many chunks, don't provide more |
|  | if (getMembers()->m_chunksInUse.insert(sharedChunk)) |
|  | { |
|  | return cxx::success<const mepoo::ChunkHeader*>( |
|  | const_cast<const mepoo::ChunkHeader*>(sharedChunk.getChunkHeader())); |
|  | } |
|  | else |
|  | { |
|  | // release the chunk |
|  | sharedChunk = nullptr; |
|  | return cxx::error(ChunkReceiveResult::TOO_MANY_CHUNKS_HELD_IN_PARALLEL); |
|  | } |
|  | } |
|  | return cxx::error(ChunkReceiveResult::NO_CHUNK_AVAILABLE); |
|  | } |


```

**LINE 05 \~ LINE 05：** 调用基类（即：上一小节介绍的`ChunkQueuePopper`）成员函数`tryPop`从`ChunkQueueData`维护的消息队列中弹出一个消息元素，返回值为`SharedChunk`，下一小节详细介绍这个函数的具体实现。


**LINE 07 ～ LINE 23：** 从消息队列中成功获取消息，将其存入缓存数组中，目的是：（1）后续根据`ChunkHeader *`指针获取对应的`SharedChunk`；（2）若不存起来，离开这个函数，`SharedChunk`实例被析沟，意味着共享内存被释放。有人会问为什么不直接返回`SharedChunk`呢？其实也是可以的，但还是需要有个地方缓存`SharedChunk`实例的，直到这块共享内存不再被需要。首次获得就缓存起来是一种更好的方式。


**LINE 24 ～ LINE 24：**若缓存数组已满，则释放`SharedChunk`，即：释放了共享内存，并返回错误`ChunkReceiveResult::TOO_MANY_CHUNKS_HELD_IN_PARALLEL`。


### 1\.5 ChunkQueuePopper::tryPop


**职责：**
从消息队列`m_queue`（存放于共享内存中，见：[iceoryx源码阅读（四）——共享内存通信（二）](https://github.com "iceoryx源码阅读（四）——共享内存通信（二）")）获取消息描述数据，并将其转化为`SharedChunk`实例返回。


**入参：**


无


**返回值：** `cxx::optional`


这里不需要说明错误原因等额外信息，所以不像前一个函数那样，使用expect作为返回类型。


* 正常情况下返回值类型为`mepoo::SharedChunk`。
* 错误情况下返回特殊值`nullopt_t`。


**逐段代码分析：**



```


|  | template <typename ChunkQueueDataType> |
| --- | --- |
|  | inline cxx::optional ChunkQueuePopper::tryPop() noexcept |
|  | { |
|  | auto retVal = getMembers()->m_queue.pop(); |
|  |  |
|  | // check if queue had an element that was poped and return if so |
|  | if (retVal.has_value()) |
|  | { |
|  | auto chunk = retVal.value().releaseToSharedChunk(); |
|  |  |
|  | auto receivedChunkHeaderVersion = chunk.getChunkHeader()->chunkHeaderVersion(); |
|  | if (receivedChunkHeaderVersion != mepoo::ChunkHeader::CHUNK_HEADER_VERSION) |
|  | { |
|  | LogError() << "Received chunk with CHUNK_HEADER_VERSION '" << receivedChunkHeaderVersion |
|  | << "' but expected '" << mepoo::ChunkHeader::CHUNK_HEADER_VERSION << "'! Dropping chunk!"; |
|  | errorHandler(PoshError::POPO__CHUNK_QUEUE_POPPER_CHUNK_WITH_INCOMPATIBLE_CHUNK_HEADER_VERSION, |
|  | ErrorLevel::SEVERE); |
|  | return cxx::nullopt_t(); |
|  | } |
|  | return cxx::make_optional(chunk); |
|  | } |
|  | else |
|  | { |
|  | return cxx::nullopt_t(); |
|  | } |
|  | } |


```

**LINE 04 ～ LINE 04：** 从共享内存的消息队列`m_queue`中获取`ShmSafeUnmanagedChunk`元素（请参考[《iceoryx源码阅读（三）——共享内存通信（一）》](https://github.com "《》")）。


**LINE 07 \~ LINE 21：** 获取成功


通过`ShmSafeUnmanagedChunk`的成员方法`releaseToSharedChunk`找到对应的共享内存区域，据此创建`SharedChunk`实例，并返回。注意，LINE 11 ～ LINE 19做了版本检查，如果和接收端版本不一致，同样认为获取失败。版本号是一个整数，用于标识某种不兼容的修改，iceory有如下解释：



```


|  | /// @brief From the 1.0 release onward, this must be incremented for each incompatible change, e.g. |
| --- | --- |
|  | ///            - data width of members changes |
|  | ///            - members are rearranged |
|  | ///            - semantic meaning of a member changes |
|  | static constexpr uint8_t CHUNK_HEADER_VERSION{1U}; |


```

**LINE 22 ～ LINE 25：** 获取失败


直接返回特殊值`cxx::nullopt_t()`，这里会隐式调用`optional`的如下构造函数：



```


|  | optional(const nullopt_t) noexcept; |
| --- | --- |


```

## 2 回调函数接收机制


如果接收端先启动，此时共享内存队列中为空，此时会返回错误`ChunkReceiveResult::NO_CHUNK_AVAILABLE`，如果接收端一直轮询，则会浪费性能。是否可以在发送端通知的机制实现消息的异步监听和处理？答案是肯定的，我们将在《iceoryx源码阅读（九）——等待与通知机制》对消息发送端的通知逻辑和接收端的等待逻辑进行深入的介绍。


## 3 小结


本文主要介绍了正常的消息接收流程，对于异步回调的消息接收流程将在后续文章中进行介绍。


