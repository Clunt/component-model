# `wit`格式

Wasm接口类型（Wasm Interface Type, WIT）格式是一种[IDL]，它以两种主要方式为[WebAssembly组件模型][components]提供工具：

* WIT是一种开发人员友好的格式，用于描述组件的导入和导出。其易于阅读和编写，并为使用客户语言生成组件以及使用宿主语言使用组件提供了基础。

* WIT包是组件生态系统中共享类型和定义的基础。作者可以在生成组件时从其他WIT包导入类型，发布表示宿主嵌入的WIT包，或者协作制定跨平台共享API集的WIT定义。

WIT包是在同一目录下以`wit`为扩展名的文件集合，其中定义了[`interfaces`][interfaces]和[`world`][worlds]，例如`foo.wit`。文件编码为有效的 utf-8 字节。可以使用非限定名称在包内的接口之间导入类型，还可以通过命名空间（namespace-
and-package-qualified）和包限定名称从其他包导入类型。

本文档将介绍WIT文档的句法结构的用途、伪形式[语法规范][lexical-structure]，以及适合分发的WIT包的[包格式(package format)][package-format]的规范。

[IDL]: https://en.wikipedia.org/wiki/Interface_description_language
[components]: https://github.com/webassembly/component-model

## 包名（Package Names）

所有WIT包均分配一个*包名*。包名类似于`foo:bar@1.0.0`，包含三个字段：

* *命名空间字段（namespace field）*，例如`foo:bar`中的`foo`。此命名空间旨在消除注册表、顶层组织等之间的歧义。例如，WASI接口（WASI interfaces）使用`wasi`命名空间。

* *包字段（package field）*，例如`wasi:clocks`中的`clocks`。“包”将一组interfaces和worlds组合在一起，否则它们会使用共同的前缀命名。

