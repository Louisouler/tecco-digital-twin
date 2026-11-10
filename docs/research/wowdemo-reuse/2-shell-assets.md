# 调研：showtime-wowdemo 的外壳与资产生成流程，以及"自动 vs 手工"的比例

调研对象：showtime-wowdemo `feat/house-v1`（浅克隆，只有一个提交 `394e3f6`），和 showroom-space-3d-fortis 的 `projects/fortis-double-bay`。两边都只读。

出处写法：
- `W:` = `/private/tmp/claude-501/-Users-louis-orca-tecco-digital-twin/d2b292f3-bc72-410a-92f8-853d779fde44/scratchpad/wow/showtime-wowdemo/`
- `F:` = `/Users/louis/orca/showroom-space-3d-fortis/projects/fortis-double-bay/`

---

## 0. 结论先行

1. **wowdemo 只有建筑外壳和灯光"骨架"是自动生成的，其余几乎全是手工。** 墙、楼板、屋顶、门窗、楼梯、踢脚线由 TS 脚本从住宅契约生成。全部家具、外立面、场地、景观、窗帘、灯具模型、材质库，都是 Mac 上的 Blender agent 逐间手工建的。灯光先按通用规则自动布置，之后又叠了约 20 个针对这套房子的补丁脚本（约 1,100 行），外加两份"人工审计清单"（356 条灯带、127 个面）。按工作量估：**自动约 15–20%，手工或针对此房子约 80–85%**。
2. **运行时的组装方式和之前的推荐一致**：固定的 `main.tscn` 只有一个节点加一个脚本；外壳 GLB 运行时读入，室内 GLB 由 headless 导入；SDFGI 加每间一个盒投影反射探针；没有 LightmapGI 或 VoxelGI，也没有 Godot 内的离线烘焙。用到 Blender 的有三处：内容创作（相当于 Twinwise 的"上游"）、网站用的离线 Cycles 渲染（和 Godot 无关）、几个可以替换的小工具。这些都**不构成让 Twinwise 改走 Blender 的理由**。但有一条证据要求**补强渲染设置**：只靠 SDFGI 撑不起灯带、灯槽的展厅效果。wowdemo 最后靠"AreaLight3D + 按房间分接收层 + 漫反射回光代理"才达标，而且至今仍有"白天天花偏暗"的未解问题。这套做法要写成 Twinwise 的通用规则，不能再靠每间手调。
3. **可以复用的**：外壳生成器（作兜底或简单体块）、契约校验规则、`lighting.gd` 的通用分层规则、"生成 JSON 清单 + 运行时校验后应用"这种补丁形态，以及一组验收工具（悬空检查、截图拼版、闪烁和行走检查、发布 MANIFEST）。**不能复用的**：所有 `*_finish.gd`、`lighting_repairs.gd`、`missing_fixtures.gd`、`showroom_data.gd`、`main.gd` 里写死的网格名和坐标。
4. **fortis-double-bay 是一次把 wowdemo 变通用的实测**：`main.gd` 不写死户型，所有数据放在 `data/*.json`；有节点命名契约、家具组库、"自动给初值，再由输入覆盖"的模式，还有射线验收脚本。这几样都值得借鉴，它也反过来证明推荐路线可行。

---

## 1. wowdemo 各类资产是怎么来的

### 1.1 总览表

