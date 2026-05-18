# EmoPal 数据库详细设计文档

## 1. 数据库架构概述

EmoPal 采用 **多数据库混合存储策略**，以充分利用不同数据库系统的优势：

- **MySQL (关系型数据库)**: 负责核心用户数据、账户安全及实体关系（如好友关系）。利用其事务一致性保证数据安全。
- **MongoDB (文档型数据库)**: 负责高频读写的业务内容，如日记、聊天记录、广场动态等。利用其灵活的 Schema 和水平扩展能力。
- **Redis (缓存/内存数据库)**: 负责用户在线状态实时管理及会话缓存。

---

## 2. MySQL 数据库设计 (Emo_pal)

### 2.1 表结构详细设计

#### 2.1.1 用户认证表 (`users`)
用于存储用户的登录凭证和基本账户状态。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | INT | PK, AUTO_INCREMENT | 用户唯一标识 ID |
| `username` | VARCHAR(50) | UNIQUE, NOT NULL | 用户名 |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | 邮箱地址 |
| `password_hash` | VARCHAR(255) | NOT NULL | 密码哈希值 |
| `status` | ENUM | DEFAULT 'active' | 账户状态 (active, inactive, suspended) |
| `last_login_at` | TIMESTAMP | NULLABLE | 最后登录时间 |
| `created_at` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 注册时间 |
| `updated_at` | TIMESTAMP | ON UPDATE CURRENT_TIMESTAMP | 资料更新时间 |

**索引策略**:
- `idx_username`: 加速登录查询。
- `idx_email`: 加速基于邮箱的查找。
- `idx_status`: 优化按状态筛选用户的操作。

#### 2.1.2 用户资料表 (`user_profiles`)
存储用户的详细个人信息，与 `users` 表一对一关联。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | INT | PK, AUTO_INCREMENT | 资料记录 ID |
| `user_id` | INT | FK (users.id), UNIQUE | 关联的用户 ID |
| `nickname` | VARCHAR(100) | NULLABLE | 昵称 |
| `avatar_url` | LONGTEXT | NULLABLE | 头像 URL 或 Base64 编码 |
| `avatar_type` | ENUM | DEFAULT 'default' | 头像类型 (upload, url, default) |
| `bio` | TEXT | NULLABLE | 个人简介 |
| `birthday` | DATE | NULLABLE | 生日 |
| `gender` | ENUM | NULLABLE | 性别 (male, female, other, prefer_not_to_say) |
| `location` | VARCHAR(100) | NULLABLE | 所在地 |
| `phone` | VARCHAR(20) | NULLABLE | 手机号码 |
| `mood_score_avg` | FLOAT | NULLABLE | 平均情绪评分 (冗余字段，用于快速展示) |
| `diary_count` | INT | NULLABLE | 日记总数 (冗余字段，用于快速展示) |

#### 2.1.3 好友关系表 (`friendships`)
记录用户之间的好友申请与确认状态。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | INT | PK, AUTO_INCREMENT | 关系记录 ID |
| `user_id` | INT | FK (users.id) | 发起方用户 ID |
| `friend_id` | INT | FK (users.id) | 接收方用户 ID |
| `status` | ENUM | DEFAULT 'pending' | 关系状态 (pending, accepted, rejected, blocked) |
| `created_at` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | TIMESTAMP | ON UPDATE CURRENT_TIMESTAMP | 状态更新时间 |

**索引策略**:
- `unique_friendship`: 确保同一对用户之间只有一条关系记录。
- `idx_user_status`: 优化查询某用户的所有好友。

#### 2.1.4 用户会话表 (`user_sessions`)
管理用户的登录令牌 (Token)。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | INT | PK, AUTO_INCREMENT | 会话 ID |
| `user_id` | INT | FK (users.id) | 关联用户 ID |
| `token` | VARCHAR(255) | UNIQUE, NOT NULL | 认证令牌 |
| `ip_address` | VARCHAR(45) | NULLABLE | 登录 IP 地址 |
| `user_agent` | TEXT | NULLABLE | 浏览器/设备信息 |
| `expires_at` | TIMESTAMP | NOT NULL | 过期时间 |

---

## 3. MongoDB 数据库设计

### 3.1 集合 (Collection) 详细设计

#### 3.1.1 情绪日记 (`diaries`)
存储用户的心情记录和日记内容。

- **Schema 关键字段**:
    - `user_id` (Number): 关联用户 ID。
    - `title` (String): 日记标题。
    - `content` (String): 日记正文。
    - `mood_score` (Number): 情绪评分 (0-100)。
    - `mood_emoji` (String): 情绪 Emoji 标识。
    - `tags` (Array[String]): 标签列表。
    - `is_private` (Boolean): 是否私密。
    - `is_deleted` (Boolean): 软删除标记。
- **索引**: `{ user_id: 1, is_deleted: 1, created_at: -1 }` (复合索引优化查询)。

#### 3.1.2 AI 对话会话 (`chat_sessions`)
管理用户与 AI 助手的每一次对话会话。

- **Schema 关键字段**:
    - `user_id` (Number): 关联用户 ID。
    - `title` (String): 会话标题。
    - `mode` (String): 对话模式 (如共情、理性)。
    - `coze_session_id` (String): 外部 AI 平台会话 ID。
    - `preview` (String): 消息预览。
    - `is_archived` (Boolean): 是否归档。
    - `is_deleted` (Boolean): 软删除标记。

#### 3.1.3 AI 对话消息 (`chat_messages`)
存储具体的对话内容。

