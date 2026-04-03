# 📋 实施计划：Interview Experience Review Platform

> 生成时间：2026-04-02  
> 来源：Codex 后端分析 + Gemini 前端分析 + Claude 综合规划

---

## 任务类型
- [x] 全栈 (前端 Gemini 主导 + 后端 Codex 主导)

---

## 技术方案（综合双模型分析）

### 核心架构决策

| 决策 | 选型 | 理由 |
|------|------|------|
| 前端框架 | Next.js 14 App Router | SSR 支持 Share 页 SEO，API Routes 作后端 |
| UI 组件 | shadcn/ui + Tailwind CSS | 高度可定制，无样式冲突 |
| 数据库 | PostgreSQL + Prisma | 关系型结构，类型安全 |
| 文件上传 | 浏览器直传 R2（预签名 URL） | **不经过服务器**，避免 Vercel 函数超时 |
| 异步处理 | 阶段化状态机 + 轮询 | Vercel 函数时长限制，避免 SSE 长连接成本陷阱 |
| ASR 路由 | Groq（开发/免费）→ 阿里云（中文生产）| 可插拔 Provider，env 切换 |
| AI 处理 | Claude Sonnet / GPT-4o-mini | 一次 Clean → 一次 Format（合并为两步减少成本） |
| 全局状态 | Zustand | 跨页音频播放器状态持久化 |
| 图标 | Lucide React | 统一纤细风格 |

### 设计语言（"Midnight Dev" 风格）
- **主色**：Indigo-500（AI/处理感）
- **背景**：neutral-950 暗色模式 / neutral-50 亮色
- **字体**：Geist Sans（标题/正文）+ Geist Mono（代码/时间戳）
- **圆角**：rounded-xl（柔和现代）
- **间距**：8px 基准，卡片 p-8
- 参照 Linear / Vercel Dashboard 的专业极简风

---

## 实施步骤

### Phase 1：项目脚手架（约 0.5 天）

**步骤 1.1** — 初始化 Next.js 项目
```bash
pnpm create next-app@latest . --typescript --tailwind --app --src-dir false --import-alias "@/*"
pnpm dlx shadcn@latest init
pnpm add geist                          # Geist 字体
pnpm add lucide-react                   # 图标
pnpm add zustand                        # 全局状态
pnpm add @prisma/client prisma          # 数据库
pnpm add next-auth @auth/prisma-adapter # 认证
pnpm add zod                            # 环境变量校验 + API 校验
pnpm add @aws-sdk/client-s3 @aws-sdk/s3-request-presigner  # R2 上传
```

**产物**：可运行的空 Next.js 项目

---

**步骤 1.2** — 环境变量与 `lib/env.ts`
```typescript
// lib/env.ts — 用 Zod 解析所有环境变量，全项目统一引用此文件
export const env = z.object({
  DATABASE_URL: z.string(),
  NEXTAUTH_SECRET: z.string(),
  ASR_PROVIDER: z.enum(['groq', 'openai', 'aliyun']).default('groq'),
  GROQ_API_KEY: z.string().optional(),
  ALIYUN_ACCESS_KEY_ID: z.string().optional(),
  ALIYUN_ACCESS_KEY_SECRET: z.string().optional(),
  AI_PROVIDER: z.enum(['openai', 'anthropic']).default('openai'),
  OPENAI_API_KEY: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  STORAGE_PROVIDER: z.enum(['r2', 'vercel-blob']).default('r2'),
  CF_R2_ACCOUNT_ID: z.string().optional(),
  CF_R2_ACCESS_KEY_ID: z.string().optional(),
  CF_R2_SECRET_ACCESS_KEY: z.string().optional(),
  CF_R2_BUCKET_NAME: z.string().optional(),
  CF_R2_PUBLIC_URL: z.string().optional(),
}).parse(process.env)
```

**产物**：`.env.example` + `lib/env.ts`

---