| 资产 | 来源 | 自动 / 手工 | 出处 |
|---|---|---|---|
| 住宅契约 `house.json`（房间多边形、边界、门窗、设备实体、场景） | 主控按私有参考设计**手工数字化**，经 24 条规则校验后构建 | 数据靠手工，校验和构建是自动的 | `W:contracts/house/README.md:3,14-15`；`W:docs/decisions/0003-house-layout-from-reference-skeleton.md:13-36`；`W:docs/DATA-SYNC.md:16,24` |
| 建筑外壳 `showroom/assets/house-shell.glb` | `npm run shell:build`：`buildShell(house)` → `shellToGlb`（@gltf-transform/core） | **全自动** | `W:shell/tools/build.ts:4-6`；`W:package.json:10`；`W:shell/src/gltf.ts:1`；`W:docs/DATA-SYNC.md:17` |
| 外壳贴图 `assets/shell_paint/*` | Mac 上用 Blender 脚本处理 Poly Haven CC0 扫描图；运行时三平面投影 | 贴图是手工做的，套用是自动的 | `W:showroom/assets/shell_paint/README.md`（Sources 段）；`W:showroom/scripts/main.gd:707`（`_dress_shell`）、`:660`（`_shell_paint`） |
| 外壳补丁 `assets/shell_patch/gf-frames-frame.glb` | 手工修过的一份外壳部件，运行时替换掉原件 | 手工 | `W:showroom/scripts/main.gd:3430-3450` |
| 室内家具 `showroom/interiors/<房间>/<房间>.glb`（45 个，单个约 23–25 MB） | Mac 上的 Blender 5.1 agent 逐间建模，Syncthing 同步，Git LFS 入库。GLB 的 generator 字段都是 `Khronos glTF Blender I/O v5.1.20` | **全手工** | `W:docs/tasks/L3-02-interiors-furniture-materials.md:10,16-39`；`W:docs/tasks/L3-14-house-v1-build-out.md:15-54`；`W:showroom/tools/sync-interiors.sh:2,9`；`W:.gitattributes:2` |
| 外立面、屋顶、场地、景观、泳池盖、窗帘（`showroom/outdoor/*.glb`，9 个，合计约 250 MB） | 同上，由 Mac 的另一条 worker 线完成 | **全手工** | `W:docs/tasks/L3-14-house-v1-build-out.md:56-113`；`W:showroom/tools/sync-interiors.sh:13-17` |
| 楼梯、入户门、推拉门 | 外壳先自动生成一份；Mac 用 Blender 重建后，运行时把外壳里的那份隐藏，只保留碰撞 | 自动版不够用，改为手工替换 | `W:showroom/scripts/main.gd:3288-3297`（`_mac_doors`）、`:3430-3450`（`_mac_stairs`） |
| 材质库 `M_ref_*` | Mac 上按图纸饰面代号手工建（TIL1/ST1/VN1…） | 手工 | `W:docs/tasks/L3-13-reference-interiors-kit.md:12-16` |
| 灯具模型（`<房间>__lamp__<层>__<n>`） | Blender 里手工建，按命名约定交付 | 手工 | `W:docs/tasks/L3-14-house-v1-build-out.md:44`、`:106` |
| 灯光（Light3D 节点） | `lighting.gd` 运行时按契约里的灯光实体和灯具名分四层自动布置，自动补灯槽和暗区筒灯；之后跑约 20 个专用补丁 | 通用规则，加大量手工补丁 | `W:showroom/scripts/lighting.gd:1-8,22-30,1138,1202,1333`；`:514-575`（补丁调用链） |
| 设备点位 `data/device-layout.json`（107 个） | 参考图纸的点位私下数字化，再用 `snap-controls.ts`、`place-bedside.py` 按规则挪位 | 数据手工，挪位半自动 | `W:showroom/data/device-layout.json`（`source`/`note` 字段）；`W:tools/snap-controls.ts:1-8`；`W:tools/place-bedside.py:1-10` |
| 产品模型 `assets/products3d/` | 设计师的 `.blend` 经 Blender headless 导出，问题很多（贴图丢失、玻璃变白、朝向不一致） | 半自动导出，之后人工修 | `W:docs/reports/L3-01-blender-pipeline.md:4,107-120` |
| 视角 `data/viewpoints.json` | 66 个视角里有 23 个由 `_best_views` 自动打分选出，35 个由 Joe 按 F7 手工固定 | 约一半手工 | `W:showroom/scripts/main.gd:6562-6577,7103`；数据统计见 §1.4 |
| 导览 `data/tours/*.json`、`showroom_data.gd` VIEWS | 手写 | 手工 | `W:showroom/scripts/showroom_data.gd:2-8` |
| `stair-steps.json` | 用 Blender 脚本从外壳 GLB 里抽出踏步面 | 自动，但绕道 Blender | `W:tools/blender/stair_steps.py:1-10` |
| `surface-manifest.json`（127 条）、`strip-area-manifest.json`（356 条） | Codex agent 对运行时做审计后冻结成清单 | 半自动，只对这套房子有效 | `W:docs/reports/codex-material-lighting-2026-10-05/README.md:10`；`W:docs/reports/codex-strip-audit-2026-10-05/README.md:11,35`；`W:docs/reports/codex-strip-audit-2026-10-05/reproduce/build_manifest.py:1-10` |
| `mesh-details` / `fluted-details` / `body-details.json`（共 12.6 MB） | 离线算好的倒角、凹槽网格顶点，运行时替换原网格 | 半自动，只对这套房子有效 | `W:showroom/scripts/model_details.gd:5-34`；`W:docs/reports/codex-visual-improvements-2026-10-04/README.md:7-12` |

