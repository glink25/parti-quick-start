# Parti 开发规范

## 目标

这份规范用于快速创建可被 Parti 加载、调试、打包和发布的 Room。它必须适用于任意类型的游戏，不应在通用模板中写入某一款游戏的规则、人数、状态字段或题材假设。

## 技术栈原则

### 默认使用 Vite + TypeScript

新项目优先采用：

```text
Vite
+ TypeScript
+ CSS
+ Vitest
```

原因：启动快、构建简单、静态资源处理成熟，并且便于将规则、UI 与 Worker 源码拆成独立模块后再统一打包。

除非已有项目有明确理由，不建议为了简单 Room 引入大型应用框架。React / Vue / Svelte 等不是禁止项，但不应成为 quick-start 的硬依赖。

### 复杂画面可使用 LittleJS

当画面出现下列需求时，可以使用 LittleJS：

- Canvas 2D 主场景；
- 大量精灵或动画对象；
- 粒子、摄像机、屏幕震动等实时表现；
- 需要稳定 game loop 的高频视觉更新；
- DOM / CSS 实现会明显增加复杂度的场景。

建议保持边界：

```text
Parti Runtime / Worker -> 决定权威规则与状态
UI adapter             -> 接收 snapshot / event
LittleJS scene          -> 负责绘制和表现
```

不要把 LittleJS 的本地对象状态当作权威游戏状态，也不要让引擎自行决定多人规则结果。

## 推荐源码结构

```text
parti-room/
├─ public/
│  ├─ parti.room.json
│  └─ cover.jpg
├─ src/
│  ├─ worker/
│  │  └─ index.ts
│  ├─ rules/
│  │  ├─ types.ts
│  │  ├─ state.ts
│  │  ├─ actions.ts
│  │  └─ validation.ts
│  ├─ ui/
│  │  ├─ main.ts
│  │  └─ style.css
│  └─ shared/
│     └─ constants.ts
├─ tests/
├─ index.html
├─ package.json
├─ tsconfig.json
├─ vite.config.ts
└─ vitest.config.ts
```

复杂项目可按需增加：

```text
src/audio/
src/scene/
src/geometry/
src/content/
src/ai/
```

目录只是职责建议，不是固定游戏模型。

## Parti Runtime 架构

最终 Room Package 的核心入口是：

```text
parti.room.json
index.html
room.worker.js
```

必须保持 Host Authoritative：

- Worker 是规则与权威状态的唯一决定者；
- UI 使用 `parti.action(...)` 提交意图；
- action payload 一律视为不可信输入，由 Worker 校验；
- Worker 修改 `ctx.state`；
- Runtime 完成 snapshot 同步；
- UI 通过 `parti.onState(...)` 渲染；
- 不手写 WebRTC、PeerJS、seq、ack 或 snapshot 协议。

当前 action handler 应同步执行。需要延迟逻辑时优先使用 Parti 提供的 timer 能力，而不是让 action handler 变成异步流程。

## 状态与 privateState

为了实现简单，可以在 `ctx.state` 中维护统一的状态骨架，包括 `publicState`、`privateState`、`pendingChoices` 等字段。

例如：

```ts
{
  revision: 0,
  phase: 'lobby',
  players: {},
  publicState: {},
  privateState: {},
  pendingChoices: {},
  eventLog: [],
}
```

`privateState` 可以按 `playerId` 组织，UI 只展示当前玩家应该看到的部分。

**安全边界：Parti 不对客户端反作弊做出承诺。** `ctx.state` 的同步模型不应被当作强保密或可信执行环境。quick-start 优先保证实现简单、一致和可维护；如果某个项目自行需要更强的秘密隔离，应由该项目额外设计，而不是提升为所有 Room 的通用复杂度。

## Manifest 基线

每个项目提供：

```text
public/parti.room.json
```

通用示例：

