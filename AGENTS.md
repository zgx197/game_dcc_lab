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

## Blender 5.1 / bpy 实操注意事项

### 不要依赖节点显示名称

Blender 5.1 在中文界面中会本地化节点显示名称。例如 `Principled BSDF` 节点在界面和 `node.name` 中可能显示为：

```text
原理化 BSDF
```

因此脚本中不要用下面这种方式查找节点：

```python
bsdf = mat.node_tree.nodes.get("Principled BSDF")
```

这在中文界面下可能返回 `None`，导致脚本只修改了 `mat.diffuse_color`，没有真正修改 PBR 节点参数。

推荐按 `bl_idname` 查找节点：

```python
def find_principled_bsdf(material):
    if not material or not material.use_nodes:
        return None

    for node in material.node_tree.nodes:
        if node.bl_idname == "ShaderNodeBsdfPrincipled":
            return node

    return None
```

设置材质参数时，应修改节点输入，而不只修改 viewport diffuse color：

```python
bsdf = find_principled_bsdf(mat)
if bsdf:
    bsdf.inputs["Base Color"].default_value = (0.0, 0.001, 0.0008, 1.0)
    bsdf.inputs["Roughness"].default_value = 0.08
    bsdf.inputs["Alpha"].default_value = 1.0
```

如果需要兼容不同 Blender 版本，也应先检查输入是否存在：

```python
if "Coat Weight" in bsdf.inputs:
    bsdf.inputs["Coat Weight"].default_value = 0.6
```

### 同时检查材质节点和视口颜色

Blender 中至少有两类颜色会影响预览判断：

```text
mat.diffuse_color:
  主要影响 viewport/material preview 中的材质显示和对象色显示。

Principled BSDF Base Color / Alpha / Roughness:
  影响真实材质节点和后续导出/渲染效果。
```

制作或调试资产时，不能只看 `mat.diffuse_color`。如果视觉没有变化，应打印节点类型、输入项和值：

```python
for node in mat.node_tree.nodes:
    print(node.name, node.bl_idname, [input.name for input in node.inputs])
```

### 透明材质和视口截图不等于最终效果

Blender MCP 的 `get_viewport_screenshot` 截的是当前 Blender 视口状态。它适合快速沟通位置、层级和大致视觉，但不要把截图当成最终游戏渲染依据。

常见误差：

1. 视口可能仍处于 Solid/Object Mode 显示，材质节点效果不明显。
2. 半透明外壳会影响内部液面的可见性。
3. 透明排序可能让液面看起来偏灰、偏白，或高光被盖住。
4. 选中对象的橙色描边会影响截图判断。

调试透明容器内部液体时，建议：

```text
1. 先确认对象层级、位置、尺寸和材质槽正确。
2. 再切换 Material Preview 或 Rendered 观察。
3. 必要时临时降低外壳 alpha，或让液面接近不透明。
4. 高光可以先用独立曲线或细 mesh 做 authoring preview，正式运行时再换成 shader。
```

### 避免 HASHED / DITHERED 造成马赛克预览

Blender 5.1 的材质如果使用 `blend_method = "HASHED"` 或 `surface_render_method = "DITHERED"`，在透明容器、半透明外壳和液面叠加时很容易出现明显的马赛克/颗粒噪点。

这类噪点常见于：

```text
1. 透明油罐外壳材质。
2. 液面材质。
3. 边缘附着圈 `liquid_meniscus_rim`。
4. 深度阴影预览层 `liquid_depth_shadow_preview`。
```

如果用户反馈“液面像马赛克”“油面很脏”“透明层一堆噪点”，先打印可见对象的材质透明设置：

```python
for obj_name in ["SM_M_火车油罐", "liquid_surface", "liquid_meniscus_rim", "liquid_depth_shadow_preview"]:
    obj = bpy.data.objects.get(obj_name)
    if not obj:
        continue
    for slot in obj.material_slots:
        mat = slot.material
        if mat:
            print(obj.name, mat.name, mat.blend_method, getattr(mat, "surface_render_method", None), mat.diffuse_color)
```

调试预览阶段可以把 `HASHED / DITHERED` 改成更稳定的 `BLEND / BLENDED`：

```python
if getattr(mat, "blend_method", None) == "HASHED":
    mat.blend_method = "BLEND"

if getattr(mat, "surface_render_method", None) == "DITHERED":
    mat.surface_render_method = "BLENDED"

if hasattr(mat, "show_transparent_back"):
    mat.show_transparent_back = False
```

如果 `liquid_depth_shadow_preview` 造成液面变脏，应先隐藏它建立干净基线：

```python
shadow = bpy.data.objects.get("liquid_depth_shadow_preview")
if shadow:
    shadow.hide_viewport = True
```

白色圆弧、高光线、glint 等 authoring preview 对象只适合早期说明“这里应该有高光”。一旦它们干扰油面判断，应删除或隐藏，正式效果应由 shader 或更克制的材质参数表达。

