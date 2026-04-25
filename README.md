# game_dcc_lab

`game_dcc_lab` 是一个面向游戏项目的 AI 辅助 Blender 资产与特效制作工作台。

本仓库用于维护源资产、DCC 自动化脚本、资产协议、校验工具、预览渲染和可导入游戏运行时的导出资产。它可以为 `train_oil` 这类游戏运行时仓库提供 `.glb`、贴图、特效资源和资产制作规范。

## 仓库定位

本仓库负责“生产资产”，游戏运行时仓库负责“消费资产”。

本仓库适合维护：

- Blender 源文件，例如 `.blend`。
- Blender Python 自动化脚本。
- AI 辅助资产制作工作流。
- 未来可能接入的 MCP / Skill / DCC 工具链。
- 资产命名规范和节点协议。
- GLB / glTF 校验与导出脚本。
- 预览渲染图。
- 可同步到游戏项目中的运行时资产。

本仓库不负责：

- 游戏玩法逻辑。
- Babylon.js 运行时系统。
- 游戏发布构建。
- `scene.json` 的最终运行时配置来源。

## 当前重点

当前第一条资产工作流是 `train_oil` 项目的透明火车油罐液体效果。

文档入口：

- [火车油罐容器液体效果 TA 设计与开发指南](docs/TRAIN_TANK_LIQUID_EFFECT_TA_GUIDE.md)

该文档覆盖：

- 火车油罐 GLB 资源关系。
- 容器液体效果的 TA 分层思路。
- 油面、油体、shader 裁切三种方案。
- Blender 资产中推荐添加的 marker / socket。
- Babylon.js 运行时实现方向。
- 透明排序风险。
- 配置与测试策略。

## 推荐目录结构

```text
game_dcc_lab/
  docs/
    TRAIN_TANK_LIQUID_EFFECT_TA_GUIDE.md
  source/
    blender/
      train_tank/
  exports/
    glb/
  scripts/
    blender/
    gltf/
    sync/
  manifests/
```

目录说明：

- `docs/`：资产协议、TA 说明、Blender 工作流和实现指南。
- `source/blender/`：Blender 源文件和工作场景。
- `exports/glb/`：导出的游戏运行时 GLB 资产。
- `scripts/blender/`：Blender Python 自动化脚本。
- `scripts/gltf/`：GLB / glTF 检查、校验和优化脚本。
- `scripts/sync/`：把确认后的导出资产同步到游戏运行时仓库。
- `manifests/`：资产元信息、必需节点和导出协议。

## 与游戏运行时仓库的关系

推荐分工如下：

```text
game_dcc_lab:
  源 .blend 文件
  Blender 自动化脚本
  GLB / glTF 校验
  资产协议
  导出资产

train_oil:
  游戏玩法代码
  scene.json 运行时配置
  Babylon.js 材质、shader 和系统
  可玩广告发布构建
```

对于 `train_oil`，最终运行时静态配置仍应维护在：

```text
train_oil/src/config/scene.json
```

本仓库可以保存资产制作说明、配置草稿或导出片段，但游戏运行时应保持一个权威配置来源，避免运行时同时依赖多个配置仓库。

## Git 提交规则

本仓库提交信息必须使用：

```text
<prefix>: <中文描述>
```

示例：

```text
docs: 添加火车油罐液体效果指南
asset: 添加火车油罐 Blender 源文件
vfx: 添加油罐液体效果资产结构
```

详细规则见：

- [Git 提交规则](docs/GIT_COMMIT_RULES.md)

本仓库提供版本化 hook：

```bash
git config core.hooksPath .githooks
git config commit.template .gitmessage
```

启用后，提交信息不符合“前缀 + 中文描述”规则时会被拒绝。

## 当前 Git 身份约定

本仓库使用个人 GitHub 账号：

```text
zgx197
```

当前电脑推荐使用的 SSH remote 是：

```text
git@github.com-zgx197:zgx197/game_dcc_lab.git
```

这个 `github.com-zgx197` host alias 用于区分个人账号和公司账号。公司账号仓库继续使用默认的 `github.com`，个人仓库使用 `github.com-zgx197`。

## 第一阶段建议

第一阶段先保持简单：

1. 把火车油罐相关 `.blend` 源文件放入 `source/blender/train_tank/`。
2. 在 Blender 中为油罐模型补充：
   - `liquid_socket`
   - `liquid_min_marker`
   - `liquid_max_marker`
3. 导出 GLB 到 `exports/glb/`。
4. 编写检查脚本，验证 GLB 中是否存在必需 marker。
5. 再通过同步脚本或人工确认，把最终 GLB 同步到 `train_oil/src/assets/`。

第一阶段目标不是建立复杂资产平台，而是先跑通：

```text
.blend 源文件 -> GLB 导出 -> 节点校验 -> 同步到游戏仓库
```
