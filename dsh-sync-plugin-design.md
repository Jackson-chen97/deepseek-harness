# dsh-sync-plugin: 跨设备插件与技能同步方案

## 1. 背景与动机

DeepSeek Harness 的插件生态正在爆发式增长，社区已有 11,000+ 插件、1,800+ 技能。但当前的部署半径局限于单机单进程：

- 用户在多台设备上使用 dsh，需要手动重复安装插件
- 无法跨设备同步自定义技能和配置
- 没有统一的插件/技能配置备份机制

官方 [RFC #2276](https://github.com/deepseek-ai/deepseek-harness/discussions/2276) 已提出生态基建需求，包括跨设备同步的 identity 层。本插件作为过渡方案，使用 GitHub 仓库作为同步中枢。

## 2. 设计目标

- **跨设备同步**：将插件清单、技能文件、用户 patch、设置通过 GitHub 仓库同步
- **安全优先**：凭证、秘钥、session 数据不同步
- **可审查**：每次变更可预览，支持逐行确认
- **非侵入**：不修改 dsh 核心代码，纯插件形式工作

## 3. 同步范围

### 3.1 同步项

| 同步项 | 本地路径 | 远程格式 | 说明 |
|---|---|---|---|
| 插件依赖清单 | `~/.dsh/profiles/<profile>/package.json` (dependencies) | `profiles/<profile>/dependencies.json` | 仅记录包名和版本约束 |
| Bundle 列表 | `~/.dsh/profiles/<profile>/package.json` (dsh.profile.bundles) | `profiles/<profile>/bundles.json` | 插件激活顺序 |
| 用户技能 | `~/.dsh/skills/` | `skills/` | 目录递归同步 |
| 用户 patch | `~/.dsh/profiles/<profile>/cordis.patch.yml` | `profiles/<profile>/patch.yml` | 仅用户层，不含 bundle 内置 |
| 设置 (可选) | `~/.dsh/settings.yaml` | `settings.yaml` | 排除已知秘钥字段 |

### 3.2 不同步项

| 条目 | 原因 |
|---|---|
| `node_modules/` | 构建产物，每台机器独立安装 |
| `.credentials.yaml` | 包含 API Key 等敏感信息 |
| Session 日志 | 数据量大，隐私敏感 |
| 内置 bundle 的 patch | 由 `dsh plugin add` 管理，非用户自定义 |
| 工作区文件 | 属于项目而非 dsh 配置 |

## 4. 架构设计

### 4.1 包结构

```
dsh-sync-plugin/
├── package.json              # dsh.bundle 声明
├── cordis.patch.yml          # 注册到 profile
├── src/
│   ├── index.ts              # 主入口：注册服务和工具
│   ├── syncer.ts             # 核心同步引擎
│   ├── github.ts             # GitHub API 封装
│   ├── plugin-scanner.ts     # 扫描本地已装插件清单
│   ├── skill-scanner.ts      # 扫描本地技能文件
│   ├── settings-scanner.ts   # 扫描设置（可选）
│   ├── differ.ts             # 本地 vs 远程差异对比
│   ├── applier.ts            # 应用远程变更到本地
│   ├── config.ts             # 插件配置类型
│   └── types.ts              # 类型定义
├── client/
│   └── settings-panel.tsx    # Web UI 设置面板（可选）
└── README.md
```

### 4.2 插件注册（cordis.patch.yml）

```yaml
- insert:
    - id: dsh-sync
      name: '@your-scope/dsh-sync-plugin'
      config:
        remote:
          url: ''          # GitHub 仓库 URL
          branch: main     # 同步分支
        profile: web       # 要同步的 profile
        include:
          plugins: true
          skills: true
          patch: true
          settings: false
        schedule: ''       # 可选 cron 表达式
```

### 4.3 配置定义

```typescript
export interface Config {
  remote: {
    url: string           // GitHub 仓库 URL，如 github:user/repo
    branch: string        // 同步分支，默认 main
  }
  profile: string         // 目标 profile 名称，如 web
  include: {
    plugins: boolean      // 同步插件清单
    skills: boolean       // 同步技能文件
    patch: boolean        // 同步用户 patch
    settings: boolean     // 同步设置（可选，默认 false）
  }
  schedule?: string       // 定时同步的 cron 表达式
  localOnlyPaths?: string[]  // 仅保留本地的路径模式
}
```

## 5. 同步协议

### 5.1 远程仓库结构

```
<sync-repo>/
├── profiles/
│   └── web/
│       ├── dependencies.json    # 插件依赖清单
│       ├── bundles.json         # bundle 激活列表
│       └── patch.yml            # 用户层 patch
├── skills/                      # 用户技能目录
│   ├── my-skill/
│   │   └── SKILL.md
│   └── ...
├── settings.yaml                # 设置（可选）
└── .dsh-sync.json               # 同步元信息
```

### 5.2 dependencies.json 格式

```json
{
  "version": 1,
  "profile": "web",
  "updatedAt": "2026-09-01T10:00:00Z",
  "dependencies": {
    "@studyzy/dsh-web-remote-access": "github:studyzy/dsh-web-remote-access#main",
    "dsh-market": "^0.3.0"
  },
  "bundles": [
    "@deepseek-ai/dsh-base",
    "dsh-market",
    "@studyzy/dsh-web-remote-access"
  ]
}
```

### 5.3 .dsh-sync.json 元信息

```json
{
  "version": 1,
  "lastSyncAt": "2026-09-01T10:00:00Z",
  "lastCommit": "abc123",
  "machineId": "device-a",
  "credentials": {
    "gitHubTokenRef": "DSH_SYNC_GITHUB_TOKEN"
  }
}
```

## 6. 核心流程

### 6.1 Push（推送本地 → 远程）

```
1. 扫描本地插件清单
   ├── 读取 profile/package.json 的 dependencies
   ├── 读取 dsh.profile.bundles
   └── 过滤掉 @deepseek-ai/dsh-* 内置包（它们由框架管理）

2. 扫描本地技能
   ├── 遍历 ~/.dsh/skills/ 目录
   ├── 收集所有 *.md 和 SKILL.md 文件
   └── 构建 skills 目录结构

3. 扫描用户 patch
   ├── 读取 ~/.dsh/profiles/<profile>/cordis.patch.yml
   └── 仅提取用户自定义的 insert 块（非 override 系统 id 的行）

4. 构建同步 payload
   ├── 排除凭证和秘钥字段
   ├── 排除配置的 local-only 路径
   └── 写入 .dsh-sync.json 元信息

5. Git 操作
   ├── git add .
   ├── git commit -m "sync from <machine> at <timestamp>"
   └── git push origin <branch>
```

### 6.2 Pull（拉取远程 → 本地）

```
1. Git 操作
   ├── git fetch origin <branch>
   └── git diff HEAD..origin/<branch> 获取变更

2. 差异分析
   ├── 对比 dependencies.json → 插件增减
   ├── 对比 bundles.json → bundle 顺序变化
   ├── 对比 skills/ → 技能文件增删改
   └── 对比 patch.yml → 用户 patch 变更

3. 应用变更（逐项确认）
   ├── 新增插件 → 提示用户确认，执行 dsh plugin add
   ├── 移除插件 → 提示用户确认，执行 dsh plugin remove
   ├── 技能变更 → 直接同步文件（安全）
   └── patch 变更 → 逐行审查后应用（影响代码执行）

4. 生成变更报告
   ├── 新增 N 个插件
   ├── 移除 M 个插件
   ├── 更新 K 个技能文件
   └── 变更 X 行 patch
```

### 6.3 冲突处理

```
本地 vs 远程 同时变更同一文件：
├── settings.yaml → 按 key 级别 cherry-pick
├── dependencies.json → 包级别合并（取并集，版本冲突时提示）
├── skills/* → 按文件级别选择（keep local / take remote）
└── patch.yml → 逐行审查（可单独 toggle 每行）
```

## 7. Model-facing Tool 设计

### 7.1 sync_push

```typescript
ctx.tools.register(defineTool({
  name: 'sync_push',
  description: 'Push local plugins, skills, and config to the sync repository.',
  parameters: {
    message: { type: 'string', description: 'Optional commit message' },
    include: {
      type: 'object',
      properties: {
        plugins: { type: 'boolean' },
        skills: { type: 'boolean' },
        patch: { type: 'boolean' },
      },
    },
  },
  async execute(args, exec) {
    // 执行同步推送
    const result = await syncer.push(args)
    return {
      pushed: true,
      commit: result.commit,
      changes: {
        plugins: result.pluginChanges,
        skills: result.skillChanges,
        patch: result.patchChanges,
      },
    }
  },
  presentCall: args => ({ card: 'generic', title: 'Sync push', kind: 'other' }),
  presentResult: (_args, result) => ({
    card: 'generic',
    title: 'Sync pushed',
    content: `Commit ${result.commit.slice(0, 7)}: ${result.changes.plugins} plugins, ${result.changes.skills} skills`,
  }),
}))
```

### 7.2 sync_pull

```typescript
ctx.tools.register(defineTool({
  name: 'sync_pull',
  description: 'Pull plugins, skills, and config from the sync repository.',
  parameters: {
    dryRun: { type: 'boolean', description: 'Preview changes without applying' },
    autoApply: { type: 'boolean', description: 'Auto-apply safe changes (skills only)' },
  },
  async execute(args, exec) {
    const diff = await syncer.fetchDiff()
    if (args.dryRun) {
      return { dryRun: true, changes: diff }
    }
    const result = await syncer.apply(diff, { autoApply: args.autoApply ?? false })
    return result
  },
}))
```

### 7.3 sync_status

```typescript
ctx.tools.register(defineTool({
  name: 'sync_status',
  description: 'Show sync status between local and remote.',
  async execute() {
    const local = await syncer.getLocalState()
    const remote = await syncer.getRemoteState()
    return {
      lastSyncAt: local.lastSyncAt,
      localCommit: local.commit,
      remoteCommit: remote.commit,
      ahead: local.ahead,     // 本地领先的 commit 数
      behind: local.behind,   // 本地落后的 commit 数
      localPlugins: local.pluginCount,
      remotePlugins: remote.pluginCount,
      localSkills: local.skillCount,
      remoteSkills: remote.skillCount,
    }
  },
}))
```

## 8. 安全设计

### 8.1 凭证管理

使用 dsh 内置的 `CredentialProvider` 存储 GitHub Token：

```yaml
# ~/.dsh/.credentials.yaml
version: 1
records:
  dsh-sync-plugin/github:
    kind: api-key
    key: ghp_xxxxxxxxxxxx
```

### 8.2 秘钥过滤

同步 `settings.yaml` 时自动过滤已知秘钥字段：

```typescript
const SECRET_FIELDS = [
  'apiKey', 'api_key', 'apikey',
  'token', 'secret', 'password',
  // 可配置扩展
]
```

### 8.3 执行变更确认

patch 文件和 plugin 清单变更影响代码执行，需要用户显式确认：

```typescript
if (change.kind === 'executable') {
  const confirmed = await exec.agent.inject({
    content: `The following changes may affect code execution:\n${change.diff}\n\nApply?`,
    source: { kind: 'plugin', plugin: 'dsh-sync' },
  })
  if (!confirmed) return { skipped: true }
}
```

## 9. 与现有生态对比

| 特性 | dsh-sync (现有) | dsh-plugin-hub | dsh-sync-plugin (本方案) |
|---|---|---|---|
| 同步插件清单 | ✗ | ✗ | ✓ |
| 同步技能文件 | ✗ | ✗ | ✓ |
| 同步设置 | ✓ | ✗ | ✓ (可选) |
| 同步 patch | ✓ | ✗ | ✓ |
| 跨设备同步 | ✓ | ✗ | ✓ |
| 插件市场浏览 | ✗ | ✓ | ✗ |
| Model-facing Tool | ✗ | ✗ | ✓ |
| 秘钥感知 | ✓ | — | ✓ |
| 冲突处理 | ✓ | — | ✓ |
| 定时同步 | ✗ | ✗ | ✓ (可选) |

## 10. 开发路线图

### Phase 1: 核心同步引擎
- [ ] GitHub API 封装（clone/pull/push/commit/diff）
- [ ] 插件清单扫描与序列化
- [ ] 技能文件扫描与同步
- [ ] 基本的 push/pull 流程

### Phase 2: Model-facing Tools
- [ ] `sync_push` tool 实现
- [ ] `sync_pull` tool 实现（含 dry-run 预览）
- [ ] `sync_status` tool 实现
- [ ] 差异对比与冲突检测

### Phase 3: 安全与可用性
- [ ] 集成 CredentialProvider 管理 GitHub Token
- [ ] 秘钥字段过滤
- [ ] 执行变更确认机制
- [ ] 备份与恢复（中断写入保护）

### Phase 4: Web UI 与定时同步
- [ ] Web 设置面板（GitHub 配置、同步范围选择）
- [ ] 一键 push/pull 按钮
- [ ] 变更预览 UI
- [ ] 定时同步（cron 表达式调度）

## 11. 技术依赖

| 依赖 | 用途 | 来源 |
|---|---|---|
| `@deepseek-ai/cordis` | 插件框架 | peerDependency |
| `@deepseek-ai/dsh-tools` | 工具注册 | peerDependency |
| `@deepseek-ai/dsh-credentials` | 凭证存储 | peerDependency |
| `@deepseek-ai/dsh-settings` | 设置读取 | peerDependency |
| `simple-git` | Git 操作 | dependency |
| `zod` | 配置校验 | dependency |