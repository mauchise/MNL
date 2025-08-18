# Morf 规范

Morf 是一种极简且通用的树形数据结构。它的设计哲学是：**万物皆 Stage**。通过一种统一的结构来描述一切，Morf 为领域特定语言 (DSL)、配置文件和数据交换格式提供了一个简单而强大的基础。

Morf Notation Language (MNL) 是 Morf 数据结构的统一文本表示法，旨在实现人类可读、机器可解析的目标。

## 1. Morf：核心数据结构

Morf 的世界里只存在一种构建块，即 **Stage**。任何复杂的结构都是由 Stage 嵌套组合而成。

### 1.1. Stage 的定义

一个 Stage 由三个部分组成：

-   `id` (`string`): **类型标识符**。用于标记一个 Stage 的类型或意图，例如 `div`, `user`, `@name`。
-   `payload` (`any`): **载荷**。用于承载基元值，例如字符串、数字、布尔值等。如果一个 Stage 仅用于组织结构，其 payload 可以为空。
-   `stages` (`Stage[]`): **子节点列表**。一个有序的 Stage 列表，用于构建层级结构。

一个典型的 Stage 可以用以下伪代码表示：

```typescript
interface Stage {
  id: string;
  payload: any;
  stages: Stage[];
}
```

例如，一个表示纯文本的 Stage 可以是：

```json
{
  "id": "text-literal",
  "payload": "hello world",
  "stages": []
}
```

## 2. Morf Notation Language (MNL)：统一文本表示

MNL 是 Morf 结构的文本化语言。它的语法核心极度精简，但通过可扩展的解析规则，能够优雅地表达丰富的数据结构。

### 2.1. 基础语法

#### 2.1.1. Stage 表达式

一个 Stage 的基本表达形式为：`[id] { [children] }`

-   `[id]` 是 Stage 的 `id`。
-   花括号 `{}` 用于包裹所有的子 Stage (`stages` 列表)。子 Stage 之间用空格分隔。

```mnl
database {
  host: "localhost"
  port: 5432
  user: "admin"
}
```

#### 2.1.2. 语法简写

如果一个 Stage 只有一个子 Stage，可以使用冒号 `:` 来代替花括号，使结构更紧凑。

`[id]: [child]` 等价于 `[id] { [child] }`

例如，上面的 `database` 示例可以写成：

```mnl
database {
  host: "localhost"
  port: 5432
  user: "admin"
}
```

### 2.2. 超越语法：自定义解析与语义约定

MNL 的核心语法并不包含字符串、数字等字面量的定义。这些是由 **自定义解析规则** 提供的。解析器可以定义规则来识别特定模式（如 `"..."` 或 `123`），将其吞噬，并返回一个带有 `payload` 的 Stage。

这种设计将 **结构语法** 与 **词法解析** 分离，赋予了 MNL 极大的灵活性。

#### 2.2.1. 字面量解析

通过配置解析器，我们可以轻松支持常见的字面量。例如，解析器可以添加以下规则：

-   识别 `"..."` 形式的文本，生成 `{ id: "text-literal", payload: "..." }`。
-   识别 `123` 或 `3.14` 形式的数字，生成 `{ id: "number-literal", payload: ... }`。

*为了方便下文展示，我们约定用 `id(payload)` 的形式来表示解析后生成的、带有载荷的 Stage。*

#### 2.2.2. 键值对的表达（约定）

MNL 本身是一个树形结构，没有内置“键值对”的概念。然而，我们可以通过 **约定** 来优雅地表达它。

我们推荐使用以 `@` 符号开头的 `id` 来表示“属性”或“字段名”，它的子 Stage 则是“值”。这种方式直观且符合许多现有语言的习惯。

```mnl
human name: "John Doe" age: 30
```

这段 MNL 会被解析为如下的 Morf 结构：

```
human {
  name: text-literal("John Doe")
  age: number-literal(30)
}
```

## 3. 实践示例

以下示例展示了 MNL 如何在不同场景下描述数据。

### 3.1. 表达一个数字列表

```mnl
my-list {
  1 2 3 4 5
}
```

解析后的 Morf 结构：

```
my-list {
  number-literal(1)
  number-literal(2)
  number-literal(3)
  number-literal(4)
  number-literal(5)
}
```

### 3.2. 表达一个用户信息

```mnl
user {
  id: 1001
  username: "morf-user"
  email: "user@example.com"
  is-active: true
  roles {
    role: "admin"
    role: "editor"
  }
}
```

解析后的 Morf 结构：

```
user {
  id: number-literal(1001)
  username: text-literal("morf-user")
  email: text-literal("user@example.com")
  is-active: boolean-literal(true)
  roles {
    role: text-literal("admin")
    role: text-literal("editor")
  }
}
```

### 3.3. 表达一个 SVG 元素

MNL 非常适合描述像 HTML/XML 这样的标记语言。

```mnl
g {
  @id: "my-group"
  @fill: "blue"
  @stroke: "black"
  @stroke-width: "3"
  @transform: "translate(50, 50)"
  @opacity: "0.8" 

  rect { 
    @x: "0" @y: "0"
    @width: "100" @height: "60"
  }
  circle {
    @cx: "100" @cy: "60"
    @r: "40" @fill: "red"
  }
}
```

这清晰地表达了一个包含矩形和圆形的 SVG 组，所有属性都通过 `@` 约定来表示。

## 4. 核心设计哲学

1.  **统一的结构 (Stage)**：所有数据，无论简单还是复杂，都由 Stage 构成。这提供了概念上的一致性。
2.  **最小化的语法**：MNL 的核心语法只定义了如何组织 Stage 树。它不关心“值”的具体形态。
3.  **强大的可扩展性**：通过自定义解析规则，Morf 可以适应任何领域，从简单的配置文件到复杂的领域特定语言。
4.  **约定优于配置**：像 `@` 这样的键值对表示法是一种社区约定，而非语言的强制规定，保持了核心的灵活性。

Morf 致力于成为构建未来数据表示和语言的通用基石。
