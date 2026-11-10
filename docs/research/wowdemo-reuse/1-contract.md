# 调研报告：showtime-wowdemo 住宅契约与数据模型（对 Twinwise 的可借鉴性）

- 参考仓库：`showtime-wowdemo`，分支 `feat/house-v1`，提交 `394e3f6`（仅 1 个提交的浅克隆）。以下路径均相对仓库根目录。
- 契约版本：`contracts/house/VERSION:1` = `1.4.2`；`dist/house.json` 统计：4 层、60 个房间、143 条边界、17 根梁、50 种实体类型、351 个实体、76 个场景（脚本统计）。
- 行号说明：`schema/house.schema.json` 是压缩成一行的 JSON，引用行号按 `python3 -m json.tool` 格式化后的行号（记作 `schema(pretty):N`）。
- 术语冲突提醒：契约里的 "rules"（`src/rules.ts`）是**户型校验规则**；Twinwise 的 **规则** 指上游给定的设备行为与场景约定。两者不是一回事，后文写成「校验规则」和「规则」来区分。

---

## 0. 结论速览

1. `house.json` 是一份**语义户型 + 设备清单 + 场景**的契约：几何只用**正交的二维房间多边形（墙中心线）+ 楼层标高**，墙、门、窗、洞口都由一张**房间对房间的「边界图」**推出来。**设备没有坐标**，只挂到房间上。
2. 数据来源分两层：**APP 公开 Demo 的 JSON**（房间 ID、拓扑、216 个实体、6 个场景，原样存放）加上**这套房子的手工数据**（全部几何、边界、梁、实体类型、新增实体与覆盖、70 个场景、场景元数据）。几何**不是**从 APP 或网站导入的，APP 的户型数据里根本没有几何。
3. 构建、校验、产物一致性这套机制可以直接借：`compose → JSON Schema → 校验规则 → 写 dist`，校验不过就不写出；有测试保证已提交的 dist 和重新构建的结果完全一致。
4. 作为 Twinwise 项目文件夹格式的起点：**楼层、房间、边界图、实体类型、场景这几部分可以借**；但 Twinwise 关心的**点位、回路到灯具、产品与款式、家具、贴图、灯带、墙板、地理位置**都不在契约里。它们散在 `showroom/data/*.json`、Blender 导出的 GLB 节点命名和 GDScript 常量里，而且加产品要改代码。另外，契约里有大量只适用于这套房子的硬编码。

---

## 1. house.json 数据模型

### 1.1 顶层

| 字段 | 必填 | 说明 | 出处 |
|---|---|---|---|
| `schemaVersion` | 是 | semver，等于 `VERSION` 文件 | `src/types.ts:19`；`schema(pretty):41-44`；`README.md:5` |
| `houseId` | 是 | 固定写死为 `'wowdemo-signature'` | `src/compose.ts:22` |
| `floors[]` `rooms[]` `boundaries[]` `entityTypes[]` `entities[]` `scenes[]` | 是 | 见下 | `schema(pretty):6-15` |
| `beams[]` | 否 | 结构梁，只给结构校验用 | `src/types.ts:18` |
| 顶层不允许多余字段 | — | `additionalProperties:false` | `schema(pretty):16` |

坐标约定：单位米，**x 向东、y 向北**，楼层标高见 `floors[].elevation`（`README.md:6`）。Godot 那边的换算是 x 东、y 上、−z 北（`showroom/scripts/sun_sim.gd:5`）。

### 1.2 楼层 `floors[]`

| 字段 | 类型 | 说明 | 出处 |
|---|---|---|---|
| `id` | `'b1'\|'gf'\|'ff'\|'rf'` | **枚举写死为 4 层** | `src/types.ts:1,3`；`schema(pretty):31-38` |
| `name` | string | Basement / Ground / First / Roof | `data/floors.json:4,10,17,24` |
| `elevation` | number（米） | −3.5 / 0 / 3.58 / 6.66 | `data/floors.json:5,11,18,25` |
| `ceilingHeight` | number > 0 | 层高 | `schema(pretty):69-72` |
| `appFloor` | `'GF'\|'1F'\|'RF'` | 对应 APP 的楼层名 | `src/types.ts:3` |

### 1.3 房间 `rooms[]`：范围、名称、标高