### 使用多角度截图理解场景关系

使用 Blender MCP 做视觉确认时，不要只截当前用户视角。单一视角很容易误判对象之间的空间关系，尤其是透明容器、内部液面、空物体 root、marker 和高光预览对象同时存在时。

推荐至少截三类视角：

```text
Top / 正交顶视图：
  检查液面是否在容器内部、是否贴边、是否偏离中心。

左前或右前 45 度透视：
  检查液面高度、透明外壳遮挡、高光是否可见。

左前或右前 25 度透视：
  当 45 度被端面、外壳或选择描边挡住时，用更接近游戏斜俯视的角度检查内部液面。

侧视或端视：
  检查液面是否穿出容器、是否与顶部透明壳发生深度冲突。
```

对透明容器尤其建议使用：

```text
Top:
  判断液面形状。

Left 45 / Right 45:
  判断常规游戏视角下是否可见。

Left 25 / Right 25:
  判断液面在更接近游戏常见视角下是否清晰可见。
```

切换视角后应重新 `view_selected`，并尽量只选中需要构图的对象，例如容器 mesh、`liquid_surface` 和必要的高光对象。不要让巨大的 root empty 框占满截图，否则会影响判断。

示例 bpy 片段：

```python
import bpy

bpy.ops.object.select_all(action="DESELECT")
for name in ["SM_M_火车油罐", "liquid_surface", "liquid_meniscus_rim"]:
    obj = bpy.data.objects.get(name)
    if obj:
        obj.select_set(True)

bpy.context.view_layer.objects.active = bpy.data.objects.get("liquid_surface")

for window in bpy.context.window_manager.windows:
    screen = window.screen
    for area in screen.areas:
        if area.type != "VIEW_3D":
            continue

        region = next((r for r in area.regions if r.type == "WINDOW"), None)
        space = next((s for s in area.spaces if s.type == "VIEW_3D"), None)
        if not region or not space:
            continue

        with bpy.context.temp_override(window=window, screen=screen, area=area, region=region, space_data=space):
            bpy.ops.view3d.view_axis(type="TOP", align_active=False)
            bpy.ops.view3d.view_selected(use_all_regions=False)
            space.region_3d.view_perspective = "ORTHO"
            space.shading.type = "MATERIAL"
```

如果场景里有 `frame_change` handler、driver、动画或预览控制器，修改自定义属性后可能不会立即刷新画面。需要播放时间轴、拖动帧，或者在脚本里调用：

```python
bpy.context.scene.frame_set(bpy.context.scene.frame_current + 1)
bpy.context.scene.frame_set(bpy.context.scene.frame_current - 1)
bpy.context.view_layer.update()
```

调试动态效果时，先用夸张参数确认机制有效，再切回正式参数。比如石油液面可以先把波纹和晃动放大，确认截图能看到变化后，再恢复到慢、重、粘稠的状态。

### 按播放周期切分截图

分析动态效果时，不要只截某一帧。单帧只能说明某个瞬间看起来如何，不能说明播放过程中是否连续、是否跳变、是否有不合理的极值。

推荐先确定完整播放周期：

```text
frame_start:
  当前时间轴起始帧。

frame_end:
  当前时间轴结束帧。

cycle_length:
  frame_end - frame_start + 1。
```

然后根据周期长度按固定间隔截图。常用切分方式：

```text
4 等分:
  适合快速检查整体动态方向。

6 等分:
  适合检查液面晃动、高光漂移、循环衔接。

8 等分:
  适合检查复杂动画或容易出现跳变的 shader / driver / frame handler。
```

例如当前播放周期为 `1..144` 帧，可以先用 6 等分：

```text
1, 24, 48, 72, 96, 120, 144
```

分析每一帧时，应同时记录：

```text
当前帧号
当前视角
关键对象是否可见
液面高度是否合理
高光是否跳变
波纹是否过强
循环首尾是否接近
```

如果动态效果由自定义属性驱动，截图前应明确记录关键参数，例如：

```text
liquid_wave_amplitude
liquid_wave_speed
liquid_slosh_strength
liquid_slosh_speed
liquid_viscosity
liquid_highlight_drift
```

示例 bpy 片段：按时间轴 6 等分跳帧，并在每一帧刷新视图后截图。

```python
import bpy

scene = bpy.context.scene
frame_start = scene.frame_start
frame_end = scene.frame_end
steps = 6

frames = [
    round(frame_start + (frame_end - frame_start) * i / steps)
    for i in range(steps + 1)
]

for frame in frames:
    scene.frame_set(frame)
    bpy.context.view_layer.update()
    print("capture frame", frame)
    # 这里再调用 Blender MCP get_viewport_screenshot。
```

动态截图建议和多角度截图组合使用：

```text
Top + 周期切分:
  检查液面形状、波纹强度和高光漂移。

Left 25 / Right 25 + 周期切分:
  检查接近游戏视角时液体是否始终可见。

Left 45 / Right 45 + 周期切分:
  检查透明外壳遮挡和高度关系。
```

