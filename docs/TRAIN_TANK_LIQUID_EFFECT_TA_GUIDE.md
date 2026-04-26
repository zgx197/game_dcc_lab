# 火车油罐容器液体效果 TA 设计与开发指南

## 1. 背景

当前火车油罐模型已经具备半透明外壳表现。半透明容器是制作“罐内液体”效果的良好前提，因为玩家可以透过外壳看到内部石油。

本项目中与火车油罐相关的资源主要包括：

- `src/assets/SM_火车油罐.glb`
- `src/assets/new_point火车油罐.glb`
- `src/assets/SM_火车油罐底.glb`
- `src/assets/SM_火车油罐身.glb`

当前代码中已经注册并使用的运行时油罐车厢资源是：

- `sm_huoche_youtong` -> `src/assets/new_point火车油罐.glb`
- `sm_huoche_youtong_di` -> `src/assets/SM_火车油罐底.glb`
- `sm_huoche_youtong_shen` -> `src/assets/SM_火车油罐身.glb`

运行时追加油罐车厢的位置在：

- `src/systems/SceneRailGameplaySystem.ts`
- `buildTankCarriage()`

目前运行时车厢由底座和罐体组合：

```ts
const carriageBaseInstance = modelPool.acquire('sm_huoche_youtong_di');
const carriageBodyInstance = modelPool.acquire('sm_huoche_youtong_shen');
```

如果后续确认统一切换到完整模型 `SM_火车油罐.glb`，需要先把它注册到 `src/assets/index.ts` 的 `MODEL_URL_MAP`，并在 `scene.json` 的 `modelRegistry` 中建立对应 `modelId`。

## 2. TA 目标

本需求不是做真实流体模拟，而是在 playable 广告项目中用低成本方法制造可信的容器液体效果。

目标效果：

1. 透明油罐内部能看到深色石油。
2. 石油表面具有轻微高光和流动感。
3. 石油液位可以随游戏油量变化。
4. 火车运动时液面可以有轻微晃动，但不追求真实流体物理。
5. 移动端性能开销可控。

非目标：

1. 不做真实流体模拟。
2. 不做复杂折射或屏幕空间 refraction。
3. 不在第一版中实现精确容器裁切。
4. 不在第一版中实现多节油罐真实容量分摊。
5. 不把油面尺寸、位置等静态调参硬编码到 TS 代码中。
6. 不把方案做成只能服务火车油罐的一次性特效。

## 3. 核心思路

容器液体效果通常由三层组成：

1. 透明容器外壳
2. 内部液体体积
3. 顶部液体表面

从视觉优先级看，第一版最重要的是顶部液体表面。玩家从上方斜视角看到一个深色、有高光、有轻微流动的油面，就会相信油罐里有石油。

推荐的技术方向：

```text
透明罐壳
  └── 内部油面 mesh
        └── 深色 glossy shader / material
```

正式版可以升级为：

```text
container_root
  ├── visual_mesh
  ├── liquid_volume
  └── liquid_surface
```

其中：

- `visual_mesh` 是容器本体，例如火车油罐、水桶、玻璃瓶或储液罐。
- `liquid_volume` 负责从侧面看到的液体厚度，可选。
- `liquid_surface` 负责从上方看到的波纹、高光和流动。

### 3.1 通用液体容器效果

虽然当前第一个落地对象是火车油罐，但技术设计不应做成 `TrainTankOnly` 的一次性方案。更合理的抽象是：

```text
通用 Liquid Container Effect
  + 每个容器自己的 marker / bounds / tuning
```

用 Unity Prefab 类比：

```text
LiquidContainer prefab
  ≈ Babylon 里的 LiquidContainerService

Prefab 挂点
  ≈ GLB 里的 liquid_socket / liquid_min_marker / liquid_max_marker

Prefab 参数
  ≈ scene.json 或 manifest 里的 liquid 配置

Prefab 可见子对象
  ≈ liquid_surface / liquid_volume mesh
```

通用的应该是系统能力，而不是每个容器的具体尺寸。不同容器仍然需要自己的适配数据：

1. 液体中心在哪里。
2. 最低液位在哪里。
3. 最高液位在哪里。
4. 液面宽度和长度是多少。
5. 液面形状是圆形、椭圆、圆角矩形还是自定义 mesh。
6. 高度轴使用哪个局部轴。
7. 使用哪种液体材质预设，例如 `dark_oil`、`clear_water`、`gasoline`。

因此推荐长期架构是：

```text
LiquidContainerService
  通用运行时系统

Liquid Container Protocol
  通用 GLB 节点协议

container-specific tuning
  每个容器的尺寸、液位、材质和形状参数
```

火车油罐只是第一个使用这套协议的样板资产。

## 4. 三种实现等级

### 4.1 方案 A：单油面平面

在透明油罐内部创建一个椭圆或圆角矩形油面 mesh。

优点：

- 实现最快。
- 性能最便宜。
- 适合当前 playable 项目。
- 当前摄像机多为上方斜视角，视觉收益高。

缺点：

- 侧面体积感弱。
- 透明外壳很清晰时，油可能像一张贴片。

适用阶段：

- 第一版。
- 用于验证视觉方向和液位联动。

### 4.2 方案 B：油体 + 油面

在油罐内部创建一个深色油体 mesh，再在顶部放油面 mesh。

优点：

- 侧面能看到油量高度。
- 透明容器下更有真实体积感。
- 仍然比真实流体便宜很多。

缺点：

- 需要更多尺寸调参。
- 透明排序问题更明显。

适用阶段：

- 第一版通过后作为正式视觉版本。

### 4.3 方案 C：Shader 裁切液体体积

创建一个完整液体体积 mesh，在 shader 中根据 `fillLevel` 裁掉液位以上部分。

优点：

- 体积与液位表现更统一。
- 可扩展液面倾斜、晃动和更复杂裁切。

缺点：

- 开发和调试成本高。
- 透明排序、背面渲染、深度写入更难处理。
- 对当前需求属于过早复杂化。

适用阶段：

- 后期精修。
- 需要较强 TA 投入时再考虑。

## 5. 推荐落地路线

### Phase 1：可见油面

目标：

- 油罐内部出现深色油面。
- 油面不会穿出透明外壳。
- 油面有基础高光。

实现内容：

1. 在容器 root 下创建 `liquid_surface` mesh。
2. 使用椭圆形或圆角矩形平面。
3. 创建深色石油材质，可作为 `dark_oil` 材质预设。
4. 把油面位置、缩放、最小液位、最大液位写入 `scene.json`。

### Phase 2：油量驱动液位

目标：

- 油面高度随 `OilInventorySystem` 中的油量比例变化。

实现内容：

