# EmoPal 系统功能模块实现文档

## 一、系统架构概览

EmoPal 是一个基于前后端分离架构的情绪管理与社交应用，采用以下技术栈：
- **前端**: React + TypeScript + Vite + TailwindCSS + Framer Motion
- **后端**: Node.js + Express + TypeScript
- **数据库**: MySQL（用户数据）+ MongoDB（日记、聊天、帖子等数据）
- **实时通信**: WebSocket (Socket.IO) + Redis（在线状态管理）
- **AI服务**: Google Gemini API + Coze 智能体平台

---

## 二、功能模块详细实现

### 2.1 用户认证与授权模块

#### 2.1.1 功能概述
提供用户注册、登录、JWT Token 认证、个人资料管理等功能。

#### 2.1.2 实现细节

**注册流程**:
1. 用户提交 `username`、`email`、`password`、`nickname`（可选）
2. 后端检查用户名/邮箱是否已存在
3. 密码以明文存储（生产环境应加密）
4. 创建用户账户和用户资料两条记录
5. 注册成功后自动登录

**登录流程**:
1. 用户提交 `username`（支持用户名或邮箱）和 `password`
2. 后端验证用户凭据
3. 检查账户状态（是否为 active）
4. 更新最后登录时间
5. 生成 JWT Token（24小时有效期）
6. 返回用户信息和 Token

**JWT 认证机制**:
- Token 包含: `userId`、`username`、`email`
- 有效期: 24小时
- 前端存储在 `localStorage`，包含过期时间戳
- 每5分钟检查一次 Token 是否过期
- 请求时通过 `Authorization: Bearer <token>` 和 `x-user-id` 头传递

**关键代码位置**:
- 后端控制器: `backend/src/controllers/user.controller.ts`
- 认证中间件: `backend/src/middlewares/auth.middleware.ts`
- 前端登录页: `frontend/src/pages/LoginPage.tsx`

---

### 2.2 AI 对话模块

#### 2.2.1 功能概述
提供多模式 AI 对话，包括陪伴模式、激励模式、冥想模式、逻辑分析模式和 Coze 智能体。

#### 2.2.2 实现细节

**对话模式路由**:
```typescript
const MODE_TO_MODEL_MAP = {
  companion: { type: 'gemini', modelId: 'gemini-pro' },
  motivation: { type: 'gemini', modelId: 'gemini-pro' },
  zen: { type: 'gemini', modelId: 'gemini-pro' },
  logic: { type: 'gemini', modelId: 'deepseek-chat' },
  coze: { type: 'coze' }
}
```

**AI 响应流程**:
1. 前端发送 `prompt`、`mood`、`mode`、`sessionId`
2. 后端根据模式选择 AI 服务：
   - **Gemini 模式**: 调用 Google Gemini API，传入系统指令和用户提示
   - **Coze 模式**: 调用 Coze 智能体 API，支持流式思考和分段回复
3. 返回 AI 回复（Coze 模式还返回思考过程和分段内容）

**会话管理**:
- 会话存储在 MongoDB (`chat_sessions` 集合)
- 消息存储在 MongoDB (`chat_messages` 集合)
- 支持会话的创建、重命名、归档、软删除
- 按最后活跃时间排序

**打字机效果实现**:
1. AI 回复按字符逐步显示
2. 基础速度 20ms/字符
3. 标点符号处增加延迟（逗号80ms，句号/问号150ms）
4. 多段落之间有400ms停顿

**思考链展示**:
- Coze 模式返回完整思考内容
- 前端使用打字机效果展示（10ms/字符）
- 右侧面板可展开查看思考过程

**关键代码位置**:
- 后端控制器: `backend/src/controllers/ai.controller.ts`
- Gemini 服务: `backend/src/services/gemini.service.ts`
- Coze 服务: `backend/src/services/coze.service.ts`
- 前端界面: `frontend/src/views/ChatInterface.tsx`

---

### 2.3 情绪日记模块

#### 2.3.1 功能概述
提供心情日记的创建、编辑、查看、搜索功能，支持 AI 自动情绪分析。

#### 2.3.2 实现细节

