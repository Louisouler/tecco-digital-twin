# showtime-wowdemo（feat/house-v1）运行时能力与验收手段调研

> 用于 Twinwise ticket #7「v1 Godot 项目能力清单与验收标准」。参考仓库只读。
> 路径约定：`main.gd` 等指 `showroom/scripts/` 下的文件，`docs/…` 指仓库根下的 docs。行号以 feat/house-v1 当前检出为准。
> 对照基准是 #7 已定的 v1 要求：内部演示；Mac 上运行，可一键导出 macOS 应用；8GB M1「流畅」档稳定 30 fps；键盘 + 控制面板；楼层切换为 1F/2F/全部（选中层以上隐藏，揭开本层天花，行走时自动跟随楼层，「全部」显示外观）；行走有碰撞、重力，能走楼梯；门可交互（默认关，按 E 或点击开关）；能跳转到指定房间和楼层；产品特写和款式切换照搬 wowdemo（图钉、抽屉棚拍、单品换款、全屋换系列、整套方案）。

---

## 0. 结论速览

1. 运行时能力很多，但它是按**触屏展台 + Hub 后端**设计的：所有设备、场景、温度、人员、安防状态都来自独立的 Node Hub 进程（`main.gd:2-3`，`README.md` 接口一节）。Twinwise v1 是单机 macOS 应用，场景执行和状态模拟要搬进 Godot，或者随包带上。
2. 和 v1 要求差得最多的四处：
   - 楼层切换只显示选中层（上下都隐藏）；
   - 行走没有真正的重力；
   - 室内门默认开着、不能交互，外门只能点击、没有 E 键，E 被自由视角占用；
   - 没有画质档位，也没有导出配置。
   另外，「整套方案」在 wowdemo 里**没有实现**：`SCHEMES` 常量定义了但没有任何地方引用。
3. 性能只在 Windows RTX 3060（1080p）上测过，没有 Mac 或 M1 的数据。最新一轮各站 51–60 fps，Godot 显存 3.6–4.2 GB，可见灯 145–636 盏。按这个负载，8GB M1 达到 30 fps 没有依据，需要一个专门的低档。
4. 验收手段大多可以直接拿来用：
   - 固定路线性能巡检（CSV）；
   - 行走可达性和楼梯卡点检查；
   - 闪烁检测；
   - 设备和场景的「改变可见」像素检测；
   - 浮空、遮挡、照度覆盖审计；
   - 脚本加载检查。
   它的「视觉达标」最终仍靠人看对照图，加上和 Cycles 参考图做亮度比对，没有带阈值的截图回归，也没有 CI。
5. 写死的内容：`main.gd` 7235 行里，按函数粗分，通用运行时约 40%，这套房子的逐条修补约 24%，测试和审计钩子约 14%，Hub/Agent/安防/能源/APP 集成约 21%。另有 32 个、共 1888 行的 `*_finish/*_details/*_repairs` 小脚本，全部针对这套房子。通用部分已经有 `house.json` 契约和 `<room>__<category>__<name>` 命名约定做接缝，可以抽成数据驱动的运行时。

---

## 1. 运行时能力对照

标记：✅ 一致；⚠️ 有差异；❌ 缺失；— 不在 v1 范围或 v1 未定。

### 1.1 视角与相机

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 三种相机模式 | `overview`：按视图自动取景的鸟瞰；`free`：环绕、平移、缩放；`walkin`：第一人称。界面上只有 free / walkin 两个按钮，overview 由跳转房间触发 | `main.gd:2986-3015`、`main.gd:5265`、`main.gd:3017-3039` | ✅ 鸟瞰 + 行走都有 |
| 自由视角操作 | 左键拖动（模拟触屏）转视角；右键或中键平移；滚轮或触控板捏合缩放（6–90 m）；WASD 平移，Q/E 下降/上升，Shift 加速 | `main.gd:3196-3238`、`main.gd:4197-4223`、`project.godot:17` | ⚠️ 可用，但 **E 被占作「上升」**，和 v1 的「按 E 开门」冲突 |
| 相机过渡 | 1.3 s 三次缓动飞行；巡游时放慢一倍 | `main.gd:2501-2516` | ✅ |
| FOV | 鸟瞰 40°，行走 62° | `main.gd:3006`、`main.gd:3013` | ✅ |
| 跟随变化 | 设备变化时相机飞去框住相关房间，过后飞回 | `main.gd:3111-3135` | — 依赖 Hub 事件 |