**步骤 1.3** — Prisma Schema（升级版，含 ProcessRun）
```prisma
// prisma/schema.prisma

model User {
  id         String      @id @default(cuid())
  email      String      @unique
  name       String?
  avatar     String?
  createdAt  DateTime    @default(now())
  interviews Interview[]
  favorites  Favorite[]
}

model Interview {
  id            String        @id @default(cuid())
  userId        String
  user          User          @relation(fields: [userId], references: [id])
  title         String
  company       String
  companyNorm   String        // lowercase normalized, for filter
  position      String
  positionNorm  String        // lowercase normalized
  level         String?
  date          DateTime?
  audioUrl      String?
  audioKey      String?       // R2 object key (private)
  audioSize     Int?
  audioDuration Int?
  audioChecksum String?       // SHA-256，去重
  transcript    String?
  briefContent  String?       // Markdown
  qaContent     Json?         // QA[]
  status        ProcessStatus @default(PENDING)
  tags          Tag[]
  isPublic      Boolean       @default(false)
  shareToken    String?       @unique
  viewCount     Int           @default(0)
  favorites     Favorite[]
  processRuns   ProcessRun[]
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  @@index([userId, createdAt(sort: Desc)])
  @@index([isPublic, createdAt(sort: Desc)])
  @@index([status, updatedAt(sort: Desc)])
  @@index([companyNorm, positionNorm])
}

// 处理运行记录（支持重试/审计）
model ProcessRun {
  id               String      @id @default(cuid())
  interviewId      String
  interview        Interview   @relation(fields: [interviewId], references: [id])
  stage            String      // TRANSCRIBE | CLEAN | FORMAT | QA_EXTRACT
  status           RunStatus   @default(PENDING)
  provider         String?     // groq | aliyun | openai
  providerTaskId   String?     // 阿里云异步任务 ID
  attempt          Int         @default(1)
  startedAt        DateTime?
  finishedAt       DateTime?
  lastError        String?
  nextRetryAt      DateTime?
  callbackTokenHash String?    // 阿里云 callback 验证
  rawPayload       Json?
  createdAt        DateTime    @default(now())

  @@index([interviewId, stage])
  @@index([status, nextRetryAt])
}

model Tag {
  id         String      @id @default(cuid())
  name       String      @unique
  interviews Interview[]
}

model Favorite {
  userId      String
  interviewId String
  user        User      @relation(fields: [userId], references: [id])
  interview   Interview @relation(fields: [interviewId], references: [id])
  createdAt   DateTime  @default(now())

  @@id([userId, interviewId])
  @@index([interviewId])
}

enum ProcessStatus {
  PENDING
  UPLOADING
  UPLOAD_COMPLETE
  TRANSCRIBING
  TRANSCRIPT_READY
  PROCESSING
  COMPLETED
  FAILED
}

enum RunStatus {
  PENDING
  RUNNING
  WAITING_CALLBACK
  COMPLETED
  FAILED
  RETRYING
}
```

**产物**：`prisma/schema.prisma`，执行 `pnpm prisma migrate dev --name init`

---

### Phase 2：文件上传系统（约 0.5 天）

**步骤 2.1** — 存储抽象层
```typescript
// lib/storage/types.ts
export interface StorageProvider {
  getPresignedUploadUrl(key: string, contentType: string, maxSize: number): Promise<{ url: string; fields?: Record<string, string> }>
  getPresignedDownloadUrl(key: string, expiresIn?: number): Promise<string>
  delete(key: string): Promise<void>
}

// lib/storage/r2.ts — 实现 R2 预签名上传
// lib/storage/index.ts — 工厂：根据 env.STORAGE_PROVIDER 返回对应实现
```

**步骤 2.2** — API：生成上传凭证
```typescript
// app/api/upload/presign/route.ts
// POST { filename, contentType, size } → { uploadUrl, key }
// 检查：文件大小 ≤ 500MB，格式：mp3/mp4/m4a/wav/webm/ogg/flac
// 生成 key: `audio/${userId}/${cuid()}.${ext}`
// 同时创建 Interview 记录（status=UPLOADING）
```

**步骤 2.3** — API：确认上传完成
```typescript
// app/api/upload/confirm/route.ts
// POST { interviewId, key, checksum } → 更新 status=UPLOAD_COMPLETE
// 触发异步处理（enqueue → /api/process/[id] via fetch 后台调用）
```

**步骤 2.4** — 前端 AudioUploader 组件
```typescript
// components/upload/AudioUploader.tsx
// react-dropzone + 拖拽区域 + 文件校验
// wavesurfer.js 显示波形预览（上传后）
// 上传进度条（XHR 直传至 R2 预签名 URL）
// 上传成功后触发 confirm API
```

**产物**：完整上传流程，音频直传 R2 不经过服务器

---

### Phase 3：ASR Provider 层（约 1 天）

**步骤 3.1** — 接口定义
```typescript
// lib/asr/types.ts
export interface ASRProvider {
  transcribe(audioUrl: string, options?: ASROptions): Promise<ASRResult>
  // 同步返回 or 异步（submitJob）
}
export interface ASROptions {
  language?: 'zh-CN' | 'en-US' | 'auto'
  speakerDiarization?: boolean  // 说话人分离
}
export interface ASRResult {
  transcript: string
  segments?: Array<{ start: number; end: number; text: string; speaker?: string }>
}
```

