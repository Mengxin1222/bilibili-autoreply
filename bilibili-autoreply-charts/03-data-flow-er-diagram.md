# B站自动回复工具 数据流E-R图 (Data Flow & E-R Diagram)

## 1. 整体数据流图

```mermaid
graph TB
    subgraph 外部实体
        Bilibili[哔哩哔哩服务器]
        User[用户/UP主]
    end

    subgraph 自动回复工具系统
        subgraph 认证模块
            Auth[登录认证]
            CookieMgr[Cookie管理]
        end

        subgraph 扫描回复模块
            Scanner[评论扫描器]
            ReplyBot[自动回复器]
            TimeFilter[时间过滤器]
        end

        subgraph 浏览器引擎
            Chrome[Chrome浏览器]
            Driver[Selenium WebDriver]
        end
    end

    subgraph 本地持久化存储
        UD[(.userdata / .cookie)]
        RT[(.refresh_token)]
        LS[(.local-storage)]
        SS[(.session-storage)]
        CUD[chrome_user_data/]
    end

    subgraph 内存状态
        RC[(replied_comments<br/>内存Set)]
        LST[(last_seen_timestamp<br/>内存datetime)]
    end

    User -->|启动命令/时间参数| Auth
    User -->|启动命令/时间参数| Scanner

    Bilibili -->|二维码URL| Auth
    Auth -->|扫码状态查询| Bilibili
    Bilibili -->|登录凭证/Cookie| Auth
    Auth -->|保存Cookie| UD
    Auth -->|保存refresh_token| RT
    Auth -->|保存localStorage| LS
    Auth -->|保存sessionStorage| SS
    Auth -->|保存Chrome配置| CUD

    UD -->|加载Cookie| CookieMgr
    RT -->|加载刷新令牌| CookieMgr
    CookieMgr -->|Cookie刷新请求| Bilibili
    Bilibili -->|新Cookie/refresh_csrf| CookieMgr
    CookieMgr -->|更新Cookie| UD
    CookieMgr -->|更新refresh_token| RT

    UD -->|注入Cookie| Driver
    LS -->|注入localStorage| Driver
    SS -->|注入sessionStorage| Driver
    CUD -->|加载用户配置| Chrome
    Driver -->|驱动浏览器| Chrome

    Chrome -->|访问评论管理页| Bilibili
    Bilibili -->|评论列表HTML| Chrome
    Chrome -->|解析DOM| Scanner

    Scanner -->|评论时间字符串| TimeFilter
    TimeFilter -->|datetime对象| Scanner
    Scanner -->|评论标识mid-时间| RC
    Scanner -->|关注状态标签| ReplyBot
    Scanner -->|更新最大时间| LST

    ReplyBot -->|点击回复/输入内容| Chrome
    Chrome -->|提交回复| Bilibili
    Bilibili -->|回复成功响应| Chrome
    Chrome -->|回复成功| ReplyBot
    ReplyBot -->|记录已回复| RC

    LST -->|时间阈值| Scanner
    RC -->|去重判断| Scanner
```

### 图表解释

#### 1. 整体概述
- 这张图是B站自动回复工具的系统级数据流图（DFD Level 0），展示了工具与B站服务器、用户之间的完整数据交互边界
- 系统分为三大核心模块：认证模块（负责登录与会话维持）、扫描回复模块（负责评论检测与自动回复）、浏览器引擎（负责页面渲染与交互）
- 数据流覆盖了从登录凭证获取、本地持久化、浏览器状态注入，到评论扫描、过滤、自动回复的全链路

#### 2. 关键元素说明
- **哔哩哔哩服务器**：外部数据源与操作目标，提供二维码、登录凭证、评论列表、接收回复请求
- **认证模块**：处理扫码登录流程，管理Cookie生命周期，支持自动刷新
- **Cookie管理**：负责本地Cookie文件的读写和过期自动刷新
- **评论扫描器**：通过Selenium驱动浏览器访问评论管理页，解析DOM提取评论信息
- **时间过滤器**：将评论时间字符串解析为datetime对象，与内存中的时间阈值比较
- **自动回复器**：根据关注状态选择回复模板，执行点击、输入、提交操作
- **内存Set（replied_comments）**：存储已回复评论的mid-时间组合标识，防止重复回复
- **内存datetime（last_seen_timestamp）**：记录上次扫描的最大评论时间，作为本轮过滤阈值

