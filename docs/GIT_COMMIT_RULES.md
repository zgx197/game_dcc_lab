# Git 提交规则

本仓库要求提交信息使用固定格式：

```text
<prefix>: <中文描述>
```

示例：

```text
docs: 添加火车油罐液体效果指南
asset: 添加火车油罐 Blender 源文件
vfx: 添加油罐液体效果资产结构
blender: 添加油罐液体 marker 检查脚本
```

## 允许的 Prefix

当前允许的前缀：

- `asset`：源资产或导出资产变更。
- `vfx`：特效相关资产、脚本或文档变更。
- `blender`：Blender 文件、Blender Python 或 DCC 工作流变更。
- `docs`：文档变更。
- `chore`：仓库维护类变更。
- `feat`：新增能力。
- `fix`：修复问题。
- `refactor`：重构。
- `test`：测试或校验脚本。
- `build`：构建或导出流程。
- `ci`：CI 配置。
- `config`：配置变更。
- `sync`：同步脚本或同步产物。
- `release`：发布相关变更。

## 中文描述要求

冒号后的描述必须包含中文，并且应说明本次变更的实际内容。

推荐：

```text
docs: 添加火车油罐液体效果指南
asset: 添加火车油罐 Blender 源文件
```

不推荐：

```text
docs: update
asset: add files
fix: bug
```

## 本地 Hook

本仓库提供版本化 hook：

```text
.githooks/commit-msg
```

启用方式：

```bash
git config core.hooksPath .githooks
git config commit.template .gitmessage
```

启用后，提交信息不符合规则时，`git commit` 会被拒绝。
