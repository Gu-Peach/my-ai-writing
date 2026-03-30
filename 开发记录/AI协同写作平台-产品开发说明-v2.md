# AI 协同写作平台 — 产品开发说明文档

> **版本**: v2.0  
> **状态**: 规划中  
> **预计总周期**: 11–12 周

---

## 目录

1. [项目背景](#一项目背景)
2. [项目目标与范围](#二项目目标与范围)
3. [整体技术架构](#三整体技术架构)
4. [核心功能模块](#四核心功能模块)
5. [AI 能力实现方案](#五ai-能力实现方案)
6. [协同编辑技术架构](#六协同编辑技术架构)
7. [房间权限与成员管理方案](#七房间权限与成员管理方案)
8. [网关层方案](#八网关层方案)
9. [数据库设计](#九数据库设计)
10. [开发里程碑与阶段规划](#十开发里程碑与阶段规划)
11. [技术难点与解决方案](#十一技术难点与解决方案)
12. [成功指标](#十二成功指标)

---

## 一、项目背景

当前已有一套基于 **Python + LangGraph + MCP** 的 AI 写作服务，具备以下核心能力：

- 通过 LangGraph 构建多节点工作流（路由 → 大纲生成 → 资料检索 → 文档撰写）
- 通过 FastMCP 标准化工具调用（网络搜索、文档生成、内容增强等）
- 通过 SSE（Server-Sent Events）将 AI 生成内容实时推流至前端
- 支持 Prompt 模板化管理、分节并发生成、MCP Progress Notification 进度推送

现有前端采用 **Vue**，功能较为分散，缺乏现代化协同编辑体验。为此，本项目决定基于 **Next.js 15 全栈框架**对前端进行重构，并在保留现有 Python AI 服务的基础上，新增以下核心能力：

- **统一网关层**（Nginx 反向代理，统一入口、限流、SSL）
- **实时多人协同编辑**（基于 Yjs CRDT + TipTap）
- **房间权限管理**（角色权限配置、成员邀请与踢出）
- **AI 生成内容与协同编辑无缝集成**（SSE 流式插入 + Yjs 事务同步）
- **版本历史管理**（YDoc 快照 + 版本对比回滚）

本文档是对上述改造目标的完整产品开发说明，涵盖功能定义、技术实现方案和开发里程碑，供全栈开发团队执行参考。

---

## 二、项目目标与范围

### 2.1 核心目标

| 目标 | 说明 |
|------|------|
| 统一网关 | Nginx 作为唯一入口，统一限流、SSL、反向代理 |
| 前端现代化重构 | Vue → Next.js 15（App Router），TypeScript 全量类型安全 |
| 实时协同编辑 | 多用户同时编辑同一文档，毫秒级光标同步，自动冲突解决 |
| 房间权限管理 | 支持角色级别的编辑权限配置、邀请成员、踢出成员 |
| AI 能力集成 | 保留 Python AI 服务，通过 BFF 层转发 SSE；AI 内容实时写入编辑器 |
| 版本管理 | 基于 Yjs 快照的版本历史，支持对比与回滚 |

### 2.2 移除模块

- ~~AI PPT 生成功能~~（已从当前规划中移除，后续可按需追加）

### 2.3 技术栈总览

**网关层**

- Nginx（反向代理、限流、SSL 终止、CORS 统一处理）

**前端层**

- Next.js 15（App Router）+ React 18 + TypeScript
- Tailwind CSS + shadcn/ui
- Zustand（状态管理）、React Query（数据获取）

**协同编辑层**

- Yjs（CRDT 数据结构，自动冲突解决）
- y-websocket（WebSocket 提供者）
- TipTap Editor（基于 ProseMirror 的富文本编辑器）
- y-prosemirror（Yjs ↔ TipTap 绑定）

**后端层**

- Next.js API Routes / Server Actions（BFF）
- Prisma ORM + PostgreSQL（元数据持久化）
- NextAuth.js（认证鉴权）
- Redis（YDoc 实时持久化 + 缓存）
- WebSocket Server（协同连接管理，内嵌于 Next.js Custom Server）

**AI 服务层（保留现有 Python 服务）**

- FastAPI + LangGraph（工作流编排）
- LangChain（LLM 调用 + 工具包装）
- FastMCP（MCP 工具服务）
- SSE（流式输出）

---

## 三、整体技术架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                            │
│   TipTap Editor  ↔  Yjs Doc (CRDT)  ↔  WebSocket Provider      │
└───────────────────────┬─────────────────────┬───────────────────┘
                        │ HTTP / SSE           │ WebSocket
                        ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx 网关层                                │
│                                                                 │
│   /api/*  /ws/*  /ai/*                                          │
│   ├── 限流（按 IP / 按用户）                                    │
│   ├── SSL 终止                                                  │
│   ├── CORS 统一处理                                             │
│   └── 反向代理路由                                              │
└───────┬──────────────────┬──────────────────┬───────────────────┘
        │ /api/*  /ws/*    │                  │ /ai/*
        ▼                  ▼                  ▼
┌───────────────┐  ┌──────────────┐  ┌────────────────────────┐
│ Next.js BFF   │  │ WebSocket    │  │ Python AI Service      │
│ API Routes    │  │ Server       │  │ FastAPI + LangGraph     │
│ (Auth, CRUD,  │  │ (Yjs 协同   │  └──────────┬─────────────┘
│  SSE Proxy)   │  │  连接管理)   │             │ MCP Protocol
└───────┬───────┘  └──────┬───────┘  ┌──────────▼─────────────┐
        │                 │          │ Python MCP Service      │
        │          ┌──────▼──────┐   │ FastMCP                 │
        │          │   Redis     │   └──────────┬─────────────┘
        │          │ (YDoc 持久化│             │
        │          │  + 缓存)    │          LLM API
        │          └─────────────┘    (OpenAI / Claude)
        │
┌───────▼───────┐
│  PostgreSQL   │
│  (元数据 +    │
│   版本快照)   │
└───────────────┘
```

---

## 四、核心功能模块

### 4.1 用户认证与管理

**功能点：**

- 邮箱注册 / 登录
- OAuth 登录（Google、GitHub）
- 用户信息管理与头像上传（MinIO / S3）

**实现方式：**

- NextAuth.js 处理认证流程
- Prisma 管理用户数据
- MinIO 对象存储存放头像及附件

---

### 4.2 AI 文档生成

**功能点：**

- 主题输入与配置（目标读者、文档风格、章节数量）
- 实时流式生成（SSE），支持大纲逐节展开
- Markdown 渲染与预览（TipTap 内渲染）
- 大纲可视化编辑（生成前可调整结构）
- 导出 Word / PDF

**核心工作流（LangGraph）：**

```
START
  → Function Router（路由判断：文档类型）
  → Outline Node（大纲生成 + 用户确认）
  → Research Node（ReAct Agent 调用 MCP 搜索工具）
  → Writer Reporter Node（分节并发生成正文）
END
```

**SSE 事件类型：**

| 事件类型 | 说明 |
|----------|------|
| `message` | LLM 流式输出的文本 Delta |
| `tool_call` | MCP 工具调用开始 |
| `tool_result` | 工具调用完成及结果 |
| `progress` | MCP Progress Notification（章节生成进度）|
| `done` | 全流程结束 |

---

### 4.3 实时协同编辑

**功能点：**

- 多用户同时编辑文档，毫秒级实时同步
- 彩色光标与选区（每位协作者独立颜色）
- 在线状态感知（Awareness API）
- 操作冲突自动解决（CRDT，无需手动合并）
- 离线编辑，网络恢复后自动同步（IndexedDB 本地缓存）
- AI 生成期间编辑区域锁定机制

---

### 4.4 房间权限管理（新增）

**功能点：**

- **权限配置**：房间 Owner 可设置哪些角色允许编辑（如仅 Editor 可编辑，Viewer 只读；或所有人均可编辑）
- **成员邀请**：通过邮箱邀请指定用户，指定其加入角色（Editor / Viewer）；或生成带角色的邀请链接
- **踢出成员**：Owner 可移除任意协作者；Editor 可移除 Viewer；被踢出后 WebSocket 连接立即强制断开，前端跳转至无权限页面
- **成员列表**：实时展示当前在线 / 离线成员及其角色
- **权限变更实时生效**：Owner 修改成员角色或权限配置后，所有在线客户端立即感知，无需重新连接

详细实现见第七章。

---

### 4.5 图表生成

**功能点：**

- 数据表格输入
- 图表类型选择（柱状图、折线图、饼图等）
- ECharts 可视化渲染
- PNG / SVG 导出
- 图表作为富文本节点嵌入文档

---

### 4.6 文档管理与协作

**功能点：**

- 文档列表、新建、编辑、删除、收藏
- 协作者邀请（通过邮箱或链接）
- 权限控制：Owner / Editor / Viewer 三级
- 分享链接生成（只读或可编辑，支持设置过期时间）
- 文件上传与引用管理（图片插入文档）

---

### 4.7 版本历史

**功能点：**

- 自动定期快照（按时间间隔或操作次数触发）
- 版本列表展示与版本标签
- 版本内容对比（diff 可视化）
- 版本回滚（恢复到指定快照）

---

## 五、AI 能力实现方案

### 5.1 LangGraph 工作流

#### 状态管理（State Schema）

```python
# state_types.py
from langgraph.graph import MessagesState
from typing import Annotated

def set_value(old, new):
    return new

class State(MessagesState):
    task_name: Annotated[str, set_value]
    final_outline: str
    final_plan: str
    final_report: Any
    execution_messages: list[AnyMessage]
    quotations: list[Quotation]
    user_id: str
    workspace_id: str
    workflow: str
    next: str
```

关键特性：
- `MessagesState` 内置消息管理
- `Annotated[type, reducer]` 自定义状态更新逻辑
- Checkpointer 持久化（SQLite / PostgreSQL），支持中断恢复

#### 图构建（Graph Builder）

```python
# builder.py
def build_graph():
    builder = StateGraph(State)
    builder.add_node(NODE_FUNCTION_ROUTER, function_router_node)
    builder.add_node(NODE_OUTLINE, build_outline_graph())  # 嵌套子图
    builder.add_node(NODE_RESEARCH, researcher_node)
    builder.add_node(NODE_WRITER_GENERATOR_REPORTER, writer_reporter_node)

    builder.add_edge(START, NODE_FUNCTION_ROUTER)
    builder.add_conditional_edges(NODE_OUTLINE, route_by_outline_result)
    builder.add_conditional_edges(NODE_RESEARCH, route_by_research_result)

    return builder.compile(
        checkpointer=get_sqlite_saver("checkpoints.db"),
        name="main"
    )
```

#### ReAct Agent（工具调用）

Research Node 采用预构建的 ReAct Agent 驱动 MCP 工具调用：

```python
async def get_react_agent(tools, state_schema, prompt):
    return create_react_agent(
        model=get_llm_by_type(LLMType.BASIC),
        state_schema=state_schema,
        tools=tools,
        prompt=prompt
    )
```

ReAct 执行循环：LLM 选择工具 → 调用工具 → 观察结果 → 继续推理，直至得出最终答案。

---

### 5.2 MCP 服务架构

#### MCP Server（FastMCP）

```python
# search.py
mcp = FastMCP("Search", stateless_http=True)

@standard_tool(mcp)
async def search_html(query: str, max_results: int, usages: list[Usage]) -> SearchResults:
    search_tool = TavilySearch(max_results=max_results)
    response = await search_tool.ainvoke({"query": query})
    return SearchResults(query=query, results=response)
```

**已实现的 MCP 工具服务：**

| 服务名 | 功能 |
|--------|------|
| `search` | Tavily + Jina 网络搜索 |
| `write` | 文档正文生成（流式 + Progress Notification）|
| `enhancement` | 内容增强（改写、摘要、扩写）|
| `ingest` | 数据摄入（文件解析）|
| `deepsearch` | 深度研究（多轮搜索 + 综合）|

#### MCP Progress Notification（流式进度推送）

```python
# write.py（MCP Server 端）
@standard_tool(mcp)
async def generate_report(outline: str, references: list, ctx: Context):
    async for chunk in llm.astream(messages):
        await ctx.session.send_progress_notification(
            ctx.request_context.meta.progressToken,
            progress=1, total=1,
            message=json.dumps({"type": "html", "content": chunk.content})
        )
```

---

### 5.3 Next.js BFF 层（SSE 转发）

```typescript
// app/api/documents/[id]/generate/route.ts
export async function POST(req: Request, { params }: { params: { id: string } }) {
  const { prompt } = await req.json()

  const aiResponse = await fetch('http://ai-service:8000/task/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prompt })
  })

  const stream = new ReadableStream({
    async start(controller) {
      const reader = aiResponse.body?.getReader()
      const decoder = new TextDecoder()
      while (true) {
        const { done, value } = await reader!.read()
        if (done) break
        const chunk = decoder.decode(value)
        for (const line of chunk.split('\n')) {
          if (line.startsWith('data: ')) {
            const data = JSON.parse(line.slice(6))
            if (data.event === 'message') {
              controller.enqueue(
                new TextEncoder().encode(`data: ${JSON.stringify(data)}\n\n`)
              )
            }
          }
        }
      }
      controller.close()
    }
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive'
    }
  })
}
```

---

### 5.4 前端 AI 生成 Hook

```typescript
// hooks/useAIGeneration.ts
export function useAIGeneration(editor: Editor | null, ydoc: Y.Doc, docId: string) {
  const generateContent = async (prompt: string) => {
    const eventSource = new EventSource(
      `/api/documents/${docId}/generate?prompt=${encodeURIComponent(prompt)}`
    )

    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data)
      if (data.event === 'message') {
        ydoc.transact(() => {
          editor?.commands.insertContent(data.data.delta.content)
        })
      }
      if (data.event === 'tool_call') {
        showToolCallStatus(data.data.tool_name)
      }
    }

    eventSource.onerror = () => eventSource.close()
  }

  return { generateContent }
}
```

---

## 六、协同编辑技术架构

### 6.1 架构分层

```
Client Layer
  TipTap Editor  ↔  Yjs Doc (CRDT)  ↔  WebSocket Provider
                            ↕ WebSocket
WebSocket Server
  Connection Manager  ↔  YDoc Manager  ↔  Auth + Permission Middleware
                            ↕
         Redis (YDoc 实时持久化)   PostgreSQL (元数据、版本)
```

### 6.2 WebSocket Server 实现

```typescript
// server.ts（Next.js Custom Server）
const wss = new WebSocketServer({ server, path: '/api/collab' })

wss.on('connection', async (ws, req) => {
  // 1. 鉴权
  const auth = await authenticateCollabConnection(ws, req)
  if (!auth) return

  // 2. 权限校验（是否有编辑权）
  const canEdit = await checkEditPermission(auth.documentId, auth.userId)

  setupWSConnection(ws, req, {
    persistence: {
      bindState: async (docName, ydoc) => {
        const data = await redis.get(`ydoc:${docName}`)
        if (data) Y.applyUpdate(ydoc, Buffer.from(data, 'base64'))
      },
      writeState: async (docName, ydoc) => {
        const update = Y.encodeStateAsUpdate(ydoc)
        await redis.set(`ydoc:${docName}`, Buffer.from(update).toString('base64'))
      }
    }
  })

  // 3. 若只读，拦截所有写操作
  if (!canEdit) {
    ws.send(JSON.stringify({ type: 'permission', editable: false }))
  }
})
```

### 6.3 AI 生成时的协同锁定机制

```typescript
// 方案 A：全局只读锁（适合短时间生成）
const lockEditorDuringAI = (editor: Editor) => {
  editor.setEditable(false)
  showAIGeneratingIndicator()
}

// 方案 B：区域锁定（适合分节生成，不影响其他区域协作）
const lockRegionDuringAI = (from: number, to: number) => {
  provider.awareness.setLocalStateField('aiGenerating', { from, to })
}
```

---

## 七、房间权限与成员管理方案

### 7.1 权限模型设计

系统采用 **房间级角色权限** 模型，每个文档即一个"房间"，支持三种角色：

| 角色 | 说明 | 默认能力 |
|------|------|----------|
| `OWNER` | 房间创建者 | 读、写、管理权限、邀请、踢人、删除文档 |
| `EDITOR` | 协作编辑者 | 读、写（受房间权限配置约束）、邀请 Viewer |
| `VIEWER` | 只读观察者 | 仅读 |

此外，Owner 可通过 **房间权限配置** 动态调整允许编辑的角色范围：

```
房间权限配置（editableRoles）示例：
  - ['OWNER']                → 仅 Owner 可编辑
  - ['OWNER', 'EDITOR']      → Owner 和 Editor 可编辑（默认）
  - ['OWNER', 'EDITOR', 'VIEWER'] → 所有人均可编辑（开放模式）
```

---

### 7.2 权限配置功能

**Owner 可在房间设置中调整 `editableRoles`，修改立即对所有在线成员生效。**

实现流程：

```
Owner 修改权限配置
    ↓
API Route PATCH /api/documents/[id]/permissions
    ↓
更新 PostgreSQL 中 Document.editableRoles
    ↓
通过 Redis Pub/Sub 向该文档所有 WebSocket 连接广播权限变更事件
    ↓
各客户端收到事件后重新校验本地编辑权，更新 TipTap editable 状态
```

WebSocket 广播事件格式：

```typescript
// 权限变更推送
{
  type: 'room:permission_updated',
  editableRoles: ['OWNER', 'EDITOR'],
  affectedUserId: null  // null 表示影响所有人
}
```

---

### 7.3 成员邀请功能

支持两种邀请方式：

#### 方式一：邮箱邀请

```
邀请人填写邮箱 + 指定角色（Editor / Viewer）
    ↓
后端创建 Collaborator 记录（pending 状态）
    ↓
发送邀请邮件（含带 token 的确认链接）
    ↓
被邀请人点击链接确认
    ↓
Collaborator 状态变为 active，可加入房间
```

API 设计：

```typescript
// POST /api/documents/[id]/invite
{
  email: 'user@example.com',
  role: 'EDITOR'
}

// 响应
{
  inviteId: 'xxx',
  status: 'pending'
}
```

#### 方式二：链接邀请

```
邀请人生成邀请链接（指定角色 + 可选过期时间）
    ↓
生成 InviteLink 记录，token 唯一
    ↓
分享链接：https://app.com/invite/[token]
    ↓
访客点击链接，登录后自动加入房间并赋予指定角色
```

API 设计：

```typescript
// POST /api/documents/[id]/invite-link
{
  role: 'VIEWER',
  expiresAt: '2025-12-31T00:00:00Z',  // 可选
  maxUses: 10  // 可选，最多使用次数
}

// 响应
{
  token: 'abc123',
  url: 'https://app.com/invite/abc123',
  expiresAt: '2025-12-31T00:00:00Z'
}
```

---

### 7.4 踢出成员功能

**权限规则：**

- `OWNER` 可踢出任意 `EDITOR` 或 `VIEWER`
- `EDITOR` 可踢出 `VIEWER`（可选，通过房间配置开关控制）
- 不能踢出自己，不能踢出 Owner

**踢出流程：**

```
操作人点击踢出某成员
    ↓
API Route DELETE /api/documents/[id]/members/[userId]
    ↓
后端校验操作人权限
    ↓
删除 Collaborator 记录 / 标记为 removed
    ↓
通过 Redis Pub/Sub 向该文档所有 WS 连接广播踢出事件
    ↓
WebSocket Server 找到该用户的连接，发送强制断开通知
    ↓
被踢用户客户端收到事件，WebSocket 关闭，前端跳转至「无访问权限」页面
```

WebSocket 踢出事件格式：

```typescript
// 发送给被踢用户
{
  type: 'room:kicked',
  reason: 'removed_by_owner',
  redirectUrl: '/dashboard'
}

// 广播给房间内其他用户（更新在线成员列表）
{
  type: 'room:member_removed',
  userId: 'xxx',
  removedBy: 'ownerUserId'
}
```

WebSocket Server 实现踢出逻辑：

```typescript
// 维护 documentId → Set<{userId, ws}> 的连接映射
const roomConnections = new Map<string, Set<{userId: string, ws: WebSocket}>>()

// 踢出处理
async function kickUser(documentId: string, targetUserId: string) {
  const connections = roomConnections.get(documentId)
  if (!connections) return

  for (const conn of connections) {
    if (conn.userId === targetUserId) {
      // 发送踢出通知
      conn.ws.send(JSON.stringify({
        type: 'room:kicked',
        reason: 'removed_by_owner',
        redirectUrl: '/dashboard'
      }))
      // 延迟关闭，确保客户端收到消息
      setTimeout(() => conn.ws.close(1008, 'Kicked'), 500)
    } else {
      // 通知其他用户
      conn.ws.send(JSON.stringify({
        type: 'room:member_removed',
        userId: targetUserId
      }))
    }
  }
}
```

---

### 7.5 成员列表实时同步

利用 Yjs **Awareness API** 实时同步在线成员信息，无需额外轮询：

```typescript
// 客户端设置本地 Awareness 状态
provider.awareness.setLocalStateField('user', {
  userId: currentUser.id,
  name: currentUser.name,
  avatar: currentUser.avatarUrl,
  role: currentUserRole,
  color: generateUserColor(currentUser.id)
})

// 监听成员变化（加入 / 离开 / 角色变更）
provider.awareness.on('change', () => {
  const states = Array.from(provider.awareness.getStates().values())
  const onlineMembers = states
    .filter(s => s.user)
    .map(s => s.user)
  updateMemberList(onlineMembers)
})
```

---

### 7.6 前端权限管理 UI

**房间设置面板包含：**

- 权限配置区：下拉选择哪些角色可编辑
- 成员列表区：展示所有成员（在线状态 + 角色标签 + 踢出按钮）
- 邀请区：邮箱邀请表单 + 邀请链接生成按钮

**权限变更实时反馈：**

```typescript
// 监听 WebSocket 权限事件，实时更新编辑器状态
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data)

  if (msg.type === 'room:permission_updated') {
    const canEdit = msg.editableRoles.includes(currentUserRole)
    editor.setEditable(canEdit)
    showPermissionChangedToast(canEdit ? '您现在可以编辑此文档' : '您的编辑权限已被收回')
  }

  if (msg.type === 'room:kicked') {
    editor.setEditable(false)
    showKickedModal()
    router.push(msg.redirectUrl)
  }
}
```

---

## 八、网关层方案

### 8.1 Nginx 部署架构

所有外部请求统一经过 Nginx，Nginx 根据路径路由到对应内部服务：

```
外部请求（80 / 443）
        ↓
      Nginx
        ├── /api/*   → Next.js:3000（HTTP API）
        ├── /ws/*    → Next.js:3000（WebSocket 升级）
        ├── /ai/*    → Python AI Service:8000（SSE 代理）
        └── /*       → Next.js:3000（页面路由）
```

### 8.2 Nginx 配置

```nginx
# nginx.conf

upstream nextjs {
    server nextjs:3000;
}

upstream ai_service {
    server ai-service:8000;
}

# HTTP → HTTPS 重定向
server {
    listen 80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # ── 全局限流（防刷）──
    limit_req_zone $binary_remote_addr zone=global:10m rate=60r/m;

    # ── AI 生成接口专项限流（防止 LLM 配额耗尽）──
    limit_req_zone $binary_remote_addr zone=ai_gen:10m rate=5r/m;

    # ── WebSocket 协同连接 ──
    location /ws/ {
        proxy_pass http://nextjs;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;   # 长连接不超时
        proxy_send_timeout 3600s;
    }

    # ── AI 生成 SSE 接口 ──
    location /api/documents/ {
        limit_req zone=ai_gen burst=3 nodelay;
        proxy_pass http://nextjs;
        proxy_http_version 1.1;
        proxy_set_header Connection "";             # SSE 需要关闭 keep-alive chunked
        proxy_buffering off;                        # 关闭缓冲，确保 SSE 实时推送
        proxy_cache off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 300s;
    }

    # ── 普通 API 接口 ──
    location /api/ {
        limit_req zone=global burst=20 nodelay;
        proxy_pass http://nextjs;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # ── 页面路由 ──
    location / {
        proxy_pass http://nextjs;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 8.3 CORS 统一处理

由 Nginx 统一添加 CORS 头，各服务无需重复配置：

```nginx
add_header Access-Control-Allow-Origin  "https://app.example.com" always;
add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;

if ($request_method = OPTIONS) {
    return 204;
}
```

### 8.4 网关提供的能力总结

| 能力 | 实现方式 |
|------|----------|
| SSL 终止 | 证书配置在 Nginx，内部服务走 HTTP |
| 全局限流 | `limit_req_zone`，按 IP 60次/分钟 |
| AI 接口限流 | 独立 zone，5次/分钟，防止 LLM 配额耗尽 |
| SSE 实时推送 | `proxy_buffering off` |
| WebSocket 支持 | `Upgrade` 头透传 |
| CORS 统一处理 | 集中配置，服务内无需处理 |
| 反向代理路由 | 按路径分发至各内部服务 |

---

## 九、数据库设计

### 9.1 核心数据模型（Prisma Schema）

```prisma
model User {
  id             String         @id @default(cuid())
  email          String         @unique
  name           String?
  avatarUrl      String?
  documents      Document[]
  collaborations Collaborator[]
  sentInvites    Invite[]
  createdAt      DateTime       @default(now())
}

model Document {
  id             String            @id @default(cuid())
  title          String
  ydoc           Bytes?            // Yjs 二进制（Redis 为主，DB 为冷备）
  ownerId        String
  owner          User              @relation(fields: [ownerId], references: [id])

  // 房间权限配置：允许编辑的角色列表
  editableRoles  Role[]            @default([OWNER, EDITOR])

  collaborators  Collaborator[]
  versions       DocumentVersion[]
  shareLinks     ShareLink[]
  aiGenerations  AIGeneration[]
  invites        Invite[]
  inviteLinks    InviteLink[]

  createdAt      DateTime          @default(now())
  updatedAt      DateTime          @updatedAt
}

model Collaborator {
  id         String    @id @default(cuid())
  documentId String
  document   Document  @relation(fields: [documentId], references: [id])
  userId     String
  user       User      @relation(fields: [userId], references: [id])
  role       Role
  status     CollaboratorStatus @default(ACTIVE)
  removedBy  String?   // 记录踢出操作人
  removedAt  DateTime?
  createdAt  DateTime  @default(now())

  @@unique([documentId, userId])
}

// 邮箱邀请
model Invite {
  id         String       @id @default(cuid())
  documentId String
  document   Document     @relation(fields: [documentId], references: [id])
  inviterId  String
  inviter    User         @relation(fields: [inviterId], references: [id])
  email      String
  role       Role
  token      String       @unique
  status     InviteStatus @default(PENDING)
  expiresAt  DateTime?
  createdAt  DateTime     @default(now())
}

// 链接邀请
model InviteLink {
  id         String    @id @default(cuid())
  documentId String
  document   Document  @relation(fields: [documentId], references: [id])
  token      String    @unique
  role       Role
  maxUses    Int?
  usedCount  Int       @default(0)
  expiresAt  DateTime?
  createdAt  DateTime  @default(now())
}

model ShareLink {
  id         String    @id @default(cuid())
  documentId String
  document   Document  @relation(fields: [documentId], references: [id])
  token      String    @unique
  role       Role
  expiresAt  DateTime?
  createdAt  DateTime  @default(now())
}

model DocumentVersion {
  id          String   @id @default(cuid())
  documentId  String
  document    Document @relation(fields: [documentId], references: [id])
  snapshot    Bytes
  stateVector Bytes
  label       String?
  createdBy   String?
  createdAt   DateTime @default(now())
}

model AIGeneration {
  id          String           @id @default(cuid())
  documentId  String
  document    Document         @relation(fields: [documentId], references: [id])
  prompt      String
  outline     Json?
  status      GenerationStatus
  result      String?          @db.Text
  createdAt   DateTime         @default(now())
  completedAt DateTime?
}

enum Role {
  OWNER
  EDITOR
  VIEWER
}

enum CollaboratorStatus {
  ACTIVE
  REMOVED
}

enum InviteStatus {
  PENDING
  ACCEPTED
  EXPIRED
  REVOKED
}

enum GenerationStatus {
  PENDING
  GENERATING
  COMPLETED
  FAILED
}
```

---

## 十、开发里程碑与阶段规划

**总周期：11–12 周**

---

### 第一阶段：基础架构搭建（第 1–2 周）

**目标：** 完成项目初始化、核心架构与网关层配置

**任务清单：**

- Next.js 15 项目初始化（App Router + TypeScript）
- Prisma Schema 定义与数据库迁移
- NextAuth.js 认证配置（邮箱 + OAuth）
- shadcn/ui 基础组件库集成
- Zustand 状态管理架构搭建
- Docker Compose 开发环境（PostgreSQL + Redis + Nginx）
- **Nginx 配置：反向代理路由 + SSL + 基础限流**
- WebSocket Server 基础架构（Custom Next.js Server）
- Yjs 环境配置与联调
- 与现有 Python AI Service 连接冒烟测试

**交付物：**

- 可运行的项目骨架
- 用户注册 / 登录功能可用
- Nginx 统一入口配置完成
- WebSocket 连接测试通过
- AI Service SSE 端点通过 Nginx 调通

---

### 第二阶段：协同编辑核心功能（第 3–4.5 周）

**目标：** 实现多人实时协同编辑基础功能

**任务清单：**

- TipTap Editor 集成与基础富文本功能
- Yjs + y-prosemirror 绑定
- WebSocket Provider 配置
- 多用户光标与选区颜色显示（Awareness API）
- 在线状态管理（用户列表、进入 / 离开提示）
- YDoc 持久化到 Redis
- 文档初始化加载流程
- 离线编辑与网络恢复同步（IndexedDB 缓存）
- 基础权限控制（只读 / 可编辑）

**交付物：**

- 多人可实时编辑同一文档
- 光标位置与选区实时同步
- 数据持久化到 Redis + PostgreSQL

---

### 第三阶段：房间权限与成员管理（第 5–6 周）

**目标：** 实现完整的房间权限配置、成员邀请与踢出功能

**任务清单：**

- 数据模型扩展（Document.editableRoles、Collaborator.status、Invite、InviteLink）
- 权限配置 API（PATCH /api/documents/[id]/permissions）
- Redis Pub/Sub 权限变更广播机制
- WebSocket Server 连接映射表（documentId → 连接集合）
- 踢出 API（DELETE /api/documents/[id]/members/[userId]）
- 踢出后 WebSocket 强制断开 + 前端跳转
- 邮箱邀请 API + 邀请邮件发送
- 链接邀请 API（含 maxUses、expiresAt）
- 邀请确认页面（token 校验 → 自动加入房间）
- 房间设置面板 UI（权限配置 + 成员列表 + 邀请入口）
- 前端权限变更实时响应（WebSocket 事件监听）
- Editor 是否可踢 Viewer 的开关配置

**交付物：**

- 完整房间权限配置功能
- 邀请链接与邮箱邀请均可用
- 踢出后连接立即断开，前端实时跳转
- 权限变更实时同步给所有在线成员

---

### 第四阶段：AI 文档生成集成（第 6.5–8 周）

**目标：** AI 生成能力与协同编辑器无缝集成

**任务清单：**

- 文档配置页面 UI（主题、大纲、参数设置）
- Next.js BFF 层 SSE 转发 API Route
- `useAIGeneration` Hook 开发
- AI 内容通过 Yjs 事务实时插入 TipTap
- 大纲可视化编辑组件
- AI 生成时协同编辑锁定机制
- 工具调用状态气泡组件
- 文档导出（Word / PDF）
- **Nginx AI 接口专项限流配置（5次/分钟）**
- 生成历史记录

**交付物：**

- 完整 AI 文档生成流程
- AI 内容与协同编辑无缝集成
- 可导出的文档

---

### 第五阶段：版本控制与历史（第 8.5–9.5 周）

**目标：** 实现完整的版本管理体系

**任务清单：**

- YDoc 快照自动保存（时间间隔 + 操作次数策略）
- 版本列表页面
- 版本 diff 可视化
- 版本回滚功能
- 版本标签与备注

**交付物：**

- 完整版本控制系统
- 版本对比与回滚功能可用

---

### 第六阶段：文档管理与完善（第 10 周）

**目标：** 完善文档管理和多人协作体验

**任务清单：**

- 文档列表页（搜索、排序、筛选）
- 文件上传功能（图片 + 附件）+ MinIO 集成
- 收藏与标签功能
- ECharts 图表生成与文档嵌入
- AI 内容增强工具（调用 MCP enhancement 服务）
- WebSocket 连接池管理
- 大文档虚拟滚动 + 分片加载
- 网络断线重连机制
- 响应式布局优化（移动端适配）
- 错误处理与用户反馈体系

**交付物：**

- 完整文档管理系统
- 图表功能可用
- 协同编辑性能满足目标指标

---

### 第七阶段：测试与部署（第 11–12 周）

**目标：** 确保系统稳定性，完成上线准备

**任务清单：**

- 单元测试（核心业务逻辑）
- 集成测试（API Routes + WebSocket + 权限边界）
- 协同编辑压力测试（50+ 用户并发）
- 房间权限功能专项测试（踢出、权限变更边界场景）
- E2E 测试（Playwright）
- 性能测试 & 安全审计
- Nginx 生产配置调优
- Docker 生产镜像构建
- CI/CD 流程配置（GitHub Actions）

**交付物：**

- 测试报告
- 部署文档
- 系统正式上线

---

### 里程碑总览

| 阶段 | 时间节点 | 关键成果 |
|------|----------|----------|
| 第一阶段 | 第 1–2 周 | 项目骨架完成，认证可用，Nginx 网关就绪，AI 服务联通 |
| 第二阶段 | 第 3–4.5 周 | 实时协同编辑功能上线 |
| 第三阶段 | 第 5–6 周 | 房间权限管理、邀请、踢人功能上线 |
| 第四阶段 | 第 6.5–8 周 | AI 文档生成功能上线并集成协同编辑 |
| 第五阶段 | 第 8.5–9.5 周 | 版本控制系统完成 |
| 第六阶段 | 第 10 周 | 文档管理与增强功能完成 |
| 第七阶段 | 第 11–12 周 | 测试完成，系统正式上线 |

---

## 十一、技术难点与解决方案

### 11.1 协同编辑冲突处理

**难点：** 多用户并发编辑同一区域可能产生冲突

**解决方案：** 使用 Yjs CRDT 算法，在数学层面保证最终一致性，无需手动合并。

---

### 11.2 权限变更实时生效

**难点：** Owner 修改权限后，需要在不断开连接的前提下让所有在线客户端立即生效

**解决方案：**
- WebSocket Server 维护 `documentId → 连接集合` 的内存映射
- 权限变更通过 Redis Pub/Sub 广播到所有 WS 实例
- 各客户端收到 `room:permission_updated` 事件后本地重新校验，动态切换 TipTap `editable` 状态
- 不需要重连，用户无感知

---

### 11.3 踢出后连接强制断开

**难点：** 被踢用户的 WebSocket 连接需要立即终止，且前端要有清晰的状态跳转

**解决方案：**
- 服务端先发送 `room:kicked` 消息（含跳转 URL），延迟 500ms 再调用 `ws.close()`
- 客户端监听到该消息后主动调用 `provider.disconnect()`，清理 Yjs 状态
- 前端 Router 跳转至无权限页面，避免用户困惑

---

### 11.4 AI 生成内容与协同编辑的一致性

**难点：** SSE 流式内容插入时，其他用户的并发操作可能与 AI 内容产生位置偏移

**解决方案：**
- 所有 AI 内容插入通过 `ydoc.transact()` 包裹，保证原子性
- AI 生成区域通过 Awareness API 广播锁定状态
- 生成完成后立即解锁

---

### 11.5 大文档性能

**难点：** 大型文档（>10 万字）加载和同步延迟高

**解决方案：**
- Yjs 增量同步（只传输变更 Update）
- 文档按章节分片懒加载
- 编辑器内容区域使用虚拟滚动

---

### 11.6 WebSocket 多实例扩展性

**难点：** 水平扩展时多个 WS 实例无法共享 YDoc 状态和连接映射（踢出、权限广播失效）

**解决方案：**
- Redis Pub/Sub 实现跨实例的 YDoc Update 和控制事件（踢出、权限变更）广播
- Nginx 启用 Sticky Session（同一文档的连接路由到同一实例）
- 长期引入 `y-redis` 作为生产级持久化与同步层

---

### 11.7 Nginx SSE 代理

**难点：** Nginx 默认开启缓冲，导致 SSE 内容无法实时到达客户端

**解决方案：** AI 生成接口路由块中配置 `proxy_buffering off` 和 `proxy_cache off`，同时设置足够长的 `proxy_read_timeout`（300s），避免长时间生成被误判为超时断开。

---

## 十二、成功指标

| 指标 | 目标值 |
|------|--------|
| 核心功能完整度 | 100% |
| 协同编辑实时延迟 | < 100ms |
| 单文档支持并发用户 | > 50 人 |
| AI 生成首 Token 延迟 | < 3s |
| 页面首屏加载时间 | < 3s |
| API 平均响应时间 | < 500ms |
| 权限变更生效延迟 | < 200ms |
| 踢出后连接断开延迟 | < 1s |
| 代码测试覆盖率 | > 70% |
| 移动端适配 | 完全响应式 |
| 系统可用性 | > 99.5% |

---

*文档整合自：方案1（PRD 需求文档）、方案2（AI 生成实现参考 + Next.js 重构方案）、方案3（LangGraph / LangChain / MCP 架构深度解析）；新增：Nginx 网关层方案、房间权限与成员管理方案*