| 字段 | 说明 | 出处 |
|---|---|---|
| `id` | 沿用 APP 的 roomId（如 `GF-6`）；新增房间用同样的格式 | `README.md:7`；`data/rooms/gf.json:2` |
| `name` | 显示名；没写就取 APP 的名称 | `src/compose.ts:10` |
| `floor` | 楼层 ID | `src/types.ts:7` |
| `kind` | `room` / `circulation` / `outdoor` / `void`（通高，上一层没有楼板） | `src/types.ts:4`；`README.md:11` |
| `use` | 18 种用途枚举（living…stair…roof） | `src/types.ts:5` |
| `polygon` | **范围**：二维点列，按**墙中心线**画，至少 4 个点，**必须正交** | `README.md:10`；`schema(pretty):138`；`src/rules.ts:57` |
| `ceilingHeight` | 房间单独的层高（如客厅通高 6.46 m）；**标高**用楼层的 `elevation`，没有房间级标高 | `data/rooms/gf.json:3`；`shell/src/shell.ts:98` |
| `targetArea` | 目标面积，校验时允许 0.85 到 1.6 倍 | `src/rules.ts:58` |
| `openGroup` | 开放式分区组（如 `kdl`），组内不许有墙 | `data/rooms/gf.json:3-5`；`src/rules.ts:116-124` |
| `daylight` / `fullHeightGlazing` | 是否要求采光 / 落地窗 | `src/types.ts:7`；`src/rules.ts:143-152` |
| `stair` | `{flightWidth, landing:'n'\|'s'\|'e'\|'w'}`，表示进出楼梯的那条边 | `src/types.ts:7`；`README.md:12` |
| `aliases` | 别名（如 Library = 书房） | `src/types.ts:7`；`CHANGELOG.md:9` |
| `appRoomId` `planAreaId` `source` | 由构建步骤填入：APP 映射，以及来源是 `app-demo` 还是 `wowdemo` | `src/compose.ts:10`；`src/data.ts:3` |

### 1.4 墙、门、窗、洞口：边界 `boundaries[]`

墙**不单独存**，由边界推出来。两个房间之间只要有一条不是 `open` 的边界，就在它们共享的墙段上建墙，再在墙上按 `at` 和 `width` 开门、开洞、开窗；房间的边如果不和同层其他房间共享，就是外墙（`README.md:8-9`）。墙厚是常量：外墙 0.25、内墙 0.12、Basement 挡土墙 0.3（`README.md:10`；`shell/src/shell.ts:27`）。

| 字段 | 说明 | 出处 |
|---|---|---|
| `id` | 如 `gf-gf-9-gf-6` | `data/boundaries/gf.json` |
| `a` / `b` | 两侧房间 ID；`b` 可以是 `'outside'` | `src/types.ts:11`；`src/rules.ts:78` |
| `kind` | `open` / `opening` / `door` / `wall` / `glazing` | `src/types.ts:8` |
| `type` | 20 种：门（hinged、sliding、pocket、double、fire、acoustic、glass、garage、external）、窗（stacker、slider、fixed、window、highlight、obscure、skylight）、`balustrade`（栏杆）、`stair`（层间连接）、servery、concealed | `src/types.ts:9`；`README.md:8` |
| `at` | 开口中心点在平面上的坐标（必须落在墙线上）；天窗的 `at` 是屋顶上的中心 | `src/rules.ts:96-101,88-93` |
| `width` / `height` / `sill` | 宽、高、窗台高 | `src/types.ts:11` |
| `hinge` | `'low'\|'high'`：单扇门挂在哪一侧门框 | `src/types.ts:10-11` |
| `main` | 主洞口，要求 ≥ 1.8 m | `src/rules.ts:109` |

- **楼梯**由三部分组成：`use:'stair'` 的房间，每层一个；层间用 `type:'stair'` 的 `open` 边界连接（`data/boundaries/vertical.json:2-15`）；房间上的 `stair.landing`。踏步几何不在契约里，有两份：shell 生成器里的螺旋楼梯（`shell/src/shell.ts:14`），以及手工的 `showroom/data/stair-steps.json`。
- **梁** `beams[]`：`{id, floor, from, to}`，只用于「楼上外墙必须落在首层墙或梁上」这条校验（`src/types.ts:17-18`；`src/rules.ts:169-190`）。
- **门的交互**：契约里**没有门的状态**。shell 根据 `hinge` / `width` 生成门扇（`shell/src/shell.ts:17-19`），Godot 点一下开关通往室外的门（`showroom/scripts/main.gd:3237-3240`），开关状态只存在本地变量 `_door_open` 里。