### 1.2 外壳生成器（`shell/`）：通用、确定、可测

- 输入是契约的 `House`（房间多边形按墙体中心线绘制、边界五种类型、楼层标高），输出是 slab / roof / wall（墙上开洞）/ glass / frame / rail / stair / skylight / door / skirting（`W:shell/src/shell.ts:5-22`）。
- 规则全部从几何推导，代码里**没有一个写死的房间 ID**：
  - 正交多边形切成矩形（`:31-39`）
  - 上层没有覆盖的地方封屋顶，天窗处挖洞（`:93-125`）
  - 门扇开向和贴墙方式由服务的房间决定（`:127-158`）
  - 通高空间墙体加高（`:200-245`）
  - 螺旋楼梯（`:261-269`）
  - 墙角互相延伸（`:305-320`）
  - 踢脚线遇外角连续（`:324-349`）
- Joe 的反馈都被写成了**通用规则**，没有变成针对某个房间的补丁。例如：
  - 楼板下沉 12 mm，防止和地面饰面闪烁（`:24-26`）
  - 墙角缝（`:305`）
  - 踢脚线遇外角断开（`:334`）
- 材质只有 18 种纯色加粗糙度（`W:shell/src/gltf.ts:21-29`）。地面材质按房间用途或名字猜（`W:shell/src/shell.ts:41-48`，例如名字里有 pool 就当水面）。这部分要改成读输入。
- 有测试：`W:shell/tests/shell.test.ts`（22 KB）。发布流程是"测试不过就不发"（`W:tools/publish-house.sh:16-21`）。
- 局限：楼梯只会做螺旋一种，而且展厅级效果最后是 Mac 重建的（`W:showroom/scripts/main.gd:3430-3435`）。外立面由 `facade.glb` 盖住；景观、场地交付后，外壳自带的户外地面被隐藏（`W:showroom/scripts/main.gd:1120-1125`）。所以这个生成器的定位是**结构正确的室内壳体**，不是展厅级成品。

### 1.3 `showroom/scripts` 里的补丁：通用规则还是针对这套房子？

判断方法：脚本里有没有写死网格名、房间 ID、世界坐标或固定数量（例如断言 `selected.size()!=13`）。

**A. 针对这套房子的手工补丁（不能复用），共约 1,077 行：**

| 脚本 | 写死了什么 | 出处 |
|---|---|---|
| `lighting_repairs.gd` | 6 个具体灯具名加回路 ID；12 个房间 ID（SOFT_ROOMS）；GF-11 一个世界坐标 `(12.95,.8,-16.55)`；`GF-11__lamp__strip__cove_auto_1` | `:4-12,67,89` |
| `missing_fixtures.gd` | 3 条回路；用硬编码坐标（例如 `Vector3(16.18,6.085,z)`、`16.85,.13,-31.98`）现场"安装"壁挂灯和庭院灯笼 | `:3-7,25-50` |
| `lighting_finish.gd` | FF-11 → GF-X1 改回路，坐标 `(18.7,2,-21.521)`；GF-N1 | `:5-23` |
| `strip_finish.gd` | `GF-2__lamp__strip__01..06`，第 4、5 条有专门的偏移 | `:7-14` |
| `garage_*_finish.gd`、`garage_uplights.gd`、`garage_floor_strips.gd` | B1-1 回路 ID，`CENTRES` 坐标表，`B1-1__lamp__strip__%02d` | `garage_uplights.gd:2-3`；`garage_floor_strips.gd:3,13` |
| `gallery_area_finish.gd`、`gallery_strip_finish.gd` | FF-8，"必须正好 13 个光源"，回路必须等于 `["light.ff_corridor_dali_37"]` | `gallery_strip_finish.gd:7-16` |
| `cinema_seat_finish.gd`、`cinema_screen_finish.gd` | B1-5 座椅名、`B1-5__screen__canvas` | `cinema_seat_finish.gd:20`；`cinema_screen_finish.gd:9-13` |
| `cabinet_finish.gd` | 3 个玻璃柜门网格名 | `:2` |
| `interior_surface_finish.gd` | 只处理 FF-1、GF-11、FF-8 三个房间 | `:9` |
| `bath_details.gd` | 5 件摆件的绝对坐标 | `:2-7` |
| `sauna_wood_finish.gd`、`covered_car_finish.gd`、`vehicle_finish.gd` | `B1-8__sauna__benches_teak`、`B1-1__garage__car_covered_*`、`suv_paint` | 见各文件开头 |
| `room_lighting_refinement.gd` | 一半是通用规则（灯槽内缩 55 mm），一半写死 B1-1、FF-8 的坐标 | `:5,19,82-110` |
| `main.gd` 里内嵌的补丁 | `_detail_fixes`（花盆、显示器弯曲、墙顶灯带，`:1151-1185`）；`SHOWER_MOVES`（`:6530`）；户外按名字隐藏逆变器、车库门、草边（`:1124-1133`）；`GATES`、`SASHES` 实体到网格的映射（`:2015-2024`） | 同列 |

