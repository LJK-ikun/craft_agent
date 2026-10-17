# Craft Agents 第四个简历点：Bun Monorepo + Electron 跨平台构建

## 一、技术架构概述

Craft Agents 是一个大型跨平台项目，包含多个子包和应用。需要解决：

1. **依赖管理**：多个包共享依赖，如何避免重复安装？
2. **构建速度**：TypeScript 编译慢，如何加速？
3. **跨平台打包**：如何为 macOS（Intel + Apple Silicon）、Windows、Linux 构建统一的可执行文件？

**解决方案**：

- **Bun Monorepo**：用 Bun 管理工作空间，统一依赖，快速构建
- **electron-builder**：跨平台打包工具，支持 DMG、NSIS、AppImage

---

## 二、Bun Monorepo 架构

### 2.1 什么是 Monorepo？

**Monorepo**（单一仓库）：把多个相关项目放在一个 Git 仓库，共享代码和依赖。

**对比**：

| 架构                           | 优点                                 | 缺点                                 |
| ------------------------------ | ------------------------------------ | ------------------------------------ |
| **Multi-repo**（多仓库） | 每个项目独立，版本控制简单           | 共享代码困难，依赖更新麻烦，重复安装 |
| **Monorepo**（单仓库）   | 共享代码容易，统一依赖版本，一次安装 | 仓库变大，构建复杂                   |

**Craft Agents 的 Monorepo 结构**：

```
craft-agents-oss-main/
├── packages/               # 共享库
│   ├── core/              # 核心类型定义
│   ├── shared/            # 共享工具（MCP、OAuth、权限等）
│   ├── server-core/       # 服务器核心（WebSocket RPC）
│   ├── server/            # 完整服务器（CLI 入口）
│   ├── session-tools-core/  # Session 工具
│   ├── pi-agent-server/   # Pi Agent 服务器
│   └── ui/                # UI 组件库
│
├── apps/                  # 应用
│   ├── electron/          # Electron 桌面应用
│   ├── cli/               # 命令行工具
│   └── webui/             # Web UI
│
├── package.json           # 根 package.json（定义工作空间）
└── bun.lockb              # Bun 锁文件
```

### 2.2 Bun Workspaces 配置

**根 package.json** (`package.json:18-22`)：

```json
{
  "workspaces": [
    "packages/*",
    "apps/*",
    "!apps/online-docs"
  ]
}
```

**工作空间的作用**：

- `packages/*`：所有 `packages/` 下的子包都是工作空间成员
- `apps/*`：所有 `apps/` 下的应用都是工作空间成员
- `!apps/online-docs`：排除 `online-docs`（不参与构建）

**依赖共享**：

```typescript
// packages/shared/package.json
{
  "name": "@craft-agent/shared",
  "dependencies": {
    "zod": "^3.22.0"
  }
}

// apps/electron/package.json
{
  "name": "@craft-agent/electron",
  "dependencies": {
    "@craft-agent/shared": "workspace:*",
    //                     ^^^^^^^^^^^^ 引用工作空间的包
    "electron": "^39.2.7"
  }
}
```

**`workspace:*` 的含义**：

- 引用同一个 monorepo 内的包，不从 npm 下载
- Bun 会自动链接到 `packages/shared`
- 修改 `@craft-agent/shared` 的代码，Electron 立即生效（不需要发布到 npm）

### 2.3 为什么用 Bun？

**Bun vs npm/yarn/pnpm**：

| 工具           | 安装速度       | 运行速度       | 特点                          |
| -------------- | -------------- | -------------- | ----------------------------- |
| **npm**  | 慢             | 慢             | 官方工具，兼容性最好          |
| **yarn** | 中等           | 中等           | 工作空间支持好                |
| **pnpm** | 快             | 快             | 硬链接节省空间                |
| **Bun**  | **极快** | **极快** | Zig 编写，原生支持 TypeScript |

**Bun 的优势**：

- **快速安装**：并行下载，二进制缓存，比 npm 快 10-25 倍
- **原生 TypeScript**：直接运行 `.ts` 文件，不需要 `tsc` 编译
- **内置打包器**：`bun build` 内置 bundler，替代 esbuild/webpack
- **兼容 Node.js**：API 兼容，无缝迁移

**实测对比**（安装所有依赖）：

