# Mermaid 语法参考

## 1. 流程图 (Flowchart)

```mermaid
flowchart TD
    A[开始] --> B{判断}
    B -->|条件1| C[处理1]
    B -->|条件2| D[处理2]
    C --> E[结束]
    D --> E
```

### 基本语法
- `flowchart TD`：从上到下（Top Down）
- `flowchart LR`：从左到右（Left Right）
- `flowchart TB`：从上到下（Top Bottom）
- `flowchart RL`：从右到左（Right Left）
- `flowchart BT`：从下到上（Bottom Top）

### 节点形状
- `[文本]`：矩形
- `(文本)`：圆角矩形
- `((文本))`：圆形
- `{文本}`：菱形（判断）
- `[/文本/]`：平行四边形
- `[\文本\]`：平行四边形（反向）

### 箭头类型
- `-->`：实线箭头
- `---`：实线无箭头
- `-.->`：虚线箭头
- `==>`：粗线箭头
- `-->|标签|`：带标签的箭头

### 子图
```mermaid
flowchart TB
    subgraph 子图名称
        A --> B
    end
```

## 2. 时序图 (Sequence Diagram)

```mermaid
sequenceDiagram
    actor 用户
    participant 前端
    participant 后端
    participant 数据库

    用户->>前端: 点击按钮
    前端->>后端: 发送请求
    后端->>数据库: 查询数据
    数据库-->>后端: 返回结果
    后端-->>前端: 返回响应
    前端-->>用户: 展示结果
```

### 基本语法
- `actor 名称`：定义参与者（人）
- `participant 名称`：定义参与者（系统）
- `->>`：实线箭头
- `-->>`：虚线箭头（返回）
- `->>+`：激活生命线
- `-->>-`：结束激活

### 控制结构
```mermaid
sequenceDiagram
    A->>B: 请求
    alt 条件1
        B->>C: 处理1
    else 条件2
        B->>C: 处理2
    else 默认
        B->>C: 默认处理
    end
    opt 可选操作
        C->>D: 额外处理
    end
    loop 循环条件
        C->>D: 重复操作
    end
```

## 3. 类图 (Class Diagram)

```mermaid
classDiagram
    class User {
        +String id
        +String name
        +String email
        +login()
        +logout()
    }

    class Order {
        +String id
        +String userId
        +Date createTime
        +submit()
        +cancel()
    }

    User "1" --> "*" Order : 拥有
```

### 关系类型
- `-->`：关联（Association）
- `*--`：组合（Composition）
- `o--`：聚合（Aggregation）
- `--|>`：继承（Inheritance）
- `..|>`：实现（Realization）
- `..>`：依赖（Dependency）

### 可见性
- `+`：public
- `-`：private
- `#`：protected
- `~`：package

## 4. 状态图 (State Diagram)

```mermaid
stateDiagram-v2
    [*] --> 待支付
    待支付 --> 已支付: 支付成功
    待支付 --> 已取消: 超时取消
    已支付 --> 已发货: 发货
    已发货 --> 已签收: 签收
    已签收 --> [*]
    已取消 --> [*]
```

## 5. E-R 图 (Entity Relationship)

```mermaid
erDiagram
    USER ||--o{ ORDER : "下单"
    USER {
        int id PK
        string username
        string email
    }

    ORDER {
        int id PK
        int user_id FK
        string status
    }
```

### 关系基数
- `||--o{`：一对多
- `||--||`：一对一
- `}o--o{`：多对多

## 6. 用例图 (Use Case Diagram)

```mermaid
usecaseDiagram
    actor 用户
    actor 管理员

    package 系统 {
        usecase "登录" as UC1
        usecase "注册" as UC2
        usecase "管理用户" as UC3
    }

    用户 --> UC1
    用户 --> UC2
    管理员 --> UC3
    UC3 ..> UC1 : include
```

### 关系
- `-->`：关联
- `..>`：包含（include）
- `..>`：扩展（extend）

## 7. 甘特图 (Gantt)

```mermaid
gantt
    title 项目计划
    dateFormat YYYY-MM-DD
    section 阶段1
    任务1    :a1, 2024-01-01, 7d
    任务2    :after a1, 5d
    section 阶段2
    任务3    :2024-01-10, 10d
```

## 8. 饼图 (Pie)

```mermaid
pie
    title 分布图
    "A" : 30
    "B" : 50
    "C" : 20
```

## 常见错误与修复

### 1. 特殊字符问题
```
错误: A[内容[1]]
修复: A["内容[1]"]

错误: B(方法())
修复: B("方法()")
```

### 2. 括号不匹配
```
错误: subgraph 名称
      A --> B
修复: subgraph 名称
      A --> B
      end
```

### 3. 关键字错误
```
错误: sequanceDiagram
修复: sequenceDiagram

错误: classdiagram
修复: classDiagram
```

### 4. 箭头格式
```
错误: A -> B
修复: A --> B

错误: A ==> B
修复: A ==> B（仅在 flowchart 中有效）
```

## 渲染环境

- GitHub/GitLab：支持大部分 Mermaid 图表
- VS Code：安装 Mermaid 插件可预览
- Markdown 阅读器：需支持 Mermaid 渲染
- 在线编辑器：https://mermaid.live
