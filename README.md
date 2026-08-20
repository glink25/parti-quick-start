# Parti Quick Start

面向 Parti Room 的通用开发基线。这个仓库用于快速启动**任意类型的游戏项目**，只描述 Parti Runtime、工程结构、UI / 美术、测试与发布约束，不绑定任何具体游戏、规则或题材。

## 默认技术选择

- **优先使用 Vite + TypeScript** 开发 UI 与规则代码。
- 默认使用 DOM / CSS 实现界面；当项目需要复杂 Canvas 场景、大量精灵、粒子、摄像机或实时画面时，可以引入 **LittleJS** 作为游戏引擎。
- LittleJS 是可选能力，不应成为所有项目的默认依赖。
- 游戏规则与状态转换尽量写成可独立测试的纯 TypeScript 模块。
- `room.worker.js` 是最终构建产物，不要求源码写成单文件。

## 文档

- [开发规范](docs/development.md)：Parti 心智模型、推荐工程结构、Manifest、Vite / Worker 构建约束。
- [Runtime 与测试规范](docs/runtime-and-testing.md)：权威状态、action 校验、状态机、随机、重连与测试基线。
- [UI 与美术规范](docs/ui-and-art.md)：移动端、首屏、游戏感、交互反馈、响应式与性能要求。
- [发布规范](docs/release.md)：构建产物、版本、Release、package branch 与市场登记建议。

## 最小开发心智模型

Parti Room 最终由三个核心入口组成：

```text
parti.room.json   -> 包元信息、入口、人数与权限
index.html        -> 玩家 UI，运行在沙箱 iframe
room.worker.js    -> Host 权威逻辑，运行在 Web Worker
```

数据流保持为：

```text
玩家 UI
  -> parti.action(name, payload)
  -> Host Runtime
  -> room.worker.js action handler
  -> 修改 ctx.state
  -> Runtime 广播 state snapshot
  -> 各客户端 parti.onState(state)
```

核心原则：**UI 只提交意图，Worker 决定结果。** 不重复实现 Parti 已经提供的网络、snapshot、seq / ack 等同步机制。

## 推荐开始方式

```bash
npm create vite@latest my-parti-game -- --template vanilla-ts
cd my-parti-game
npm install
```

之后按 [开发规范](docs/development.md) 调整目录、加入 Worker 构建和 `public/parti.room.json`。

> 本仓库是开发基线，不替代 Parti 官方文档。发生冲突时，以 `glink25/Parti` 当前官方文档与 Runtime 实际行为为准。
