## 请求并发概述
### 请求单并发
默认情况下，在调用函数时，云函数会分配一个并发实例处理请求或事件。函数代码运行完毕返回后，该实例会处理其他请求。如果在请求到来时，所有实例都在运行中，云函数则会分配一个新的并发实例。一个并发实例同一时刻仅处理一个事件的运行逻辑，保障每个事件的处理效率和稳定性。

### 请求多并发
在大多数情况下，请求单并发都是值得推荐使用的模式，无需在写代码时考虑多个请求同时处理带来的典型并发难题，例如线程安全、阻塞调用、异常处理等。

而在 Web 应用中，典型的业务场景是 IO 密集型——函数内访问数据库或其他系统的接口等下游服务，会有较多时间在等待这些下游服务响应。这种等待一般都是在做 iowait，不消耗 CPU，此时，如果开启了请求多并发，让一个实例可以同时处理多个请求，则可以更充分利用单个实例的 CPU 资源。

Web 函数目前已经支持 [开启请求多并发](#.E6.93.8D.E4.BD.9C.E6.AD.A5.E9.AA.A4) 配置，您可以根据业务需要进行启用和配置。请求多并发支持**自定义静态并发**、**智能动态并发**两种模式。
- **自定义静态并发**
启用后，当同时有多个请求，将不超过指定并发值的请求调度到同一函数实例内执行。并发增多，将增加函数实例的CPU、内存等消耗，建议配合压力测试进行合理设置，避免函数执行异常。目前支持的并发范围为 2～100 并发。

- **智能动态并发**
启用后，在函数实例负载允许的情况下，智能动态调度更多请求到同一函数实例内运行。将于后续推出，敬请期待。

## 操作步骤
### 开启请求多并发
1. 登录云函数控制台，选择左侧导航栏中的 **[函数服务](https://console.cloud.tencent.com/scf/list)**。
2. 在“函数服务”列表页面，选择需进行配置的 Web 函数名。
3. 在“函数管理”页面中，选择**函数配置**。
4. 在“函数配置”页面中，单击“编辑”，进入编辑模式。
5. 在“请求多并发”中，勾选“启用”，开启请求多并发模式，在弹出的“自定义静态并发”下方的输入框中输入需要的并发值。如下图所示： 
![](https://qcloudimg.tencent-cloud.cn/raw/3231fc273649a2b7ec8785c7d6f713f8.png)
6. 单击“保存”完成配置。

## 请求并发优势
- 在 IO 密集型场景中，如 Websocket 长连接业务，可减少计费执行时长，节省费用。
- 多个请求并发在同一个实例中可复用数据库连接池，减缓下游服务压力。
- 请求并发密集时，多个请求只需要一个实例进行处理，无需拉起多个实例，从而降低实例冷启动几率，降低响应延迟。

## 注意事项
### 计费
未开启请求多并发时，单个函数实例一次只会处理一个请求，第一个请求处理完成才会开始处理下一个请求，内存时间的计费时长是每个请求的执行时长的加和，如下图所示： 
![未开启请求多并发时的计费](https://qcloudimg.tencent-cloud.cn/raw/97581bf1ea06f6ead6d0d8903c4ada7b.png)

开启请求多并发之后，单个函数实例一次会处理多个并发请求，第一个请求未结束时，如果第二个请求进来，则会有一段时间两个请求同时在处理，此时，交叠的这段时间只会计算一次。如下图所示： 
![](https://qcloudimg.tencent-cloud.cn/raw/f4291dacd007dd7d476fffc5bd8f79d1.png)

其他计费项保持不变，详情见 [计费概述](https://cloud.tencent.com/document/product/583/17299)。

### 日志
开启请求多并发后，由于多个并发请求同时处理，每个请求产生的日志在流式上报时，可能会出现日志和 RequestID 无法一一对应。此时，应该在代码中正确设置 logger，将 RequestID 打印到日志中，以解决该问题。RequestID 从 Web 函数中接收到的公共请求头里的 `X-Scf-Request-Id` 字段获取。   

### 超限错误
#### 内存超限
请求多并发会增加内存超限的概率，在内存超限 OOM 发生时，实例会进行重启，此时，实例内正在处理的多个请求会同时出现 abort 中断错误，错误码为 434 MemoryLimitReached。请在设置并发值之前，先对函数进行压测以确定安全的并发值，以避免内存超限带来的影响。

#### 超时
请求耗时过长，在配置的执行超时时间范围内没有执行完成时，会终止该请求，向客户端返回错误码 433 TimeLimitReached。实例内的其他正在进行中的请求不受影响。

### 监控

开启请求多并发后，在监控页面会出现“并发请求个数”面板，可直观看到指定时间段内的请求并发情况。
![](https://qcloudimg.tencent-cloud.cn/raw/dda7d3103ae958dae3efbf35f68663fb.png)
 
