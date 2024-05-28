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

## WIT Packages and `use`
[use]: #wit-packages-and-use

A WIT package represents a unit of distribution that can be published to a
registry, for example, and used by other WIT packages. WIT packages are a flat
list of interfaces and worlds defined in `*.wit` files. The current thinking
for a convention is that projects will have a `wit` folder where all
`wit/*.wit` files within describe a single package.

The purpose of the `use` statement is to enable sharing types between
interfaces, even if they're defined outside of the current package in a
dependency. The `use` statement can be used both within interfaces and worlds
and at the top-level of a WIT file.

#### Interfaces, worlds, and `use`

A `use` statement inside of an `interface` or `world` block can be used to
import types:

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

The `use` target, `types`, is resolved within the scope of the package to an
interface, in this case defined prior. Afterwards a list of types are provided
as what's going to be imported with the `use` statement. The interface `types`
may textually come either after or before the `use` directive's interface.
Interfaces linked with `use` must be acyclic.

Names imported via `use` can be renamed as they're imported as well:

```wit
package local:demo;

interface my-host-functions {
  use types.{errno as my-errno};
}
```

This form of `use` is using a single identifier as the target of what's being
imported, in this case `types`. The name `types` is first looked up within the
scope of the current file, but it will additionally consult the package's
namespace as well. This means that the above syntax still works if the
interfaces are defined in sibling files:

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

Here the `types` interface is not defined in `host.wit` but lookup will find it
as it's defined in the same package, just instead in a different file. Since
files are not ordered, but type definitions in the Component Model are ordered
and acyclic, the WIT parser will perform an implicit topological sort of all
parsed WIT definitions to find an acyclic definition order (or produce an error
if there is none).

When importing or exporting an [interface][interfaces] in a [world][worlds]
the same syntax is used in `import` and `export` directives:

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

When referring to an interface, a fully-qualified [interface name] can be used.
For example, in this WIT document:
```wit
package local:demo;

world my-world {
  import wasi:clocks/monotonic-clock;
}
```
The `monotonic-clock` interface of the `wasi:clocks` package is being imported.
This same syntax can be used in `use` as well:

```wit
package local:demo;

interface my-interface {
  use wasi:http/types.{request, response};
}
```

#### Top-level `use`

If a package being referred to has a version number, then using the above syntax
so far it can get a bit repetitive to be referred to:

```wit
package local:demo;

interface my-interface {
  use wasi:http/types@1.0.0.{request, response};
}

world my-world {
  import wasi:http/handler@1.0.0;
  export wasi:http/handler@1.0.0;
}
```

To reduce repetition and to possibly help avoid naming conflicts the `use`
statement can additionally be used at the top-level of a file to rename
interfaces within the scope of the file itself. For example the above could be
rewritten as:

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

The meaning of this and the previous world are the same, and `use` is purely a
developer convenience for providing smaller names if necessary.

The interface referred to by a `use` is the name that is defined in the current
file's scope:

```wit
package local:demo;

use wasi:http/types;   // defines the name `types`
use wasi:http/handler; // defines the name `handler`
```

Like with interface-level-`use` the `as` keyword can be used to rename the
inferred name:

```wit
package local:demo;

use wasi:http/types as http-types;
use wasi:http/handler as http-handler;
```

Note that these can all be combined to additionally import packages with
multiple versions and renaming as different WIT identifiers.

```wit
package local:demo;

use wasi:http/types@1.0.0 as http-types1;
use wasi:http/types@2.0.0 as http-types2;

// ...
```

### Transitive imports and worlds

A `use` statement is not implemented by copying type information around but
instead retains that it's a reference to a type defined elsewhere. This
representation is plumbed all the way through to the final component, meaning
that `use`d types have an impact on the structure of the final generated
component.

For example this document:

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

would generate this component:

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

Here it can be seen that despite the `world` only listing `host` as an import
the component additionally imports a `local:demo/shared` interface. This is due
to the fact that the `use shared.{ ... }` implicitly requires that `shared` is
imported into the component as well.