1. 增加 `setLiquidContainerFillRatio(containerId, ratio)` 之类的运行时接口。
2. 在 `Game.initSystems()` 的 `oilInventorySystem.onOilChanged` 中同步液位。
3. 对 ratio 做 clamp，保证范围为 `0..1`。
4. 使用插值让液位变化更柔和。

### Phase 3：表面流动和轻微晃动

目标：

- 石油表面不再是静态黑面。
- 火车移动时油面有轻微动态。

实现内容：

1. 使用 `ShaderMaterial`。
2. 复用现有 `reference-water-normalmap.png` 做表面细节。
3. 通过 `time` uniform 让 UV 缓慢漂移。
4. 根据 `speedRatio` 增加轻微波纹强度。

### Phase 4：油体体积

目标：

- 从侧面看也能感知油量高度。

实现内容：

1. 在容器内增加 `liquid_volume` mesh。
2. 油体高度随 `fillRatio` 缩放。
3. 顶部油面放在油体顶部。

## 6. 资产要求

推荐先建立通用的 `Liquid Container Protocol v0.1`，再让火车油罐作为第一个实现该协议的资产。

通用容器层级：

```text
container_root
  ├── visual_mesh
  ├── liquid_socket
  ├── liquid_min_marker
  ├── liquid_max_marker
  └── liquid_surface
```

节点说明：

- `container_root`：容器资产根节点。可以是油罐、水桶、瓶子、锅或其他容器。
- `visual_mesh`：原始可见模型。具体名字可以保持资产语义，例如 `SM_M_火车油罐`。
- `liquid_socket`：液体系统中心挂点，用于建立液体局部空间。
- `liquid_min_marker`：最低液位标记。
- `liquid_max_marker`：最高液位标记。
- `liquid_surface`：可见液面。第一版建议在 Blender 中创建，便于直接观察和调参。

第一版必需节点：

```text
liquid_socket
liquid_min_marker
liquid_max_marker
liquid_surface
```

后续可选扩展节点：

```text
liquid_volume
liquid_clip_bounds
liquid_flow_direction
```

当前火车油罐建议层级：

```text
container_train_tank_root
  ├── SM_M_火车油罐
  ├── liquid_socket
  ├── liquid_min_marker
  ├── liquid_max_marker
  └── liquid_surface
```

注意：不要把协议节点命名成 `train_tank_liquid_socket`、`bucket_liquid_socket` 这类资产专用名字。不同资产的 root 可以不同，但协议节点名应保持一致。运行时只需要识别 `liquid_*` 协议节点，就可以复用同一套液体容器系统。

如果无法修改 GLB，也可以用配置方式指定液体参数。

推荐配置形状：

```json
{
  "tuning": {
    "liquidContainers": {
      "trainTank": {
        "enabled": true,
        "containerRootName": "container_train_tank_root",
        "materialPreset": "dark_oil",
        "surfaceShape": "ellipse",
        "fillAxis": "localZ",
        "surfaceLocalPosition": {
          "x": 0,
          "y": 0,
          "z": 1.45
        },
        "surfaceLocalScale": {
          "x": 1.15,
          "y": 2.8,
          "z": 1
        },
        "minFill": 0.15,
        "maxFill": 0.85,
        "surfaceColor": {
          "r": 0.02,
          "g": 0.025,
          "b": 0.025
        },
        "highlightColor": {
          "r": 0.32,
          "g": 0.45,
          "b": 0.5
        }
      }
    }
  }
}
```

说明：

- `surfaceLocalPosition`：油面在油罐局部空间中的基础位置。
- `surfaceLocalScale`：油面局部缩放，用于把圆形拉成椭圆。
- `fillAxis`：液位高度使用的局部轴。Blender 中通常是 `localZ`，Babylon 运行时需要结合导出轴向确认。
- `minFill`：空罐时的归一化最低液位。
- `maxFill`：满罐时的归一化最高液位。
- `materialPreset`：液体材质预设，例如 `dark_oil`。
- `surfaceColor`：石油主体颜色。
- `highlightColor`：高光颜色。

按照项目规范，这些静态场景和调参数据应维护在 `scene.json`，不要硬编码到 TS/JS 代码里。

如果第一版为了最小改动继续使用 `trainTankLiquid` 作为配置 key，也应把内部字段设计成可迁移到 `liquidContainers` 的通用结构，避免后续接入其他容器时重写系统。

## 7. Babylon 实现要点

### 7.1 油面 Mesh

第一版可以使用 `MeshBuilder.CreateDisc` 创建椭圆油面：

```ts
const oilSurface = MeshBuilder.CreateDisc(
  'liquid_surface',
  {
    radius: 1,
    tessellation: 64,
  },
  scene,
);

oilSurface.parent = containerRoot;
oilSurface.rotation.x = Math.PI / 2;
oilSurface.position.set(0, 1.45, 0);
oilSurface.scaling.set(1.15, 2.8, 1);
```

注意：实际轴向需要根据 GLB 局部坐标确认。如果 `CreateDisc` 的平面朝向不对，需要调整 `rotation`。

也可以使用 `MeshBuilder.CreateGround` 创建矩形油面，再在 shader 里裁出圆角形状。

### 7.2 石油材质

第一版可以先使用 `PBRMaterial`：

```ts
const oilMaterial = new PBRMaterial('train_tank_oil_material', scene);
oilMaterial.albedoColor = new Color3(0.02, 0.025, 0.025);
oilMaterial.emissiveColor = new Color3(0.006, 0.008, 0.01);
oilMaterial.metallic = 0;
oilMaterial.roughness = 0.18;
oilMaterial.alpha = 0.95;
```

更正式的版本建议使用 `ShaderMaterial`，增加：

- 时间流动
- normal map 扰动
- 拉长高光
- 视角边缘 Fresnel
- 轻微顶点起伏

现有项目中可参考：

- `src/services/OilSurfaceService.ts`
- `src/services/WaterSurfaceService.ts`

这两个服务已经展示了如何在 Babylon 中注册 shader、设置 uniform、更新 `time`。

### 7.3 液位更新

只做油面时：

```ts
const fillRatio = Math.max(0, Math.min(1, ratio));
oilSurface.position.y = minFillY + (maxFillY - minFillY) * fillRatio;
```

油体 + 油面时：

```ts
const fillRatio = Math.max(0, Math.min(1, ratio));
const height = maxFillY - minFillY;

oilVolume.scaling.y = Math.max(0.001, fillRatio);
oilVolume.position.y = minFillY + height * fillRatio * 0.5;
oilSurface.position.y = minFillY + height * fillRatio;
```

为了避免油量突变造成跳动，可以用平滑插值：

