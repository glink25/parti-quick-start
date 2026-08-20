# 发布规范

## 构建产物

标准输出：

```text
dist/
├─ parti.room.json
├─ index.html
├─ room.worker.js
├─ assets/...
└─ cover.*
```

CI 至少验证：

```bash
test -f dist/parti.room.json
test -f dist/index.html
test -f dist/room.worker.js
```

并确认 Worker 已 bundle，不包含项目内部相对模块 import。

## 版本一致性

保持：

```text
package.json.version
=
public/parti.room.json.version
=
Git tag 去掉 v 后的版本
```

例如 tag `v1.2.3` 对应版本 `1.2.3`。

## 本地发布前命令

```bash
npm ci
npm test
npm run build
npm run package
```

生成：

```text
parti.room.zip
```

ZIP 根目录应直接包含 Room Package 文件，而不是再套一层目录。

## GitHub Release

每个正式版本建议保留 GitHub Release：

```text
Release vX.Y.Z
├─ parti.room.zip
└─ parti.room.json
```

Release 提供可下载、可回归的版本归档，并作为市场安装之外的人工导入兜底。

## 不可变 package branch

为了避免长期使用同一个可变 ref 引发缓存问题，正式市场版本建议发布到不可变分支：

```text
parti-package-vX.Y.Z
```

例如：

```text
v1.2.3
parti-package-v1.2.3
```

一旦发布，不 force-push 修改该版本分支。修复通过发布新版本完成。

可以额外维护：

```text
parti-package
```

作为 latest alias，但正式市场登记优先指向精确版本 ref。

## 推荐 CI 流程

tag 推送后：

```text
checkout tag
-> 校验版本一致
-> npm ci
-> npm test
-> npm run build
-> 校验 dist
-> 生成 parti.room.zip
-> 创建 GitHub Release
-> 创建不可变 parti-package-vX.Y.Z 分支
-> 可选更新 parti-package latest alias
-> 登记 Parti Market 精确版本 ref
```

推荐触发：

```yaml
on:
  push:
    tags: ['v*']
```

## 市场登记

市场登记应使用精确版本 package branch：

```text
[parti-room] owner/repo@parti-package-vX.Y.Z
```

不要让正式市场入口只依赖长期滚动的 `parti-package`。

市场流程与格式如果发生变化，以 `glink25/Parti` 当前官方文档为准。

## 发布检查

发布前：

- [ ] `npm test` 通过；
- [ ] `npm run build` 通过；
- [ ] `npm run package` 通过；
- [ ] package / manifest / tag 版本一致；
- [ ] `dist/parti.room.json` 存在；
- [ ] `dist/index.html` 存在；
- [ ] `dist/room.worker.js` 存在；
- [ ] Worker 已 bundle；
- [ ] ZIP 可导入；
- [ ] 最小 / 最大玩家配置均验证；
- [ ] 手机尺寸人工验收；
- [ ] 断线 / 重连人工验收；
- [ ] 终局和重新开始流程（若支持）验证。

发布后：

- [ ] GitHub Release 存在；
- [ ] Release 包含 `parti.room.zip`；
- [ ] 不可变 package branch 存在；
- [ ] package branch 根目录是完整 Room Package；
- [ ] 市场登记指向精确版本 ref；
- [ ] 实际安装并运行验证。