```bash
# npm
npm install  # ~120 秒

# Bun
bun install  # ~8 秒
```

---

## 三、构建流程（Bun + esbuild）

### 3.1 构建脚本总览

**根 package.json 的构建命令** (`package.json:58-65`)：

```json
{
  "scripts": {
    "electron:build:main": "bun run scripts/electron-build-main.ts",
    "electron:build:preload": "bun run scripts/electron-build-preload.ts",
    "electron:build:renderer": "bun run scripts/electron-build-renderer.ts",
    "electron:build:resources": "bun run scripts/electron-build-resources.ts",
    "electron:build:assets": "cd apps/electron && bun scripts/copy-assets.ts",
    "electron:build": "bun run electron:build:main && bun run electron:build:preload && bun run electron:build:renderer && bun run electron:build:resources && bun run electron:build:assets"
  }
}
```

**构建步骤**：

```
1. electron:build:main
   ├─ 构建主进程（Main Process）：apps/electron/src/main → dist/main.cjs
   ├─ 构建网络拦截器：packages/shared/src/unified-network-interceptor.ts → dist/interceptor.cjs
   ├─ 构建 Session MCP Server：packages/session-mcp-server/src → dist/index.js
   ├─ 构建 Pi Agent Server：packages/pi-agent-server/src → dist/index.js
   └─ 构建 WhatsApp Worker：packages/messaging-whatsapp-worker/src → dist/worker.cjs

2. electron:build:preload
   └─ 构建预加载脚本：apps/electron/src/preload → dist/preload.cjs

3. electron:build:renderer
   └─ 构建渲染进程（Renderer Process）：apps/electron/src/renderer → dist/renderer/

4. electron:build:resources
   └─ 复制 Python 脚本、二进制工具到 resources/

5. electron:build:assets
   └─ 复制静态资源（图标、主题、文档）到 dist/resources/
```

### 3.2 主进程构建（electron-build-main.ts）

**代码位置**：`scripts/electron-build-main.ts`

**核心流程**：

```typescript
// 1. 加载环境变量（OAuth credentials）
loadEnvFile();  // 读取 .env 文件

// 2. 获取构建时的环境变量定义
const defines = getBuildDefines();
// 例如: ["--define:process.env.SLACK_OAUTH_CLIENT_ID=\"abc123\""]

// 3. 构建主进程
spawn({
  cmd: [
    "bun", "run", "esbuild",
    "apps/electron/src/main/index.ts",  // 入口文件
    "--bundle",                          // 打包所有依赖
    "--platform=node",                   // Node.js 环境
    "--format=cjs",                      // CommonJS 格式
    "--outfile=apps/electron/dist/main.cjs",  // 输出文件
    "--external:electron",               // 不打包 Electron（运行时提供）
    "--external:@anthropic-ai/claude-agent-sdk",  // 不打包 SDK（运行时加载）
    ...defines,                          // 注入环境变量
  ],
});

// 4. 构建网络拦截器（用于 SDK 子进程）
buildInterceptor();

// 5. 构建 Session MCP Server
buildSessionServer();

// 6. 构建 Pi Agent Server
buildPiAgentServer();

// 7. 构建 WhatsApp Worker
buildWhatsAppWorker();
```

**为什么需要多个子进程？**

| 子进程                        | 用途                                            | 为什么独立？                        |
| ----------------------------- | ----------------------------------------------- | ----------------------------------- |
| **Session MCP Server**  | 提供 Session 专属工具（SubmitPlan）             | 隔离 Session 状态，崩溃不影响主进程 |
| **Pi Agent Server**     | 支持非 Anthropic 模型（OpenAI、GitHub Copilot） | 不同 SDK，独立进程                  |
| **WhatsApp Worker**     | WhatsApp 消息处理                               | Baileys 库较大，独立进程节省内存    |
| **Network Interceptor** | 拦截 HTTP 请求（错误捕获、MCP schema 注入）     | 用 `--require` 注入到 SDK 子进程  |

### 3.3 渲染进程构建（electron-build-renderer.ts）

**渲染进程**：Electron 的 UI 部分，运行在浏览器环境。

**构建工具**：Vite（专为前端优化，支持 React、TypeScript、CSS Modules）

**流程**：