```ts
displayFillRatio += (targetFillRatio - displayFillRatio) * Math.min(1, deltaTime * 6);
```

## 8. 透明排序注意事项

透明容器液体最大的坑不是 shader，而是透明排序。

常见问题：

1. 油面被透明外壳挡住，看不到。
2. 油面渲染到外壳前面，看起来穿帮。
3. 多节车厢透明排序互相影响。
4. 外壳写入深度后，内部液体被错误遮挡。

建议：

1. 透明外壳和液体使用独立材质。
2. 外壳 alpha 控制在 `0.35..0.65`。
3. 油面 alpha 控制在 `0.85..1.0`。
4. 油面 mesh 略微缩小，避免贴到外壳内壁。
5. 必要时控制 `renderingGroupId` 或 `alphaIndex`。
6. 如果油面仍不稳定，优先让油面更接近不透明，减少透明混合复杂度。

理想渲染顺序：

```text
不透明底盘和车架
石油液体
透明油罐外壳
```

如果后续需要把这些排序参数配置化，可以扩展 `scene.json` 或材质配置结构，而不是在业务代码中写死具体节点名。

## 9. 建议代码架构

不建议把液体逻辑直接塞进 `Game.ts`。

推荐新增通用服务：

```text
src/services/LiquidContainerService.ts
```

职责：

```ts
class LiquidContainerService {
  attachToContainer(root: TransformNode, config: LiquidContainerConfig): LiquidContainerHandle;
  setFillRatio(containerId: string, ratio: number): void;
  update(deltaTime: number, speedRatio: number): void;
  dispose(): void;
}
```

其中：

- `attachToContainer()`：给某个容器 root 创建或绑定液体表面。
- `setFillRatio()`：设置目标油量比例。
- `update()`：更新液位插值、时间 uniform、晃动强度。
- `dispose()`：清理 mesh 和 material。

火车油罐不应拥有一套完全独立的液体系统，而应作为 `LiquidContainerService` 的第一个配置实例：

```text
LiquidContainerService
  └── trainTank config / preset
```

接入点：

1. `SceneRailGameplaySystem.buildTankCarriage()`
   - 运行时创建油罐车厢时 attach 对应容器液体。
2. `Game.initSystems()`
   - `oilInventorySystem.onOilChanged` 中同步液位。
3. `SceneRailGameplaySystem.update()`
   - 每帧把 `deltaTime` 和 `speedRatio` 传给液体服务。

如果第一版想保持最小改动，也可以先在 `SceneRailGameplaySystem` 内部维护：

```ts
private readonly tankOilSurfaces: Mesh[] = [];
private tankOilFillRatio = 0;
```

但长期看，单独服务更清晰。

## 10. 与当前油量系统的关系

当前油量系统在 `Game.initSystems()` 中监听油量变化：

```ts
this.oilInventorySystem.onOilChanged((newOil, _delta, capacity) => {
  this.cashBackpackOverlay?.setOil(newOil, capacity);
});
```

未来可以扩展为：

```ts
this.oilInventorySystem.onOilChanged((newOil, _delta, capacity) => {
  this.cashBackpackOverlay?.setOil(newOil, capacity);
  const ratio = capacity <= 0 ? 0 : newOil / capacity;
  this.sceneRailGameplaySystem?.setLiquidContainerFillRatio('trainTank', ratio);
});
```

第一版建议所有油罐车厢使用同一个油量比例。后续如果要做更真实的多车厢分配，再引入 per-carriage fill。

## 11. Shader 方向

石油表面 shader 第一版不需要复杂。

核心视觉：

1. 深色底色。
2. 慢速流动噪声。
3. 拉长高光条纹。
4. 视角边缘高光。
5. 轻微波纹。

伪代码：

```glsl
float wave = sin(worldPos.x * 4.0 + time * 1.2)
           + sin(worldPos.z * 2.5 - time * 0.8);

float sheen = smoothstep(0.65, 0.9, noise + wave * 0.1);

vec3 color = mix(oilDark, oilMid, noise);
color += oilHighlight * sheen;

gl_FragColor = vec4(color, alpha);
```

颜色建议：

```text
oilDark:      #050708
oilMid:       #11191C
oilHighlight: #34505A
oilSheen:     #6A8790
```

如果要节省开发时间，可以先复用：

```text
src/assets/textures/reference-water-normalmap.png
```

作为油面 normal map，只需要把颜色和速度调成更粘稠、更暗即可。

## 12. 液面是否应该保持世界水平

真实液体受重力影响，容器倾斜时液面应保持世界水平。

但当前项目里火车主要是沿轨道 yaw 旋转，不会明显 pitch/roll，所以第一版可以让油面直接 parent 到油罐 root，跟着车厢一起运动。

第一版：

```text
liquidSurface.parent = containerRoot
```

后续高级版：

```text
油面位置跟随 containerRoot
油面旋转尽量保持世界水平
```

高级版需要用世界矩阵和 parent inverse matrix 计算局部姿态，第一版不建议做。

## 13. 测试与验收

自动化测试建议覆盖：

1. `liquidContainers` 配置解析默认值。
2. `fillRatio` 被 clamp 到 `0..1`。
3. 创建多节油罐车厢时，每节车厢都会创建液体 handle。
4. `dispose()` 后 mesh/material 不泄漏。

人工验收重点：

1. 油罐空、半满、满三种状态都清晰可读。
2. 油面不会穿出透明外壳。
3. 摄像机靠近和远离时透明排序稳定。
4. 多节车厢同时存在时不闪烁。
5. 移动端性能稳定。

## 14. Blender 原型制作记录

本节记录第一次使用 Blender MCP / bpy 制作火车油罐石油液面原型时遇到的问题。后续继续制作液体容器资产时，应优先参考这些经验。

### 14.1 本次会话内原型结构

当前原型没有保存回源 `.blend` 文件，只在 Blender 会话内通过 MCP/bpy 创建。

原型层级：

```text
container_train_tank_root
  ├── SM_M_火车油罐
  ├── liquid_socket
  ├── liquid_min_marker
  ├── liquid_max_marker
  ├── liquid_depth_shadow_preview
  ├── liquid_surface
  ├── liquid_meniscus_rim
  ├── liquid_surface_highlight
  ├── liquid_surface_glint
  ├── liquid_oil_reflection_arc
  └── liquid_oil_reflection_short_arc
```

关键自定义属性：

```text
liquid_protocol_version: 0.1
liquid_container_type: train_tank
liquid_type: petroleum_oil
default_fill_ratio: 0.606
```

这里的 `liquid_oil_reflection_arc` 和 `liquid_oil_reflection_short_arc` 是 DCC 预览用高光线，主要帮助人在 Blender 视口中读出“有光泽的油面”。正式游戏实现时，高光更适合交给 shader、normal map 或运行时材质参数处理。