推荐把截图组织成“视角组”，而不是把所有截图混在一起：

```text
Top 组:
  top_f001
  top_f024
  top_f048
  top_f072
  top_f096
  top_f120
  top_f144

Left 25 组:
  left25_f001
  left25_f024
  left25_f048
  left25_f072
  left25_f096
  left25_f120
  left25_f144

Right 25 组:
  right25_f001
  right25_f024
  right25_f048
  right25_f072
  right25_f096
  right25_f120
  right25_f144
```

分析顺序建议：

```text
1. 先看 Top 组，确认形状、波纹和高光漂移是否合理。
2. 再看 Left 25 / Right 25 组，确认接近游戏视角时是否一直可见。
3. 最后只在需要时补 Left 45 / Right 45 或侧视组，检查遮挡和高度问题。
```

不要一开始就对所有角度做高密度切片。推荐先做：

```text
Top:
  6 等分

Left 25:
  6 等分

Right 25:
  6 等分
```

如果这三组已经能说明问题，就不需要继续扩大截图矩阵。只有当发现遮挡、穿帮、首尾跳变或动态极值时，再补充 45 度、侧视、端视或更密的 8 等分截图。

### 图片较多时先生成 AI Review Packet

不要把大量原始 PNG 直接交给 AI 分析。原始图过多时容易导致分析慢、上下文过大、视觉工具报错，或者让模型只关注个别图片而漏掉整体周期。

推荐先把原始截图降维成：

```text
1. contact sheet:
   把同一组视角或同一组帧切片拼成 1 张总结图。

2. numeric samples:
   用 JSON 记录关键数值，例如 z_min / z_max / z_range / highlight location。

3. AI_REVIEW_PACKET.md:
   用文本说明应该看哪几张总结图、当前参数是什么、不要默认读取哪些原始 PNG。
```

推荐目录结构：

```text
/tmp/game_dcc_lab/liquid_preview/
  liquid_preview_top_f001.png
  liquid_preview_top_f025.png
  ...
  liquid_preview_numeric_samples.json
  liquid_preview_manifest.json
  review_sheets_v3/
    01_top_cycle_2col.png
    02_left_right_25_2col.png
    AI_REVIEW_PACKET.md
    liquid_review_ai_packet.json
```

给 AI 分析时，默认只提供：

```text
review_sheets_v3/01_top_cycle_2col.png
review_sheets_v3/02_left_right_25_2col.png
review_sheets_v3/AI_REVIEW_PACKET.md
```

不要默认提供全部原始 PNG。只有当 AI 明确指出某一帧或某一角度需要放大检查时，再单独打开对应原图，例如：

```text
liquid_preview_top_f120.png
liquid_preview_right25_f072.png
```

contact sheet 排版建议：

```text
1. 优先使用 1-2 列，不要使用过宽的 3-4 列宽表。
2. 每个小图必须带帧号和视角标签。
3. Top 周期切片可以用 2 列多行。
4. Left / Right 对比可以用 2 列，让同一帧左右视角成对出现。
5. 保留原始 PNG 作为 debug source，但不要作为第一轮 AI 输入。
```

如果用 Blender 脚本把 PNG 拼成 contact sheet，需要注意：手动创建 mesh 平面时必须写入 UV。否则 Image Texture 会采样不到图片，渲染结果可能是一组黑色方块，但原始截图本身是正常的。

示例：

```python
mesh = bpy.data.meshes.new("sheet_quad_mesh")
verts = [(-1, -1, 0), (1, -1, 0), (1, 1, 0), (-1, 1, 0)]
mesh.from_pydata(verts, [], [(0, 1, 2, 3)])
mesh.update()

uv_layer = mesh.uv_layers.new(name="UVMap")
for loop, uv in zip(mesh.loops, [(0, 0), (1, 0), (1, 1), (0, 1)]):
    uv_layer.data[loop.index].uv = uv
```

如果周期首尾不连续，应检查 frame handler、driver 或 shader time 是否按可循环周期设计。对于 authoring preview，可以先接受非循环；对于正式游戏循环动画，应避免首尾明显跳变。

### 动态液面不要使用中心扇形拓扑

制作会被 frame handler、driver、shader 或顶点动画驱动的液面时，不要把液面做成“中心点 + 外圈边界”的 triangle fan。

这种拓扑在静态时看起来很省面，但动态预览中容易出现：

```text
1. 中心点向外辐射的亮暗分割。
2. X 型或放射状折痕。
3. 透明材质下被放大的假阴影。
4. `shade_smooth` 后看起来像液面被拉成几块三角区域。
```

这类问题不是石油材质本身导致的，而是拓扑和动态位移叠加造成的视觉伪影。调试时应先检查：

```python
from collections import Counter

obj = bpy.data.objects.get("liquid_surface")
print(Counter(len(poly.vertices) for poly in obj.data.polygons))
```