```typescript
// 使用 Vite 构建 React 应用
spawn({
  cmd: [
    "bunx", "vite", "build",
    "--config", "apps/electron/vite.config.ts",
    "--outDir", "apps/electron/dist/renderer",
  ],
});
```

**Vite 配置** (`apps/electron/vite.config.ts`)：

```typescript
export default defineConfig({
  root: 'src/renderer',
  build: {
    outDir: '../../dist/renderer',
    rollupOptions: {
      input: {
        index: 'src/renderer/index.html',
      },
    },
  },
  plugins: [react()],
});
```

### 3.4 构建产物结构

**构建完成后的目录**：

```
apps/electron/dist/
├── main.cjs                       # 主进程（单文件）
├── preload.cjs                    # 预加载脚本（单文件）
├── interceptor.cjs                # 网络拦截器（单文件）
├── renderer/                      # 渲染进程（多文件）
│   ├── index.html
│   ├── assets/
│   │   ├── index-abc123.js       # React 应用（带哈希）
│   │   └── index-def456.css      # 样式（带哈希）
│   └── ...
└── resources/                     # 静态资源
    ├── bin/                       # 平台特定二进制（uv, Bun）
    │   ├── darwin-arm64/
    │   ├── darwin-x64/
    │   ├── win32-x64/
    │   └── linux-x64/
    ├── scripts/                   # Python 工具脚本
    │   ├── pdf-tool.py
    │   ├── xlsx-tool.py
    │   └── ...
    ├── permissions/               # 权限配置
    │   └── default.json
    ├── themes/                    # UI 主题
    └── docs/                      # 内置文档
```

---

## 四、Electron 跨平台打包

### 4.1 electron-builder 配置

**配置文件**：`apps/electron/electron-builder.yml`

**支持的平台**：

| 平台              | 架构                  | 打包格式    | 文件名                        |
| ----------------- | --------------------- | ----------- | ----------------------------- |
| **macOS**   | arm64 (Apple Silicon) | DMG, ZIP    | `Craft-Agents-arm64.dmg`    |
| **macOS**   | x64 (Intel)           | DMG, ZIP    | `Craft-Agents-x64.dmg`      |
| **Windows** | x64                   | NSIS 安装包 | `Craft-Agents-x64.exe`      |
| **Linux**   | x64                   | AppImage    | `Craft-Agents-x64.AppImage` |

### 4.2 核心配置解析

**macOS 配置** (`electron-builder.yml:84-133`)：

```yaml
mac:
  category: public.app-category.productivity  # 应用分类
  icon: resources/icon.icns                  # 应用图标
  target:
    - target: dmg                            # 打包为 DMG
      arch:
        - arm64                              # Apple Silicon
        - x64                                # Intel
    - target: zip                            # 打包为 ZIP
      arch:
        - arm64
        - x64
  hardenedRuntime: true                      # 启用强化运行时（macOS 安全要求）
  gatekeeperAssess: false                    # 禁用 Gatekeeper 评估（本地构建）
  entitlements: build/entitlements.mac.plist # 权限声明（网络、相机、麦克风等）
  extraResources:
    # Claude SDK（~4 MB 核心 + 210 MB 原生二进制）
    - from: node_modules/@anthropic-ai/claude-agent-sdk
      to: app/node_modules/@anthropic-ai/claude-agent-sdk
    - from: node_modules/@anthropic-ai/claude-agent-sdk-binary
      to: app/node_modules/@anthropic-ai/claude-agent-sdk-binary
    # ripgrep（代码搜索工具）
    - from: node_modules/@vscode/ripgrep
      to: app/node_modules/@vscode/ripgrep
  files:
    # 排除其他平台的二进制
    - "!**/resources/bin/win32-*/**"
    - "!**/resources/bin/linux-*/**"
```

**Windows 配置** (`electron-builder.yml:155-188`)：

