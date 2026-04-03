# Interview Experience Review — 项目文档

> 变更记录 (Changelog)
>
> | 版本 | 日期 | 说明 |
> |------|------|------|
> | 0.2.0 | 2026-04-02 | 完整架构重规划：从文档型仓库升级为全栈 Web 应用，含语音转文字、AI 处理、面经展示 |
> | 0.1.0 | 2026-04-02 | 初始文档生成（架构师扫描） |

---

## 项目概述与目标

**Interview Experience Review** 是一个面试录音处理与展示平台。核心流程：上传面试录音 → 语音转文字 → AI 清洗与结构化 → 生成两种格式（简洁面经 + 详细 QA）→ 展示、搜索、分享。

**核心价值主张：**
- 将零散的面试录音自动转化为可复用、可分享的面经内容
- 提供类牛客网风格的简洁面经，同时保留完整 QA 对话
- 支持按公司、岗位、标签筛选，建立个人面试知识库

**目标用户：** 求职中的开发者、想沉淀面试经验的工程师

---

## 技术架构决策

### 技术栈选型

| 层级 | 技术选型 | 选型理由 |
|------|---------|---------|
| 前端框架 | Next.js 14 (App Router) | 同时支持 SSR/SSG，利于 SEO；API Route 可直接作为后端；生态成熟 |
| UI 组件库 | shadcn/ui + Tailwind CSS | 高度可定制、无样式冲突；组件可直接复制到项目中 |
| 后端 | Next.js API Routes | 减少运维复杂度；与前端共享类型定义 |
| 数据库 | PostgreSQL + Prisma ORM | 关系型数据适合面经结构化数据；Prisma 提供类型安全 |
| 文件存储 | Vercel Blob / Cloudflare R2 | 面试录音文件需对象存储；R2 零出口费用 |
| 认证 | NextAuth.js (Auth.js v5) | 支持 GitHub/Google OAuth；集成简单 |
| 部署 | Vercel | 与 Next.js 原生集成；免费额度够初期使用 |

### 语音转文字方案对比

| 方案 | 价格 | 准确率 | 延迟 | 中文支持 | 推荐度 |
|------|------|--------|------|---------|--------|
| **Groq Whisper Large v3** | 免费（速率限制内）/ $0.111/hr | 高 | 极快（~10s/hr音频） | 良好 | **首选（开发阶段）** |
| OpenAI Whisper API | $0.006/min | 高 | 中等 | 良好 | 备选（生产稳定） |
| 阿里云 ASR（录音文件识别） | ¥0.00035/秒 ≈ $0.003/min | 高（中文最佳） | 异步（分钟级） | 极佳 | **中文场景首选** |
| 腾讯云 ASR（录音文件识别） | ¥0.0005/秒 ≈ $0.004/min | 高（中文好） | 异步（分钟级） | 极佳 | 备选 |
| 本地 Whisper（faster-whisper） | 免费（需 GPU/CPU） | 高 | 慢（无 GPU）/ 快（有 GPU） | 良好 | 自托管场景 |

**推荐策略：**
1. 开发/测试阶段：使用 **Groq Whisper**（免费，速度最快）
2. 生产中文场景：优先 **阿里云 ASR**（中文识别最准，支持说话人分离）
3. 生产英文/多语言：**OpenAI Whisper API**（稳定性高）
4. 设计为**可插拔 Provider 模式**，通过环境变量 `ASR_PROVIDER` 切换

### AI 处理方案

| 用途 | 推荐模型 | 理由 |
|------|---------|------|
| 文本清洗 + 结构化提取 | Claude claude-sonnet-4-6 / GPT-4o-mini | 指令遵循强，格式输出稳定 |
| 生成简洁面经 | GPT-4o-mini / Gemini Flash 1.5 | 成本低，速度快，批量处理 |
| 生成详细 QA | Claude claude-sonnet-4-6 | 长上下文理解好，逻辑连贯 |

**成本估算（单次处理一段 30 分钟录音）：**
- ASR：阿里云 ≈ ¥0.63 / Groq ≈ $0
- AI 处理（~8000 tokens 输入 + 4000 tokens 输出）：GPT-4o-mini ≈ $0.008
- 存储：Cloudflare R2 ≈ $0.000015/GB/月
- **单次总成本：约 ¥0.7（含 ASR）或 ¥0.06（仅 Groq + AI）**

---

## 推荐目录结构