如果输出里存在大量三角面，尤其是所有三角面都共享中心点，就应该改为行列网格或裁切后的圆角矩形网格。

推荐做法：

```text
1. 用规则 row-column grid 生成液面主体。
2. 用圆角矩形、近矩形胶囊或容器内壁投影裁切网格边界。
3. 尽量让动态位移作用在均匀分布的顶点上。
4. 边缘附着圈 `liquid_meniscus_rim` 单独做，不要依赖中心扇形面的边缘阴影。
5. 预览截图时记录 `surface_topology`，例如 `row_column_grid_no_center_fan`。
```

当前油罐石油原型已验证的优化方向：

```text
liquid_surface_shape: rounded_rect_grid
liquid_surface_topology: row_column_grid_no_center_fan
liquid_surface_grid_x_segments: 18
liquid_surface_grid_y_segments: 52
surface_faces: 936
face_vertex_counts: [4]
```

如果用户看到“液面上有 X 型纹理/折痕”，优先怀疑中心扇形三角拓扑，而不是先调材质颜色、alpha 或透明排序。

### 容器液体优先使用 spring surface 预览

如果用户反馈“液体像一整块平面在晃动”，不要继续只调 pitch / roll 参数。刚性平面倾斜只能证明液面在动，但不像容器里的液体。

推荐把液面网格作为 spring surface 处理：

```text
每个顶点有自己的 displacement。
每个顶点有自己的 velocity。
容器运动或预览参数给出目标高度 target。
顶点通过 spring stiffness 向 target 回弹。
顶点通过 viscosity / damping 表达粘滞感。
边缘使用 edge_lift 做轻微爬升或堆积。
```

这样可以得到：

```text
1. 前后区域不同步。
2. 边缘靠容器壁的位置有轻微滞后和爬升。
3. 中部波动更柔，不再像硬板。
4. 石油可以通过高 viscosity 表达慢、重、粘。
```

当前油罐 spring surface 预览建议只暴露少量 root 自定义属性：

```text
liquid_spring_enabled
liquid_motion_preview_scale
liquid_inertia_strength
liquid_spring_stiffness
liquid_viscosity
liquid_edge_lift
liquid_surface_noise
liquid_preview_preset
```

推荐强预览参数，用于确认机制有效：

```text
liquid_motion_preview_scale: 1.8
liquid_inertia_strength: 0.055
liquid_spring_stiffness: 9.0
liquid_viscosity: 5.5
liquid_edge_lift: 0.03
liquid_surface_noise: 0.003
```

如果播放时幅度太大，优先降低：

```text
liquid_motion_preview_scale
liquid_inertia_strength
liquid_edge_lift
```

不要用白色圆弧线表达高光。spring surface 成立后，高光应后续由材质、shader 或非常克制的贴图/法线表达。

如果 spring surface 仍然像一张软板，继续升级为 2D damped wave surface。关键差异是：顶点不再只追自己的 target，而是受到邻居平均高度影响，让波从边缘向内部传播。

推荐模型：

```text
height[i]:
  当前顶点局部高度。

velocity[i]:
  当前顶点垂直速度。

neighbor_average - height[i]:
  邻居传播项，让起伏横向扩散。

external edge drive:
  只在边缘/端部注入惯性能量，避免整面一起抬升。

viscosity:
  阻尼，石油应偏高。
```

当前 2D wave root 控制项：

```text
liquid_wave_surface_enabled
liquid_motion_preview_scale
liquid_inertia_strength
liquid_wave_propagation
liquid_viscosity
liquid_edge_lift
liquid_surface_detail
liquid_preview_preset
```

推荐干净预览参数：

```text
liquid_motion_preview_scale: 1.5
liquid_inertia_strength: 0.045
liquid_wave_propagation: 18.0
liquid_viscosity: 7.5
liquid_edge_lift: 0.020
liquid_surface_detail: 0.0012
```

如果用户明确说“液面没有上下浮动”，要区分两类运动：

```text
local wave / z_range:
  局部起伏，表现为边缘波、中部滞后和表面不平。

level bob / z_avg:
  整体液位上下浮动，表现为整层油面平均高度变化。
```

2D wave surface 可以有 `z_range`，但如果 `z_avg` 基本不变，用户从斜俯视角仍可能感觉没有上下浮动。调试时可以增加一个明确的 authoring 参数：

```text
liquid_level_bob_amplitude
liquid_level_bob_frequency
```

推荐强可见预览值：

```text
liquid_level_bob_amplitude: 0.030
liquid_level_bob_frequency: 0.42
liquid_motion_preview_scale: 2.2
```

注意：整体液位 bob 是 authoring preview，用来确认动态可读性；正式游戏里应根据油量、车辆运动和相机距离收敛到更克制的幅度。

对于火车油罐这种顶部大面积可见容器，要谨慎使用整体 `level bob`。当前资产已验证：

```text
liquid_surface / inner cavity X ratio: 0.8772
liquid_surface / inner cavity Y ratio: 0.9562
```

