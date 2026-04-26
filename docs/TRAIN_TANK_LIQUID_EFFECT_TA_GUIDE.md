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

## 14. 第一版完成标准

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
