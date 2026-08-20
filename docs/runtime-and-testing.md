# Runtime 与测试规范

## 权威逻辑

- Worker 是唯一权威规则源；
- 客户端只提交 action 与渲染状态；
- action 必须校验当前玩家、阶段、权限、参数合法性和重复提交；
- UI 不复制一套会影响最终结果的规则逻辑；
- 所有状态转换尽量由纯 TypeScript 函数完成，Worker 只负责 Parti lifecycle 与接线。

## 推荐状态骨架

状态字段应由具体项目裁剪，不要求所有项目完全一致。一个通用起点是：

```ts
{
  revision: 0,
  phase: 'lobby',
  round: 0,
  activePlayerId: null,
  players: {},
  publicState: {},
  privateState: {},
  pendingChoices: {},
  lockedPlayers: [],
  eventLog: [],
}
```

其中 `privateState` 可以按玩家保存只应在 UI 层选择性展示的数据。Parti 不承诺客户端反作弊，因此这里以开发简单和状态可恢复为优先，不把“客户端不可见”当成强安全边界。

## 状态机

中等以上复杂度项目建议显式使用 `phase`。

要求：

- 每个 phase 有明确的合法 action 白名单；
- phase 转换由 Worker 原子完成；
- 不依赖客户端自行推进阶段；
- 状态恢复尽量只依赖 `ctx.state`；
- 浏览器本地计时器不应成为恢复所必需的唯一信息。

不要把某一套固定 phase 名称强加给所有类型的游戏；具体项目按自身规则定义。

## Action 校验

每个 action 至少考虑：

```text
玩家是否仍在房间
当前 phase 是否允许
当前玩家是否有权限执行
payload 是否属于合法集合
是否已经锁定 / 已经结算
重复提交是否会重复产生副作用
```

不要相信客户端传来的分数、结果、目标合法性、随机结果或“已校验”标记。

## 重复提交与幂等

有锁定、确认、购买、结算等不可重复副作用时，应保证同一意图重复到达不会重复结算。

可以使用：

- 状态中的 `revision`；
- action 内部业务唯一键；
- `clientActionId`；
- 已完成记录；
- phase / locked 状态。

选择最简单且足够的方案，不要求每个项目都实现完整事务协议。

## 同时决策

需要多人同时选择时，可采用：

```text
choose -> lock -> allLocked -> reveal / resolve
```

建议：

- 未锁定选择允许覆盖；
- 锁定后拒绝修改；
- 断线玩家的已锁定选择继续有效；
- 未锁定断线玩家由项目自己定义等待、默认动作或终止策略。

## 随机

所有会影响规则结果的随机应由 Worker 决定。

为了测试与复盘，推荐将 RNG 作为规则函数依赖注入：

```ts
resolveAction(state, input, rng)
```

测试使用固定 seed 或固定 RNG，使结果确定可重复。

## Timer

不要让 action handler 依赖 `async` 延迟流程。需要延迟 / 超时时使用 Parti Runtime 提供的 timer 能力，并把足以恢复逻辑的状态写入 `ctx.state`。

如果项目并不需要实时倒计时，优先不增加 timer 复杂度。

## 测试分层

### 规则单元测试

测试纯函数：

- 初始化；
- 合法动作；
- 非法输入；
- 资源边界；
- 计分 / 胜负；
- 固定随机种子下的确定性。

### 状态机测试

覆盖：

- phase 转换；
- action 白名单；
- 重复 action；
- 锁定后修改；
- 终局后不再接受普通 gameplay action。

### 多玩家模拟

对存在长流程或大量状态组合的项目，批量生成合法动作直到终局，用于发现：

- 死锁；
- 永远无法结束；
- revision 异常；
- 资源负数或溢出；
- phase 无出口。

### 重连测试

至少验证：

- 玩家离开 / 返回后 UI 能从 snapshot 恢复；
- 当前 phase 不错乱；
- 已锁定 / 已结算状态不会丢失；
- 不依赖已丢失的 DOM 或本地 timer 才能继续。

### Worker 产物兼容性检查

不能只检查 TypeScript 源码。每次构建后都应直接验证最终 Worker JavaScript：

```bash
grep "@parti/worker-sdk" dist/room.worker.js
grep "defineRoom" dist/room.worker.js
```

要求：

- Worker 产物仍包含来自 `@parti/worker-sdk` 的 import；
- `defineRoom` 没有被 Vite/Rollup/esbuild 内联或替换掉；
- 项目内部相对模块已经 bundle，不再残留运行时无法解析的内部 import；
- Worker 保持可识别的默认导出；
- `parti.room.json` 的 `entry.worker` 与实际产物名称一致。

这项检查属于构建契约测试，和规则单元测试同样重要。

## 最低验收

每个项目至少覆盖：

- [ ] 最少人数可以进入完整流程；
- [ ] 最大人数可以进入完整流程；
- [ ] 非法阶段 action 被拒绝；
- [ ] 非法参数被拒绝；
- [ ] 重复 action 不会重复结算；
- [ ] 关键边界条件有测试；
- [ ] 玩家断线 / 重连后状态一致；
- [ ] 正常终局可达；
- [ ] rematch / restart（若项目支持）不继承脏状态；
- [ ] 固定 RNG 下结果可复现；
- [ ] `dist/parti.room.json`、`dist/index.html`、`dist/room.worker.js` 均存在；
- [ ] `dist/room.worker.js` 仍保留 `@parti/worker-sdk` import；
- [ ] `dist/room.worker.js` 中仍存在 `defineRoom` 引用。