**日记创建流程**:
1. 用户输入标题和内容
2. 如果未提供 `mood_score` 或 `tags`，调用 AI 情绪分析服务
3. 保存到 MongoDB (`diaries` 集合)
4. 关联用户 ID

**AI 情绪分析实现** (`emotion-analysis.service.ts`):
- **情绪关键词映射表**: 包含10种情绪类型（开心、非常开心、难过、非常难过、生气、焦虑、平静、思考、疲惫、有动力）
- **评分算法**:
  1. 匹配文本中的情绪关键词
  2. 取匹配数最多的情绪类型作为基准
  3. 基础分数 ± 随机波动（-5到+5）
  4. 文本长度影响（>500字+3分，<100字-3分）
- **标签提取**:
  - 18个主题标签（工作、学习、生活、朋友、家人、爱情等）
  - 最多提取5个标签
  - 无匹配时默认添加"生活"、"日常"

**日历视图渲染**:
- 根据日期生成 7xN 网格
- 情绪分数映射到不同颜色：
  - 85+: 红色 `#FFB7B2`
  - 70-84: 橙色 `#FFDAC1`
  - 50-69: 绿色 `#E2F0CB`
  - 30-49: 浅绿 `#B5EAD7`
  - <30: 紫色 `#C7CEEA`

**日记详情弹窗**:
- 显示情绪分数和能量值
- 可编辑日记内容
- 显示 AI 分析的心理洞察和标签

**关键代码位置**:
- 后端控制器: `backend/src/controllers/diary.controller.ts`
- 情绪分析: `backend/src/services/emotion-analysis.service.ts`
- 前端日历: `frontend/src/views/MoodCalendar.tsx`

---

### 2.4 好友社交模块

#### 2.4.1 功能概述
提供好友搜索、添加、申请管理、实时聊天、在线状态等功能。

#### 2.4.2 实现细节

**好友申请流程**:
1. 搜索用户（按用户名/昵称）
2. 发送好友申请（存储到 `friend_requests` 集合）
3. 接收方查看申请列表
4. 接受/拒绝申请
5. 接受后在 `friendships` 表（MySQL）中创建双向好友关系

**实时聊天实现**:

**WebSocket 连接建立**:
1. 前端使用 JWT Token 连接 Socket.IO
2. 后端验证 Token，提取 `userId`
3. 建立 `userId -> socketId` 映射
4. 在 Redis 中设置在线状态
5. 通知所有好友"我上线了"

**消息发送流程**:
1. 前端调用 `wsService.sendMessage()`
2. 后端验证好友关系
3. 保存消息到 MongoDB (`friend_messages` 集合)
4. 查找接收者的 socketId
5. 如果在线，推送 `newMessage` 事件
6. 如果离线，消息已保存在数据库，等上线后拉取

**消息已读回执**:
1. 打开聊天窗口时调用 `markAsRead`
2. 批量更新该好友发来的所有未读消息为已读
3. 通知对方"消息已被读取"

**在线状态管理**:
- Redis 存储在线用户集合
- 上线/下线时通知所有好友
- 好友列表显示在线状态指示灯

**好友列表增强**:
- 显示未读消息数量
- 显示最后一条消息预览
- 按最后消息时间排序
- 实时接收新消息时将该好友置顶

**关键代码位置**:
- WebSocket 服务: `backend/src/services/websocket.service.ts`
- 好友聊天控制器: `backend/src/controllers/friend-chat.controller.ts`
- 好友申请控制器: `backend/src/controllers/friend-request.controller.ts`
- 前端好友列表: `frontend/src/views/FriendsList.tsx`
- 前端聊天界面: `frontend/src/views/ChatInterface.tsx` (mode='friend')

---

### 2.5 情感广场模块

#### 2.5.1 功能概述
提供类似朋友圈的社交功能，用户可以发布心情动态、点赞、评论。

#### 2.5.2 实现细节

**帖子发布流程**:
1. 用户输入文本内容、选择心情表情、可选上传图片
2. 后端获取用户信息（昵称、头像）
3. 创建帖子记录（存储到 `square_posts` 集合）
4. 初始化点赞数为0，点赞用户列表为空，评论列表为空

**帖子列表加载**:
1. 按创建时间倒序分页查询
2. 动态更新每个帖子的头像为数据库最新头像
3. 同时更新评论中的头像
4. 返回帖子列表和分页信息

