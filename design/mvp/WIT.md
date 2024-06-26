# `wit`格式（The wit format）

Wasm接口类型（Wasm Interface Type, WIT）格式是一种[IDL]，它以两种主要方式为[WebAssembly组件模型][components]提供工具：

* WIT是一种开发人员友好的格式，用于描述组件的导入和导出。其易于阅读和编写，并为宾客语言(guest languages)生成组件及宿主语言(host languages)使用组件提供了基础。

* WIT包是组件生态系统中共享类型和定义的基础。作者可以在生成组件时从其他WIT包导入类型、发布一个表示为宿主嵌入的WIT包，或者在跨平台共享API集合的WIT定义上协作。

WIT包是在同一目录下以`wit`为扩展名的文件集合，其定义了[`interface`][interfaces]和[`world`][worlds]，例如`foo.wit`。文件编码为有效的utf-8字节。类型可以使用无限制的名称从包的接口(interface)导入，还可以使用限定的命名空间(namespace)和包(package)名从其他包导入。

本文档将介绍WIT文档句法结构的用途、非正式[语法规范][lexical-structure](pseudo-formal grammar specification)，以及适合分发的WIT包的[包格式(package format)][package-format]规范。

[IDL]: https://en.wikipedia.org/wiki/Interface_description_language
[components]: https://github.com/webassembly/component-model

## 包名（Package Names）

所有WIT包均分配一个*包名*。包名类似于`foo:bar@1.0.0`，包含三个字段：

* *命名空间字段（namespace field）*，例如`foo:bar`中的`foo`。此命名空间旨在消除注册表、顶级(top-level)组织等之间的歧义。例如，WASI接口使用`wasi`命名空间。

* *包字段（package field）*，例如`wasi:clocks`中的`clocks`。“包(package)”将一组接口(interface)和世界(world)聚合，否则将使用共同的前缀命名。

