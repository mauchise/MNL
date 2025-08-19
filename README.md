# Morf 规范 (Morf Specification)

Morf 是一种极简且通用的树形数据结构。其核心设计哲学是：**万物皆 Stage**。通过这一统一的抽象，Morf 为领域特定语言 (DSL)、配置文件和数据交换格式提供了一个简单、一致且功能强大的基础。

Morf Notation Language (MNL) 是 Morf 数据结构的规范文本表示法，旨在实现极致的人类可读性与机器可解析性。

## 1. Morf: 核心数据结构

Morf 的世界中仅存在一种构建单元，即 **Stage**。任何复杂的结构都是由 Stage 嵌套组合而成。

### 1.1. Stage 的定义

一个 Stage 由三个部分组成：

-   `id` (`string`): **类型标识符**。用于标记一个 Stage 的类型或意图，例如 `div`, `user`, `host`。
-   `payload` (`any`): **载荷**。用于承载基元值，如字符串、数字等。如果一个 Stage 仅用于组织结构，其 payload 可以为空。
-   `stages` (`Stage[]`): **子节点列表**。一个有序的 Stage 列表，用于构建层级结构。

一个 Stage 的抽象表示如下：

```typescript
interface Stage {
  id: string;
  payload: any;
  stages: Stage[];
}
```

## 2. Morf Notation Language (MNL): 文本表示法

MNL 是 Morf 结构的文本化语言。它的语法被设计为最小化的，同时通过可扩展的解析规则支持无限的表达能力。

### 2.1. 基础词法 (Lexical Elements)

#### 2.1.1. Stage ID

一个 Stage `id` 是用于标识 Stage 类型的字符序列。

-   **构成**：`id` 可以包含除**分隔符**之外的任何字符。
-   **分隔符 (Delimiters)**：以下字符在语法上有特殊含义，因此会终止一个 `id` 的解析：
    -   **空白符**：空格 (` `)、制表符 (`\t`)、换行符 (`\n`, `\r`)。
    -   **结构符**：`{`, `}`, `:`。
-   **转义 (Escaping)**：若 `id` 必须包含上述结构符，必须使用反斜杠 `\` 进行转义。

| 期望的 ID | 在 MNL 中的表示 |
| :--- | :--- |
| `my-id` | `my-id` |
| `path/to/file`| `path/to/file` |
| `config:dev` | `config\:dev` |

#### 2.1.2. 空白符 (Whitespace)

-   空格、制表符和换行符被视为空白符。
-   空白符用于分隔**同级的 Stage**。
-   连续的多个空白符等同于一个。
-   **缩进没有语法意义**，仅用于提高可读性。

#### 2.1.3. 注释 (Comments)

-   以 `;` (分号) 开头的行被视为单行注释。
-   解析器会完全忽略从 `;` 到行尾的所有内容。

### 2.2. 核心语法 (Core Syntax)

一个 Stage 由一个 `id` 和紧随其后的子 Stage 容器组成。定义子 Stage 只有以下两种方式：

#### 2.2.1. 块容器 (Block Container)

使用花括号 `{}` 包裹**零个或多个**子 Stage。这是定义多个子 Stage 的标准方式。

`[id] { [child1] [child2] ... }`

```mnl
; 一个有多个子 Stage 的 Stage
user {
  profile
  settings
}

; 一个没有子 Stage 的 Stage
empty-node {}
```

#### 2.2.2. 单一容器 (Single Container)

使用冒号 `:` 指定**唯一一个**子 Stage。这是 `[id] { [child] }` 的语法简写。

`[id]: [child]`

```mnl
user: profile
```
**重要**: 一个 Stage ID 后面只能跟随一种容器 (`{}` 或 `:`)。`[id] [child] { ... }` 这样的混合语法是**无效的**。

### 2.3. 可扩展性与模式

MNL 的核心语法不直接定义字符串、数字等字面量。这些是通过**自定义解析规则**来实现的。

#### 2.3.1. 字面量与 Payload

当解析器在流中遇到一个不符合核心语法的 token 时，它会尝试使用一系列**自定义字面量解析器**去匹配。

*为方便下文展示，我们约定用 `type(payload)` 的形式来表示解析后生成的、带有载荷的 Stage。*

#### 2.3.2. 表达键值对：模式与约定

MNL 的树形结构天然支持键值对模式。

**通用模式 (General Pattern):**
最直接的模式是使用 `id` 作为“键”，其子 Stage 作为“值”。

```mnl
database {
  host: "localhost"
  port: 5432
  user: "admin"
}
```

**属性约定 (`@` Convention):**
在需要同时表达**属性 (Attributes)**和**子元素 (Child Elements)**的场景下（类似 XML/HTML），我们推荐使用 `@` 前缀来区分“属性”。

根据核心语法，**属性也是子 Stage**，它们必须和其它子 Stage 一样，被放置在同一个容器中。

```mnl
; 一个类似 HTML 元素的 Stage
div {
  ; 属性和子元素都是 div 的子 Stage
  @id: "main-content"
  @class: "container"

  h1: "Welcome to Morf"
  p: "A new era of data representation."
}
```
这种方式完美遵循了“万物皆 Stage”的哲学，并保持了语法的绝对一致性。

### 3. 实践示例

#### 3.1. 表达一个数字列表

```mnl
my-list {
  1 2 3 4 5
}
```

#### 3.2. 表达一个复杂的配置文件

```mnl
; 服务配置
service {
  name: "api-gateway"
  port: 8080
  debug-mode: true

  database {
    host: "db.internal"
    user: "gateway_user"
  }
}
```

#### 3.3. 表达一个 SVG 元素 (使用 `@` 约定)

```mnl
g {
  @id: "my-group"
  @fill: "blue"
  @stroke: "black"

  rect {
    @x: 0 
    @y: 0 
    @width: 100 
    @height: 60
  }
  
  circle {
    @cx: 100 
    @cy: 60 
    @r: 40 
    @fill: "red"
  }
}
```

### 4. 可能的应用方向
#### 4.1 自定义编程语言
MNL 的语法可以以低语法噪音的表达编程语言逻辑。
```mnl
define: factorial @params: n {
  if @cond { n <= 1 }: 1
  else { n * factorial { n - 1 } }
}
```

#### 4.2 UI 布局描述
MNL 的层级结构天然地契合 UI 组件树的结构。

```mnl
window @title: "登录" {
  vbox @spacing: 10 {
    label: "用户名"
    input: @id: username

    label: "密码"
    password_input: @id: password

    hbox @align: right {
      button: "取消"
      button { @primary: true "登录" }
    }
  }
}
```

#### 4.3 场景图或实体定义
游戏引擎中的场景或 Prefab 可以用 MNL 来描述，方便人类阅读和编辑。

```mnl
entity: Player#1 {
  component: Transform {
    position: List { 0 1.8 0 }
    rotation: List { 0 0 0 }
  }
  component: CharacterController {
    speed: 5.0
    jump_force: 10.0
  }
  component: MeshRenderer {
    mesh: "player_model.obj"
    material: "player_material"
  }
}
```

#### 4.4 工作流或流程定义
```mnl
pipeline @name: "Web App CI" {
  trigger { on_push @branch: "main" }

  stage: "Build" {
    job "Compile & Bundle" {
      runner: "ubuntu-latest"
      steps {
        checkout
        run: "pnpm install"
        run: "pnpm build"
      }
    }
  }

  stage: "Deploy" {
    job "Publish to Production" {
      runner: "ubuntu-latest"
      steps {
        deploy { @target: "production_server" }
      }
    }
  }
}
```