### 14.2 Blender 中文界面会本地化节点名称

本次调试中遇到一个重要问题：Blender 5.1 中文界面里，`Principled BSDF` 节点的 `node.name` 会显示为：

```text
原理化 BSDF
```

如果脚本使用下面这种写法：

```python
bsdf = mat.node_tree.nodes.get("Principled BSDF")
```

在中文界面下可能拿不到节点。结果是脚本看起来执行成功了，也修改了 `mat.diffuse_color`，但真正的 PBR 节点参数没有变化，所以材质在视口中仍然偏灰或不符合预期。

正确做法是按节点类型查找：

```python
def get_principled_node(mat):
    if not mat or not mat.use_nodes:
        return None

    for node in mat.node_tree.nodes:
        if node.bl_idname == "ShaderNodeBsdfPrincipled":
            return node

    return None
```

然后修改节点输入：

```python
bsdf = get_principled_node(mat)
if bsdf:
    if "Base Color" in bsdf.inputs:
        bsdf.inputs["Base Color"].default_value = (0.0, 0.001, 0.0008, 1.0)
    if "Alpha" in bsdf.inputs:
        bsdf.inputs["Alpha"].default_value = 1.0
    if "Roughness" in bsdf.inputs:
        bsdf.inputs["Roughness"].default_value = 0.08
    if "Coat Weight" in bsdf.inputs:
        bsdf.inputs["Coat Weight"].default_value = 0.6
```

结论：资产自动化脚本不要依赖界面显示名称，尤其是在中文 Blender 环境中。对象名、材质名可以按项目约定使用中文，但 Blender 内置节点应优先按 `bl_idname`、输入名称或类型处理。

### 14.3 只改 `diffuse_color` 不等于改了材质

Blender 材质调试时需要区分：

```text
mat.diffuse_color:
  主要影响视口对象色和部分预览显示。

Principled BSDF Base Color / Alpha / Roughness:
  影响实际材质节点。
```

本次最初把石油材质设成很深的颜色，但油面截图仍然偏灰，原因就是没有真正改到 Principled BSDF 节点。

调试材质时建议打印：

```python
for node in mat.node_tree.nodes:
    print(node.name, node.bl_idname)
    print([input.name for input in node.inputs])
```

如果材质效果不符合预期，先确认节点输入是否真的被修改，再考虑透明排序或视口显示问题。

### 14.4 MCP 截图适合沟通，不适合作最终渲染依据

Blender MCP 的 `get_viewport_screenshot` 非常适合快速确认：

1. 对象是否创建成功。
2. 液面位置是否大致正确。
3. 层级和选中对象是否符合预期。
4. 透明外壳内是否能看到液体的大致形状。

但它不等价于最终游戏画面。原因包括：

1. 视口可能处于 Solid/Object Mode，而不是完整材质或渲染预览。
2. 半透明油罐外壳会把内部深色油面混成灰色。
3. 透明排序会影响油面、高光和外壳之间的显示结果。
4. 选中对象橙色描边会干扰截图判断。
5. 视口材质预览不等于 Babylon/WebGL 中的真实 shader 和排序规则。

因此 MCP 截图只能作为 DCC 迭代反馈。正式验收仍要在运行时场景里检查：

```text
Babylon 渲染顺序
PBR/ShaderMaterial 参数
alphaIndex / renderingGroupId
摄像机角度
移动端真实性能
```

### 14.5 透明容器内液体要优先保证可读性

本次原型中，黑色液面放在半透明蓝色油罐顶部下方时，会被视口混合成灰蓝色。为了让第一版可读，可以接受一些非真实但稳定的 authoring 做法：

1. 液面材质尽量接近不透明。
2. 液面 mesh 比内壁略小，避免贴边闪烁。
3. 液面略微抬高，避免和透明顶面发生深度或透明排序冲突。
4. 边缘附着圈 `liquid_meniscus_rim` 可以帮助表达液体接触容器壁。
5. 预览阶段可以用细曲线或细 mesh 做高光，正式版再替换为 shader 高光。

第一版的目标不是物理真实，而是玩家能立即读懂：

```text
这个透明油罐里有深色、有光泽的石油。
```

### 14.6 动态液面不要使用中心扇形三角面

本次原型中曾经出现过一种很典型的问题：从 Top 视角看，液面上有明显的 X 型或放射状纹理。它看起来像材质、高光或透明排序问题，但根因不是材质，而是液面 mesh 的拓扑。

早期液面使用过类似下面的结构：

```text
中心点
  向外连接一圈边界点
  形成 triangle fan
```

这种结构静态时很省面，但不适合做动态液体。原因是所有三角面都共享中心点，波纹、晃动或 `shade_smooth` 会把中心点附近的法线和亮暗变化放大，最终在透明外壳下形成 X 型、放射状或中心收束的假折痕。

更稳定的做法是把液面主体换成规则行列网格：

```text
rounded_rect_grid
row_column_grid_no_center_fan
```

当前油罐原型验证过的网格数据：

```text
surface vertices: 1007
surface faces: 936
face vertex counts: [4]
grid x segments: 18
grid y segments: 52
```

优化后的周期截图里，Top 组不再出现中心 X 型收束。数值样本显示液面动态高度范围约为：

```text
0.0168m .. 0.0447m
```

这个范围已经从“夸张的折面运动”收敛到更像重油/石油的低幅度慢波动。

经验规则：

1. 静态液面可以用简单面片，但动态液面不要用中心 fan。
2. 会做顶点位移、shader 波纹或 frame handler 预览的液面，应优先用均匀网格。
3. 边缘附着圈 `liquid_meniscus_rim` 单独做，不要依赖液面 mesh 的三角边缘制造厚度。
4. 如果看到 X 型液面，先查拓扑，再调材质。

检查拓扑时可以使用：

```python
from collections import Counter

obj = bpy.data.objects.get("liquid_surface")
print(Counter(len(poly.vertices) for poly in obj.data.polygons))
```

如果输出里有大量三角面，并且这些三角面共享中心点，就应该重建液面网格。

### 14.7 Review sheet 也要当成可验证工具来做

这次批量截图分析还遇到一个工具层面的细节：用 Blender 脚本把 PNG 拼成 contact sheet 时，如果手动创建 mesh 平面但没有写 UV，Image Texture 会采样不到图片，最终拼图会变成黑色方块。

这不代表原始截图失败。判断顺序应该是：

1. 先单独打开一张原始 PNG，确认截图是否正常。
2. 如果原图正常、拼图发黑，检查 contact sheet 的材质和 UV。
3. 手动 `from_pydata` 创建平面时，必须补 UV。

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