### 1.5 设备：实体类型 `entityTypes[]` 和实体 `entities[]`

**实体类型**（相当于 Twinwise **规则**的一部分：值域、过渡时长、动画语义）：

| 字段 | 说明 | 出处 |
|---|---|---|
| `type` | 与 APP 的 entityType 同名，如 `DALI Dimming`、`Blind` | `data/entity-types.json:2-51` |
| `domain` | light / cover / climate / fan / switch / sensor / security / media… | 同上 |
| `state` | `{kind:number\|boolean\|enum\|none, min, max, values}` | `src/types.ts:12` |
| `transitionMs` / `animation` | 过渡时长；动画是 dim / on-off / open-close / colour / climate 等枚举 | `src/types.ts:12` |

单位约定：亮度 0–100、窗帘 0 关 1 开、温度 16–30 ℃（`README.md:17-21`）。

**实体**：

| 字段 | 说明 | 出处 |
|---|---|---|
| `id` | APP 的 entityId（有些是 UUID）；新增实体用 `<domain>.<floor>_<room>_<type>_<n>` | `README.md:7`；`src/rules.ts:5,196` |
| `name` / `type` | `type` 必须在 entityTypes 里声明过 | `src/rules.ts:194` |
| `roomId` / `floor` | **唯一的位置信息**，可以为 null | `src/types.ts:13` |
| `groupOf` | 灯组或帘组的成员 ID（一层，不许嵌套） | `src/rules.ts:198-207` |
| `capabilities` / `semantic` | 原样透传 APP 的数据（含 IP、密码等设备配置；`semantic.extraConfig` 里有 `layeredLightingType`、`curtainType`） | `dist/house.json` 第一个实体；`src/compose.ts:48-63` |
| `source` | `app-demo` / `wowdemo` | `src/types.ts:13` |

要点：

- **一个实体就是一路「回路」**，例如 `light.gf_kitchen_dali_4` 是一路 DALI 回路。灯具和回路之间的对应**不在契约里**，有三处：
  - GLB 网格的命名 `<room>__lamp__<layer>__…`；
  - 「按名称把回路归到某一层」的启发式（`showroom/scripts/lighting.gd:1-8,21-30`）；
  - 修补表 `showroom/scripts/lighting_repairs.gd:5-10`，以及 `showroom/data/strip-area-manifest.json` 里的 `cid`。
- **点位不在契约里**：开关、面板、传感器、音箱、摄像头、门锁的坐标和产品编码在 `showroom/data/device-layout.json`（字段 `kind, label, product, floor, x, y, room, surface, heightMm`，共 107 个），这个文件的真相源和契约是分开的（`docs/DATA-SYNC.md:18`）。这些点位**和契约实体之间没有 ID 关联**。
- **产品和款式**：`device-layout.json` 只写 HMV2 产品编码（如 `UI-BTN-LUN-06-GLD`）。编码到模型文件、到款式的映射**写死在代码里**：`showroom/scripts/devices3d.gd:120-126`（系列与款式表）和 `devices3d.gd:325-345`（编码转文件名的 if 链），模型放在 `showroom/assets/products3d/<系列>_<款式>.glb`（`devices3d.gd:265-267`）。
- schema 对实体、实体类型、场景不封闭，没有 `additionalProperties:false`（`schema(pretty):309-374`），所以可以带任意附加字段。

### 1.6 场景 `scenes[]`

| 字段 | 说明 | 出处 |
|---|---|---|
| `id` / `label` / `scope` | `scope` 是 `whole_house` 或房间 ID | `src/types.ts:16`；`data/scenes.json:3-5` |
| `targets[]` | `{entityId, state, attributes}`；`attributes` 里有 `brightness`（0–100）、`rgb` 等 | `src/types.ts:14`；`data/scenes.json:6-30` |
| `kind` / `name` / `icon` / `intent` / `rooms` / `aliases` / `behaviours` | 由 scene-meta 合入；`behaviours` 必须在 Agent 的行为列表里 | `src/compose.ts:25-31`；`schema(pretty):423-455` |

