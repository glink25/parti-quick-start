# UI 与美术规范

## 核心目标

Parti Room 首先应让玩家感受到“这是一个游戏界面”，而不是后台、表单或普通长网页。

通用优先级：

- 游戏感优先于网页感；
- 主界面完整呈现在当前视口；
- 移动端触控优先；
- 核心状态尽量视觉化；
- 重要操作有连续、可感知反馈；
- 空间不足通过重新布局解决，而不是让整个页面纵向增长。

## 主界面不要滚动

推荐基础：

```css
html,
body,
#app {
  width: 100%;
  height: 100%;
  margin: 0;
  overflow: hidden;
}

body {
  min-height: 100dvh;
  height: 100dvh;
  overscroll-behavior: none;
  touch-action: manipulation;
}

.game-shell {
  width: 100vw;
  height: 100dvh;
  overflow: hidden;
  display: grid;
}
```

空间不足时优先：

- 缩减低优先级文字与装饰；
- 调整元素大小 / 间距；
- 切换布局方向；
- 使用重叠、分页、横向轨道、缩略视图；
- 将低频信息放入 modal / drawer / tooltip；
- 针对不同屏幕比例使用不同布局。

规则、百科、历史、设置可以在独立弹层内部滚动，但不要让主游戏 shell 滚动。

## 移动端优先

至少验证：

```text
320 x 568
360 x 640
390 x 844
430 x 932
手机横屏
平板
桌面浏览器
```

主要触控目标：

- 推荐至少 `48 x 48 px`；
- 不低于 `44 x 44 px`；
- 高风险操作之间保留明显间隔；
- 关键能力不得依赖 hover；
- 主要游戏对象本身应可点击，而不是只提供很小的附属按钮。

## Safe Area

同时考虑移动设备安全区与 Parti 全屏右上角宿主控件。

```css
.game-shell {
  padding-top: env(safe-area-inset-top, 0px);
  padding-left: env(safe-area-inset-left, 0px);
  padding-bottom: env(safe-area-inset-bottom, 0px);
  padding-right: max(88px, env(safe-area-inset-right, 0px));
}
```

关键按钮不要固定在可能被宿主控件覆盖的右上角。

## 游戏化表达

不要把核心玩法呈现成：

```text
状态文本
下拉框
输入框
提交按钮
数据表格
```

有天然实体概念时，优先转成：

```text
卡牌
棋子
图标
筹码 / token
轨道 / 仪表
棋盘区域
角色 / 玩家席位
场景对象
```

这条原则同样适用于 LittleJS：画布中的对象应传达游戏状态，而不是把 Canvas 当成普通仪表盘背景。

## 视觉层级

同一屏幕应清楚区分：

1. 玩家现在需要做的决定；
2. 公共状态；
3. 当前玩家相关状态；
4. 其他玩家可见状态；
5. 历史、帮助等低频信息。

玩家不应通过阅读大段文字才能判断“现在要做什么”。

## 交互反馈

主要交互元素至少考虑：

```text
idle
hover（桌面增强）
pressed
selected
locked
disabled
resolved
error / invalid
```

提交 action 后立刻给本地视觉反馈；Worker 返回新 snapshot 后再更新权威结果。

不要出现“点了但看起来什么也没发生”。

## 动画

动画的作用是解释状态变化，不是拖慢流程。

建议时长：

```text
普通反馈           100-180ms
对象移动 / Reveal  180-400ms
关键结算           400-900ms
```

Worker 决定规则结果，UI 动画负责表现结果。

支持 reduced motion：

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

核心信息不能只靠动画表达。

## DOM 更新稳定性

避免每次 snapshot 都：

```ts
app.innerHTML = renderEverything(state);
```

因为会销毁 pointer capture、focus 和正在进行中的触摸交互。

推荐 phase 切换时 mount，新 snapshot 只 patch 变化。

如果使用 LittleJS，不要因为 snapshot 到达就重新初始化整个引擎；只更新场景模型和必要对象。

## 响应式

响应式应重新组织布局，而不是只做整体缩放。

推荐使用：

```text
CSS Grid
Flexbox
clamp()
min()
max()
aspect-ratio
container queries
```

例如：

```css
.game-object {
  width: clamp(64px, 18vw, 132px);
}
```

Canvas / LittleJS 项目也要按 viewport 重新计算镜头、UI safe area 与交互热区。

## 美术完整度

不要求高成本原创插画，但应至少具备：

- 明确主题色；
- 统一的表面 / 面板层级；
- 一套统一图标；
- 主题化背景或场景；
- 一致的圆角、边框、阴影语言；
- 可选 / 禁用 / 危险 / 成功等状态视觉；
- 明确的胜负 / 结束表现。

避免默认 Bootstrap、Tailwind dashboard、admin panel、白底表单风格直接作为最终游戏 UI。

## 封面

正式发布建议提供封面，并保证在市场缩略图尺寸下仍能辨认。

简单图形封面可以使用自绘 SVG。复杂插画封面建议使用 JPG，并控制文件大小；若沿用现有发布基线，复杂 JPG 建议不超过 `200 KB`。

不要使用占位图、开发截图、带水印素材或明显侵权素材作为正式封面。

## 中文

正式版本应完整支持中文：

- 核心按钮；
- 阶段提示；
- 胜负信息；
- 关键规则说明；
- 主要资源 / 对象名称。

中文文案需要适配窄屏，不应通过增加主页面高度来容纳翻译。

## 性能

移动端优先：

- 避免同时加载大量高分辨率图片；
- 优先 WebP / AVIF，图标可用 SVG；
- 动画优先 `transform` / `opacity`；
- 避免高频 layout thrashing；
- 不在每个 snapshot 全量重建 DOM；
- Canvas / WebGL / LittleJS 项目限制 DPR，例如：

```ts
const dpr = Math.min(window.devicePixelRatio || 1, 2);
```

## 发布前 UI 检查

- [ ] 主页面和游戏根节点没有纵向滚动；
- [ ] 320×568 能完成核心操作；
- [ ] 手机横屏布局可用；
- [ ] Parti 右上角宿主区域不遮挡关键操作；
- [ ] 主要触控目标至少 44×44px；
- [ ] 关键功能不依赖 hover；
- [ ] selected / locked / waiting / resolved / error 有反馈；
- [ ] snapshot 更新不会破坏正在进行的触控；
- [ ] reduced motion 下仍可正常使用；
- [ ] 中文不会造成首屏溢出；
- [ ] 桌面大屏不是简单把手机版无限拉宽；
- [ ] 正式版视觉不像后台或普通表单页面。