### 1.2 楼层切换

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 楼层按钮 | Basement / Ground / Upper / Roof 四档，固定 b1/gf/ff/rf | `main.gd:5219`、`main.gd:45` | ⚠️ 层数写死为 3 层加屋顶；v1 要支持任意层数 |
| 选中层显示规则 | 鸟瞰时**只画选中层**（`f == focus`），上下层都隐藏。这是 Joe 09-26 的要求（「Show only the selected floor」） | `main.gd:2435-2436`、`docs/reviews/2026-09-26-joe-feedback-checklist.md:8` | ⚠️ v1 是「选中层**以上**隐藏」，下层保留 |
| 揭开本层天花 | 一层的天花就是上一层的楼板（`shell.ts:93-104`），上层隐藏后天花自然揭开。屋顶只留阴影，室内仍像在屋顶下受光；天花上的灯带、筒灯、传感器等吊顶件在鸟瞰时也隐藏 | `main.gd:2443-2452`、`main.gd:1847-1897` | ✅ 做法可以直接沿用（「只投影」很关键） |
| 「全部」 | Roof 档（`focus == "rf"`）：所有层、屋顶、外立面都画 | `main.gd:2436`、`main.gd:2453-2454` | ✅ 语义接近 v1 的「全部显示外观」，只是名字不同 |
| 自由视角进入室内 | 相机进入房子包围盒后，屋顶自动盖上。**包围盒坐标写死** | `main.gd:2423-2428` | ⚠️ 需要改为从户型计算 |
| 行走时楼层 | 行走时整栋都画（`_walk_drawn`），按高度阈值只裁地下室和地上的内饰。**楼层按钮不跟随**：`_track_room` 换房间时不调用 `set_floor`，`focus` 一直停在原层。行走中手动切层会被传送到该层的大厅（`HALLS` 写死） | `main.gd:2432-2433`、`main.gd:1746-1764`、`main.gd:1705-1744`、`main.gd:3065-3067`、`main.gd:70` | ⚠️ v1 要求行走时自动跟随楼层：楼层判定要按实际楼层标高表来做，并更新 `focus` 和楼层按钮 |
| 楼层判定 | 高度阈值 `-1.2 / 1.8` 写死；房间所属楼层按名字前缀 `FF`/`B1` 推断 | `main.gd:1717`、`main.gd:1748`、`main.gd:1060` | ⚠️ 写死 |

### 1.3 行走、碰撞、重力、楼梯

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 行走体 | `CharacterBody3D`，`MOTION_MODE_FLOATING`，胶囊半径 0.22 m、高 1.3 m；眼高 1.6 m | `main.gd:1505-1517`、`main.gd:69` | ✅ |
| 键盘移动 | WASD 或方向键，1.6 m/s；Shift ×2.5；转视角靠左键拖动；APP 摇杆也能驱动 | `main.gd:1600-1631` | ✅ 键盘可用。v1 可考虑加鼠标锁定视角（现在是拖动） |
| 碰撞 | 外壳网格生成 trimesh 碰撞；家具小件用盒子，大件或环绕件用 trimesh；地毯、窗帘、灯、门不碰撞；车道和坡道按节点名前缀加碰撞 | `main.gd:1447-1519` | ✅ 规则通用，靠命名约定 |
| 重力 | **没有真正的重力**。每个物理帧从胶囊向下打射线，把身体插值到地面上方 0.95 m；射线没打到地面就原地不动，不会下落 | `main.gd:1779-1798` | ⚠️ v1 要求有重力，需要补落体，或者至少在没有地面时下落 |
| 台阶和楼梯 | 胶囊离地 0.3 m 浮着，踏步顶不住它；被墙挡住时依次试 0.2/0.32/0.45 m 抬升。螺旋楼梯：外壳原楼梯隐藏但保留碰撞，显示 Mac 重做的模型 | `main.gd:1763-1777`、`main.gd:3435-3450` | ✅ 能走楼梯，`walk-check.sh` 验过上下行零卡点（见 §2） |
| 行走平滑 | 视角输入指数平滑；APP 的网络包用抖动缓冲；眼睛位置在两个物理帧之间插值 | `main.gd:1647-1703` | ✅ 可沿用 |
| 进入时脱困 | 起点和家具重叠时，按环形搜索最近的空位 | `main.gd:1566-1590` | ✅ |
| 电梯 | 点击厅门或呼叫面板叫梯：轿厢瞬移到目标层，门滑开 7 s。**人不能乘坐**；`STOPS` 高度写死 | `lift.gd:1-12`、`lift.gd:74-112` | — v1 未提；做的话要改成数据驱动 |

### 1.4 门

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 默认状态 | 外壳生成时，**室内门开成直角靠墙**，外门和车库门关着 | `shell/src/shell.ts:127-129` | ⚠️ v1 是默认关 |
| 交互 | 只有 `type == "external"` 的门能点击开关：转 90° 朝向室内，1.2 s 补间；没有键盘触发 | `main.gd:3240-3295`（判定在 `:3256`）、`main.gd:3547` | ⚠️ v1 要求所有门可交互、E 或点击；E 键还和自由视角冲突（`main.gd:4210`） |
| 自动门 | 游戏厅玻璃推拉门：相机进入 2.2 m 内自动滑开。**坐标写死** | `main.gd:4134-4136`、`main.gd:3297-3324`、`main.gd:3452-3467` | — 写死 |
| 门和家具冲突 | 门扇开到会撞家具时，自动换角度 | `main.gd:1523-1556` | ✅ 思路可沿用（v1 默认关门时也要检查开门扫掠） |
| 特定门修补 | 入户门换模型、单锁内装、侧门把手反向等，都按节点名写死 | `main.gd:3297-3433` | 不可复用 |

### 1.5 房间跳转

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 房间列表 | 左侧面板：楼层按钮下面是该层房间列表。列表是**人工挑选**的（`FLOOR_NAV`、`ROOM_NAV`、`VIEWS`），不是全部房间 | `main.gd:2933-2965`、`showroom_data.gd:7-30`、`showroom_data.gd:83-93` | ⚠️ 机制可用，列表要改成从户型自动生成 |
| 任意房间进入 | `"@<roomId>"` 通用路径：从主门洞往里走 1.3 m，朝能看到的最深处看（`_entry_view` 只读 `house.json` 的房间多边形、门洞和家具），然后切到行走 | `main.gd:2611-2720`、`main.gd:2721-2766`、`main.gd:2967-2984` | ✅ 通用，可直接用 |
| 鸟瞰点房间 | 点一下框住房间，8 s 内再点进入行走 | `main.gd:3555-3587` | ✅ |
| 站位点 | 每个房间 1–3 个调好的站位（`viewpoints.json`：47 个房间、66 个视点，其中 20 个由 Joe 按 F7 手工固定，23 个是自动评分），没有调过的房间自动生成。F7 存当前视点 | `booth.gd:2-6`、`booth.gd:50-70`、`main.gd:6565-6593`、`data/viewpoints.json` | ✅ 自动选视点（`_best_views`，`main.gd:7103-7228`）通用。手工视点在 Twinwise 里只能作为输入覆盖 |