```
interview-experience-review/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # 认证相关页面（路由组）
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── (main)/                   # 主应用页面（路由组）
│   │   ├── layout.tsx            # 主布局（含导航栏）
│   │   ├── page.tsx              # 首页/面经列表
│   │   ├── interview/
│   │   │   ├── [id]/page.tsx     # 面经详情（简洁模式）
│   │   │   └── [id]/qa/page.tsx  # 面经详情（QA 模式）
│   │   ├── upload/page.tsx       # 上传录音页
│   │   └── dashboard/page.tsx    # 管理后台
│   └── api/                      # API Routes（后端）
│       ├── auth/[...nextauth]/   # NextAuth 路由
│       ├── interviews/
│       │   ├── route.ts          # GET 列表 / POST 创建
│       │   └── [id]/route.ts     # GET/PUT/DELETE 单条
│       ├── upload/
│       │   └── route.ts          # 上传录音文件
│       └── process/
│           └── [id]/route.ts     # 触发 ASR + AI 处理
├── components/                   # React 组件
│   ├── ui/                       # shadcn/ui 基础组件
│   ├── interview/                # 面经相关组件
│   │   ├── InterviewCard.tsx     # 面经卡片（列表项）
│   │   ├── InterviewDetail.tsx   # 面经详情
│   │   ├── QABlock.tsx           # 单条 QA 展示
│   │   └── TagBadge.tsx          # 标签徽章
│   ├── upload/                   # 上传相关组件
│   │   ├── AudioUploader.tsx     # 拖拽上传组件
│   │   └── ProcessingStatus.tsx  # 处理进度展示
│   ├── layout/                   # 布局组件
│   │   ├── Navbar.tsx
│   │   ├── Sidebar.tsx
│   │   └── Footer.tsx
│   └── search/                   # 搜索/筛选组件
│       ├── SearchBar.tsx
│       └── FilterPanel.tsx
├── lib/                          # 核心业务逻辑
│   ├── asr/                      # 语音转文字（可插拔）
│   │   ├── index.ts              # Provider 工厂
│   │   ├── groq.ts               # Groq Whisper
│   │   ├── openai.ts             # OpenAI Whisper
│   │   ├── aliyun.ts             # 阿里云 ASR
│   │   └── types.ts              # ASR 接口类型
│   ├── ai/                       # AI 处理层
│   │   ├── cleaner.ts            # 文本清洗
│   │   ├── formatter.ts          # 生成简洁面经
│   │   ├── qa-extractor.ts       # 提取 QA 对话
│   │   └── prompts/              # Prompt 模板
│   │       ├── clean.md
│   │       ├── format-brief.md
│   │       └── format-qa.md
│   ├── db/                       # 数据库层
│   │   └── prisma.ts             # Prisma client 单例
│   ├── storage/                  # 文件存储
│   │   ├── index.ts              # Storage Provider 工厂
│   │   ├── r2.ts                 # Cloudflare R2
│   │   └── vercel-blob.ts        # Vercel Blob
│   └── utils/                    # 工具函数
│       ├── format.ts
│       └── validate.ts
├── prisma/                       # 数据库 Schema
│   ├── schema.prisma
│   └── migrations/
├── types/                        # TypeScript 全局类型
│   └── index.ts
├── hooks/                        # React 自定义 hooks
│   ├── useInterviews.ts
│   └── useUpload.ts
├── stores/                       # 客户端状态（Zustand）
│   └── uploadStore.ts
├── public/                       # 静态资源
├── .env.example                  # 环境变量模板
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## 模块结构图

```mermaid
graph TD
    A["(根) Interview Experience Review"] --> B["app/"]
    A --> C["components/"]
    A --> D["lib/"]
    A --> E["prisma/"]

    B --> B1["(auth)/ 认证页面"]
    B --> B2["(main)/ 主页面"]
    B --> B3["api/ 后端接口"]

    B2 --> B21["page.tsx 面经列表"]
    B2 --> B22["interview/[id] 面经详情"]
    B2 --> B23["upload/ 上传页"]
    B2 --> B24["dashboard/ 管理后台"]

    B3 --> B31["api/interviews 面经 CRUD"]
    B3 --> B32["api/upload 文件上传"]
    B3 --> B33["api/process ASR+AI 触发"]

    C --> C1["ui/ shadcn 基础组件"]
    C --> C2["interview/ 面经展示组件"]
    C --> C3["upload/ 上传组件"]
    C --> C4["search/ 搜索筛选组件"]

    D --> D1["asr/ 语音转文字层"]
    D --> D2["ai/ AI 处理层"]
    D --> D3["db/ 数据库层"]
    D --> D4["storage/ 文件存储层"]

    D1 --> D11["groq.ts Groq Whisper"]
    D1 --> D12["aliyun.ts 阿里云 ASR"]
    D1 --> D13["openai.ts OpenAI Whisper"]

    D2 --> D21["cleaner.ts 文本清洗"]
    D2 --> D22["formatter.ts 简洁面经"]
    D2 --> D23["qa-extractor.ts QA提取"]

    E --> E1["schema.prisma 数据模型"]
    E --> E2["migrations/ 迁移文件"]
