# Cesium Terrain Deformation — 实施方案（设计阶段）

> 项目：OceanView + JxOcean，Cesium for Unreal **v2.31.0** / cesium-native **v0.66.0**，UE 5.7（Windows，后续 Android）。
>
> 状态：**仅设计文档，尚未实现、编译或测试代码**。
>
> 核心结论：优先使用已有的 `ACesium3DTileset::SetGltfModifier` + `Cesium3DTilesSelection::GltfModifier`，**不直接修改 Cesium 网格创建核心函数**。Fork 的 `main` 仍跟随官方；此文档放在 `custom/terrain-deform` 分支中。

## 1. 目标与边界

1. 以 JxOcean 的 Polygon / MultiPolygon GeoJSON 为水域范围；支持外环、内环（岛屿孔洞）及每个区域的水面高程。
2. 只修改指定的 Cesium **地形 Tileset**，水域内按水面高度压平/下凹，陆地及岛屿不受影响。
3. 原 Cesium Tile LOD、流式加载和坐标系统继续工作。**同一份修改后的 glTF 顶点**在 Cesium 正常网格加载时生成可见网格和 Chaos 碰撞。
4. 避免 645 个 Cartographic Polygon Actor 和频繁对 Raster Overlay 刷新的首次加载成本；数据仅解析一次，工作线程复用不可变快照。
5. 禁止对普通 3D 建筑/摄影测量 Tileset 默认实施下凹；要求显式绑定选中的地形 Tileset。
6. 首版不做大规模性能基准测试，不开发运行时海岸线建模工具，不承诺任意低精度 LOD 都能精确表达小岛。

## 2. 已在 v2.31.0 源码中确认的公开入口

| 源码位置 | 作用 | 是否需要修改 |
|---|---|---|
| `Source/CesiumRuntime/Public/Cesium3DTileset.h` | 已有 `GetGltfModifier()` / `SetGltfModifier(std::shared_ptr<GltfModifier>)` | **不需要** |
| `Source/CesiumRuntime/Private/Cesium3DTileset.cpp` | `LoadTileset()` 将 `_pGltfModifier` 传入 `TilesetExternals`；设置不同 Modifier 会重建 Tileset | **不需要** |
| `extern/cesium-native/Cesium3DTilesSelection/include/Cesium3DTilesSelection/GltfModifier.h` | `GltfModifier::apply(GltfModifierInput&&)` 在工作线程修改模型；`trigger()` 激活/重新应用 | **不需要** |
| `Source/CesiumRuntime/Private/UnrealPrepareRendererResources.cpp` | 后续在加载线程将 glTF 转为 Unreal 的资源数据 | **不需要** |
| `Source/CesiumRuntime/Private/CesiumGltfComponent.cpp` | 创建 PositionVertexBuffer、法线/切线和 `BuildChaosTriangleMeshes` 碰撞 | **不需要** |
| `CesiumGltf/AccessorWriter.h`（cesium-native） | 在支持的 accessor 类型上写入 glTF POSITION 数据 | **不需要** |

**证明点**：

- `GltfModifierInput` 提供 `previousModel`、`tileTransform`（模型 → Tileset 坐标）、`asyncSystem` 和处理版本。
- `apply` 返回 `Future<std::optional<GltfModifierOutput>>`。未命中水域返回 `std::nullopt`；命中后返回修改过的 `CesiumGltf::Model`。
- `trigger()` 首次激活 Modifier；再次触发可以对已加载瓦片重新修改，期间可能短暂混合新旧版本。
- `CesiumGltfComponent.cpp` 在生成 `FPositionVertexBuffer` 后才执行法线/切线相关处理以及 `BuildChaosTriangleMeshes`。只要在进入这一流程前修改 POSITION，渲染与新建碰撞使用的就是修改后的几何。

官方源码：

- https://github.com/CesiumGS/cesium-unreal/blob/v2.31.0/Source/CesiumRuntime/Public/Cesium3DTileset.h
- https://github.com/CesiumGS/cesium-native/blob/v0.66.0/Cesium3DTilesSelection/include/Cesium3DTilesSelection/GltfModifier.h

## 3. 第一版新增文件（推荐放 OceanView 的独立插件）