### 1.6 巡游与待机

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 巡游 | 4 条路线（`arrive/day/security/comfort`），JSON 写步骤：view、time、tap 场景、person 移动、caption、hold。触摸接管，20 s 后继续 | `tours.gd:1-12`、`data/tours/*.json` | — v1 未要求。格式可借鉴，但内容引用了写死的房间、场景和人物 |
| 待机轮播 | 90 s 无操作后播放巡游的精彩片段；三指长按进员工模式 | `idle.gd:2-5`、`main.gd:347-351` | — 展台专用 |

### 1.7 日夜与日照

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 太阳位置 | NOAA 近似公式，**经纬度写死为悉尼**（-33.87, 151.21），时区按悉尼夏令时规则写死；日期取系统日期 | `sun_sim.gd:7-8`、`sun_sim.gd:12-38`、`main.gd:5790-5823` | ⚠️ 算法可复用，地理位置和朝向要来自输入（和 #4 的结论一致） |
| 时间控制 | `set_time(h)`：太阳方向、强度、色温、天空亮度随高度角变化；高度角低于 -4° 进入夜景。T 键扫一天，工具栏按钮跳时段；场景可以带上自己的时刻（`SCENE_TIME`） | `main.gd:5790-5837`、`main.gd:3173`、`main.gd:5232` | ✅ |
| 夜景 | 天空和环境光降到 0.02–0.03，SDFGI 只剩灯光反弹；入夜时如果室外灯全关，自动跑 Evening 场景；路灯 | `main.gd:3589-3625`、`main.gd:3627-3673` | ✅ 机制可用；自动场景的 ID 写死 |
| 天光 | 天窗光柱随太阳高度和遮阳状态变化；天窗与遮阳实体的对照表写死 | `main.gd:3469-3503` | ⚠️ 写死 |
| 环境光照 | Forward+、AgX、SDFGI（含遮挡）、SSAO、SSIL、SSR、glow、薄雾、盒投影反射探针、平面镜（最多 2 面实时） | `main.gd:589-657`、`main.gd:930-967`、`mirrors.gd:1-10` | ✅ 和 #5 的「SDFGI + 反射探针」路线一致 |

### 1.8 天气

grep 不到 weather、rain、cloud 之类的实现，**没有天气**。❌（v1 没明确要天气；#1 地图的「Not yet specified」里提到「场景、时段、天气怎么定义」。）

### 1.9 灯光与场景

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 分层灯光 | 每个房间最多四层（筒灯、灯带、线性灯、吊灯），按回路名称归层；灯具网格按 `<room>__lamp__<layer>__…` 命名，自己发光；支持 `color_temp_kelvin` | `lighting.gd:1-13`、`lighting.gd:724-725` | ✅ 和 #4 的「lights.json + 场景调光表」吻合 |
| 场景执行 | 场景定义在 `house.json`，**由 Hub 执行**并推送 delta。大屏底部 dock 分两排：全屋场景（顺序写死）和当前房间的场景 | `ui/dock.gd:2-11`、`ui/dock.gd:124`、`main.gd:3762-3956` | ⚠️ v1 是单机应用，场景执行要放进 Godot |
| 场景配套 | 每个场景有自己的时刻、观看视角、淡入时长（`SCENE_TIME/SCENE_VIEW/SCENE_FADE`，按场景 ID 写死） | `main.gd:5822-5836` | ⚠️ 写死，可以变成场景数据的字段 |
| 灯光性能策略 | 只亮行走所在房间的灯（scope）；最近 12 盏灯投影，其中最多 2 盏 omni；灯带和灯槽不进 SDFGI | `lighting.gd:838-884`、`docs/reports/2026-09-28-lighting-perf.md:24-27` | ✅ 应该写进 v1 的渲染规则 |

### 1.10 温度、能源、人员、安防

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 温度热力图 | 每个房间以探头（开关、面板、存在传感器）做反距离插值的平滑场，每层一张；房间温度来自 Hub 的集总热模型 | `heatmap.gd:1-10`、`heatmap.gd:157-158`、`heatmap.gd:205`、`hub/src/sim/thermal.ts:1-15` | — v1 温度演示「由哪些设备驱动」还没定。热力图本身可复用，热模型要移进 Godot 或预先算好 |
| 设备状态可视化 | 空调出风、地暖发光、风扇、排气、电视升降、桑拿、泳池加热 | `states3d.gd:2-11` | ✅ 可复用，触发依赖设备类型 |
| 能源看板 | 5 个 KPI、功率流图、当天曲线。数据是 Hub 回放的一户真实 Homira 家庭数据 | `ui/energy_sheet.gd:2-4` | — v1 未提 |
| 人员 | 4 个家庭成员和访客的真人模型（idle/walk 动画），位置来自 Hub；自动找座位坐下 | `people3d.gd:1-10`、`people3d.gd:52-55` | — v1 未提 |
| 安防 | CCTV 视锥、预生成的监控短片、Theatre 大画面、对讲、布防撤防标记。摄像头表写死 | `cones.gd:1-3`、`ui/feed.gd:1-5`、`ui/theatre.gd:1-3`、`main.gd:4297-4348`、`main.gd:6169-6238`、`showroom_data.gd:66-71` | — v1 未提 |
| 存在感应 | 天花圆片；有人时地面出现金色呼吸区域 | `sensors3d.gd:1-6` | — |