```yaml
win:
  icon: resources/icon.ico
  target:
    - target: nsis                           # NSIS 安装包
      arch:
        - x64
  files:
    # 排除其他平台的二进制
    - "!**/resources/bin/darwin-*/**"
    - "!**/resources/bin/linux-*/**"
    # Windows 特殊处理：Bun 和 uv 放到 extraResources（避免 EBUSY 错误）
    - "!vendor/bun/**/*"
    - "!**/resources/bin/win32-x64/**"
  extraResources:
    # Bun 运行时（Windows .exe 文件锁定问题，必须用 extraResources）
    - from: vendor/bun/bun.exe
      to: vendor/bun/bun.exe
    - from: resources/bin/win32-x64
      to: app/resources/bin/win32-x64

nsis:
  oneClick: true                             # 一键安装
  perMachine: false                          # 用户级安装（到 %LOCALAPPDATA%）
  deleteAppDataOnUninstall: true             # 卸载时删除用户数据
```

**为什么 Windows 要用 extraResources？**

- electron-builder 在复制 `node_modules` 时会扫描和锁定文件
- 同时复制 `.exe` 文件会导致 **EBUSY 错误**（文件被占用）
- `extraResources` 在扫描之前复制，避免锁定

**Linux 配置** (`electron-builder.yml:197-219`)：

```yaml
linux:
  icon: resources/icon.png
  category: Utility
  target:
    - target: AppImage                       # AppImage（单文件可执行）
      arch:
        - x64
  extraResources:
    - from: node_modules/@anthropic-ai/claude-agent-sdk
      to: app/node_modules/@anthropic-ai/claude-agent-sdk
    - from: node_modules/@anthropic-ai/claude-agent-sdk-binary
      to: app/node_modules/@anthropic-ai/claude-agent-sdk-binary
  files:
    # 排除其他平台的二进制
    - "!**/resources/bin/darwin-*/**"
    - "!**/resources/bin/win32-*/**"
```

### 4.3 打包命令

**本地开发打包**：

```bash
# macOS（不签名，快速测试）
bun run electron:dist:dev:mac

# Windows
bun run electron:dist:dev:win

# Linux
bun run electron:dist:linux
```

**生产环境打包**（带签名和公证）：

```bash
# macOS（需要设置 CSC_LINK、APPLE_ID 等环境变量）
bun run electron:dist:mac

# Windows（需要设置 CSC_LINK）
bun run electron:dist:win

# Linux
bun run electron:dist:linux
```

**打包流程**：

```
1. 运行 electron:build（构建所有代码）
   ↓
2. electron-builder 读取 electron-builder.yml
   ↓
3. 复制文件到临时目录：
   ├─ dist/**/* （构建产物）
   ├─ package.json
   ├─ resources/（Python 脚本、二进制工具）
   └─ extraResources/（SDK、ripgrep、Bun）
   ↓
4. 平台特定处理：
   ├─ macOS: 生成 .app 包 → 签名 → 公证 → 打包为 DMG
   ├─ Windows: 生成 NSIS 安装包 → 签名
   └─ Linux: 生成 AppImage
   ↓
5. 输出到 apps/electron/release/：
   ├─ Craft-Agents-arm64.dmg
   ├─ Craft-Agents-x64.dmg
   ├─ Craft-Agents-x64.exe
   └─ Craft-Agents-x64.AppImage
```

### 4.4 平台特定二进制处理

**问题**：不同平台需要不同的二进制（Bun、uv、Python 工具的依赖）

**解决方案**：构建时只打包当前平台的二进制

**目录结构**：

```
resources/bin/
├── darwin-arm64/
│   ├── uv             # macOS ARM 版 uv
│   └── ...
├── darwin-x64/
│   ├── uv             # macOS Intel 版 uv
│   └── ...
├── win32-x64/
│   ├── uv.exe         # Windows 版 uv
│   └── ...
└── linux-x64/
    ├── uv             # Linux 版 uv
    └── ...
```

**electron-builder 配置中的排除规则**：

```yaml
# macOS 构建：排除 Windows 和 Linux 二进制
mac:
  files:
    - "!**/resources/bin/win32-*/**"
    - "!**/resources/bin/linux-*/**"

# Windows 构建：排除 macOS 和 Linux 二进制
win:
  files:
    - "!**/resources/bin/darwin-*/**"
    - "!**/resources/bin/linux-*/**"

# Linux 构建：排除 macOS 和 Windows 二进制
linux:
  files:
    - "!**/resources/bin/darwin-*/**"
    - "!**/resources/bin/win32-*/**"
```

**结果**：

- macOS 安装包只包含 `darwin-arm64/` 或 `darwin-x64/`
- Windows 安装包只包含 `win32-x64/`
- Linux 安装包只包含 `linux-x64/`
- **每个平台的安装包小 200+ MB**（不包含其他平台的二进制）