```text
OceanView/
└─ Plugins/
   ├─ JxOcean/                            # 现有海洋插件，暂不改水面渲染
   └─ CesiumTerrainDeformer/
      ├─ CesiumTerrainDeformer.uplugin
      ├─ Source/CesiumTerrainDeformer/
      │  ├─ CesiumTerrainDeformer.Build.cs
      │  ├─ Public/
      │  │  └─ CesiumTerrainDeformerComponent.h
      │  └─ Private/
      │     ├─ CesiumTerrainDeformerComponent.cpp  # 编辑器属性/快照准备/挂载 Modifier
      │     ├─ TerrainGeoJsonImporter.h/.cpp       # 解析 Polygon/MultiPolygon + 洞
      │     ├─ TerrainRegionSnapshot.h/.cpp        # 不可变数据和地理空间索引
      │     ├─ TerrainGltfModifier.h/.cpp          # 派生 cesium-native GltfModifier
      │     └─ TerrainVertexDeform.h/.cpp          # 地理高程下凹和缓和函数
      └─ Content/                         # MVP 不要求材质资源
```

依赖：UE 5.7、`CesiumRuntime`（PublicDependencyModuleNames）、`Core`、`CoreUObject`、`Engine`、`Json`、`JsonUtilities`；C++20。CesiumRuntime 的现有 Build.cs 已暴露 Cesium Native 第三方头文件和平台库，但仍应对实际项目构建验证。

**不建议**让 CesiumRuntime 直接引用 JxOceanRuntime；会造成反向依赖和未来合并冲突。让两者通过 GeoJSON 文件/已预处理的数据快照进行关联。

## 4. 组件编辑器参数

组件 `UCesiumTerrainDeformerComponent` 应可附加到选中的 `ACesium3DTileset` 上：

| 参数 | 默认 | 含义 |
|---|---|---|
| Enable Terrain Deformation | false | 默认不触碰原始 Cesium 地形 |
| Water Region GeoJSON | 空 | Polygon / MultiPolygon，支持内环 |
| Default Water Height Meters | 0 | 数据缺失高度时的 WGS84 椭球水面高程 |
| Water Surface Height Offset Meters | 0 | 与 JxOcean `ActorSurfaceHeightOffsetMeters` 对齐 |
| Depression Depth Meters | 5 | 水面以下目标地形深度 |
| Shore Blend Width Meters | 10 | 岸线过渡宽度，0 = 硬过渡 |
| Deform Collision | true | 使用原有 `CreatePhysicsMeshes`，不实现第二套碰撞网格 |
| Rebuild/Reload Deformation | CallInEditor | 更改 GeoJSON、高度等后重新准备快照、同步渲染 |

水位统一使用 **WGS84 椭球高程**。如果 GeoJSON 属性或 DEM 是正高/其他基准，必须先转换，不能直接将海平面 0m 当成任意 Cesium 数据的共同基准。

## 5. GeoJSON 导入与空间索引

1. 在游戏线程/编辑器预处理阶段解析 GeoJSON；可以参考 `Plugins/JxOcean/Source/JxOcean/Private/JxOceanComponent.cpp` 的现有解析器及 `IsPointInRegion`。
2. 按 GeoJSON 规范分组：每个 Polygon 的 `rings[0]` 是外环，`rings[1..]` 是洞；MultiPolygon 逐块处理，不靠经纬度环方向推断内外。
3. 删除重复闭合点、退化环；必要时简化与修复无效几何，但**不要仅靠移除小岛来追求性能**，要有用户明确的精度/面积阈值。
4. 转换为不可变 `std::shared_ptr<const FTerrainRegionSnapshot>`，并建立外环/孔洞/线段的分层包围盒索引（地理 R-tree、经纬度瓦片桶或等效结构）。
5. 提供 `QueryRelevantPolygons(bounds)`、`ContainsWater(lon,lat)`、`DistanceToShoreMeters(lon,lat)`、`GetWaterHeight`。点位判定：`insideOuter && !insideAnyHole`。
6. 可选的后续优化：将已解析数据和索引保存为版本化本地二进制缓存，启动时避免重新解析 JSON；修改源文件才重新生成。
7. 快照完成后才启用/触发 GltfModifier；加载线程不得读取 UPROPERTY、World、UObject 或重新解析 GeoJSON。

## 6. glTF 顶点下凹：GltfModifier 的实现重点

### 6.1 生命周期

```text
CesiumTerrainDeformerComponent::OnRegister / 显式 Apply
    ↓ 解析 GeoJSON、创建不可变空间索引
    ↓ make_shared<FTerrainGltfModifier>(snapshot)
    ↓ TargetTileset->SetGltfModifier(modifier)  [GameThread]
    ↓ modifier->trigger()                       [GameThread]
    ↓ Cesium 已有 Tile loader / cesium-native   [Worker]
    ↓ FTerrainGltfModifier::apply(input)
    ↓ 返回修改后的 glTF Model
    ↓ Cesium 原有 Unreal 网格 + Chaos 碰撞加载流程
```