* *版本字段（version field）*[可选的], 定义为[full semver](https://semver.org/).

🪺 使用“嵌套命名空间和包”，包名称类似于`foo:bar:baz/quux`，其中`bar`是`foo`的嵌套命名空间、`quux`是`baz`的嵌套包。有关更多详细信息，请参阅[包声明][package-declaration]部分。

在WIT文件顶部通过`package`声明指定包名：

```wit
package wasi:clocks;
```

或者

```wit
package wasi:clocks@1.2.0;
```

WIT包可以由一组文件定义，且至少有一个文件须指定包名。多个文件可以指定`package`，但它们必须统一包名。

或者，如果使用“显示的(explicit)”包表示法，可以在一个或多个文件中连续声明多个包：

```wit
package local:a {
  interface foo {}
}

package local:b {
  interface bar {}
}
```

包名用于生成组件模型中表示[`接口(interface)`][interfaces]和[`世界(world)`]的[导入名和导出名]，具体描述[如下](#package-format)。

[导入名和导出名]: Explainer.md#import-and-export-definitions

## WIT接口（WIT Interfaces）
[interfaces]: #wit接口wit-interfaces

“接口(interface)”的概念是WIT的核心，它是[函数(function)][functions]和[类型(type)][types]的集合。接口可以被视为WebAssembly组件模型中的一个实例，例如，从宿主导入或由组件实现供宿主消费的功能单元。所有函数和类型都属于接口。

接口示例：

```wit
package local:demo;

interface host {
  log: func(msg: string);
}
```

表示一个名为`host`的接口，它提供一个函数`log`，该函数接受一个`string`参数。如果将其导入到组件中，则它将对应于：

```wasm
(component
  (import "local:demo/host" (instance $host
    (export "log" (func (param "msg" string)))
  ))
  ;; ...
)
```

一个接口`interface`可以包含[`use`][use]语句, [类型][types]定义和[函数][functions]定义。例如：

```wit
package wasi:filesystem;

interface types {
  use wasi:clocks/wall-clock.{datetime};

  record stat {
    ino: u64,
    size: u64,
    mtime: datetime,
    // ...
  }

  stat-file: func(path: string) -> result<stat>;
}
```

有关[`use`][use]和[类型][types]的更多信息将在下文中介绍，但此为`interface`内部项集合的示例。`interface`中定义的所有项（包括[`use`][use]），均被视为接口的导出。这意味着此interface的类型可被其他interface使用。接口具有单个命名空间，这意味着定义的名称都不会发生冲突。

WIT包可以包含任意数量的接口(interface)，这些接口在顶级(top-level)列出且顺序任意。WIT验证器将确保接口之间的所有引用都是格式正确且无环的。

## WIT世界（WIT Worlds）
[worlds]: #wit世界wit-worlds

除了[`interface`][interfaces]定义之外，WIT包还可以在顶级(top-level)包含`world`定义。world是组件导入和导出的完整描述。world可以被视为组件模型中`component`类型的等价物。例如：

```wit
package local:demo;

world my-world {
  import host: interface {
    log: func(param: string);
  }

  export run: func();
}
```

可以视为如下组件类型(component type)：

```wasm
(type $my-world (component
  (import "host" (instance
    (export "log" (func (param "param" string)))
  ))
  (export "run" (func))
))
```

世界描述了一个具体的组件，是生成绑定的基础。来宾语言将使用`world`来确定导入并命名哪些函数、导出哪些函数及其名称。

世界可以包含任意数量的导入和导出，并且可以为函数(function)或接口(interface)。

```wit
package local:demo;

world command {
  import wasi:filesystem/filesystem;
  import wasi:random/random;
  import wasi:clocks/monotonic-clock;
  // ...

  export main: func(args: list<string>);
}
```

有关`wasi:random/random`语法的更多信息，请参阅下方[`use`][use]描述。

导入或导出接口对应于组件模型中的导入或导出实例。函数相当于裸组件函数(bare component functions)。此外，接口可以用显式的[纯名称(plain name)][plain name]内联定义，从而避免了外联定义需要。

```wit
package local:demo;

interface out-of-line {
  the-function: func();
}

world your-world {
  import out-of-line;
  // ... 大致相当于 ...
  import out-of-line: interface {
    the-function: func();
  }
}
```

`import`或`export`语句的纯名称用作最终组件`import`或`export`定义的纯名称。

在组件模型中，导入组件可以使用纯名称(plain name)或接口名称(interface name)，在WIT中对应的语法：

```wit
package local:demo;

interface my-interface {
  // ..
}

world command {
  // 生成接口名称`local:demo/my-interface`的导入
  import my-interface;

  // 生成接口名称`wasi:filesystem/types`的导入
  import wasi:filesystem/types;

  // 生成纯名称`foo`的导入
  import foo: func();

  // 生成纯名称`bar`的导入
  import bar: interface {
    // ...
  }
}
```

每个名称在声明的范围内必须是唯一的（不区分大小写）。在world中，所有导入的名称都在同一范围内，但区分于所有导出的名称，因此同一个名称*不能*导入两次，但*能够*同时导出并导出。

[Plain Name]: Explainer.md#import-and-export-definitions

### 通过`include`合并多个世界（Union of Worlds with `include`）

可以通过合并两个或多个world来创建一个world。此操作允许从较小的world构建更大的world。

下面是一个world包含另外两个world的简单示例。

```wit
package local:demo;

// 省略了a、b、c、foo、bar、baz的定义

world my-world-a {
    import a;
    import b;
    export c;
}

world my-world-b {
    import foo;
    import bar;
    export baz;
}

world union-my-world {
     include my-world-a;
     include my-world-b;
}
```

该`include`语句用于将另一个world的导入和导出引入当前world。它表示新的world应能运行其包含world的全部组件及更多。

上面定义的`union-my-world`等同于下面的world：

```wit
world union-my-world {
    import a;
    import b;
    export c;
    import foo;
    import bar;
    export baz;
}
```

### 接口去重（De-duplication of interfaces）

如果两个world共享一个导入或导出[接口名称][interface name]，则两个world的并集将仅包含该导入或导出名称的一个副本。例如，下面的两个世界`union-my-world-a`和`union-my-world-b`是等效的：

```wit
package local:demo;

world my-world-a {
    import a1;
    import b1;
}

world my-world-b {
    import a1;
    import b1;
}

world union-my-world-a {
    include my-world-a;
    include my-world-b;
}

world union-my-world-b {
    import a1;
    import b1;
}
```

### 名称冲突及`with`（Name Conflicts and `with`）

当包含两个或更多world的导入或导出的纯名称(*plain* name)同名时，不能使用自动重复数据删除（因为两个同名的导入/导出在不同的world中可能有不同的含义），因此必须使用关键字`with`手动解决冲突。

以下示例说明如何解决`union-my-world-a`和`union-my-world-b`等效的名称冲突：
```wit
package local:demo;

world world-one { import a: func(); }
world world-two { import a: func(); }

world union-my-world-a {
    include world-one;
    include world-two with { a as b }
}

world union-my-world-b {
  import a: func();
  import b: func();
}
```

但是`with`不能用于重命名接口名称，因此以下world将是错误的：
```wit
package local:demo;

interface a {
    foo: func();
}

world world-using-a {
    import a;
}

world invalid-union-world {
    include my-using-a with { a as b }  // 错误：'a'是'local:demo/a'的缩写，为接口名称
}

```

### 关于子类型的注释（A Note on Subtyping）

未来，当支持`optional`导出时，world的作者可能会显示地标记导出为可选，以使一个目标为包含world的组件成为联合世界(union World)的子类型。

目前，我们不遵循该`include`语句的子类型规则。也就是说，该`include`语句不隐含任何其包含世界与联合世界之间的子类型关系。

## WIT包和`use`（WIT Packages and `use`）
[use]: #wit包和usewit-packages-and-use

WIT包表示分发单元，例如，可以发布到注册表并由其他WIT包使用。WIT包是`*.wit`文件中定义的一系列接口(interface)和世界(world)的集合。目前的惯例是，项目都会有一个`wit`文件夹，其中所有的`wit/*.wit`文件联合起来描述一个完整的包。

`use`语句的目的是接口之间共享类型，即使它们在当前包之外的依赖项中定义。`use`语句可以在interface和world中使用，也可以用在WIT文件的顶级(top-level)。

#### 接口、世界、和`use`（Interfaces, worlds, and use）

`interface`或`world`块内的`use`语句可用于导入类型：

```wit
package local:demo;

interface types {
  enum errno { /* ... */ }

  type size = u32;
}

interface my-host-functions {
  use types.{errno, size};
}
```

`use`的目标是`types`，在包的范围内会被解析为接口，在本例中是预先定义的。然后，提供了用`use`语句导入的类型列表。接口`type`在文本上可以出现在接口`use`指令之后或之前。与`use`关联的接口必须是无环的。

通过`use`导入的名称可以在导入时重命名：

```wit
package local:demo;

interface my-host-functions {
  use types.{errno as my-errno};
}
```

这种形式的`use`是使用单个标识符作为导入目标，在本例中为`types`。首先在当前文件范围中查找名称`types`，但它同时会查重包的命名空间。这意味着如果接口定义在同级文件中时上述语法仍然有效：

```wit
// types.wit
interface types {
  enum errno { /* ... */ }

  type size = u32;
}

// host.wit
package local:demo;

interface my-host-functions {
  use types.{errno, size};
}
```

此处的`types`接口未定义在`host.wit`中但会找到它，因为它在同一个包中定义，只是在不同的文件中。由于文件没有排序，但组件模型中的类型定义是有序且无环的，因此WIT解析器将对所有解析的WIT定义进行隐式拓扑排序，以找到无环定义顺序（如果没有则报错）

在[world][worlds]中导入或导出[interface][interfaces]，使用`import`和`export`指令的相同语法：

```wit
// a.wit
package local:demo;

world my-world {
  import host;

  export another-interface;
}

interface host {
  // ...
}

// b.wit
interface another-interface {
  // ...
}
```

引用接口时，可以使用完全限定的[接口名称(interface name)][interface name]。
例如，在此WIT文档：
```wit
package local:demo;

world my-world {
  import wasi:clocks/monotonic-clock;
}
```
`wasi:clocks`的`monotonic-clock`接口被导入。
同样的语法可以用于`use`：

```wit
package local:demo;

interface my-interface {
  use wasi:http/types.{request, response};
}
```

#### 顶层(Top-level)`use`

如果引用的包有版本号，那么使用上述语法到目前为止可能会有点重复：

```wit·
package local:demo;

interface my-interface {
  use wasi:http/types@1.0.0.{request, response};
}

world my-world {
  import wasi:http/handler@1.0.0;
  export wasi:http/handler@1.0.0;
}
```

为了减少重复并可能有助于避免命名冲突，`use`语句可以在文件顶层用于文件自身范围内的接口重命名。例如，上面的代码可以重写为：

```wit
package local:demo;

use wasi:http/types@1.0.0;
use wasi:http/handler@1.0.0;

interface my-interface {
  use types.{request, response};
}

world my-world {
  import handler;
  export handler;
}
```

这与之前世界的含义相同，`use`纯粹是为了方便开发人员在必要时提供较小的名字。

`use`引用的接口是在当前文件范围内定义的名称：

```wit
package local:demo;

use wasi:http/types;   // 定义名称`types`
use wasi:http/handler; // 定义名称`handler`
```

与接口级`use`类似，关键字`as`可以用于重命名推断名称：

```wit
package local:demo;

use wasi:http/types as http-types;
use wasi:http/handler as http-handler;
```

注意这些都可以组合使用以导入多版本包并重命名为不同的WIT标识符。

```wit
package local:demo;

use wasi:http/types@1.0.0 as http-types1;
use wasi:http/types@2.0.0 as http-types2;

// ...
```

### 传递导入和世界（Transitive imports and worlds）

`use`语句的实现不是通过复制类型信息，而是保留对其他地方定义的类型的引用。这种表示一直贯穿到最终组件，这意味着`use`类型会影响最终生成的组件结构。

例如此文档：

```wit
package local:demo;

interface shared {
  record metadata {
    // ...
  }
}

world my-world {
  import host: interface {
    use shared.{metadata};

    get: func() -> metadata;
  }
}
```

将生成此组件：

```wasm
(component
  (import "local:demo/shared" (instance $shared
    (type $metadata (record (; ... ;)))
    (export "metadata" (type (eq $metadata)))
  ))
  (alias export $shared "metadata" (type $metadata_from_shared))
  (import "host" (instance $host
    (export $metadata_in_host "metadata" (type (eq $metadata_from_shared)))
    (export "get" (func (result $metadata_in_host)))
  ))
)
```

此处可以看出尽管组件`world`仅列出`host`作为导入，但组件额外导入了`local:demo/shared`接口。这是因为`use shared.{ ... }`隐式地需要`shared`导入到组件中。

注意此处`"local:demo/shared"`名字是由`interface`加上包名`local:demo`组成。

对于`export`接口，任何可传递的`use`接口都被视为导入，除非明确将其列为导出。例如，这里`w1`相当于`w2`：
```wit
interface a {
  resource r;
}
interface b {
  use a.{r};
  foo: func() -> r;
}

world w1 {
  export b;
}
world w2 {
  import a;
  export b;
}
```

> **注意**：未来计划使用“高级用户语法”来更细粒度地配置导出，例如能够配置某个use接口是特定的导入还是特定的导出。

## WIT函数（WIT Functions）
[functions]: #wit函数wit-functions

函数定义于[`interface`][interfaces]，或在[`world`][worlds]中列为`import`或`export`。函数参数必须全部命名，并且名称在不区分大小写的情况下是唯一的：

```wit
package local:demo;

interface foo {
  a1: func();
  a2: func(x: u32);
  a3: func(y: u64, z: f32);
}
```

函数最多可以返回一个未命名类型：

```wit
package local:demo;

interface foo {
  a1: func() -> u32;
  a2: func() -> string;
}
```

并且函数还可以通过命名来返回多种类型：

```wit
package local:demo;

interface foo {
  a: func() -> (a: u32, b: f32);
}
```

请注意，从函数返回多个值并不等同于从函数返回一组值。这些选项在组件二进制格式中以不同的方式表示。

## WIT类型（WIT Types）
[types]: #wit类型wit-types

目前，WIT文件只能在[`interface`][interfaces]中定义类型。WIT中支持的类型与组件模型本身支持的类型相同：

```wit
package local:demo;

interface foo {
  // "命名字段包（package of named fields）"
  record r {
    a: u32,
    b: string,
  }

  // 此类型的值将是指定的情况之一
  variant human {
    baby,
    child(u32), // 可选类型载荷(payload)
    adult,
  }

  // 类似于`variant`，但没有类型载荷
  enum errno {
    too-big,
    too-small,
    too-fast,
    too-slow,
  }

  // 位标志(bitflags)类型
  flags permissions {
    read,
    write,
    exec,
  }

  // 基本类型允许使用类型别名，另外这里还有一些其他类型示例
  type t1 = u32;
  type t2 = tuple<u32, u64>;
  type t3 = string;
  type t4 = option<u32>;
  type t5 = result<_, errno>;           // 无"ok"类型
  type t6 = result<string>;             // 无"err"类型
  type t7 = result<char, errno>;        // 两种类型指定("ok"或"err")
  type t8 = result;                     // 无"ok"或"err"类型
  type t9 = list<string>;
  type t10 = t9;
}
```

`记录(record)`、`变量(variant)`、`枚举(enum)`、和`标志(flags)`类型都必须有与之关联的名称。`列表(list)`、`可选项(option)`、`结果(result)`、`元组(tuple)`和原始类型(primitive type)无需名称且可在任何上下文中提及。此限制是为了帮助在所有语言生成代码，尽可能地利用语言的内置类型，同时也适应哪些需要在美中语言中单独定义的类型。

## WIT标识符（WIT Identifiers）
[identifiers]: #wit-identifiers

WIT中的标识符可以使用两种不同的格式定义。第一种是组件模型文本格式中的[烤串命名法(kebab-case)][kebab-case][`label`](Explainer.md#import-and-export-names)。

```wit
foo: func(bar: u32);

red-green-blue: func(r: u32, g: u32, b: u32);

resource XML { ... }
parse-XML-document: func(s: string) -> XML;
```

这种格式在词汇上不能表示WIT关键字，因此第二种形式与第一种形式具有相同的语法和相同的限制，但以“％”为前缀：

```wit
%foo: func(%bar: u32);

%red-green-blue: func(%r: u32, %g: u32, %b: u32);

// 此表单还支持标识符，否则将是关键字。
%variant: func(%enum: s32);
```

[kebab-case]: https://en.wikipedia.org/wiki/Letter_case#Kebab_case

# 词汇结构（Lexical structure）
[lexical-structure]: #lexical-structure

`wit`格式是基于花括号的格式，其中空白是可选的（但建议使用）。`wit`文档被解析为unicode字符串，且当被存储在文件中时，预期会被编码为utf-8。

此外，wit文件必须不包含任何双向覆盖标量值，除换行符、回车符和水平制表符之外的控制代码或Unicode官方弃用或强烈不推荐的代码点。

目前的标记结构如下：

```ebnf
token ::= whitespace
        | operator
        | keyword
        | integer
        | identifier
```

解析此处其他地方定义的结构时，空格和注释将被忽略。

### 空白（Whitespace）

`wit`中的`whitespace`标记可以是空格、换行符、回车符、制表符、或注释：

```ebnf
whitespace ::= ' ' | '\n' | '\r' | '\t' | comment
```

### 注释（Comments）

`wit`中的`comment`标记要么是以`//`开头、换行符(`\n`)结尾的行注释，要么是以`/*`开头、`*/`结尾的块注释。请注意，块注释可以嵌套且其分隔符必须匹配。

```ebnf
comment ::= '//' character-that-isnt-a-newline*
          | '/*' any-unicode-character* '*/'
```

### 运算符（Operators）

在`wit`的词法结构中，有一些常见的运算符用于各种构造。需要注意的是，像`{`和`(`这样的定界符必须是配对的。

```ebnf
operator ::= '=' | ',' | ':' | ';' | '(' | ')' | '{' | '}' | '<' | '>' | '*' | '->' | '/' | '.' | '@'
```

### 关键字（Keywords）

某些标识符为WIT文档保留使用，不能直接用作标识符。其用于帮助解析格式，并且关键字列表目前仍在变化中，但当前的集合是：

```ebnf
keyword ::= 'use'
          | 'type'
          | 'resource'
          | 'func'
          | 'record'
          | 'enum'
          | 'flags'
          | 'variant'
          | 'static'
          | 'interface'
          | 'world'
          | 'import'
          | 'export'
          | 'package'
          | 'include'
```

### 整数（Integers）

整数目前仅用于包版本，是连续的数字序列：

```ebnf
integer ::= [0-9]+
```

## 顶级项（Top-level items）

`wit`文档是一系列在顶级指定的项。这些项一个接一个的出现，建议使用换行符将它们分开以提高可读性，但这不是必需的。

具体来说，`wit`文件的具体结构如下：

```ebnf
wit-file ::= explicit-package-list | implicit-package-definition
```

文件可以按两种方式组织。第一种是作为一系列连续的多个“显示”`package ... {...}`声明，包的内容在括号内。

```ebnf
explicit-package-list ::= explicit-package-definition*

explicit-package-definition ::= package-decl '{' package-items* '}'
```

或者，文件可以“隐式地”由可选`package ...;`声明，和随后的包项目(package items)列表组成。

```ebnf
implicit-package-definition ::= package-decl? package-items*
```

这两种结构不能混合：文件可以由显式或隐式样式写入，但不能同时使用两种样式。

`wit`文档中的所有其他声明都与包相关联，并定义如下。包定义由一个或多个这样的项组成：

```ebnf
package-items ::= toplevel-use-item | interface-item | world-item
```

### 特性门控（Feature Gates）

各种wit项可以被“门控”，以反映该项是不稳定功能的一部分，或该项是作为次要版本更新的一部分添加的，不应在针对早起次要版本时使用。

例如，以下接口有4个项目，其中3个是门控的：
```wit
interface foo {
  a: func();

  @since(version = 0.2.1)
  b: func();

  @since(version = 0.2.2, feature = fancy-foo)
  c: func();

  @unstable(feature = fancier-foo)
  d: func();
}
```
`@since`门表示`b`和`c`是在`0.2.1`和`0.2.2`版本中添加的。因此，当构建一个目标版本为`0.2.1`的组件时，可以使用`b`，但不能使用`c`。`@since`门设定的一个重要期望是，一旦将其应用到一个项目，该项目将不会向前进行不兼容的修改（根据一般的语义版本控制规则）。

相反，`d`上的`@unstable`门表示`d`是仍在积极开发的`fancier-foo`功能的一部分，因此`d`可能改变类型或随时移除。`@unstable`门设定的一个重要期望是，工具链默认不会暴露`@unstable`功能，除非开发者明确选择。

这两个门支持一种开发流程，在这种流程中，新功能在细节仍在讨论中时以`@unstable`门开始。然后，一旦功能稳定（并且，在WASI上下文中，经过投票），`@unstable`门会切换为`@since`门。为了实现平滑过渡（在此期间，生产工具链的目标版本早于 `@since`指定的`version`），`@since`门包含一个可选的`feature`字段，当该字段存在时，表示当目标版本大于*或*等于，*或者*开发者明确启用了特性(feature)名称时，启用该特性。因此，如果版本是`0.2.2`或更高，或者开发者明确启用了`fancy-foo`特性，`c`就会被启用。一旦生产工具链更新了他们的默认版本以默认启用该特性，就可以移除特性字段。

具体来说，特性门控的语法是：
```wit
gate ::= unstable-gate
       | since-gate
unstable-gate ::= '@unstable' '(' feature-field ')'
feature-field ::= 'feature' '=' id
since-gate ::= '@since' '(' 'version' '=' <valid semver> ( ',' feature-field )? ')'
```

作为WIT验证的一部分，任何项必须进行门控以兼容所引用的另一个门控项项。例如，这是一个错误：
```wit
interface i {
  @since(version = 1.0.1)
  type t1 = u32;

  type t2 = t1; // 错误
}
```
此外，如果某项*包含*在门控项中，则该项也必须兼容门控。例如，这是一个错误：
```wit
@since(version = 1.0.2)
interface i {
  foo: func();  // 错误: 没有门控

  @since(version = 1.0.1)
  bar: func();  // 同样错误: 宽松门控
}
```

## 包声明（Package declaration）
[package declaration]: #包声明package-declaration

WIT文件可以选择以定义包名称的包声明开头。

```ebnf
package-decl        ::= 'package' ( id ':' )+ id ( '/' id )* ('@' valid-semver)?  ';'
```

`valid-semver`项按[语义版本2.0(Semantic Versioning 2.0)](https://semver.org/)定义并且是可选的。

## 项：`toplevel-use`（Item: `toplevel-use`）

文件顶级(top-level)的`use`语句可以用于将接口引入当前文件的范围，并/或为了方便在本地重命名接口：

```ebnf
toplevel-use-item ::= 'use' use-path ('as' id)? ';'

use-path ::= id
           | id ':' id '/' id ('@' valid-semver)?
           | ( id ':' )+ id ( '/' id )+ ('@' valid-semver)? 🪺
```

此处的`use-path`是[接口名称(interface name)][interface name]。裸形式`id`指的是在当前包内定义的接口，而完全形式则指的是在包依赖中的接口。

`as`语法可以选择性地用来指定应赋予接口的名称。否则，名称将从`use-path`中推断而来。

作为未来的扩展，WIT、组件和组件注册表可能允许嵌套命名空间和包，这将会使得`use-path`的语法更加通用，如 🪺 后缀规则所示。

[Interface Name]: Explainer.md#import-and-export-definitions

## 项：`world`（Item: `world`）

世界定义了一个组件类型([`componenttype`])，它是一系列可以进行控制的导入和导出的集合。

具体来说，world的结构如下：

```ebnf
world-item ::= gate 'world' id '{' world-items* '}'

world-items ::= gate world-definition

world-definition ::= export-item
                   | import-item
                   | use-item
                   | typedef-item
                   | include-item

export-item ::= 'export' id ':' extern-type
              | 'export' use-path ';'
import-item ::= 'import' id ':' extern-type
              | 'import' use-path ';'

extern-type ::= func-type ';' | 'interface' '{' interface-items* '}'
```

请注意，world可以导入类型并定义自己的类型，以便从组件的根导出并在导入和导出的函数中使用。此处`interface`项还定义了用于引用`interface`项的ID的语法。


[`componenttype`]: Explainer.md#type-definitions

## 项：`include`（Item: `include`）

`include`语句可以将当前world与另一个world合并。`include`语句的结构如下：

```wit
include wasi:io/my-world-1 with { a as a1, b as b1 };
include my-world-2;
```

```ebnf
include-item ::= 'include' use-path ';'
               | 'include' use-path 'with' '{' include-names-list '}'

include-names-list ::= include-names-item
                     | include-names-list ',' include-names-item

include-names-item ::= id 'as' id
```

## 项：`interface`（Item: `interface`）

接口可以在`wit`文件中定义。接口有一个名称和一系列可以进行控制的项目和函数。

具体来说，接口的结构如下：

> **注意**：符号`ε`，也被称为Epsilon，表示一个空字符串。

```ebnf
interface-item ::= gate 'interface' id '{' interface-items* '}'

interface-items ::= gate interface-definition

interface-definition ::= typedef-item
                       | use-item
                       | func-item

typedef-item ::= resource-item
               | variant-items
               | record-item
               | flags-items
               | enum-items
               | type-item

func-item ::= id ':' func-type ';'

func-type ::= 'func' param-list result-list

param-list ::= '(' named-type-list ')'

result-list ::= ϵ
              | '->' ty
              | '->' '(' named-type-list ')'

named-type-list ::= ϵ
                  | named-type ( ',' named-type )*

named-type ::= id ':' ty
```


## 项：`use`（Item: `use`）

`use`语句允许从其他wit包或接口导入类型或资源定义。use语句的结构如下：

```wit
use an-interface.{a, list, of, names}
use my:dependency/the-interface.{more, names as foo}
```

具体来说，其结构如下：

```ebnf
use-item ::= 'use' use-path '.' '{' use-names-list '}' ';'

use-names-list ::= use-names-item
                 | use-names-item ',' use-names-list?

use-names-item ::= id
                 | id 'as' id
```

注意：此处`use-names-list?`表示至少一个`use-name-list`术语。

## 项：类型（Items: type）

在`wit`包中定义类型的方法有很多种，并且`wit`中所有可以定义的类型都旨在直接映射到[组件模型](https://github.com/WebAssembly/component-model)中的类型。

### 项：`type`(别名)（Item: `type` (alias)）

`type`语句在`wit`文档中声明一个新的命名类型。后续在使用此类型定义项时可以引用此名称。此构造类似于其他语言中的类型别名。

```wit
type my-awesome-u32 = u32;
type my-complicated-tuple = tuple<u32, s32, string>;
```

具体来说，其结构如下：

```ebnf
type-item ::= 'type' id '=' ty ';'
```

### 项：`record`(命名字段组)（Item: `record` (bag of named fields)）

`record`语句声明一个具有命名字段的新命名结构。record类似于许多语言中的`struct`。`record`实例始终具有已定义的字段。

```wit
record pair {
    x: u32,
    y: u32,
}

record person {
    name: string,
    age: u32,
    has-lego-action-figure: bool,
}
```

具体来说，其结构如下：

```ebnf
record-item ::= 'record' id '{' record-fields '}'

record-fields ::= record-field
                | record-field ',' record-fields?

record-field ::= id ':' ty
```

### 项：`flags`(布尔值组)（Item: `flags` (bag-of-bools)）

`flags`表示位集结构，每个位都有一个名称。该`flags`类型在规范ABI中表示为位标志(bit flags)表达。

```wit
flags properties {
    lego,
    marvel-superhero,
    supervillan,
}
```

具体来说，其结构如下：

```ebnf
flags-items ::= 'flags' id '{' flags-fields '}'

flags-fields ::= id
               | id ',' flags-fields?
```

### 项：`variant` (类型集合中的一个)（Item: `variant` (one of a set of types)）

`variant`语句定义了一种新类型，该类型的实例与其列出的变体之一完全匹配。这类似于代数数据类型中的"sum"类型（或者如果你熟悉 Rust，那就是`enum`）。变体(variant)也可以被认为是带标签的集合。

variant的每个分支都可以有一个可选类型与之关联，当值具有该特定分支的标签时，这个类型就会出现。

所有的`variant`类型必须至少指定一项(`variant-case`)。

```wit
variant filter {
    all,
    none,
    some(list<string>),
}
```

具体来说，其结构如下：

```ebnf
variant-items ::= 'variant' id '{' variant-cases '}'

variant-cases ::= variant-case
                | variant-case ',' variant-cases?

variant-case ::= id
               | id '(' ty ')'
```

### 项：`enum`（无载荷的variant）（Item: `enum` (variant but with no payload)）

`enum`语句定义了一种新类型，其语义等同于`variant`，但无有效荷载类型的情况。然而，这种情况被特殊处理，可能在语言ABI中有不同的表示形式，或者针对不同的语言生成不同的绑定。

```wit
enum color {
    red,
    green,
    blue,
    yellow,
    other,
}
```

具体来说，其结构如下：

```ebnf
enum-items ::= 'enum' id '{' enum-cases '}'

enum-cases ::= id
             | id ',' enum-cases?
```

### 项：`resource`（Item: `resource`）

`resource`语句为*资源*定义了一个新的抽象类型，资源是一种具有生命周期的实体，只能通过[句柄值(handle values)](#handles)间接地传递。资源类型在接口(interface)中用于描述不能或不应通过值复制的事物。

例如，以下Wit定义了一种资源类型，以及一个接受并返回`blob`句柄的函数：
```wit
resource blob;
transform: func(blob) -> blob;
```

作为语法糖，resource语句也可以声明任意数量的*方法(methods)*，此函数隐式接收一个为句柄类型的`self`参数。资源(resource)语句还可以包含任意数量的*静态方法(static function)*，其没有隐式的`self`参数但应在词法上嵌套在资源类型的范围内。最后，资源语句最多可以包含一个*构造器*(*constructor*)函数，它是返回包含资源类型句柄的函数的语法糖。

例如，以下资源定义：
```wit
resource blob {
    constructor(init: list<u8>);
    write: func(bytes: list<u8>);
    read: func(n: u32) -> list<u8>;
    merge: static func(lhs: borrow<blob>, rhs: borrow<blob>) -> blob;
}
```
解析为：
```wit
resource blob;
%[constructor]blob: func(init: list<u8>) -> blob;
%[method]blob.write: func(self: borrow<blob>, bytes: list<u8>);
%[method]blob.read: func(self: borrow<blob>, n: u32) -> list<u8>;
%[static]blob.merge: func(lhs: borrow<blob>, rhs: borrow<blob>) -> blob;
```
这些前缀为`%`的[`名称`](Explainer.md)内嵌资源类型的名称，以便绑定生成器可以为目标语言生成惯用语法，或者（对于像C这样的语义）回退到带有适当前缀的自由函数名称。

当直接使用资源类型名称时（例如，当`blob`用作上述构造函数的返回值时），它代表“自有”句柄，当丢弃时将调用资源的析构函数。当资源类型名称被`borrow<...>`包裹时，它代表“借用”句柄，当丢弃时*不会*调用析构函数。如上所示，method(方法)总是解析为一个借用的self参数，而constructor(构造函数)总是解析为一个自有返回值。

具体来说，资源定义的语法是：
```ebnf
resource-item ::= 'resource' id ';'
                | 'resource' id '{' resource-method* '}'
resource-method ::= func-item
                  | id ':' 'static' func-type ';'
                  | 'constructor' param-list ';'
```

句柄类型的语法[如下](#句柄handles)所示。

## 类型（Types）

如前所述，`wit`旨在允许定义与接口类型规范相对应的类型。上面的许多顶级项都引入了新的命名类型，但也支持“匿名(anonymous)”类型，如内置的(built-ins)。例如：

```wit
type number = u32;
type fallible-function-result = result<u32, string>;
type headers = list<string>;
```

具体来说，有以下类型可供选择：

```ebnf
ty ::= 'u8' | 'u16' | 'u32' | 'u64'
     | 's8' | 's16' | 's32' | 's64'
     | 'f32' | 'f64'
     | 'char'
     | 'bool'
     | 'string'
     | tuple
     | list
     | option
     | result
     | handle
     | id

tuple ::= 'tuple' '<' tuple-list '>'
tuple-list ::= ty
             | ty ',' tuple-list?

list ::= 'list' '<' ty '>'

option ::= 'option' '<' ty '>'

result ::= 'result' '<' ty ',' ty '>'
         | 'result' '<' '_' ',' ty '>'
         | 'result' '<' ty '>'
         | 'result'
```

`tuple`类型在语义上等同于具有数值字段的`record`，但其经常可以具有特定于语言的含义，所以她被视为一种一等类型。

类似地，`option`和`result`类型在语义上等同于variant：  

```wit
variant option {
    none,
    some(ty),
}

variant result {
    ok(ok-ty),
    err(err-ty),
}
```

这些类型经常被使用，并且经常具有特定于语言的含义，所以它们也被提供为一等类型。

最后，`ty`的最后一种情况就是简单的`id`，其目的是引用文档中定义的另一种类型或资源。请注意，这些定义可以来源于`use`语句，也可以在本地定义。

## 句柄（Handles）

Wit有两种句柄类型：“自有(owned)”句柄和“借用(borrowed)”句柄。自有句柄表示在两个组件间传递资源的唯一所有权。当自有句柄的所有者丢弃句柄时，资源会被销毁。相比之下，借用句柄表示在调用期间从调用者(caller)到被调用者(callee)的句柄的临时借用。

句柄的语法是：
```ebnf
handle ::= id
         | 'borrow' '<' id '>'
```


`id`表示一个自有句柄，其中`id`是先前的`resource`项。因此，资源在组件之间传递的“默认”方式是通过唯一所有权的转移。

上面定义资源方法的语法是语法糖，它扩展为单独的函数项，这些函数项接受一个名为`self`的第一个参数，参数的类型为`borrow`。例如，复合定义：
```wit
resource file {
    read: func(n: u32) -> list<u8>;
}
```
扩展为：
```wit
resource file
%[method]file.read: func(self: borrow<file>, n: u32) -> list<u8>;
```
其中`%[method]file.read`是方法根据组件模型的[命名(`name`)](Explainer.md)定义的解析后的名称。

## 名称解析（Name resolution）

`wit`文档在解析(parse)后进行解析(resolve)，以确保所有名称都能正确解析。例如这不是有效的`wit`文档：

```wit
type foo = bar;  // 错误：名称`bar`未定义
```

类型引用主要通过`ty`的`id`产生。

此外，`wit`文档中的名称只能定义一次：

```wit
type foo = u32;
type foo = u64;  // 错误：名称`foo`已定义
```

名称不需要在使用前定义（与C或C++不同），可以在使用后定义类型：

```wit
type foo = bar;

record bar {
    age: u32,
}
```

但是类型不能是递归的：

```wit
type foo = foo;  // 错误：不能引用自身

record bar1 {
    a: bar2,
}

record bar2 {
    a: bar1,    // 错误：record不能引用自身
}
```

# 包格式（Package Format）
[package-format]: #包格式package-format

每个顶层WIT定义可以编译成单个规范的组件模型[类型定义(type definition)](Explainer.md#type-definitions)，该定义捕获上述类型解析执行的结果。这些组件模型类型可以与其他类别和导出的组件一同被导出，从而允许单个组件同时打包运行时功能和开发时WIT接口。因此，WIT不需要自己单独的包格式；WIT可以作为组件二进制打包。

以这种方式使用组件二进制文件打包WIT有几个优点：
* 我们可以重用组件的[二进制格式](Binary.md)，特别是棘手的类型位。
* 下游工具不需要复制WIT包生产者者的解析逻辑或解析环境（目录，注册表，路径，参数等）；它可以重用更简单的编译结果。
* WIT语法的许多方面可以随着时间的推移演变，而不会破坏下游工具，这与Core WebAssembly WAT文本格式随着时间的推移发生的情况类似。
* 当组件在注册表中发布并分配名称时（参见[导入和导出定义](Explainer.md#import-and-export-definitions)中的命名讨论），WIT接口(interface)和世界(world)可以使用相同的工具发布，并使用相同的`namespace:package/export`命名方案命名。
* 单个包可以包含一个实现，以及由该实现导入的一系列`interface`和`world` 定义（例如，引擎组件可以定义和导出自己的插件`world`）。

作为第一个例子，以下WIT：
```wit
package local:demo;

interface types {
  resource file {
    read: func(off: u32, n: u32) -> list<u8>;
    write: func(off: u32, bytes: list<u8>);
  }
}

interface namespace {
  use types.{file};
  open: func(name: string) -> file;
}
```
可以打包成一个组件：
```wasm
(component
  (type (export "types") (component
    (export "local:demo/types" (instance
      (export $file "file" (type (sub resource)))
      (export "[method]file.read" (func
        (param "self" (borrow $file)) (param "off" u32) (param "n" u32)
        (result (list u8))
      ))
      (export "[method]file.write" (func
        (param "self" (borrow $file))
        (param "bytes" (list u8))
      ))
    ))
  ))
  (type (export "namespace") (component
    (import "local:demo/types" (instance $types
      (export "file" (type (sub resource)))
    ))
    (alias export $types "file" (type $file))
    (export "local:demo/namespace" (instance
      (export "open" (func (param "name" string) (result (own $file))))
    ))
  ))
)
```
此示例说明了接口的基本结构：
* 每个顶层WIT定义（在此示例中为：`types`和`namespace`）都变成相同的烤串命名(kebab-name)的类型导出。
* 每个WIT接口都映射到一个组件类型，该组件类型导出一个具有完全限定[接口名称][interface name]的实例（在此示例中为：`local:demo/types`和`local:demo/namespace`）。注意，此嵌套方案允许单个组件定义和实现WIT接口，而不会发生名称冲突。
* 包装的组件类型中接口的每个`use`都会`import`，将所有`use`类型引入到作用域中，以便在构建实例类型时可以对它们进行别名化。组件类型可以被认为是“参数化”接口的编译实例类型（∀T.{instance type}）。注意，即使接口(interface)不包含`use`也*始终*存在外部包装的组件类型。

这种编码方案的一个有用结果是每个顶层定义都是自包含的并且是有效的（根据组件模型验证规则），独立于其他定义。这允许轻松地拆分或合并包（假设结果不必是有效的包，而只是非导出类型定义的原始列表）。

另一个预期是，当包含WIT定义的组件发布到注册表时，注册表会验证组件内部的完全限定的WIT接口名称是否与注册表分配的软件包名称一致。例如，上述组件只有在发布的包名为`local:demo`时才有效；任何其他软件包名称都会与内部`local:demo/types`和`local:demo/namespace`导出的接口名称不一致。

包间引用在结构上与包内引用没有区别，除了引用的 WIT 定义不在组件中。例如，以下WIT：
```wit
package local:demo

interface foo {
  use wasi:http/types.{request};
  frob: func(r: request) -> request;
}
```
编码为：
```wasm
(component
  (type (export "foo") (component
    (import "wasi:http/types" (instance $types
      (export "request" (type (sub resource)))
    ))
    (alias export $types "request" (type $request))
    (export "local:demo/foo" (instance
      (export "frob" (func (param "r" (own $request)) (result (own $request))))
    ))
  ))
)
```

世界(world)的编码与接口类似，但将内部导出的instance替换为内部导出的*component*。例如，此WIT：
```wit
package local:demo;

world the-world {
  export test: func();
  export run: func();
}
```
编码为：
```wasm
(component
  (type (export "the-world") (component
    (export "local:demo/the-world" (component
      (export "test" (func))
      (export "run" (func))
    ))
  ))
)
```
在当前版本的WIT中，外部包装的组件类型将只包含一个`export`，因此仅用于将烤串命名导出与内部导出的接口名称分开，并提供与上面展示的`interface`的编码的一致性。

当世界(world)导入或导出接口时，为了生成有效的组件类型，接口的编译实例类型最终会被复制到组件类型中。例如，以下WIT：
```wit
package local:demo;

world the-world {
  import console;
}

interface console {
  log: func(arg: string);
}
```
编码为：
```wasm
(component
  (type (export "the-world") (component
    (export "local:demo/the-world" (component
      (import "local:demo/console" (instance
        (export "log" (func (param "arg" string)))
      ))
    ))
  ))
  (type (export "console") (component
    (export "local:demo/console" (instance
      (export "log" (func (param "arg" string)))
    ))
  ))
)
```
这种重复在跨包引用或拆分包的情况下很有用，允许编译的`world`定义完全自包含，并且能够用于编译组件而无需额外的类型信息。

综上所述，WIT 定义如下：
```wit
// wasi-http repo

// wit/types.wit
interface types {
  resource request { ... }
  resource response { ... }
}

// wit/handler.wit
interface handler {
  use types.{request, response};
  handle: func(r: request) -> response;
}

// wit/proxy.wit
package wasi:http;

world proxy {
  import wasi:logging/logger;
  import handler;
  export handler;
}
```
编码为：
```wasm
(component
  (type (export "types") (component
    (export "wasi:http/types" (instance
      (export "request" (type (sub resource)))
      (export "response" (type (sub resource)))
      ...
    ))
  ))
  (type (export "handler") (component
    (import "wasi:http/types" (instance $http-types
      (export "request" (type (sub resource)))
      (export "response" (type (sub resource)))
    ))
    (alias export $http-types "request" (type $request))
    (alias export $http-types "response" (type $response))
    (export "wasi:http/handler" (instance
      (export "handle" (func (param "r" (own $request)) (result (own $response))))
    ))
  ))
  (type (export "proxy") (component
    (export "wasi:http/proxy" (component
      (import "wasi:logging/logger" (instance
        ...
      ))
      (import "wasi:http/types" (instance $http-types
        (export "request" (type (sub resource)))
        (export "response" (type (sub resource)))
        ...
      ))
      (alias export $http-types "request" (type $request))
      (alias export $http-types "response" (type $response))
      (import "wasi:http/handler" (instance
        (export "handle" (func (param "r" (own $request)) (result (own $response))))
      ))
      (export "wasi:http/handler" (instance
        (export "handle" (func (param "r" (own $request)) (result (own $response))))
      ))
    ))
  ))
)
```
这个例子展示了，在具体world（`wasi:http/proxy`）的上下文中，独立的接口定义（如`wasi:http/handler`）不再是“参数化”形式：没有外部包装的组件类型，而是所有的`use`都被替换为由WIT解析过程确定的先前类型导入的直接别名。

与大多数其他WIT构造不同，`@since`和`@unstable`限制不会在组件二进制文件中表示出来。相反，它们被视为“宏(macro)”构造，代替维护单个WIT文档的两个副本。具体而言，在将一组WIT文档编码为二进制时，目标版本和一组显式启用的功能名称决定了各个限制功能是否包含在编码类型中。

例如，以下WIT文档：
```wit
package ns:p@1.1.0;

interface i {
  f: func();

  @since(version = 1.1.0)
  g: func();
}
```
当目标版本为`1.0.0`时，被编码为以下组件：
```wat
(component
  (type (export "i") (component
    (export "ns:p/i@1.0.0" (instance
      (export "f" (func))
    ))
  ))
)
```
如果目标版本为`1.1.0`，则相同的WIT文档将被编码为：
```wat
(component
  (type (export "i") (component
    (export "ns:p/i@1.1.0" (instance
      (export "f" (func))
      (export "g" (func))
    ))
  ))
)
```
因此，`@since`和`@unstable`限制不是组件运行时语义的一部分，而只是用于生成组件的源级工具的一部分。