液面几乎铺满整个可见内腔，任何全局 `z_avg` 上下变化都会被用户读成“一整页在上下动”，而不是液体起伏。

更适合的做法：

```text
1. 保持 z_avg 基本恒定。
2. 每步积分后移除 height 的平均值，避免全局漂移。
3. 在容器端部和边缘注入相反方向的能量。
4. 通过邻居传播让波向中部扩散。
5. 只使用局部 z_range 表达液体起伏。
```

当前 travelling edge wave 预览参数：

```text
liquid_motion_preview_scale: 1.8
liquid_inertia_strength: 0.085
liquid_wave_propagation: 42.0
liquid_viscosity: 6.2
liquid_edge_lift: 0.045
liquid_surface_detail: 0.0012
```

验证目标：

```text
z_avg:
  基本恒定。

z_range:
  有 0.008m .. 0.034m 左右的局部变化。
```

如果用户仍然看不清局部起伏，可以临时创建 `debug_liquid_wave_profile_centerline` 曲线，沿液面中心线采样顶点高度并上移少量显示。它只用于确认局部波形，不属于正式效果：

```text
debug_liquid_wave_profile_centerline:
  临时中心线波形。
  只用于 authoring preview。
  不导出，不保存进正式资产。
```

当用户要求“看到局部液面起伏而不是整块升降”时，可以使用更直观的 local travelling wave 预览：

```text
liquid_local_wave_enabled
liquid_motion_preview_scale
liquid_local_wave_amplitude
liquid_local_wave_speed
liquid_local_wave_frequency
liquid_edge_wave_strength
liquid_cross_wave_strength
liquid_viscosity
```

关键约束：

```text
每帧移除平均高度。
保持 z_avg 基本恒定。
用 z_range 和 debug centerline 判断局部波。
不要重新启用全局 level bob。
```

如果用户反馈“油面黑灰相间”“像一排灰色横条”“液体不细腻”，不要只增加网格精度或继续放大 `liquid_local_wave_amplitude`。当前油罐液面面积较小，低频、大幅、方向单一的 travelling wave 会被材质预览直接读成黑灰块。

推荐拆分职责：

```text
几何位移:
  只做低幅、多方向、去均值的局部干涉波。
  用来证明液面局部起伏，而不是制造强烈条纹。

材质 / shader:
  表达石油的细腻高光、微小扰动和暖黑棕颜色。
  细腻感优先通过 normal / bump / 运行时 shader 处理。
```

更适合当前油罐的调试参数范围：

```text
liquid_local_wave_amplitude: 0.006 .. 0.014
liquid_local_wave_frequency: 3.0 .. 5.0
liquid_edge_wave_strength: 0.002 .. 0.006
liquid_cross_wave_strength: 0.001 .. 0.004
liquid_viscosity: 9.0 .. 12.0
z_range: 0.008m .. 0.020m
```

如果需要更明显的动态，优先改成多方向干涉波，而不是单一方向横向波带：

```text
primary length wave:
  低幅主波，沿容器长度传播。

diagonal secondary waves:
  两到三个不同方向和速度的次级波，用于打破平行条纹。

mean removal:
  每帧减掉全部顶点高度平均值，避免整块液面上下浮动。
```

当前已验证的较稳定预览名称：

```text
liquid_preview_preset: irregular_micro_interference_oil_surface
liquid_wave_model: multi_direction_local_interference_no_global_bob
```

材质上应避免让大面积灰色高光覆盖油面。Blender 材质预览阶段可以先使用暖黑棕不透明基线：

```text
Base Color:
  约 (0.04 .. 0.06, 0.02 .. 0.04, 0.006 .. 0.012, 1.0)

Roughness:
  约 0.40 .. 0.55

Specular IOR Level:
  约 0.20 .. 0.35

Coat Weight:
  约 0.05 .. 0.15
```

正式游戏效果再用环境反射、normal map 或 shader 参数增加石油高光，不要用白色圆弧或强灰色条纹代替。

如果用户反馈“高光太亮”“波形太重复”“看不出油罐运动状态下液体在晃”，应切到专门的 motion debug 模式，而不是继续增强正式材质高光。

推荐 debug 设计：

```text
正式液面:
  保持深色、低高光，用于判断最终石油颜色。

运动输入:
  用模拟火车加速 / 松油 / 刹车的 debug acceleration 驱动液面。
  液面长向出现高低端差异，表达液体滞后。

高度可视化:
  用临时 debug 曲线显示液面高度，而不是把高光做得很亮。
```

当前已验证的 debug 属性：

```text
liquid_debug_motion_enabled: true
liquid_debug_motion_mode: train_accel_brake_slosh
liquid_debug_motion_strength: 1.0
liquid_debug_motion_period: 5.6
liquid_debug_height_overlay_enabled: true
liquid_debug_height_overlay_mode: contour_lines_and_high_low_end_bars
liquid_debug_arrow_visible: true
liquid_preview_preset: debug_train_motion_slosh_height_overlay
liquid_wave_model: debug_accel_brake_slosh_with_irregular_residuals
```