**点赞功能**:
1. 检查用户是否已点赞（检查 `likedBy` 数组）
2. 如果已点赞：从数组移除，点赞数-1
3. 如果未点赞：添加到数组，点赞数+1
4. 保存帖子

**评论功能**:
1. 用户输入评论文本
2. 获取用户信息
3. 创建评论对象（包含 id、作者、头像、文本、时间戳）
4. 添加到帖子的 `comments` 数组
5. 保存帖子

**我的帖子**:
- 按作者ID查询用户的所有帖子
- 模态框展示历史帖子列表

**关键代码位置**:
- 后端控制器: `backend/src/controllers/square.controller.ts`
- 前端界面: `frontend/src/views/EmotionSquare.tsx`

---

### 2.6 情绪洞察模块

#### 2.6.1 功能概述
基于用户日记数据，生成每周情绪趋势图、情感分布图和个性化建议。

#### 2.6.2 实现细节

**数据计算逻辑**:

**周情绪趋势**:
1. 获取最近7天的日期
2. 查找每天的日记记录
3. 提取 `mood_score`（无日记则为0）
4. 生成数组: `[{name: '周一', value: 75}, ...]`

**情感标签分布**:
1. 遍历所有日记的 `tags` 数组
2. 统计每个标签出现次数
3. 取前5个热门标签
4. 生成饼图数据

**平均情绪分数**:
- 计算有日记日期的平均分

**可视化组件**:
- **折线图**: 使用 Recharts 的 `LineChart`，展示7天情绪波动
- **饼图**: 使用 Recharts 的 `PieChart`，展示标签分布（环形图）
- **配色方案**: `['#FFB7B2', '#FFDAC1', '#E2F0CB', '#B5EAD7']`

**个性化建议**:
根据平均分生成建议：
- ≥80: "状态极佳，适合分享快乐"
- 60-79: "情绪平稳，建议适度运动"
- 40-59: "感到压力，建议增加休息"
- <40: "需要关怀，建议和朋友聊聊或冥想"

**关键代码位置**:
- 前端界面: `frontend/src/views/InsightDashboard.tsx`
- 数据源: `frontend/src/services/diary-api.ts`

---

### 2.7 主题与个性化模块

#### 2.7.1 功能概述
提供12种预设主题和自定义颜色功能，支持全局主题切换。

#### 2.7.2 实现细节

**主题系统**:
- 使用 CSS 变量实现主题切换
- 核心变量:
  - `--bg-app`: 应用背景色
  - `--bg-sidebar`: 侧边栏背景色
  - `--color-text`: 文字颜色
  - `--color-accent`: 主色调
  - `--color-accent-soft`: 柔和背景色
  - `--color-accent-hover`: 悬停色
  - `--color-border`: 边框颜色

**预设主题**（12种）:
浅蓝、紫色、深色、黑色、橙色、绿色、粉色、青色、靛蓝、玫瑰、琥珀、自定义

**主题切换逻辑**:
1. 用户选择主题
2. 更新 `currentTheme` 状态
3. 保存到 `localStorage`
4. 通过 `style` 属性应用到根元素
5. 所有组件使用 CSS 变量自动适配

**自定义主题**:
- 颜色选择器选择主色调
- 自动生成:
  - `--color-accent`: 用户选择颜色
  - `--color-accent-soft`: 20%透明度版本
  - `--color-accent-hover`: 同主色
- 基础背景保持清新简约风格

**主题选择器 UI**:
- 弹出式网格布局（3列）
- 显示主题预览色块
- 选中状态显示图标和边框高亮
- Framer Motion 动画效果

**关键代码位置**:
- 主应用: `frontend/src/App.tsx`（themes 对象和主题逻辑）

---

### 2.8 数据管理与清理模块

#### 2.8.1 功能概述
提供自动数据清理、会话统计、数据导入导出功能。

#### 2.8.2 实现细节

**定时清理任务**:
- MongoDB 连接成功后启动
- 默认每24小时执行一次（可通过 `CLEANUP_INTERVAL_HOURS` 配置）
- 清理类型:
  1. **软删除会话**: 清理90天前标记删除的会话
  2. **归档会话**: 自动归档180天前的旧会话

