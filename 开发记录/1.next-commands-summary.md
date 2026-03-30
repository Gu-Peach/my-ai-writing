# Next.js 开发命令总结

## 1. Cursor 中如何打开 AI 编辑面板

| 面板 | 快捷键 | 说明 |
|------|--------|------|
| Chat（对话模式） | `Ctrl + L` | 普通 AI 对话 |
| Composer（多文件编辑） | `Ctrl + I` | 可执行命令、编辑多个文件 |

> Composer 面板支持引用 `CLAUDE.md` 作为项目级背景上下文。

---

## 2. 四个 npm 命令的区别

| 命令 | 作用 | 特点 |
|------|------|------|
| `next dev` | 启动**开发服务器** | 热更新、详细报错、性能未优化 |
| `next build` | **构建**生产包 | 编译打包、代码压缩、生成 `.next` 目录 |
| `next start` | 启动**生产服务器** | 必须先 build、性能最优、无热更新 |
| `eslint` | **代码检查** | 检查语法/规范问题，不启动服务 |

### 使用场景

- **开发阶段** → `npm run dev`
- **生产部署** → 先 `npm run build`，再 `npm run start`

```bash
npm run build   # 先构建（只需一次）
npm run start   # 再启动生产服务器
```

---

## 3. `npm run build` 执行后发生了什么

### 生成 `.next` 目录

```
my-ai-writing/
├── .next/          ← 构建产物都在这里
│   ├── static/     ← 静态资源（JS、CSS）
│   ├── server/     ← 服务端渲染文件
│   └── cache/      ← 构建缓存
```

### 终端输出构建报告

```bash
Route (app)          Size    First Load JS
┌ ○ /                1.2 kB  85 kB
├ ○ /about           800 B   84 kB
└ ƛ /api/chat        0 B     0 B
```

### 构建后的操作

```bash
# 本地测试生产环境
npm run build → npm run start

# 部署到服务器
# 将项目（含 .next 目录）上传服务器后直接执行
npm run start
```

### ⚠️ 注意事项

- `.next` 目录不要提交到 git（已在 `.gitignore` 中）
- 每次代码修改后需要**重新 build** 才能生效
- build 过程中有类型错误或编译错误会直接失败

---

## 4. Build 是一次性的还是分阶段的？

### 开发过程中不需要手动 build

```
开发阶段全程只用 npm run dev
代码改了 → 浏览器自动热更新 → 继续改
```

`next dev` 内部有增量编译，每次保存文件自动重新编译，无需手动操作。

### Build 只在以下时机执行

| 时机 | 说明 |
|------|------|
| 准备部署上线前 | 最常见，构建一次然后 start |
| 测试生产环境表现 | 想看真实性能、检查 SSR 是否正常 |
| CI/CD 流水线中 | 每次推代码自动触发 build + deploy |
| 排查 dev 和生产的差异 | dev 下没问题但生产有 bug 时 |

### 直觉理解

```
npm run dev    = 草稿模式，边写边看
npm run build  = 定稿打印，交付用
```

> 处于开发阶段时，一直用 `dev` 即可，等功能完成准备部署时再 `build` 一次。