---

## 五、完整构建与发布流程

### 5.1 开发构建（本地测试）

```bash
# 1. 安装依赖
bun install

# 2. 构建所有代码
bun run electron:build

# 3. 启动 Electron（开发模式）
bun run electron:start

# 4. 打包为可执行文件（不签名）
bun run electron:dist:dev:mac    # macOS
bun run electron:dist:dev:win    # Windows
bun run electron:dist:linux      # Linux
```

### 5.2 生产构建（发布）

```bash
# 1. 清理旧构建
bun run electron:clean

# 2. 运行测试和类型检查
bun run validate:ci

# 3. 构建所有代码
bun run electron:build

# 4. 打包并签名
# macOS（需要证书）
export CSC_LINK=/path/to/certificate.p12
export CSC_KEY_PASSWORD=your_password
export APPLE_ID=your_apple_id
export APPLE_APP_SPECIFIC_PASSWORD=your_password
export APPLE_TEAM_ID=your_team_id
bun run electron:dist:mac

# Windows（需要证书）
export CSC_LINK=/path/to/certificate.pfx
export CSC_KEY_PASSWORD=your_password
bun run electron:dist:win

# Linux（不需要签名）
bun run electron:dist:linux

# 5. 发布产物到 CDN
# release/ 目录下的文件：
# - Craft-Agents-arm64.dmg
# - Craft-Agents-x64.dmg
# - Craft-Agents-x64.exe
# - Craft-Agents-x64.AppImage
# - latest-mac.yml（自动更新清单）
# - latest.yml（自动更新清单）
```

### 5.3 自动更新机制

**electron-updater** 配置 (`electron-builder.yml:77-79`)：

```yaml
publish:
  provider: generic
  url: https://agents.craft.do/electron/latest
```

**工作原理**：

1. 应用启动时检查更新：

   ```typescript
   autoUpdater.checkForUpdates();
   // 请求: https://agents.craft.do/electron/latest/latest-mac.yml
   ```
2. 服务器返回清单文件（`latest-mac.yml`）：

   ```yaml
   version: 0.11.5
   files:
     - url: Craft-Agents-arm64.dmg
       sha512: abc123...
       size: 234567890
   path: Craft-Agents-arm64.dmg
   sha512: abc123...
   releaseDate: '2026-01-15T10:00:00.000Z'
   ```
3. 比对版本号：

   - 当前版本：`0.11.4`
   - 最新版本：`0.11.5`
   - 有更新 → 下载 DMG
4. 下载完成后提示用户重启安装

---

## 六、关键技术点总结

### 6.1 Bun Monorepo

- **问题**：多个包共享依赖，如何避免重复安装？
- **方案**：Bun Workspaces，用 `workspace:*` 引用本地包
- **优势**：
  - 统一依赖版本（所有包用同一个 zod 版本）
  - 安装快（Bun 比 npm 快 10-25 倍）
  - 修改共享包立即生效（不需要发布到 npm）

### 6.2 Bun 构建

- **问题**：TypeScript 编译慢，每次修改要等很久
- **方案**：Bun 原生支持 TypeScript，直接运行 `.ts` 文件
- **优势**：
  - 不需要 `tsc` 编译步骤
  - `bun build` 内置 bundler，速度快
  - 开发体验好（热更新快）

### 6.3 electron-builder 跨平台打包

- **问题**：如何为 macOS、Windows、Linux 打包？
- **方案**：electron-builder 读取 YAML 配置，自动处理平台差异
- **优势**：
  - 统一配置，一次定义，多平台打包
  - 自动签名和公证（macOS）
  - 支持自动更新（electron-updater）

### 6.4 平台特定二进制

- **问题**：Bun、uv 等二进制每个平台不同，如何打包？
- **方案**：构建时用 `files` 规则排除其他平台的二进制
- **优势**：
  - macOS 安装包只包含 macOS 二进制
  - 每个平台的安装包小 200+ MB

### 6.5 子进程隔离

- **问题**：为什么要拆分多个子进程？
- **方案**：Session MCP Server、Pi Agent Server、WhatsApp Worker 独立进程
- **优势**：
  - 隔离状态（Session 崩溃不影响主进程）
  - 节省内存（WhatsApp Worker 只在需要时启动）
  - 支持不同 SDK（Pi Agent 用 OpenAI SDK）

