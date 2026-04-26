# Codex 工作约定

本仓库使用 Blender 作为 DCC 执行环境，用于资产检查、预览渲染、GLB 导出，以及后续自动化资产设置。

## Blender 路径

当前机器上的 Blender 安装位置：

```text
/Applications/Blender.app
```

命令行调用 Blender 时使用这个可执行文件：

```text
/Applications/Blender.app/Contents/MacOS/Blender
```

不要假设 `blender` 已经在 `PATH` 里。优先使用上面的绝对路径，或者让脚本读取 `BLENDER_BIN` 环境变量，并使用这个默认值：

```text
${BLENDER_BIN:-/Applications/Blender.app/Contents/MacOS/Blender}
```

已验证可用的 Blender 版本：

```text
Blender 5.1.1
```

## Codex 沙箱注意事项

在 macOS 上，Blender 后台模式可能会在 Codex 默认沙箱内初始化 Metal GPU 后端时崩溃。如果 Blender 命令在沙箱内失败，不要直接判断 `.blend` 文件损坏；应使用同一个 Blender 命令重新申请非沙箱执行权限。

下面这个目标文件已经验证过，可以在非沙箱环境中正常打开：

```text
source/blender/train_tank/SM_火车油罐_liquid_setup.blend
```

## bpy 标准使用方式

不要用系统 Python 执行 Blender 自动化脚本，也不要在系统 Python 中直接 `import bpy`。

正确方式是让脚本在 Blender 进程内执行：

```bash
/Applications/Blender.app/Contents/MacOS/Blender \
  --background \
  --factory-startup \
  path/to/asset.blend \
  --python path/to/script.py
```

一次性检查命令可以使用：

```bash
/Applications/Blender.app/Contents/MacOS/Blender \
  --background \
  --factory-startup \
  path/to/asset.blend \
  --python-expr "import bpy; print([obj.name for obj in bpy.data.objects])"
```

Blender 自带 Python 运行时：

```text
/Applications/Blender.app/Contents/Resources/5.1/python/bin/python3.13
```

已观察到的 Blender Python 版本：

```text
Python 3.13.9
```

系统 Python 不是 Blender 脚本的权威运行环境。需要使用 Blender 进程执行脚本，确保 `bpy` 可用，并且和当前安装的 Blender 版本匹配。

## 资产编辑安全规则

`.blend` 文件是二进制文件，Git diff 很难审查。Codex 处理资产时应遵守：

1. 先检查，再修改。
2. 预览图和临时导出默认写到 `/tmp`，除非用户明确要求纳入版本管理。
3. 自动生成或修改 `.blend` 时，优先写入副本，不直接覆盖源文件。
4. 只有在用户明确同意后，才覆盖源 `.blend` 文件。

## Blender MCP 约定

后续可以使用 Blender MCP 做交互式 AI 辅助场景检查、实时修改和预览迭代。MCP 应作为交互式辅助通道，而不是唯一的生产流水线。

推荐分工：

```text
Blender Python 脚本：
  可重复的检查、预览、导出和校验任务

Blender MCP:
  实时场景检查、小步交互修改、视口截图、材质试验、用户引导的预览迭代
```

预期使用第三方 `ahujasid/blender-mcp` 项目。它由两部分组成：

```text
1. Blender addon：在 Blender 内部创建 socket server。
2. MCP server：通过 `uvx blender-mcp` 启动，负责连接 AI 客户端和 Blender。
```

本机 Blender MCP addon 文件放在：

```text
/Users/admin/work/UGIT/tools/blender-mcp/addon.py
```

在 Blender 中安装时，进入 `偏好设置 -> 插件 -> 从磁盘安装...`，选择上面的 `addon.py`。

### Blender 侧启动步骤