### 1.11 画质档位

| 能力 | wowdemo 实现 | 出处 | 对照 v1 |
|---|---|---|---|
| 档位 | **没有档位**，只有调试参数：`--envoff=ssil,ssr,sdfgi,ssao,glow,fog`、`--shadowmax`、`--lodthr`、`--fsr`、`--aa` | `main.gd:194-215` | ❌ v1 至少要有「流畅」档（8GB M1） |
| 动态分辨率 | 室内帧率连续 2 s 低于 55 时，切到 FSR 2 82%。展台启动时用 `--nodynres --aa=fsr2` 关掉了，因为 Joe 要画质 | `main.gd:6633-6665`、`tools/booth/start-booth.cmd:13` | ⚠️ 可作为「流畅」档的手段之一 |
| 抗锯齿 | F9 在三种方案间轮换：MSAA2+FXAA、FSR2 原生、MSAA4 | `main.gd:3182`、`main.gd:6594-6631` | ✅ |
| 项目渲染设置 | Forward+；方向光阴影 4096；点光阴影图集 8192；反射图集 48×512；各向异性 4；遮挡剔除开启；贴图导入走显存压缩 | `project.godot:19-40` | ⚠️ 对 8GB M1 偏重 |
| 其他裁剪 | 契约生成的遮挡体（`occluders.gd:2-6`）；小物件可见距离（`main.gd:1426`）；网格 LOD 阈值（`main.gd:210`） | — | ✅ 可复用 |

### 1.12 UI 结构

| 区域 | 内容 | 出处 |
|---|---|---|
| 顶行 | 品牌、相机模式（free/walkin）、楼层、工具按钮（款式方案、时段、跟随、巡游） | `main.gd:5185-5300` |
| 左侧 | 楼层按钮下面是该层房间列表，可滚动 | `main.gd:2933-2965` |
| 底部 dock | 全屋场景 + 房间场景，下面一条可点击的实时状态条（人员、安防、灯、空调、能源） | `ui/dock.gd:2-3` |
| 右侧抽屉 | 有变化或点功能时弹出，变化类的抽屉会自动隐藏；房间设备磁贴、产品、安防等都用它 | `ui/drawer.gd:2-3`、`main.gd:2805-2920` |
| 场景内 | 状态徽章（贴在房间上）、产品图钉、toast、站位圆盘和门箭头 | `ui/badges.gd:2-4`、`ui/hotspots.gd:2-5`、`booth.gd:2-4` |
| 其他 | 「Try a moment」面板（Moments/Tours 两个页签）、Agent 对话卡、Theatre、能源 sheet | `main.gd:5349`、`main.gd:4225-4277` |

对照 v1：⚠️ 整体按 1920×1080 触屏设计（`project.godot:11-17`、`ui/drawer.gd:10-12` 写死像素位置）。v1 的「键盘 + 控制面板」可以沿用这套布局，但要去掉 Agent、APP、安防、能源等不在 v1 范围的部分，并补上键盘快捷键（开门 E、切层、跳房间）。

### 1.13 产品特写与款式切换（v1 照搬项逐条核对）

| v1 照搬项 | wowdemo 实现 | 出处 | 结论 |
|---|---|---|---|
| 图钉 | 墙上设备上方一根细杆加一个小图钉，显示当前款式；人靠近或打开时让开 | `ui/hotspots.gd:2-5`、`main.gd:287-301` | ✅ 存在。产品表写死 |
| 抽屉棚拍 | 相机飞到设备前 2.4 m；右侧抽屉里用产品真实模型实时渲染棚拍图：透明背景，主光、补光、轮廓光加一个中性天空 | `main.gd:4721-4782`、`ui/product_shot.gd:2-6` | ✅ 存在，可复用 |
| 单品换款 | 抽屉里的款式按钮：换开关款式时**全屋同系列开关一起换**，换 10 寸屏款式时所有主面板一起换 | `main.gd:4754-4767` | ✅ 存在。注意：它是「全屋同类一起换」，不是「只换这一个点位」 |
| 全屋换系列 | 「Interior schemes」抽屉：每个系列一张卡片（Luna/Flux/Levi/Breeze），点色点就把全屋开关换成该系列该款式，温控器跟着换 | `main.gd:4958-5028`、`devices3d.gd:119-130` | ✅ 存在。系列表写死 |
| 整套方案 | `SCHEMES` 定义了 3 套组合（heritage/modern/soft），**任何地方都没有引用**；抽屉底部写着「Whole-house interior styles are coming next.」 | `showroom_data.gd:59-63`、`main.gd:5027` | ❌ **wowdemo 没有实现，没法照搬**，要新做 |

### 1.14 运行形态

