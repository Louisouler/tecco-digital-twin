# 生成路线：直接组装 Godot 还是经过 Blender

Research ticket：[#5](https://github.com/Louisouler/tecco-digital-twin/issues/5)（地图 [#1](https://github.com/Louisouler/tecco-digital-twin/issues/1)）。调研日期 2026-11-10。

## 结论

**推荐 Godot 直装，Blender 不放进每个项目的生成流程。** 具体做法：

1. 网站模型要求上游直接交 glTF（.glb）。如果只能拿到 three.js JSON，Twinwise 在无头浏览器里用 `ObjectLoader` 读进来，再用 `GLTFExporter` 转成 glb。
2. 用一个普通程序（Python 或 Node）做 glTF 后处理，按输入换材质和贴图。
3. 写出 Godot 项目。项目由三部分组成：**固定的模板脚本**、**生成的数据 JSON**（灯具、回路、场景、点位、探针位置）、**资产**（glb 和贴图）。场景树在运行时由模板脚本按数据装出来，这和参考仓库 `godot/main.gd` 的做法一样。
4. 运行 `godot --headless --import` 完成导入，再用 `godot --headless --script` 做加载校验。
5. 光照用 SDFGI，加上生成器自动布置的 ReflectionProbe。**不用 LightmapGI。**

Blender 只留作离线工具：给材质库或产品库烘焙程序化贴图。只有输入里出现 glTF 表达不了的程序化材质时，才加一个可选的 Blender 节点。

理由（下文逐条附出处）：

- **LightmapGI 只能在编辑器里烘焙**，没有脚本 API。烘焙需要 GPU。灯设成 Static 后运行时完全不能调。这三点和「命令行可重复」「场景要调光」冲突。
- **SDFGI 不需要烘焙**，在运行时计算，支持动态灯。ReflectionProbe 也是运行时采集。所以「自动放置 + 配置」本质上就是写几个节点属性，headless 下就能完成，不需要渲染。
- **Godot 4.7.2 用 `PackedScene.pack()` 保存 .tscn 时，每个节点的 `unique_id` 都是随机的**（实测，源码也确认），而且脚本没法设定这个值。如果每次都程序化保存 .tscn，产物就不会逐字节一致。改成固定模板 + 数据驱动，就绕开了这个问题。
- **Blender 中转本身是可重复的**（实测同机两次输出逐字节一致），但它带来的东西基本用不上：展厅级贴图、墙板、灯带都由输入给出（见 #1）；Godot 导入 glTF 不吃 Blender 的面光源和程序化节点；每个项目还多一个 300 MB 级的依赖和一份版本锁。参考仓库从 Blender 导出后，也要再用 Python 打补丁（透射材质改成 alpha），灯光还得另外导出 JSON。

## 1. Godot 4.x headless 能力与限制

版本：文档为 Godot 4.7 stable，源码为 tag `4.7.2-stable`，本机实测用 Godot 4.7.2.stable.official.ed1daf0bf（Apple M1，8 GB）。

### 1.1 命令行

| 能力 | 结论 | 出处 |
|---|---|---|
| `--headless` | 等同于 `--display-driver headless --audio-driver Dummy`，「Useful for servers and with `--script`」。 | https://docs.godotengine.org/en/stable/tutorials/editor/command_line_tutorial.html |
| headless 不渲染 | headless 显示服务器只提供 `dummy` 渲染驱动。所有需要真正渲染的事（烘焙、截图、探针采集）都不能在 headless 下做。 | https://github.com/godotengine/godot/blob/4.7.2-stable/servers/display/display_server_headless.h#L44-L48 |
| `--import` | 「Starts the editor, waits for any resources to be imported, and then quits. Implies `--editor` and `--quit`.」 | 同上命令行文档 |
| `--script` | 脚本必须继承 `SceneTree` 或 `MainLoop`；`--check-only` 只做语法检查。 | 同上命令行文档 |
| 导出 | `--export-release <preset> <path>`。在没有 GPU 的平台（如 CI）上，导出必须加 `--headless`。 | 同上命令行文档 |
| 录制或截图 | `--write-movie` 配合 `--fixed-fps` 可以按固定帧率写出画面，适合做渲染验收。需要 GPU，不能加 headless。 | 同上命令行文档 |

### 1.2 导入 glTF

- glTF 2.0 是官方推荐的格式：https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/importing_3d_scenes/available_formats.html
- 编辑器导入：导入参数写在 `*.glb.import` 里，导入结果缓存在 `.godot/imported/`。导入时可以挂 `EditorScenePostImport` 脚本（`_post_import(scene)`）做后处理，还能生成 LOD、阴影网格和 UV2：https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/importing_3d_scenes/import_configuration.html
- 运行时导入：`GLTFDocument.append_from_file()` 加 `generate_scene()`。文档说它「first-class support ... both in the editor and exported projects」，但也提醒，对项目自带的资源，运行时加载「less efficient and doesn't allow benefiting from Godot's resource handling」：https://docs.godotengine.org/en/stable/tutorials/io/runtime_file_loading_and_saving.html 、 https://docs.godotengine.org/en/stable/classes/class_gltfdocument.html
- **4.7.2 支持的 glTF 扩展**（源码中的 `supported_extensions` 列表）：`KHR_lights_punctual`、`KHR_materials_emissive_strength`、`KHR_materials_unlit`、`KHR_materials_pbrSpecularGlossiness`、`KHR_texture_transform`、`KHR_node_visibility`、`KHR_animation_pointer`、`GODOT_single_root`。另有扩展模块支持 `EXT_texture_webp`、`KHR_texture_basisu` 和物理相关扩展。**不支持** `KHR_materials_transmission`、`clearcoat`、`sheen`、`ior`、`volume`、`specular`，也不支持 Draco 或 meshopt 压缩：
  - https://github.com/godotengine/godot/blob/4.7.2-stable/modules/gltf/gltf_document.cpp#L6969-L6979
  - https://github.com/godotengine/godot/tree/4.7.2-stable/modules/gltf/extensions
  - 参考仓库的 `godot/import_web.py` 已经踩过这个坑：它把透射材质手工改成 alpha 混合的薄片。
- **实测**：参考仓库的 3 个 glb（共 15.5 MB，内嵌约 128 张 WebP）放进一个空项目，`godot --headless --import` 用时 24.7 s（M1），没有报错，贴图被自动解出到 `models/`。

### 1.3 程序化生成并保存 .tscn

- 写法：`PackedScene.pack(root)` 后接 `ResourceSaver.save()`。**只有 `owner` 设成根节点的子节点才会被保存**：https://docs.godotengine.org/en/stable/classes/class_packedscene.html
- 实测 `godot --headless --script build.gd` 能完成以下动作：实例化导入好的 glb，加 ReflectionProbe、WorldEnvironment（SDFGI）和 LightmapGI 节点，再保存 `main.tscn`。返回值都是 OK。
- **可重复性问题（实测并对照源码确认）**：
  - 同样的脚本跑两次，生成的 .tscn 只有每个节点的 `unique_id=` 不同。
  - 原因：`SceneState` 保存时遇到未分配的节点，会用 `ResourceUID::create_id()` 随机分配一个值：https://github.com/godotengine/godot/blob/4.7.2-stable/scene/resources/packed_scene.cpp#L1099-L1119
  - `Node::set_unique_scene_id` 没有绑定给脚本（`node.cpp` 里没有对应的 `bind_method`），所以脚本里无法设定：https://github.com/godotengine/godot/blob/4.7.2-stable/scene/main/node.cpp#L2117
  - 子资源 id 默认由「current date, time, and a random value」生成，但可以用 `resource_scene_unique_id` 指定（实测有效）：https://docs.godotengine.org/en/stable/classes/class_resource.html
  - .tscn 格式文档说明 `unique_id`「is only present in scenes saved with Godot 4.6 or later」，也就是说这个字段可以省略：https://docs.godotengine.org/en/stable/engine_details/file_formats/tscn.html
- **对策**（任选其一，推荐 a）：
  - (a) `main.tscn` 做成固定模板，只挂一个脚本。场景在运行时按数据 JSON 组装，参考仓库就是这样做的，它的 `main.tscn` 只有 151 字节。
  - (b) 生成器直接写 .tscn 文本，不写 `unique_id`。
  - (c) 保存后把 `unique_id=` 剥掉。
- `.godot/imported/*.scn` 两次全新导入的结果不是逐字节一致（实测）。它属于缓存，不算产物，比较产物时排除 `.godot/`。
- `.import` 里的 `uid` 在两次全新导入中保持一致（实测，仅此一例）。官方文档没有承诺这一点，列入风险。

### 1.4 自动布置 ReflectionProbe、配置 SDFGI、LightmapGI 烘焙

| 技术 | 要不要烘焙 | 能否 headless 自动化 | 动态灯 | 出处 |
|---|---|---|---|---|
| SDFGI | 不要。「fully automatic; it only needs to be enabled in the Environment resource」 | 能。headless 下只是写 Environment 属性 | 支持动态灯，但不支持动态遮挡物和动态自发光面；只能用 Forward+；默认要约 25 帧收敛；是最吃性能的 GI 之一 | https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/using_sdfgi.html 、 https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/introduction_to_global_illumination.html |
| ReflectionProbe | 不要。运行时采集（`UPDATE_ONCE` 在之后 6 帧内生成） | 能。位置、`size`、`box_projection` 由生成器按房间或展区算出 | 周围几何变化不会自动刷新，需要手动重新触发采集 | https://docs.godotengine.org/en/stable/classes/class_reflectionprobe.html |
| LightmapGI | 要 | **不能**。「Lightmap baking functionality is only available in the editor」，类上没有 `bake()` 方法（实测 `has_method("bake") == false`）；入门文档也写明「Not viable for procedurally generated levels」 | 灯设为 Static 后「will not be adjustable at all」；烘焙在 GPU 上进行；产物是 EXR 加 `.lmbake` | https://docs.godotengine.org/en/stable/classes/class_lightmapgi.html 、 https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/using_lightmap_gi.html |
| VoxelGI | 要（烘焙快） | 也是编辑器烘焙，未深入 | 墙和楼板太薄时漏光 | 入门 GI 文档 |

参考仓库实测过的坑（`godot/main.gd` 注释，Godot 4.7）：

- SDFGI 几乎不吸收聚光灯的反弹，所以参考仓库用成对的 omni 灯补反弹光。
- 切换场景后，SDFGI 会保留旧光照，需要关掉再开（`_restart_gi`），然后重新采集探针（`_refresh_probes`）。
- 玻璃要设 `GI_MODE_DISABLED`，否则会被 SDFGI 当成墙挡住日光。

这些都是运行时脚本逻辑，和生成路线无关，模板脚本可以直接沿用。

## 2. 经 Blender 中转（`blender -b --python`）

版本：Blender 5.2 LTS 手册，本机 Blender 5.2.2 LTS。

### 2.1 能力

- 命令行参数：`-b/--background`（无界面）、`-P/--python`、`--python-exit-code`（脚本抛异常时返回非零，便于流水线判断失败）、`--factory-startup`（不读用户的 startup.blend，避免本机配置混入）、`--threads`：https://docs.blender.org/manual/en/latest/advanced/command_line/arguments.html
- 布尔运算：Boolean 修改器提供 Exact 和 Float 两种求解器；「Only Manifold meshes are guaranteed to give proper results」：https://docs.blender.org/manual/en/latest/modeling/modifiers/generate/booleans.html
- 烘焙：烘焙用的是 Cycles，用途之一就是「Baking ... procedural textures」和「for export to game engines」。烘焙需要 UV 和目标图像节点，采样等参数沿用渲染设置：https://docs.blender.org/manual/en/latest/render/cycles/baking.html 。噪声种子可以固定（`Seed`）：https://docs.blender.org/manual/en/latest/render/cycles/render_settings/sampling.html

### 2.2 导出 glTF 的限制

出处：https://docs.blender.org/manual/en/latest/addons/scene_gltf2.html

- 导出器只识别 Metal/Rough PBR（Principled BSDF 加图像贴图）和 unlit 两类材质，「construct a glTF material based on the nodes it recognizes」。**程序化节点必须先烘焙成图**，参考仓库 `export_web.py` 就是这样烘焙地毯和天花的。
- 「Blender's Area light type and World lighting are not supported by glTF」。面光源和天空不会导出。参考仓库因此把灯光另外写进 `lights.json`。
- 导出器能写 `KHR_materials_transmission`、`KHR_draco_mesh_compression`、`EXT_meshopt_compression` 等扩展，但这些 Godot 4.7.2 都不吃（见 1.2）。所以从 Blender 导出时要避免用到它们，或者导出后再处理。

### 2.3 可重复性与速度（实测）

测试脚本：建一面墙，用 Exact 布尔挖一个门洞，Smart UV 展开，把 Noise 程序化材质烘焙成 256² 贴图，然后导出 glb。命令为 `--factory-startup`，用 CPU，`seed=0`。

- 同一台机器连跑两次，`room.glb` 和贴图的 sha1 完全相同。
- 单次构建加烘焙约 0.4–0.7 s，导出约 0.2 s；进程启动加退出约 1–2 s。
- 这只证明「同机、同版本、CPU」时可重复。官方没有承诺跨版本、跨平台或 GPU 烘焙也逐字节一致，所以要锁死 Blender 版本（参考仓库 README 已要求 5.2）。

### 2.4 对 Godot 导入的影响

- 好处：导出的 glb 是规范的 PBR 材质，Godot 直接能吃；Godot 的 UV2 可以在导入时生成（`meshes/light_baking`），不必在 Blender 里展开。
- 代价：
  - Blender 的灯光、面光源和场景数据要另外导出。
  - 透射、清漆这类材质，Godot 导入时会被丢掉，需要补丁。
  - 单位和坐标轴要换算（Z-up 转 Y-up，导出器负责）。
  - 每台生成机器都要装同一版本的 Blender。
- Godot 也能直接导入 `.blend`，前提是机器上装了 Blender：https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/importing_3d_scenes/available_formats.html 。这会把 Blender 依赖带进 Godot 项目本身，不推荐。

## 3. three.js 网站模型怎样变成 glTF

版本：three.js r187（2026-10-30 发布）的源码和文档。

| 路径 | 做法 | 风险 | 出处 |
|---|---|---|---|
| A. 上游直接给 glTF（推荐） | 网站本来就在 three.js 里，可以调用 `GLTFExporter.parseAsync(scene, {binary:true})` 导出 glb，作为交付物 | 最低。要约定好单位、坐标轴、楼层和节点命名 | https://threejs.org/docs/pages/GLTFExporter.html |
| B. 只给 three.js JSON，Twinwise 自己转 | 在无头浏览器里用 `ObjectLoader` 读入，再用 `GLTFExporter` 导出 glb | 见下文风险 | https://github.com/mrdoob/three.js/blob/r187/src/loaders/ObjectLoader.js |
| C. 只给 JSON，自写转换器 | 直接解析 JSON 里的 BufferGeometry 和材质，写成 glTF | 要自己覆盖每一种材质和几何类型，工作量和维护成本最高 | https://github.com/mrdoob/three.js/wiki/JSON-Object-Scene-format-4 （这份 wiki 自己标注「Work in progress」） |

只给 three.js JSON 时的风险：

- **格式没有规范**：JSON Object/Scene 格式本质上是 three.js 的序列化格式，只有一份标注「Work in progress」的 wiki。`ObjectLoader` 会随 three.js 版本变化，所以需要上游写明用的是哪个 three.js 版本。
- **有些内容转不过去**（GLTFExporter r187 源码中的告警）：
  - `ShaderMaterial not supported`（L1648-1650）。网站用了自定义着色器的部分会丢失。
  - 非 `MeshStandardMaterial`/`MeshBasicMaterial` 的材质会被近似（L1660-1662）。
  - 灯光只支持方向光、点光源和聚光灯（L2869），`RectAreaLight`、`HemisphereLight` 会丢。光照衰减也可能丢（L2910）。
  - 出处：https://github.com/mrdoob/three.js/blob/r187/examples/jsm/exporters/GLTFExporter.js
- **导出贴图需要 canvas**：导出器靠 `OffscreenCanvas` 或 `document.createElement('canvas')` 编码图片（L535-L541）。所以在纯 Node 环境里，带贴图的导出会失败，只能放在无头浏览器里跑。这是根据源码推断的，没有实测。
- **转换结果仍然要经过 1.2 的过滤**：three.js 能导出 transmission、clearcoat 等扩展，Godot 不认，还是要做后处理。
- 参考仓库的「网页模型」本身就是 glTF（`export_web.py` 把 glb 转成内嵌 glTF JSON，网页再加载），不是 three.js JSON。这说明在网页端交付 glTF 是可行的。

## 4. 候选路线取舍

| 路线 | 可重复性 | 展厅级效果 | 复杂度 | 风险 | 出处 |
|---|---|---|---|---|---|
| **R1 Godot 直装、数据驱动（推荐）**：glTF 后处理 → 固定模板脚本 + 数据 JSON + 资产 → `godot --headless --import` → 运行时组装，用 SDFGI + 自动反射探针 | 高。源文件可以逐字节一致；`.godot/` 是缓存，不纳入比较 | 与参考仓库 Godot 版相同：SDFGI 反弹光、盒投影探针、AgX、SSAO。程序化材质要求输入给贴图 | 低到中。只有一个运行时依赖（Godot），模板脚本可以从 `main.gd` 抽出来 | SDFGI 性能和漏光；4.7 下 SDFGI 几乎不吃聚光灯反弹（参考仓库已有绕法）；`.import` uid 稳定性没有文档保证 | §1；参考仓库 `godot/main.gd`、`main.tscn` |
| R2 Godot 直装、每次生成 .tscn（`PackedScene.pack`） | 中。`unique_id` 每次随机，需要剥掉或自己写 .tscn | 同 R1 | 中 | 每次 Godot 升级都要回归 .tscn 格式；生成器要在 Godot 进程里写 | §1.3，packed_scene.cpp#L1099 |
| R3 每个项目都经过 Blender（参考仓库流程）：`blender -b` 建模、布尔、烘焙 → glb → Godot | 同机同版本下高（实测）；跨平台、GPU 烘焙没有保证 | 只多出「程序化材质烘焙」和「Cycles 照度验算」；面光源和天空导不出去，最终画面仍由 Godot 决定 | 高。两个运行时依赖，灯光和场景要另走 JSON，导出后要打补丁 | Blender 版本锁死；Blender 能写的扩展 Godot 不吃；流水线更长、更慢 | §2；`export_web.py`、`import_web.py` |
| R4 LightmapGI 烘焙 | 低。只能在编辑器烘焙，没有 CLI 或脚本 API | 静态间接光质量最高（最多 16 次反弹） | 高 | 灯设为 Static 后不能调光，和场景切换冲突；需要 GPU；需要 UV2 | §1.4 |
| **混合（推荐落地方式）**：主线用 R1；Blender 只作为离线的「材质库或产品库烘焙工具」，或作为一个默认关闭、只在输入含程序化材质时启用的可选节点 | 主线同 R1 | 同 R1，并能接住程序化材质 | 主线低，可选节点中 | 可选节点要锁 Blender 版本 | §2.1、§2.3 |

## 5. 生成流程的节点划分（供「dashboard 的节点与状态」ticket 使用）

每个节点只读上游节点的产物，把结果写进项目文件夹下各自的目录，失败时输出明确的错误。

1. **读取输入**：读取并校验 schema，包括网站模型、平面图、规则、点位、空间数据和配置覆盖。
2. **网站模型 → glTF**：上游已经给 glb 时只做校验；只给 three.js JSON 时，在无头浏览器里转换。
3. **几何归一化**：统一单位和坐标轴，拆分楼层，按配置覆盖几何，过滤 Godot 不支持的扩展（transmission 改成 alpha，参照 `import_web.py`）。
4. **材质与贴图装配**：按输入分配定制贴图、墙板和灯带发光面；从产品库取产品 glb 和款式。
5. **点位与家具摆放**：根据设备点位和家具点位生成实例清单（数据 JSON）。
6. **灯光与场景数据**：把流明和色温换算成 Godot 能量，写出回路、场景、时段和天气数据（参照 `lights.json`）。
7. **渲染设置**：确定 SDFGI 和 Environment 参数、反射探针的位置和尺寸、画质档。
8. **写出 Godot 项目**：模板脚本、`main.tscn`、`project.godot`、数据和资产。
9. **Godot 导入**：运行 `godot --headless --import`。
10. **校验**：用 `godot --headless --script` 加载主场景，检查资源缺失、节点和点位数量、错误日志；对源文件做哈希，用来比较可重复性。
11. **渲染验收（可选，需要 GPU）**：不加 headless，从固定视角截图（`--write-movie` 或抓取视口），和基准图对比。
12. **打包或导出（可选）**：`--export-release`。

可选节点 **B. Blender 程序化材质烘焙**：插在第 4 步之前，默认关闭。

## 6. 未决与风险

- `.import` 的 `uid` 和导出结果是否跨机器稳定，文档没有承诺。下一步应该在第二台机器（或 CI）上跑一遍，对比源文件哈希。
- 第 2 步依赖上游的交付格式，需要和上游确认能不能直接交 glb（关联 #1 的「网站模型出错时配置能覆盖哪些几何」）。
- SDFGI 在多层户型里的漏光和性能（楼板厚度、`y_scale`、`use_occlusion`）没有实测，只有参考仓库单层展厅的经验。
- 参考资料说明：scratchpad 里的 `ref/showroom-space-3d` 只有 `godot/project.godot`，其余文件为空，所以实际读的是本机完整副本 `/Users/louis/orca/showroom-space-3d`（同一仓库，只读）。