scene-meta 还支持一组**宏**，构建时展开：`allLights: off`、`allLights: {on:'outdoor'}`、`allMedia`、`allCovers`、`except`、`add`、`drop`（`src/compose.ts:65-86`）。场景只描述目标状态，动画时序由大屏按实体类型执行（`docs/specs/2026-09-25-wowdemo-architecture.md:173`）。

### 1.7 「规则」

- **设备行为规则**：等于 `entityTypes`（值域、过渡、动画）、场景、scene-meta，加上 Hub 里的模拟器。契约里没有自动化或时段的定义；规格里写了「自动化」（`spec:78`），但 schema 里没有这个字段。
- **校验规则**：`src/rules.ts` 一共有 **37 个规则 ID**，覆盖引用、几何、相邻、开放式分区、可达、采光、宽度、楼梯、竖向结构、实体命名、分组、场景引用。`README.md:14` 写的是「24 条」，已经过时。

---

## 2. 数据从哪来、怎么校验、怎么构建

### 2.1 来源：上游是 Homira APP Demo，不是网站

- `sources/app-demo/` 下原样保存 4 个文件：`room_list.json`、`entity_list.json`、`scene_actions.json`、`mock-request.json`。来源是 homira-app 的一次提交，**规定不得手改**（`sources/app-demo/SOURCE.md:3-10`）。
- `src/sources.ts:10-13` 读取这 4 个文件。`src/import-app-demo.ts` 做三件事：
  - `importRoomSeeds`：房间 ID、名称、楼层，楼层映射**只认 GF 和 1F**（`:3-14`）；
  - `importEntities`：展开分组控制（`:18-30`）；
  - `importScenes`：亮度从 0–255 换算成 0–100（`:31-35`）。
- `mock-request.json` 是 APP 户型图的交接数据，只有 roomId、planAreaId、roomType 和图片 URL，**没有任何几何**。规格也明确说 APP 户型只是气泡图，无墙、门、窗，无比例（`spec:90`）。测试锁定了来源的规模：27 个房间、216 个实体、6 个场景（`tests/sources.test.ts:4-12`）。
- 所以对 Twinwise 来说：这套契约的「上游导入」**只负责 ID、拓扑和设备清单**；几何是人照参考设计重新画的（ADR-0003，`docs/decisions/0003-house-layout-from-reference-skeleton.md:17-36`）。

### 2.2 手工数据和通用机制的分界

**这套房子的手工数据**（`contracts/house/data/`，读取入口在 `src/data.ts:9-24`）：

| 文件 | 内容 |
|---|---|
| `floors.json` | 4 层的标高和层高 |
| `rooms/{b1,gf,ff,rf}.json` | 全部房间多边形，含 33 个新增房间 |
| `boundaries/{floor}.json`、`vertical.json` | 全部边界图 |
| `beams.json` | 17 根梁 |
| `entity-types.json` | 50 种类型的值域 |
| `entities.json` | `overrides`（第 2 行起）、`additions`（第 231 行起，135 个新增实体）、`removed`（第 1389 行起） |
| `scenes.json` | 70 个新增场景 |
| `scene-meta.json` | 场景元数据和宏 |
| `behaviours.json` | Agent 行为 ID 快照 |

**通用机制**（理论上换一套房子也能用）：`geometry.ts`（正交多边形的共享边、重叠、点是否在多边形内）、`graph.ts`（基于边界图的广度优先可达）、`schema.ts`（Ajv 2020）、`compose.ts` 的合并、覆盖、删除流程、`tools/build.ts`、`tools/render-plan.ts`、`shell/`（从契约生成建筑外壳 GLB）。

**通用机制里夹带的这套房子的硬编码**（迁移时要剥掉）：

| 硬编码 | 出处 |
|---|---|
| 楼层只能是 4 个固定值 | `src/data.ts:11`；`src/types.ts:1`；`schema(pretty):31-38` |
| `houseId` 写死 | `src/compose.ts:22` |
| 入口房间 `ENTRANCE='GF-6'`；客餐厨三件套按房间名判断；落地窗规则 | `src/rules.ts:4,10` |
| 实体命名正则里的楼层码 | `src/rules.ts:5` |
| 拼写修正表、按名称推断灯光分层和窗帘类型 | `src/compose.ts:34-47` |
| 只认 GF / 1F 的楼层映射 | `src/import-app-demo.ts:3` |
| 「只有首层托二层」的结构检查 | `src/rules.ts:170-171` |

