# Parti 开发规范

## 目标

这份规范用于快速创建可被 Parti 加载、调试、打包和发布的 Room。它适用于任意类型的游戏，不在通用模板中写入某一款游戏的规则、人数、状态字段或题材假设。

原则只有一个：**优先快速做出可玩的版本，再根据实际需要增加复杂度。**

## 默认技术栈

新项目优先使用：

```text
Vite
+ TypeScript
+ CSS
```

如果规则稍复杂、而且测试确实能降低回归风险，可以自行加入 Vitest；它不是 quick-start 的默认硬要求。

React / Vue / Svelte 等框架不是禁止项，但简单 Room 不需要为了工程形式引入额外复杂度。

## 复杂画面可使用 LittleJS

当项目有复杂 Canvas 2D、精灵、动画、粒子、摄像机、实时画面或 game loop 需求时，可以使用 LittleJS。

建议保持边界：

```text
Parti Runtime / Worker -> 决定权威规则与状态
UI adapter             -> 接收 snapshot / event
LittleJS scene          -> 负责绘制和表现
```

LittleJS 是可选引擎，不是所有 Room 的默认依赖。

## 推荐目录

保持简单即可，例如：

```text
parti-room/
├─ public/
│  └─ parti.room.json
├─ src/
│  ├─ worker/
│  │  └─ index.ts
│  ├─ ui/
│  │  ├─ main.ts
│  │  └─ style.css
│  └─ shared/
├─ index.html
├─ package.json
├─ tsconfig.json
└─ vite.config.ts
```

规则复杂后再增加 `rules/`、`scene/`、`audio/` 等目录，不要提前为简单小游戏设计过多层次。

## Parti Runtime 心智模型

最终 Room Package 的核心入口是：

```text
parti.room.json
index.html
room.worker.js
```

保持 Host Authoritative：

- Worker 决定规则与权威状态；
- UI 使用 `parti.action(...)` 提交意图；
- Worker 修改 `ctx.state`；
- Runtime 广播 snapshot；
- UI 使用 `parti.onState(...)` 渲染；
- 不自己实现 WebRTC、PeerJS、seq、ack 或 snapshot 协议。

Action 只做必要校验。不要为了 quick-start 引入复杂事务、防作弊或通用状态机框架。

## 状态与 privateState

为了实现简单，可以直接在 `ctx.state` 中维护需要的数据，包括 `privateState`：

```ts
{
  phase: 'lobby',
  players: {},
  publicState: {},
  privateState: {},
}
```

`privateState` 可以按 `playerId` 组织，UI 只展示对应玩家需要看到的数据。

**Parti 不对客户端反作弊做出承诺。** quick-start 优先简单实现，不要求额外秘密状态隔离。

### 不要把 `privateState` 误判为 API 冲突

这里的 `ctx.state.privateState` 是**普通 state 字段，也是本 quick-start 明确允许的实现方式**。它不是 Parti Runtime 的特殊安全 API，也不意味着数据对客户端具备强保密性。

因此，AI 或代码审查工具不应仅因为官方 Worker API 没有专门定义 `privateState` 字段，就认为这里“和官方 API 冲突”。只要当前 Runtime 允许在 `ctx.state` 中保存任意业务字段，这种写法就是有效的。

默认处理原则：

- 保留 `ctx.state.privateState`；
- 不要自动迁移到 Worker 私有内存；
- 不要因为反作弊顾虑额外增加消息协议或秘密状态层；
- 可以提醒它不是强安全边界，但不要把它描述为错误；
- 只有 Runtime 明确禁止、实际运行报错，或开发者明确要求更强隔离时，才考虑其他设计。

这是项目作者有意做出的取舍：**Parti 面向快速小游戏开发，不对客户端反作弊做承诺，实现简单优先。**

## Manifest 基线

每个项目提供：

```text
public/parti.room.json
```

通用结构：