| 项 | wowdemo | 出处 | 对照 v1 |
|---|---|---|---|
| 后端 | 必须连 Hub（Node 22，HTTP/WS/MQTT，约 1600 行 TS）。没连上时状态显示 CONNECTING，设备状态全部为空 | `README.md`、`main.gd:3675-3679`、`hub/src/` | ⚠️ v1 是单机，要内置 |
| 平台 | 展台是 Windows，用 Godot 编辑器的 exe 加 `--path` 运行；更新靠 git pull 后重新导入 | `tools/booth/start-booth.cmd`、`main.gd:3150-3157` | ❌ 没有 `export_presets.cfg`，没有导出流程 |
| 资产 | `interiors/`、`outdoor/`、`cctv/` 不进 git，另行同步 | `README.md`（大文件一段） | — 对应 #1 待定的「项目文件夹存在哪」 |

---

## 2. 性能实测与验收手段

### 2.1 实测平台

- **只在 Windows RTX 3060 12 GB、1920×1080 窗口上测过**（`docs/perf/LOG.md:10`）。
- 规格里的目标机是 RTX 2080 8 GB，「待 Joe 在目标机上跑」，没有数据（`docs/perf/LOG.md:9-10`）。展台后来是 RTX 2070 跑 4K（`main.gd:6594` 注释），也没有 perftour 数据。
- Joe 09-28 在 Mac 上走查过一次（`docs/tasks/L3-29-joe-0928-mac-walkthrough-models.md:3`），只留下截图和问题，**没有任何 Mac 或 Apple Silicon 的帧率、显存记录**。
- 规格里的预算：2080 上 4K 目标 60 fps，最低 30 fps；显存 ≤ 6 GB；切房间没有可感知的停顿（`docs/specs/2026-09-25-wowdemo-architecture.md:181`、`:283`）。

### 2.2 数据（3060，1080p）

| 阶段 | 帧率 | 显存 / 负载 | 出处 |
|---|---|---|---|
| 09-26 基线（贴图未压缩） | 锁 60 | Godot 6.3 GB，其中纹理 6.06 GB；整卡峰值 7.65 GB | `docs/perf/LOG.md:12-40` |
| 贴图改显存压缩后 | 锁 60 | Godot 1.8 GB；整卡 3.05 GB | `docs/perf/LOG.md:42-68` |
| 09-27 写实化一轮（realism1） | 行走客厅 39、夜间 38、露台夜间 43；最差帧 130–150 ms | Godot 4.1 GB；可见灯最多 684 盏 | `docs/perf/LOG.md:182-200` |
| 09-27 加动态分辨率（LOG 最后一轮） | 鸟瞰 58.8–60；行走客厅日间 51.4、夜间 53.1；最差帧 25–83 ms | Godot 3.6–4.2 GB（纹理 3.2–3.6 GB）；draw call 1300–5100；图元最多 2.9 M；可见灯 145–636 | `docs/perf/LOG.md:502-520` |
| 09-28 灯光调优后 | 鸟瞰 57–60；行走 58–60；客厅夜间 51–52 | — | `docs/reports/2026-09-28-lighting-perf.md:14-27` |
| 10-05 Codex 定点采样（关 vsync，120 帧） | p50 13–18 ms；二楼全开时进浴室约 46 fps。报告原话「Do not promise locked 60 FPS or zero stutter」 | — | `docs/reports/codex-lighting-performance-2026-10-05/README.md`、`performance.csv` |

**瓶颈**（按文档的归因）：

1. 贴图显存：521 张贴图无损导入，约 6 GB。改成 S3TC/BPTC 后降到约 1.6 GB（`docs/perf/LOG.md:34-35`、`:62-64`）。
2. 着色器首次编译卡顿：切进房间时 89–134 ms（`docs/perf/LOG.md:66`、`:92`）。
3. 灯和阴影：36 张阴影图让客厅掉到 24 fps，后来限制为 12 张、omni 最多 2 张；作用域外的灯直接隐藏（`docs/reports/2026-09-28-lighting-perf.md:24-27`）。
4. 通高客厅：一眼看到三个房间，是全程最慢的一站（同上）。
5. SDFGI 移动后要重新收敛，日间行走的帧率每次跑都不一样（同上:28）。
6. 镜面反射：一次只开最近的一面，分辨率上限 768×512（Codex 10-05 README）。

**对 8GB M1 的推断（未验证）**：

- wowdemo 当前负载是 Godot 显存 3.6–4.2 GB、可见灯数百盏、draw call 数千。M1 的 8 GB 是 CPU 和 GPU 共用的。按这个负载，30 fps 没有依据。
- 「流畅」档至少要考虑：
  - 关掉 SSIL/SSR，降低 SDFGI 级联，或者干脆不用 SDFGI；
  - 阴影图数量降到个位数；
  - 灯光作用域收紧到当前房间；
  - FSR 2 降分辨率；
  - 降贴图分辨率（mip 偏移）；
  - 启动时预热着色器。
- 这些都要先在真机上跑同一条巡检路线验证。

### 2.3 验收手段清单

所有工具都通过 `main.gd` 的命令行参数驱动 Godot，跑完自动退出。参数共 122 个（`grep args.has/get` 去重），参数说明在 `main.gd:7-11`，挂载点在 `main.gd:422-586`。