这些补丁的共同点：每个都"保持回路 ID 不变，失败就拒绝执行"，并且对应一份 Codex 报告（`W:docs/reports/codex-*`）。它们是**交付前几天逐间修出来的效果**，不是规则。例如 kitchen raft 和 master cove 各是一个单独的高度例外，后来才"泛化"到 16 个灯具（`W:docs/reports/codex-kitchen-raft-2026-10-05/README.md:5`；`W:docs/reports/codex-master-cove-2026-10-05/README.md:3`；`W:docs/reports/codex-wholehouse-slot-2026-10-05/README.md:5`）。

**B. 通用的执行引擎，但数据只对这套房子有效（引擎可复用，数据要重新生成），共约 479 行：**

- `wholehouse_strip_areas.gd`：读 `strip-area-manifest.json`，把点光源换成 AreaLight3D 并加漫反射回光。清单"在几何或灯光变化后失效，fail closed"（`W:docs/reports/codex-strip-audit-2026-10-05/README.md:35`）。生成器 `build_manifest.py` 自己也写死了 `soft` 房间集合（`reproduce/build_manifest.py` 约第 40 行）。
- `wholehouse_surface_finish.gd`：读 `surface-manifest.json`，按"网格名 + 面序号 + 材质名"降低法线强度（`:7-31`）。
- `model_details.gd`、`fluted_details.gd`、`body_details.gd`：读预先算好的顶点 JSON，替换网格，并校验包围盒（`model_details.gd:5-34`）。
- `upholstery_refine.gd`：通用的细分算法（`:14-36`）。

**C. 通用运行时（可复用的骨架）：**

- `lighting.gd`（1,467 行）：
  - 回路按实体名归入 downlight / strip / linear / pendant 四层（`:22-30,32-60`）
  - 灯具网格按 `<房间>__lamp__<层>__` 发光
  - 自动灯槽（`_coves`，`:1202`）
  - 暗区补筒灯（`_cover_gaps`、`coverage`，`:1138,1333`）
  - 自检（`audit`，`:805`）
- `main.gd` 的 `_load_interiors`（`:1048-1150`）：
  - 按目录约定加载
  - 同名材质去重
  - 地毯抬 6 mm 防闪烁
  - 契约里没有风扇的房间隐藏吊扇（`:1096-1106`）
- `_add_reflection_probes`：按房间多边形自动放盒投影探针（`:930-960`）
- `_build_environment`：AgX、SDFGI、SSAO、SSIL、SSR（`:589-650`）
- 其他：`occluders.gd`、`mirrors.gd`、`reach.gd`、`devices3d.gd`、`states3d.gd` 等

注意：`main.gd` 有 7,235 行，含 147 处 "Joe" 注释、48 个写死的房间或网格名。它是通用骨架和本房补丁的混合体，复用时要逐段剥离。

### 1.4 比例估算

| 维度 | 自动 | 手工或针对此房子 | 依据 |
|---|---|---|---|
| 资产体量（GLB 字节数） | `house-shell.glb` 约 1.2 MB | interiors 45 个（约 1 GB 级）+ outdoor 9 个（约 250 MB）+ products3d | `ls` 统计；`W:showroom/assets/house-shell.glb` |
| 任务卡（工时） | 外壳和契约由主控一条线完成 | L3-01 到 L3-31 约 25 张 Blender 任务卡（`W:docs/tasks/L3-*`），外加 9 份 Codex 补丁报告 | `W:docs/tasks/`、`W:docs/reports/codex-*` |
| 运行时脚本 | 通用骨架约 3.7k 行（lighting、devices3d、states3d、occluders、mirrors、reach），加 `main.gd` 的大部分 | 专用补丁约 1.1k 行；清单引擎约 0.5k 行，配本房数据 12.6 MB；`main.gd` 内嵌补丁 | 本次 `wc -l` 统计（§1.3） |
| 数据 | 视角中 23 个自动打分；设备的挪位规则 | 设备 107 个手工数字化；视角中 35 个手工；导览全部手写；清单 483 条 | §1.1 |