### 2.3 构建、校验、出图

| 命令 | 做什么 | 出处 |
|---|---|---|
| `npm run contract:build` | `composeHouse(loadAppSources(), loadContractData())` → `validateSchema` 加 `validateHouse` → 有问题就逐条打印，退出码 1，**不写 dist**；否则写 `dist/house.json` | `package.json` scripts；`tools/build.ts:5-14` |
| `npm run contract:plan` | 先构建，校验通过才为每层画 SVG 平面：北朝上，40 px = 1 m，墙、门、窗、洞口、栏杆、上层轮廓、梁 → `dist/plans/*.svg` | `tools/render-plan.ts:6-62` |
| `npm test` | `tests/build.test.ts:3-10` 断言**已提交的 dist 和重新构建的结果深度相等**，并且没有任何问题；`tests/` 下一共约 92 个 `test()` | `tests/*.test.ts` |
| `npm run shell:build` | 契约 → `showroom/assets/house-shell.glb`，同时复制 `house.json` | `shell/tools/build.ts:1-7` |
| `dist/agent/` | 用 Python 脚本从 house.json 派生 Agent 同步数据 S1–S4，**不在 contract:build 里**，要手动运行 | `dist/agent/README.md:3,17-21` |
| `tools/publish-house.sh` | 构建 → 出图 → shell → 测试 → 私有词表扫描 → 分发到各端 | `tools/publish-house.sh:1-30`；`docs/DATA-SYNC.md:5-10` |

修改流程是：改 `data/` → build → plan → test → 把 dist 一起提交（`README.md:15`）。这和 Twinwise 的「只改输入，不手改产物」「同样输入产物一致」是同一种思路。

---

## 3. 能不能直接作为 Twinwise 项目文件夹或配置格式的起点

**结论：可以借它的骨架和机制，但不能直接拿来当格式。**

值得借的：
- 楼层、房间、边界图的三层语义模型；
- 实体类型的状态值域，加上场景的目标状态表达；
- 「data/ 原样来源 + 手工补充 → compose → schema → 校验规则 → dist」的确定性构建；
- dist 与重新构建结果一致的测试；
- 自动出 SVG 平面供人审阅。

不能直接用的原因有三个：
- 一是它假设几何由人手画，Twinwise 的几何来自上游的 glb；
- 二是 Twinwise 最核心的点位、产品、款式、贴图都在契约外面；
- 三是硬编码太多，见 §2.2。

### 3.1 逐项对照 Twinwise 约束