当前 debug 对象命名：

```text
debug_liquid_motion_vector_arrow:
  橙色箭头，表示当前车辆运动输入方向。

debug_liquid_high_side_level_hint:
  高位端提示线。

debug_liquid_height_contour_length_*:
  沿油罐长度方向的液面高度轮廓线。

debug_liquid_height_contour_width_*:
  沿油罐宽度方向的液面高度轮廓线。

debug_liquid_low_end_bar / debug_liquid_high_end_bar:
  蓝色低端和橙色高端提示。
```

这些对象只用于 authoring debug，不属于正式资产：

```text
不导出。
不保存进正式 .blend。
正式交付前隐藏或删除。
```

性能注意：高度轮廓线不要每帧做全量最近点搜索。创建 debug overlay 时应预计算每个曲线采样点对应的 `liquid_surface` 顶点索引，frame handler 每帧只读取这些顶点高度并更新曲线点，否则 Blender 视口播放帧率会明显下降。

### 液体场景 Debug 控制器

当用户需要模拟“车厢前进、加油、拐弯”等业务状态时，不要继续调单一波纹参数。应使用统一场景参数：

```text
liquid_debug_enabled
liquid_debug_scenario
liquid_debug_auto_cycle
liquid_debug_playback_strength
liquid_debug_viscosity
liquid_debug_show_overlay
liquid_debug_show_motion_arrow
liquid_debug_show_height_contours
liquid_debug_show_high_low_bars
liquid_debug_show_inlet
```

当前已实现的场景：

```text
train_forward:
  模拟车厢前进、加速和刹车。
  local +Y 视作车头方向。
  加速时 local -Y 端液体堆高。
  刹车时 local +Y 端液体堆高。
  非加油状态应保持 z_avg 稳定。

refueling:
  模拟加油过程。
  平均液位 z_avg 会随时间上升。
  注入口附近有局部扰动和向外扩散的慢涟漪。
  会显示 debug_liquid_scenario_inlet_circle。

train_turn:
  模拟车厢拐弯。
  liquid_debug_turn_direction = 1 表示右转，液体向 local -X 侧堆积。
  liquid_debug_turn_direction = -1 表示左转，液体向 local +X 侧堆积。
  非加油状态应保持 z_avg 稳定。
```

场景专用参数：

```text
train_forward:
  liquid_debug_train_speed
  liquid_debug_train_accel
  liquid_debug_slosh_strength
  liquid_debug_slosh_damping
  liquid_debug_slosh_period

refueling:
  liquid_debug_fill_rate
  liquid_debug_fill_target
  liquid_debug_fill_cycle_seconds
  liquid_debug_inlet_strength
  liquid_debug_inlet_radius
  liquid_debug_inlet_offset_x
  liquid_debug_inlet_offset_y
  liquid_debug_refuel_ripple_strength
  liquid_debug_refuel_level_range

train_turn:
  liquid_debug_turn_direction
  liquid_debug_turn_strength
  liquid_debug_turn_period
  liquid_debug_lateral_slosh_strength
  liquid_debug_lateral_damping
```

当前场景控制器 handler：

```text
game_dcc_lab_liquid_scenario_debug_surface_handler:
  根据 liquid_debug_scenario 更新 liquid_surface 顶点高度。

game_dcc_lab_liquid_scenario_debug_overlay_handler:
  更新 debug overlay，包括运动箭头、高低端、注入口和高度轮廓线。
```

当前 overlay 对象：

```text
debug_liquid_scenario_motion_arrow
debug_liquid_scenario_high_bar
debug_liquid_scenario_low_bar
debug_liquid_scenario_inlet_circle
debug_liquid_scenario_height_contour_length_*
debug_liquid_scenario_height_contour_width_*
```

这些对象只用于 authoring debug：

```text
不导出。
不保存进正式 .blend。
正式截图或交付前隐藏或删除。
```

验证时推荐采样多帧：

```text
train_forward / train_turn:
  z_avg 应基本稳定。
  z_range 随运动状态变化。

refueling:
  z_avg 应随播放时间上升。
  注入口附近应有局部扰动。
```

### Babylon 标准资产状态

当决定在 Babylon 中实现最终石油 shader 后，Blender 文件应收敛为标准资产状态，不再保留 Blender-only debug 和 preview 效果。

标准场景对象应尽量保持最小：

```text
Camera
Light
container_train_tank_root
  SM_M_火车油罐
  liquid_surface
  liquid_meniscus_rim
  liquid_socket
  liquid_min_marker
  liquid_max_marker
```

必须删除或不导出的对象：

```text
debug_*
liquid_flow_sheen_preview_*
liquid_broad_reflection_preview_*
liquid_depth_shadow_preview
SM_M_火车油罐_without_transparent_shell_preview
contact sheet / review packet 对象:
  card_*
  img_*
  label_*
  sheet_*
  background / bg / cam
```