#### 3. 关键流程/关系说明
1. 用户启动工具，输入扫描起始时间参数
2. 认证模块检查本地Cookie，若无效则触发二维码登录流程，从B站获取新Cookie并持久化到本地文件
3. Cookie管理模块定期检测Cookie有效性，过期时调用刷新API获取新凭证
4. 浏览器引擎加载本地Cookie、Storage数据和Chrome用户配置，建立与B站的会话
5. 评论扫描器驱动浏览器访问评论管理页，获取评论列表HTML
6. 扫描器解析每条评论的时间、用户mid、关注状态，生成评论标识
7. 时间过滤器将字符串时间解析为datetime，与last_seen_timestamp比较，只处理新评论
8. 自动回复器根据关注状态（已关注/粉丝/陌生人）选择不同回复模板
9. 回复成功后，评论标识写入内存Set，同时更新最大时间到last_seen_timestamp

#### 4. 关键技术解释
- **双模式认证**：API模式（requests.Session）用于高效的Cookie刷新和状态查询，Selenium模式用于需要浏览器环境的扫码登录和页面操作
- **RSA+OAEP加密**：Cookie刷新时，使用B站公钥对时间戳加密生成correspond_path，获取refresh_csrf
- **内存状态管理**：replied_comments和last_seen_timestamp仅存于内存，程序重启后重新构建，依赖时间过滤避免重复处理历史评论
- **Selenium WebDriver**：通过XPath定位DOM元素，模拟真实用户的点击、输入、提交行为

#### 5. 设计意图
- **为什么要这样设计**：将认证、扫描、回复、浏览器操作分离为独立模块，各自职责清晰，便于维护和扩展
- **解决了什么痛点**：UP主需要频繁手动回复大量相似评论，工具通过自动化释放人力；Cookie自动刷新避免频繁重新登录
- **带来了什么好处**：模块化架构使得认证方式、回复策略、浏览器配置均可独立替换；内存去重+时间过滤双重机制确保不重复回复
- **如果不这样会怎样**：若所有逻辑耦合在一个脚本中，Cookie刷新、评论解析、回复操作的错误会相互影响，难以定位和修复；缺少时间过滤会导致每次扫描重复处理历史评论，效率低下

---

## 2. 详细E-R图

```mermaid
erDiagram
    USER ||--o{ COMMENT : receives
    USER ||--o{ REPLY_RECORD : "is replied to"
    COMMENT ||--o| REPLY_RECORD : "generates"
    COMMENT ||--o{ COMMENT_TAG : has
    AUTH_SESSION ||--o| COOKIE_DATA : stores
    AUTH_SESSION ||--o| REFRESH_TOKEN : stores
    AUTH_SESSION ||--o| BROWSER_STORAGE : stores

    USER {
        string mid PK "用户唯一标识"
        string username "用户名"
        string avatar_url "头像URL"
        string follow_status "关注状态:已关注/粉丝/陌生人"
    }

    COMMENT {
        string comment_id PK "mid-时间组合"
        string mid FK "评论用户mid"
        string username "评论用户名"
        string content "评论内容"
        datetime comment_time "评论时间"
        string follow_status "关注状态标签"
        boolean has_reply_tag "是否已有回复标记"
    }

    REPLY_RECORD {
        string record_id PK "同comment_id"
        string mid FK "用户mid"
        datetime reply_time "回复时间"
        string reply_content "回复内容"
        boolean is_follower "是否为粉丝/已关注"
    }

    COMMENT_TAG {
        string tag_id PK "标签标识"
        string comment_id FK "所属评论"
        string tag_type "标签类型:relation-label/ci-title-split"
        string tag_text "标签文本"
        string tag_style "style属性值"
    }

    AUTH_SESSION {
        string session_id PK "会话标识"
        string login_type "登录方式:qrcode"
        datetime created_at "创建时间"
        boolean is_valid "是否有效"
    }

    COOKIE_DATA {
        string cookie_id PK "Cookie标识"
        string session_id FK "所属会话"
        string SESSDATA "会话凭证"
        string DedeUserID "用户ID"
        string DedeUserID__ckMd5 "用户ID校验"
        string bili_jct "CSRF令牌"
        string sid "会话ID"
    }

    REFRESH_TOKEN {
        string token_id PK "令牌标识"
        string session_id FK "所属会话"
        string refresh_token "刷新令牌值"
        datetime expired_at "过期时间"
    }

    BROWSER_STORAGE {
        string storage_id PK "存储标识"
        string session_id FK "所属会话"
        string storage_type "类型:local/session"
        string key "键名"
        string value "键值"
    }

    SCAN_SESSION {
        string scan_id PK "扫描会话ID"
        datetime start_time "开始时间"
        datetime end_time "结束时间"
        int comments_scanned "扫描评论数"
        int comments_replied "回复评论数"
        datetime last_seen_timestamp "本次最大评论时间"
    }
```