- **Schema 关键字段**:
    - `session_id` (ObjectId): 关联的会话 ID。
    - `sender` (Enum): 发送者 ('user' 或 'ai')。
    - `text` (String): 消息文本。
    - `mood` (String): 识别出的情绪。
    - `is_thinking` (Boolean): 是否为思考过程记录。
- **索引**: `{ session_id: 1, timestamp: 1 }` (支持按时间顺序加载历史消息)。

#### 3.1.4 好友私聊消息 (`friend_messages`)
存储用户之间的实时聊天内容。

- **Schema 关键字段**:
    - `sender_id` (Number): 发送者 ID。
    - `receiver_id` (Number): 接收者 ID。
    - `text` (String): 消息内容。
    - `is_read` (Boolean): 已读状态。
- **索引**: `{ sender_id: 1, receiver_id: 1, timestamp: -1 }` (双向索引支持快速检索聊天记录)。

#### 3.1.5 好友申请 (`friend_requests`)
处理社交申请的中间状态。

- **Schema 关键字段**:
    - `fromUserId` (Number): 申请人 ID。
    - `toUserId` (Number): 被申请人 ID。
    - `status` (Enum): pending, accepted, rejected。
    - `message` (String): 申请留言。

#### 3.1.6 广场动态 (`square_posts`)
社区分享功能的数据存储。

- **Schema 关键字段**:
    - `authorId` (Number): 作者 ID。
    - `text` (String): 动态内容。
    - `image` (String): 配图 URL。
    - `likes` (Number): 点赞数。
    - `likedBy` (Array[Number]): 点赞用户 ID 列表。
    - `comments` (Array[Object]): 内嵌评论数组 (包含 authorId, text, timestamp)。

---

## 4. 数据库 ER 图 (Entity-Relationship Diagram)

```mermaid
erDiagram
    %% MySQL 关系型实体
    USERS ||--|| USER_PROFILES : "拥有"
    USERS ||--o{ FRIENDSHIPS : "发起/接收"
    
    %% MongoDB 文档型实体映射
    USERS ||--o{ DIARIES : "记录"
    USERS ||--o{ FRIEND_REQUESTS : "发送/接收"
    USERS ||--o{ FRIEND_MESSAGES : "发送/接收"
    USERS ||--o{ CHAT_SESSIONS : "开启"
    CHAT_SESSIONS ||--o{ CHAT_MESSAGES : "包含"
    USERS ||--o{ SQUARE_POSTS : "发布"

    %% MySQL 表定义
    USERS {
        int id PK
        varchar username UK
        varchar email UNI
        varchar password_hash
        varchar status
        datetime created_at
        datetime last_login_at
    }
    
    USER_PROFILES {
        int user_id PK, FK
        varchar nickname
        varchar avatar_url
        varchar avatar_type
        text bio
        date birthday
        varchar gender
        varchar location
        varchar phone
        float mood_score_avg
        int diary_count
    }
    
    FRIENDSHIPS {
        int user_id PK, FK
        int friend_id PK, FK
        varchar status
        datetime created_at
    }

    %% MongoDB 集合定义
    FRIEND_REQUESTS {
        object_id _id PK
        int from_user_id FK
        int to_user_id FK
        varchar status
        varchar message
        datetime created_at
    }

    FRIEND_MESSAGES {
        object_id _id PK
        int sender_id FK
        int receiver_id FK
        text text
        varchar mood
        boolean is_read
        datetime timestamp
    }

    DIARIES {
        object_id _id PK
        int user_id FK
        varchar title
        text content
        int mood_score
        varchar mood_emoji
        array tags
        boolean is_private
        datetime created_at
    }

    CHAT_SESSIONS {
        object_id _id PK
        int user_id FK
        varchar title
        varchar mode
        varchar coze_session_id
        boolean is_archived
        datetime last_active_at
    }

    CHAT_MESSAGES {
        object_id _id PK
        object_id session_id FK
        int user_id FK
        varchar sender
        text text
        varchar mood
        boolean is_thinking
        datetime timestamp
    }

    SQUARE_POSTS {
        object_id _id PK
        int author_id FK
        varchar author
        text text
        varchar image
        int likes
        array liked_by
        varchar mood
        array comments "内嵌评论"
        datetime created_at
    }
```

---

## 5. 数据库实体设计与约束说明

### 5.1 实体完整性
- **主键约束**: 所有 MySQL 表和 MongoDB 集合均定义了唯一的主键 (`id` 或 `_id`)。
- **外键约束**: MySQL 中通过 `FOREIGN KEY` 确保了用户资料、好友关系与会话数据的引用完整性。在 MongoDB 中，通过应用层逻辑维护 `user_id` 的一致性。

### 5.2 域完整性
- **数据类型**: 严格定义了字段类型，例如 `mood_score` 限制为 0-100 的数字。
- **枚举约束**: 使用 `ENUM` 类型规范了 `status`, `gender`, `sender` 等字段的取值范围。

### 5.3 性能优化设计
- **索引优化**: 
    - 针对高频查询（如获取某用户的日记列表）建立了复合索引。
    - 针对聊天记录的时序性，在 `timestamp` 字段上建立了索引。
- **软删除机制**: 在 `diaries` 和 `chat_sessions` 中使用 `is_deleted` 字段，避免物理删除带来的数据恢复困难，同时通过索引过滤已删除数据。

### 5.4 数据一致性策略
- **事务处理**: 在涉及跨表更新（如注册时同时写入 `users` 和 `user_profiles`）时，后端服务层采用事务或补偿机制确保数据最终一致性。
- **冗余字段**: 在 `user_profiles` 中冗余 `diary_count` 和 `mood_score_avg`，以减少对 MongoDB 频繁聚合查询的压力，提高首页加载速度。