| Twinwise 约束 | 契约现状 | 缺口 | 出处 |
|---|---|---|---|
| **多层** | 有楼层、标高、层高、void 通高、层间楼梯边界 | 楼层 ID 枚举写死为 4 个；数据加载器写死 4 个文件；结构检查只看首层和二层 | `src/types.ts:1`；`src/data.ts:11`；`src/rules.ts:170-171` |
| **只做室内，外墙简单体块** | 外墙由「不共享的边」自动推出；有 `outdoor` 房间 | 契约包含大量户外（泳池、车道），Twinwise 用不上；可以保留外墙推导 | `README.md:9` |
| **门可交互** | 门以边界表达（类型、宽、位置、门轴），shell 生成门扇 | 契约里没有门的状态或交互声明；Godot 只对外门做点击开关，状态是本地变量 | `src/types.ts:11`；`shell/src/shell.ts:17-19`；`main.gd:3237-3240` |
| **房间、楼层跳转** | 房间有 ID、名称、别名和多边形，可以算范围；楼层有标高 | 每个房间的视角（`eye`/`look`）在 `showroom/data/viewpoints.json`，手调，不在契约里 | `showroom/data/viewpoints.json`；`main.gd:6564` |
| **点位**（由上游给定） | 实体只有 `roomId`/`floor`，**没有坐标** | 控件类点位在 `showroom/data/device-layout.json`，和实体没有 ID 关联；灯具点位只在 GLB 命名里，或按网格自动布灯（`lighting.gd:3`）。需要一个点位对象：`{id, entityId, product, variant, floor, x, y, z/heightMm, surface, facing}` | `src/types.ts:13`；`docs/DATA-SYNC.md:18` |
| **回路**（由上游给定） | 实体就是回路；有 `groupOf` 分组 | 灯具和回路的对应靠名称启发式加修补表；需要点位上显式写 `circuit` | `lighting.gd:7`；`lighting_repairs.gd:5-10` |
| **灯沿用真实流明和色温** | 没有 | 色温写死在 GDScript 常量里；契约里没有灯具光度参数 | `lighting.gd:10-11` |
| **家具点位** | 没有 | 家具在 Mac 导出的室内 GLB 里，用网格名 `<room>__furniture__…` 表示 | `showroom/data/surface-manifest.json` |
| **定制贴图、灯带、墙板** | 没有 | 材质在 `surface-manifest.json`（网格到材质）和 `data/materials/`；灯带在 `strip-area-manifest.json`（灯具、房间、回路、位置）；还有大量 `*_finish.gd` 脚本按房间写死。需要一个「饰面」对象：`{target: 房间/墙段/边界, material/texture, 尺寸约定}`，以及灯带对象 `{path, circuit, 光度参数}` | `showroom/data/`；`showroom/scripts/*_finish.gd` |
| **产品库与款式**（加产品只加资产） | 没有 | 产品编码到模型文件、款式的映射写死在代码里，**违反**「只加资产」；需要产品库清单 `{productId, variants[], glb, 安装面, 特写机位}` 由数据驱动 | `devices3d.gd:120-126,325-345` |
| **同一点位换款式、产品特写** | 只有「全屋开关一键换系列」 | 需要点位级的 `variant`，以及产品库给的特写机位 | `devices3d.gd:118-126` |
| **地理位置和正北朝向** | 约定 +y 就是真北（1.0 版「泳池朝真北」），**没有字段** | 经纬度写死为悉尼；没有正北偏角。需要 `site:{lat, lon, northAngle, utcOffset}` | `README.md:6`；`CHANGELOG.md:83`；`sun_sim.gd:7-8` |
| **上游网站模型（glb）** | 没有引用；外壳 GLB 由契约多边形生成 | Twinwise 的几何来源是上游 glb（#5 已定）。边界图更适合当**语义层或覆盖层**：哪段是门、哪个房间、可达关系，而不是几何源。还要解决 glb 节点和房间、门的 ID 对应 | `shell/tools/build.ts:3-5` |
| **非正交户型** | 校验强制正交 | 网站模型可能有斜墙，`geometry.ts` 全部基于正交假设 | `src/rules.ts:57`；`src/geometry.ts:6-16` |
| **同样输入产物一致** | dist 与重新构建一致的测试，加上校验失败不写出 | 可以直接沿用 | `tests/build.test.ts:3-10`；`tools/build.ts:11` |
| **规则（场景、温度）** | 有实体类型、场景和宏 | 没有时段、天气；没有温度驱动设备的声明（热模型在 Hub 里）；没有自动化 | `src/compose.ts:65-86`；`spec:78,211` |
| **只读 dashboard、按节点显示进度** | 构建是一次性 CLI，只打印最后结果 | 没有分节点的状态输出 | `tools/build.ts:10-14` |

### 3.2 给 Twinwise 格式的建议（从这套契约改造）

1. **保留**：`floors`（楼层 ID 改成自由字符串，按标高排序）、`rooms`（多边形加 `kind`/`use`/`name`/`aliases`/`ceilingHeight`）、`boundaries`（门、窗、洞口、栏杆、楼梯的语义，并加 `interactive` 和门状态）、`entityTypes`、`entities`（即回路）、`scenes`；保留构建、校验、dist 一致性测试和 SVG 出图。
2. **去掉**：APP Demo 专属的导入器和拼写修正表、`ENTRANCE`/KDL 等按名称写死的校验规则、Agent 派生数据、户外细节、结构梁校验（Twinwise 不验证结构）。
3. **新增**：
   - `site`：经纬度、正北偏角；
   - `model`：上游 glb 路径，以及节点到房间、门的映射；
   - `placements[]`：点位，含产品、款式、回路、坐标、安装面；
   - `fixtures[]`：灯具，含回路、流明、色温；或者并入点位；
   - `furniture[]`：家具点位，平面图给出或人工补充；
   - `finishes[]`：贴图、墙板、灯带；
   - `viewpoints`：可选，作为覆盖项；
   - 独立的**产品库清单**，由数据驱动加载模型和款式，取代 `devices3d.gd` 里的 if 链。