```

---

## 模块索引

| 模块路径 | 语言/类型 | 状态 | 一句话职责 |
|----------|-----------|------|------------|
| `./app/(main)/` | Next.js TSX | 规划中 | 面经列表、详情、上传、管理页面 |
| `./app/api/` | TypeScript | 规划中 | RESTful API 接口，处理业务逻辑 |
| `./components/interview/` | React TSX | 规划中 | 面经展示相关 UI 组件 |
| `./components/upload/` | React TSX | 规划中 | 音频上传与处理状态 UI |
| `./lib/asr/` | TypeScript | 规划中 | 可插拔语音转文字 Provider 层 |
| `./lib/ai/` | TypeScript | 规划中 | AI 文本清洗、面经格式化、QA 提取 |
| `./lib/db/` | TypeScript | 规划中 | Prisma ORM 数据库访问层 |
| `./lib/storage/` | TypeScript | 规划中 | 音频文件对象存储抽象层 |
| `./prisma/` | Prisma Schema | 规划中 | 数据库 Schema 与迁移管理 |

---

## 数据模型设计

### 核心表结构（Prisma Schema 草案）

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  avatar    String?
  createdAt DateTime @default(now())
  interviews Interview[]
}

model Interview {
  id          String          @id @default(cuid())
  userId      String
  user        User            @relation(fields: [userId], references: [id])
  title       String          // "字节跳动 前端 2026-03"
  company     String
  position    String          // "前端工程师"
  level       String?         // "P6", "T3", "社招"
  date        DateTime?       // 面试日期
  audioUrl    String?         // 原始录音文件 URL
  audioSize   Int?            // 文件大小（字节）
  audioDuration Int?          // 时长（秒）
  transcript  String?         // 原始转写文本
  briefContent String?        // 简洁面经（Markdown）
  qaContent   Json?           // QA 对话（JSON 数组）
  status      ProcessStatus   @default(PENDING)
  tags        Tag[]
  isPublic    Boolean         @default(false)
  shareToken  String?         @unique
  viewCount   Int             @default(0)
  createdAt   DateTime        @default(now())
  updatedAt   DateTime        @updatedAt
}

model Tag {
  id         String      @id @default(cuid())
  name       String      @unique
  interviews Interview[]
}

enum ProcessStatus {
  PENDING      // 待处理
  UPLOADING    // 上传中
  TRANSCRIBING // 转写中
  PROCESSING   // AI 处理中
  COMPLETED    // 完成
  FAILED       // 失败
}
```

---

## 关键接口设计

### API 端点清单

```
GET    /api/interviews              # 面经列表（支持分页、筛选）
POST   /api/interviews              # 创建面经记录
GET    /api/interviews/:id          # 获取单条面经
PUT    /api/interviews/:id          # 更新面经信息
DELETE /api/interviews/:id          # 删除面经

POST   /api/upload                  # 上传音频文件（返回存储 URL）
POST   /api/process/:id             # 触发 ASR + AI 处理流程
GET    /api/process/:id/status      # 查询处理状态（轮询 or SSE）

GET    /api/interviews/share/:token # 通过分享 token 获取面经（无需登录）
```

### 处理流程（Pipeline）

```
用户上传音频
    → POST /api/upload（存入 R2/Vercel Blob）
    → POST /api/process/:id（触发异步处理）
        → ASR Provider（Groq/阿里云/OpenAI）→ transcript
        → AI Cleaner → 清洗后文本
        → AI Formatter → briefContent（简洁面经）
        → AI QA Extractor → qaContent（QA 数组）
    → 更新 Interview.status = COMPLETED
    → 前端轮询 /api/process/:id/status 或 SSE 推送
```

---

## 运行与开发

