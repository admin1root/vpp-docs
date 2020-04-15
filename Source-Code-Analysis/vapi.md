# VPP API模块

## 概述
vpp API模块可以通过共享内存接口与VPP进行通信。

API包含3部分：

* 通用代码（Common code） - low-level API
* 被生成的代码（generated code） - high-level API
* 代码生成器（code generator） - 用于生成high-level API 例如 用户的插件

### 通用代码

#### C通用代码 
C通用代码是基本的low-level API，提供connect/disconnect功能，执行消息，发现以及发送/接收消息的功能。
C通用代码位于vapi.h中

#### C++通用代码
C++通用代码由vapi.hpp提供，并包含高级API模板，这些模板由generated code专用。

### 被生成的代码（generated code）

源代码目录中的每个API文件都会被自动转化为JSON文件，代码生成器（code generator）会解析JSON文件并生成相应的C（`vapi_c_gen.py`）或C++（`vapi_cpp_gen.py`）代码。

这些代码可以被用户端应用程序引用，以实现与vpp的便捷通信。它们包括：

* 自动字节序交换（byte-swapping）
* 基于上下文（context）的自动请求-响应（request-response）匹配
* 调用回调时自动强制转换为适当的类型（类型安全）
* 自动为dump message发送control-pings

API支持两种通信模式：

* 阻塞blocking
* 非阻塞non-blocking

在阻塞模式下，每当启动操作时，代码就会等待，直到操作完成为止。这就意味着，在发送消息时，调用会阻塞，直到消息可以写入共享内存为止。同样，接收消息也会被阻塞直到消息可用为止。在更高的层次，也就意味着当发送一个请求，调用会一直阻塞直到收到回复为止。

在非阻塞模式下，它们是解耦的，发送请求后，API都会返回VAPI_EAGAIN，由客户端等待和处理响应。

### 代码生成器（code generator）

python代码生成器会生成两种样式的代码：C和C++，并生成high-level API头文件。所有的代码都在头文件中。

## 用法

### Low-level API

有关功能的描述，请参阅`vapi.h`头文件中注释。 建议使用头文件（例如“ vpe.api.vapi.h”或“ vpe.api.vapi.hpp”）提供的更安全的高级API。

### C high-level API

#### Callbacks

为了实现高效率，C high-level API严格基于回调。初始化操作时，会为操作生成一个带有上下文的回调。当绑定请求的响应（或多个请求）到达时，将调用回调。同样的，注册了回调的event到达时，也会调用回调。所有的响应/event的指针都指向共享内存，在回调完成后会立即释放，因此客户端需将数据拷贝出来。

#### 阻塞模式

在函数调用期间，整个操作（即请求或dump）已经完成，并调用了其回调（dump可能为多次调用回调）。

此模式下简单请求的伪代码示例：

```
vapi_show_version(message, callback, callback_context)

1. 生成唯一的内部上下文并将其分配给message.header.context
2. 将消息转换为网络字节顺序
3. 向vpp发送消息（消息已被消费，vpp将释放它）
4. 创建内部“outstanding request context”，其中存储了回调，回调上下文和内部上下文值
5. 调用 dispatch，其作用是接收并处理响应，直到内部“outstanding requests”队列为空。 在阻止模式下，此队列始终最多包含一个项目
```

**注意**：在某些情况下，可能会在调用响应回调之前调用不同的-不相关的回调，例如event存储在共享内存队列中。

#### 非阻塞模式

在非阻塞模式下，所有请求仅被字节序交换（byte-swapped），上下文信息和回调一起存储在本地（因此，在以上示例中，仅执行步骤1-4，而跳过了步骤5）。 调用dispatch取决于客户端应用程序。 这允许在发送/接收消息之间交替变换或具有专用线程来调用dispatch。

### C++ high level API

#### Callbacks

在C ++ API中，响应会自动绑定到相应的“Request”，“Dump”或“Event_registration”对象。 可以选择指定一个回调，然后在收到响应时调用该回调。

**注意**：响应占用共享内存空间，应在不再需要时手动释放（对于结果集）或自动释放（通过销毁拥有它们的对象）。 一旦执行了Request或Dump对象，就无法重新发送该对象，因为vpp占用了请求本身（存储在共享内存中）并且无法访问（设置为nullptr）。

#### 用法

##### Requests & dumps

0. 创建Connection对象并调用connect（）以连接到vpp。
1. 使用typedef（例如Show_version）创建“ Request”或“ Dump”类型的对象
2. 如果需要，使用`get_request（）`获取并处理基础请求。
3. 使用`execute（）`发送请求。
4. 使用“ wait_for_response（）”或“ dispatch（）”来等待响应。
5. 使用get_response_state（）获取状态，并使用get_response（）读取响应。

##### Events

0. 创建一个“Connection”对象并执行相应的“Request”以订阅事件（例如“ Want_stats”）
1. 创建一个带有模板参数的“ Event_registration”，该参数是event的类型。
2. 调用`dispatch（）`或`wait_for_response（）`等待event。 event发生时将调用回调（如果传递给`Event_registration（）`构造函数）。 或者，读取结果集。