**综合判断：按工作量，自动约 15–20%，手工约 80–85%。** 自动的部分集中在"结构"（外壳、灯光分层骨架）；手工的部分集中在"效果"（家具、材质、灯具、外立面、逐间调灯）。这和 Twinwise 的分工正好对应：效果类内容由**上游写进输入**（`gh issue 1` 已定的约束："定制贴图、灯带、墙板都写进输入；Twinwise 负责组装和渲染设置"）。wowdemo 的手工补丁对 Twinwise 来说，就是"输入不够完整"或"渲染规则不够通用"的清单。

---

## 2. 这条路线和"不经过 Blender、Godot 运行时组装"的推荐是否一致

### 2.1 一致的部分（Godot 端）

- **固定模板场景**：`main.tscn` 只有一个 `Node3D` 加 `main.gd`（`W:showroom/main.tscn:1-6`）。全部内容由脚本在运行时组装（`W:showroom/scripts/main.gd:150-185` 的 `_ready` 顺序：环境 → 外壳 → 外壳贴图 → 室内 → 细节修补 → 碰撞 → 灯光）。
- **外壳在运行时读 GLB**：用 `GLTFDocument.append_from_buffer`，不走导入（`W:showroom/scripts/main.gd:690-700`）。室内和户外 GLB 用 Godot **headless 导入**（`W:showroom/tools/sync-interiors.sh:18-24`），导入默认值是 VRAM 压缩（`W:showroom/project.godot` 的 `[importer_defaults]`）。
- **光照**：
  - `sdfgi_enabled = true` 加 SSAO、SSIL、SSR（`W:showroom/scripts/main.gd:605-650`）
  - 每个房间一个 `box_projection`、`interior` 反射探针（`:930-960`）
  - 全仓**没有 LightmapGI 或 VoxelGI 节点**；`bake` 只出现在 `light_bake_mode=BAKE_DISABLED/DYNAMIC`（grep 结果）
  - ADR-0001 选 Godot 的理由之一就是"场景和脚本全是文本，命令行可驱动"（`W:docs/decisions/0001-wowdemo-technical-route.md:35`）

### 2.2 用到 Blender 或离线渲染的地方，以及各自对 Twinwise 意味着什么

| 用途 | 出处 | 对 Twinwise 的含义 |
|---|---|---|
| ADR-0001 第 5 条："Blender（Mac）是唯一的设计来源，导出 glTF 进 Godot"；规格 §6 写明动画也在 Blender 里做 | `W:docs/decisions/0001-wowdemo-technical-route.md:19`；`W:docs/specs/2026-09-25-wowdemo-architecture.md:177-180` | 这是**内容创作**，对应 Twinwise 的"上游"和"产品库"。Twinwise 只收 glb，不需要自带 Blender。只是"窗帘用形态键 `open` 做开合"（`L3-14:60`）这类约定要写进输入规格 |
| `tools/showroom-render/*`（build_room、build_living、build_dollhouse、clips、turntable、autoexpose、site_materials） | `W:tools/showroom-render/build_room.py:1-10`；`autoexpose.py:1-7` | 给**公司网站**做的 Cycles 预渲染片段和 turntable（Joe 10-14），和 Godot 仿真无关。**不是**让仿真改走离线渲染的证据 |
| `tools/blender/stair_steps.py` | `:1-10` | 从外壳 GLB 抽踏步。`shell.ts` 已经算出每级踏步（`W:shell/src/shell.ts:261-269`），可以直接写出 JSON，不需要 Blender |
| `tools/blender/sofa_clip.py` | `:1-4` | 人物动画片段，产品库或资产侧的事 |
| 产品模型的 Blender 导出 | `W:docs/reports/L3-01-blender-pipeline.md:4,116-120` | 证据指向**产品库交付规范**（只用 Principled、贴图打包、统一朝向和单位、一个款式一个文件），不是让 Twinwise 去跑 Blender |

### 2.3 有没有证据要求改变路线

**没有证据要求加入 Blender 或离线烘焙。但有三条证据要求补强"SDFGI + 自动反射探针"这条推荐：**