**步骤 3.2** — Groq Whisper（开发/免费）
```typescript
// lib/asr/groq.ts
// 使用 Groq SDK 调用 whisper-large-v3
// 注意：需要先下载音频 → 转 FormData → 上传给 Groq
// 输入：R2 公开 URL 或临时预签名 URL
// 同步返回 transcript
```

**步骤 3.3** — 阿里云 ASR（中文生产，异步）
```typescript
// lib/asr/aliyun.ts
// submitJob(audioUrl) → providerTaskId（存入 ProcessRun.providerTaskId）
// callback endpoint: /api/asr/callback（需验证 token）
// pollStatus(providerTaskId) → transcript | pending | failed
// 注意：音频 URL 需在提交时公开可访问（R2 公开 bucket 或长效预签名 URL）
```

**步骤 3.4** — 阿里云 Callback 端点
```typescript
// app/api/asr/callback/route.ts
// 验证 callbackTokenHash（应用层签名，非 Aliyun 原生）
// 幂等更新 ProcessRun（避免重复处理）
// 存储 rawPayload（调试用）
// 更新 transcript → 触发下一阶段（CLEAN）
```

**步骤 3.5** — Cron 轮询（兜底，防 callback 丢失）
```typescript
// app/api/cron/asr-poll/route.ts
// 查询所有 status=WAITING_CALLBACK 且 nextPollAt < now 的 ProcessRun
// 调用阿里云 query API 检查状态
// Vercel Cron: 每 2 分钟触发一次
```

**产物**：lib/asr/ 完整实现，支持 Groq 同步 + 阿里云异步

---

### Phase 4：AI 处理 Pipeline（约 1 天）

**步骤 4.1** — Prompt 模板（Markdown 文件，便于版本管理）

```markdown
<!-- lib/ai/prompts/clean.md -->
你是专业文字编辑。以下是面试录音的 ASR 转写文本，可能含口语填充词（嗯、啊、那个）、
重复短语、标点错误。请清洗为干净的书面语，保留完整语义，不添加原文没有的内容。
同时标注说话人（【面试官】/【候选人】），若无法区分则保持原样。

输入文本：
{{transcript}}

输出：清洗后的书面文本（保留说话人标注）

<!-- lib/ai/prompts/format-brief.md -->
你是技术面试经验整理专家。以下是一段面试对话。
请提取所有技术问题，为每题写简洁参考答案（100-200字）。
输出 Markdown，每题为三级标题（###），答案在标题下方。
最后添加"## 面试总结"：公司、职位、难度（1-5）、通过率评估、整体建议。

<!-- lib/ai/prompts/format-qa.md -->
从以下面试对话提取完整问答对。
输出 JSON 数组：
[{
  "question": "面试官的问题",
  "answer": "候选人的回答",
  "topic": "技术领域（如：React原理、系统设计、算法、行为问题）",
  "difficulty": 1-5,
  "hasCode": false
}]
若候选人描述了代码逻辑，将 hasCode 设为 true，answer 中用 ```语言 代码块 ``` 格式化。
```

**步骤 4.2** — Pipeline 实现（两阶段，减少成本）
```typescript
// lib/ai/pipeline.ts
// Stage 1: clean(transcript) → cleanedText（清洗 + 说话人标注）
// Stage 2a: formatBrief(cleanedText) → briefContent (Markdown)
// Stage 2b: extractQA(cleanedText) → qaContent (JSON[])
// 2a 和 2b 可并行执行
// 每个 stage 更新 Interview.status 和对应 ProcessRun
```

**步骤 4.3** — 处理触发 API
```typescript
// app/api/process/[id]/route.ts
// POST → 检查 Interview.status == UPLOAD_COMPLETE
// 幂等：若已有进行中的 ProcessRun，返回当前状态
// 创建 ProcessRun（stage=TRANSCRIBE）→ 触发 ASR
// 链式：ASR 完成 → CLEAN → FORMAT + QA_EXTRACT（并行）
```

**步骤 4.4** — 状态查询 API
```typescript
// app/api/process/[id]/status/route.ts
// GET → 返回 { status, currentStage, progress, error }
// 前端每 3 秒轮询（不用 SSE，避免 Vercel 长连接费用）
// 仅在 /dashboard 或处理中的详情页开启轮询
```

