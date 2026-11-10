# 参考仓库 showroom-space-3d 的可复用性

回答 #4（属于地图 #1）：参考仓库里哪些逻辑和数据结构通用、可以给 Twinwise 复用，哪些写死在 146 Arthur St B1 展厅里。

- **来源**：`http://gitlab.teccoibms.com/super-designer/showroom-space-3d.git`，commit `9c05ca3`（浅克隆）。下文的 `文件:行号` 都指这个 commit。
- **方法**：通读 `build_v2.py`、`switch_v2.py`、`export_web.py`、`godot/import_web.py`、`godot/main.gd`、`godot/project.godot`、`web/lights.json`（用脚本统计字段，没有逐行看）、`web/index.html`、`web/agent.js`，另外看了 `README.md` 和几个辅助脚本的文件头。Godot、Blender 的行为以官方文档为准，链接附在相应位置。
- 仓库注释里写着"实测"、但文档没写到的内容，下文标为 **仓库实测**。这些结论没有复现过。

## 结论速览

1. **Godot 运行时（`main.gd`）的骨架是数据驱动的，可以改造成 Twinwise 的通用运行时。** 灯光、场景、时段、天气都在启动时从 `lights.json` 读入（`main.gd:61`）；画质档位、环绕/行走/预设视角、开关面板也不依赖具体空间。要改的是十几处写死的常数和命名约定（见[写死的内容](#写死在-b1-展厅里的内容)）。
2. **`lights.json` 的灯具字段基本能直接用作 Twinwise 的灯光数据格式。** 每盏灯带 `circuit/type/pos/dir/up/color/lm`，再按类型带 `cd/angle/penumbra` 或 `w/h/nits`，单位是光度学单位。缺的是：可调光还是只能开关、色温、所属房间/楼层、引用的产品。
3. **场景按"回路 → 0..1 调光比例"的表来定义（`switch_v2.py:15-27`），这个结构通用；表里的数值是 B1 专用的。** 可以直接作为上游"规则"里灯光场景的格式起点。
4. **灯光换算里，通用的是物理推导，写死的是校准量。** 1/683 单位制、spot 的 cd、troffer 用 89.5° spot 模拟余弦分布、灯带"洗墙"的闭式积分、bounce 用成对 omni 补光，这些都不依赖空间。`L4_BAR = 0.75` 是在补 Blender 建模时的一个副作用（球形光源有一半埋进灯杆），新空间不成立。Twinwise 如果不经过 Blender，这个常数就没有意义。
5. **`build_v2.py` 基本不可复用。** 它是"在 Ettie 手工建的 .blend 外壳上，按 Figma 毫米坐标叠加长墙"的一次性脚本：坐标、墙段、贴图矩形、产品位置、柱子、楼梯、窗帘路径、灯盘网格都是常数。能拆出来的是十来个几何/光度小函数（墙面局部坐标系、带 UV 矩形的板、色温转 RGB、流明转能量、灯带宽度承载流明、窗帘开合两套网格）。
6. **网页端（`index.html`、`agent.js`）对 Twinwise 只有参考价值**（Twinwise 只产出 Godot 项目）。`agent.js` 里"区域 / 音箱 / 摄像头 / 动线"的数据形状，可以给"设备种类与行为"做参考。

## 参考仓库的流水线

```
Ettie 的 V1-Pre .blend（手工外壳）
  └─ build_v2.py：按 Figma 1px=1mm 叠加长墙、产品、灯、相机 → V2 .blend
       └─ switch_v2.py：场景 / 时段 / 天气数据（Blender 内切换，也供导出用）
            └─ export_web.py：烘焙程序化材质 → glTF（嵌入式 .json）+ lights.json
                 └─ godot/import_web.py：.json → .glb，拷贝 lights.json 和烘焙贴图
                      └─ godot/main.gd：运行时读 lights.json，建灯、洗墙、探针、UI
```

出处：`README.md:10-19,43`、`export_web.py:1-5,169-171`、`godot/import_web.py:1-4`。

Twinwise 的输入是上游的 three.js 网站模型和数据，不是手工 .blend（见 `CONTEXT.md`、#1）。所以这条流水线**最前面一段（手工外壳 + build_v2）没法照搬**。要不要保留 Blender 这一段，由 #5 决定。

## 可复用性总表

可复用程度：**直接** = 拿来就能用，最多改名字；**改造** = 逻辑通用，但要把写死的值改成从配置读；**不可** = 只属于 B1，或者对 Twinwise 没有用。

| 模块 | 可复用程度 | 理由 | 出处 |
|---|---|---|---|
| `lights.json` 灯具条目字段 | 直接 | `name/circuit/type/pos/dir/up/color/lm`，spot 另带 `cd/angle/penumbra`，面光另带 `w/h/nits/visible`。光度学单位，坐标是 glTF 的 Y-up，不依赖空间 | `export_web.py:78-98`；`web/lights.json`（共 70 条：L6 面光 29、L4 spot 16、L5 面光 12、L2 面光 6、L1 面光 4、L3 spot 3） |
| `lights.json` 顶层：`scenes/times/weathers/sky/exposure/strip_emit/screen_emit/view_transform` | 改造 | 结构通用；`sky[时段\|天气]` 是预先算好的太阳和天空，`indoor` 来自 B1 的 Cycles 日光测算 | `export_web.py:108-122` |
| 场景表（回路 → 0..1，外加 `screens`） | 改造 | 形状通用，可以作为"规则"里灯光场景的格式；数值、回路编号、屏幕名是 B1 的 | `switch_v2.py:15-27` |
| 回路 = Blender 集合名的前两个字符（`L1`…`L6`） | 不可 | 靠命名约定截取 `coll.name[:2]`，回路数也写死在 `for c in [...]` 里 | `export_web.py:78`；`switch_v2.py:36`；`main.gd:479` |
| 总开关 / 天花灯单独开关 / L6 只开不调 | 改造 | 通用需求：总开关，以及某类回路单独开关。但"天花灯 = L6"写死了，"只开不调"只写在注释里，数据里没有字段 | `main.gd:460-465`；`switch_v2.py:20-21` |
| 太阳位置（NOAA）+ 晴/阴天空模型 + 天气表 | 改造 | 算法通用；`LAT/LON/TZ/DATE` 写死为北悉尼 2026-10-22，`TZ=11` 不处理夏令时 | `switch_v2.py:72-139` |
| 曝光规则 `EV = log2(π·683/E)` | 改造 | 公式通用；夜间 `NIGHT_LUX = 781` 来自 B1 的照度计算 | `switch_v2.py:76-82` |
| 色温 → 线性 RGB（`kelvin_rgb`） | 直接 | 普朗克黑体 × CIE 1931 拟合，带自检 | `build_v2.py:988-1003` |
| 流明 → Blender 能量（`spot_watts`、`lamp`） | 改造 | 仅在经过 Blender 时需要；"Blender 的 spot 功率按全向点光计"是仓库的说法，Blender 手册 Light Objects 页没写这一点（见下文"Blender 单位"） | `build_v2.py:1009-1035` |
| 灯带 = 自发光面，宽度承载流明（`glow_strip`） | 改造 | 方法通用；`LED_EMIT` 按 5 W/m、10 mm 面、80 lm/W 推出，都是 B1 灯具参数 | `build_v2.py:1040-1066` |
| 墙面局部坐标系 + 带 UV 矩形的板（`wall_rig/slab/uv_rect`） | 改造 | 定制贴图、墙板、灯带都"写进输入"（#1 约束），这套"沿墙 X、出墙 -Y、Figma 1px=1mm 映射 UV"的表达可以直接作为墙面装饰点位的格式参考 | `build_v2.py:6-7,172-212` |
| 长墙段 / 贴图矩形 / 展盒 / 产品毫米位置 / 屏幕 | 不可 | `D_SOUTH/D_NORTH/S0/SEGMENTS/T_*/BOXES/PRODUCTS/SCREENS` 都是 B1 常数；但它们的**元组形状**可以作为点位格式的参考（见 #6 要点） | `build_v2.py:18-114` |
| 外壳修补（柱子、栏杆、楼梯井、入口玻璃、窗台、卷帘盒） | 不可 | 按现场照片和 2008 年租赁平面图手工估计，直接改 Ettie 的网格顶点 | `build_v2.py:726-893` |
| 窗帘（沿玻璃折线的轨道 + 打开/拉上两套网格） | 改造 | 机制通用（两套网格切显隐，`main.gd:497-500`）；折线坐标和避让柱子的区间是 B1 的 | `build_v2.py:895-958` |
| 灯盘网格（L6） | 不可 | 4 条 x 线、2.4 m 间距，房间东侧边界用 `room_x_max` 分段写死 | `build_v2.py:1140-1165` |
| 程序化地毯/天花材质与烘焙 | 改造 | "程序化材质烘焙一个周期 + 运行时按世界坐标三平面贴图"通用；材质名 `Material_0/1/2` 来自 Ettie 的外壳 | `build_v2.py:285-386`；`export_web.py:25-56`；`main.gd:92-123` |
| 产品 GLB 的修补（隐藏盖板玻璃、LUNA 温控屏、HomeGlow 竖装 UI） | 不可（应移到产品库） | 这些是产品资产自身的问题；按 #1 的约束"新增产品只加资产不改代码"，应该在产品库里修好，不进生成代码 | `build_v2.py:533-583` |
| 小物体贴图降到 512/1024 | 直接 | 通用的显存优化 | `build_v2.py:1222-1244` |
| Blender Y-up 转换 `gl()`、超过 16000 px 的贴图缩放 | 直接 | 通用 | `export_web.py:20-22,58-62` |
| glTF 拆分 + base64 嵌入 `.json`，再在 Godot 侧转回 `.glb` | 不可 | 只是为了绕开 artifact 网页 16 MB 文本限制、不能托管 .glb；Twinwise 直接产出 GLB 就行 | `export_web.py:124-166`；`import_web.py:12-34` |
| 玻璃：去掉 `KHR_materials_transmission`，改成 alpha 混合薄片 | 改造 | **仓库实测**：Godot 的 glTF 导入器会丢掉这个扩展；透明度表按材质名 `Material_4` 写死 | `import_web.py:2-3,9,16-21` |
| `main.gd` 加载与材质归类 | 改造 | 逻辑通用；靠材质名（`Material_0/1/2`）和网格名前缀（`灯带背_`、`屏幕背光_`、`窗帘合_`、`窗帘_`）识别地面、天花、灯带、窗帘 → 需要一套语义标记约定 | `main.gd:107-141` |
| Godot 灯光换算（1/683、E = cd/π、平方反比、penumbra × 0.6） | 直接 | 和空间无关；E = cd/π 和 penumbra 的映射是**仓库实测** | `main.gd:3-5,12,170-178,219-232` |
| troffer 用 89.5° spot 模拟朗伯面板 + 发光面片 | 改造 | 原理通用；`light_size = 0.6` 对应 300×1200 灯盘，应该改成由 `w/h` 推出；"天花灯 = L6"写死 | `main.gd:179-204` |
| 灯带"洗墙"叠加层（闭式积分 + 射线找墙面 + 反照率取样） | 直接 | 算法通用，按 `lights.json` 里的灯带分组，射线自动找被照的面；前提是灯带离墙面只有几厘米 | `main.gd:249-354,402-426` |
| bounce 补光（每回路每 3 m 格子一对 ±E 的 omni） | 直接 | 绕开 SDFGI 对 spot 光吃不进去的问题（**仓库实测**，见下文），和空间无关 | `main.gd:205-217,235-246` |
| 反射探针 | 改造 | 每组展示灯带一个探针，逻辑通用；入口墙的探针坐标 `(6.44, 0, -28.0)`、探针尺寸 `4.5×2.75×3.6` 写死 | `main.gd:360-383` |
| SDFGI、SSAO、SSR、AgX、自定义天空 shader | 直接 | 环境设置通用；切换状态后重启 SDFGI 并重拍探针 | `main.gd:430-457,512-516,386-399` |
| 画质档位（高 / 流畅） | 直接 | 流畅 = 关掉 troffer 阴影、关 SSR、关 MSAA、FSR2 0.75 倍、GI 半分辨率；macOS 默认流畅 | `main.gd:69,502-511`；`project.godot` |
| 预设视角（glTF 相机 `Cam_*` + 1–8 键） | 改造 | 机制通用（按名字排序，`KEEP_WIDTH` 保持出图的水平视野）；相机本身是 build_v2 按 B1 摆的，`STILL_ASPECT` 依赖 1920×1080 出图 | `main.gd:20,77-79,521-541`；`build_v2.py:1167-1195` |
| 环绕 / 行走 | 改造 | 通用；但眼高 1.6、"y > -0.5 才算在 C-2 层"、"高于 2.75 就揭掉天花"都假设单层、层高 2.7 m；**行走没有碰撞**，会穿墙，也不会上下楼 | `main.gd:597-617,698-722` |
| 自动预览（巡游） | 改造 | 停靠点之间 smoothstep 插值、每站停留 3 s 的机制通用；停靠点由 D 墙的 `D_SOUTH/ALONG/OUTWARD/SECTIONS` 算出，入口机位写死 | `main.gd:21-28,551-594` |
| 快捷键（F5/F6/F7/F9/Ctrl+F9/F10/Shift+`/P/H/1–8） | 直接 | 和 Blender 插件、网页一致 | `main.gd:663-695`；`README.md:35` |
| UI 面板 | 改造 | 通用；标题"Homira B1 新展厅"、地址、脚注里的"万泰参数 3500 K"都写死 | `main.gd:784-820` |
| `web/index.html`（three.js 查看器） | 不可（仅作参考） | Twinwise 不产出网页 3D（#1 Out of scope）；为了 WebGL 编译时间把灯盘合成行面板、L5 合成环形面板，bounce 用一个全屋估算值 | `web/index.html:265-294,318,377-379` |
| `web/agent.js`（讲解 Agent 演示） | 不可（数据形状可参考） | 区域、音箱、摄像头、访客和销售的关键帧全写死在 D 墙坐标里；"区域 / 音箱 / 摄像头视锥"的数据形状可以给设备种类做参考 | `web/agent.js:7-52,56-90` |
| 照度验算 `lux_calc.py`、`daylight_v2.py`、备用出图 | 改造 | 逐点照度、日光测算和"每个相机 × 每组状态出图"作为验收手段可以参考；都依赖 Blender/Cycles，路径写死在 `E:\` | 各文件头；`backup_renders.py:1-3` |

## 关键点细看

### lights.json 的结构

顶层键（`export_web.py:119-122`）：

```
lights[]       每盏灯或每段灯带一条
scenes{}       场景名 → {回路: 0..1, screens: {屏幕名: 0|1}}
strip_emit     灯带自发光面 100% 时的强度（18.64）
screen_emit    屏幕自发光强度（0.6，约 400 nit）
times[]        时段名，如 "中午_1230"
weathers[]     天气名，如 "晴天"
sky{}          "时段|天气" → {alt, sun[3], e_dn, e_dh, colour[3], indoor}
exposure{}     时段 → EV
view_transform "AgX"
```

灯具条目（下面是 `web/lights.json` 的实际例子）：

```json
{"name":"射灯_Living_Board","circuit":"L3","type":"SPOT","pos":[5.4635,2.2912,-15.925],
 "dir":[-0.3955,-0.8827,0.254],"up":[-0.7427,0.47,0.477],"color":[1.0,0.5768,0.2605],
 "lm":1215.0,"cd":3950.95,"angle":0.3142,"penumbra":0.15}
{"name":"灯盘_01","circuit":"L6","type":"AREA","pos":[5.5,2.68,-9.2],"dir":[0,-1,0],
 "up":[-0.9999,0,-0.0125],"color":[1.0,0.5768,0.2605],"lm":4000,"w":0.3,"h":1.2,
 "nits":3536.78,"visible":true}
```

- 通用的地方：用光度学单位（lm、cd、nit）描述，不用引擎单位；朝向用 `dir + up` 两个向量；灯带也按 `AREA` 写，Godot 侧再按回路区分怎么画。
- 不够的地方：
  - `color` 是从色温算出来的 RGB，原始的色温 K 没留下。
  - 没有"可调光 / 只能开关"的字段，L6 只能开关只写在注释里（`switch_v2.py:20-21`）。
  - 没有房间/楼层，也没有引用的产品或款式。
  - `circuit` 来自集合名，不是数据本身。
  - 灯带的 `up` 写死为 `[0,1,0]`，朝向从 D 墙的坐标系推出（`export_web.py:91-97`），换成别的墙会出错。

### 场景 / 回路调光表

`SCENES` 是一个两层字典：场景 → 回路 → 0..1，外加 `screens`（`switch_v2.py:15-27`）。Blender、网页、Godot 三端都只读这一张表，切场景时对每个灯具算 `base × level(circuit)`（`switch_v2.py:32-60`；`main.gd:460-483`）。

这个结构能回答 #1 里"是否沿用按回路调光的场景"：**可以沿用**，前提是回路由上游给定（#1 已定）。需要补的是：可调光属性、场景之间的过渡（现在是瞬切）、除灯和屏幕以外的设备状态（窗帘现在是独立的开关，不在场景里，`main.gd:497-500`）。

### Godot 侧的灯光换算和 AreaLight 的替代方案

`main.gd:2-10,154-232` 的做法：

| 光源 | Godot 实现 | 能量 |
|---|---|---|
| spot（L3、L4） | `SpotLight3D`，`spot_attenuation = 2`（平方反比），`angle_attenuation = penumbra × 0.6` | `cd / 683 / π` |
| 灯盘（L6，面光） | `SpotLight3D` 89.5°、`angle_attenuation = 1`，近似余弦分布；加一块自发光 Quad 当灯罩 | `nits·w·h / 683 / π` |
| 灯带（L1、L2、L5） | 不建灯，算它照在墙上的"洗墙"图，作为加法混合的叠加层画上去 | 闭式积分 |
| 间接光 | 每回路、每 3 m 格子一对 omni（+E 进 SDFGI，-E 抵消它的直射） | 按灯具光通量 |

- **为什么不用 AreaLight3D**：注释说 Godot 4.7 的 `AreaLight3D` 衰减不像真实面板，1–3 m 内差 2–3 倍，再近更糟（`main.gd:5-6`，**仓库实测**）。
  - 官方文档：4.7 确实有 `AreaLight3D`，`area_attenuation` 默认是 `1.0`，写 `2.0` 才是物理上的平方反比；另有 `area_normalize_energy`。它比 omni/spot 更耗 GPU，在 Forward+ 里视野内只要有一盏，就会给所有物体加开销，文档建议只用在过场或高端机器上（[AreaLight3D](https://docs.godotengine.org/en/latest/classes/class_arealight3d.html)）。
  - 仓库没有记录当时测的是哪个 `area_attenuation`，所以"AreaLight3D 不准"**值得用 `area_attenuation = 2.0` 再测一次**。即使测下来是准的，30 盏级别的性能开销仍然是不用它的理由。
- **SDFGI 和 spot**：注释说 SDFGI 几乎不吃 spot 光（朝上还行，侧向约 1/4，朝下几乎没有），所以另外用 omni 对补间接光（`main.gd:235-238`，**仓库实测**）。官方文档只说 SDFGI 支持动态光、可以用 Indirect Energy 调，没写 spot 的这种限制；同时写明 SDFGI 只在 Forward+ 下可用，不跟踪动态自发光面（[Using SDFGI](https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/using_sdfgi.html)）。后一点正好解释了灯带为什么要画成叠加层，而不是靠自发光面去照亮墙。
- **对新空间**：上面这套换算只依赖 `lights.json` 里的光度学数据，**对新空间成立**。要改的只有"哪个回路是灯带、哪个是天花灯"这种按回路编号写死的判断（`main.gd:159,175,463`），应该改成按灯具类型字段判断。

### L4_BAR 等校准常数

| 常数 | 值 | 是什么 | 新空间是否成立 |
|---|---|---|---|
| `L4_BAR` | 0.75 | Blender 里 L4 壁灯的球形光源（`shadow_soft_size = 0.02`）挂在灯杆下 3 mm，一半被灯杆吞掉，Cycles 出图比流明值暗；Godot 的点光不会被埋，所以乘 0.75 对齐 Cycles 出图（按 0.65–0.8 读出） | **不成立**。这是在补 B1 的 Blender 建模细节（`build_v2.py:1027,1119`），不是物理量。Twinwise 如果直接生成 Godot，应该去掉 |
| `SPOT_OUT` | 0.01 m | spot 从灯头前 1 cm 开始发光，免得被自己的灯壳挡住 | 成立（通用的小偏移） |
| penumbra × 0.6 | — | Blender 的 `spot_blend` 映射到 Godot 的角度衰减（**仓库实测**） | 成立（引擎之间的映射） |
| `light_size` | 0.03 / 0.6 | spot 灯头 / 300×1200 灯盘的软阴影尺寸 | 部分成立；灯盘的值应该由 `w/h` 推出 |
| `omni_range` 10、格子 3 m | — | bounce 补光的范围和合并粒度 | 成立（性能上的折中，`ponytail:` 注释已标出上限） |
| `NIGHT_LUX` | 781 | B1 默认场景下通道地面的照度，用来定夜间曝光 | **不成立**，每个空间要重新算，或者改用固定 EV |
| `STRIP_EMIT/LED_EMIT` | 18.64 | 5 W/m × 80 lm/W，10 mm 面 | 不成立，应该按每条灯带自己的参数算 |
| `SCREEN_EMIT` | 0.6 | 约 400 nit 的电视 | 可以作为默认值 |
| `STILL_ASPECT` | 16/9 | 预设视角按 1920×1080 出图对齐 | 视有没有参考出图而定 |
| `TROFFER_LM` | 4000 | 按现场照片估的灯盘流明 | 不成立 |

### 行走、视角、巡游

- **环绕**：左键转、右键/中键平移、滚轮缩放，转动中心由射线打到的面决定（`main.gd:521-537,620-660`）。通用。
- **行走**：WASD，Shift 加速，左键拖动看（`main.gd:597-617,698-713`）。眼高固定 1.6 m，没有碰撞、重力和楼梯，**多层户型（#1 约束）用不了**。v1 至少要加按楼层的地面高度和墙体碰撞。
- **揭天花**：相机高于 2.75 m 就隐藏天花（`main.gd:714-722`）。单层、固定层高的假设。多层时要按"相机所在楼层以上的所有楼板和天花"来处理。
- **巡游**：通用的部分是停靠点列表 + smoothstep 插值 + 每站停留（`main.gd:556-594`）；停靠点本身是按 B1 的长墙分段算的。Twinwise 里停靠点应该来自配置（或者由房间/点位自动排出），不能写在代码里。

### 画质档位

两档，一个布尔量切换（`main.gd:502-511`）；项目级设置是阴影图集 8192、软阴影质量 4、SDFGI probe ray 4（`project.godot`）。**可以直接用**。macOS 默认"流畅"（`main.gd:69`），理由是 8 GB 的 M1 跑得动。

## 写死在 B1 展厅里的内容

运行时这一侧（`main.gd`）要改成从配置读：

- `D_SOUTH/ALONG/OUTWARD/SECTIONS`（`main.gd:24-27`），也就是巡游用的 D 墙坐标系。
- `SUN_TARGET = (8, 0, -16)`（`main.gd:19`），太阳阴影的中心。
- 入口探针的位置和朝向（`main.gd:370`），探针尺寸（`main.gd:375`）。
- 材质名 `Material_0/1/2`（地毯/地面/天花，`main.gd:117-123`），玻璃靠透明度判断。
- 网格名前缀 `灯带背_`、`屏幕背光_`、`窗帘_`、`窗帘合_`；屏幕材质 `V2_<屏幕名>`；灯带材质 `V2_灯带发光面_<回路>`（`main.gd:109,138-141,480,483`）。
- 回路语义：L1/L2/L5 是灯带，L4 乘校准系数，L6 是天花灯并且受单独开关控制（`main.gd:159,175,463,726`）。
- 眼高 1.6、C-2 层的判断 `y > -0.5`、揭天花的高度 2.75（`main.gd:602-607,715`）。
- 标题、地址、脚注（`main.gd:784,788,817`）。

生成这一侧（`build_v2.py`、`switch_v2.py`、`export_web.py`）：

- Windows 绝对路径 `E:\Louis's projects\new_showroom`（`build_v2.py:12-16`；`scenes_v2.py`、`daylight_v2.py`、`lux_calc.py` 的 `sys.path`）。
- 依赖 Ettie 外壳里的对象名（`D_wall`、`wall_10`、`floor_C2_E`、`B_dropface_*`、集合 `05_C1栏杆与B下沉`、`06_A楼梯(示意)`、`08_天花`、`09_人物比例`）（`build_v2.py:245,392-399,764-855`）。
- 所有 Figma 毫米坐标和贴图矩形（`build_v2.py:33-114`），贴图源 `tietu_1x.png` 21161×3917（`build_v2.py:33`）。
- 地理位置和日期（`switch_v2.py:73-75`）。
- 产品拆成两个文件时按名字前缀分组（`export_web.py:131`）。

## 对后续 ticket 的影响

### #6 项目文件夹和配置格式

- **灯光数据**：以 `lights.json` 的灯具条目为基础，补上：
  - `cct_k`（色温，替代或补充 `color`）
  - `dimmable`（布尔）
  - `kind`（spot / panel / strip，取代"按回路编号判断"）
  - `room` / `floor`
  - `product`，加上可选的 `variant`（款式）

  `circuit` 改成显式字段，由上游给，不再从集合名截取。
- **场景表**：沿用"场景 → 回路 → 0..1"，并把屏幕、窗帘等设备状态统一放进场景，不再各管各的。
- **时段 / 天气**：太阳和天空可以在生成时算好写进数据（参考仓库就是这么做的，`export_web.py:108-117`），也可以在 Godot 运行时算。两种做法都需要项目配置提供**纬度、经度、时区、日期、模型的正北朝向**（参考仓库假设 +Y 约等于北，`switch_v2.py:72`）。这也回答了 #1 里"日光是否需要地理位置和朝向"：需要。
- **墙面装饰 / 点位**：参考仓库用"墙面局部坐标系（沿墙 s、出墙 d、高 z，毫米）+ 贴图源上的矩形 `(x, y, w, h)`"来描述墙段、展盒、产品和屏幕（`build_v2.py:42-114`）。点位和贴图矩形的格式可以从这里起步，并把 Figma 1px = 1mm 定为贴图约定（#1 未决项之一）。
- **语义标记**：Godot 运行时要知道哪些网格是地面、天花、玻璃、窗帘的开/合态、灯带、屏幕。参考仓库靠材质名和中文名前缀识别。Twinwise 应该在配置或 glTF `extras` 里给出明确的语义标签。
- **展示元信息**：标题、地址、预设视角、巡游停靠点、反射探针的位置，都应该进配置。
- **大文件**：参考仓库把 21161 px 的贴图缩到 16000（`export_web.py:58-62`），并对小物体降贴图分辨率（`build_v2.py:1222-1244`）。这两步可以作为生成节点里固定的处理，项目文件夹只放原图。

### #7 v1 Godot 项目能力清单与验收标准

- **可以直接列入 v1 的能力**（参考仓库已在 Godot 4.7.2 上做到）：
  - 预设视角 1–8、环绕、行走、自动预览
  - 场景（F5）、时段（F6）、天气（F7）
  - 总开关、天花灯开关、窗帘开合
  - 两档画质，macOS 默认"流畅"
  - SDFGI + 反射探针
  - 控制面板和状态条

  出处：`README.md:33-43`、`main.gd:663-695,744-823`。
- **参考仓库没有、但 #1 的约束要求的能力**：
  - 多层：按楼层的眼高和地面、揭开上层的楼板和天花、上下楼。
  - 行走碰撞。
  - 产品特写，以及同一点位切换款式。
  - 温度演示。
- **验收手段**：参考仓库把"8 个视角 × 4 组状态"和 Cycles 出的备用图逐张对比（`README.md:36`；`backup_renders.py:1-3`）。如果 Twinwise 不经过 Blender（#5），就没有 Cycles 参考图，验收得换成别的标准，比如关键点的照度和流明守恒检查、截图回归。
- **性能基线**：参考仓库的规模是约 70 条灯具数据，加上 29 个灯盘软阴影；"流畅"档面向 8 GB 的 M1（`main.gd:69,502-503`）。可以作为 v1 性能验收的起点。

### #10 设备种类与行为

- **参考仓库里有行为的设备只有这几类**：
  - 灯：按回路 0..1 调光，L6 只能开关。
  - 屏幕：自发光开关，亮度固定，画面是 Figma 的占位色。
  - 窗帘：两套网格切换显隐。

  出处：`switch_v2.py:15-69`、`build_v2.py:440-451`。
- **只摆了模型、没有行为的设备**：开关面板、温控器、HomeGlow 屏、CoreBox、对讲机、Sonos（`build_v2.py:64-105,499-507,653-665,691-724`）。温控器全都是静态模型，**参考仓库里没有任何温度演示可以复用**。
- `agent.js` 里出现的音箱分区、摄像头视锥、区域划分、到访流程（`agent.js:15-37,56-90`），是"展厅 Agent"方案的演示，不是设备仿真。要不要把这类设备放进 v1，需要在 #10 里明确取舍。
- **产品资产的修补**（盖板玻璃、温控屏、竖装 UI，`build_v2.py:533-583`）说明：同一个产品不同的安装方式（横装/竖装）可能需要不同的 UI 贴图。这一点影响 #3 的产品库约定：款式之外还可能有安装朝向这个维度。

### 其他（#5 生成路线）

- 参考仓库走 Blender 的主要收益：Cycles 参考图和照度验算，程序化材质烘焙，glTF 导出。
- 代价：
  - 要绕开 glTF 不支持面光的问题，另写 `lights.json`（`export_web.py:68`）。
  - Godot 导入时会丢透射材质（`import_web.py:2-3`）。
  - 产生 `L4_BAR` 这类"让 Godot 对齐 Cycles"的校准常数。

  这些对比留给 #5。

## Blender 单位

`build_v2.py:1010` 写着"Blender 的 spot 功率按全向点光计"，`main.gd:4` 写着"Blender 的 1 W = 683 lm"。Blender 5.2 手册的 [Light Objects](https://docs.blender.org/manual/en/latest/render/lights/light_object.html) 页只写了：点光、spot、面光的功率单位是瓦（辐射功率，不是电功率）；开启 Normalize 时，太阳强度的单位是 W/m²。这两条说法手册里都**没有**。它们只影响"要不要经过 Blender"的那条路线；如果 Twinwise 直接生成 Godot，可以绕开。