1. **SDFGI 不处理动态自发光**，而灯带和灯槽恰恰是展厅效果的重点。所以 wowdemo 把灯带全部设成 `BAKE_DISABLED`（`W:docs/reports/codex-material-lighting-2026-10-04/RESEARCH.md:7`；`W:docs/reports/2026-09-28-lighting-perf.md:25`）。之后靠 AreaLight3D、12 个可复用的房间接收层位、"漫反射回光"代理补光，才去掉点光源的光斑和串光（`W:docs/reports/codex-strip-audit-2026-10-05/README.md:11-17`；`W:docs/reports/codex-material-lighting-2026-10-05/README.md:7-8`）。**Twinwise 要把"灯带网格 → 矩形 AreaLight3D + 回光 + 房间接收层"做成生成规则**，不能逐间审计。
2. **白天天花偏暗至今未解决**："daylight electric-lights-off captures still show dark ceilings … must not be described as a completed daylight solution"（`W:docs/reports/codex-ceiling-return-2026-10-05/README.md:11`）。规格当初的风险预案是"重点房间优先 VoxelGI/LightmapGI"（`W:docs/specs/2026-09-25-wowdemo-architecture.md:270`），一直没有实施。调研里引用的 ArchViz 例子是"在 Blender 里烘焙光照贴图"，但同一处也指出：把可开关的人工光永久烘焙，会和实时调光冲突，不能照搬（`RESEARCH.md:13`）。所以就算烘焙，也只能烘日光或环境光这部分。建议 Twinwise 的规格把"白天室内天花亮度"列为验收项，并预留一个**可选**的 Godot 内烘焙选项（LightmapGI 或 VoxelGI，headless 可跑）作为退路。它仍然在"不经过 Blender"的范围内，但会和"同样输入产物一致"有张力（烘焙结果的确定性要验证），这一条需要做实验确认。
3. **性能靠一组规则才达标**：投影阴影只给离视点最近的 12 个、光源超出作用范围就隐藏、灯带点间距 1.1 m、FSR2 动态分辨率（`W:docs/reports/2026-09-28-lighting-perf.md:24-28`）。客厅这类通高空间仍是最慢的，全开灯时浴室入口约 46 fps（`W:docs/reports/codex-lighting-performance-2026-10-05/README.md:13`）。这些规则要原样写进 Twinwise 的渲染模板。

另外，自动外壳里的楼梯和门最后都被 Mac 的手工模型替换了（`W:showroom/scripts/main.gd:3288-3297,3430-3450`）。这说明"程序生成的建筑构件"达不到展厅级。对 Twinwise 来说，楼梯、门等构件要么由网站模型或上游带来，要么从产品库按点位摆放。

---

## 3. 能复用到 Twinwise 的工具和脚本清单

### 3.1 可以复用（要改造）