必须移除的 Blender handler：

```text
game_dcc_lab_liquid_*_handler
所有包含 liquid 或 game_dcc_lab 的 frame_change handler
```

`liquid_surface` 标准状态：

```text
平整液面，不保留动态 z 位移。
topology: row_column_grid_no_center_fan
shape: rounded_rect_grid
vertices: 1007
faces: 936
face vertex counts: 4 only
material: MAT_liquid_runtime_placeholder_dark_oil
```

`container_train_tank_root` 标准 metadata：

```text
asset_role: liquid_container_root
liquid_schema_version: 1.0.0
liquid_runtime_target: babylon
liquid_runtime_implementation: Babylon shader/material; Blender data is mesh+markers+metadata only
liquid_container_type: train_tank
liquid_axis_forward: local +Y
liquid_axis_right: local +X
liquid_axis_up: local +Z
liquid_fill_axis: local +Z
liquid_babylon_material: LiquidOilMaterial
liquid_export_policy: export mesh markers metadata; exclude debug and Blender handlers
liquid_standard_state: true
```

`liquid_surface` 标准 metadata：

```text
liquid_protocol_role: liquid_surface
liquid_runtime_role: surface_mesh_for_babylon_shader
liquid_surface_shape: rounded_rect_grid
surface_topology: row_column_grid_no_center_fan
liquid_base_local_z
liquid_fill_ratio
export_include: true
runtime_hint: Babylon replaces placeholder material and drives height/slosh/flow at runtime
```

`liquid_meniscus_rim` 在当前标准状态中是可选 authoring reference：

```text
export_include: false
liquid_runtime_role: optional_authoring_reference_not_required_for_babylon
```

标准化后应验证：

```text
object_count:
  9

root_children:
  SM_M_火车油罐
  liquid_surface
  liquid_meniscus_rim
  liquid_socket
  liquid_min_marker
  liquid_max_marker

remaining liquid handlers:
  []

liquid_surface z_range:
  0.0
```

注意：恢复原始透明油罐外壳后，Blender Material Preview 里仍可能看到透明噪点或颗粒。这是视口透明显示问题，不代表标准资产中还有 debug 对象。最终透明排序和石油材质应在 Babylon 中验收。

### 透明油罐玻璃预览材质

油罐透明外壳如果在 Blender Material Preview 中出现雪花、黑白颗粒或大面积抖点，优先检查透明外壳材质，而不是液面 mesh。

当前油罐透明外壳材质：

```text
object: SM_M_火车油罐
material slot: 0
material: M_油罐_SM_M_火车油罐
```

常见雪花原因：

```text
Alpha 太低，例如 0.10 .. 0.20。
Screen refraction 在 Material Preview 中叠加深色液面产生噪点。
Transmission / IOR 预览和 Blender 视口透明排序不稳定。
透明背面参与显示，导致多层透明面叠加。
```

Blender 预览阶段推荐稳定玻璃参数：

```text
blend_method: BLEND
surface_render_method: BLENDED
show_transparent_back: false
use_screen_refraction: false
alpha_threshold: 0.0

Base Color:
  (0.44, 0.70, 0.98, 0.48)

Alpha:
  0.48

Roughness:
  0.028

Specular IOR Level:
  0.95

Coat Weight:
  0.72

Coat Roughness:
  0.025

Transmission Weight:
  0.0

IOR:
  1.45
```

这是 Blender 视口中的稳定玻璃预览，不是最终游戏玻璃。Alpha 低于约 `0.35` 时更透明，但 Material Preview 容易重新出现雪花；Alpha 高于约 `0.55` 时更稳定，但会偏乳白。当前 `0.48` 是“可看见内部 + 避免雪花”的折中。

最终 Babylon 玻璃应由运行时材质实现：

```text
TankGlassMaterial
PBR / shader material
environment reflection
Fresnel
proper alpha sorting
optional refraction
```

### 深色石油流动不要只靠 base color

黑色石油如果只使用很暗的 Base Color，在 Blender 材质预览里会接近纯黑，几何起伏和真实流动都很难看清。不要通过把 Base Color 调黄、调灰或调亮来解决，否则会变成泥水或灰色塑料。

推荐分层：

```text
Base Color:
  保持接近黑褐色石油。

Geometry slosh:
  低频、低幅，用于表达油罐加速 / 刹车时的高低端。

Shader flow:
  两层或多层移动 normal / bump，速度、方向、尺度不同。

Roughness variation:
  用移动 noise 轻微改变粗糙度，让反射锐度漂移。

Sheen preview:
  在 Blender 视口里可临时加低亮度油膜反射线，帮助看黑油流动。
```

当前已验证的 shader flow 属性：

```text
liquid_shader_flow_enabled: true
liquid_shader_flow_mode: dual_moving_noise_bump_roughness
liquid_shader_flow_strength: 1.0
liquid_shader_flow_speed_a: 0.018
liquid_shader_flow_speed_b: -0.011
liquid_shader_flow_roughness_drift: 0.006
```