| 手段 | 做法 | 输出 / 判据 | 出处 |
|---|---|---|---|
| **性能巡检** | `--perftour`：固定 12 站（4 个鸟瞰、5 个房间行走、3 个夜景），每站稳定 2 s，再采 6 s；同时用 nvidia-smi 每秒采一次 | CSV 字段：fps、最差帧、Godot 显存、纹理、缓冲、draw call、物体数、图元、可见灯、节点、静态内存；摘要追加到 LOG.md | `perf.gd:1-89`、`main.gd:5751-5788`、`showroom/tools/perf-tour.sh` |
| **行走验收** | `walk-check.sh` 两部分。① 楼梯：按户型算出螺旋路线，用 `--walkpath` 自动走，8 s 内到不了的点记为 STUCK。② 可达性：每层 0.15 m 网格测站立点，从入口和楼梯口泛洪，每个房间判 ok / CUT-OFF / UNREACHABLE / NO-STANDING-ROOM，输出彩色网格图。附带三项：家具挡住墙上设备、门扇撞家具、外窗没有窗帘 | 末行 `VERDICT: stairs stuck N · indoor rooms unreachable N · placement clashes N · rooms without an entry view N · outward windows without a blind N` | `showroom/tools/walk-check.sh`、`reach.gd:1-235`、`main.gd:1800-1813`、`main.gd:479-506` |
| **闪烁检测** | 24 个视角，每个视角相机不动、连拍 6 帧；相邻两帧亮度差超过 40 的像素数大于 150 就判闪烁（z-fighting、阴影抖动） | 每个视角输出像素数；红色叠加图 | `showroom/tools/flicker-check.sh`、`main.gd:5731-5748`（`--seq`） |
| **改变可见检测** | `--effects`：每个设备、每个场景，分别在行走和鸟瞰视角拍执行前后两张图，算变化像素占比，低于 0.8% 判「看不出来」 | `report.json` + 弱项对照拼图 | `main.gd:6039-6117`、`tools/effects-report.py` |
| **控制可达** | `--controltest`：把每个磁贴的点击按真实请求发给 Hub，检查状态是否真的变了；再把所有场景过一遍 | FAIL 行排在最前 | `main.gd:6739-6789` |
| **浮空检测** | `--floatcheck`：每件家具底部 9 个点向下打射线，离支撑面超过 3 cm、又不属于悬挂类或贴墙件的，判浮空 | `floats.txt` | `main.gd:6659-6737`、`tools/float-check.py`（离线读 GLB 的版本） |
| **设备审计** | `--devaudit`：在每个墙上设备前 0.6 m 往回打射线，看是否被挡；再看背后有没有墙，没有就判浮空 | `DEV … blocked by / FLOATING` | `main.gd:507-543` |
| **设备特写** | `--shots` 加 `{"devices":true}`：每个设备从正面、更近、左右四个角度拍，四个角度都被挡才判 BLOCKED | `devices.txt` + 图 | `main.gd:5578-5616`、`main.gd:5631-5660` |
| **全屋巡拍** | `--shots` 加 `{"room":id,"auto":n}`：每个房间自动选 n 个可站点，按北东南西各拍一张；`review-shots.py` 每间房的四个角各拍一张，出接触印样，人工逐张看 | PNG + `sheet-*.jpg` | `main.gd:5555-5680`、`tools/review-shots.py` |
| **照度覆盖** | `--coverage`：全部灯开到最亮，按真实衰减算地面照度，输出每个房间最暗 3% 和中位数的比值；`--ceilings` 拍天花有没有黑影；`--roomlights` 逐房间单独开灯，查漏光 | 每层照度图 + 数值 | `main.gd:7004-7036`、`main.gd:7038-7094`、`main.gd:6967-7002` |
| **光源审计** | `--lightaudit`：没有灯具的光、没人控制的自发光 | `LA …` 行 | `main.gd:472-476`、`lighting.gd` 的 `audit` |
| **脚本加载** | headless `--quit`，抓 SCRIPT ERROR 和 Parse Error | 退出码 1 表示失败 | `tools/check-scripts.sh` |
| **单帧截图** | `--capture --wait --noui [--view --floor --night --campose …]` | PNG | `showroom/tools/capture.sh`、`main.gd:5731-5748` |
| **Cycles 参考对比** | Mac 在 Blender Cycles 里按大屏相机的同一机位渲染 4 个房间的日景和夜景；`LOOK` 参数照着调，结果是 4 个房间里有 3 个亮度落在 Cycles 的 ±20% 以内 | 人工比对 + 亮度数值 | `docs/tasks/L3-30-joe-0928-2nd-walkthrough.md:12-26`、`main.gd:101-104` |
| **前后对比证据包** | Codex 每轮交付：前后 jpg、`pixel-difference.json`、关键文件哈希保全检查、`review.html` | — | `docs/reports/codex-*-2026-10-0*/` |
| **人工走查清单** | Joe 截图标问题，汇总成逐条清单，每条标 ✅（截图确认）或 🔧（已改未确认） | — | `docs/reviews/2026-09-26-joe-feedback-checklist.md:1-4` |

**它没有的东西**：

- 没有带阈值的截图回归（golden image）；
- 没有 CI：`npm test` 只覆盖契约、Hub 和外壳（`package.json:6`），仓库里也没有 CI 配置；
- 所有 Godot 工具都写死了 WSL 下的 Windows 路径（`WOWDEMO_WIN`、`GODOT` 指向 `C:/temp/...`），要改造才能在 Mac 上跑。

### 2.4 哪些可以直接变成 Twinwise 的验收标准

Twinwise 没有离线渲染图可以逐张对比（#7 原文），所以优先用**和参考图无关、可以自动判定**的检查。