### 环境要求

- Node.js >= 18.17.0
- PostgreSQL >= 14（本地开发可用 Docker）
- pnpm >= 8（推荐）或 npm/yarn

### 快速启动

```bash
# 克隆项目
git clone git@github.com:hzqsns/Interview_Experience_Review.git
cd Interview_Experience_Review

# 安装依赖
pnpm install

# 配置环境变量
cp .env.example .env.local
# 编辑 .env.local，填入数据库连接、API Key 等

# 初始化数据库
pnpm prisma migrate dev

# 启动开发服务器
pnpm dev
```

### 环境变量（.env.example）

```bash
# 数据库
DATABASE_URL="postgresql://user:password@localhost:5432/interview_review"

# NextAuth
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="your-secret-key"
GITHUB_CLIENT_ID=""
GITHUB_CLIENT_SECRET=""

# ASR Provider（选择一个）
ASR_PROVIDER="groq"  # groq | openai | aliyun | tencent
GROQ_API_KEY=""
OPENAI_API_KEY=""
ALIYUN_ACCESS_KEY_ID=""
ALIYUN_ACCESS_KEY_SECRET=""

# AI 处理
AI_PROVIDER="openai"  # openai | anthropic | google
ANTHROPIC_API_KEY=""

# 文件存储
STORAGE_PROVIDER="r2"  # r2 | vercel-blob
CLOUDFLARE_R2_ACCOUNT_ID=""
CLOUDFLARE_R2_ACCESS_KEY_ID=""
CLOUDFLARE_R2_SECRET_ACCESS_KEY=""
CLOUDFLARE_R2_BUCKET_NAME=""
CLOUDFLARE_R2_PUBLIC_URL=""
```

### 常用脚本

```bash
pnpm dev          # 启动开发服务器（localhost:3000）
pnpm build        # 构建生产版本
pnpm start        # 启动生产服务器
pnpm lint         # ESLint 检查
pnpm typecheck    # TypeScript 类型检查
pnpm prisma studio  # 打开 Prisma Studio（数据库可视化）
pnpm prisma migrate dev  # 应用数据库迁移
```

---

## 开发路线图

### 里程碑 1：基础骨架（约 1 周）

- [ ] Next.js 14 项目初始化（`pnpm create next-app`）
- [ ] 配置 shadcn/ui、Tailwind CSS
- [ ] 配置 Prisma + PostgreSQL，完成数据库 Schema
- [ ] 配置 NextAuth.js（GitHub OAuth）
- [ ] 基础页面路由搭建（首页、上传页、详情页）

### 里程碑 2：核心上传与处理流程（约 1.5 周）

- [ ] 音频文件上传组件（支持拖拽，限制格式/大小）
- [ ] 文件上传 API Route（存入 Cloudflare R2）
- [ ] ASR Provider 层实现（先接入 Groq Whisper）
- [ ] AI 处理 Pipeline（清洗 → 面经 → QA）
- [ ] Prompt 模板编写与调优
- [ ] 处理状态轮询/SSE 推送

### 里程碑 3：展示与搜索（约 1 周）

- [ ] 面经列表页（分页、卡片展示）
- [ ] 面经详情页（简洁面经 + QA 切换）
- [ ] 搜索与筛选（公司、岗位、标签）
- [ ] 标签系统
- [ ] 分享链接生成

### 里程碑 4：管理后台与优化（约 1 周）

- [ ] 个人管理后台（我的面经列表、状态管理）
- [ ] 收藏功能
- [ ] 接入阿里云 ASR（中文场景优化）
- [ ] 响应式移动端适配
- [ ] SEO 优化（面经公开页的 metadata）

### 里程碑 5：生产就绪（按需）

- [ ] Vercel 部署配置
- [ ] 错误监控（Sentry）
- [ ] 数据库连接池（PgBouncer 或 Supabase）
- [ ] Rate Limiting（防止 API 滥用）
- [ ] 自动化测试（Vitest + Playwright）

---

## 测试策略

| 测试类型 | 工具 | 覆盖目标 |
|---------|------|---------|
| 单元测试 | Vitest | `lib/asr/`、`lib/ai/` 核心逻辑 |
| 集成测试 | Vitest + MSW | API Routes、数据库交互 |
| E2E 测试 | Playwright | 上传流程、面经展示、搜索筛选 |
| 类型检查 | TypeScript strict | 全项目 |

**测试目录结构：**