这类 review sheet 虽然只是临时分析工具，但它直接影响 AI 和人的判断质量，所以也要记录 manifest、帧号、视角和关键参数。

### 14.8 马赛克感通常来自透明预览方式

本次播放预览中，液面曾经出现明显的马赛克/颗粒感，并且液面上有白色圆弧线。后续判断如下：

1. 白色圆弧线来自早期 authoring preview 高光对象。
2. 大片马赛克感主要来自 `HASHED / DITHERED` 透明预览方式。
3. `liquid_depth_shadow_preview` 这种深色叠层会让油面变脏，应先隐藏它建立干净基线。
4. 透明油罐外壳和液面叠加时，Blender 视口预览会比最终运行时更容易出现噪点。

当前已删除的预览高光对象：

```text
liquid_oil_reflection_arc
liquid_oil_reflection_short_arc
liquid_surface_highlight
liquid_surface_glint
```

这些对象只适合早期说明“油面应该有高光”。当它们变成画面里的白色圆弧干扰物时，应直接删除或隐藏。正式游戏效果不应该依赖这些 DCC 圆弧线，而应交给 shader、高光参数或更克制的贴图/法线方案。

检查材质透明方式：

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

调试预览阶段，如果发现材质是：

```text
blend_method: HASHED
surface_render_method: DITHERED
```

可以先切成：

```python
mat.blend_method = "BLEND"
mat.surface_render_method = "BLENDED"
mat.show_transparent_back = False
```

对于石油液面，第一版更应该先追求：

```text
深色
稳定
低噪点
少量慢速动态
```

而不是一开始就叠加很多高光线、阴影层和强反射。等油面本体成立后，再逐步加回高光和深浅变化。

### 14.9 用 spring surface 替代整面倾斜

后续预览中又暴露了一个问题：即使液面可以播放动态，如果只是把整张 `liquid_surface` 做统一 pitch / roll 倾斜，它仍然会像一块硬板在晃，而不是容器里的液体。

因此当前原型已改为 spring surface 方向：

```text
每个顶点维护自己的高度位移 displacement。
每个顶点维护自己的速度 velocity。
低频惯性目标推动液面。
spring_stiffness 控制回弹速度。
viscosity 控制粘滞和阻尼。
edge_lift 让靠近容器壁的区域有轻微爬升/滞后。
surface_noise 只保留很低的细节。
```

这比整面倾斜更适合火车油罐，因为油罐顶部可见面积很大，玩家容易看出液面是否是一整块硬板。

当前 root 上保留的 spring surface 控制项：

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

当前强预览参数：

```text
liquid_motion_preview_scale: 1.8
liquid_inertia_strength: 0.055
liquid_spring_stiffness: 9.0
liquid_viscosity: 5.5
liquid_edge_lift: 0.03
liquid_surface_noise: 0.003
```

逐帧播放验证时，液面高度变化可达到约：

```text
z_range: 0.12m .. 0.16m
```

这是 authoring 强预览，不是最终参数。它的目的只是让人清楚看到：

```text
液面不是整块刚性平面；
不同区域存在滞后；
边缘有轻微堆积和爬升；
石油整体仍然偏慢、偏重。
```

如果后续要进入更接近正式效果的状态，应优先降低：

```text
liquid_motion_preview_scale
liquid_inertia_strength
liquid_edge_lift
```

而不是重新加白色圆弧或强深度阴影。高光和油面反射应在液面动态成立后，再用材质或运行时 shader 处理。

### 14.10 用邻居传播的 2D wave 继续优化

spring surface 解决了“整张刚性平面一起转”的问题，但如果每个顶点仍然主要追自己的目标高度，视觉上仍可能像一张软板，而不是液体波在容器里传播。

因此当前原型继续升级为 2D damped wave surface：

```text
height[i]:
  当前顶点局部高度。

velocity[i]:
  当前顶点垂直速度。

neighbor_average - height[i]:
  邻居传播项，让波横向扩散。

external edge drive:
  只在边缘和端部注入惯性能量，避免整面一起抬升。

viscosity:
  阻尼和粘滞感，石油应偏高。
```

当前 `liquid_surface` 已经有足够的基础网格：

```text
vertices: 1007
faces: 936
average neighbor count: 3.86
```

这说明当前问题不是“面数太少”，而是动态模型要从独立弹簧点升级到邻居传播。

当前 2D wave 控制项：

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

当前干净预览参数：

```text
liquid_motion_preview_scale: 1.5
liquid_inertia_strength: 0.045
liquid_wave_propagation: 18.0
liquid_viscosity: 7.5
liquid_edge_lift: 0.020
liquid_surface_detail: 0.0012
```

如果用户观察时说“没有上下浮动”，需要区分：

```text
局部起伏 z_range:
  说明液面内部有波、有边缘传播、有局部高度差。

平均液位 z_avg:
  说明整层液面是否真的整体上下移动。
```

上一版 2D wave surface 已经有局部 `z_range`，但平均高度基本稳定，所以从斜俯视角看仍可能觉得“没有上下浮动”。因此当前强预览又增加了整体液位 bob：

```text
liquid_level_bob_amplitude: 0.030
liquid_level_bob_frequency: 0.42
liquid_motion_preview_scale: 2.2
```

逐帧采样时，平均液位可以从约：

```text
z_avg: 1.32 .. 1.44
```

这能让播放时的上下浮动更容易被肉眼看到。需要注意：这是 authoring 强预览，不等于最终物理正确参数。正式效果应降低 `liquid_motion_preview_scale` 或 `liquid_level_bob_amplitude`。

后续实际观察证明：对于火车油罐这种顶部大面积可见液面，整体 bob 虽然明显，但容易被读成“一整页液面在上下移动”。这是由资产比例决定的：

```text
liquid_surface / inner cavity X ratio: 0.8772
liquid_surface / inner cavity Y ratio: 0.9562
```

也就是说，液面几乎铺满可见内腔。只要整体平均高度 `z_avg` 明显变化，玩家就会看到一整块大面在升降，而不是液体局部起伏。

因此当前方案改为 travelling edge wave：

```text
1. 去掉整体 level bob。
2. 每步积分后移除 height 平均值，让 z_avg 基本恒定。
3. 在油罐前后端和侧边注入相反方向的能量。
4. 依靠邻居传播把波推向中部。
5. 用局部 z_range 表达液体起伏，而不是让整层液面升降。
```

当前 travelling edge wave 参数：

```text
liquid_motion_preview_scale: 1.8
liquid_inertia_strength: 0.085
liquid_wave_propagation: 42.0
liquid_viscosity: 6.2
liquid_edge_lift: 0.045
liquid_surface_detail: 0.0012
```

逐帧采样结果显示：