* *版本字段（version field）*[可选的], 定义为[full semver](https://semver.org/).

🪺 使用“嵌套命名空间和包”，包名称类似于`foo:bar:baz/quux`，其中`bar`是`foo`的嵌套命名空间、`quux`是`baz`的嵌套包。有关更多详细信息，请参阅[包声明]部分。

通过在WIT文件顶部声明`package`指定包名：

```wit
package wasi:clocks;
```

或者

```wit
package wasi:clocks@1.2.0;
```

WIT包可以由一组文件定义，且至少有一个文件须指定包名。多个文件可以指定`package`，但它们必须统一包名。

Alternatively, many packages can be declared consecutively in one or more
files, if the "explicit" package notation is used:

```wit
package local:a {
  interface foo {}
}

package local:b {
  interface bar {}
}
```

包名用于生成在组件模型中代表[导入名和导出名]的[`接口(interface)`][interfaces]和[`世界(world)`][worlds]，具体描述[如下](#package-format)。

[导入名和导出名]: Explainer.md#import-and-export-definitions

## WIT接口（Interfaces）
[interfaces]: #wit-interfaces

“接口”的概念是WIT的核心，它是[函数(functions)]和[类型(types)]的集合。接口可以被视为WebAssembly组件模型中的一个实例，例如从宿主导入或由组件实现的供宿主使用的功能单元。所有函数和类型都属于接口。

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

一个接口`interface`可以包含[`use`][use]语句, [type][types]定义和[function][functions]定义。例如：

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

有关[`use`][use]和[types]的更多信息将在下文中介绍，但这是`interface`中项目集合的示例。`interface`中定义的所有项目（包括[`use`][use]），均被视为接口的导出。这意味着此interface的types可被其他interface使用。接口具有单个命名空间，这意味着定义的名称都不会发生冲突。

WIT包可以包含任意数量的接口(interface)，这些接口在顶层列出且顺序任意。WIT验证器将确保接口之间的所有引用都是格式正确且无循环的。

## WIT世界（Worlds）
[worlds]: #wit-worlds

除了[`interface`][interfaces]定义之外，WIT包还可以在顶层包含`world`定义。world是组件导入和导出的完整描述。世界(world)可以被视为组件模型中`component`类型的等价物。例如：

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

世界描述了一个具体的组件，是生成绑定的基础。客户语言将使用`world`来确定导入并命名哪些函数、导出哪些函数及其名称。

世界可以包含任意数量的导入和导出，并且可以是函数(function)或接口(interface)。

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

导入或导出接口对应于组件模型中的导入或导出实例。函数相当于裸组件函数(bare component functions)。此外，接口可以用显式的[简单名称(plain name)][plain name]内联定义，从而避免了外联定义需要。

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

`import`或`export`语句的简单名称用于组件`import`或`export`最终定义的简单名称。

在组件模型中导入组件可以使用简单名称(plain name)或接口名称(interface name)，在WIT中对应的语法：

```wit
package local:demo;

interface my-interface {
  // ..
}

world command {
  // 生成导入接口名称 `local:demo/my-interface`
  import my-interface;

  // 生成导入接口名称 `wasi:filesystem/types`
  import wasi:filesystem/types;

  // 生成导入简单名称 `foo`
  import foo: func();

  // 生成导入简单名称 `bar`
  import bar: interface {
    // ...
  }
}
```

每个名称在声明的范围内必须是唯一的（不区分大小写）。在world中，所有导入的名称都在同一范围内，但区分于所有导出的名称，因此同一个名称*不能*导入两次，但*能够*同时导出并导出。

[Plain Name]: Explainer.md#import-and-export-definitions

### 通过`include`合并多个世界(world)

可以通过合并两个或多个world来创建一个world。此操作允许从较小的world构建更大的world。

下面是一个world包含另外两个world的简单示例。

```wit
package local:demo;

// definitions of a, b, c, foo, bar, baz are omitted

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

该`include`语句用于将另一个world的导入和导出引入当前world。它表示新world能够运行针对所包含world的所有组件等。

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

### 接口去重

如果两个world共享一个导入或导出[接口名称][interface name]，则两个world的并集将金包含该导入或导出名称的一个副本。例如，下面的两个世界`union-my-world-a`和`union-my-world-b`是等效的：

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

### 名称冲突及`with`

当两个或多个包含的world对于具有*plain* name的导入或导出具有相同的名称时，不能使用自动重复数据删除（因为两个同名的导入/导出在不同的 World 中可能有不同的含义），因此必须使用关键字`with`手动解决冲突。

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

但是`with`不能用于重命名接口名称，因此以下世界将是无效的：
```wit
package local:demo;

interface a {
    foo: func();
}

world world-using-a {
    import a;
}

world invalid-union-world {
    include my-using-a with { a as b }  // invalid: 'a'是'local:demo/a'的缩写，是一个接口名称
}

```

### 关于子类型的注释

将来当支持导出`optional`时，world作者可能会明确将导出标记为可选，以使针对包含的世界的组件成为union world的子类型。

目前，我们不遵循该`include`语句的子类型规则。也就是说，该`include`语句不暗示所包含的世界与联合世界之间的任何子类型关系。

## WIT包(Packages)和`use`
[use]: #wit-packages-and-use

WIT包表示分发单元，例如，可以发布到注册表并由其他WIT包使用。WIT包是`*.wit`文件中定义的一系列接口(interface)和世界(world)的集合。目前的惯例是，项目都会有一个`wit`文件夹，其中所有的`wit/*.wit`文件联合起来描述一个完整的包。

`use`语句的目的是接口之间共享类型，即使它们在当前包之外的依赖项中定义。`use`语句可以在interface和world中使用，也可以用在WIT文件的顶层。

#### 接口(interface)、世界(world)、和`use`

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
[functions]: #wit-functions

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
[types]: #wit-types

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

# Lexical structure
[lexical-structure]: #lexical-structure

The `wit` format is a curly-braced-based format where whitespace is optional (but
recommended). A `wit` document is parsed as a unicode string, and when stored in
a file is expected to be encoded as utf-8.

Additionally, wit files must not contain any bidirectional override scalar
values, control codes other than newline, carriage return, and horizontal tab,
or codepoints that Unicode officially deprecates or strongly discourages.

The current structure of tokens are:

```ebnf
token ::= whitespace
        | operator
        | keyword
        | integer
        | identifier
```

Whitespace and comments are ignored when parsing structures defined elsewhere
here.

### Whitespace

A `whitespace` token in `wit` is a space, a newline, a carriage return, a
tab character, or a comment:

```ebnf
whitespace ::= ' ' | '\n' | '\r' | '\t' | comment
```

### Comments

A `comment` token in `wit` is either a line comment preceded with `//` which
ends at the next newline (`\n`) character or it's a block comment which starts
with `/*` and ends with `*/`. Note that block comments are allowed to be nested
and their delimiters must be balanced

```ebnf
comment ::= '//' character-that-isnt-a-newline*
          | '/*' any-unicode-character* '*/'
```

### Operators

There are some common operators in the lexical structure of `wit` used for
various constructs. Note that delimiters such as `{` and `(` must all be
balanced.

```ebnf
operator ::= '=' | ',' | ':' | ';' | '(' | ')' | '{' | '}' | '<' | '>' | '*' | '->' | '/' | '.' | '@'
```

### Keywords

Certain identifiers are reserved for use in WIT documents and cannot be used
bare as an identifier. These are used to help parse the format, and the list of
keywords is still in flux at this time but the current set is:

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

### Integers

Integers are currently only used for package versions and are a contiguous
sequence of digits:

```ebnf
integer ::= [0-9]+
```

## Top-level items

A `wit` document is a sequence of items specified at the top level. These items
come one after another and it's recommended to separate them with newlines for
readability but this isn't required.

Concretely, the structure of a `wit` file is:

```ebnf
wit-file ::= explicit-package-list | implicit-package-definition
```

Files may be organized in two arrangements. The first of these is as a series
of multiple consecutive "explicit" `package ... {...}` declarations, with the
package's contents contained within the brackets.

```ebnf
explicit-package-list ::= explicit-package-definition*

explicit-package-definition ::= package-decl '{' package-items* '}'
```

Alternatively, a file may "implicitly" consist of an optional `package ...;`
declaration, followed by a list of package items.

```ebnf
implicit-package-definition ::= package-decl? package-items*
```

These two structures cannot be mixed: a file may be written in either in the
explicit or implicit styles, but not both at once.

All other declarations in a `wit` document are tied to a package, and defined
as follows. A package definition consists of one or more such items:

```ebnf
package-items ::= toplevel-use-item | interface-item | world-item
```

### Feature Gates

Various WIT items can be "gated", to reflect the fact that the item is part of
an unstable feature or that the item was added as part of a minor version
update and shouldn't be used when targeting an earlier minor version.

For example, the following interface has 4 items, 3 of which are gated:
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
The `@since` gate indicates that `b` and `c` were added as part of the `0.2.1`
and `0.2.2` releases, resp. Thus, when building a component targeting, e.g.,
`0.2.1`, `b` can be used, but `c` cannot. An important expectation set by the
`@since` gate is that, once applied to an item, the item is not modified
incompatibly going forward (according to general semantic versioning rules).

In contrast, the `@unstable` gate on `d` indicates that `d` is part of the
`fancier-foo` feature that is still under active development and thus `d` may
change type or be removed at any time. An important expectation set by the
`@unstable` gate is that toolchains will not expose `@unstable` features by
default unless explicitly opted-into by the developer.

Together, these gates support a development flow in which new features start
with an `@unstable` gate while the details are still being hashed out. Then,
once the feature is stable (and, in a WASI context, voted upon), the
`@unstable` gate is switched to a `@since` gate. To enable a smooth transition
(during which producer toolchains are targeting a version earlier than the
`@since`-specified `version`), the `@since` gate contains an optional `feature`
field that, when present, says to enable the feature when *either* the target
version is greator-or-equal *or* the feature name is explicitly enabled by the
developer. Thus, `c` is enabled if the version is `0.2.2` or newer or the
`fancy-foo` feature is explicitly enabled by the developer. The `feature` field
can be removed once producer toolchains have updated their default version to
enable the feature by default.

Specifically, the syntax for feature gates is:
```wit
gate ::= unstable-gate
       | since-gate
unstable-gate ::= '@unstable' '(' feature-field ')'
feature-field ::= 'feature' '=' id
since-gate ::= '@since' '(' 'version' '=' <valid semver> ( ',' feature-field )? ')'
```

As part of WIT validation, any item that refers to another gated item must also
be compatibly gated. For example, this is an error:
```wit
interface i {
  @since(version = 1.0.1)
  type t1 = u32;

  type t2 = t1; // error
}
```
Additionally, if an item is *contained* by a gated item, it must also be
compatibly gated. For example, this is an error:
```wit
@since(version = 1.0.2)
interface i {
  foo: func();  // error: no gate

  @since(version = 1.0.1)
  bar: func();  // also error: weaker gate
}
```

## Package declaration
[package declaration]: #package-declaration

WIT files optionally start with a package declaration which defines the name of
the package.

```ebnf
package-decl        ::= 'package' ( id ':' )+ id ( '/' id )* ('@' valid-semver)?  ';'
```

The production `valid-semver` is as defined by
[Semantic Versioning 2.0](https://semver.org/) and optional.

## Item: `toplevel-use`

A `use` statement at the top-level of a file can be used to bring interfaces
into the scope of the current file and/or rename interfaces locally for
convenience:

```ebnf
toplevel-use-item ::= 'use' use-path ('as' id)? ';'

use-path ::= id
           | id ':' id '/' id ('@' valid-semver)?
           | ( id ':' )+ id ( '/' id )+ ('@' valid-semver)? 🪺
```

Here `use-path` is an [interface name]. The bare form `id`
refers to interfaces defined within the current package, and the full form
refers to interfaces in package dependencies.

The `as` syntax can be optionally used to specify a name that should be assigned
to the interface. Otherwise the name is inferred from `use-path`.

As a future extension, WIT, components and component registries may allow
nesting both namespaces and packages, which would then generalize the syntax of
`use-path` as suggested by the 🪺 suffixed rule.

[Interface Name]: Explainer.md#import-and-export-definitions

## Item: `world`

Worlds define a [`componenttype`] as a collection of imports and exports, all
of which can be gated.

Concretely, the structure of a world is:

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

Note that worlds can import types and define their own types to be exported
from the root of a component and used within functions imported and exported.
The `interface` item here additionally defines the grammar for IDs used to refer
to `interface` items.

[`componenttype`]: Explainer.md#type-definitions

## Item: `include`

A `include` statement enables the union of the current world with another world. The structure of an `include` statement is:

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

## Item: `interface`

Interfaces can be defined in a `wit` file. Interfaces have a name and a
sequence of items and functions, all of which can be gated.

Specifically interfaces have the structure:

> **Note**: The symbol `ε`, also known as Epsilon, denotes an empty string.

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


## Item: `use`

A `use` statement enables importing type or resource definitions from other
wit packages or interfaces. The structure of a use statement is:

```wit
use an-interface.{a, list, of, names}
use my:dependency/the-interface.{more, names as foo}
```

Specifically the structure of this is:

```ebnf
use-item ::= 'use' use-path '.' '{' use-names-list '}' ';'

use-names-list ::= use-names-item
                 | use-names-item ',' use-names-list?

use-names-item ::= id
                 | id 'as' id
```

Note: Here `use-names-list?` means at least one `use-name-list` term.

## Items: type

There are a number of methods of defining types in a `wit` package, and all of
the types that can be defined in `wit` are intended to map directly to types in
the [component model](https://github.com/WebAssembly/component-model).

### Item: `type` (alias)

A `type` statement declares a new named type in the `wit` document. This name can
be later referred to when defining items using this type. This construct is
similar to a type alias in other languages

```wit
type my-awesome-u32 = u32;
type my-complicated-tuple = tuple<u32, s32, string>;
```

Specifically the structure of this is:

```ebnf
type-item ::= 'type' id '=' ty ';'
```

### Item: `record` (bag of named fields)

A `record` statement declares a new named structure with named fields. Records
are similar to a `struct` in many languages. Instances of a `record` always have
their fields defined.

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

Specifically the structure of this is:

```ebnf
record-item ::= 'record' id '{' record-fields '}'

record-fields ::= record-field
                | record-field ',' record-fields?

record-field ::= id ':' ty
```

### Item: `flags` (bag-of-bools)

A `flags` represents a bitset structure with a name for each bit. The `flags`
type is represented as a bit flags representation in
the canonical ABI.

```wit
flags properties {
    lego,
    marvel-superhero,
    supervillan,
}
```

Specifically the structure of this is:

```ebnf
flags-items ::= 'flags' id '{' flags-fields '}'

flags-fields ::= id
               | id ',' flags-fields?
```

### Item: `variant` (one of a set of types)

A `variant` statement defines a new type where instances of the type match
exactly one of the variants listed for the type. This is similar to a "sum" type
in algebraic datatypes (or an `enum` in Rust if you're familiar with it).
Variants can be thought of as tagged unions as well.

Each case of a variant can have an optional type associated with it which is
present when values have that particular case's tag.

All `variant` type must have at least one case specified.

```wit
variant filter {
    all,
    none,
    some(list<string>),
}
```

Specifically the structure of this is:

```ebnf
variant-items ::= 'variant' id '{' variant-cases '}'

variant-cases ::= variant-case
                | variant-case ',' variant-cases?

variant-case ::= id
               | id '(' ty ')'
```

### Item: `enum` (variant but with no payload)

An `enum` statement defines a new type which is semantically equivalent to a
`variant` where none of the cases have a payload type. This is special-cased,
however, to possibly have a different representation in the language ABIs or
have different bindings generated in for languages.

```wit
enum color {
    red,
    green,
    blue,
    yellow,
    other,
}
```

Specifically the structure of this is:

```ebnf
enum-items ::= 'enum' id '{' enum-cases '}'

enum-cases ::= id
             | id ',' enum-cases?
```

### Item: `resource`

A `resource` statement defines a new abstract type for a *resource*, which is
an entity with a lifetime that can only be passed around indirectly via [handle
values](#handles). Resource types are used in interfaces to describe things
that can't or shouldn't be copied by value.

For example, the following Wit defines a resource type and a function that
takes and returns a handle to a `blob`:
```wit
resource blob;
transform: func(blob) -> blob;
```

As syntactic sugar, resource statements can also declare any number of
*methods*, which are functions that implicitly take a `self` parameter that is
a handle. A resource statement can also contain any number of *static
functions*, which do not have an implicit `self` parameter but are meant to be
lexically nested in the scope of the resource type. Lastly, a resource
statement can contain at most one *constructor* function, which is syntactic
sugar for a function returning a handle of the containing resource type.

For example, the following resource definition:
```wit
resource blob {
    constructor(init: list<u8>);
    write: func(bytes: list<u8>);
    read: func(n: u32) -> list<u8>;
    merge: static func(lhs: borrow<blob>, rhs: borrow<blob>) -> blob;
}
```
desugars into:
```wit
resource blob;
%[constructor]blob: func(init: list<u8>) -> blob;
%[method]blob.write: func(self: borrow<blob>, bytes: list<u8>);
%[method]blob.read: func(self: borrow<blob>, n: u32) -> list<u8>;
%[static]blob.merge: func(lhs: borrow<blob>, rhs: borrow<blob>) -> blob;
```
These `%`-prefixed [`name`s](Explainer.md) embed the resource type name so that
bindings generators can generate idiomatic syntax for the target language or
(for languages like C) fall back to an appropriately-prefixed free function
name.

When a resource type name is used directly (e.g. when `blob` is used as the
return value of the constructor above), it stands for an "owning" handle
that will call the resource's destructor when dropped. When a resource
type name is wrapped with `borrow<...>`, it stands for a "borrowed" handle
that will *not* call the destructor when dropped. As shown above, methods
always desugar to a borrowed self parameter whereas constructors always
desugar to an owned return value.

Specifically, the syntax for a `resource` definition is:
```ebnf
resource-item ::= 'resource' id ';'
                | 'resource' id '{' resource-method* '}'
resource-method ::= func-item
                  | id ':' 'static' func-type ';'
                  | 'constructor' param-list ';'
```

The syntax for handle types is presented [below](#handles).

## Types

As mentioned previously the intention of `wit` is to allow defining types
corresponding to the interface types specification. Many of the top-level items
above are introducing new named types but "anonymous" types are also supported,
such as built-ins. For example:

```wit
type number = u32;
type fallible-function-result = result<u32, string>;
type headers = list<string>;
```

Specifically the following types are available:

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

The `tuple` type is semantically equivalent to a `record` with numerical fields,
but it frequently can have language-specific meaning so it's provided as a
first-class type.

Similarly the `option` and `result` types are semantically equivalent to the
variants:

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

These types are so frequently used and frequently have language-specific
meanings though so they're also provided as first-class types.

Finally the last case of a `ty` is simply an `id` which is intended to refer to
another type or resource defined in the document. Note that definitions can come
through a `use` statement or they can be defined locally.

## Handles

There are two types of handles in Wit: "owned" handles and "borrowed" handles.
Owned handles represent the passing of unique ownership of a resource between
two components. When the owner of an owned handle drops that handle, the
resource is destroyed. In contrast, a borrowed handle represents a temporary
loan of a handle from the caller to the callee for the duration of the call.

The syntax for handles is:
```ebnf
handle ::= id
         | 'borrow' '<' id '>'
```

The `id` case denotes an owned handle, where `id` is the name of a preceding
`resource` item. Thus, the "default" way that resources are passed between
components is via transfer of unique ownership.

The resource method syntax defined above is syntactic sugar that expands into
separate function items that take a first parameter named `self` of type
`borrow`. For example, the compound definition:
```wit
resource file {
    read: func(n: u32) -> list<u8>;
}
```
is expanded into:
```wit
resource file
%[method]file.read: func(self: borrow<file>, n: u32) -> list<u8>;
```
where `%[method]file.read` is the desugared name of a method according to the
Component Model's definition of [`name`](Explainer.md).


## Name resolution

A `wit` document is resolved after parsing to ensure that all names resolve
correctly. For example this is not a valid `wit` document:

```wit
type foo = bar;  // ERROR: name `bar` not defined
```

Type references primarily happen through the `id` production of `ty`.

Additionally names in a `wit` document can only be defined once:

```wit
type foo = u32;
type foo = u64;  // ERROR: name `foo` already defined
```

Names do not need to be defined before they're used (unlike in C or C++),
it's ok to define a type after it's used:

```wit
type foo = bar;

record bar {
    age: u32,
}
```

Types, however, cannot be recursive:

```wit
type foo = foo;  // ERROR: cannot refer to itself

record bar1 {
    a: bar2,
}

record bar2 {
    a: bar1,    // ERROR: record cannot refer to itself
}
```

# Package Format
[package-format]: #package-format

Each top-level WIT definition can be compiled into a single canonical
Component Model [type definition](Explainer.md#type-definitions) that
captures the result of performing the type resolution described above. These
Component Model types can then be exported by a component along with other
sorts of exports, allowing a single component to package both runtime
functionality and development-time WIT interfaces. Thus, WIT does not need its
own separate package format; WIT can be packaged as a component binary.

Using component binaries to package WIT in this manner has several advantages:
* We get to reuse the [binary format](Binary.md) of components, especially the
  tricky type bits.
* Downstream tooling does not need to replicate the resolution logic nor the
  resolution environment (directories, registries, paths, arguments, etc) of
  the WIT package producer; it can reuse the simpler compiled result.
* Many aspects of the WIT syntax can evolve over time without breaking
  downstream tooling, similar to what has happened with the Core WebAssembly
  WAT text format over time.
* When components are published in registries and assigned names (see the
  discussion of naming in [Import and Export Definitions](Explainer.md#import-and-export-definitions)),
  WIT interfaces and worlds can be published with the same tooling and named
  using the same `namespace:package/export` naming scheme.
* A single package can both contain an implementation and a collection of
  `interface` and `world` definitions that are imported by that implementation
  (e.g., an engine component can define and exports its own plugin `world`).

As a first example, the following WIT:
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
can be packaged into a component as:
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
This example illustrates the basic structure of interfaces:
* Each top-level WIT definition (in this example: `types` and `namespace`)
  turns into a type export of the same kebab-name.
* Each WIT interface is mapped to a component-type that exports an
  instance with a fully-qualified [interface name]  (in this example:
  `local:demo/types` and `local:demo/namespace`). Note that this nested
  scheme allows a single component to both define and implement a WIT interface
  without name conflict.
* The wrapping component-type has an `import` for every `use` in the interface,
  bringing any `use`d types into scope so that they can be aliased when
  building the instance-type. The component-type can be thought of as
  "parameterizing" the interface's compiled instance type (∀T.{instance type}).
  Note that there is *always* an outer wrapping component-type, even when the
  interface contains no `use`s.

One useful consequence of this encoding scheme is that each top-level
definition is self-contained and valid (according to Component Model validation
rules) independent of each other definition. This allows packages to be
trivially split or unioned (assuming the result doesn't have to be a valid
package, but rather just a raw list of non-exported type definitions).

Another expectation is that, when a component containing WIT definitions is
published to a registry, the registry validates that the fully-qualified WIT
interface names inside the component are consistent with the registry-assigned
package name. For example, the above component would only be valid if published
with package name `local:demo`; any other package name would be inconsistent
with the internal `local:demo/types` and `local:demo/namespace` exported
interface names.

Inter-package references are structurally no different than intra-package
references other than the referenced WIT definition is not present in
the component. For example, the following WIT:
```wit
package local:demo

interface foo {
  use wasi:http/types.{request};
  frob: func(r: request) -> request;
}
```
is encoded as:
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

Worlds are encoded similarly to interfaces, but replace the inner exported
instance with an inner exported *component*. For example, this WIT:
```wit
package local:demo;

world the-world {
  export test: func();
  export run: func();
}
```
is encoded as:
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
In the current version of WIT, the outer wrapping component-type will only ever
contain a single `export` and thus only serves to separate the kebab-name
export from the inner exported interface name and to provide consistency with
the encoding of `interface` shown above.

When a world imports or exports an interface, to produce a valid
component-type, the interface's compiled instance-type ends up getting copied
into the component-type. For example, the following WIT:
```wit
package local:demo;

world the-world {
  import console;
}

interface console {
  log: func(arg: string);
}
```
is encoded as:
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
This duplication is useful in the case of cross-package references or split
packages, allowing a compiled `world` definition to be fully self-contained and
able to be used to compile a component without additional type information.

Putting this all together, the following WIT definitions:
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
are encoded as:
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
This examples shows how, in the context of concrete world (`wasi:http/proxy`),
standalone interface definitions (such `wasi:http/handler`) are no longer in a
"parameterized" form: there is no outer wrapping component-type and instead all
`use`s are replaced by direct aliases to preceding type imports as determined
by the WIT resolution process.

Unlike most other WIT constructs, the `@since` and `@unstable` gates are not
represented in the component binary. Instead, they are considered "macro"
constructs that take the place of maintaining two copies of a single WIT
document. In particular, when encoding a collection of WIT documents into a
binary, the target version and set of explicitly-enabled feature names
determine whether individual gated features are included in the encoded type or
not.

For example, the following WIT document:
```wit
package ns:p@1.1.0;

interface i {
  f: func();

  @since(version = 1.1.0)
  g: func();
}
```
is encoded as the following component when the target version is `1.0.0`:
```wat
(component
  (type (export "i") (component
    (export "ns:p/i@1.0.0" (instance
      (export "f" (func))
    ))
  ))
)
```
If the target version was instead `1.1.0`, the same WIT document would be
encoded as:
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
Thus, `@since` and `@unstable` gates are not part of the runtime semantics of
components, just part of the source-level tooling for producing components.