**产物**：完整 AI Pipeline，支持断点恢复

---

### Phase 5：核心页面（约 2 天）

**步骤 5.1** — 布局组件
```typescript
// components/layout/Navbar.tsx
// Logo + 导航（Home, Upload, Dashboard）+ 用户头像 + 主题切换
// 桌面端：顶部导航
// 移动端：底部 TabBar（bottom-nav 固定定位）

// components/layout/Sidebar.tsx（可选，用于搜索/筛选侧边栏）
```

**步骤 5.2** — 首页：面经列表（`app/(main)/page.tsx`）
- `InterviewCard` 卡片网格（响应式 1→2→3 列）
- 每张卡片：公司名 + 职位 + 日期 + 状态徽章 + 标签 + 简介摘要
- 顶部：搜索栏 + 筛选栏（公司 Combobox + 职位 + 难度 + 标签）
- 分页 + 骨架屏 Loading

```typescript
// components/interview/InterviewCard.tsx
// components/search/SearchBar.tsx — 带防抖的搜索输入
// components/search/FilterPanel.tsx — 公司/职位/标签筛选抽屉
// components/interview/TagBadge.tsx
```

**步骤 5.3** — 上传页（`app/(main)/upload/page.tsx`）
多步骤向导 Wizard：
1. **文件上传**：AudioUploader（拖拽 + 波形预览 + 格式/大小校验）
2. **上下文填写**：公司名、职位、级别、面试日期、初始标签
3. **AI 调优（可选）**：侧重方向（技术/行为）
4. **处理中**：ProcessingStatus 垂直步进条 + 骨架预览

```typescript
// components/upload/AudioUploader.tsx — react-dropzone + wavesurfer.js
// components/upload/ProcessingStatus.tsx — 垂直 stepper + 阶段动画
// components/upload/UploadWizard.tsx — 多步骤状态机
```

**步骤 5.4** — 面经详情页（`app/(main)/interview/[id]/page.tsx`）
- 顶部：公司 + 职位 + 日期 + 难度 + 标签
- **视图切换器**：shadcn Tabs（"面经摘要" / "完整 QA"）
- **Brief 视图**：2 列布局（左：Markdown 渲染；右：公司信息 + 笔记 + 分享）
- **QA 视图**：聊天气泡样式，面试官左对齐，候选人右对齐，hasCode 代码高亮
- 底部：粘性音频播放器（wavesurfer.js），点击 QA 条目跳转至对应时间戳
- 收藏按钮 + 分享按钮 + 公开/私密切换

```typescript
// components/interview/InterviewDetail.tsx
// components/interview/QABlock.tsx — 单条 QA，含代码高亮（shiki 或 highlight.js）
// components/interview/AudioPlayer.tsx — 粘性底部播放器（Zustand 管理状态）
// components/interview/BriefView.tsx — Markdown 渲染（react-markdown + remark-gfm）
```

**步骤 5.5** — 管理后台（`app/(main)/dashboard/page.tsx`）
- 我的面经列表（含处理状态）
- 处理失败的 → 重新触发按钮
- 批量管理：删除、公开/私密
- 统计卡片：总数、完成数、总浏览量

**步骤 5.6** — 公开分享页（`app/share/[token]/page.tsx`）
- Server Component，SEO 优化
- `generateMetadata` 动态 OG 图（vercel/og）
- 只显示 Brief + QA，无音频播放器
- 页脚 CTA："上传你的面经"

---

### Phase 6：认证与安全（约 0.5 天）

**步骤 6.1** — NextAuth.js 配置
```typescript
// app/api/auth/[...nextauth]/route.ts
// Providers: GitHub + Google
// Adapter: @auth/prisma-adapter
// Callbacks: 在 session 中附加 userId
```

**步骤 6.2** — 路由保护
```typescript
// middleware.ts — 保护 /upload, /dashboard, /api/interviews (写操作)
// Share 页（/share/[token]）无需登录
// 面经列表页（/）公开展示 isPublic=true 的面经
```

---

### Phase 7：SEO + 性能优化（约 0.5 天）

**步骤 7.1** — 动态 OG 图
```typescript
// app/opengraph-image.tsx 或 app/share/[token]/opengraph-image.tsx
// 使用 @vercel/og 生成分享图
// 内容：公司名 + 职位 + "面经分享 | Interview Review"
```

**步骤 7.2** — 性能优化
- 面经列表：分页 + `React.Suspense` + 骨架屏
- 图片：`next/image` + 公司 logo（Clearbit API）
- 字体：Geist 字体通过 `next/font` 加载
- 音频播放器：lazy load wavesurfer.js