### 图表解释

#### 1. 整体概述
- 该图是B站自动回复工具的详细实体-关系图（E-R图），展示了系统中涉及的核心业务实体、它们的属性以及实体间的关联关系
- 实体分为三个域：用户评论域（USER、COMMENT、REPLY_RECORD、COMMENT_TAG）、认证会话域（AUTH_SESSION、COOKIE_DATA、REFRESH_TOKEN、BROWSER_STORAGE）、扫描会话域（SCAN_SESSION）
- 图中标注了主键（PK）、外键（FK）和实体间的基数关系（一对多、一对一、多对多）

#### 2. 关键元素说明
- **USER（用户）**：评论的发布者，以mid为主键，包含用户名、头像和关注状态
- **COMMENT（评论）**：B站评论管理页中的单条评论，主键为mid与时间字符串的组合，确保同一用户多次评论也能分别记录
- **REPLY_RECORD（回复记录）**：工具对评论执行回复后生成的记录，与评论一对一对应
- **COMMENT_TAG（评论标签）**：评论DOM中的标签元素，如关注状态标签（relation-label）和回复标记（ci-title-split），用于判断评论属性
- **AUTH_SESSION（认证会话）**：一次完整的登录会话，关联Cookie、刷新令牌和浏览器存储
- **COOKIE_DATA（Cookie数据）**：具体的Cookie键值对，包括SESSDATA、DedeUserID、bili_jct等核心凭证
- **REFRESH_TOKEN（刷新令牌）**：B站Cookie刷新机制所需的令牌，用于获取新Cookie
- **BROWSER_STORAGE（浏览器存储）**：localStorage和sessionStorage中的键值数据，用于恢复浏览器状态
- **SCAN_SESSION（扫描会话）**：单次扫描周期的统计信息，记录扫描范围和处理结果

#### 3. 关键流程/关系说明
1. USER与COMMENT为一对多关系：一个用户可以发布多条评论
2. USER与REPLY_RECORD为一对多关系：一个用户可能收到多条自动回复
3. COMMENT与REPLY_RECORD为一对一关系：每条评论最多产生一条回复记录
4. COMMENT与COMMENT_TAG为一对多关系：一条评论包含多个DOM标签（关注状态、回复标记等）
5. AUTH_SESSION与COOKIE_DATA为一对多关系：一个会话包含多个Cookie字段
6. AUTH_SESSION与REFRESH_TOKEN为一对一关系：一个会话对应一个刷新令牌
7. AUTH_SESSION与BROWSER_STORAGE为一对多关系：一个会话包含多条浏览器存储记录

#### 4. 关键技术解释
- **复合主键设计**：COMMENT使用mid+时间字符串作为复合主键，而非单一自增ID，这是因为工具需要跨会话识别同一条评论，且B站评论本身不提供独立ID
- **邻接表模型**：COMMENT_TAG通过comment_id外键关联评论，支持一条评论携带多个标签，便于扩展新的标签类型
- **会话隔离**：AUTH_SESSION作为中心实体，将Cookie、刷新令牌、浏览器存储统一关联，支持多账号场景的扩展（当前为单账号）
- **内存与持久化分离**：REPLY_RECORD和SCAN_SESSION在代码中主要存在于内存，设计为实体便于理解数据逻辑；实际持久化仅依赖文件系统的Cookie和令牌