材质节点建议：

```text
large slow noise:
  低频、大尺度、慢速移动，负责粘稠油膜的大形。

fine diagonal noise:
  高频、小尺度、斜向移动，负责打破重复感。

roughness drift noise:
  低强度移动粗糙度变化，负责反射细节。
```

如果 Blender 材质预览仍然太黑，可以添加临时 authoring preview：

```text
liquid_flow_sheen_preview_*:
  低亮度、低 alpha、非重复的短曲线。
  用于模拟最终 shader flow mask 的可见反射。
  不导出，不保存进正式资产。
```

注意不要把 sheen preview 做成白色圆弧或规则横线。它应该是暗蓝灰 / 棕灰、短而不连续、方向有变化、随播放缓慢漂移的油膜反射痕迹。性能上不要每帧对 sheen 曲线点做全量最近点搜索；如果只是材质预览层，可以固定抬高到液面上方少量距离。

马赛克感要和动态问题分开排查。诊断顺序：

```text
1. 显示透明外壳截图。
2. 降低透明外壳 alpha 再截图。
3. 隐藏火车外壳，只看 liquid_surface。
```

如果 `liquid_only` 干净，而显示外壳后出现颗粒，说明主要是 Blender 视口透明叠加问题，不是液体模拟精度不足。当前油罐已验证：隐藏外壳时液面本体较干净，外壳 alpha 从 0.14 降到 0.04 后马赛克明显减轻。

### 使用临时线框轮廓调试形状

制作容器内液体、碰撞体、裁切边界、挂点范围等资产时，可以先创建临时线框或曲线对象辅助判断形状，而不是直接修改正式 mesh。

这种方法适合回答下面的问题：

```text
当前液面是椭圆、胶囊、圆角矩形还是自定义形状更合适？
液面有没有贴到容器内壁？
安全边距够不够？
当前 mesh 的 top-down 投影和容器截面是否一致？
```

推荐做法：

```text
1. 用不同颜色创建多个 debug outline。
2. 明确命名为 debug_*，并写入 debug_note 自定义属性。
3. 使用 emission 材质或 show_in_front，保证线框在透明材质中可见。
4. 用 Top / 45 度 / 侧视多角度截图对比。
5. 确认形状后，再把正式 mesh 替换为选定形状。
6. debug outline 只用于 authoring preview，不导出、不保存到正式资产。
```

颜色建议：

```text
橙色:
  当前正式形状或旧方案。

绿色:
  第一候选方案。

青色:
  新推荐方案或用户反馈后的修正方案。

黄色:
  安全边界、bbox 或容器内侧截面。
```

临时线框对象应显式标记：

```python
obj["debug_note"] = "temporary authoring analysis outline; do not export"
obj.show_in_front = True
```

示例：创建一个近矩形圆角轮廓用于对比液面形状。

```python
import bpy
import math

def make_debug_emission(name, color):
    mat = bpy.data.materials.get(name) or bpy.data.materials.new(name)
    mat.use_nodes = True
    mat.diffuse_color = color
    mat.blend_method = "BLEND"
    mat.node_tree.nodes.clear()

    emission = mat.node_tree.nodes.new("ShaderNodeEmission")
    emission.inputs["Color"].default_value = color
    emission.inputs["Strength"].default_value = 1.0

    output = mat.node_tree.nodes.new("ShaderNodeOutputMaterial")
    mat.node_tree.links.new(emission.outputs["Emission"], output.inputs["Surface"])
    return mat

def make_rounded_rect_outline(name, cx, cy, z, half_x, half_y, radius, material):
    coords = []
    segments = 12
    sx = half_x - radius
    sy = half_y - radius

    arc_defs = [
        (cx + sx, cy + sy, 0, math.pi / 2),
        (cx - sx, cy + sy, math.pi / 2, math.pi),
        (cx - sx, cy - sy, math.pi, math.pi * 1.5),
        (cx + sx, cy - sy, math.pi * 1.5, math.tau),
    ]

    for ax, ay, a0, a1 in arc_defs:
        for i in range(segments + 1):
            t = a0 + (a1 - a0) * i / segments
            coords.append((ax + math.cos(t) * radius, ay + math.sin(t) * radius, z))
    coords.append(coords[0])

    curve = bpy.data.curves.new(name, "CURVE")
    curve.dimensions = "3D"
    curve.bevel_depth = 0.008
    curve.bevel_resolution = 1

    spl = curve.splines.new("POLY")
    spl.points.add(len(coords) - 1)
    for point, (x, y, point_z) in zip(spl.points, coords):
        point.co = (x, y, point_z, 1.0)

    obj = bpy.data.objects.new(name, curve)
    bpy.context.scene.collection.objects.link(obj)
    obj.data.materials.append(material)
    obj.show_in_front = True
    obj["debug_note"] = "temporary authoring analysis outline; do not export"
    return obj
```

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
