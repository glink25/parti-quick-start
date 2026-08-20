# 发布规范

## 目标

Parti Room 的发布流程应保持简单。对于快速开发的小游戏，发布前最重要的是：产物正确、能导入、能实际试玩。

不要求为了发布建立复杂 CI、完整自动化测试矩阵或视觉回归系统。

## 构建产物

标准输出：

```text
dist/
├─ parti.room.json
├─ index.html
├─ room.worker.js
└─ assets/...
```

最低构建检查：

```bash
npm run build

test -f dist/parti.room.json
test -f dist/index.html
test -f dist/room.worker.js

grep "@parti/worker-sdk" dist/room.worker.js
grep "defineRoom" dist/room.worker.js
```

重点是确认 Worker 没有被错误打包，并保留 Parti SDK 的运行时 import。

## 打包

如果项目提供 package 脚本：

```bash
npm run package
```

生成：

```text
parti.room.zip
```

ZIP 根目录应直接包含 Room Package 文件，不要再套一层目录。

## 版本

正式版本建议让下列版本保持一致：

```text
package.json.version
public/parti.room.json.version
Git tag
```

小规模迭代或开发预览不必为了版本流程增加额外负担。

## GitHub Release 与 package branch

需要公开发布或进入 Parti Market 时，可以继续使用：

```text
GitHub Release
parti-package
parti-package-vX.Y.Z
```

是否维护不可变版本分支、自动 Release 或 CI，由项目自身需求决定，不作为 quick-start 的强制要求。

市场格式和流程发生变化时，以 `glink25/Parti` 当前官方文档为准。

## 真人发布验证

发布前由开发者或玩家实际在 Parti 中验证一次：

- Room 可以正常导入 / 加载；
- 玩家可以加入；
- 核心玩法可以正常进行；
- UI 在实际设备上没有明显不可操作问题。

对于视觉效果、交互手感、布局合理性和玩法体验，应以真人试玩为主。

AI 不需要默认进行截图分析、逐分辨率视觉比对或自动操作完整游戏流程。这些方法成本高，也不能可靠替代玩家判断。

## 最小发布检查

- [ ] `npm run build` 成功；
- [ ] manifest、HTML、Worker 产物存在；
- [ ] Worker 保留 `@parti/worker-sdk` / `defineRoom` 引入；
- [ ] ZIP（若使用）可以导入；
- [ ] 开发者或玩家实际试玩过核心流程。

其他测试、CI 和验证按项目复杂度自行增加。