```text
z_avg:
  基本恒定在 1.37948 附近。

z_range:
  大约 0.008m .. 0.034m。
```

这比整体 bob 更符合当前油罐资产：液面整体油量稳定，但局部有边缘波和端部传入的轻微起伏。

如果仍然希望更清楚地看到“局部液面在起伏”，可以临时使用 local travelling wave 强预览。它不是物理最终方案，而是为了让局部波动在 Blender 视口里更可读：

```text
liquid_local_wave_enabled: true
liquid_motion_preview_scale: 1.0
liquid_local_wave_amplitude: 0.040
liquid_local_wave_speed: 0.34
liquid_local_wave_frequency: 2.4
liquid_edge_wave_strength: 0.030
liquid_cross_wave_strength: 0.012
liquid_viscosity: 7.0
```

这个预览每帧都会移除平均高度，所以：

```text
z_avg:
  基本恒定。

z_range:
  可以达到约 0.06m .. 0.12m，用来确认局部波带确实在传播。
```

为了避免透明外壳和材质预览让人看不清局部波形，可以临时创建：

```text
debug_liquid_wave_profile_centerline
```

这条曲线沿液面中心线采样高度，只用于确认局部波形。它是调试线，不是最终资产，不应导出，也不应保存进正式 `.blend`。

逐帧播放验证中，当前 wave surface 的 `z_range` 大致控制在：

```text
0.003m .. 0.014m
```

这比上一版强 spring preview 更克制，更适合石油。它不追求大幅度波浪，而是让液面有边缘传入、中部滞后和轻微扩散。

### 14.11 避免黑灰条纹和低频波带

后续调试中又出现过一种新的视觉问题：液面颜色变成黑色和灰色相间，看起来像一排排横向条纹，而且油面发糊、不细腻。

这不是单纯的网格精度问题。当前油罐液面可见面积较小，如果几何波形是低频、大幅、方向单一的 travelling wave，Blender 材质预览会把波峰和波谷的法线差异直接显示成大块灰色高光。结果就会像“黑灰相间的布面”，而不是石油。

正确的拆分方式：

```text
几何位移:
  只负责低幅、多方向、去均值的局部起伏。
  目标是让液面局部有生命感，而不是制造强条纹。

材质 / shader:
  负责石油的暖黑棕底色、细腻高光、微小扰动和反射。
  细腻感优先通过 normal / bump / 运行时 shader 表达。
```

当前油罐更适合的预览范围：

```text
liquid_local_wave_amplitude: 0.006 .. 0.014
liquid_local_wave_frequency: 3.0 .. 5.0
liquid_edge_wave_strength: 0.002 .. 0.006
liquid_cross_wave_strength: 0.001 .. 0.004
liquid_viscosity: 9.0 .. 12.0
z_range: 0.008m .. 0.020m
```

如果仍然觉得“动态不够明显”，不要直接把 amplitude 放大到强预览值。更好的做法是改成多方向干涉波：

```text
1. 一个低幅主波沿容器长度传播。
2. 两到三个不同方向、不同速度的次级波打破平行条纹。
3. 每帧减掉全部顶点高度平均值，避免整块液面一起上下浮动。
4. 边缘波只做轻微堆积，不形成白色圆弧或亮边。
```

当前已验证的较稳定预览标记：

```text
liquid_preview_preset: irregular_micro_interference_oil_surface
liquid_wave_model: multi_direction_local_interference_no_global_bob
```

Blender 材质预览阶段，可以先建立暖黑棕、不透明、低灰度高光的基线：

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

正式游戏效果再用环境反射、normal map 或 shader 参数增加石油高光。不要用白色圆弧、高对比灰色条纹或大幅几何波代替石油质感。

### 14.12 使用 motion debug 模式观察油罐运动晃动

当正式油面颜色和高光逐渐接近目标后，又会遇到一个新的调试问题：如果把高光压低，油面会更像石油，但运动不容易看清；如果把高光拉高，又会出现亮斑、重复纹理和不真实的灰色波纹。

因此需要把“正式效果”和“调试可视化”分开：

```text
正式液面:
  深色、低高光，用来判断石油颜色和最终观感。

motion debug:
  用额外的临时线条显示液面高度、运动方向和高低端。
  它只帮助 TA / 美术理解运动，不代表最终游戏画面。
```

当前原型已验证的 motion debug 模式：

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

这个模式不再使用固定方向的连续横纹，而是使用模拟车辆运动输入：

```text
加速:
  液体因惯性向后侧堆积。

松油 / 回弹:
  液面从高端向中部回落。

刹车:
  液体反向向前端堆积。

残余波:
  叠加不规则低幅残波，避免看起来像一张循环波纹贴图。
```

debug 对象命名约定：

```text
debug_liquid_motion_vector_arrow:
  橙色箭头，表示当前车辆运动输入方向。

debug_liquid_high_side_level_hint:
  高位端提示线。

debug_liquid_height_contour_length_*:
  沿油罐长度方向采样的液面高度轮廓线。

debug_liquid_height_contour_width_*:
  沿油罐宽度方向采样的液面高度轮廓线。

debug_liquid_low_end_bar:
  蓝色低端提示。

debug_liquid_high_end_bar:
  橙色高端提示。
```

这些对象都属于临时 authoring debug：

```text
1. 不导出。
2. 不保存进正式资产。
3. 正式截图或交付前应隐藏或删除。
4. 对象上应写入 debug_note，说明用途。
```

性能注意：高度轮廓线不能每帧对所有液面顶点做最近点搜索。第一次创建 overlay 时，应把每个曲线采样点绑定到最近的 `liquid_surface` 顶点索引。frame handler 每帧只读取这些顶点高度并更新曲线点，这样 debug 模式才能保持可播放。

这个 debug 模式的目标不是让液体最终看起来有彩色线条，而是回答下面的问题：

```text
1. 当前车辆运动方向是什么？
2. 液体是否因为惯性向反方向堆积？
3. 高低端是否随加速 / 刹车交替变化？
4. 局部残波是否打破了重复纹理感？
5. z_avg 是否仍然稳定，避免整层液面上下漂？
```

### 14.13 场景化液体 Debug 参数

在继续调试过程中，我们发现单一 `motion debug` 还不够。真正有用的是能快速切换业务场景，分别观察：

```text
1. 车厢前进 / 加速 / 刹车。
2. 车厢加油。
3. 车厢拐弯。
```

因此当前原型增加了统一的场景控制器：

```text
liquid_debug_enabled: true
liquid_debug_scenario: train_forward / refueling / train_turn
liquid_debug_auto_cycle: true
liquid_debug_playback_strength: 1.0
liquid_debug_viscosity: 12.0
liquid_debug_show_overlay: true
liquid_debug_show_motion_arrow: true
liquid_debug_show_height_contours: true
liquid_debug_show_high_low_bars: true
liquid_debug_show_inlet: true
```

