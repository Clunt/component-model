# 组件模型设计和规范（Component Model design and specification）
> 原文：https://github.com/WebAssembly/component-model

此仓库用于组件模型的标准化。如需面向用户的详细说明，请查看**组件模型文档**（**[Component Model Documentation]**）。

此仓库描述了组件模型高层的[目标]、[用例]、[设计选择]和[常见问题解答(FAQ)]，以及更详细的[汇编级解释器(assembly-level explainer)]、[接口描述语言(IDL)]、[二进制格式(binary format)]和[应用二进制接口(ABI)]，涵盖了初始的最简可行产品（MVP）版本。

未来，此仓库还将包含[正式规范]、参考解释器和测试套件。

## 里程碑

目前，组件模型作为[WASI Preview 2]的一部分逐步开发并稳定。后续的"Preview 3"里程碑将主要关注[异步支持(Async Support)]的添加。

## 贡献

所有组件模型的工作都是作为 [W3C WebAssembly 社区组] 的一部分进行的。要为这些仓库做出贡献，请参阅社区组的 [贡献指南]。

[Component Model Documentation]: https://component-model.bytecodealliance.org/
[目标]: design/high-level/Goals.md
[用例]: design/high-level/UseCases.md
[设计选择]: design/high-level/Choices.md
[常见问题解答(FAQ)]: design/high-level/FAQ.md
[汇编级解释器(assembly-level explainer)]: design/mvp/Explainer.md
[接口描述语言(IDL)]: design/mvp/WIT.md
[二进制格式(binary format)]: design/mvp/Binary.md
[应用二进制接口(ABI)]: design/mvp/CanonicalABI.md
[正式规范]: spec/
[W3C WebAssembly 社区组]: https://www.w3.org/community/webassembly/
[贡献指南]: https://webassembly.org/community/contributing/
[WASI Preview 2]: https://github.com/WebAssembly/WASI/tree/main/preview2
[异步支持(Async Support)]: https://docs.google.com/presentation/d/1MNVOZ8hdofO3tI0szg_i-Yoy0N2QPU2C--LzVuoGSlE/edit?usp=share_link
