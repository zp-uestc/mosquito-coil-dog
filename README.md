# mosquito-coil-dog
蚊香狗（以下简称MCDog），一个基于nodejs的api网关

### 特性图

![](https://github.com/kazaff/mosquito-coil-dog/blob/master/docs/MCDog.png)

图中，"灰色"表示预留，放在未来实现，"绿色"表示首要任务，"黄色"表示待定。

该中间件主要受到[freedom-api](https://github.com/zengwenfu/freedom-api)的启发，再加上我司项目的需要，所有我决定实践一套满足我们自己需求的网关中间件。为了避免烂尾，我尽可能的在上图中标注出目前计划使用的技术栈，如果有高人，可以将满足图中提到的所有特性的实现共享在ISSUE里，相互交流，感激不尽。

### 蓝图

![](https://github.com/kazaff/mosquito-coil-dog/blob/master/docs/layer.png)

#### 服务编排

在特性图中，详细的描述出了关于服务编排所期望的特性。目前结合我们团队的实际情况，并不是特别急需一套DSL，一个表达力足够的JSON会让使用MCDog的用户使用起来更有亲和力，如此一来也可以暂时略过解析DSL的复杂度。

MCDog将服务编排抽象成两个概念：

- 控制型任务
- 工作型任务

控制型任务表示的是和工作流程控制相关的任务，常见的控制型任务有：

	 1. if ... [elseif ... [else ... ]] / case ... [case ... [default ...]]
	 2. begin ... [then ... [then ... ]]
	 3. pass
	 4. loopSeries
	 5. loopParallel

其中`pass`类型，主要是为测试使用，它会输出给定的output数据。

后两类对应编程语言的"for"或"while"，不过拆分成了串行和并行两种。考虑到实际场景下，应该留意循环类型的任务（意味着循环调用），这种类型的任务通常都预示着接口设计的不合理。所以这里我们暂且将这种情况视为违规配置处理。

各种控制型任务必须支持嵌套来适配复杂的工作流，目前暂时没有计划限制嵌套的层级数，但个人认为实际场景中很少会存在超过5层的嵌套。

工作型任务则表示的实际计算任务单元，常见的有：

	 1. http(s)请求
	 2. 自定义函数（同步或异步）
	 3. RPC等其它通信方式（暂不支持）

目前，我们的项目中只使用了REST API，所以MCDog会内置http(s)任务模块，暂时不会内置其它协议的通信模块。不过考虑到自定义函数中使用者可能需要使用第三方类库，我们还需要找一种 **动态安装加载类库文件的方案**。否则，只能要求用户以http的方式暴露api来完成需要依赖第三方类库的逻辑了，这么做除了不人性化之外，对性能也会有伤害。不过，我们可以将这个问题放在后期再解决，眼前第一阶段的核心问题还是解决针对REST API的服务编排需要。

此外，MCDog会为每个工作流提供一个独立的上下文对象实体，贯穿该工作流中的所有任务，允许任务读写其中的数据。这样，横跨多个任务之间的两个任务也可以通过该上下文对象来传递数据。对于相邻的两个任务，前一个任务的输出会自动作为后一个任务的输入（该特性是否需要支持暂时不确定）。

客户端发送到MCDog暴露出的api上的所有携带参数，都会放在上下文对象实体中，根据参数的携带方式不同，会分别放在不同的属性中，如url携带的，表单提交的，请求头携带的，cookie等。

最后要提到的是，初始化变量定义。在工作流的起始端（也就是json的顶层），允许用户定义一些初始化变量（通常是常量），这些初始化变量也可以引用上下文对象中的属性。

#### 插件机制

MCDog提供基于事件的功能扩展接口SPI，允许用户自定义插件扩展MCDog的功能，但这并不代表会支持热插拔（若有这方面的好思路，不妨在Issue中留下宝贵建议）。插件机制是面向开发者的，仅仅方便二次开发来扩展MCDog的功能以满足个性化的场景。

MCDog提供的事件 hook包含两种大类：

 - 任务级别HOOK
 - 框架级别HOOK

我们举个框架级别的Hook例子：假如你试图自定义一个符合自己需要的控制型任务，你可以写一个自定义的插件，并且绑定到MCDog的"Parse"事件上（事件名称暂定）。


#### 管理界面

MCDog管理后台目前会围绕服务编排模块提供相关的操作界面，例如：服务编排描述语法的排版，纠错等。

#### 持久化

用户创建的服务编排描述，最终都会以字符串的方式保存在redis中，这没有什么好说的~~

### 流程图

![](https://github.com/kazaff/mosquito-coil-dog/blob/master/docs/flow.png)

### DSL例子

[例子](https://github.com/kazaff/mosquito-coil-dog/blob/master/docs/example.js)

### 参考

- [Amazon States Language](https://states-language.net/spec.html)
- [freedom-api](https://github.com/zengwenfu/freedom-api)
- [conductor](https://netflix.github.io/conductor/)
- [JsonPath](http://goessner.net/articles/JsonPath/)