对应的 handler：

```text
game_dcc_lab_liquid_scenario_debug_surface_handler:
  根据当前 scenario 计算液面高度场，并更新 liquid_surface 顶点 z。

game_dcc_lab_liquid_scenario_debug_overlay_handler:
  更新箭头、高低端提示、注入口范围和高度轮廓线。
```

#### train_forward

用于模拟车厢前进、加速和刹车。

当前约定：

```text
local +Y:
  视作车头方向。

加速:
  液体因惯性向 local -Y 端堆积。

刹车:
  液体向 local +Y 端堆积。

匀速 / 回稳:
  液面逐渐回到平均高度，只保留残波。
```

参数：

```text
liquid_debug_train_speed
liquid_debug_train_accel
liquid_debug_slosh_strength
liquid_debug_slosh_damping
liquid_debug_slosh_period
```

验证目标：

```text
z_avg:
  基本稳定，表示没有凭空增减油量。

z_range:
  随加速 / 刹车状态变化。
```

#### refueling

用于模拟车厢加油。

视觉目标：

```text
1. 平均液位随时间上升。
2. 注入口附近有局部扰动。
3. 涟漪慢速向外扩散。
4. 不应该整个液面剧烈翻滚。
```

参数：

```text
liquid_debug_fill_rate
liquid_debug_fill_target
liquid_debug_fill_cycle_seconds
liquid_debug_inlet_strength
liquid_debug_inlet_radius
liquid_debug_inlet_offset_x
liquid_debug_inlet_offset_y
liquid_debug_refuel_ripple_strength
liquid_debug_refuel_level_range
```

验证目标：

```text
z_avg:
  应随播放时间上升。

debug_liquid_scenario_inlet_circle:
  显示注入口影响范围。
```

#### train_turn

用于模拟车厢拐弯。

当前约定：

```text
liquid_debug_turn_direction = 1:
  右转，液体向 local -X 侧堆积。

liquid_debug_turn_direction = -1:
  左转，液体向 local +X 侧堆积。
```

参数：

```text
liquid_debug_turn_direction
liquid_debug_turn_strength
liquid_debug_turn_period
liquid_debug_lateral_slosh_strength
liquid_debug_lateral_damping
```

验证目标：

```text
z_avg:
  基本稳定。

高低端 bars:
  应从前后方向切换成左右方向。
```

#### Overlay 对象

当前场景化 debug overlay 使用：

```text
debug_liquid_scenario_motion_arrow:
  运动方向或输入方向。

debug_liquid_scenario_high_bar:
  当前高位端。

debug_liquid_scenario_low_bar:
  当前低位端。

debug_liquid_scenario_inlet_circle:
  加油模式下的注入口范围。

debug_liquid_scenario_height_contour_length_*:
  沿长度方向采样的液面高度轮廓。

debug_liquid_scenario_height_contour_width_*:
  沿宽度方向采样的液面高度轮廓。
```

这些对象只用于 authoring debug，不导出，不保存进正式 `.blend`。如果要做正式运行时表现，应把这些调试信息转成引擎侧的调试绘制或开发者面板。

当前三种模式的初步采样结果：

```text
train_forward:
  z_avg 稳定在 1.37948 左右。
  z_range 大约 0.026m .. 0.036m。

refueling:
  z_avg 从约 1.34449 上升到约 1.42886。
  说明加油模式确实改变平均液位。

train_turn:
  z_avg 稳定在 1.37948 左右。
  z_range 随转弯状态变化，强预览帧可达到约 0.078m。
```

### 14.14 收敛到 Babylon 标准资产状态

确定最终石油效果将在 Babylon 中实现后，Blender 文件不应该继续保留大量 Blender-only debug、handler 和临时反射对象。Blender 的职责收敛为：

```text
1. 提供干净的油罐 mesh。
2. 提供液面 mesh。
3. 提供 socket / min / max marker。
4. 提供 Babylon 运行时需要读取的 metadata。
5. 不再保存 debug 曲线、preview 反射带或 bpy handler。
```

当前标准对象层级：

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

已清理的对象类型：

```text
debug_*
liquid_flow_sheen_preview_*
liquid_broad_reflection_preview_*
liquid_depth_shadow_preview
SM_M_火车油罐_without_transparent_shell_preview
contact sheet / review packet 残留对象:
  card_*
  img_*
  label_*
  sheet_*
  background / bg / cam
```

已移除的 handler：

```text
game_dcc_lab_liquid_shader_flow_handler
game_dcc_lab_liquid_broad_reflection_preview_handler
game_dcc_lab_liquid_scenario_debug_surface_handler
game_dcc_lab_liquid_scenario_debug_overlay_handler
```

标准化后的 `liquid_surface`：

```text
vertices: 1007
faces: 936
face_vertex_counts: 4 only
z_range: 0.0
shape: rounded_rect_grid
topology: row_column_grid_no_center_fan
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
liquid_default_fill_ratio: 0.6759999394416809
liquid_surface_object: liquid_surface
liquid_socket_object: liquid_socket
liquid_min_marker_object: liquid_min_marker
liquid_max_marker_object: liquid_max_marker
liquid_babylon_material: LiquidOilMaterial
liquid_export_policy: export mesh markers metadata; exclude debug and Blender handlers
liquid_standard_state: true
```

`liquid_surface` 标准 metadata：

```text
liquid_protocol_role: liquid_surface
liquid_runtime_role: surface_mesh_for_babylon_shader
liquid_surface_shape: rounded_rect_grid
surface_shape: rounded_rect_grid
surface_topology: row_column_grid_no_center_fan
liquid_base_local_z: 1.3794814948410594
liquid_fill_ratio: 0.6759999394416809
export_include: true
runtime_hint: Babylon replaces placeholder material and drives height/slosh/flow at runtime
```

`liquid_meniscus_rim` 当前保留为可选 authoring reference：

```text
export_include: false
liquid_runtime_role: optional_authoring_reference_not_required_for_babylon
runtime_hint: Optional authoring reference; Babylon can generate meniscus in shader or runtime mesh
```

标准状态验证结果：

```text
object_count: 9
object_type_counts:
  CAMERA: 1
  EMPTY: 4
  LIGHT: 1
  MESH: 3

remaining_liquid_handlers: []

root_children:
  SM_M_火车油罐
  liquid_surface
  liquid_meniscus_rim
  liquid_socket
  liquid_min_marker
  liquid_max_marker
```