#### 5. 设计意图
- **为什么要这样设计**：将运行时内存中的逻辑概念（已回复记录、扫描会话）显式建模为实体，使数据流转关系清晰化
- **解决了什么痛点**：避免了"已回复"和"时间过滤"两个机制在代码中隐式存在、难以追踪的问题
- **带来了什么好处**：E-R图为后续引入数据库持久化（如SQLite）提供了直接的Schema设计依据；扫描会话实体支持生成统计报表
- **如果不这样会怎样**：若缺乏实体关系建模，开发者难以理解replied_comments和last_seen_timestamp与其他数据的关联，扩展功能时容易破坏数据一致性

---

## 3. 数据字典

```mermaid
graph TB
    subgraph 认证数据项
        A1["SESSDATA: 会话凭证<br/>类型: string<br/>来源: B站登录响应<br/>用途: 身份认证核心凭证"]
        A2["DedeUserID: 用户ID<br/>类型: string<br/>来源: B站登录响应<br/>用途: 标识当前登录用户"]
        A3["bili_jct: CSRF令牌<br/>类型: string<br/>来源: B站登录响应<br/>用途: 防止CSRF攻击，刷新Cookie时必需"]
        A4["sid: 会话ID<br/>类型: string<br/>来源: B站登录响应<br/>用途: 会话跟踪"]
        A5["refresh_token: 刷新令牌<br/>类型: string<br/>来源: B站二维码登录响应<br/>用途: Cookie过期时获取新凭证"]
        A6["refresh_csrf: 刷新校验码<br/>类型: string<br/>来源: B站correspond页面<br/>用途: Cookie刷新API参数"]
    end

    subgraph 评论数据项
        C1["mid: 用户唯一标识<br/>类型: string<br/>来源: DOM元素mid属性<br/>用途: 识别评论用户"]
        C2["username: 用户名<br/>类型: string<br/>来源: DOM元素card属性或文本<br/>用途: 显示和排除自身回复"]
        C3["comment_time: 评论时间<br/>类型: datetime<br/>来源: DOM date元素文本<br/>格式: YYYY-MM-DD HH:MM:SS<br/>用途: 时间过滤和排序"]
        C4["comment_id: 评论标识<br/>类型: string<br/>来源: mid + '-' + time_str<br/>示例: '123456-2025-03-25 21:27:38'<br/>用途: 内存去重唯一键"]
        C5["follow_status: 关注状态<br/>类型: string<br/>来源: relation-label标签文本<br/>取值: 已关注/粉丝/空<br/>用途: 决定回复模板"]
        C6["has_reply_tag: 回复标记<br/>类型: boolean<br/>来源: ci-title-split标签文本<br/>用途: 判断评论是否已被回复"]
    end

    subgraph 回复数据项
        R1["reply_content: 回复内容<br/>类型: string<br/>来源: 模板选择逻辑<br/>取值: '发过去了！' / '麻烦关注一下哈，不然收不到消息～'<br/>用途: 自动回复的文本内容"]
        R2["replied_comments: 已回复集合<br/>类型: Set<string><br/>来源: 内存维护<br/>用途: 防止同一会话内重复回复"]
        R3["last_seen_timestamp: 最大时间<br/>类型: datetime<br/>来源: 用户输入/内存更新<br/>用途: 过滤旧评论，只处理新评论"]
    end

    subgraph 浏览器配置数据项
        B1["user_data_dir: Chrome用户目录<br/>类型: string<br/>来源: os.getcwd() + chrome_user_data/<br/>用途: 隔离浏览器会话配置"]
        B2["local_storage: 本地存储<br/>类型: JSON对象<br/>来源: 浏览器localStorage<br/>用途: 恢复页面状态"]
        B3["session_storage: 会话存储<br/>类型: JSON对象<br/>来源: 浏览器sessionStorage<br/>用途: 恢复临时会话数据"]
    end

    subgraph 二维码登录数据项
        Q1["qr_url: 二维码URL<br/>类型: string<br/>来源: B站二维码生成API<br/>用途: 生成终端可扫描的二维码"]
        Q2["qrcode_key: 二维码密钥<br/>类型: string<br/>来源: B站二维码生成API<br/>用途: 轮询扫码状态"]
        Q3["login_code: 登录状态码<br/>类型: int<br/>来源: B站扫码状态轮询API<br/>取值: 0成功/86101待扫码/86090已扫码/86038过期<br/>用途: 驱动登录状态机"]
    end
```