**聊天统计**:
- 总会话数
- 总消息数
- 平均会话长度
- 活跃天数

**前端数据管理**:
- 导出聊天记录为 JSON
- 从 JSON 导入聊天记录
- 清空所有聊天数据
- 数据库统计信息展示

**关键代码位置**:
- 清理服务: `backend/src/services/data-cleanup.service.ts`
- Redis 服务: `backend/src/services/redis.service.ts`
- 设置页面: `frontend/src/pages/SettingsPage.tsx`

---

## 三、数据库设计（简要）

### 3.1 MySQL 数据库
- **users**: 用户账户表（id, username, email, password_hash, status）
- **user_profiles**: 用户资料表（user_id, nickname, avatar_url, bio, birthday, location, phone）
- **friendships**: 好友关系表（user_id, friend_id, status, created_at）

### 3.2 MongoDB 数据库
- **diaries**: 日记集合（user_id, title, content, mood_score, mood_emoji, tags, created_at）
- **chat_sessions**: AI 对话会话（user_id, title, mode, preview, cozeSessionId, isArchived, isDeleted, lastActiveAt）
- **chat_messages**: AI 对话消息（session_id, sender, text, mood, timestamp, isThinking）
- **friend_messages**: 好友消息（sender_id, receiver_id, text, mood, timestamp, is_read）
- **friend_requests**: 好友申请（sender_id, receiver_id, status, created_at）
- **square_posts**: 广场帖子（author, authorId, avatar, text, mood, image, likes, likedBy, comments, createdAt）

---

## 四、关键技术亮点

### 4.1 实时通信架构
- Socket.IO 实现双向实时通信
- Redis 支持多实例部署的在线状态同步
- JWT 认证 WebSocket 连接
- 消息可靠传递（离线消息持久化）

### 4.2 AI 集成策略
- 多 AI 服务路由（Gemini + Coze）
- 系统指令定制不同对话模式
- 流式思考过程展示
- 智能分段打字机效果

### 4.3 情绪分析算法
- 基于关键词匹配的轻量级情绪识别
- 多维度情绪分类（10种情绪类型）
- 文本主题标签自动提取
- 情绪评分动态计算

### 4.4 前端动画与交互
- Framer Motion 实现流畅页面切换
- 打字机效果模拟真实对话
- 响应式布局适配不同屏幕
- CSS 变量实现主题热切换

### 4.5 数据一致性保障
- 头像动态更新机制（帖子、评论实时同步最新头像）
- 会话状态管理（本地状态与服务端同步）
- Token 过期自动检测和登出

---

## 五、部署与运行

### 5.1 环境要求
- Node.js 16+
- MySQL 5.7+
- MongoDB 4.4+
- Redis 6+（可选）

### 5.2 启动命令
```bash
# 后端
cd backend
npm install
npm run dev

# 前端
cd frontend
npm install
npm run dev
```

### 5.3 环境变量
- `BACKEND_PORT`: 后端端口（默认5000）
- `FRONTEND_URL`: 前端地址
- `JWT_SECRET`: JWT 密钥
- `GEMINI_API_KEY`: Google Gemini API Key
- `COZE_API_KEY`: Coze API Key
- `CLEANUP_INTERVAL_HOURS`: 数据清理间隔（小时）

---

## 六、总结

EmoPal 系统通过整合 AI 对话、情绪日记、社交互动和数据分析，为用户提供全方位的情绪管理解决方案。系统采用现代化的前后端分离架构，结合实时通信和智能分析技术，实现了流畅的用户体验和强大的功能特性。

**核心优势**:
1. **多模式 AI 对话**: 支持5种对话模式，满足不同场景需求
2. **智能情绪分析**: 自动识别情绪和主题标签
3. **实时社交互动**: WebSocket 实现即时通讯和在线状态
4. **数据可视化**: 直观的图表展示情绪趋势
5. **高度可定制**: 12种主题+自定义颜色
6. **双数据库架构**: MySQL + MongoDB 各司其职，性能优化

本文档详细说明了各个功能模块的实现细节，可作为系统维护、二次开发和功能扩展的参考。