| 项 | 用途 | 要改什么 |
|---|---|---|
| `W:shell/src/shell.ts`、`gltf.ts`、`tools/build.ts` 及测试 | 从空间数据生成室内壳体 GLB；可作网站模型缺失或出错时的兜底，或"外墙只做简单体块" | ① 输入改为上游空间数据的适配层，不再读 `house.json`；② 常量改成配置（墙厚、门高等，`:27`）；③ 地面材质不再按名字猜（`:41-48`），改读输入；④ 楼梯从"只有螺旋"改成按输入类型生成，或改成"只出碰撞体"；⑤ 先定清楚它和网站模型谁是几何真相源 |
| `W:contracts/house/src/rules.ts`、`geometry.ts`（24 条规则，含 `vertical.*` 楼上墙落在楼下墙或梁上） | 输入校验节点：校验不过就不生成 | 规则 ID 和错误信息接到 dashboard；房间命名、实体命名相关规则换成 Twinwise 的 ID 规则 |
| `W:showroom/scripts/lighting.gd` 通用部分 | 按回路分层、`__lamp__` 命名约定发光、自动灯槽、暗区补灯、`audit` | 删掉 `:514-575` 的补丁调用链；分层规则 `layer_of_name`（`:22-30`）改为由输入显式给出层，按名字猜只作兜底；补上 §2.3 的 AreaLight3D 灯带规则 |
| `wholehouse_strip_areas.gd` 和 `build_manifest.py` 的**模式** | "生成器产出确定的 JSON 清单 → 运行时先校验身份，失败就整体拒绝，通过才应用" | 清单改由 Twinwise 的一个节点从输入生成，不再由人审计冻结；去掉写死的 `soft` 集合；fail-closed 校验保留 |
| `wholehouse_surface_finish.gd` | 按"网格 + 面 + 材质名"覆盖材质参数 | 变成输入里的"材质覆盖"段，例如 `{mesh, surface, material, normal_scale…}`，由上游填 |
| `main.gd`：`_load_interiors`、`_dress_shell`、`_shell_paint`、`_add_reflection_probes`、`_build_environment`、`_set_visibility_ranges` | 模板脚本的组装骨架 | 抽成独立模块；户外按名字隐藏东西、吊扇按契约显隐这类规则改成输入里的显式开关；`_detail_fixes`、`SHOWER_MOVES`、`GATES`、`SASHES` 删除，实体到网格的绑定改由输入的点位表给出 |
| `occluders.gd`、`mirrors.gd`、`reach.gd`、`devices3d.gd`、`states3d.gd`、`heatmap.gd` | 遮挡、平面镜、可达性、设备和状态表现 | 逐个检查有没有写死 ID（统计显示这几个文件里 0–6 处）；`devices3d.gd` 的型号表改读产品库 |
| `_best_views`（`main.gd:7103`）和 F7 手工固定（`:6562-6577`） | 自动选视角，加人工固定 | F7 现在直接写产物数据，违反"只改输入"。要改成写回**输入文件**的视角覆盖段 |
| `W:tools/float-check.py` | 每个室内 GLB 的悬空件检查 | 直接复用为验收节点，`HANG` 白名单改为配置 |
| `W:tools/review-shots.py`、`showroom/tools/capture.sh` | 每个房间对角四视图截图拼版 | 作为生成后的"出图"节点，结果给 dashboard |
| `showroom/tools/flicker-check.sh`、`walk-check.sh`、`perf-tour.sh`、`tools/check-scripts.sh` | 闪烁、行走、性能、脚本加载检查 | 去掉 Windows 和 WSL 路径，参数化 Godot 路径 |
| `showroom/tools/tex-compress.py`、`project.godot` 的 `[importer_defaults]` | 贴图 VRAM 压缩 | 直接放进模板 |
| `W:tools/publish-house.sh` 的 MANIFEST 做法（版本、提交、每个文件的 sha256）和私有词表拦截 | 产物可追溯、"同样输入产物一致"的校验依据 | 拿掉多机同步部分；sha256 清单作为每次生成的产物指纹 |
| `W:tools/snap-controls.ts`、`W:tools/place-bedside.py` | 面板贴墙、避开门扇、床头开关 | **只用作校验，不用来挪位**（CONTEXT："点位由输入给定，Twinwise 不推导"）。规则保留为警告：面板被门扇挡住、不在实墙上 |
| `showroom-render/autoexpose.py` 的思路（近黑像素比例 ≤ 目标） | 曝光验收指标 | 移植成 Godot 截图后的检查，不用 Blender |

### 3.2 不要复用

- 所有针对这套房子的补丁，见 §1.3 A 表。
- `showroom_data.gd`（手挑的视角、方案、CCTV）、`data/tours/*.json`、`viewpoints.json`、`device-layout.json`、`stair-steps.json` 以及三个 details JSON：都是这套房子的数据。
- `tools/showroom-render/*`、`tools/area3d/*`（网站和面板的产物）、`tools/booth/*`（展台部署）。
- `tools/blender/stair_steps.py`：改成从外壳生成器直接输出。

### 3.3 fortis-double-bay 值得借鉴的通用化做法