---

## 关键文件清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `prisma/schema.prisma` | 新建 | 含 Interview + ProcessRun + Favorite 完整 Schema |
| `lib/env.ts` | 新建 | Zod 环境变量校验 |
| `lib/storage/r2.ts` | 新建 | R2 预签名上传 |
| `lib/asr/groq.ts` | 新建 | Groq Whisper 同步 ASR |
| `lib/asr/aliyun.ts` | 新建 | 阿里云 ASR 异步提交 + 轮询 |
| `lib/ai/pipeline.ts` | 新建 | 两阶段 AI 处理（Clean → Brief+QA） |
| `lib/ai/prompts/*.md` | 新建 | 三个 Prompt 模板 |
| `app/api/upload/presign/route.ts` | 新建 | 生成 R2 预签名上传 URL |
| `app/api/upload/confirm/route.ts` | 新建 | 确认上传完成，触发处理 |
| `app/api/process/[id]/route.ts` | 新建 | 触发 ASR + AI Pipeline |
| `app/api/process/[id]/status/route.ts` | 新建 | 状态轮询接口 |
| `app/api/asr/callback/route.ts` | 新建 | 阿里云 ASR 异步回调接收 |
| `app/api/cron/asr-poll/route.ts` | 新建 | 定时轮询兜底（Vercel Cron） |
| `app/(main)/page.tsx` | 新建 | 面经列表首页 |
| `app/(main)/upload/page.tsx` | 新建 | 上传向导页 |
| `app/(main)/interview/[id]/page.tsx` | 新建 | 面经详情（Brief/QA 切换） |
| `app/(main)/dashboard/page.tsx` | 新建 | 个人管理后台 |
| `app/share/[token]/page.tsx` | 新建 | 公开分享页（SEO 优化） |
| `components/upload/AudioUploader.tsx` | 新建 | 拖拽上传 + 波形预览 |
| `components/upload/ProcessingStatus.tsx` | 新建 | 垂直步进条处理状态 |
| `components/interview/QABlock.tsx` | 新建 | 聊天气泡式 QA 展示 |
| `components/interview/AudioPlayer.tsx` | 新建 | 粘性底部音频播放器 |
| `stores/playerStore.ts` | 新建 | Zustand 音频播放器全局状态 |
| `middleware.ts` | 新建 | 路由认证保护 |

---

## 风险与缓解

| 风险 | 严重程度 | 缓解措施 |
|------|---------|---------|
| Vercel 函数超时（ASR > 10s） | 高 | 浏览器直传 R2，ASR/AI 在独立后台任务中执行 |
| 阿里云 Callback 丢失 | 高 | Cron 轮询兜底，ProcessRun 存储 nextRetryAt |
| 音频格式兼容性（m4a/webm） | 高 | 前端预检格式，后端考虑 ffmpeg 转码（按需） |
| AI 处理成本失控 | 中 | 合并为两步 Pipeline，用户级每日配额，文件 checksum 去重 |
| R2 私有音频暴露 | 中 | 音频始终私有（预签名 URL），Share 页只展示文字内容 |
| 长文本 QA 提取不稳定 | 中 | 多次重试 + JSON 解析容错 + 保留原始 transcript |

---

## "Delight" 功能（后续迭代）

1. **情感热力图**：AI 分析面试片段的技术难度/紧张程度，在播放器下方显示可视化条形
2. **AI 追问**：面试主人可对 AI 提问"这道题我还能怎么回答更好？"
3. **代码块自动格式化**：QA 提取时识别代码描述并高亮显示
4. **一键导出牛客格式**：生成纯文本版本，可直接粘贴到求职论坛
5. **相似面经推荐**：同公司/同职位的其他面经侧边推荐
6. **面试日历**：在 Dashboard 中按时间轴展示面试历程

---

## 开发优先级排序

```
Week 1: Phase 1 + 2 + 3（骨架 + 上传 + Groq ASR）
Week 2: Phase 4 + 5.1~5.4（AI Pipeline + 核心页面）
Week 3: Phase 5.5~5.6 + 6 + 7（Dashboard + 认证 + 分享页 + SEO）
Week 4: 测试 + 部署 + 阿里云 ASR 接入
```

---

## SESSION_ID（供 /ccg:execute 使用）
- CODEX_SESSION: `019d4dcd-0f8f-7482-80f9-95db816e021b`
- GEMINI_SESSION: `9b8aec3b-5470-46bf-adcb-fc6d8d174537`