```json
{
  "partiVersion": "0.1.0",
  "protocolVersion": 1,
  "id": "room-id",
  "name": "Room Name",
  "version": "0.1.0",
  "description": "...",
  "packageMode": "filesystem",
  "entry": {
    "ui": "index.html",
    "worker": "room.worker.js"
  },
  "room": {
    "minPlayers": 1,
    "maxPlayers": 8,
    "allowSpectators": true
  },
  "sync": { "mode": "snapshot" },
  "permissions": {
    "network": false,
    "storage": "session"
  }
}
```

注意：

- `protocolVersion` 当前为 `1`；
- 当前以 `snapshot` 为有效同步基线；
- 有模块、图片、CSS、音频等资源时优先使用 `filesystem`；
- `entry.ui` / `entry.worker` 必须与最终产物一致；
- 权限按最小需求申请；
- `package.json.version` 与 manifest `version` 保持一致；
- `room.minPlayers` / `maxPlayers` 应由具体项目定义，不在通用模板里假设某类游戏人数。

## Worker 必须打成单文件

源码可以自由拆分，但最终必须输出：

```text
dist/room.worker.js
```

建议用 esbuild 或 Vite 插件额外构建 Worker：

```ts
await esbuild({
  entryPoints: ['src/worker/index.ts'],
  outfile: `${outDir}/room.worker.js`,
  bundle: true,
  format: 'esm',
  target: 'es2022',
  external: ['@parti/worker-sdk'],
});
```

产物约束：

- 保留 `@parti/worker-sdk` import；
- 默认导出 `defineRoom(...)` 结果；
- 不保留项目内部相对模块 import；
- Worker 内部规则模块被 bundle；
- `initialState` 使用函数。

## Vite 输出与 Parti Harness

独立构建默认：

```text
npm run build -> dist/
```

同时建议支持 Parti monorepo Harness 使用的环境变量：

```text
PARTI_ROOM_DEV_OUT_DIR
PARTI_ROOM_BUILD_OUT_DIR
```

不要把 `apps/web/public/rooms/...` 写死到项目源码。

推荐输出目录解析：

```ts
function resolveOutDir(mode: string) {
  if (mode === 'room-dev') {
    if (!process.env.PARTI_ROOM_DEV_OUT_DIR) {
      throw new Error('missing PARTI_ROOM_DEV_OUT_DIR');
    }
    return process.env.PARTI_ROOM_DEV_OUT_DIR;
  }

  if (mode === 'room-build') {
    if (!process.env.PARTI_ROOM_BUILD_OUT_DIR) {
      throw new Error('missing PARTI_ROOM_BUILD_OUT_DIR');
    }
    return process.env.PARTI_ROOM_BUILD_OUT_DIR;
  }

  return 'dist';
}
```

Vite 构建建议：

```ts
build: {
  outDir,
  emptyOutDir: true,
  assetsInlineLimit: 0,
  target: 'es2022',
}
```

## package.json 脚本

推荐至少提供：

```json
{
  "scripts": {
    "dev": "vite",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "build": "npm run typecheck && vite build",
    "package": "npm run build && cd dist && zip -qrFS ../parti.room.zip .",
    "dev:room": "vite build --watch --mode room-dev",
    "build:room": "vite build --mode room-build"
  }
}
```

最低本地验收：

```bash
npm ci
npm test
npm run build
npm run package
```

## UI 更新原则

`parti.onState` 可能频繁触发。不要在每个 snapshot 上销毁整个交互 DOM：

```ts
parti.onState((state) => {
  if (state.phase !== currentPhase) {
    mountPhase(state.phase);
  }

  updatePlayers(state);
  updateScene(state);
  updateControls(state);
});
```

保持 pointer target、focus、拖拽对象与关键 DOM 节点稳定。LittleJS 场景同样应以 snapshot 更新模型，而不是重建整个引擎实例。

## 上游优先级

发生冲突时按以下优先级处理：

```text
1. glink25/Parti 当前官方 docs
2. glink25/Parti 当前 Runtime / 源码行为
3. 本仓库 quick-start 规范
4. 其他示例项目
```
