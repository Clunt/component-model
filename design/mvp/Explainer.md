# 组件模型解释器

解释器本文介绍了组件的汇编级(assembly-level)定义，以及原生JavaScript运行时组件嵌入提案。如需面向用户的详细说明，请查看[**组件模型文档**][Component Model Documentation]。

* [特性封闭（Gated features）](#特性封闭gated-features)
* [语法](#语法)
  * [组件定义（Component Definitions）](#组件定义component-definitions)
    * [索引空间（Inedx Spaces）](#索引空间inedx-spaces)
  * [实例定义（Instance definitions）](#实例定义instance-definitions)
  * [别名定义（Alias definitions）](#别名定义alias-definitions)
  * [类型定义（Type definitions）](#类型定义type-definitions)
    * [基本值类型（Fundamental value types）](#基本值类型fundamental-value-types)
      * [数字类型（Numeric types）](#数字类型numeric-types)
      * [容器类型（Container types）](#容器类型container-types)
      * [句柄类型（Handle types）](#handle句柄类型-types)
    * [特殊值类型（Specialized value types）](#特殊值类型specialized-value-types)
    * [定义类型（Definition types）](#定义类型definition-types)
    * [声明符（Declarators）](#声明符declarators)
    * [类型检查（Type Checking）](#类型检查type-checking)
  * [规范定义（Canonical Definitions）](#规范定义canonical-definitions)
    * [规范 ABI（Canonical ABI）](#规范-ABIcanonical-built-ins)
    * [规范内置（Canonical built-ins）](#规范内置canonical-built-ins)
  * [值定义（Value definitions）](#值定义value-definitions)
  * [启动定义（Start Definitions）](#开始定义start-definitions)
  * [导入和导出定义（Import and export definitions）](#导入和导出定义import-and-export-definitions)
* [组件不变性（Component invariants）](#组件不变性component-invariants)
* [JavaScript嵌入（JavaScript embedding）](#JavaScript嵌入JavaScript-embedding)
  * [JS API](#JS-API)
  * [ESM集成（ESM-integration）](#ESM集成ESM-integration)
* [示例（Examples）](#示例examples)
* [TODO](#TODO)

## 特性封闭（Gated features）

默认情况下，本解释器中描述的功能（以及支持的[Binary.md](Binary.md)、[WIT.md](WIT.md)和[CanonicalABI.md](CanonicalABI.md)）已实现并包含在[WASI Preview 2]稳定性里程碑中。不属于 Preview 2 的功能由以下列出的表情符号之一划定；这些表情符号将在实现、被视为稳定并包含在未来的里程碑中后被删除：
* 🪙: 值导入/导出(imports/exports)和组件级启动函数(component-level start function)
* 🪺: 导入/导出名称中的嵌套命名空间和包(nested namespaces and packages)
* 🧵: 线程内置

（基于之前向 WebAssembly CG 提出的范围和[分层提案][scoping and layering]，此仓库合入并取代了[模块链接(module-linking)][module-linking]和[接口类型(interface-types)][interface-types]提案，将它们的一些原始功能推送到MVP后续代办的[未来功能](FutureFeatures.md)中。

## 语法

本节使用[EBNF语法]定义组件，该语法解析介于纯粹抽象语法树（如Core WebAssembly规范的[结构部分][Structure Section]）和完整文本格式（如Core WebAssembly规范的[文本格式部分][Text Format Section]之间的内容。目标是平衡完整性和简洁性，只需提供足够的细节来编写示例并以[二进制格式][Binary Format Section]部分的样式定义[二进制格式(binary format)](Binary.md) ，详细严谨的语法将推迟到[正式规范](../../spec/)。

语法简化的主要方法是定义的使用，在实际文本格式中使用标识符（`<id>`）显示地引用定义`X`的索引（写作`<Xidx>`），在解析时检查标识符是否解析为`X`定义，然后将解析后的索引嵌入到AST中。

此外，假定了下面未明确定义的Core WebAssembly文本格式定义的标准[缩写][abbreviations]（例如，内联导出定义*inline export definitions*）。

[EBNF语法]: https://zh.wikipedia.org/wiki/%E6%89%A9%E5%B1%95%E5%B7%B4%E7%A7%91%E6%96%AF%E8%8C%83%E5%BC%8F

### 组件定义（Component Definitions）

顶层`component`是各种类型定义的序列：
```ebnf
component  ::= (component <id>? <definition>*)
definition ::= core-prefix(<core:module>)
             | core-prefix(<core:instance>)
             | core-prefix(<core:type>)
             | <component>
             | <instance>
             | <alias>
             | <type>
             | <canon>
             | <start> 🪺
             | <import>
             | <export>
             | <value> 🪙

其中，当 X 解析为 '(' Y ')' 时，core-prefix(X) 解析为 '(' 'core' Y ')'
```
组件类似于Core WebAssembly模块，其包含的定义是无环的：定义只能引用先前的定义（在AST、文本格式和二进制格式中）。但与模块不同的是，组件可以任意交错不同类型的定义。

元函数(meta-function)`core-prefix`把用于解析Core WebAssembly定义的语法规则进行相同的转换，但会在最左侧`(`后添加`core`标志。例如，当`core:module`代表`(module (func))`时，则`core-prefix(<core:module>)`为`(core module (func))`。请注意，内部的`func`不需要`core`前缀；前缀`core`标识用于表示解析从组件定义*过渡*为核心定义。

组件模型未修改[`core:module`]项目，因此组件嵌入了当前标准的Core WebAssembly（文本和二进制格式）模块，允许复用未经修改的Core WebAssembly实现。目前Core WebAssembly不包含`core:instance`项目，但当Core WebAssembly采纳[模块链接(module-linking）][module-linking]提案后则会涵盖。下面将介绍新的核心定义及对应的组件级部分。最后，现有的[`core:type`]项目按模块链接提案扩展增加核心模块类型。因此，总体思路是将核心定义（在AST、二进制和文本格式中）作为已添加至Core WebAssembly中，因此最终可以分层共享解码和验证的实现。

接下来的定义类型是组件自身的递归定义，由此形成的组件树中所有其他类型定义只出现在叶子节点上。例如，基于目前定义的内容，我们可以编写如下组件：
```wasm
(component
  (component
    (core module (func (export "one") (result i32) (i32.const 1)))
    (core module (func (export "two") (result f32) (f32.const 2)))
  )
  (core module (func (export "three") (result i64) (i64.const 3)))
  (component
    (component
      (core module (func (export "four") (result f64) (f64.const 4)))
    )
  )
  (component)
)
```

上述顶层组件构成了一棵树，叶子节点包含4个模块和1个组件。然而，由于没有任何`instance`定义（后续介绍），运行时不会实例化或执行任何内容；它们均是无用代码(dead code)。

#### 索引空间（Inedx Spaces）

类似于[Core WebAssembly][Core Indices]，组件模型将每个`definition`放入一组固定的*索引空间*，从而允许后续定义（在文本和二进制格式中）通过非负整数*索引*引用该定义。在定义、验证和执行组件时，有5个组件级索引空间（component-level index spaces）：
* (component) functions
* (component) values
* (component) types
* component instances
* components

WebAssembly 1.0也存在5个核心索引空间：
* (core) functions
* (core) tables
* (core) memories
* (core) globals
* (core) types

以及2个额外的核心索引空间，其中包含由组件模型引入的核心定义，但WebAssembly 1.0中未定义（然而：[模块链接(module-linking)][module-linking]提案将会添加）：
* module instances
* modules

实现需要维护共12个索引空间，例如验证组件。此处12个索引空间与下方`类别`项目的终端符 1:1对应，因此 “类别(sort)” 和 “索引空间(index space)” 可以互换使用。

类似于[Core WebAssembly][Core Identifiers]，组件模型的文本格式允许使用*标识符*(*identifiers*)代替索引，这些标识符会被解析为AST的索引（在此基础上定义验证和执行）。因此，下面两个组件等价：
```wasm
(component
  (core module (; empty ;))
  (component   (; empty ;))
  (core module (; empty ;))
  (export "C" (component 0))
  (export "M1" (core module 0))
  (export "M2" (core module 1))
)
```
```wasm
(component
  (core module $M1 (; empty ;))
  (component $C    (; empty ;))
  (core module $M2 (; empty ;))
  (export "C" (component $C))
  (export "M1" (core module $M1))
  (export "M2" (core module $M2))
)
```


### 实例定义（Instance definitions）

鉴于模块和组件代表不可变的*代码*，实例(instance)将代码与潜在可变的*状态*（potentially-mutable state，例如线性内存）相关联，因此在*运行*代码前必须创建实例。实例定义通过选择一个模块或组件，并提供一组命名的*参数(arguments)*来满足所选模块或组件的所有命名*导入(imports)*，从而创建模块或组件实例。

定义core:instance实例的语法为：
```ebnf
core:instance       ::= (instance <id>? <core:instancexpr>)
core:instanceexpr   ::= (instantiate <core:moduleidx> <core:instantiatearg>*)
                      | <core:inlineexport>*
core:instantiatearg ::= (with <core:name> (instance <core:instanceidx>))
                      | (with <core:name> (instance <core:inlineexport>*))
core:sortidx        ::= (<core:sort> <u32>)
core:sort           ::= func
                      | table
                      | memory
                      | global
                      | type
                      | module
                      | instance
core:inlineexport   ::= (export <core:name> <core:sortidx>)
```
当通过`instantiate`实例化模块时，核心模块的双层导入(two-level imports)按如下方式解析：
1. 导入的`core:name`的第一项（如：`import "a" "one"`中的`"a"`），通过在`core:instantiatearg`的命名列表中查找以选择核心模块实例(core module instance)。（未来，当core wasm增加单层导入(single-level imports)时支持其他的`core:sort`导入）
2. 导入的`core:name`的第二项（如：`import "a" "one"`中的`"one"`），通过在上述选择的核心模块实例查找导出(export)以确定导入的核心定义(core definition)

每个`core:sort`都与包含*该类*核心定义的独立[索引空间][index space]1:1匹配。`core:sortidx`中的`u32`字段用于选择一个其对应索引空间的定义。

在此基础上，我们可以将两个核心模块$A和$B以下组件链接在一起：
```wasm
(component
  (core module $A
    (func (export "one") (result i32) (i32.const 1))
  )
  (core module $B
    (func (import "a" "one") (result i32))
  )
  (core instance $a (instantiate $A))
  (core instance $b (instantiate $B (with "a" (instance $a))))
)
```
查看其他类别的案例，我们需要下一节介绍的`alias`定义。

`core:instanceexpr`的`<core:inlineexport>*`形式允许把先前定义组合在一起直接创建模块实例，无需`instantiate`辅助模块。`core:instantiatearg`的`<core:inlineexport>*`形式为语法糖，在文本解析时扩展通过`with`引用外部实例定义。为了展示这些示例，我们依然需要下一节介绍的`alias`定义。

定义组件实例的语法与核心模块实例保持对称，但是在组件级别扩展了`sort`定义：
```ebnf
instance       ::= (instance <id>? <instanceexpr>)
instanceexpr   ::= (instantiate <componentidx> <instantiatearg>*)
                 | <inlineexport>*
instantiatearg ::= (with <name> <sortidx>)
                 | (with <name> (instance <inlineexport>*))
name           ::= <core:name>
sortidx        ::= (<sort> <u32>)
sort           ::= core <core:sort>
                 | func
                 | value 🪙
                 | type
                 | component
                 | instance
inlineexport   ::= (export <exportname> <sortidx>)
```
由于组件级(component-level)的函数、类型和实例与核心级(core-level)有所区别，它们被放置在不同的索引空间中并进行单独索引。组件可以导入和导出各种核心定义（需与[无共享(shared-nothing)][shared-nothing]模型兼容，当前仅适用于`module`，未来可能包含`data`）。因此，组件级`sort`将完整的`core:sort`集合注入其中用于引用（由验证规则丢弃上下文不允许的核心类别）。

`name`复用Core WebAssembly的`core:name`引号字符串字面量语法（出现于核心模块的导入和导出且可包含任何有效UTF-8字符串）。

🪙 sort的`value`指的是实例化过程中提供和消耗的制。其工作方式在[值定义](#值定义value-definitions)部分详细介绍。

在介绍组件实例化的复杂示例之前，我们需引入几个其他定义，它们允许组件导入、定义和导出组件函数。


### 别名定义（Alias Definitions）

别名定义项将其他组件索引空间中的定义映射到当前组件索引空间中。如下述的AST所示，别名有三种"目标(targets)"：组件实例的导出`export`、核心模块实例的核心导出`core export`以及外部组件(`outer` component)定义（包含当前组件）：
```ebnf
alias            ::= (alias <aliastarget> (<sort> <id>?))
aliastarget      ::= export <instanceidx> <name>
                   | core export <core:instanceidx> <core:name>
                   | outer <u32> <u32>
```
如果存在别名，其`id`会关联所添加的新索引，然后可使用于任何`id`所在的地方。

关于`export`别名，校验保证`name`是目标实例的导出且匹配类别(sort)。

关于`outer`别名，`u32`对作为[德布鲁因索引(de Bruijn index)][de Bruijn index]，第一个`u32`是需跳过的封装组件/模块的数量，第二个`u32`是目标类别索引空间的索引。尤其，第一个`u32`允许为`0`，此时外部别名(outer alias)引用当前组件。为了保持模块实例化的无环性，外部别名只被允许指向*先前*的外部定义。

包含外部别名(outer aliases)的组件在实例化时，实际上会产生一个[闭包][closure]，包含外部别名定义的副本。因为普遍假设组件是不可变的值，所以仅限于引用不可变的定义：非资源类型、模块和组件。（未来，可以通过某种"`stateful`"类型属性记录结果组件(resulting component)在其类型中的状态，从而允许所有类别的定义支持外部别名）

这两种别名都具备内联隐式声明的语法糖：

对于`export`别名，内联语法糖扩展了`sortidx`的定义和各种类别明确(sort-specific)的索引：
```ebnf
sortidx     ::= (<sort> <u32>)          ;; as above
              | <inlinealias>
Xidx        ::= <u32>                   ;; as above
              | <inlinealias>
inlinealias ::= (<sort> <u32> <name>+)
```
如果`<sort>`引用了`<core:sort>`，那么`inlinealias`的`<u32>`则是`<core:instanceidx>`；否则是`<instanceidx>`。例如，下面的代码片段使用了两个内联函数别名：
```wasm
(instance $j (instantiate $J (with "f" (func $i "f"))))
(export "x" (func $j "g" "h"))
```
脱糖后为：
```wasm
(alias export $i "f" (func $f_alias))
(instance $j (instantiate $J (with "f" (func $f_alias))))
(alias export $j "g" (instance $g_alias))
(alias export $g_alias "h" (func $h_alias))
(export "x" (func $h_alias))
```

对于`outer`别名，内联语法糖仅为外部定义的标识符，使用正常的词法作用域规则解析。例如，以下组件：
```wasm
(component
  (component $C ...)
  (component
    (instance (instantiate $C))
  )
)
```
脱糖后为：
```wasm
(component $Parent
  (component $C ...)
  (component
    (alias outer $Parent $C (component $Parent_C))
    (instance (instantiate $Parent_C))
  )
)
```

最后，为了与[imports][func-import-abbrev]对称，别名可以颠倒书写顺序，把类别放到前面：
```wasm
    (func $f (import "i" "f") ...type...) ≡ (import "i" "f" (func $f ...type...))   (WebAssembly 1.0)
          (func $f (alias export $i "f")) ≡ (alias export $i "f" (func $f))
   (core module $m (alias export $i "m")) ≡ (alias export $i "m" (core module $m))
(core func $f (alias core export $i "f")) ≡ (alias core export $i "f" (core func $f))
```

通过目前的定义，我们能够以任意重命名的方式链接模块：
```wasm
(component
  (core module $A
    (func (export "one") (result i32) (i32.const 1))
    (func (export "two") (result i32) (i32.const 2))
    (func (export "three") (result i32) (i32.const 3))
  )
  (core module $B
    (func (import "a" "one") (result i32))
  )
  (core instance $a (instantiate $A))
  (core instance $b1 (instantiate $B
    (with "a" (instance $a))                      ;; 未重命名
  ))
  (core func $a_two (alias core export $a "two")) ;; ≡ (alias core export $a "two" (core func $a_two))
  (core instance $b2 (instantiate $B
    (with "a" (instance
      (export "one" (func $a_two))                ;; 重命名，使用界外(out-of-line)别名
    ))
  ))
  (core instance $b3 (instantiate $B
    (with "a" (instance
      (export "one" (func $a "three"))            ;; 重命名，使用<inlinealias>
    ))
  ))
)
```
为了展示链接组件的类似示例，我们需要组件级类型和函数定义，这将在接下来的两节中介绍。


### 类型定义（Type Definitions）

定义核心类型语法基于现有进行了扩展，增加`module`类型构造器：
```ebnf
core:rectype     ::= ... 来自Core WebAssembly规范
core:typedef     ::= ... 来自Core WebAssembly规范
core:subtype     ::= ... 来自Core WebAssembly规范
core:comptype    ::= ... 来自Core WebAssembly规范
                   | <core:moduletype>
core:moduletype  ::= (module <core:moduledecl>*)
core:moduledecl  ::= <core:importdecl>
                   | <core:type>
                   | <core:alias>
                   | <core:exportdecl>
core:alias       ::= (alias <core:aliastarget> (<core:sort> <id>?))
core:aliastarget ::= outer <u32> <u32>
core:importdecl  ::= (import <core:name> <core:name> <core:importdesc>)
core:exportdecl  ::= (export <core:name> <core:exportdesc>)
core:exportdesc  ::= strip-id(<core:importdesc>)

当 X 解析为 '(' sort <id>? Y ')'，则 strip-id(X) 解析为 '(' sort Y ')'
```

此处，[GC]提案中定义的`core:comptype`（复合类型"composite type"的简称）扩展了`module`类型构造器。GC提案还在核心wasm类型中增加了递归和显式的子类型。由于它们有不同的需求和预期使用方式，模块类型支持隐式自类型话且非递归。以鸟巢，现有的核心验证规则需要模块类型的声明超类为空并禁止递归使用模块类型。

在MVP中，验证规则会丢弃`core:moduletype`定义或`core:moduletype`的别名，因为在模块链接之前，核心模块不能自己导入或导出其他核心模块。

模块类型主体包含有序的“模块声明符(module declarators)”列表，它们在类型级别描述了模块的导入和导出。在模块类型上下文中，导入和导出声明符都可以在WebAssembly 1.0定义的[`core:importdesc`]中复用，唯一区别是在文本格式中`core:importdesc`可以绑定一个标识符在后续复用，但`core:exportdesc`不可以。

随着Core WebAssembly的[类型导入(type-imports)][type-imports]，模块类型将需要根据导入类型的能力来定义导出类型。为此，模块类型以空类型索引空间开始，该空间由`type`声明符填充。以便未来这些`type`声明符可以引用模块类型自身的本地类型导入。例如，未来以下模块类型将可表达为：
```wasm
(component $C
  (core type $M (module
    (import "" "T" (type $T))
    (type $PairT (struct (field (ref $T)) (field (ref $T))))
    (export "make_pair" (func (param (ref $T)) (result (ref $PairT))))
  ))
)
```
在此示例中，`$M`具有不同于`$C`的类型索引空间，元素0是导入类型，元素1是`struct`类型，元素2是隐式创建的引用了前两个元素的`func`类型。

最后，`core:alias`模块声明符允许模块类型定义通过`outer` `type`别名在闭合组件核心的类型索引空间中复用（而不是重新定义）类型定义。MVP中，验证限制`core:alias`模块声明*仅*允许`outer` `type`别名（闭合组件或组件类型的核心类型索引空间）。未来，更多类型的别名将有意义并被允许。

举例来说，下面的组件定义了两个语义等价的模块类型，前者通过`type`声明符定义函数类型，后者通过`alias`声明符引用函数类型。
```wasm
(component $C
  (core type $C1 (module
    (type (func (param i32) (result i32)))
    (import "a" "b" (func (type 0)))
    (export "c" (func (type 0)))
  ))
  (core type $F (func (param i32) (result i32)))
  (core type $C2 (module
    (alias outer $C $F (type))
    (import "a" "b" (func (type 0)))
    (export "c" (func (type 0)))
  ))
)
```

组件级类型定义同核心级类型定义对称，但使用一组完全不同的值类型。不同于假设共享线性内存来传递复合值的低级类型[`core:valtype`]，组件级值类型假设没有共享内存，因此其必须高级别的，能描述全部的复合值。
```ebnf
type          ::= (type <id>? <deftype>)
deftype       ::= <defvaltype>
                | <resourcetype>
                | <functype>
                | <componenttype>
                | <instancetype>
defvaltype    ::= bool
                | s8 | u8 | s16 | u16 | s32 | u32 | s64 | u64
                | f32 | f64
                | char | string
                | (record (field "<label>" <valtype>)+)
                | (variant (case "<label>" <valtype>?)+)
                | (list <valtype>)
                | (tuple <valtype>+)
                | (flags "<label>"+)
                | (enum "<label>"+)
                | (option <valtype>)
                | (result <valtype>? (error <valtype>)?)
                | (own <typeidx>)
                | (borrow <typeidx>)
valtype       ::= <typeidx>
                | <defvaltype>
resourcetype  ::= (resource (rep i32) (dtor <funcidx>)?)
functype      ::= (func <paramlist> <resultlist>)
paramlist     ::= (param "<label>" <valtype>)*
resultlist    ::= (result "<label>" <valtype>)*
                | (result <valtype>)
componenttype ::= (component <componentdecl>*)
instancetype  ::= (instance <instancedecl>*)
componentdecl ::= <importdecl>
                | <instancedecl>
instancedecl  ::= core-prefix(<core:type>)
                | <type>
                | <alias>
                | <exportdecl>
                | <value> 🪙
importdecl    ::= (import <importname> bind-id(<externdesc>))
exportdecl    ::= (export <exportname> bind-id(<externdesc>))
externdesc    ::= (<sort> (type <u32>) )
                | core-prefix(<core:moduletype>)
                | <functype>
                | <componenttype>
                | <instancetype>
                | (value <valuebound>) 🪙
                | (type <typebound>)
typebound     ::= (eq <typeidx>)
                | (sub resource)
valuebound    ::= (eq <valueidx>) 🪙
                | <valtype> 🪙

当 X 解析为 '(' sort Y ')'， bind-id(X) 解析为 '(' sort <id>? Y ')'
```
因为这种类型语法中没有类似于[gc]提案的[`rectype`]，所以这些类型都是非递归的。

#### 基本值类型（Fundamental value types）

值类型在`valtype`可被分为两类：*基本（fundamental）*值类型和*特殊（specialized）*值类型，其中特殊值类型定义由扩展基本值类型而来。*基本值类型*包括以下几组抽象值：
| Type                      | Values |
| ------------------------- | ------ |
| `bool`                    | `true` 和 `false` |
| `s8`, `s16`, `s32`, `s64` | [-2<sup>N-1</sup>, 2<sup>N-1</sup>-1]范围内的整数 |
| `u8`, `u16`, `u32`, `u64` | [0, 2<sup>N</sup>-1]范围内的整数 |
| `f32`, `f64`              | [IEEE754] 浮点数，只有一个 NaN 值 |
| `char`                    | [Unicode标量值（Unicode Scalar Values）][Unicode Scalar Values] |
| `record`                  | 命名值的异构[元组（tuples）][tuples] |
| `variant`                 | 命名值的异构[标签联合（tagged unions）][tagged unions] |
| `list`                    | 相同类型、长度可变的[值序列（sequences）][sequences] |
| `own`                     | 唯一、地址不透明的资源，当此值被丢弃时该资源将被销毁 |
| `borrow`                  | 地址不透明的资源，当前导出调用返回之前该资源必须被丢弃 |

组件如何通过*canonical lifting and lowering definitions*配置Core WebAssembly值和线性内存生产和使用这些抽象值，将在[下面](#canonical-definitions)介绍。例如，尽管抽象`变量（variant）`包含按名称标记的`实例（case）`列表，但canonical lifting and lowering会将每个实例映射到一个从`0`开始的`i32`值。

##### 数字类型（Numeric types）

虽然核心数字类型根据一组位模式(bit-patterns)和解释位操作的多种方式定义，但组件级数字类型根据值集合定义。其允许在使用不同值表示的源语言和协议之间转换值。

核心整数类型仅是不区分正负符号的位模式，而组件级整数类型则是包含或不包含负值的整数集。核心浮点数类型有多种不同的NaN位模式，而组件级浮点数类型仅有一个NaN值。在核心wasm中，布尔值通常表示为`i32`，其所有零(all-zeros)运算解释为`false`，而组件级具有含`true`和`false`的`bool`类型。

##### 容器类型（Container types）

`record`、`variant`和`list`类型允许对包含的值进行分组、分类和排序。

##### 句柄类型（Handle types）

`own`和`borrow`值类型均为*句柄类型*。句柄在逻辑上包含资源的不透明地址，避免在跨组件边界传递时复制资源。通过操作系统来比喻，句柄类似于文件描述符，它们存储在表中并只能通过其表中的整数索引被不受信任的用户模式进程间接地使用。

在组件模型中，句柄是从封装的每个组件实例(per-component-instance)*句柄表*中lifted-from和lowered-into的`i32`值，该表由[下面](#canonical-definitions)介绍的规范函数定义所维护。未来，句柄可以向后兼容引用类型的lifted和lowered（通过增加新的`canonopt`，[如下](#canonical-abi)介绍）。

上述的唯一性和丢弃条件在运行时通过这些规范定义由组件模型强制执行。句柄类型目前的`typeidx`必须引用`resource`类型（如下所述），该类型静态地分类句柄可以指向特定种类的资源。

#### 特殊值类型（Specialized value types）

其余*特殊值类型*允许的值集由以下映射定义：
The sets of values allowed for the remaining *specialized value types* are
defined by the following mapping:
```
                    (tuple <valtype>*) ↦ (record (field "𝒊" <valtype>)*) for 𝒊=0,1,...
                    (flags "<label>"*) ↦ (record (field "<label>" bool)*)
                     (enum "<label>"+) ↦ (variant (case "<label>")+)
                    (option <valtype>) ↦ (variant (case "none") (case "some" <valtype>))
(result <valtype>? (error <valtype>)?) ↦ (variant (case "ok" <valtype>?) (case "error" <valtype>?))
                                string ↦ (list char)
```

特殊值类型具有与其对应的非专门类型相同的语义值集，但具有不同的类型构造器（这些构造器于非专门的类型构造器不同），因此它们具有不同的二进制编码。这使得特殊值类型可以传达更具体的意图。例如，`result`不仅是变量variant，它还表示成功或失败的*含义*，因此源码绑定可以通过源语言的错误报告来显示它。此外，有时能以不同的方式表示值。例如，规范ABI(Canonical ABI)的`string`使用各种Unicode编码，而`list<char>`使用4字节的`char`代码序列。同样，规范ABI中的`flags`使用位向量(bit-vector)，而等效的布尔字段使用布尔值字节序列。

请注意，至少在初期，变量需要有非空项列表。将来可能会放开限制允许空项列表，其中空`(variant)`实际上作为一个[空类型(empty type)][empty type]且表示不可达。


#### 定义类型（Definition types）

`deftype`的剩余的4种类型构造器使用`valtype`描述无共享函数、资源、组件和组件实例：

`func`类型构造器描述接收并返回`valtype`列表的组件级函数定义。与[`core:functype`]相反，`functype`的参数和返回值可以关联需校验唯一的名称。为了提升常见的单值函数(single-value-returning functions)的易用性和性能，函数类型可能还有一个单独的未命名的返回类型。对于这种特殊情况，建议绑定生成器直接返回单个值，而不要将其包装在record/object/struct之中

`resource`类型构造器为包含组件的每个实例创建新(fresh)类型（"freshness"及其与一般类型检查的交互在[下面](#type-checking)详细描述）。资源类型可以被句柄类型（如`own`和`borrow`）以及[下面](#canonical-built-ins)描述的规范哪件类型引用。`resource`类型目前的`rep`指定其*核心表示类型(core representation type)*，目前固定为`i32`，但未来会放宽（至少包括`i64`，但也可能包括其他类型）。当最后一个指向资源的句柄被丢弃时，将调用`dtor`当前指定的资源的析构函数（如果存在），允许实现组件执行清理，如释放线性内存分配。

`instance`类型构造器描述了可以由组件导入或导出命名的、类型化的定义列表。通俗地说，实例类型(instance types)对应于“接口(interface)”的常见概念，因此实例类型作为静态接口描述。除了在此定义中的S-Expression文本格式外（旨在放入组件定义中），接口还可以使用[接口定义语言(Interface Definition Language)][Interface Definition Language][`wit`](WIT.md)定义为独立的、人类友好的文本文件 the [`wit`](WIT.md)


`component`类型构造器于核心`module`类型构造器对称，包含*两个*命名定义列表，分别用于组件的导入和导出。如上所述，实例(instance)类型可以在组件(component)类型的导入和导出*两者*中出现。

`instance`和`component`类型构造器都是由四种类型&mdash;`type`、`alias`、`import` 和 `export`&mdash;“声明符(declarators)”序列构成，其中只有`component`类型构造器可以包含`import`声明符。这些声明符的含义与上面介绍的核心模块声明符基本相同，但扩展至覆盖组件模型的额外功能。

#### 声明符（Declarators）

`importdecl`和`exportdecl`声明符分别对应组件的`import`和`export`定义，允许绑定标识符被后续声明符使用。`label`、`importname`和`exportname`定义在下面的[导入和导出](#import-and-export-definitions)部分给出。
按照[`core:typeuse`]的先例，文本格式允许引用界外类型定义（通过`(type <typeidx>)`）和内联类型表达式，问呗格式将其脱糖为界外类型定义。

🪙 `externdesc`的`value`项描述了在下面的[值定义](#value-definitions)章节描述的实例化时导入或导出的运行时值。

`externdesc`的`type`项描述了导入或导出的类型及其"绑定(bound)"：

`sub`绑定声明导入/导出类型为其他*子类型*的*抽象类型(abstract type)*，目前，唯一支持绑定的是`resource`（按照[GC]提案的命名约定），其表示“任何资源(resource)类型”。因此，只用资源类型可以被抽象的导入/导出，而不是任意值类型。这允许类型导入总是可以独立于它们的参数使用句柄值的“通用表示”（即 `i32`，由[规范ABI](CanonicalABI.md)定义）进行编译。未来，`sub`可能会扩展以允许引用其他资源类型，从而允许抽象资源子类型。

`eq`绑定说明导入/导出的类型必须与某个先前的类型定义在结构上相等。其允许：
* 导入的抽象类型被重新导出;
* 组件为前置的抽象类型引入另一个标签（当实现具有相同资源的多个独立接口时，这可能是必要的）；
* 组件将透明类型别名(transparent type aliases)附加到结构类型(structural types)上，以反映在源级别的绑定中（例如，`(export "bytes" (type (eq (list u64))))`可以在C++中生成`typedef std::vector<uint64_t> bytes`或在JS中生成一个名为`bytes`的导出字段，该字段别名`Uint64Array`。

放宽上述`core:alias`声明符的限制，`alias`声明符允许区分`type`和`instance`的别名`outer`和`export`。这允许后续类型声明符使用`instance`类型的导入和导出声明符导出的类型：
```wasm
(component
  (import "fancy-fs" (instance $fancy-fs
    (export $fs "fs" (instance
      (export "file" (type (sub resource)))
      ;; ...
    ))
    (alias export $fs "file" (type $file))
    (export "fancy-op" (func (param "f" (borrow $file))))
  ))
)
```

`type`声明符受到验证限制不允许`resource`类型定义，从而防止组件类型中出现“私有(private)”资源类型定义并避开[预防问题(avoidance problem)][avoidance problem]。因此，只有通过`importdecl`或`exportdecl`引入的资源类型才可能出现在`instancetype`或`componenttype`中。

到目前为止的定义，我们可以使用混合类型定义来定义组件类型：
```wasm
(component $C
  (type $T (list (tuple string bool)))
  (type $U (option $T))
  (type $G (func (param "x" (list $T)) (result $U)))
  (type $D (component
    (alias outer $C $T (type $C_T))
    (type $L (list $C_T))
    (import "f" (func (param "x" $L) (result (list u8))))
    (import "g" (func (type $G)))
    (export "g2" (func (type $G)))
    (export "h" (func (result $U)))
    (import "T" (type $T (sub resource)))
    (import "i" (func (param "x" (list (own $T)))))
    (export "T2" (type $T' (eq $T)))
    (export "U" (type $U' (sub resource)))
    (export "j" (func (param "x" (borrow $T')) (result (own $U'))))
  ))
)
```
注意`$G`和`$U`的内联使用是`outer`别名的语法糖。

#### 类型检查（Type Checking）

类似于核心模块，组件在前期验证阶段会检查组件定义确保基本的一致性。类型检查是验证的核心部分，例如，验证实例化（[`instantiate`](#instance-definitions)）表达式`with`参数与被实例化组件的`import`是否类型兼容时会进行类型检查。

为了逐步描述类型检查是如何工作的，我们将从非资源、非句柄、本地类型定义的*类型等价性*开始并逐步加强。

几乎所有类型（除了下面描述的）的类型等价性都是纯粹的*结构性的(structural)*。在结构性设定中，类型被认为是抽象语法树，其节点是类型构造器，像`u8`和`string`被认为是出现在叶子上的"零元"类型构造器，而像`list`和`record`这样的非零元类型构造器出现在父节点上。然后，类型等价性被定义为AST等价性。重要的是，这些类型AST*不*包含任何类型索引或依赖于索引空间布局；这些二进制格式的细节被解码以构成AST。例如，在以下的复合组件中：
```wasm
(component $A
  (type $ListString1 (list string))
  (type $ListListString1 (list $ListString1))
  (type $ListListString2 (list $ListString1))
  (component $B
    (type $ListString2 (list string))
    (type $ListListString3 (list $ListString2))
    (type $ListString3 (alias outer $A $ListString1))
    (type $ListListString4 (list $ListString3))
    (type $ListListString5 (alias outer $A $ListListString1))
  )
)
```
所有 5 种变体`$ListListStringX`都被视为相等，因为解码后，它们都具有相同的AST。

接下来，AST的类型等价关系被放宽为更灵活的[子类型][subtyping]关系。当前，仅有`instance`和`component`类型的子类型被放宽，但将来可能会为更多的类型构造器放宽子类关系，以便更好地支持API演进(API Evolution)（注意理解子类型在各种源语言中如何表现，以便子类型兼容的更新不会无意中破坏源级客户端）。

组件和实例子类型允许子类型导出比超类型(supertype)声明的更多内容并导入更少内容，忽略导入和导出的确切顺序，仅考虑名称。例如下方，`$I1`是`$I2`的子类型：
```wat
(component
  (type $I1 (instance
    (export "foo" (func))
    (export "bar" (func))
    (export "baz" (func))
  ))
  (type $I2 (instance
    (export "bar" (func))
    (export "foo" (func))
  ))
)
```
并且`$C1`是`$C2`的子类型：
```wat
(component
  (type $C1 (component
    (import "a" (func))
    (export "x" (func))
    (export "y" (func))
  ))
  (type $C2 (component
    (import "a" (func))
    (import "b" (func))
    (export "x" (func))
  ))
)
```

当我们接下来考虑导入和导出类型时，`typebound`有两个不同的子情况需要考虑：`eq`和`sub`。

`eq`绑定增加了类型相等规则（扩展上面提到的内置子类型规则集），表示导入类型在结构上等同与边界中引用的类型。例如，在组件中：
```wasm
(component
  (type $L1 (list u8))
  (import "L2" (type $L2 (eq $L1)))
  (import "L3" (type $L2 (eq $L1)))
  (import "L4" (type $L2 (eq $L3)))
)
```
所有4种`$L*`类型都相等（从子类型的角度看，它们均是彼此的子类型）。

相反，`sub`绑定引入一种新的*抽象*类型，组件的其余部分必须谨慎的假设该抽象类型可以是绑定的子类型的*任何*类型。这对于类型检查而言，每个子类型绑定的类型导入/导出都会引入一种*新*的类型抽象，该抽象类型与每个先前的类型定义都不相等。
目前（且可能在MVP中），仅有`resource`支持类型绑定（这意味着“任何资源类型”），因此唯一的抽象类型是抽象`resource`类型。例如，在下面的组件中：
```wasm
(component
  (import "T1" (type $T1 (sub resource)))
  (import "T2" (type $T2 (sub resource)))
)
```
`$T1`和`$T2`类型不相等。

一旦导入了类型，它就可以被后续的等价绑定类型导入引用，从而添加更多它等价的类型。例如，下方组件：
```wasm
(component $C
  (import "T1" (type $T1 (sub resource)))
  (import "T2" (type $T2 (sub resource)))
  (import "T3" (type $T3 (eq $T2)))
  (type $ListT1 (list (own $T1)))
  (type $ListT2 (list (own $T2)))
  (type $ListT3 (list (own $T3)))
)
```
`$T2`和`$T3`类型彼此相等但不等于`$T1`。根据上述传递结构相等规则，`$List2`和`$List3`彼此相等但不等于`$List1`。

句柄类型（`own`和`borrow`）是结构化类型（类似于`list`），但它们引用资源类型，因此可以传递性地“继承(inherit)”抽象资源类型的新鲜度(freshness)。例如，下方组件：
```wasm
(component
  (import "T" (type $T (sub resource)))
  (import "U" (type $U (sub resource)))
  (type $Own1 (own $T))
  (type $Own2 (own $T))
  (type $Own3 (own $U))
  (type $ListOwn1 (list $Own1))
  (type $ListOwn2 (list $Own2))
  (type $ListOwn3 (list $Own3))
  (type $Borrow1 (borrow $T))
  (type $Borrow2 (borrow $T))
  (type $Borrow3 (borrow $U))
  (type $ListBorrow1 (list $Borrow1))
  (type $ListBorrow2 (list $Borrow2))
  (type $ListBorrow3 (list $Borrow3))
)
```
`$Own1`和`$Own2`类型彼此相等但不等于`$Own3`或任何`$Borrow*`。相同的，`$Borrow1`和`$Borrow2`彼此相等但不等于`$Borrow3`。传递性地，`$ListOwn1`和`$ListOwn2`类型彼此相等但不等于`$ListOwn3`或任何`$ListBorrow*`。类型导入的类型检查规则反映了[通用类型(universal types)(∀T)][universal types]的*引入*规则。

上述示例均展示了*导入*方面的抽象类型，但当为另一个组件设置*导出*别名时，同样的“新鲜度(freshness)”条件也适用。例如，下方组件：
```wasm
(component
  (import "C" (component $C
    (export "T1" (type (sub resource)))
    (export "T2" (type $T2 (sub resource)))
    (export "T3" (type (eq $T2)))
  ))
  (instance $c (instantiate $C))
  (alias export $c "T1" (type $T1))
  (alias export $c "T2" (type $T2))
  (alias export $c "T3" (type $T3))
)
```
`$T2`和`$T3`类型彼此相等单不等于`$T1`。这些针对类型导出别名的类型检查规则反映了[存在类型(existential types)(∃T)][existential types]的*消除*规则。

接下来，我们讨论抽象类型的第三个来源是资源类型*定义*。不同于导入和导出引入的抽象类型，资源类型定义为设置和获取资源的私有表示值（将在[下面](#canonical-built-ins)介绍）提供了规范的内置功能。这些内建功能必然被限制在生成资源类型的组件实例中，从而隐藏了对资源类型表示的外部访问。因为每个组件实例都生成了与同一组件的所有先前实例不同的新资源类型，所以资源类型是["生成性的(generative)"]["Generative"]。

例如，在下方示例组件中：
```wasm
(component
  (type $R1 (resource (rep i32)))
  (type $R2 (resource (rep i32)))
  (func $f1 (result (own $R1)) (canon lift ...))
  (func $f2 (param (own $R2)) (canon lift ...))
)
```
`$R1`和`$R2`类型不相等，因此返回类型`$f1`与参数类型`$f2`不兼容。

资源类型定义的生成性与上面提到的类型导出的抽象类型规则相匹配，该规则强制组件的所有客户端绑定一个新的抽象类型。例如，在下方组件中：
```wasm
(component
  (component $C
    (type $r1 (export "r1") (resource (rep i32)))
    (type $r2 (export "r2") (resource (rep i32)))
  )
  (instance $c1 (instantiate $C))
  (instance $c2 (instantiate $C))
  (type $c1r1 (alias export $c1 "r1"))
  (type $c1r2 (alias export $c1 "r2"))
  (type $c2r1 (alias export $c2 "r1"))
  (type $c2r2 (alias export $c2 "r2"))
)
```
外部组件中的所有4种类型别名都不相等，反映出每个`$C`的实例生成两种新的资源类型的事实。

如果一个资源类型定义被导出多次，那么第一次之后的导出将等价绑定到第一次导出。例如，以下组件：
```wasm
(component
  (type $r (resource (rep i32)))
  (export "r1" (type $r))
  (export "r2" (type $r))
)
```
被分配了下面的`componenttype`：
```wasm
(component
  (export "r1" (type $r1 (sub resource)))
  (export "r2" (type (eq $r1)))
)
```
因此，从外部角度来看，`r1`和`r2`是同一类型的两个标签。

如果一个组件想要避免这个实现并强制客户端假设`r1`和`r2`为不同类型（从而允许实现在未来实际使用不同的类型而不破坏客户端），可以为导出指定明确的次严格的`sub`绑定替换`eq`绑定类型（使用[下方](#import-and-export-definitions)介绍的语法）。
```wasm
(component
  (type $r (resource (rep i32)))
  (export "r1" (type $r))
  (export "r2" (type $r) (type (sub resource)))
)
```
该组件分配如下`componenttype`：
```wasm
(component
  (export "r1" (type (sub resource)))
  (export "r2" (type (sub resource)))
)
```
该类型对上述组件的分配反映了[存在类型(existential types)(∃T)][existential types]的*引入*规则。

当通过`instantiate`为导入类型提供资源类型（导入*或*定义）时，类型检查会指向替换，将实例化组件中的所有使用`import`通过`with`替换为实际类型。例如，下方组件验证：
```wasm
(component $P
  (import "C1" (component $C1
    (import "T" (type $T (sub resource)))
    (export "foo" (func (param (own $T))))
  ))
  (import "C2" (component $C2
    (import "T" (type $T (sub resource)))
    (import "foo" (func (param (own $T))))
  ))
  (type $R (resource (rep i32)))
  (instance $c1 (instantiate $C1 (with "T" (type $R))))
  (alias export $c1 "foo" (func $foo))
  (instance $c2 (instantiate $C2 (with "T" (type $R)) (with "foo" (func $foo))))
)
```
这主要取决于在验证`$c1`和`$c2`的实例化时，`$C1`和`$C2`是否已被替换。这些用于实例化类型导入的类型检查规则反映了[通用类型(∀T)][universal types]的*消除*规则。

重要的是，这种由父级进行的类型替换在验证或运行时对子级不可见。特别是，没有运行时强制转换可以“看透”原始类型参数，从而避免了常见的[动态强制转换类型暴露问题(type-exposure problems with dynamic casts)][non-parametric parametricity]。

总结：所有类型构造器都是*结构化(structural)*的，除了`resource`，他是`抽象(abstract)`和`生成性(generative)`的。具有子类型绑定的类型导入和导出也引入了抽象类型，并遵循通用和存在类型的标准引入和消除规则。

最后，由于“名义上的(nominal)”通常被认为是“结构性的对立面(the opposite of structural)”，因此一个有效的问题是上述任何一种类型是否是“名义类型(nominal typing)”。在组件内部，资源类型表现为“名义上地”：每个资源类型定义为资源类型提供了不同于所有先前定义的资源类型的新的本地“名称”。有趣的情况是当从组件*外部*考虑资源类型等价性，，特别是当单个组件被多次实例化时。在这种情况下，使用单一`exportname`导出的单一资源类型定义将在每个组件实例都会得到一个新的类型，上面提到的抽象类型规则确保组件实例的每个资源类型保持不同。因此，从某种意义上说，资源类型的生成性*概括了*传统的基于名称的名义类型，提供了比共享全局命名空间可以实现的更细粒度的隔离。


### 规范定义（Canonical Definitions）

从运行在组件内部的Core WebAssembly角度来看，组件模型是一个[嵌入器][embedder]。因此，组件模型定义了传递给[`module_instantiate`]的Core WebAssembly导入和通过[`func_invoke`]调用的Core WebAssembly导出。这允许组件模型指定核心模块如何链接在一起（如上所示），但它还允许组件模型任意合成由Core WebAssembly导入的Core WebAssembly函数（通过[`func_alloc`]）。这些合成的核心函数是通过下面定义的几个*规范定义(canonical definitions)*之一创建的。

#### 规范 ABI（Canonical ABI）

要实现或调用一个组件级函数，我们需要跨越一个共享无关的边界。传统上，这个问题是通过定义一个序列化格式来解决的。组件模型MVP大致上使用了这种方法，定义了一个基于线性内存的[ABI]，成为“规范ABI(Canonical ABI)”，它为任何`functype`指定了一个[相应的(corresponding)](CanonicalABI.md#flattening)`core:functype`，以及将值从线性内存中复制进/出的[规则](CanonicalABI.md#lifting-and-lowering)。然而，组件模型与传统方法不同之处在于，BAI是可配置的，允许同一个抽象值有多种不同的内存表示。在MVP中，这种可配置型仅限于下面展示的小型`canonopt`集。然而，MVP后续，可以添加[适配器函数][adapter functions]以允许更多的程序控制。

规范ABI明确地应用于以两个方向之一“包装”现有的函数：
* `lift`包装一个核心函数（类型为`core:functype`），生成一个组件函数（类型为`functype`），可以传递给其他组件
* `lower`包装一个组件函数（类型为`functype`），生成一个核心函数（类型为`core:functype`），可以从当前组件内的Core WebAssembly代码导入和调用

规范定义指定这两个包装方向之一、要包装的函数和配置选项列表：
```ebnf
canon    ::= (canon lift core-prefix(<core:funcidx>) <canonopt>* bind-id(<externdesc>))
           | (canon lower <funcidx> <canonopt>* (core func <id>?))
canonopt ::= string-encoding=utf8
           | string-encoding=utf16
           | string-encoding=latin1+utf16
           | (memory <core:memidx>)
           | (realloc <core:funcidx>)
           | (post-return <core:funcidx>)
```
虽然`externdesc`接受任何`sort`，但`canon lift`的验证规则仅允许`func`类别。未来，可能会增加其他类别（即，类型），因此需要明确的类别。

`string-encoding`选项指定了Canonical ABI将如何对字符串类型进行编码。`latin1+utf16`编码能适应Java，JavaScript和.NET VMs的常见字符串编码方式，并允许在Latin-1（固定1字节编码，但码位有限）或UTF-16（可以表达所有码位，但每个码位占用2或4字节）之间动态选择。如果没有指定`string-encoding`，默认为UTF-8。同时指定多个字符串编码选项会校验错误。

`(memory ...)`选项指定了Canonical ABI将用于加载和存储值的内存。如果Canonical ABI需要加载或存储，校验需要此选项存在（无默认值）。

`(realloc ...)`选项指定了一个核心函数，该函数被校验需为下面的核心函数类型：
```wasm
(func (param $originalPtr i32)
      (param $originalSize i32)
      (param $alignment i32)
      (param $newSize i32)
      (result i32))
```
Canonical ABI将使用`realloc`进行内存分配（allocate，第一、二个参数为`0`）以及内存重新分配(reallocate)。如果Canonical ABI需要`realloc`，那么校验需要此选项存在（无默认值）。

`(post-return ...)`选项只能在`canon lift`中出现并指定一个核心函数，该函数将在读取完原始返回值后使用原始返回值进行调用，从而允许释放内存并调用析构函数。这个立即数是可选的，但是如果存在，则验证其参数是否与被调用者的返回类型匹配且结果为空。

基于AST的描述，[规范ABI解释器(Canonical ABI explainer)][Canonical ABI]给出了`lift`和`lower`的静态和动态语义的详细解析。

规范ABI解释器中给出的`canon lift`的动态语义的一个高层级结果是，组件函数语核心函数不同，所有的控制流转移都在其类型中明确反映。
例如，使用Core WebAssembly的[异常处理(exception-handling)][exception-handling]和[堆栈切换(stack-switching)][stack-switching]，类型为`(func (result i32))`的核心函数可以返回`i32`，抛出、暂停或捕获异常。相反，类型为`(func (result string))`的组件函数仅可能返回一个`string`或捕获异常。为了表达失败，组件函数可以返回`result`，具有异常处理的语言可以将异常绑定到`error`情况。类似的，即将添加的[future和stream类型][future and stream types]将在组件函数签名中明确声明堆栈切换的模式。

与上面显示的`import`和`alias`类似，`canon`定义也能以倒置形式编写，将类别放在第一位：
```wasm
(func $f (import "i" "f") ...type...) ≡ (import "i" "f" (func $f ...type...))       (WebAssembly 1.0)
(func $g ...type... (canon lift ...)) ≡ (canon lift ... (func $g ...type...))
(core func $h (canon lower ...))      ≡ (canon lower ... (core func $h))
```
注意：未来，`canon`可能会被推广到定义函数以外的其他类别（例如类型），因此需显示的`sort`。

使用规范的函数定义，我们最终可以熟悉一个不平凡的组件，它接收一个字符串，进行一些记录，然后返回一个字符串。
```wasm
(component
  (import "logging" (instance $logging
    (export "log" (func (param string)))
  ))
  (import "libc" (core module $Libc
    (export "mem" (memory 1))
    (export "realloc" (func (param i32 i32) (result i32)))
  ))
  (core instance $libc (instantiate $Libc))
  (core func $log (canon lower
    (func $logging "log")
    (memory (core memory $libc "mem")) (realloc (func $libc "realloc"))
  ))
  (core module $Main
    (import "libc" "memory" (memory 1))
    (import "libc" "realloc" (func (param i32 i32) (result i32)))
    (import "logging" "log" (func $log (param i32 i32)))
    (func (export "run") (param i32 i32) (result i32)
      ... (call $log) ...
    )
  )
  (core instance $main (instantiate $Main
    (with "libc" (instance $libc))
    (with "logging" (instance (export "log" (func $log))))
  ))
  (func $run (param string) (result string) (canon lift
    (core func $main "run")
    (memory (core memory $libc "mem")) (realloc (func $libc "realloc"))
  ))
  (export "run" (func $run))
)
```
此示例展示了从特定组件的不可复用模块（`$Main`）中分离出可复用的语言运行时模块（`$Libc`）。
除了减少代码大小和增加多组件场景中的代码共享之外，这种分离方式还允许`$libc`先被创建，这样它的导出就可以被`canon lower`引用。如果没有这种分离（也就是说`$Main`包含`memory`和分配函数），那么`canon lower`和`$Main`之间就会存在循环以来关系，必须使用辅助模块执行`call_indirect`打破这种依赖循环。

#### 规范内置（Canonical Built-ins）

除了适配*现有*函数的`lift`和`lower`的规范函数定义之外，还有一组规范“内置(built-ins)”，它们从无到有定义可以被核心模块导入的核心函数，从而与资源等规范ABI实体动态交互（以及当提案中添加了异步(async)、[任务(task)][Future and Stream Types]时）。
canon ::= ...
        | (canon resource.new <typeidx> (core func <id>?))
        | (canon resource.drop <typeidx> (core func <id>?))
        | (canon resource.rep <typeidx> (core func <id>?))
        | (canon thread.spawn <typeidx> (core func <id>?)) 🧵
        | (canon thread.hw_concurrency (core func <id>?)) 🧵
```

##### 资源（Resources）

内置`resource.new`具有`[i32] -> [i32]`类型并创建一个新的资源（具有资源类型`typeidx`），其表示为给定的`i32`值并返回指向此资源的新句柄的`i32`索引。

内置`resource.drop`具有`[i32] -> []`类型并删除给定`i32`索引的资源句柄（具有资源类型`typeidx`）。如果删除的句柄拥有资源，那么资源如果存在`dtor`则会被调用。

内置`resource.rep`具有`[i32] -> [i32]`类型并返回由给定`i32`索引处的句柄指向的资源（具有资源类型`typeidx`）的`i32`表示。

举个例子，以下组件导入了内置`resource.new`，使其能够创建并返回新资源给其客户端：
```wasm
(component
  (import "Libc" (core module $Libc ...))
  (core instance $libc (instantiate $Libc))
  (type $R (resource (rep i32) (dtor (func $libc "free"))))
  (core func $R_new (param i32) (result i32)
    (canon resource.new $R)
  )
  (core module $Main
    (import "canon" "R_new" (func $R_new (param i32) (result i32)))
    (func (export "make_R") (param ...) (result i32)
      (return (call $R_new ...))
    )
  )
  (core instance $main (instantiate $Main
    (with "canon" (instance (export "R_new" (func $R_new))))
  ))
  (export $R' "r" (type $R))
  (func (export "make-r") (param ...) (result (own $R'))
    (canon lift (core func $main "make_R"))
  )
)
```
这里，由`resource.new`返回的`i32`，是组件句柄表的索引，被`make_R`立即返回，从而将新创建资源的所有权转移给导出的调用者。

##### 🧵 线程（Threads）

提案[共享所有线程(shared-everything-threads)][shared-everything-threads]为线程管理增加了组件模型内置。这些被指定为内置而非核心WebAssembly指令的原因是浏览器希望这些功能由现有的Web/JS API提供。

内置`thread.spawn`具有`[f:(ref null $f) c:i32] -> [i32]`类型，它通过调用共享函数`f`并向其传递`c`来生成新线程，返回值表示是否成功创建了线程。

内置`resource.hw_concurrency`具有`[] -> [i32]`类型，它返回可以并发执行的线程数量。

请参阅[CanonicalABI.md](CanonicalABI.md#canonical-definitions)获取内置(built-ins)及其交互的详细定义。

### 🪙 值定义（Value Definitions）

值定义（在值索引空间中）类似于Core WebAssembly中的不可变`global`定义，只是验证要求它们在实例化时(instantiation-time)只被使用一次（即，它们是[线性的(linear)][linear]）。

组件可以使用以下语法在值索引空间中定义值：
```ebnf
value    ::= (value <id>? <valtype> <val>)
val      ::= false | true
           | <core:i64>
           | <f64canon>
           | nan
           | '<core:stringchar>'
           | <core:name>
           | (record <val>+)
           | (variant "<label>" <val>?)
           | (list <val>*)
           | (tuple <val>+)
           | (flags "<label>"*)
           | (enum "<label>")
           | none | (some <val>)
           | ok | (ok <val>) | error | (error <val>)
           | (binary <core:datastring>)
f64canon ::= <core:f64> without the `nan:0x` case.
```

`value`的校验规则要求`val`与`valtype`匹配。

`(binary ...)`表达式提供了一种替代语法，允许将值定义的二进制内容直接以文本格式写入，类似于数据段(data segments)，避免在编码或解码时需要理解类型信息。

例如：
```wasm
(component
  (value $a bool true)
  (value $b u8  1)
  (value $c u16 2)
  (value $d u32 3)
  (value $e u64 4)
  (value $f s8  5)
  (value $g s16 6)
  (value $h s32 7)
  (value $i s64 8)
  (value $j f32 9.1)
  (value $k f64 9.2)
  (value $l char 'a')
  (value $m string "hello")
  (value $n (record (field "a" bool) (field "b" u8)) (record true 1))
  (value $o (variant (case "a" bool) (case "b" u8)) (variant "b" 1))
  (value $p (list (result (option u8)))
    (list
      error
      (ok (some 1))
      (ok none)
      error
      (ok (some 2))
    )
  )
  (value $q (tuple u8 u16 u32) (tuple 1 2 3))

  (type $abc (flags "a" "b" "c"))
  (value $r $abc (flags "a" "c"))

  (value $s (enum "a" "b" "c") (enum "b"))

  (value $t bool (binary "\00"))
  (value $u string (binary "\07example"))

  (type $complex
    (tuple
      (record
        (field "a" (option string))
        (field "b" (tuple (option u8) string))
      )
      (list char)
      $abc
      string
    )
  )
  (value $complex1 (type $complex)
    (tuple
      (record
        none
        (tuple none "empty")
      )
      (list)
      (flags)
      ""
    )
  )
  (value $complex2 (type $complex)
    (tuple
      (record
        (some "example")
        (tuple (some 42) "hello")
      )
      (list 'a' 'b' 'c')
      (flags "b" "a")
      "hi"
    )
  )
)
```

与所有定义类别一样，值可以由组件导入和导出。以下是值导入的示例：
```wasm
(import "env" (value $env (record (field "locale" (option string)))))
```
正如该示例所示，值导入可以作为通用[环境变量][environment variables]，不仅允许`string`，还允许`valtype`的全部范围。

值也可以导出。例如：
```wasm
(component
  (import "system-port" (value $port u16))
  (value $url string "https://example.com")
  (export "default-url" (value $url))
  (export "default-port" (value $port))
)
```
该组件的推断类型是：
```wasm
(component
  (import "system-port" (value $port u16))
  (value $url string "https://example.com")
  (export "default-url" (value (eq $url)))
  (export "default-port" (value (eq $port)))
)
```
因此，默认情况下，导出的精确常量和导入会传递至组件类型从而成为公共接口。这样，值导出可以作为组件提供给主机或其他客户端工具的语义配置数据。组件可以使用后续[导入和导出](#import-and-export-definitions)提到的“类型归属(type ascription)”功能将导出的精确值保持为抽象（以便精确值不帅说类型和公共接口）。

### 🪙 启动定义（Start Definitions）

与模块一样，组件可以有在实例化期间调用的启动函数。与模块不同，组件可以在实例化期间的多个点调用启动函数，每个此类调用都有参数和结果。因此，组件中的`start`定义类似于函数调用：
```ebnf
start ::= (start <funcidx> (value <valueidx>)* (result (value <id>?))*)
```
`(value <valueidx>)*`列表通过索引到*值索引空间(value index space)*来指定传递给`funcidx`的参数。两个值列表的参数数量和类型都经过校验匹配`funcidx`的签名。

通过这个，我们可以定义一个组件，在实例化时导入一个字符串并计算一个新的导出的字符串：
```wasm
(component
  (import "name" (value $name string))
  (import "libc" (core module $Libc
    (export "memory" (memory 1))
    (export "realloc" (func (param i32 i32 i32 i32) (result i32)))
  ))
  (core instance $libc (instantiate $Libc))
  (core module $Main
    (import "libc" ...)
    (func (export "start") (param i32 i32) (result i32)
      ... general-purpose compute
    )
  )
  (core instance $main (instantiate $Main (with "libc" (instance $libc))))
  (func $start (param string) (result string) (canon lift
    (core func $main "start")
    (memory (core memory $libc "mem")) (realloc (func $libc "realloc"))
  ))
  (start $start (value $name) (result (value $greeting)))
  (export "greeting" (value $greeting))
)
```
如此例所示，启动函数重用了与正常导入和导出相同的规范ABI机制，将组件级值引入和导出核心线性内存。

### 导入和导出定义（Import and Export Definitions）

导入和导出定义都会将新元素附加到导入/导出`sort`的索引空间，该元素在文本格式中可以选择绑定一个标识符。对于导入，标识符的绑定类似于Core WebAssembly，作为`externdesc`的一部分（例如，`(import "x" (func $x))`绑定标识符`$x`）。
对于导出，紧接着`export`后面的`<id>?`会被绑定，而`<sortidx>`中的`<id>`则是对正在被导出的先前定义的引用（例如，`(export $x "x" (func $f))`绑定新的标识符`$x`）。
```ebnf
import ::= (import "<importname>" bind-id(<externdesc>))
export ::= (export <id>? "<exportname>" <sortidx> <externdesc>?)
```
所有的导入名称都必须是唯一的，所有的导出名称也必须是唯一的。导入和导出的其余语法定义了导入和导出名称内容的结构化语法。在语法上，这些名称出现在带引号的字符串字面量中。因此，语法限制了这些字符串字面量的内容，以提供更多的结构化信息，这些信息可以被工具链和运行时机械地解释，以支持习惯用的开发者工作流程和源语言绑定。下面定义此结构化名称语法的规则将被解释为定义单个标记的词法语法，因此不会自动插入空格，所有终端都用单引号引起来，并且所有未加引号的内容都是元字符。
```ebnf
exportname    ::= <plainname>
                | <interfacename>
importname    ::= <exportname>
                | <depname>
                | <urlname>
                | <hashname>
plainname     ::= <label>
                | '[constructor]' <label>
                | '[method]' <label> '.' <label>
                | '[static]' <label> '.' <label>
label         ::= <fragment>
                | <label> '-' <fragment>
fragment      ::= <word>
                | <acronym>
word          ::= [a-z] [0-9a-z]*
acronym       ::= [A-Z] [0-9A-Z]*
interfacename ::= <namespace> <label> <projection> <version>?
                | <namespace>+ <label> <projection>+ <version>? 🪺
namespace     ::= <words> ':'
words         ::= <word>
                | <words> '-' <word>
projection    ::= '/' <label>
version       ::= '@' <valid semver>
depname       ::= 'unlocked-dep=<' <pkgnamequery> '>'
                | 'locked-dep=<' <pkgname> '>' ( ',' <hashname> )?
pkgnamequery  ::= <pkgpath> <verrange>?
pkgname       ::= <pkgpath> <version>?
pkgpath       ::= <namespace> <words>
                | <namespace>+ <words> <projection>* 🪺
verrange      ::= '@*'
                | '@{' <verlower> '}'
                | '@{' <verupper> '}'
                | '@{' <verlower> ' ' <verupper> '}'
verlower      ::= '>=' <valid semver>
verupper      ::= '<' <valid semver>
urlname       ::= 'url=<' <nonbrackets> '>' (',' <hashname>)?
nonbrackets   ::= [^<>]*
hashname      ::= 'integrity=<' <integrity-metadata> '>'
```
组件提供了六种命名导入选项:
* **普通名称**，让开发人员“阅读文档”或以其他方式弄清楚要提供什么来导入；
* **接口名称**，假设它唯一地标识了组件正在请求一个*非特定的*wasm或本地实现的更高级的语义契约；
* **URL名称**，组件请求通过[获取(fetching)][fetching]URL来解析*特定的*wasm实现；
* **哈希名称（hash name）**，包含*特定的*wasm实现的字节的内容哈希(content-hash)，但不指定字节的位置；
* **锁定依赖项名称（locked dependency name）**，组件请求通过一些上下文提供的注册表解析到*特定的*wasm实现，使用给定的分层名称和版本；
* **未锁定依赖项名称（unlocked dependency name）**，组件请求通过一些上下文提供的注册表解析到*一组可能的*wasm实现*之一*，使用给定的分层名称和版本范围。

并非所有主机都应支持所有六个导入命名选项，并且通常，构建工具可能需要使用外部组件包装要部署的组件，该外部组件仅使用目标主机可以理解的导入名称。例如：
* 离线主机可能只实现一组固定的接口名称，需要构建工具来**捆绑**URL、依赖项和哈希名称（用嵌套定义替换导入）；
* 浏览器可能仅支持纯文本和URL名称（通过导入映射或JS API解析纯文本名称），需要构建过程发布或捆绑依赖项，将依赖项名称转换为嵌套定义或URL名称；
* 生产服务器环境可能只允许部署从一组固定的接口和锁定的依赖项名称导入的组件，从而要求事先锁定和部署所有依赖项；
* 没有直接开发人员界面（例如 JS API 或导入映射）的主机嵌入可能会拒绝所有普通名称，需要构建过程事先解决这些问题；
* 没有内容可寻址存储的主机可能会拒绝哈希名称（因为它们无法找到内容）。

URL名称的语法和验证允许嵌入的URL包函任何UTF-8字符序列（除了用于[分割URL][delimit the URL]的尖括号外），在[获取][fetching]URL的准备阶段，将URL的结构良好性作为[解析][parsing]URL过程的一部分进行检查。传递给URL规范解析算法的[基础URL][base URL]操作数由主机确定，并且可能因不存在导致不允许相对URL。因此，URL导入的解析和获取是主机定义的操作，发生在组件解码和校验之后，但在组件实例化之前。

When a particular implementation is indicated via URL or dependency name,
`importname` allows the component to additionally specify a cryptographic hash
of the expected binary representation of the wasm implementation, reusing the
[`integrity-metadata`] production defined by the W3C Subresource Integrity
specification. When this hash is present, a component can express its intention
to reuse another component or core module with the same degree of specificity
as if the component or core module was nested directly, thereby allowing
components to factor out common dependencies without compromising runtime
behavior. When *only* the hash is present (in a `hashname`), the host must
locate the contents using the hash (e.g., using an [OCI Registry]).

The "registry" referred to by dependency names serves to map a hierarchical
name and version to a particular module, component or exported definition. For
example, in the full generality of nested namespaces and packages (🪺), in a
registry name `a:b:c/d/e/f`, `a:b:c` traverses a path through namespaces `a`
and `b` to a component `c` and `/d/e/f` traverses the exports of `c` (where `d`
and `e` must be component exports but `f` can be anything). Given this abstract
definition, a number of concrete data sources can be interpreted by developer
tooling as "registries":
* a live registry (perhaps accessed via [`warg`])
* a local filesystem directory (perhaps containing vendored dependencies)
* a fixed set of host-provided functionality (see also the [built-in modules] proposal)
* a programmatically-created tree data structure (such as the `importObject`
  parameter of [`WebAssembly.instantiate()`])

The `valid semver` production is as defined by the [Semantic Versioning 2.0]
spec and is meant to be interpreted according to that specification. The
`verrange` production embeds a minimal subset of the syntax for version ranges
found in common package managers like `npm` and `cargo` and is meant to be
interpreted with the same [semantics][SemVerRange]. (Mostly this
interpretation is the usual SemVer-spec-defined ordering, but note the
particular behavior of pre-release tags.)

The `plainname` production captures several language-neutral syntactic hints
that allow bindings generators to produce more idiomatic bindings in their
target language. At the top-level, a `plainname` allows functions to be
annotated as being a constructor, method or static function of a preceding
resource. In each of these cases, the first `label` is the name of the resource
and the second `label` is the logical field name of the function. This
additional nesting information allows bindings generators to insert the
function into the nested scope of a class, abstract data type, object,
namespace, package, module or whatever resources get bound to. For example, a
function named `[method]C.foo` could be bound in C++ to a member function `foo`
in a class `C`. The JS API [below](#JS-API) describes how the native JavaScript
bindings could look. Validation described in [Binary.md](Binary.md) inspects
the contents of `plainname` and ensures that the function has a compatible
signature.

The `label` production used inside `plainname` as well as the labels of
`record` and `variant` types are required to have [kebab case]. The reason for
this particular form of casing is to unambiguously separate words and acronyms
(represented as all-caps words) so that source language bindings can convert a
`label` into the idiomatic casing of that language. (Indeed, because hyphens
are often invalid in identifiers, kebab case practically forces language
bindings to make such a conversion.) For example, the `label` `is-XML` could be
mapped to `isXML`, `IsXml`, `is_XML` or `is_xml`, depending on the target
language/convention. The highly-restricted character set ensures that
capitalization is trivial and does not require consulting Unicode tables.

Because some casing schemes (such as all-lowercase) would lead to clashes if
two `label`s differed only in case, in all cases where "uniquness" is required
between a set of names (viz., import/export names, record field labels, variant
case labels, and function parameter/result names), two `label`s that differ
only in case are considered equal and thus rejected.

Components provide two options for naming exports, symmetric to the first two
options for naming imports:
* a **plain name** that leaves it up to the developer to "read the docs"
  or otherwise figure out what the export does and how to use it; and
* an **interface name** that is assumed to uniquely identify a higher-level
  semantic contract that the component is claiming to implement with the
  given exported definition.

As an example, the following component uses all 9 cases of imports and exports:
```wasm
(component
  (import "custom-hook" (func (param string) (result string)))
  (import "wasi:http/handler" (instance
    (export "request" (type $request (sub resource)))
    (export "response" (type $response (sub resource)))
    (export "handle" (func (param (own $request)) (result (own $response))))
  ))
  (import "url=<https://mycdn.com/my-component.wasm>" (component ...))
  (import "url=<./other-component.wasm>,integrity=<sha256-X9ArH3k...>" (component ...))
  (import "locked-dep=<my-registry:sqlite@1.2.3>,integrity=<sha256-H8BRh8j...>" (component ...))
  (import "unlocked-dep=<my-registry:imagemagick@{>=1.0.0}>" (instance ...))
  (import "integrity=<sha256-Y3BsI4l...>" (component ...))
  ... impl
  (export "wasi:http/handler" (instance $http_handler_impl))
  (export "get-JSON" (func $get_json_impl))
)
```
Here, `custom-hook` and `get-JSON` are plain names for functions whose semantic
contract is particular to this component and not defined elsewhere. In
contrast, `wasi:http/handler` is the name of a separately-defined interface,
allowing the component to request the ability to make outgoing HTTP requests
(through imports) and receive incoming HTTP requests (through exports) in a way
that can be mechanically interpreted by hosts and tooling.

The remaining 4 imports show the different ways that a component can import
external implementations. Here, the URL and locked dependency imports use
`component` types, allowing this component to privately create and wire up
instances using `instance` definitions. In contrast, the unlocked dependency
import uses an `instance` type, anticipating a subsequent tooling step (likely
the one that performs dependency resolution) to select, instantiate and provide
the instance.

Validation of `export` requires that all transitive uses of resource types in
the types of exported functions or values refer to resources that were either
imported or exported (concretely, via the type index introduced by an `import`
or `export`). The optional `<externdesc>?` in `export` can be used to
explicitly ascribe a type to an export which is validated to be a supertype of
the definition's type, thereby allowing a private (non-exported) type
definition to be replaced with a public (exported) type definition.

For example, in the following component:
```wasm
(component
  (import "R1" (type $R1 (sub resource)))
  (type $R2 (resource (rep i32)))
  (export $R2' "R2" (type $R2))
  (func $f1 (result (own $R1)) (canon lift ...))
  (func $f2 (result (own $R2)) (canon lift ...))
  (func $f2' (result (own $R2')) (canon lift ...))
  (export "f1" (func $f1))
  ;; (export "f2" (func $f2)) -- invalid
  (export "f2" (func $f2) (func (result (own $R2'))))
  (export "f2" (func $f2'))
)
```
the commented-out `export` is invalid because its type transitively refers to
`$R2`, which is a private type definition. This requirement is meant to address
the standard [avoidance problem] that appears in module systems with abstract
types. In particular, it ensures that a client of a component is able to
externally define a type compatible with the exports of the component.

Similar to type exports, value exports may also ascribe a type to keep the precise
value from becoming part of the type and public interface.

For example:
```wasm
(component
  (value $url string "https://example.com")
  (export "default-url" (value $url) (value string))
)
```

The inferred type of this component is:
```wasm
(component
  (export "default-url" (value string))
)
```

Note, that the `url` value definition is absent from the component type

## Component Invariants

As a consequence of the shared-nothing design described above, all calls into
or out of a component instance necessarily transit through a component function
definition. Thus, component functions form a "membrane" around the collection
of core module instances contained by a component instance, allowing the
Component Model to establish invariants that increase optimizability and
composability in ways not otherwise possible in the shared-everything setting
of Core WebAssembly. The Component Model proposes establishing the following
three runtime invariants:
1. Components define a "lockdown" state that prevents continued execution
   after a trap. This both prevents continued execution with corrupt state and
   also allows more-aggressive compiler optimizations (e.g., store reordering).
   This was considered early in Core WebAssembly standardization but rejected
   due to the lack of clear trapping boundary. With components, each component
   instance is given a mutable "lockdown" state that is set upon trap and
   implicitly checked at every execution step by component functions. Thus,
   after a trap, it's no longer possible to observe the internal state of a
   component instance.
2. Components prevent unexpected reentrance by setting the "lockdown" state
   (in the previous bullet) whenever calling out through an import, clearing
   the lockdown state on return, thereby preventing reentrant export calls in
   the interim. This establishes a clear contract between separate components
   that both prevents obscure composition-time bugs and also enables
   more-efficient non-reentrant runtime glue code (particularly in the middle
   of the [Canonical ABI](CanonicalABI.md)). This implies that components by
   default don't allow concurrency and multi-threaded access will trap.


## JavaScript Embedding

### JS API

The [JS API] currently provides `WebAssembly.compile(Streaming)` which take
raw bytes from an `ArrayBuffer` or `Response` object and produces
`WebAssembly.Module` objects that represent decoded and validated modules. To
natively support the Component Model, the JS API would be extended to allow
these same JS API functions to accept component binaries and produce new
`WebAssembly.Component` objects that represent decoded and validated
components. The [binary format of components](Binary.md) is designed to allow
modules and components to be distinguished by the first 8 bytes of the binary
(splitting the 32-bit [`core:version`] field into a 16-bit `version` field and
a 16-bit `layer` field with `0` for modules and `1` for components).

Once compiled, a `WebAssembly.Component` could be instantiated using the
existing JS API `WebAssembly.instantiate(Streaming)`. Since components have the
same basic import/export structure as modules, this means extending the [*read
the imports*] logic to support single-level imports as well as imports of
modules, components and instances. Since the results of instantiating a
component is a record of JavaScript values, just like an instantiated module,
`WebAssembly.instantiate` would always produce a `WebAssembly.Instance` object
for both module and component arguments.

Types are a new sort of definition that are not ([yet][type-imports]) present
in Core WebAssembly and so the [*read the imports*] and [*create an exports
object*] steps need to be expanded to cover them:

For type exports, each type definition would export a JS constructor function.
This function would be callable iff a `[constructor]`-annotated function was
also exported. All `[method]`- and `[static]`-annotated functions would be
dynamically installed on the constructor's prototype chain. In the case of
re-exports and multiple exports of the same definition, the same constructor
function object would be exported (following the same rules as WebAssembly
Exported Functions today). In pathological cases (which, importantly, don't
concern the global namespace, but involve the same actual type definition being
imported and re-exported by multiple components), there can be collisions when
installing constructors, methods and statics on the same constructor function
object. In such cases, a conservative option is to undo the initial
installation and require all clients to instead use the full explicit names
as normal instance exports.

For type imports, the constructors created by type exports would naturally
be importable. Additionally, certain JS- and Web-defined objects that correspond
to types (e.g., the `RegExp` and `ArrayBuffer` constructors or any Web IDL
[interface object]) could be imported. The `ToWebAssemblyValue` checks on
handle values mentioned below can then be defined to perform the associated
[internal slot] type test, thereby providing static type guarantees for
outgoing handles that can avoid runtime dynamic type tests.

Lastly, when given a component binary, the compile-then-instantiate overloads
of `WebAssembly.instantiate(Streaming)` would inherit the compound behavior of
the abovementioned functions (again, using the `layer` field to eagerly
distinguish between modules and components).

For example, the following component:
```wasm
;; a.wasm
(component
  (import "one" (func))
  (import "two" (value string)) 🪙
  (import "three" (instance
    (export "four" (instance
      (export "five" (core module
        (import "six" "a" (func))
        (import "six" "b" (func))
      ))
    ))
  ))
  ...
)
```
and module:
```wasm
;; b.wasm
(module
  (import "six" "a" (func))
  (import "six" "b" (func))
  ...
)
```
could be successfully instantiated via:
```js
WebAssembly.instantiateStreaming(fetch('./a.wasm'), {
  one: () => (),
  two: "hi", 🪙
  three: {
    four: {
      five: await WebAssembly.compileStreaming(fetch('./b.wasm'))
    }
  }
});
```

The other significant addition to the JS API would be the expansion of the set
of WebAssembly types coerced to and from JavaScript values (by [`ToJSValue`]
and [`ToWebAssemblyValue`]) to include all of [`valtype`](#type-definitions).
At a high level, the additional coercions would be:

| Type | `ToJSValue` | `ToWebAssemblyValue` |
| ---- | ----------- | -------------------- |
| `bool` | `true` or `false` | `ToBoolean` |
| `s8`, `s16`, `s32` | as a Number value | `ToInt8`, `ToInt16`, `ToInt32` |
| `u8`, `u16`, `u32` | as a Number value | `ToUint8`, `ToUint16`, `ToUint32` |
| `s64` | as a BigInt value | `ToBigInt64` |
| `u64` | as a BigInt value | `ToBigUint64` |
| `f32`, `f64` | as a Number value | `ToNumber` |
| `char` | same as [`USVString`] | same as [`USVString`], throw if the USV length is not 1 |
| `record` | TBD: maybe a [JS Record]? | same as [`dictionary`] |
| `variant` | see below | see below |
| `list` | create a typed array copy for number types; otherwise produce a JS array (like [`sequence`]) | same as [`sequence`] |
| `string` | same as [`USVString`]  | same as [`USVString`] |
| `tuple` | TBD: maybe a [JS Tuple]? | TBD |
| `flags` | TBD: maybe a [JS Record]? | same as [`dictionary`] of optional `boolean` fields with default values of `false` |
| `enum` | same as [`enum`] | same as [`enum`] |
| `option` | same as [`T?`] | same as [`T?`] |
| `result` | same as `variant`, but coerce a top-level `error` return value to a thrown exception | same as `variant`, but coerce uncaught exceptions to top-level `error` return values |
| `own`, `borrow` | see below | see below |

Notes:
* Function parameter names are ignored since JavaScript doesn't have named
  parameters.
* If a function's result type list is empty, the JavaScript function returns
  `undefined`. If the result type list contains a single unnamed result, then
  the return value is specified by `ToJSValue` above. Otherwise, the function
  result is wrapped into a JS object whose field names are taken from the result
  names and whose field values are specified by `ToJSValue` above.
* In lieu of an existing standard JS representation for `variant`, the JS API
  would need to define its own custom binding built from objects. As a sketch,
  the JS values accepted by `(variant (case "a" u32) (case "b" string))` could
  include `{ tag: 'a', value: 42 }` and `{ tag: 'b', value: "hi" }`.
* For `option`, when Web IDL doesn't support particular type
  combinations (e.g., `(option (option u32))`), the JS API would fall back to
  the JS API of the unspecialized `variant` (e.g.,
  `(variant (case "some" (option u32)) (case "none"))`, despecializing only
  the problematic outer `option`).
* When coercing `ToWebAssemblyValue`, `own` and `borrow` handle types would
  dynamically guard that the incoming JS value's dynamic type was compatible
  with the imported resource type referenced by the handle type. For example,
  if a component contains `(import "Object" (type $Object (sub resource)))` and
  is instantiated with the JS `Object` constructor, then `(own $Object)` and
  `(borrow $Object)` could accept JS `object` values.
* When coercing `ToJSValue`, handle values would be wrapped with JS objects
  that are instances of the handles' resource type's exported constructor
  (described above). For `own` handles, a [`FinalizationRegistry`] would be
  used to drop the `own` handle (thereby calling the resource destructor) when
  its wrapper object was unreachable from JS. For `borrow` handles, the wrapper
  object would become dynamically invalid (throwing on any access) at the end
  of the export call.
* The forthcoming addition of [future and stream types] would allow `Promise`
  and `ReadableStream` values to be passed directly to and from components
  without requiring handles or callbacks.
* When an imported JavaScript function is a built-in function wrapping a Web
  IDL function, the specified behavior should allow the intermediate JavaScript
  call to be optimized away when the types are sufficiently compatible, falling
  back to a plain call through JavaScript when the types are incompatible or
  when the engine does not provide a separate optimized call path.


### ESM-integration

Like the JS API, [ESM-integration] can be extended to load components in all
the same places where modules can be loaded today, branching on the `layer`
field in the binary format to determine whether to decode as a module or a
component.

For URL import names, the embedded URL would be used as the [Module Specifier].
For plain names, the whole plain name would be used as the [Module Specifier]
(and an import map would be needed to map the string to a URL). For locked and
unlocked dependency names, ESM-integration would likely simply fail loading the
module, requiring a bundler to map these registry-relative names to URLs.

TODO: ESM-integration for interface imports and exports is still being
worked out in detail.

The main remaining question is how to deal with component imports having a
single string as well as the new importable component, module and instance
types. Going through these one by one:

For component imports of module type, we need a new way to request that the ESM
loader parse or decode a module without *also* instantiating that module.
Recognizing this same need from JavaScript, there is a TC39 proposal called
[Import Reflection] that adds the ability to write, in JavaScript:
```js
import Foo from "./foo.wasm" as "wasm-module";
assert(Foo instanceof WebAssembly.Module);
```
With this extension to JavaScript and the ESM loader, a component import
of module type can be treated the same as `import ... as "wasm-module"`.

Component imports of component type would work the same way as modules,
potentially replacing `"wasm-module"` with `"wasm-component"`.

In all other cases, the (single) string imported by a component is first
resolved to a [Module Record] using the same process as resolving the
[Module Specifier] of a JavaScript `import`. After this, the handling of the
imported Module Record is determined by the import type:

For imports of instance type, the ESM loader would treat the exports of the
instance type as if they were the [Named Imports] of a JavaScript `import`.
Thus, single-level imports of instance type act like the two-level imports
of Core WebAssembly modules where the first-level has been factored out. Since
the exports of an instance type can themselves be instance types, this process
must be performed recursively.

Otherwise, function or value imports are treated like an [Imported Default Binding]
and the Module Record is converted to its default value. This allows the following
component:
```wasm
;; bar.wasm
(component
  (import "./foo.js" (func (result string)))
  ...
)
```
to be satisfied by a JavaScript module via ESM-integration:
```js
// foo.js
export default () => "hi";
```
when `bar.wasm` is loaded as an ESM:
```html
<script src="bar.wasm" type="module"></script>
```


## Examples

For some use-case-focused, worked examples, see:
* [Link-time virtualization example](examples/LinkTimeVirtualization.md)
* [Shared-everything dynamic linking example](examples/SharedEverythingDynamicLinking.md)
* [Component Examples presentation](https://docs.google.com/presentation/d/11lY9GBghZJ5nCFrf4MKWVrecQude0xy_buE--tnO9kQ)


## TODO

The following features are needed to address the [MVP Use Cases](../high-level/UseCases.md)
and will be added over the coming months to complete the MVP proposal:
* concurrency support ([slides][Future And Stream Types])
* optional imports, definitions and exports (subsuming
  [WASI Optional Imports](https://github.com/WebAssembly/WASI/blob/main/legacy/optional-imports.md)
  and maybe [conditional-sections](https://github.com/WebAssembly/conditional-sections/issues/22))



[Structure Section]: https://webassembly.github.io/spec/core/syntax/index.html
[Text Format Section]: https://webassembly.github.io/spec/core/text/index.html
[Binary Format Section]: https://webassembly.github.io/spec/core/binary/index.html
[Core Indices]: https://webassembly.github.io/spec/core/syntax/modules.html#indices
[Core Identifiers]: https://webassembly.github.io/spec/core/text/values.html#text-id

[Index Space]: https://webassembly.github.io/spec/core/syntax/modules.html#indices
[Abbreviations]: https://webassembly.github.io/spec/core/text/conventions.html#abbreviations

[`core:i64`]: https://webassembly.github.io/spec/core/text/values.html#text-int
[`core:f64`]: https://webassembly.github.io/spec/core/syntax/values.html#floating-point
[`core:stringchar`]: https://webassembly.github.io/spec/core/text/values.html#text-string
[`core:name`]: https://webassembly.github.io/spec/core/syntax/values.html#syntax-name
[`core:module`]: https://webassembly.github.io/spec/core/text/modules.html#text-module
[`core:type`]: https://webassembly.github.io/spec/core/text/modules.html#types
[`core:importdesc`]: https://webassembly.github.io/spec/core/text/modules.html#text-importdesc
[`core:externtype`]: https://webassembly.github.io/spec/core/syntax/types.html#external-types
[`core:valtype`]: https://webassembly.github.io/spec/core/text/types.html#value-types
[`core:typeuse`]: https://webassembly.github.io/spec/core/text/modules.html#type-uses
[`core:functype`]: https://webassembly.github.io/spec/core/text/types.html#function-types
[`core:datastring`]: https://webassembly.github.io/spec/core/text/modules.html#text-datastring
[func-import-abbrev]: https://webassembly.github.io/spec/core/text/modules.html#text-func-abbrev
[`core:version`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-version

[Embedder]: https://webassembly.github.io/spec/core/appendix/embedding.html
[`module_instantiate`]: https://webassembly.github.io/spec/core/appendix/embedding.html#mathrm-module-instantiate-xref-exec-runtime-syntax-store-mathit-store-xref-syntax-modules-syntax-module-mathit-module-xref-exec-runtime-syntax-externval-mathit-externval-ast-xref-exec-runtime-syntax-store-mathit-store-xref-exec-runtime-syntax-moduleinst-mathit-moduleinst-xref-appendix-embedding-embed-error-mathit-error
[`func_invoke`]: https://webassembly.github.io/spec/core/appendix/embedding.html#mathrm-func-invoke-xref-exec-runtime-syntax-store-mathit-store-xref-exec-runtime-syntax-funcaddr-mathit-funcaddr-xref-exec-runtime-syntax-val-mathit-val-ast-xref-exec-runtime-syntax-store-mathit-store-xref-exec-runtime-syntax-val-mathit-val-ast-xref-appendix-embedding-embed-error-mathit-error
[`func_alloc`]: https://webassembly.github.io/spec/core/appendix/embedding.html#mathrm-func-alloc-xref-exec-runtime-syntax-store-mathit-store-xref-syntax-types-syntax-functype-mathit-functype-xref-exec-runtime-syntax-hostfunc-mathit-hostfunc-xref-exec-runtime-syntax-store-mathit-store-xref-exec-runtime-syntax-funcaddr-mathit-funcaddr

[`WebAssembly.instantiate()`]: https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/instantiate
[`FinalizationRegistry`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry
[Fetching]: https://fetch.spec.whatwg.org/
[Parsing]: https://url.spec.whatwg.org/#url-parsing
[Base URL]: https://url.spec.whatwg.org/#concept-base-url
[`integrity-metadata`]: https://www.w3.org/TR/SRI/#the-integrity-attribute
[Semantic Versioning 2.0]: https://semver.org/spec/v2.0.0.html
[Delimit The URL]: https://www.rfc-editor.org/rfc/rfc3986#appendix-C

[JS API]: https://webassembly.github.io/spec/js-api/index.html
[*read the imports*]: https://webassembly.github.io/spec/js-api/index.html#read-the-imports
[*create an exports object*]: https://webassembly.github.io/spec/js-api/index.html#create-an-exports-object
[Interface Object]: https://webidl.spec.whatwg.org/#interface-object
[`ToJSValue`]: https://webassembly.github.io/spec/js-api/index.html#tojsvalue
[`ToWebAssemblyValue`]: https://webassembly.github.io/spec/js-api/index.html#towebassemblyvalue
[`USVString`]: https://webidl.spec.whatwg.org/#es-USVString
[`sequence`]: https://webidl.spec.whatwg.org/#es-sequence
[`dictionary`]: https://webidl.spec.whatwg.org/#es-dictionary
[`enum`]: https://webidl.spec.whatwg.org/#es-enumeration
[`T?`]: https://webidl.spec.whatwg.org/#es-nullable-type
[`Get`]: https://tc39.es/ecma262/#sec-get-o-p
[Import Reflection]: https://github.com/tc39-transfer/proposal-import-reflection
[Module Record]: https://tc39.es/ecma262/#sec-abstract-module-records
[Module Specifier]: https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#prod-ModuleSpecifier
[Named Imports]: https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#prod-NamedImports
[Imported Default Binding]: https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#prod-ImportedDefaultBinding
[JS Tuple]: https://github.com/tc39/proposal-record-tuple
[JS Record]: https://github.com/tc39/proposal-record-tuple
[Internal Slot]: https://tc39.es/ecma262/#sec-object-internal-methods-and-internal-slots
[Built-in Modules]: https://github.com/tc39/proposal-built-in-modules

[Kebab Case]: https://en.wikipedia.org/wiki/Letter_case#Kebab_case
[De Bruijn Index]: https://en.wikipedia.org/wiki/De_Bruijn_index
[Closure]: https://en.wikipedia.org/wiki/Closure_(computer_programming)
[Empty Type]: https://en.wikipedia.org/w/index.php?title=Empty_type
[IEEE754]: https://en.wikipedia.org/wiki/IEEE_754
[Unicode Scalar Values]: https://unicode.org/glossary/#unicode_scalar_value
[Tuples]: https://en.wikipedia.org/wiki/Tuple
[Tagged Unions]: https://en.wikipedia.org/wiki/Tagged_union
[Sequences]: https://en.wikipedia.org/wiki/Sequence
[ABI]: https://en.wikipedia.org/wiki/Application_binary_interface
[Environment Variables]: https://en.wikipedia.org/wiki/Environment_variable
[Linear]: https://en.wikipedia.org/wiki/Substructural_type_system#Linear_type_systems
[Interface Definition Language]: https://en.wikipedia.org/wiki/Interface_description_language
[Subtyping]: https://en.wikipedia.org/wiki/Subtyping
[Universal Types]: https://en.wikipedia.org/wiki/System_F
[Existential Types]: https://en.wikipedia.org/wiki/System_F

[Generative]: https://www.researchgate.net/publication/2426300_A_Syntactic_Theory_of_Type_Generativity_and_Sharing
[Avoidance Problem]: https://counterexamples.org/avoidance.html
[Non-Parametric Parametricity]: https://people.mpi-sws.org/~dreyer/papers/npp/main.pdf

[module-linking]: https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Explainer.md
[interface-types]: https://github.com/WebAssembly/interface-types/blob/main/proposals/interface-types/Explainer.md
[type-imports]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md
[exception-handling]: https://github.com/WebAssembly/exception-handling/blob/main/proposals/exception-handling/Exceptions.md
[stack-switching]: https://github.com/WebAssembly/stack-switching/blob/main/proposals/stack-switching/Overview.md
[esm-integration]: https://github.com/WebAssembly/esm-integration/tree/main/proposals/esm-integration
[gc]: https://github.com/WebAssembly/gc/blob/main/proposals/gc/MVP.md
[`rectype`]: https://webassembly.github.io/gc/core/text/types.html#text-rectype
[shared-everything-threads]: https://github.com/WebAssembly/shared-everything-threads
[WASI Preview 2]: https://github.com/WebAssembly/WASI/tree/main/preview2

[Adapter Functions]: FutureFeatures.md#custom-abis-via-adapter-functions
[Canonical ABI]: CanonicalABI.md
[Shared-Nothing]: ../high-level/Choices.md
[Use Cases]: ../high-level/UseCases.md
[Host Embeddings]: ../high-level/UseCases.md#hosts-embedding-components

[Component Model Documentation]: https://component-model.bytecodealliance.org
[`wizer`]: https://github.com/bytecodealliance/wizer
[`warg`]: https://warg.io
[SemVerRange]: https://semver.npmjs.com/
[OCI Registry]: https://github.com/opencontainers/distribution-spec

[Scoping and Layering]: https://docs.google.com/presentation/d/1PSC3Q5oFsJEaYyV5lNJvVgh-SNxhySWUqZ6puyojMi8
[Future and Stream Types]: https://docs.google.com/presentation/d/1MNVOZ8hdofO3tI0szg_i-Yoy0N2QPU2C--LzVuoGSlE