| 做法 | 出处 | 对 Twinwise 的意义 |
|---|---|---|
| 场景脚本**不写死户型**，全部配置放在 `data/*.json`，改完 JSON 直接重跑，不用重新导入 | `F:CONFIG.md:3`；`F:README.md:111-116`；`main.gd` 3,935 行里只有 5 处 `"L4"`，都是默认值 | 实测证明"固定模板 + 数据 JSON"能换一套户型跑起来，是对推荐路线最直接的支持 |
| **白模节点命名契约** `<层>__<部件>__<id>`，原 id 和属性放在 extras（门扇带 hinge、hingePoint、swingInto；opening 是空标记节点） | `F:tools/ENGINE-EXPORT.md:49-70`；`F:CONFIG.md:7-23` | 可以直接作为 Twinwise 对"网站模型 glb"的输入约定草案。导出器复用网站引擎的 `buildHouseScene`（`ENGINE-EXPORT.md:4`），说明网站侧已有现成的出口 |
| **缺什么自动补**：缺地板或天花就按 rooms.json 轮廓补；完全没有白模就按轮廓画 2.7 m 墙 | `F:CONFIG.md:22-23` | 正对应 issue 里待定的"网站模型出错时，配置能覆盖哪些几何" |
| **自动给初值，输入可覆盖**：筒灯按轮廓自动排格（间距、离墙距离、每间投影数、补光都可配），`rooms.json` 给了 `"lights"` 就按图纸排 | `F:CONFIG.md` §5；`F:README.md:88` | Twinwise 的灯位、家具点位、视角都可以用这个模式 |
| **家具组库**：从 wowdemo 室内模型切出 14 个家具组和一批单件，正面统一朝 +Z，`sets.json` 记录尺寸和来源；`layout.json` 每行是 `{set, room, x, z, yaw, lift, scale, hide}` | `F:tools/extract_sets.py:1-35`；`F:tools/extract_fixtures.gd:1-11`（不用 Blender，Godot headless 切）；`F:CONFIG.md` §3 | 可作为 Twinwise **家具点位输入格式**的现成样板；wowdemo 室内模型因此变成可复用的产品库来源。注意 `SETS` 的正则写死了 wowdemo 的房间名，`FIX` 表还要人工修朝向（`extract_sets.py:77`），只能作一次性的入库工具，不能放进生成流程 |
| `ingest.sh`：交付文件拷成固定文件名，再 headless 导入 | `F:tools/ingest.sh:1-18` | 和推荐的导入步骤一致 |
| 射线验收：`check_devices3d.gd`（面板背后贴墙、朝向房内、高度档位）、`check_furniture.gd`（不穿墙、不出房间、正面留 0.6 m、不挡面板） | `F:README.md:90,96` | 直接作为 Twinwise 的验收节点 |
| `RULES.md` 给每条规则标出处：【展厅】或【本次】 | `F:RULES.md:36-58` | Twinwise 的规则文档可以照这个写法 |
| `suggest_layout.py` 自动摆家具和站点，结果写成 JSON 再手调；文件已存在时不覆盖 | `F:tools/suggest_layout.py:1-32`；`F:README.md:66-68,102` | 自动布局在 Twinwise 范围之外（"平面图没标家具时自动设计布局" out of scope），**不放进产品**；但其中的 `footprint_ok` 和 0.6 m 通道检查可以当校验规则 |
| 反例：`fittings.py` 的 `JOINERY`、`ROBES`、`FIREPLACES` 是工具里写死的像素框 | `F:tools/fittings.py:37,105,119`；`F:CONFIG.md` §7 | 柜体和壁炉在 Twinwise 里必须进输入，不能写在工具里 |

fortis 的自动比例明显高于 wowdemo：白模由 trace 工具或引擎导出，设备来自 HMV2，家具从组库自动摆放后再手调，灯位按轮廓自动排。代价是效果上限停在"复用展厅素材的白模级"（`F:README.md:3,7`），站点和家具仍要手调（`F:README.md:66-68,94-102`）。

---

## 4. 给 Twinwise 规格的建议（从以上证据推出）

1. 路线保持不变：上游交 glb，Twinwise 生成固定模板加数据 JSON，Godot headless 导入，运行时组装。wowdemo 和 fortis 的 Godot 端都已经这样跑通。
2. 渲染模板要补三条通用规则：
   - 灯带走 AreaLight3D，加回光，再加按房间分的接收层
   - 投影阴影预算和超出作用范围的光源隐藏
   - 防闪烁的固定偏移（楼板下沉 12 mm、地毯抬高 6 mm）
3. 白天室内天花亮度列为验收项；要不要引入 LightmapGI 或 VoxelGI 作可选退路，先做一个确定性实验。
4. 输入规格吸收 wowdemo 的手工交付约定：
   - 对象命名 `<房间>__<类别>__<名称>`，灯具 `__lamp__<层>__`
   - 窗帘用形态键 `open`
   - 只用 Principled，贴图打包，统一朝向和单位
   - 每间三角面和贴图预算
   - 实体到网格的绑定表（替代 `GATES`、`SASHES`）
   - 材质覆盖段、视角覆盖段
5. 网站模型 glb 的节点命名约定，可以从 fortis 的 `ENGINE-EXPORT.md` 起草。
6. "只改输入"要落实到工具上：F7 固定视角、`snap-controls.ts` 这类直接改产物数据的工具，要改成写回输入，或者降级为只出警告。