### 图表解释

#### 1. 整体概述
- 该图是B站自动回复工具的数据字典，以结构化方式定义了五类核心数据项的语义、类型、来源和用途
- 数据字典覆盖认证凭证、评论信息、回复状态、浏览器配置和二维码登录五个子域
- 作为系统元数据的核心组成部分，为开发者提供统一的数据定义规范，消除理解歧义

#### 2. 关键元素说明
- **SESSDATA/DedeUserID/bili_jct/sid**：B站Cookie的四大核心字段，构成会话认证的基础
- **refresh_token/refresh_csrf**：Cookie刷新双因子，refresh_token持久化存储，refresh_csrf通过RSA加密实时获取
- **mid/username/comment_time**：评论的三元组信息，mid来自DOM属性，时间来自DOM文本
- **comment_id**：由mid和时间字符串拼接的复合标识，是内存去重集合的唯一键
- **follow_status**：从relation-label标签提取，决定使用"发过去了！"还是"麻烦关注一下哈，不然收不到消息～"
- **replied_comments/last_seen_timestamp**：内存状态数据，分别负责同会话去重和跨会话时间过滤
- **qr_url/qrcode_key/login_code**：二维码登录流程的三要素，驱动扫码状态机运转

#### 3. 关键流程/关系说明
1. 认证数据项流向：B站登录响应 → 本地文件（.userdata/.refresh_token）→ 内存Session → 浏览器Cookie注入
2. 评论数据项流向：B站评论管理页HTML → Selenium DOM解析 → 时间解析/关注状态识别 → 评论标识生成
3. 回复数据项流向：关注状态判断 → 模板选择 → 回复内容输入 → 提交成功 → 写入replied_comments集合
4. 时间过滤流程：DOM时间字符串 → datetime.strptime解析 → 与last_seen_timestamp比较 → 新评论进入回复流程
5. 浏览器配置流向：本地文件（.cookie/.local-storage/.session-storage）→ Chrome启动参数 → 页面状态恢复

#### 4. 关键技术解释
- **Cookie四字段机制**：SESSDATA是核心会话凭证，DedeUserID标识用户，bili_jct是CSRF防护令牌（所有写操作必需），sid是会话跟踪ID。四字段共同构成完整的B站认证状态
- **RSA-OAEP加密获取refresh_csrf**：使用B站公钥对`refresh_时间戳`加密，访问correspond页面获取refresh_csrf，这是B站Cookie刷新协议的安全校验环节
- **复合标识设计**：`mid-时间字符串`作为评论唯一键，利用了B站评论列表中每条评论必然有用户和时间的特性。潜在冲突：同一用户同一秒多次评论会视为同一条，但概率极低且不影响功能正确性
- **双Storage持久化**：localStorage持久化页面长期状态（如主题、配置），sessionStorage持久化临时会话状态（如当前导航位置），两者分别恢复确保页面行为一致性
- **login_code状态机**：86101（待扫码）→ 86090（已扫码待确认）→ 0（成功）/ 86038（过期），构成完整的二维码登录生命周期

#### 5. 设计意图
- **为什么要这样设计**：将分散在代码各处的数据定义集中显式化，建立团队对数据语义的一致理解
- **解决了什么痛点**：避免开发者对同一字段（如bili_jct的作用、comment_id的生成规则）理解不一致，减少集成时的接口摩擦
- **带来了什么好处**：为代码审查、故障排查、功能扩展提供直接的数据定义依据；新开发者可通过数据字典快速理解系统数据模型
- **如果不这样会怎样**：若缺乏统一数据字典，不同开发者可能对Cookie字段的用途、评论标识的生成逻辑、状态码的含义产生分歧，导致代码维护困难、Bug难以定位、新功能开发时容易破坏现有数据约定

---

*生成时间: 2026-05-14*
*模式: 深度*
*所属项目: B站自动回复工具*
*文件包含: 3 张图*