注意：恢复原始透明油罐外壳后，Blender Material Preview 中可能仍然看到透明颗粒或噪点。这是 Blender 视口透明显示问题，不代表资产仍有 debug 对象。最终石油视觉、透明排序、Fresnel、normal flow 和 roughness variation 都应在 Babylon 中验收。

### 14.15 透明油罐外壳玻璃预览

标准资产状态恢复原始透明外壳后，Blender Material Preview 中曾出现明显雪花颗粒。这不是液面拓扑问题，也不是 debug 对象残留，而是透明外壳材质和视口透明显示叠加造成的。

当前透明外壳材质位置：

```text
object: SM_M_火车油罐
material slot: 0
material: M_油罐_SM_M_火车油罐
polygon_count: 14
```

优化前的关键问题：

```text
Alpha:
  0.14，过低，Material Preview 中容易产生抖点。

Transmission Weight:
  0.55，和视口透明叠加后噪点明显。

IOR:
  1.0，不符合常见玻璃预期。

use_screen_refraction:
  true，Blender 视口里容易和深色液面叠出雪花。
```

当前采用的稳定玻璃预览参数：

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

调试结论：

```text
Alpha 0.32:
  内部更清楚，但雪花仍明显。

Alpha 0.62:
  雪花基本消失，但外壳偏乳白，像磨砂盖子。

Alpha 0.48:
  当前折中值，雪花明显减少，同时仍能看出玻璃外壳。
```

这只是 Blender 视口稳定预览材质，不是最终游戏玻璃。真正的玻璃效果应在 Babylon 中实现：

```text
TankGlassMaterial
PBRMaterial / NodeMaterial / ShaderMaterial
environment reflection
Fresnel
proper alpha sorting
optional refraction
```

如果后续用户继续追求“更清透”的玻璃，不应在 Blender Material Preview 中无限降低 alpha。更稳的方向是：

```text
1. 保持 Blender 中的稳定预览材质。
2. 在 Babylon 中实现最终玻璃 shader。
3. 用 Babylon 的透明排序和环境反射验收真实效果。
```

### 14.16 黑色石油需要 shader flow 层

当石油颜色接近真实黑色时，会出现一个新的问题：液面太暗，几何起伏和法线变化在 Blender 材质预览里很难读出来。如果直接把 Base Color 调亮，就会变成黄褐色泥水；如果把高光拉得很强，又会出现灰白亮斑和重复波纹。

更合理的做法是把流动感分成几层：

```text
Base Color:
  保持接近黑褐色石油。

Geometry slosh:
  只负责油罐运动时的低频高低端变化。

Shader normal flow:
  两层或多层移动 normal / bump，方向、尺度、速度不同。

Roughness variation:
  用移动 noise 轻微改变粗糙度，让反射锐度漂移。

Sheen preview:
  在 Blender 视口中临时显示低亮度油膜反射痕，帮助看黑油流动。
```

这和常见实时渲染液体做法一致：流动通常靠移动 normal map / flow map / roughness variation，而不是只靠 mesh 变形或 base color 变亮。

当前原型已验证的 shader flow 属性：

```text
liquid_shader_flow_enabled: true
liquid_shader_flow_mode: dual_moving_noise_bump_roughness
liquid_shader_flow_strength: 1.0
liquid_shader_flow_speed_a: 0.018
liquid_shader_flow_speed_b: -0.011
liquid_shader_flow_roughness_drift: 0.006
liquid_preview_preset: dark_oil_shader_flow_low_highlight
```

材质节点结构：

```text
liquid_flow_mapping_large_slow
liquid_flow_noise_large_viscous
liquid_flow_mapping_fine_diagonal
liquid_flow_noise_fine_breakup
liquid_flow_mapping_roughness_drift
liquid_flow_noise_roughness_variation
liquid_flow_bump_moving_micro_normal
liquid_flow_roughness_color_ramp
```

如果 Blender 材质预览仍然太黑，可以添加临时低亮度油膜预览层：

```text
liquid_flow_sheen_preview_*
```

这层不是最终资产，它只是帮助人在 Blender 里读出“黑色油面正在流动”。当前使用的是低 alpha 的暗蓝灰 / 棕灰短曲线，随播放缓慢漂移和扭动，避免规则横线或白色高光圆弧。

当前已验证：

```text
liquid_flow_sheen_preview_enabled: true
liquid_preview_preset: dark_oil_shader_flow_with_subtle_sheen_preview
```

注意：

```text
1. sheen preview 不导出。
2. sheen preview 不保存进正式资产。
3. 正式游戏中应替换为 shader flow mask / normal map / roughness variation。
4. sheen preview 不要做成亮白色，也不要做成规则横向条纹。
5. 为了性能，sheen preview 可以固定抬高到液面上方少量距离，不必每帧采样所有液面顶点。
```

参考实现方向：

```text
1. 两层移动 normal：一层大尺度慢速，一层小尺度斜向。
2. 一层移动 roughness noise：控制反射锐度，而不是改变颜色。
3. 低幅几何 slosh：只保留油罐运动高低端。
4. 低亮度 sheen preview：只用于 Blender authoring 观察。
```

### 14.17 马赛克主要来自透明外壳叠加

针对“液面仍有马赛克感”的问题，本次做了三组诊断：

```text
A_with_shell:
  显示原透明外壳。

B_shell_more_transparent:
  降低透明外壳 alpha。

C_liquid_only:
  隐藏火车外壳，只看液面。
```

结果是：`C_liquid_only` 中液面本体比较干净；显示外壳后马赛克明显加重；把外壳 alpha 从约 `0.14` 降到 `0.04` 后噪点明显减轻。

结论：

```text
当前马赛克主要不是模拟精度不足；
也不是液体网格面数不足；
主要是 Blender 视口中透明油罐外壳叠加深色液面产生的预览噪点。
```

后续分析液体动态时，应先使用 `liquid_only` 或更低外壳 alpha 的视图确认液面本体，再回到带外壳视图看综合效果。最终游戏中仍需要用 Babylon/WebGL 的透明排序和材质参数重新验收。

## 15. 第一版完成标准

第一版不追求完美液体，只追求“可信且可维护”。

完成标准：

1. 透明油罐内部有深色油面。
2. 油面高度跟随油量变化。
3. 油面有轻微高光或流动。
4. 静态尺寸和颜色配置在 `scene.json`。
5. 代码逻辑集中在通用 `LiquidContainerService` 或清晰的系统边界内。
6. 不破坏现有火车、交付、码头吊机流程。
7. 火车油罐作为 `Liquid Container Effect` 的第一个样板资产，而不是一次性专用实现。

一句话总结：

```text
透明油罐 + 黑色高光油面 + 油量驱动高度
```

这是当前项目最合适的容器石油效果第一步。