Note that the name `"local:demo/shared"` here is derived from the name of the
`interface` plus the package name `local:demo`.

For `export`ed interfaces, any transitively `use`d interface is assumed to be an
import unless it's explicitly listed as an export. For example, here `w1` is
equivalent to `w2`:
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

> **Note**: It's planned in the future to have "power user syntax" to configure
> this on a more fine-grained basis for exports, for example being able to
> configure that a `use`'d interface is a particular import or a particular
> export.

## WIT Functions
[functions]: #wit-functions

Functions are defined in an [`interface`][interfaces] or are listed as an
`import` or `export` from a [`world`][worlds]. Parameters to a function must all
be named and have case-insensitively unique names:

```wit
package local:demo;

interface foo {
  a1: func();
  a2: func(x: u32);
  a3: func(y: u64, z: f32);
}
```

Functions can return at most one unnamed type:

```wit
package local:demo;

interface foo {
  a1: func() -> u32;
  a2: func() -> string;
}
```

And functions can also return multiple types by naming them:

```wit
package local:demo;

interface foo {
  a: func() -> (a: u32, b: f32);
}
```

Note that returning multiple values from a function is not equivalent to
returning a tuple of values from a function. These options are represented
distinctly in the component binary format.

## WIT Types
[types]: #wit-types

Types in WIT files can only be defined in [`interface`s][interfaces] at this
time. The types supported in WIT is the same set of types supported in the
component model itself:

```wit
package local:demo;

interface foo {
  // "package of named fields"
  record r {
    a: u32,
    b: string,
  }

  // values of this type will be one of the specified cases
  variant human {
    baby,
    child(u32), // optional type payload
    adult,
  }

  // similar to `variant`, but no type payloads
  enum errno {
    too-big,
    too-small,
    too-fast,
    too-slow,
  }

  // a bitflags type
  flags permissions {
    read,
    write,
    exec,
  }

  // type aliases are allowed to primitive types and additionally here are some
  // examples of other types
  type t1 = u32;
  type t2 = tuple<u32, u64>;
  type t3 = string;
  type t4 = option<u32>;
  type t5 = result<_, errno>;           // no "ok" type
  type t6 = result<string>;             // no "err" type
  type t7 = result<char, errno>;        // both types specified
  type t8 = result;                     // no "ok" or "err" type
  type t9 = list<string>;
  type t10 = t9;
}
```

The `record`, `variant`, `enum`, and `flags` types must all have names
associated with them. The `list`, `option`, `result`, `tuple`, and primitive
types do not need a name and can be mentioned in any context. This restriction
is in place to assist with code generation in all languages to leverage
language-builtin types where possible while accommodating types that need to be
defined within each language as well.

## WIT Identifiers
[identifiers]: #wit-identifiers

Identifiers in WIT can be defined with two different forms. The first is the
[kebab-case] [`label`](Explainer.md#import-and-export-names) production in the
Component Model text format.

```wit
foo: func(bar: u32);

red-green-blue: func(r: u32, g: u32, b: u32);

resource XML { ... }
parse-XML-document: func(s: string) -> XML;
```

This form can't lexically represent WIT keywords, so the second form is the
same syntax with the same restrictions as the first, but prefixed with '%':

```wit
%foo: func(%bar: u32);

%red-green-blue: func(%r: u32, %g: u32, %b: u32);

// This form also supports identifiers that would otherwise be keywords.
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
wit-file ::= package-decl? (toplevel-use-item | interface-item | world-item)*
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

Worlds define a [componenttype](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#type-definitions) as a collection of imports and exports.

Concretely, the structure of a world is:

```ebnf
world-item ::= 'world' id '{' world-items* '}'

world-items ::= export-item | import-item | use-item | typedef-item | include-item

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
sequence of items and functions.

Specifically interfaces have the structure:

> **Note**: The symbol `ε`, also known as Epsilon, denotes an empty string.

```ebnf
interface-item ::= 'interface' id '{' interface-items* '}'

interface-items ::= typedef-item
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

