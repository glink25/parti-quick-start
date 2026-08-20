# Parti Quick Start

面向 Parti Room 的通用开发基线。这个仓库用于快速启动**任意类型的游戏项目**，只描述 Parti Runtime、工程结构、UI / 美术、轻量验证与发布方式，不绑定任何具体游戏、规则或题材。

## 默认技术选择

- **优先使用 Vite + TypeScript** 开发 UI 与规则代码。
- 默认使用 DOM / CSS；复杂 Canvas、精灵、粒子、摄像机或实时画面可以使用 **LittleJS**。
- LittleJS 是可选能力，不是所有项目的默认依赖。
- `room.worker.js` 是最终构建产物，源码可以自由拆分。
- Worker 构建必须保留 `@parti/worker-sdk` 中 `defineRoom` 的运行时 import。

## 快速开发优先

Parti 的目标是快速开发、试玩和迭代小游戏，因此默认验证流程应保持轻量：

```text
能构建
-> 能被 Parti 加载
-> 开发者 / 玩家实际试玩
```

自动化测试按项目需要添加，不要求为了 quick-start 建立复杂测试体系。

视觉、交互手感和“是否好玩”主要由开发者和玩家实际验证。AI 可以做代码、配置和构建检查，但不应默认用截图分析、像素比对或大量页面操作代替真人做视觉与体验验收。

## 文档

- [开发规范](docs/development.md)：Parti 心智模型、工程结构、Manifest、Vite / Worker 构建。
- [Runtime 与验证](docs/runtime-and-testing.md)：权威状态、必要 action 校验、轻量验证和真人试玩原则。
- [UI 与美术规范](docs/ui-and-art.md)：移动端、首屏、游戏感、交互反馈与响应式建议。
- [发布规范](docs/release.md)：构建产物、打包和最小发布检查。

## 最小开发心智模型

Parti Room 最终由三个核心入口组成：

```text
parti.room.json   -> 包元信息、入口、人数与权限
index.html        -> 玩家 UI，运行在沙箱 iframe
room.worker.js    -> Host 权威逻辑，运行在 Web Worker
```

数据流：

```text
玩家 UI
  -> parti.action(name, payload)
  -> Host Runtime
  -> room.worker.js action handler
  -> 修改 ctx.state
  -> Runtime 广播 state snapshot
  -> 各客户端 parti.onState(state)
```

核心原则：**UI 提交意图，Worker 决定结果。** 不重复实现 Parti 已经提供的网络、snapshot、seq / ack 等同步机制。

## 推荐开始方式

```bash
npm create vite@latest my-parti-room -- --template vanilla-ts
cd my-parti-room
npm install
```

之后按 [开发规范](docs/development.md) 加入 Worker 构建和 `public/parti.room.json`。

> 本仓库是开发基线，不替代 Parti 官方文档。发生冲突时，以 `glink25/Parti` 当前官方文档与 Runtime 实际行为为准。