1. 打开目标 `.blend` 文件。
2. 在 Blender 中确认已启用 `Blender MCP` 插件。
3. 在 3D View 中按 `N` 打开右侧 Sidebar。
4. 进入 `BlenderMCP` 面板。
5. 保持端口为 `9876`。
6. 默认不要勾选外部资产或模型生成服务：
   - `Use assets from Poly Haven`
   - `Use Hyper3D Rodin 3D model generation`
   - `Use assets from Sketchfab`
   - `Use Tencent Hunyuan 3D model generation`
7. 点击 `Connect to MCP server`。
8. 面板显示 `Running on port 9876` 后，Blender 侧服务即已启动。

不要同时让多个 AI 客户端连接同一个 Blender MCP 会话，避免端口和状态冲突。

本机已验证存在：

```text
uv:  /Users/admin/.local/bin/uv
uvx: /Users/admin/.local/bin/uvx
```

### Codex 侧 MCP 配置

Codex 使用 STDIO 方式启动 MCP server。图形界面中添加自定义 MCP 时填写：

```text
Name:
blender

Type:
STDIO

Command to launch:
/Users/admin/.local/bin/uvx

Arguments:
blender-mcp

Environment variables:
留空

Environment variable passthrough:
留空

Working directory:
/Users/admin/work/UGIT/game_dcc_lab
```

等价命令：

```bash
codex mcp add blender -- /Users/admin/.local/bin/uvx blender-mcp
```

检查配置：

```bash
codex mcp list
codex mcp get blender
```

当前已验证的 Codex MCP 配置：

```text
name: blender
transport: stdio
command: /Users/admin/.local/bin/uvx
args: blender-mcp
cwd: /Users/admin/work/UGIT/game_dcc_lab
status: enabled
```

Blender MCP 默认连接参数：

```text
BLENDER_HOST=localhost
BLENDER_PORT=9876
```

同一个 Blender 会话只运行一个 Blender MCP server 实例，避免端口和会话状态冲突。

### Codex 使用 MCP 获取 Blender 信息

确认 Blender 侧显示 `Running on port 9876`，并且 Codex 已加载 `blender` MCP server 后，Codex 可以使用 Blender MCP 工具读取当前场景。

常用工具：

```text
get_scene_info:
  读取当前场景名称、对象数量、对象名称、对象类型、位置和材质数量。

get_object_info:
  读取指定对象的详细信息。

get_viewport_screenshot:
  获取当前 Blender 视口截图，用于确认视觉效果。

execute_blender_code:
  在 Blender 进程内执行 bpy 代码。优先用于只读检查；写入或保存 .blend 前必须先得到用户确认。
```

示例：读取当前场景概览。

```text
Codex 调用 Blender MCP 的 get_scene_info。
```

当前油罐文件已验证返回：

```text
Scene
object_count: 3
Light
Camera
SM_M_火车油罐
materials_count: 5
```

示例：读取 `SM_M_火车油罐` 的层级和材质槽。

```python
import bpy
import json
from collections import Counter

obj = bpy.data.objects.get("SM_M_火车油罐")
mesh = obj.data if obj and obj.type == "MESH" else None
counts = Counter(poly.material_index for poly in mesh.polygons) if mesh else Counter()

print(json.dumps({
    "name": obj.name if obj else None,
    "type": obj.type if obj else None,
    "parent": obj.parent.name if obj and obj.parent else None,
    "children": [child.name for child in obj.children] if obj else [],
    "dimensions": [round(v, 4) for v in obj.dimensions] if obj else [],
    "mesh": {
        "vertices": len(mesh.vertices) if mesh else 0,
        "edges": len(mesh.edges) if mesh else 0,
        "polygons": len(mesh.polygons) if mesh else 0,
        "uv_layers": [uv.name for uv in mesh.uv_layers] if mesh else [],
    },
    "material_slots": [
        {
            "slot": i,
            "material": slot.material.name if slot.material else None,
            "polygon_count": counts.get(i, 0),
        }
        for i, slot in enumerate(obj.material_slots)
    ] if obj else [],
}, ensure_ascii=False, indent=2))
```

当前 `SM_M_火车油罐` 已验证为单一 mesh 对象，没有子对象；不同部位通过材质槽区分。