| 建议的 v1 验收项 | 来源手段 | 建议判据（待 grilling 确认） |
|---|---|---|
| 性能 | perftour：固定路线，覆盖每层鸟瞰、「全部」、每层至少 2 个房间行走、夜景、楼梯 | 8GB M1、「流畅」档、1080p：每站平均 ≥ 30 fps，p95 帧时间 ≤ 33 ms；进程内存（或 Godot 显存）给出上限，比如统一内存占用 ≤ 5 GB（具体数值要先在 M1 上跑基线）。数据采集要把 nvidia-smi 换成 macOS 的工具 |
| 首次切房间卡顿 | perftour 的 `frame_ms_worst` | 单帧 ≤ 100 ms（需要预热着色器） |
| 行走可达 | walk-check：可达性网格 + 楼梯自动行走 | 室内房间 UNREACHABLE / CUT-OFF 为 0；每条楼梯上下行 STUCK 为 0 |
| 门 | 新写：每扇门自动按 E 或点击，检查开到位、关回原位，开门扫掠区没有家具 | 失败数 0 |
| 跨楼层跟随 | 新写：走楼梯时记录 focus 的变化 | 到达上层后 1 s 内楼层按钮切换 |
| 楼层显示 | `--floorseq` 截图 + 新写的可见性断言 | 选 n 层时，所有标高高于 n 层的节点不可见、n 层及以下可见；「全部」时外观可见 |
| 闪烁 | flicker-check | 每个视角闪烁像素 ≤ 150 |
| 浮空和穿插 | floatcheck、devaudit、placement | 0，或者只剩白名单 |
| 设备和场景可见 | effects 前后对比 | 每个设备、每个场景，至少一个视角的变化像素 ≥ 0.8% |
| 照度 | coverage | 每个房间最暗 3% 与中位数的比值 ≥ 某个值（数值待定） |
| 灯光一致性 | lightaudit | 没有灯具的光、没人控制的自发光都为 0 |
| 款式切换 | 新写：遍历产品库每个款式，换款前后截图并检查模型文件已替换 | 每个款式都能切换，切换后的截图和切换前不同 |
| 可构建 | check-scripts + 无头导出 macOS | 脚本加载没有错误；能导出 .app 并在 M1 上启动 |
| 确定性 | 同一份输入生成两次 | 生成的项目文件（不含导入缓存）哈希一致（对应 #1「同样的输入产物一致」） |
| 「展厅级」观感 | 自动巡拍接触印样 + 人工评审 | 人工签字。可选：拿 Twinwise 的产品渲染或网站模型截图做亮度和色彩分布的粗比对，不要求逐像素一致 |

---

## 3. 写死内容的量化和抽取可行性

### 3.1 量化

**按函数粗分 `main.gd`**：用 awk 切出每个函数的行数，再按函数名人工归类，共 7086 行在函数里。归类有主观成分，误差约 ±5%。

| 类别 | 行数 | 占比 | 代表函数 |
|---|---|---|---|
| 通用运行时 | ~2843 | ~40% | 相机、楼层、行走、碰撞、门、环境、UI、日照。其中 `_ready` 439 行里大约一半是测试参数挂载 |
| 这套房子的逐条修补 | ~1733 | ~24% | `_tame_material`、`_bend_monitor`、`_pool_*`、`_apply_garage`、`_apply_cinema`、`_apply_tv`、`_apply_gates`、`_mac_doors`、`_mac_stairs`、`_entry_lock_inside`、`_side_door_levers`、`_street_lights`、`_apply_skylights`、`_move_showers`、`_build_equipment` 等 |
| 测试和审计钩子 | ~995 | ~14% | `_shots`、`_perf_tour`、`_float_check`、`_control_test`、`_ui_walk`、`_coverage_maps`、`_ceiling_shots`、`_best_views` 等 |
| Hub、Agent、安防、能源、APP 集成 | ~1515 | ~21% | `_on_changes`、`_on_security`、`open_fn`、`open_player`、`_agent_*`、`_app_fly_in`、`_follow` 等 |

**全部脚本的标记计数**：

- 含「Joe」的注释行：294 行，其中 `main.gd` 147 行。几乎每条都对应一次针对这套房子的视觉或交互修补。
- 含房间 ID（`GF-11`、`B1-X1` 之类）的行：184 行，其中 `main.gd` 70 行。
- `main.gd` 里含坐标对字面量的行：129 行。

**专门的修补脚本**：32 个 `*_finish.gd`、`*_details.gd`、`garage_*.gd`、`lighting_repairs.gd`、`missing_fixtures.gd`、`room_lighting_refinement.gd`、`wholehouse_strip_areas.gd`、`upholstery_refine.gd`，共 1888 行，全部按节点名或房间 ID 修具体物件。调用链在 `lighting.gd:515-566` 和 `main.gd:170-178`。

**写死的数据和常量**：