必须正确管理生命周期：如果已有别的插件设置了 `GltfModifier`，不能无提示覆盖；MVP 检测到非本插件的 Modifier 时明确报冲突，后续设计可组合 Modifier。卸载时仅恢复本插件设置的 Modifier。编辑器中重复 Register 不能重复触发全 Tileset 重载。

### 6.2 单瓦片算法

1. 确认当前为支持的地形 Tileset（由组件显式配置，默认不允许建筑 3D Tileset）。
2. 从 glTF 的 nodes / meshes / primitives / POSITION accessor 计算瓦片/模型实际 ECEF 地理包围范围；包含 glTF node 层级变换、`CESIUM_RTC` 偏移及 glTF up-axis。
3. 使用 GeoJSON 空间索引快速判断候选相交区域；不相交返回 `std::nullopt`，**不要复制整个 glTF**。
4. 相交时复制 `input.previousModel` 为 `GltfModifierOutput.modifiedModel`；对其中需要下凹的 POSITION 使用 `CesiumGltf::AccessorWriter` 或新的浮点 accessor。
5. 对每个顶点：glTF 本地坐标 → Tileset/ECEF（double）→ 椭球经纬度高程 → 水域/洞判断和岸线距离 → 修改高程 → ECEF → 原 glTF 本地坐标。
6. `tileTransform` 是模型到 Tileset 坐标的变换；实现必须核实最终 Tileset 坐标与椭球 ECEF 的关系。不可忽略 node matrix、RTC、up-axis，亦不可直接把 Unreal Z 减去 `Depth * 100`。
7. 更新 POSITION accessor 的 min/max，以及所有依赖顶点几何的 bounds；如果原 mesh 有 normals/tangents，变形区的法线/切线必须正确重建或在后续 Unreal 端重新生成（不能保留错误的坡面光照）。
8. 若某个 POSITION accessor 被多个具有不同 transform 的 node 共享，不能直接原地写同一 accessor 导致其他实例错误下凹；需要拆分 accessor/primitive，或 MVP 拒绝这类瓦片并给出警告。
9. 保留 indices、UV、影像映射和原材质绑定；只修改指定地形 POSITION（以及确有必要的 normal/tangent/相关元数据）。
10. 类型限制：初版先支持已经是可写浮点 `POSITION` 的三角网格；压缩/量化/整数 accessor 必须先正确解码和新建浮点 accessor，不能把 `float` 直接写进非 float 缓冲。

### 6.3 高程和缓和公式

记当前椭球高程 `h`，区域水面高程 `hw`，下凹深度 `D`，岸线内侧距离 `d`，过渡宽度 `W`：

```text
hTarget = min(h, hw - D)
t = clamp(d / W, 0, 1)
weight = t*t*(3 - 2*t)  # W <= 0 时改用硬判定
hNew = h + weight * (hTarget - h)
```

- 外部 / 岛屿洞内：`weight = 0`，完全不改。
- 水域深处：`weight = 1`，目标高度为水面高程减 `D`；若原本更低则保持不变。
- 岸线：平滑过渡，避免可见垂直断崖。
- 内环孔洞的岸线同样需要过渡，避免岛屿边缘断裂。
- 同一地理位置下凹函数必须确定性，避免不同 LOD/瓦片之间计算结果不一致。

## 7. 渲染、碰撞、LOD 和剔除风险

**渲染与碰撞一致性：** v2.31.0 的 Unreal 网格加载在 `CesiumGltfComponent.cpp` 中由 `POSITION` 创建 `FPositionVertexBuffer`，再用该 Buffer 调用 `BuildChaosTriangleMeshes`；故 GltfModifier 修改输入 glTF 顶点后，`CreatePhysicsMeshes = true` 时可以保持新建碰撞与渲染顶点一致。必须仍检查法线/切线以及模型实例化路径。

**包围盒：** glTF POSITION accessor 的 min/max 要更新，且 Cesium 的外层 Tile Bounding Volume（来自 3D Tiles 元数据）不一定因 glTF 被修改而自动扩大。下凹较深时可能被错误剔除。需要专门检查 CPU 视锥/遮挡剔除和 bounds；不能只修改 `UStaticMesh` Bounds 就声称 Tile 层包围盒已同步。

**岸线精度：** 修改顶点只会移动原有顶点，不会自动沿岸线添加新顶点。粗 LOD 跨越海岸线的三角形可能仍穿出水面；若要像素级贴岸，未来加边界三角形细分/局部精化，或者用遮罩材质辅助，不能假定 GeoJSON 边界会自动被原始网格精确表达。

