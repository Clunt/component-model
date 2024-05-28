# 组件模型的高层目标

(对比参阅[WebAssembly高层目标原稿])

1. 定义一种可移植、加载和运行时高效的二进制格式，用于由WebAssembly核心模块构建的可单独编译的组件，从而实现可移植的跨语言组合。
2. 支持定义可移植、可虚拟化、可静态分析、功能安全、语言无关的接口，尤其是[WebAssembly系统接口(WebAssembly System Interface - WASI)]定义。
3. WebAssembly独特价值主张的维护与强化：
   * *语言中立*：避免使组件模型偏向于一种语言或一类语言。
   * *可嵌入性*：以可嵌入多种主机执行环境设计组件，包括浏览器、服务器、中间设备、小型设备和数据密集型系统。
   * *可优化性*：给预先编译器(Ahead-of-Time compilers)最大化地提供静态信息，用以最小成本的实例化和启动。
   * *形式语义*：在与核心wasm相同的语义框架内定义组件模型。
   * *Web平台集成*：通过扩展已有的Webassembly集成点（[JS API]、[Web API]、[ESM-integration]）以确保浏览器可原生支持组件。在实现原生支持之前，确保组件可通过预先编译方式兼容([polyfill](https://remysharp.com/2010/10/08/what-is-a-polyfill))当前浏览器支持的功能。
4. *渐进的*定义组件模型：从一组[初始用例]开始，随时间扩展用例集，根据反馈和经验确定优先级。

## 非目标

1. 不要尝试解决100%的WebAssembly嵌入场景。
   * 某些场景需要与上述目标相冲突的功能。
   * 通过分层规范方法，可以通过替代分层规范或直接嵌入现有的WebAssembly核心规范，来解决不支持的嵌入场景。
2. 不要尝试解决那些更适合通过工具链、平台或更高层规范组合来解决的问题，包括：
   * 包管理和版本控制；
   * 部署和实时升级/动态重新配置；
   * 持久性和存储；
   * 分布式计算和部分故障。
3. 不要指定一组“组件服务”。
   * 指定可由主机实现并暴露给组件的服务属于WASI的领域，超出了组件模型的范围。
   * 另请参阅[WASI常见问题条目](FAQ.md#how-does-wasi-relate-to-the-component-model).


[WebAssembly高层目标原稿]: https://github.com/WebAssembly/design/blob/main/HighLevelGoals.md
[WebAssembly系统接口(WebAssembly System Interface - WASI)]: https://github.com/WebAssembly/WASI/blob/main/README.md
[JS API]: https://webassembly.github.io/spec/js-api/index.html
[Web API]: https://webassembly.github.io/spec/web-api/index.html
[ESM-integration]: https://github.com/WebAssembly/esm-integration/tree/main/proposals/esm-integration
[初始用例]: UseCases.md#Initial-MVP