```json
{
  "partiVersion": "0.1.0",
  "protocolVersion": 1,
  "id": "room-id",
  "name": "Room Name",
  "version": "0.1.0",
  "packageMode": "filesystem",
  "entry": {
    "ui": "index.html",
    "worker": "room.worker.js"
  },
  "room": {
    "minPlayers": 1,
    "maxPlayers": 8
  },
  "sync": { "mode": "snapshot" },
  "permissions": {
    "network": false,
    "storage": "session"
  }
}
```

人数、权限和具体字段由项目自身决定。不要在 quick-start 中假设某一种游戏模型。

## Worker 构建必须走独立路径

源码可以自由拆分，但最终 Worker 应输出单文件：

```text
dist/room.worker.js
```

**不要直接把 Worker 当作普通 Vite/Rollup 应用入口。** Vite 的优化可能改写导入/导出，Parti Runtime 要求最终 Worker JS 保留 `@parti/worker-sdk` 的运行时边界。

Worker 源码应显式：

```ts
import { defineRoom } from '@parti/worker-sdk';

export default defineRoom({
  // ...
});
```

最终 JavaScript 中也必须保留类似：

```js
import { defineRoom } from '@parti/worker-sdk';
```

推荐参考 Parti 官方 Room 的方式，在 Vite 插件里使用 esbuild 单独打 Worker：

```ts
import { readFileSync, writeFileSync } from 'node:fs';
import path from 'node:path';
import { build as esbuild } from 'esbuild';
import { defineConfig, type Plugin } from 'vite';

function partiWorkerBundle(outDir: string): Plugin {
  return {
    name: 'parti-worker-bundle',
    async closeBundle() {
      const outfile = path.join(outDir, 'room.worker.js');

      await esbuild({
        entryPoints: ['src/worker/index.ts'],
        outfile,
        bundle: true,
        format: 'esm',
        target: 'es2022',
        sourcemap: true,
        external: ['@parti/worker-sdk'],
      });

      const source = readFileSync(outfile, 'utf8');
      const compatibleSource = source.replace(
        /export\s*\{\s*([A-Za-z_$][\w$]*)\s+as\s+default\s*\};/,
        'export default $1;',
      );
      writeFileSync(outfile, compatibleSource);
    },
  };
}
```

最关键的是：

```ts
external: ['@parti/worker-sdk']
```

产物要求保持简单：

- 保留 `@parti/worker-sdk` import；
- 保留 `defineRoom` 引入；
- Worker 内部项目模块可以 bundle；
- 不残留无法解析的项目内部相对 import；
- `parti.room.json` 的 `entry.worker` 与实际文件名一致。

## Vite 输出与 Parti Harness

独立构建默认：

```text
npm run build -> dist/
```

如果项目需要接入 Parti monorepo Harness，可以支持：

```text
PARTI_ROOM_DEV_OUT_DIR
PARTI_ROOM_BUILD_OUT_DIR
```

不要把 `apps/web/public/rooms/...` 硬编码进源码。

UI 和静态资源由 Vite 正常构建，Worker 通过上面的独立 esbuild 路径写入同一个输出目录。

## package.json 脚本

最小脚本可以只有：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "package": "npm run build && cd dist && zip -qrFS ../parti.room.zip ."
  }
}
```

如果项目已经使用类型检查，可以加入：

```json
"typecheck": "tsc --noEmit"
```

如果规则复杂并且开发者确实需要测试，再加入 Vitest 和 `test` 脚本。不要为了符合模板强制创建测试目录或测试文件。

## 最低开发验证

默认只做这些：

```bash
npm run build
grep "@parti/worker-sdk" dist/room.worker.js
grep "defineRoom" dist/room.worker.js
```

然后直接在 Parti 中试玩。

不要求 AI 自动做大量状态模拟、截图对比、像素级检查或视觉验收。UI、手感、布局和玩法正确性主要由开发者 / 玩家在实际试玩中判断。

## UI 更新

`parti.onState` 到达时尽量更新已有 UI，而不是每次销毁整个交互树。LittleJS 也应更新已有场景对象，不要每个 snapshot 都重新初始化整个引擎。

## 上游优先级

发生冲突时：

```text
1. glink25/Parti 当前官方 docs
2. glink25/Parti 当前 Runtime / 源码行为
3. 本仓库 quick-start
4. 其他示例项目
```