```
__tests__/
├── unit/
│   ├── lib/asr/groq.test.ts
│   └── lib/ai/formatter.test.ts
├── integration/
│   └── api/interviews.test.ts
└── e2e/
    ├── upload.spec.ts
    └── browse.spec.ts
```

---

## 编码规范与约定

### TypeScript

- 使用 `strict: true` 模式，禁用 `any`（必要时用 `unknown` + 类型守卫）
- API Route 的请求/响应均需定义 Zod schema 并验证
- 组件 Props 使用 `interface` 定义，非 `type`（便于扩展）
- 文件导出：每个文件一个默认导出（组件），工具函数用具名导出

### 命名约定

| 类别 | 规范 | 示例 |
|------|------|------|
| 组件文件 | PascalCase | `InterviewCard.tsx` |
| 普通文件 | kebab-case | `qa-extractor.ts` |
| 组件名 | PascalCase | `<InterviewCard />` |
| 变量/函数 | camelCase | `fetchInterviews` |
| 常量 | UPPER_SNAKE_CASE | `ASR_TIMEOUT_MS` |
| 数据库表字段 | camelCase (Prisma) | `briefContent` |
| API 路径 | kebab-case | `/api/process-status` |

### 目录约定

- `components/` 只放 UI 组件，业务逻辑提取到 `lib/` 或 hooks
- `lib/asr/` 和 `lib/storage/` 必须面向接口编程，通过 `index.ts` 暴露统一 factory
- Prompt 模板统一放在 `lib/ai/prompts/`，使用 `.md` 文件，方便版本管理和调优
- 环境变量统一在 `lib/env.ts` 中用 Zod 解析校验，全项目引用 `lib/env.ts` 而非直接读 `process.env`

### Git 提交规范

使用 Conventional Commits：

```
feat: 添加 Groq Whisper ASR 支持
fix: 修复上传文件大小校验逻辑
docs: 更新 ASR Provider 文档
refactor: 重构 AI Pipeline 为异步队列
chore: 更新依赖版本
```

---

## AI 使用指引

### Prompt 编写原则

1. 所有 Prompt 模板存放在 `lib/ai/prompts/` 目录，以 Markdown 格式维护
2. Prompt 中明确指定输出格式（JSON Schema 或 Markdown 结构）
3. 对 ASR 转写文本，先进行"降噪"处理（去除口语填充词、修正标点），再做结构化提取
4. QA 提取时，指引 AI 区分"面试官提问"和"候选人回答"

### 关键 Prompt 模板（草案）

**文本清洗（`lib/ai/prompts/clean.md`）：**
```
你是一个专业的文字编辑。以下是一段面试录音的 ASR 转写文本，
可能包含口语化表达、重复词、填充词（嗯、啊、那个）、
标点错误等问题。请清洗文本，保留完整语义，输出干净的书面语体。
不要添加任何原文没有的内容。
```

**面经格式化（`lib/ai/prompts/format-brief.md`）：**
```
你是一个技术面试经验整理专家。以下是一段面试对话的清洗文本。
请提取所有面试题目，为每个题目写一个简洁的参考答案（100-200字）。
输出格式为 Markdown，每个题目作为三级标题（###），答案在标题下方。
```

**QA 提取（`lib/ai/prompts/format-qa.md`）：**
```
请从以下面试对话中提取完整的问答对。
输出 JSON 数组，格式：[{"question": "...", "answer": "...", "topic": "..."}]
其中 topic 为技术领域标签（如 "React 原理"、"系统设计"、"算法"）。
```

---

## 架构总览

```
Browser
  │
  ▼
Next.js App (Vercel)
  ├── App Router Pages      ← SSR/SSG 页面渲染
  └── API Routes            ← 后端逻辑
        ├── Auth (NextAuth)
        ├── Interview CRUD  ← Prisma → PostgreSQL
        ├── Upload          ← → Cloudflare R2
        └── Process         ← ASR Provider + AI Provider
              ├── Groq Whisper / 阿里云 ASR / OpenAI Whisper
              └── OpenAI GPT-4o-mini / Claude claude-sonnet-4-6
```

---

## 变更记录 (Changelog)

| 版本 | 日期 | 说明 |
|------|------|------|
| 0.2.0 | 2026-04-02 | 完整架构重规划：全栈 Web 应用，含 ASR、AI 处理、面经展示 |
| 0.1.0 | 2026-04-02 | 初始文档生成（架构师扫描） |
