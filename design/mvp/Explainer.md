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
  * [Canonical definitions](#canonical-definitions)
    * [Canonical ABI](#canonical-built-ins)
    * [Canonical built-ins](#canonical-built-ins)
  * [Value definitions](#value-definitions)
  * [Start definitions](#start-definitions)
  * [Import and export definitions](#import-and-export-definitions)
* [Component invariants](#component-invariants)
* [JavaScript embedding](#JavaScript-embedding)
  * [JS API](#JS-API)
  * [ESM-integration](#ESM-integration)
* [Examples](#examples)
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

类似于核心模块，组件在前期验证阶段(up-front validation phase)检查组件定义以确保基本的一致性。类型检查是验证的核心部分，例如，验证实例化（[`instantiate`](#instance-definitions)）表达式`with`参数与被实例化组件的`import`是否类型兼容时会进行类型检查。

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

Next, the type equality relation on ASTs is relaxed to a more flexible
[subtyping] relation. Currently, subtyping is only relaxed for `instance` and
`component` types, but may be relaxed for more type constructors in the future
to better support API Evolution (being careful to understand how subtyping
manifests itself in the wide variety of source languages so that
subtype-compatible updates don't inadvertantly break source-level clients).

Component and instance subtyping allows a subtype to export more and import
less than is declared by the supertype, ignoring the exact order of imports and
exports and considering only names. For example, here, `$I1` is a subtype of
`$I2`:
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
and `$C1` is a subtype of `$C2`:
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

When we next consider type imports and exports, there are two distinct
subcases of `typebound` to consider: `eq` and `sub`.

The `eq` bound adds a type equality rule (extending the built-in set of
subtyping rules mentioned above) saying that the imported type is structurally
equivalent to the type referenced in the bound. For example, in the component:
```wasm
(component
  (type $L1 (list u8))
  (import "L2" (type $L2 (eq $L1)))
  (import "L3" (type $L2 (eq $L1)))
  (import "L4" (type $L2 (eq $L3)))
)
```
all four `$L*` types are equal (in subtyping terms, they are all subtypes of
each other).

In contrast, the `sub` bound introduces a new *abstract* type which the rest of
the component must conservatively assume can be *any* type that is a subtype of
the bound. What this means for type-checking is that each subtype-bound type
import/export introduces a *fresh* abstract type that is unequal to every
preceding type definition. Currently (and likely in the MVP), the only
supported type bound is `resource` (which means "any resource type") and thus
the only abstract types are abstract *resource* types. As an example, in the
following component:
```wasm
(component
  (import "T1" (type $T1 (sub resource)))
  (import "T2" (type $T2 (sub resource)))
)
```
the types `$T1` and `$T2` are not equal.

Once a type is imported, it can be referred to by subsequent equality-bound
type imports, thereby adding more types that it is equal to. For example, in
the following component:
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
the types `$T2` and `$T3` are equal to each other but not to `$T1`. By the
above transitive structural equality rules, the types `$List2` and `$List3` are
equal to each other but not to `$List1`.

Handle types (`own` and `borrow`) are structural types (like `list`) but, since
they refer to resource types, transitively "inherit" the freshness of abstract
resource types. For example, in the following component:
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
the types `$Own1` and `$Own2` are equal to each other but not to `$Own3` or
any of the `$Borrow*`.  Similarly, `$Borrow1` and `$Borrow2` are equal to
each other but not `$Borrow3`. Transitively, the types `$ListOwn1` and
`$ListOwn2` are equal to each other but not `$ListOwn3` or any of the
`$ListBorrow*`. These type-checking rules for type imports mirror the
*introduction* rule of [universal types]  (∀T).

The above examples all show abstract types in terms of *imports*, but the same
"freshness" condition applies when aliasing the *exports* of another component
as well. For example, in this component:
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
the types `$T2` and `$T3` are equal to each other but not to `$T1`. These
type-checking rules for aliases of type exports mirror the *elimination* rule
of [existential types]  (∃T).

Next, we consider resource type *definitions* which are a *third* source of
abstract types. Unlike the abstract types introduced by type imports and
exports, resource type definitions provide canonical built-ins for setting and
getting a resource's private representation value (that are introduced
[below](#canonical-built-ins)). These built-ins are necessarily scoped to the
component instance that generated the resource type, thereby hiding access to a
resource type's representation from the outside world. Because each component
instantiation generates fresh resource types distinct from all preceding
instances of the same component, resource types are ["generative"].

For example, in the following example component:
```wasm
(component
  (type $R1 (resource (rep i32)))
  (type $R2 (resource (rep i32)))
  (func $f1 (result (own $R1)) (canon lift ...))
  (func $f2 (param (own $R2)) (canon lift ...))
)
```
the types `$R1` and `$R2` are unequal and thus the return type of `$f1`
is incompatible with the parameter type of `$f2`.

The generativity of resource type definitions matches the abstract typing rules
of type exports mentioned above, which force all clients of the component to
bind a fresh abstract type. For example, in the following component:
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
all four types aliases in the outer component are unequal, reflecting the fact
that each instance of `$C` generates two fresh resource types.

If a single resource type definition is exported more than once, the exports
after the first are equality-bound to the first export. For example, the
following component:
```wasm
(component
  (type $r (resource (rep i32)))
  (export "r1" (type $r))
  (export "r2" (type $r))
)
```
is assigned the following `componenttype`:
```wasm
(component
  (export "r1" (type $r1 (sub resource)))
  (export "r2" (type (eq $r1)))
)
```
Thus, from an external perspective, `r1` and `r2` are two labels for the same
type.

If a component wants to hide this fact and force clients to assume `r1` and
`r2` are distinct types (thereby allowing the implementation to actually use
separate types in the future without breaking clients), an explicit type can be
ascribed to the export that replaces the `eq` bound with a less-precise `sub`
bound (using syntax introduced [below](#import-and-export-definitions)).
```wasm
(component
  (type $r (resource (rep i32)))
  (export "r1" (type $r))
  (export "r2" (type $r) (type (sub resource)))
)
```
This component is assigned the following `componenttype`:
```wasm
(component
  (export "r1" (type (sub resource)))
  (export "r2" (type (sub resource)))
)
```
The assignment of this type to the above component mirrors the *introduction*
rule of [existential types]  (∃T).

When supplying a resource type (imported *or* defined) to a type import via
`instantiate`, type checking performs a substitution, replacing all uses of the
`import` in the instantiated component with the actual type supplied via
`with`. For example, the following component validates:
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
This depends critically on the `T` imports of `$C1` and `$C2` having been
replaced by `$R` when validating the instantiations of `$c1` and `$c2`. These
type-checking rules for instantiating type imports mirror the *elimination*
rule of [universal types]  (∀T).

Importantly, this type substitution performed by the parent is not visible to
the child at validation- or run-time. In particular, there are no runtime
casts that can "see through" to the original type parameter, avoiding
avoiding the usual [type-exposure problems with dynamic casts][non-parametric parametricity].

In summary: all type constructors are *structural* with the exception of
`resource`, which is *abstract* and *generative*. Type imports and exports that
have a subtype bound also introduce abstract types and follow the standard
introduction and elimination rules of universal and existential types.

Lastly, since "nominal" is often taken to mean "the opposite of structural", a
valid question is whether any of the above is "nominal typing". Inside a
component, resource types act "nominally": each resource type definition
produces a new local "name" for a resource type that is distinct from all
preceding resource types. The interesting case is when resource type equality
is considered from *outside* the component, particularly when a single
component is instantiated multiple times. In this case, a single resource type
definition that is exported with a single `exportname` will get a fresh type
with each component instance, with the abstract typing rules mentioned above
ensuring that each of the component's instance's resource types are kept
distinct. Thus, in a sense, the generativity of resource types *generalizes*
traditional name-based nominal typing, providing a finer granularity of
isolation than otherwise achievable with a shared global namespace.


### Canonical Definitions

From the perspective of Core WebAssembly running inside a component, the
Component Model is an [embedder]. As such, the Component Model defines the
Core WebAssembly imports passed to [`module_instantiate`] and how Core
WebAssembly exports are called via [`func_invoke`]. This allows the Component
Model to specify how core modules are linked together (as shown above) but it
also allows the Component Model to arbitrarily synthesize Core WebAssembly
functions (via [`func_alloc`]) that are imported by Core WebAssembly. These
synthetic core functions are created via one of several *canonical definitions*
defined below.

#### Canonical ABI

To implement or call a component-level function, we need to cross a
shared-nothing boundary. Traditionally, this problem is solved by defining a
serialization format. The Component Model MVP uses roughly this same approach,
defining a linear-memory-based [ABI] called the "Canonical ABI" which
specifies, for any `functype`, a [corresponding](CanonicalABI.md#flattening)
`core:functype` and [rules](CanonicalABI.md#lifting-and-lowering) for copying
values into and out of linear memory. The Component Model differs from
traditional approaches, though, in that the ABI is configurable, allowing
multiple different memory representations of the same abstract value. In the
MVP, this configurability is limited to the small set of `canonopt` shown
below. However, Post-MVP, [adapter functions] could be added to allow far more
programmatic control.

The Canonical ABI is explicitly applied to "wrap" existing functions in one of
two directions:
* `lift` wraps a core function (of type `core:functype`) to produce a component
  function (of type `functype`) that can be passed to other components.
* `lower` wraps a component function (of type `functype`) to produce a core
  function (of type `core:functype`) that can be imported and called from Core
  WebAssembly code inside the current component.

Canonical definitions specify one of these two wrapping directions, the function
to wrap and a list of configuration options:
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
While the production `externdesc` accepts any `sort`, the validation rules
for `canon lift` would only allow the `func` sort. In the future, other sorts
may be added (viz., types), hence the explicit sort.

The `string-encoding` option specifies the encoding the Canonical ABI will use
for the `string` type. The `latin1+utf16` encoding captures a common string
encoding across Java, JavaScript and .NET VMs and allows a dynamic choice
between either Latin-1 (which has a fixed 1-byte encoding, but limited Code
Point range) or UTF-16 (which can express all Code Points, but uses either
2 or 4 bytes per Code Point). If no `string-encoding` option is specified, the
default is UTF-8. It is a validation error to include more than one
`string-encoding` option.

The `(memory ...)` option specifies the memory that the Canonical ABI will
use to load and store values. If the Canonical ABI needs to load or store,
validation requires this option to be present (there is no default).

The `(realloc ...)` option specifies a core function that is validated to
have the following core function type:
```wasm
(func (param $originalPtr i32)
      (param $originalSize i32)
      (param $alignment i32)
      (param $newSize i32)
      (result i32))
```
The Canonical ABI will use `realloc` both to allocate (passing `0` for the
first two parameters) and reallocate. If the Canonical ABI needs `realloc`,
validation requires this option to be present (there is no default).

The `(post-return ...)` option may only be present in `canon lift`
and specifies a core function to be called with the original return values
after they have finished being read, allowing memory to be deallocated and
destructors called. This immediate is always optional but, if present, is
validated to have parameters matching the callee's return type and empty
results.

Based on this description of the AST, the [Canonical ABI explainer][Canonical
ABI] gives a detailed walkthrough of the static and dynamic semantics of `lift`
and `lower`.

One high-level consequence of the dynamic semantics of `canon lift` given in
the Canonical ABI explainer is that component functions are different from core
functions in that all control flow transfer is explicitly reflected in their
type. For example, with Core WebAssembly [exception-handling] and
[stack-switching], a core function with type `(func (result i32))` can return
an `i32`, throw, suspend or trap. In contrast, a component function with type
`(func (result string))` may only return a `string` or trap. To express
failure, component functions can return `result` and languages with exception
handling can bind exceptions to the `error` case. Similarly, the forthcoming
addition of [future and stream types] would explicitly declare patterns of
stack-switching in component function signatures.

Similar to the `import` and `alias` abbreviations shown above, `canon`
definitions can also be written in an inverted form that puts the sort first:
```wasm
(func $f (import "i" "f") ...type...) ≡ (import "i" "f" (func $f ...type...))       (WebAssembly 1.0)
(func $g ...type... (canon lift ...)) ≡ (canon lift ... (func $g ...type...))
(core func $h (canon lower ...))      ≡ (canon lower ... (core func $h))
```
Note: in the future, `canon` may be generalized to define other sorts than
functions (such as types), hence the explicit `sort`.

Using canonical function definitions, we can finally write a non-trivial
component that takes a string, does some logging, then returns a string.
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
This example shows the pattern of splitting out a reusable language runtime
module (`$Libc`) from a component-specific, non-reusable module (`$Main`). In
addition to reducing code size and increasing code-sharing in multi-component
scenarios, this separation allows `$libc` to be created first, so that its
exports are available for reference by `canon lower`. Without this separation
(if `$Main` contained the `memory` and allocation functions), there would be a
cyclic dependency between `canon lower` and `$Main` that would have to be
broken using an auxiliary module performing `call_indirect`.

#### Canonical Built-ins

In addition to the `lift` and `lower` canonical function definitions which
adapt *existing* functions, there are also a set of canonical "built-ins" that
define core functions out of nothing that can be imported by core modules to
dynamically interact with Canonical ABI entities like resources (and, when
async is added to the proposal, [tasks][Future and Stream Types]).
```ebnf
canon ::= ...
        | (canon resource.new <typeidx> (core func <id>?))
        | (canon resource.drop <typeidx> (core func <id>?))
        | (canon resource.rep <typeidx> (core func <id>?))
        | (canon thread.spawn <typeidx> (core func <id>?)) 🧵
        | (canon thread.hw_concurrency (core func <id>?)) 🧵
```

##### Resources

The `resource.new` built-in has type `[i32] -> [i32]` and creates a new
resource (with resource type `typeidx`) with the given `i32` value as its
representation and returning the `i32` index of a new handle pointing to this
resource.

The `resource.drop` built-in has type `[i32] -> []` and drops a resource handle
(with resource type `typeidx`) at the given `i32` index. If the dropped handle
owns the resource, the resource's `dtor` is called, if present.

The `resource.rep` built-in has type `[i32] -> [i32]` and returns the `i32`
representation of the resource (with resource type `typeidx`) pointed to by the
handle at the given `i32` index.

As an example, the following component imports the `resource.new` built-in,
allowing it to create and return new resources to its client:
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
Here, the `i32` returned by `resource.new`, which is an index into the
component's handle-table, is immediately returned by `make_R`, thereby
transferring ownership of the newly-created resource to the export's caller.

##### 🧵 Threads

The [shared-everything-threads] proposal adds component model built-ins for
thread management. These are specified as built-ins and not core WebAssembly
instructions because browsers expect this functionality to come from existing
Web/JS APIs.

The `thread.spawn` built-in has type `[f:(ref null $f) c:i32] -> [i32]` and
spawns a new thread by invoking the shared function `f` while passing `c` to it,
returning whether a thread was successfully spawned.

The `resource.hw_concurrency` built-in has type `[] -> [i32]` and returns the
number of threads that can be expected to execute concurrently.

See the [CanonicalABI.md](CanonicalABI.md#canonical-definitions) for detailed
definitions of each of these built-ins and their interactions.

### 🪙 Value Definitions

Value definitions (in the value index space) are like immutable `global` definitions
in Core WebAssembly except that validation requires them to be consumed exactly
once at instantiation-time (i.e., they are [linear]).

Components may define values in the value index space using following syntax:

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

The validation rules for `value` require the `val` to match the `valtype`.

The `(binary ...)` expression form provides an alternative syntax allowing the binary contents
of the value definition to be written directly in the text format, analogous to data segments,
avoiding the need to understand type information when encoding or decoding.

For example:
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

As with all definition sorts, values may be imported and exported by
components. As an example value import:
```wasm
(import "env" (value $env (record (field "locale" (option string)))))
```
As this example suggests, value imports can serve as generalized [environment
variables], allowing not just `string`, but the full range of `valtype`.

Values can also be exported.  For example:
```wasm
(component
  (import "system-port" (value $port u16))
  (value $url string "https://example.com")
  (export "default-url" (value $url))
  (export "default-port" (value $port))
)
```
The inferred type of this component is:
```wasm
(component
  (import "system-port" (value $port u16))
  (value $url string "https://example.com")
  (export "default-url" (value (eq $url)))
  (export "default-port" (value (eq $port)))
)
```
Thus, by default, the precise constant or import being exported is propagated
into the component's type and thus its public interface.  In this way, value exports
can act as semantic configuration data provided by the component to the host
or other client tooling.
Components can also keep the exact value being exported abstract (so that the
precise value is not part of the type and public interface) using the "type ascription"
feature mentioned in the [imports and exports](#import-and-export-definitions) section below.

### 🪙 Start Definitions

Like modules, components can have start functions that are called during
instantiation. Unlike modules, components can call start functions at multiple
points during instantiation with each such call having parameters and results.
Thus, `start` definitions in components look like function calls:
```ebnf
start ::= (start <funcidx> (value <valueidx>)* (result (value <id>?))*)
```
The `(value <valueidx>)*` list specifies the arguments passed to `funcidx` by
indexing into the *value index space*. The arity and types of the two value lists are
validated to match the signature of `funcidx`.

With this, we can define a component that imports a string and computes a new
exported string at instantiation time:
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
As this example shows, start functions reuse the same Canonical ABI machinery
as normal imports and exports for getting component-level values into and out
of core linear memory.


### Import and Export Definitions

Both import and export definitions append a new element to the index space of
the imported/exported `sort` which can be optionally bound to an identifier in
the text format. In the case of imports, the identifier is bound just like Core
WebAssembly, as part of the `externdesc` (e.g., `(import "x" (func $x))` binds
the identifier `$x`). In the case of exports, the `<id>?` right after the
`export` is bound while the `<id>` inside the `<sortidx>` is a reference to the
preceding definition being exported (e.g., `(export $x "x" (func $f))` binds a
new identifier `$x`).
```ebnf
import ::= (import "<importname>" bind-id(<externdesc>))
export ::= (export <id>? "<exportname>" <sortidx> <externdesc>?)
```
All import names are required to be unique and all export names are required to
be unique. The rest of the grammar for imports and exports defines a structured
syntax for the contents of import and export names. Syntactically, these names
appear inside quoted string literals. The grammar thus restricts the contents
of these string literals to provide more structured information that can be
mechanically interpreted by toolchains and runtimes to support idiomatic
developer workflows and source-language bindings. The rules defining this
structured name syntax below are to be interpreted as a *lexical* grammar
defining a single token and thus whitespace is not automatically inserted, all
terminals are single-quoted, and everything unquoted is a meta-character.
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
Components provide six options for naming imports:
* a **plain name** that leaves it up to the developer to "read the docs"
  or otherwise figure out what to supply for the import;
* an **interface name** that is assumed to uniquely identify a higher-level
  semantic contract that the component is requesting an *unspecified* wasm
  or native implementation of;
* a **URL name** that the component is requesting be resolved to a *particular*
  wasm implementation by [fetching] the URL.
* a **hash name** containing a content-hash of the bytes of a *particular*
  wasm implemenentation but not specifying location of the bytes.
* a **locked dependency name** that the component is requesting be resolved via
  some contextually-supplied registry to a *particular* wasm implementation
  using the given hierarchical name and version; and
* an **unlocked dependency name** that the component is requesting be resolved
  via some contextually-supplied registry to *one of a set of possible* of wasm
  implementations using the given hierarchical name and version range.

Not all hosts are expected to support all six import naming options and, in
general, build tools may need to wrap a to-be-deployed component with an outer
component that only uses import names that are understood by the target host.
For example:
* an offline host may only implement a fixed set of interface names, requiring
  a build tool to **bundle** URL, dependency and hash names (replacing the
  imports with nested definitions);
* browsers may only support plain and URL names (with plain names resolved via
  import map or [JS API]), requiring the build process to publish or bundle
  dependencies, converting dependency names into nested definitions or URL
  names;
* a production server environment may only allow deployment of components
  importing from a fixed set of interface and locked dependency names, thereby
  requiring all dependencies to be locked and deployed beforehand;
* host embeddings without a direct developer interface (such as the JS API or
  import maps) may reject all plain names, requiring the build process to
  resolve these beforehand;
* hosts without content-addressable storage may reject hash names (as they have
  no way to locate the contents).

The grammar and validation of URL names allows the embedded URLs to contain any
sequence of UTF-8 characters (other than angle brackets, which are used to
[delimit the URL]), leaving the well-formedness of the URL to be checked as
part of the process of [parsing] the URL in preparation for [fetching] the URL.
The [base URL] operand passed to the URL spec's parsing algorithm is determined
by the host and may be absent, thereby disallowing relative URLs. Thus, the
parsing and fetching of a URL import are host-defined operations that happen
after the decoding and validation of a component, but before instantiation of
that component.

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