---

## 七、面试可能的问题

### Q1: 为什么用 Bun 而不是 npm？

**答案**：

- Bun 安装速度快 10-25 倍（并行下载 + 二进制缓存）
- 原生支持 TypeScript（不需要 `tsc` 编译）
- 内置 bundler（替代 esbuild/webpack）
- API 兼容 Node.js（无缝迁移）

### Q2: Monorepo 的优势是什么？

**答案**：

- 共享代码容易（`workspace:*` 引用本地包）
- 统一依赖版本（所有包用同一个版本）
- 一次安装（不用每个包分别 `npm install`）
- 修改共享包立即生效（不需要发布到 npm）

### Q3: electron-builder 的 extraResources 是什么？

**答案**：

- `files` 规则打包的文件会被 electron-builder 扫描和锁定
- Windows 复制 `.exe` 文件时会 EBUSY 错误（文件被占用）
- `extraResources` 在扫描之前复制，避免锁定
- 运行时可以通过 `process.resourcesPath` 访问

### Q4: 为什么打包时要排除其他平台的二进制？

**答案**：

- macOS 不需要 Windows 的 `.exe` 文件，Windows 不需要 macOS 的二进制
- 如果不排除，每个平台的安装包会大 200+ MB
- 用 `files: ["!**/resources/bin/win32-*/**"]` 排除

### Q5: Electron 的主进程、渲染进程、预加载脚本有什么区别？

**答案**：

| 进程类型             | 环境            | 权限         | 用途                         |
| -------------------- | --------------- | ------------ | ---------------------------- |
| **主进程**     | Node.js         | 完全权限     | 文件系统、系统 API、窗口管理 |
| **渲染进程**   | Chromium 浏览器 | 受限（沙箱） | UI 渲染（React）             |
| **预加载脚本** | 桥接            | 有限权限     | 暴露安全的 API 给渲染进程    |

### Q6: 自动更新是怎么实现的？

**答案**：

1. 应用启动时检查更新：请求 `https://agents.craft.do/electron/latest/latest-mac.yml`
2. 比对版本号：当前 `0.11.4` vs 最新 `0.11.5`
3. 有更新 → 下载 DMG，校验 SHA512
4. 提示用户重启安装

### Q7: 为什么需要代码签名？

**答案**：

- **macOS**：没签名无法通过 Gatekeeper（用户打开会提示"已损坏"）
- **Windows**：没签名会被 SmartScreen 拦截（提示"不受信任的发布者"）
- **签名证书**：从 Apple Developer 或 Windows Code Signing 购买

---

## 八、代码位置索引

| 模块                  | 文件                                   | 关键行数 | 功能                       |
| --------------------- | -------------------------------------- | -------- | -------------------------- |
| Monorepo 配置         | `package.json`                       | 18-22    | workspaces 定义            |
| 构建脚本              | `package.json`                       | 58-80    | electron:build 系列命令    |
| 主进程构建            | `scripts/electron-build-main.ts`     | 1-300    | 构建主进程、拦截器、子进程 |
| electron-builder 配置 | `apps/electron/electron-builder.yml` | 1-220    | 跨平台打包配置             |
| macOS 配置            | `apps/electron/electron-builder.yml` | 84-133   | macOS 打包规则             |
| Windows 配置          | `apps/electron/electron-builder.yml` | 155-188  | Windows 打包规则           |
| Linux 配置            | `apps/electron/electron-builder.yml` | 197-219  | Linux 打包规则             |
| 自动更新              | `apps/electron/electron-builder.yml` | 77-79    | electron-updater 配置      |

---

**四个简历点全部完成！**

现在你已经深入理解了 Craft Agents 的四个核心技术点：

1. ✅ **MCP 协议 + OAuth 2.0 + AES-256-GCM 加密存储**
2. ✅ **WebSocket RPC 协议 + 三层权限系统**
3. ✅ **Session 持久化（JSONL）+ 状态管理（Jotai）**
4. ✅ **Bun Monorepo + Electron 跨平台构建**

所有技术细节、代码位置、面试问题都已整理完毕。可以用这些文档准备面试了！