| 内容 | 出处 |
|---|---|
| 视图、产品、方案、摄像头、楼层导航、房间名 | `showroom_data.gd` 全文，119 行，全部人工挑选 |
| 产品系列和款式 | `devices3d.gd:121-130` |
| 楼层 `FLOORS`、大厅 `HALLS`、楼层高度阈值、房子包围盒 | `main.gd:45`、`main.gd:70`、`main.gd:1717`、`main.gd:1748`、`main.gd:2427` |
| 按名字前缀判断楼层 | `main.gd:1060` |
| 布防和对讲的实体 UUID | `main.gd:46-47` |
| 泳池区 `POOL_ZONE`、天窗遮阳表 `SKY_SHADES`、推拉门坐标 | `main.gd:2775-2776`、`main.gd:3465`、`main.gd:4136` |
| 场景的时刻、视角、淡入表 | `main.gd:5822-5836` |
| 性能巡检站点 `PERF_STOPS`（房间 ID 写死） | `main.gd:5751-5764` |
| 电梯停靠高度 | `lift.gd:6-7` |
| 可达性检查的起点坐标 | `reach.gd:12` |
| 悉尼经纬度和夏令时 | `sun_sim.gd:7-8`、`sun_sim.gd:29-38` |
| 全屋场景顺序 | `ui/dock.gd:11` |
| 人物模型表 | `people3d.gd:10` |
| 视点、设备布点、巡游 | `data/viewpoints.json`、`data/device-layout.json`、`data/tours/*.json`（数据文件，但都是这一套房子的） |

### 3.2 已有的通用接缝

1. **`house.json` 契约**：`floors[].elevation/ceilingHeight`、`rooms[].polygon/floor/kind/use`、`boundaries[]`（门、洞口、窗，带 `at/width/type/sill/height`）、`scenes[]`、`entities[]`。运行时大部分通用逻辑只读它：
   - 进门视角 `_entry_view`（`main.gd:2611`）；
   - 房间框选 `_bounds`、`_frame_orbit`（`main.gd:1815`、`main.gd:2531`）；
   - 可达性和窗帘覆盖（`reach.gd`）；
   - 遮挡体（`occluders.gd:2-6`）；
   - 热力图（`heatmap.gd`）；
   - 外壳生成（`shell/src/shell.ts`）。
2. **命名约定 `<room>__<category>__<name>`**：碰撞过滤（`main.gd:1478`）、吊顶件识别（`main.gd:1855`）、灯具按层发光（`lighting.gd:8`）、浮空检测的白名单（`main.gd:6663`）、窗帘识别、门扇 `door-<boundaryId>`（`main.gd:3255`）都靠它。
3. **按目录加载**：`res://interiors/<room>/<room>.glb`、`res://outdoor/*.glb`（`main.gd:1048-1150`），新增房间只要加文件。
4. **自动化的选取逻辑**：最佳视点 `_best_views`、可站点 `_standing_spots`、门扇避让家具 `_settle_placement`、设备避让 `devices3d.unblock`。这些都是算法，不依赖具体房子。

### 3.3 抽成数据驱动运行时的可行性

**结论**：可行，而且和 #4、#5 的结论（`main.gd` 运行时可改造复用；固定模板 + 数据 JSON + 运行时组装）一致。做法是「保留算法、删修补、把常量换成输入字段」。

| 模块 | 处理 | 来源 |
|---|---|---|
| 相机（overview/free/walkin、飞行、环绕） | 直接抽出 | `main.gd:2486-3039` |
| 楼层显示 | 改写为「标高 > 选中层就隐藏」，楼层从输入的楼层表读，支持 N 层；保留屋顶只投影、吊顶件鸟瞰隐藏 | `main.gd:2435-2484`、`main.gd:1847-1897` |
| 行走 | 抽出，补重力；楼层判定改为查楼层标高表，并驱动 `set_floor` | `main.gd:1447-1813` |
| 门 | 新写统一的门组件：从 `boundaries` 生成，默认关，E 键或点击开关，开门扫掠检查沿用 `_settle_placement` 的思路。E 键要从自由视角的「上升」里拿掉 | — |
| 房间导航 | 列表从 `rooms` 自动生成，按楼层分组，名字取房间名；手工视点作为可选的输入覆盖 | 沿用 `_entry_view`、`_best_views`、`walk_room` |
| 日照 | `SunSim` 的经纬度、时区、朝向改为输入 | `sun_sim.gd` |
| 灯光 | 分层灯光加作用域、阴影预算，回路和场景调光表从规则里读 | `lighting.gd` 主体 |
| 场景执行 | 把 Hub 的场景执行和状态存储（`hub/src/store.ts`、`entity-model.ts`）移植成 GDScript，或者打包成单机进程 | — |
| 产品特写和换款 | 图钉、棚拍、单品换款、全屋换系列都抽出，`PRODUCTS` 和 `SERIES` 改为读产品库清单；「整套方案」新做 | `hotspots.gd`、`product_shot.gd`、`open_device`、`open_schemes` |
| 测试和审计钩子 | 整体搬成独立的 `qa/` 工具，路径改成 Mac 可用，作为 v1 验收工具链 | `perf.gd`、`reach.gd`、`flicker-check`、`effects`、`floatcheck`、`devaudit`、`coverage`、`check-scripts` |
| 删除 | 不复用 | 32 个修补脚本；`main.gd` 里约 1700 行针对具体物件的修补；Agent、APP 遥控、展台待机、CCTV、能源回放、人物 |

**风险**：

- 很多「通用」函数里也夹着写死的分支，比如 `walk_room` 会先查 `Data.VIEWS`（`main.gd:2746-2749`），`_apply_floor_visibility` 里有 `FF-` 阳台的特判（`main.gd:2459-2465`）。抽的时候要逐函数清理，不能整块拷贝。
- 修补脚本很多是为了弥补上游模型的毛病（浮空、穿插、材质过亮）。Twinwise 的上游是网站模型（three.js），质量更不可控。这类问题要靠输入校验和验收报告暴露出来，再让上游修，不能再在运行时一条条打补丁。这一点和 #1 的「只改输入，不手工改 Godot 项目」一致。