**瓦片接缝：** 对相同 ECEF 坐标应用同一个变形函数可提高接缝连续性，但需要验证 Quantized Mesh 的裙边/skirt、LOD 替换、重复点和法线一致性。

**热更新：** `GltfModifier::trigger()` 会重新处理已加载的模型，且可能暂时新旧混显。尤其注意 `previousModel` 可能已是前一次修改后的 glTF，简单再次 `min()` 不能恢复地形或处理提高水位/减小深度；MVP 可在修改配置时重建 Tileset 从原始数据重新应用，后续再引入原始 POSITION 缓存和版本化快照。

**异步线程：** `apply` 只操作自身引用的不可变快照和 glTF Model，切勿在 worker 中调用 UObject/Blueprint/GetWorld 或修改编辑器 Actor。

**适用范围：** 首版仅支持指定的地形 Tileset；不要默认应用于 3D Buildings、摄影测量和含 GPU instancing 的模型。

## 8. 是否还需要修改 Cesium Fork？

**首选方案（0 处 Cesium 核心源码改动）：** 独立插件通过 `SetGltfModifier` 注入水域变形逻辑。先实现并验证真实高程、碰撞与 LOD；这比直接深度 Fork 更可维护。

**备用方案（仅在 API 不足时）：** Fork 保留最小调用挂钩，其余逻辑放独立插件。可能涉及：

| Cesium 源码 | 备用改动内容 |
|---|---|
| `Public/Cesium3DTileset.h` / `Private/Cesium3DTileset.cpp` | 暴露/传递用户配置，或者扩展 GltfModifier 的钩子（确认现有接口确实不够时再做） |
| `Private/UnrealPrepareRendererResources.cpp` | 将不可变预处理数据快照传入 glTF 创建配置 |
| `Private/CreateGltfOptions.h` | 为模型创建流程添加可选处理对象 |
| `Private/CesiumGltfComponent.cpp` | 在 COPY POSITION 之后、Normal/Chaos 构建之前添加批量地形变形；同步 bounds/normal |

此备用方案的维护成本明显更高，不应一上来修改 Cesium 核心文件。

## 9. 分阶段实施顺序（精简验收）

1. **MVP A — 数据快照和水域判断：** GeoJSON Polygon、MultiPolygon 和带岛屿内环样本；两个场景点（海/岛）正确分类，且不会重复解析数据。
2. **MVP B — GltfModifier 接线：** 在指定 Terrain Tileset 上注册自定义 Modifier；验证单块瓦片能输出被修改的 glTF POSITION，普通 Tileset 未绑定时不受影响。
3. **MVP C — 高程下凹 + 碰撞：** 实现椭球高程计算和目标高度；打开 Cesium `CreatePhysicsMeshes`，检查一次可视位置与碰撞高度。
4. **MVP D — 岸线、内环、LOD：** 加入过渡和空间索引；验证外海、水域、岛屿、跨 LOD 接缝；必要时处理 normals、bounds 和 skirt。
5. **OceanView 集成：** JxOcean 统一水位参数；保留 Cartographic Polygon 作为编辑器调试/修改边界的可选工具，但不在正常运行中生成数百个 Actor。
6. **平台验证：** 先 UE 5.7 Win64 Editor，确认后再 Android；只做最小功能验证和一次典型场景加载检查，不做过度压测。

验收重点：
- 未启用时 Cesium 行为保持原样；
- 外环水域下凹，内环岛屿不下凹；
- 已经低于目标水底的区域不抬高；
- 碰撞与可视地形一致（开物理网格时）；
- LOD、远近镜头切换时无明显裂缝/错误剔除；
- GeoJSON 没有命中的瓦片不复制模型/不做昂贵的逐顶点水域检测。

## 10. 待验证但不阻塞设计的细节

- 检查 OceanView 实际安装的 Cesium v2.31.0 插件二进制、第三方头/库与 `GltfModifier` 链接可用性。
- 确认实际地形数据来源、几何编码和 node transform：`tileTransform` 到 ECEF 的完整变换链必须与渲染源码一致。
- 如果水域 GeoJSON 的高度是正高而不是椭球高程，需要独立高程基准转换。
- 验证 Cesium 外层 Tile BV 与 skirt 在下凹情况下的剔除行为；如果 API 不足，再考虑最小 Fork 补丁。
- 不通过修改 `cesium-native` 子模块上游代码作为第一版方案，优先在新插件中派生官方公开类。

本设计依据 **Cesium for Unreal v2.31.0** 与 **cesium-native v0.66.0** 的实际公开 API。所有代码片段和结构仅为实现指导，尚未编译运行。
