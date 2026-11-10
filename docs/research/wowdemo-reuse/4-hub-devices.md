# 调研报告：showtime-wowdemo 的 Hub 与设备行为（给 Twinwise #10 用）

范围：showtime-wowdemo `feat/house-v1`（只读）。下文路径都相对仓库根目录 `…/scratchpad/wow/showtime-wowdemo/`。行号以本次读取的版本为准。实体数量是从 `contracts/house/dist/house.json`（schemaVersion 1.4.2）统计出来的：60 个房间、351 个实体、76 个场景（9 个全屋、67 个房间）。

---

## 0. 结论先行

1. 设备的状态模型**按 domain 写成通用代码，而且有测试**：
   - 状态形状沿用 Home Assistant 风格的 `{state, attributes, changedAt, transition}`（`hub/src/entity-model.ts:5-9`）。
   - 每种 domain 一条规则（`entity-model.ts:46-107`）。
   - 每种类型一行声明（`contracts/house/data/entity-types.json:1-52`）。
   - 这三层可以直接作为 Twinwise「设备种类与行为」的骨架。
2. 模拟都很简单：
   - 温度：每个房间一个温度值的集总模型，52 行（`hub/src/sim/thermal.ts`）。
   - 人员：BFS/Dijkstra 走房间，加"人到灯亮"（`hub/src/sim/people.ts`）。
   - 安防：脚本化事件加定时器（`hub/src/sim/security.ts`）。
   - 能源：**纯回放**一天的采集数据，不响应设备（`hub/src/energy.ts:4-11`）。
   - 时间推进用墙上时钟：每 5 秒走一次，一次推进 60 秒仿真时间（`hub/src/server.ts:244-245`）。
3. Hub 之所以独立成进程，原因是 **APP、Agent、大屏三个客户端共用同一份状态**，并且"大屏崩溃不能拖垮 APP 和 Agent"（`docs/decisions/0001-wowdemo-technical-route.md:16,29`）。Twinwise v1 没有 APP 和 Agent，这条理由不成立。**推荐：v1 不带 Node Hub，把状态机移植到 Godot 内部，但保留 Hub 的消息形状**（snapshot / delta / control / scene），以后接真实设备时把本地 store 换成远端客户端即可。
4. 能直接拿来用的：类型声明表、domain 规则、场景等于目标列表并原子执行、组的扇出、`source` 标记、HA 服务映射。写死在这套房子里的：
   - 安防、人员、开场预设、Agent 演示里的实体 ID
   - 悉尼的气候和能源数据
   - 类型命名混乱
   - 没有 CoreBox，面板和开关不是实体
   - 灯只有百分比、没有流明
   - 除温度、存在、雨以外的传感器都是常量
   - 没有通用自动化引擎

---

## 1. 设备清单

### 1.1 状态模型的三层

| 层 | 内容 | 出处 |
|---|---|---|
| 类型声明 | `type → domain`；`state.kind`（number / boolean / enum / none）及值域；`transitionMs`；`animation` | `contracts/house/src/types.ts:12`，`contracts/house/data/entity-types.json:1-52` |
| 实体 | `id, name, type, roomId, floor, groupOf?, capabilities?, semantic?` | `types.ts:13` |
| 运行时状态 | `{state: string, attributes: {...}, changedAt, transition?: {attribute, from, to, start, duration}}` | `hub/src/entity-model.ts:5-8` |
| 写入来源 | `app / agent / screen / scene / automation / simulator`；传感器只接受 `simulator` 写入 | `entity-model.ts:3-4, 51-52` |
| 校验 | 越界直接报错、不截断；一批控制里有一项不合法，整批不写入 | `entity-model.ts:45`，`hub/src/store.ts:49-65` |
| 单位 | 亮度 0–100（%）；窗帘位置 0（关）到 1（开）；空调设定 16–30 °C | `contracts/house/README.md:17-21`，`entity-model.ts:13` |

### 1.2 设备种类表

数量来自 house.json。"控制"一列指 Hub 接受的补丁或服务。"场景/模拟驱动"一列说明谁会改它。"大屏表现"一列说明 Godot 怎么渲染。

| 种类（contract type） | 数量 | 状态字段 | 可执行的控制 | 如何被场景/模拟驱动 | 大屏表现 |
|---|---|---|---|---|---|
| **调光灯** DALI Dimming 85 / Phase Dimming 3 / Dimming 1 | 89 | `state` on/off；`brightness` 0–100；`last_brightness`；`color_temp_kelvin` 2000–6500 **或** `rgb`（二选一）；过渡（transition）的时长 = 类型声明的 1500 ms，或调用时给的 `transition` 秒数 | `on`（不带亮度时恢复上次亮度）/ `off` / `brightness` / 色温 / RGB / `transition`（`entity-model.ts:58-73`）；HA `light.turn_on` 系列（`hub/src/ha.ts:79-86`） | 场景目标；人员模拟"人到灯亮"，亮度 60（`people.ts:75-76`）；安防夜间开外围灯（`security.ts:33,52`） | 按回路名称分层（筒灯 / 灯带 / 线条灯 / 吊灯），用补间动画调灯光能量和颜色（`showroom/scripts/lighting.gd:2-7, 731-770`）；色温被压到偏暖（`lighting.gd:724-729`） |
| **开关回路** Switching Circuit | 11 | on/off | on / off / toggle | 同上 | 同上，亮度按 100 处理（`showroom/scripts/devices.gd:189-190`） |
| **RGB / 泳池灯** RGB Strip 3 / Pool Light 1 | 4 | 同调光灯，主要用 `rgb` | 同上 | 场景（例如 `scene.cinema` 设 `rgb [64,96,255]`） | 只给灯带层上色（`lighting.gd:4`） |
| **窗帘/遮阳** Blind 39 / Skylight 8 | 47 | `state` open/closed；`position` 0–1；`transition`（时长与移动距离成正比） | `open` / `closed` / `position`；停止 = 停在当前插值位置（`entity-model.ts:74-81`，`ha.ts:88-94`） | 场景；构建期宏 `allCovers:"closed"`（`contracts/house/src/compose.ts:81-84`）；下雨时关全部天窗（`security.ts:43-48`） | 窗帘、天窗动画（`showroom/scripts/main.gd` `_apply_curtains` / `_apply_skylights`，`main.gd:3742-3760`） |
| **其他开合类** Projector Screen 2 / Projector Lift 1 / Pool Cover 1 / Gate 2 / Access（车库门）1 / AC Damper 1 | 8 | 同窗帘（cover domain），过渡 4–30 s | 同窗帘 | 车辆事件开车道闸和车库门，20 s 后自动关（`security.ts:39-42`）；Intercom 开门，15 s 后关（`security.ts:64-67`）；APP 继电器脉冲，30 s 后自动关（`hub/src/app-bridge.ts:28-32, 78`） | 闸门、车库门、幕布动画（`main.gd:3742-3760`） |
| **门锁** access | 3 | enum locked/unlocked | locked / unlocked；对 APP 映射成 switch（`ha.ts:25-26`） | 场景 | 锁装在门扇上，随门转动（`showroom/scripts/devices3d.gd:913-949`） |
| **空调** Air Condition 14 / ACI 2 | 16 | `state` off/cool/heat/auto/dry/fan_only；`temperature` 设定 16–30；`fan_mode`；`current_temperature`（只有模拟器能写）；`last_mode` | 模式、设定、风速（`entity-model.ts:82-89`，`ha.ts:98-102`）；"on" 回到上次模式（`ha.ts:55`） | 场景；**热模型**把房间温度拉向设定（`thermal.ts:34-39`） | 出风口出粒子气流，制冷偏蓝、制热偏暖（`showroom/scripts/states3d.gd:181-196, 388-397`） |
| **地暖** Electric FH 7 / Hydronic FH 6 / AC Floor heating 3 | 16 | climate，off/heat（另有 auto） | 同空调 | 热模型按"只制热"处理，时间常数 30 min（`thermal.ts:11, 36`） | 地面淡淡暖光，3 s 渐显（`states3d.gd:198-227, 398-410`） |
| **泳池加热 / 桑拿** Pool Heater 1 / Sauna Heater 1 | 2 | climate，off/heat | 同上 | 场景；泳池在户外，不进热模型（`thermal.ts:18, 22`） | 泳池水汽粒子、桑拿石发光并点亮补光（`states3d.gd:316-353, 419-433`） |
| **风扇** Exhaust Fan 9 / Fan 1 / Range Hood 1 / Speed Fan 2 | 13 | on/off；Speed Fan 带 `speed` 0–3 和 `last_speed` | on / off / speed；HA `set_percentage` 和增减挡（`entity-model.ts:90-95`，`ha.ts:103-111`） | 场景 | 吊扇转动、排风口水汽、油烟机吸烟（`states3d.gd:50-65, 411-418, 448-452`）。注意：Godot 读的是 `percentage`，Hub 存的是 `speed`，所以 Speed Fan 在 Godot 里永远按默认 60% 转（`states3d.gd:413` 对比 `entity-model.ts:93`） |
| **开关类** Pool Pump 1 / Irrigation 2 / EV Charger 1 / Water Feature 1 | 5 | on/off | on / off / toggle | 场景 | 水景喷溅（`states3d.gd:366-380, 426-429`）；EV 充电只显示状态，**不影响能源数据** |
| **电视升降互斥组** Mutex Member 4 + mutex_no_feedback 1 | 5 | on/off | on / off | 场景 | 电视升、降、左转、右转（`states3d.gd:434-446`），实体 ID 写死 |
| **背景音乐/影音** Media Player | 10 | `state` off/idle/playing/paused；`volume` 0–1；`source` / `media_title` / `media_artist` | 播放、暂停、停止、音量（`entity-model.ts:96-101`，`ha.ts:70-76, 95-97`） | 场景；构建期宏 `allMedia:"off"`（`compose.ts:78-80`） | 同一分区的吸顶音箱冒声波圈（`devices3d.gd:876-911`）；分区按房间找播放器，外加 2 个借邻房的例外（`devices3d.gd:7, 24-26, 868-874`） |
| **门口机 Intercom** Security（HikVision 门口机） | 1 | enum idle → ringing → talking → open | `answer / open / reject / hangup`（`security.ts:61-71`，`server.ts:147`） | 来访事件触发振铃，30 s 无人接自动挂断；开门联动大门 15 s（`security.ts:31-35, 64-67`） | 10 寸主面板播放门口视频，并显示接听和开门按钮（`devices3d.gd:1138-1202`） |
| **布防** arm | 1 | enum disarmed / armed_home / armed_away / triggered | 直接设状态；确认后回到之前的布防模式（`security.ts:73-77`） | 布防状态下发生入侵 → triggered，同时开外围灯（`security.ts:50-54`）；场景 `scene.arm_away` / `scene.disarm` | 盾牌图标和状态文字（`devices.gd:134-136, 235-236`） |
| **摄像头** Camera | 3 | enum idle / motion / alarm | 无（只能由模拟器写） | 安防事件按名字指定预生成的 CCTV 片段 C1–C4，分白天和夜间版本（`security.ts:16-18, 35, 42, 54`） | 播放预生成视频；`device-layout.json` 里另有 9 个摄像头外观模型，**和摄像头实体没有关联** |
| **存在传感器** presence | 34 | on/off（只读） | 无；HA 拒绝写入（`ha.ts:68`） | 人员模拟：进出房间就写（`people.ts:73, 86-91`）；展厅访客 20 s 后熄灯（`people.ts:116-127`） | 吸顶小圆片的指示灯亮起；地面探测圈已关掉（`showroom/scripts/sensors3d.gd:124-144, 69`） |
| **温度** temperature | 17 | 数值 −10–50 | 无 | 热模型写入（变化不小于 0.05 °C 才写，`thermal.ts:45-49`） | 热力图：Hub 给房间值，Godot 自己加探头的局部偏差（`showroom/scripts/heatmap.gd:2-7`） |
| **其他环境传感器** humidity 18 / lux 11 / co2 5 / pm2.5 2 / pm10 2 / voc 2 | 40 | 数值 | 无 | **不模拟**：一直是初始常量（`entity-model.ts:12, 24`），或 APP Demo 的种子值 | 没有专门的表现 |
| **漏水/水浸/雨** Leak 10 / flood 7 / Rain 1 | 18 | on/off | 无 | 只有 Rain 由"下雨"事件写入，60 s 后恢复（`security.ts:8, 43-48`）；Leak 和 flood 不模拟 | — |
| **能源** Energy | 1 | 数值，固定为 `'0'`（`entity-model.ts:25`） | 无 | 真实数据走单独的 `energy` 通道（见 §2.2） | 能源看板 |
| **扫地机** Vacuum | 1 | enum docked / cleaning / returning | 走通用 enum 规则（`entity-model.ts:102-106`） | 无 | 无 |
| **场景实体** scene | 6 | none | — | APP 的场景按钮 | — |
| **灯组 / 窗帘组**（`groupOf`） | 19 | 跟随成员：任一成员亮，组就算亮，亮度取最亮的成员；窗帘组取开得最大的成员 | 控制组 = 扇出到每个成员（开关回路收到亮度时转成 on/off） | — | 显示成一个图块（`devices.gd:50-58, 90-111`）；见 `store.ts:21-47` |

### 1.3 你点名的几类，参考仓库里**没有作为实体**的

| 种类 | 实际情况 | 出处 |
|---|---|---|
| **CoreBox** | 整个仓库搜不到 `corebox` / `core box` | 本次全仓 grep |
| **面板 / 开关面板 / 温控面板** | 只是摆放用的外观模型，**没有 Hub 状态**。共 107 个点位（switch 37、speaker 28、presence 14、room_panel 12、camera 9、main_panel 3、lock 3、thermostat 1），来自 `showroom/data/device-layout.json`。点位**不引用实体 ID**，只按房间关联：音箱按房间找播放器（`devices3d.gd:24-26`），存在传感器按房间找实体（`sensors3d.gd:20-22`）。屏幕内容是静态截图分页（`devices3d.gd:599-646`）；温控面板显示热模型给的房间温度（`devices3d.gd:775-777`）；主面板来电时播放门口视频（`devices3d.gd:1138-1143`）。还能一键换整屋面板系列（`devices3d.gd:119-174`） | `device-layout.json`，`devices3d.gd:1-5` |
| **音响** | 状态在 Media Player 实体上；音箱只是同分区的外观 | 见上 |

### 1.4 场景是怎么定义和执行的

- 定义：`{id, label, scope, targets:[{entityId, state, attributes?}], kind: home|room, name, icon, intent, rooms, behaviours}`（`contracts/house/src/types.ts:14-16`）。示例：`scene.cinema` 有 7 个目标。
- 执行：`runScene` 把全部目标作为**一个 delta**原子提交，`source = scene`，并记录 `requestedBy`（`hub/src/store.ts:75-78`）。测试保证每个场景都能从初始状态跑通（`hub/tests/store.test.ts:34`）。
- 场景"只定义目标状态，动画时序由大屏按实体类型的开合语义执行"（`docs/specs/2026-09-25-wowdemo-architecture.md:173, 195`）。
- 构建期宏 `allLights` / `allMedia` / `allCovers` / `add` / `drop` 在契约构建时展开成具体目标（`compose.ts:65-85`），运行时只看到展开后的列表。
- `behaviours` 是 Agent 的行为 ID 表；ID 不在表里，构建就失败（`compose.ts:25-31`，`contracts/house/data/behaviours.json`）。

---

## 2. 温度、能源、人员、安防怎么模拟

### 2.1 时间推进

- **温度**：`setInterval` 每 `simTickMs = 5000` ms 调一次 `thermal.step(5 × 12 = 60 s)`，也就是 **12 倍速**，然后广播 `comfort`（`server.ts:243-245`）。
- **能源**：每 10 s 按**当前墙上时间（悉尼时区）**取一帧推给大屏和 APP（`server.ts:239-242`，`energy.ts:9-11`）。
- **人员、安防**：用 `setTimeout` 驱动步行和事件时长，都是真实时间。
- 时间源是 `store.snapshot().serverTime`，也就是 `Date.now()`（`store.ts:12, 67`）。测试里注入假的 `now` 和 `schedule`，所以测试是确定的（`hub/tests/thermal.test.ts:5`，`security.ts:21`，`people.ts:64`）。**运行时不是确定的**，因为依赖墙上时钟和时区。
- 日照和昼夜**不在 Hub**：太阳位置在 Godot 里按悉尼经纬度和时钟算（`showroom/scripts/sun_sim.gd:1-8`），热模型自己再算一遍日照（`thermal.ts:9`）。两边各算各的，没有共享同一个"仿真时间"。

### 2.2 各模拟器

| 模拟器 | 模型 | 输入 → 输出 | 出处 |
|---|---|---|---|
| **温度** | 集总模型，每个室内房间一个温度值。四项相加：① 围护结构向室外温度靠拢，τ = 3 h；地下室向 20 °C 靠拢，τ = 12 h。② 有朝北玻璃的房间加日照增益，0.8 °C/h × sin 函数。③ 开着的空调和地暖把房间温度拉向设定值，空调 τ = 10 min，地暖 τ = 30 min；自动模式双向调节，只制热的设备只往上拉。④ 开放式相邻的两间互相混合，τ = 15 min。室外温度是正弦曲线 17.5 ± 5 °C（悉尼春季）。注释原话"Not physics-grade: it only has to look right on a heatmap" | 读空调状态和户型边界 → 写温度传感器的 `state` 和空调的 `current_temperature`（只在变化不小于 0.05 时写）。**窗帘位置、人员、灯光不影响温度** | `thermal.ts:4-11, 13-16, 27-43, 45-50`；测试 `hub/tests/thermal.test.ts:7-31` |
| **能源** | **纯回放**：APP Demo 的一天采集数据（8640 帧，每 10 s 一帧），按悉尼当天时刻取帧，再盖上当前时间戳；另有 5 分钟粒度的整日曲线和每小时回路负载。规格里写的"按场景与设备负载叠加"（`architecture.md:199, 213`）**没有实现**：`energy.ts` 不读 store | 墙上时间 → `energy` 消息、MQTT `ibms/energy/{sn}`、13 个只读 HA 能源传感器 | `energy.ts:4-19, 21-50`；`hub/tests/energy.test.ts:4-11` |
| **人员** | 写死的 4 口之家（Emma、James、Leo、Mia），各有 home_room 和起始房间。`move(id, room)` 用 Dijkstra 在门、洞口、开放边、推拉玻璃、楼梯上找最短路，距离按门洞点之间的米数算，楼梯每段加 5 m。按步速（成人 1.3 m/s、儿童 1.1 m/s）、提前 1.5 m 逐个房间"进入"。"回家"按固定路径每 1.8 s 走一间，并开关前门 | 进入或离开房间 → 写房间的存在传感器；**黑房间进人就开灯**，亮度 60，`source = automation`；展厅访客离开 20 s 后，只关掉他自己触发的灯 | `people.ts:5-20, 32-56, 63-143`；测试 `hub/tests/people.test.ts:11-76` |
| **安防** | 脚本事件：`visitor / parcel / vehicle / intrusion / rain`，每个事件都有固定的联动和时长（振铃 30 s、大门 15 s、车库 20 s、移动 12 s、雨 60 s）。Intercom 是一个小状态机；入侵只有在布防时才报警，确认后回到原布防模式 | 由大屏或 HTTP 触发（`server.ts:122-124, 146-148, 216-218`）→ 写摄像头、门口机、雨传感器（simulator），控制门和灯（automation），并广播带 CCTV 片段名的 `security` 通知 | `security.ts:1-79`；测试 `hub/tests/security.test.ts:13-53` |
| **自动化** | 规格计划做"时间、存在、传感器触发；可在 APP 中新建"（`architecture.md:214`）。**代码里没有通用引擎**，只有上面写死的联动（人到灯亮、安防联动、继电器自动关） | — | `rg automation hub/src` 只命中 `people.ts`、`security.ts`、`app-bridge.ts:78`、`store.ts:59` |

---

## 3. 架构问题："大屏只按 Hub 的状态渲染"对 Twinwise 意味着什么

### 3.1 参考仓库里这句话的实际含义

- Godot 不决定任何设备状态，只发命令（`architecture.md:52`；`README.md:10`；`showroom/scripts/main.gd:3`："every state comes from the Hub"）。
- 协议：连上先收 `snapshot`，之后按 revision 顺序收 `delta`。session 或 epoch 变了、revision 出现缺口，就断开重取全量，"never guess"（`showroom/scripts/hub_client.gd:2-3, 60-85`）。大屏发出的只有 `control / scene / sim / intercom / reset` 等命令（`README.md:41`，`hub_client.gd:104-118`）。
- 动画由 Godot 根据 Hub 给的 `transition{from, to, start, duration}` 自己插值，并对齐 Hub 时钟（`hub_client.gd:26, 53-55, 63-64`，`devices.gd:163-195`）。
- **渲染层与 Hub 的接缝已经很窄**：渲染代码只依赖 `hub.entities` 字典、`hub.now_ms()` 和这些信号：`snapshot_received / entities_changed / comfort_received / security_received / people_received / energy_received`（`hub_client.gd:5-15`，`main.gd:310-323, 3742-3770`）。
- 例外：Godot 自己算太阳（`sun_sim.gd`），热力图的探头局部偏差也在 Godot 里模拟（`heatmap.gd:5-7`）。
- 独立 Hub 的原因（ADR-0001 和规格 D3）：大屏重启不影响 APP 和 Agent；状态逻辑可以无头测试；可以复用旧会话服务。被否决的备选方案正是"把 Hub 并进 Godot：部署只有一个进程。但大屏崩溃会把 APP 和 Agent 一起拖断，状态逻辑也没法在命令行下测试"（`docs/decisions/0001-wowdemo-technical-route.md:16, 29`；`architecture.md:25`）。

对 Twinwise 来说，**"渲染只读状态，状态只由一个 store 按规则改"这条纪律值得保留**；"store 必须在另一个进程里"则不必照搬。

### 3.2 两种做法的取舍

约束：纯仿真；以后可能接真实设备；在内部 Mac 上演示；能导出 macOS 应用；命令行加项目文件夹，同样输入得到同样产物；dashboard 只读，显示的是**生成流程**，不是运行时。

| 维度 | A. 生成的项目带一个 Node Hub | B. 状态机移植进 Godot（autoload `DeviceStore`） |
|---|---|---|
| 复用量 | 可以几乎原样搬 `entity-model.ts`、`store.ts`、`thermal.ts`、`people.ts`、`security.ts`，连同测试 | 要把约 200 行 TS（entity-model 加 store）和 52 行热模型改写成 GDScript；人员和安防要按 Twinwise 的规则重新写 |
| 但无论哪种都要改 | 安防、人员、开场预设、Agent 演示里写死的 ID（`security.ts:4-8`，`people.ts:11-20`，`server.ts:52-62, 98-101`）；契约格式要换成 Twinwise 的输入；悉尼气候和能源 | 同左 |
| 运行时 | 两个运行时：Node 22 加 tsx（`README.md:16`，`tools/booth/start-booth.cmd:27-28`），外加 Godot | 只有一个 Godot 进程 |
| macOS 导出 | `.app` 里要内嵌 node 运行时，或者要求机器先装 Node；还要用 `OS.create_process` 拉起子进程、管理生命周期和孤儿进程、处理端口冲突；首次监听端口会弹 macOS 防火墙"接受传入连接"；签名和公证要连 node 二进制一起做 | 一个标准 Godot macOS 导出，没有端口 |
| 同样输入产物一致 | 生成产物可以一致；运行时依赖 `Date.now`、`setTimeout` 和时区，不确定 | 可以用一个仿真时钟统一驱动太阳、温度、事件（参考仓库里这两样各算各的）；演示可复现 |
| 以后接真实设备 | Hub 天然就是网关：现成的 HA REST、MQTT 网关、HA 服务映射（`ha.ts:14-144`，`app-bridge.ts`，`server.ts:168-180`）。但真实设备接入时，状态源会是真实网关或 HA，不是这个仿真 Hub | 保留同样的状态形状和消息形状后，接真实设备 = 把本地 `DeviceStore` 换成 `HubClient`，连到真实网关或 HA 前面的一个适配层。`hub_client.gd` 就是现成的远端实现，渲染层不用改 |
| 可测试性 | `node --test` 已有 12 个测试文件 | 用 Godot headless 跑 GDScript 断言脚本（Godot 本身支持 `--headless --script`；这一点不是参考仓库里的证据）；也可以只做一个自检脚本 |
| 多客户端 | 支持 APP、Agent、多块大屏共用状态 | v1 不需要（#1 的 Destination 和约束里没有 APP 和 Agent） |
| 故障面 | 大屏和 Hub 任一方挂掉就显示"CONNECTING…"，界面停在默认状态（`main.gd:3675-3679`） | 单进程，没有连接这一层 |

### 3.3 推荐

**选 B，并且对齐 A 的接口**：

1. Godot 内部放一个 `DeviceStore`，作为 autoload：
   - 状态形状照抄 `EntityState`（`entity-model.ts:8`）。
   - 规则照抄 `applyPatch`、`commit`、`runScene` 和组扇出（`entity-model.ts:46-107`，`store.ts:21-87`）。
   - 对外提供和 `hub_client.gd` **同名的信号和方法**：`entities`、`now_ms()`、`send_control()`、`send_scene()`、`snapshot_received`、`entities_changed`。这样渲染层可以在两者之间互换。
2. 模拟器（温度、人员、可选的安防脚本）作为 `DeviceStore` 的写入方，`source = simulator / automation`，由**一个仿真时钟**驱动，太阳也用这个时钟。
3. v1 不做 HTTP、MQTT、HA、Agent 接口（YAGNI）。等真的要"接真实设备"时：
   - 加一个启动参数（参考仓库已有 `--hub=ws://…`，`main.gd:7, 289`），切换到远端客户端；
   - 远端那一侧再决定是自建网关，还是直接对 HA；`ha.ts` 的服务映射可以照搬。
4. 选 A 的唯一理由：v1 就要让 Homira APP 或 Agent 一起连进来。#1 没有这项需求，那就不要为它付出双运行时和 macOS 打包的代价。

---

## 4. Hub 的模型能不能直接作为 #10「设备种类与行为」的答案

### 4.1 可以直接沿用的（通用，与这套房子无关）

| 内容 | 出处 | 说明 |
|---|---|---|
| 类型声明表的格式：`type / domain / state.kind / 值域 / transitionMs / animation` | `types.ts:12`，`entity-types.json` | #10 的"每种设备有哪些状态"可以直接用这个格式写进上游**规则**数据 |
| HA 风格的状态形状和单位约定 | `entity-model.ts:5-9`，`contracts/house/README.md:13, 17-21` | 与 HA 和 Homira APP 同构，接真实设备时最省事 |
| 各 domain 的行为规则：light / cover / climate / fan / media / sensor / 通用 enum 和 boolean | `entity-model.ts:46-107` | 已有测试覆盖"上次亮度""开合时长与距离成正比""空调记住上次模式""色温与 RGB 互斥""传感器只能由模拟器写"（`hub/tests/entity-model.test.ts:14-76`） |
| 原子批量提交、revision 和 delta、`source` 标记 | `store.ts:49-65` | 写日志、做回放、dashboard 显示变化都靠它 |
| 场景 = 目标列表并原子执行；构建期宏 | `store.ts:75-78`，`compose.ts:65-85` | 对应 CONTEXT 里的**规则**（灯光场景） |
| 组的扇出和跟随 | `store.ts:21-47` | |
| HA 服务映射（以后接真实设备或 APP 时用） | `ha.ts:14-144` | v1 不需要，放着备用 |
| 热模型的结构 | `thermal.ts` | 回答 #10 里"温度演示由哪些设备驱动"：climate domain 的空调和地暖；表现是出风粒子、地面暖光、热力图、温控面板读数 |
| 状态 → 画面的对照 | `states3d.gd:1-11`，`devices.gd:197-248` | 每种类型"怎么看得出状态变了"的现成清单 |

### 4.2 写死在这套房子或这次项目里、不能照搬的

| 问题 | 出处 |
|---|---|
| 类型名来自 APP Demo 和 KNX 的叫法，大小写和含义都不统一：`access`（锁）和 `Access`（车库门）、`presence`、`ACI`、`AC Floor heating` | `entity-types.json:11-18, 29` |
| 实体 ID 前缀和 domain 对不上：Electric 和 Hydronic FH 的 ID 是 `switch.*`，类型却是 climate；ACI 的 ID 是 `sensor.*`；锁的 ID 是 `switch.*`、domain 是 lock；Access 和 Intercom 用 UUID 当 ID。热模型和 HA 映射只能靠特判绕过 | house.json 统计；`thermal.ts:22`；`ha.ts:26, 128` |
| 安防联动直接写了实体 ID（门口机、布防、车库、3 个摄像头、3 盏灯、2 个闸门、雨传感器） | `security.ts:3-9` |
| 家庭成员、回家路径、前门 ID | `people.ts:11-20` |
| 开场预设（地下室灯光、影院氛围）、Agent 演示的指令 | `server.ts:51-62, 98-101` |
| Godot 侧写死的 ID：桑拿、泳池加热、水景、电视升降、布防和门口机 | `states3d.gd:361-363, 379, 419-446`；`main.gd:44-45` |
| 音箱分区例外、空调出风位置例外 | `devices3d.gd:7, 179` |
| 气候和能源：悉尼正弦气温、悉尼时区、APP Demo 的一天能源采集 | `thermal.ts:4-9`，`energy.ts:4-11` |
| **没有 CoreBox**；面板、开关、温控面板不是实体，没有状态；点位与实体只按房间关联 | §1.3 |
| 灯光只有 0–100%，没有流明；色温范围全局写死为 2000–6500，没有用实体自带的 `capabilities.min/max_kelvin`（例如 2700–6000）。Twinwise 要求"沿用真实流明和色温"，所以需要在产品库或点位上补流明和色温范围 | `entity-model.ts:66`；house.json 里 `light.ff_master_bedroom_dali_22` 的 capabilities |
| 空调设定范围 16–30 全局写死，不分设备 | `entity-model.ts:13` |
| 湿度、照度、CO₂、PM、VOC、漏水、水浸都是常量，不模拟 | `entity-model.ts:12` |
| 能源不响应设备（EV、泳池泵、空调都不改变负载） | `energy.ts` 全文 |
| 没有通用自动化引擎（规格 §8 计划过） | §2.2 |
| 热模型不考虑窗帘、人员、灯；日照只算朝北玻璃 | `thermal.ts:21, 27-43` |

### 4.3 结论

#10 可以**直接采用 4.1 作为骨架**：类型声明表、domain 规则、场景格式、温度由 climate 驱动。需要 Twinwise 自己定的有：

1. 统一后的设备种类表（类型名、domain、ID 规则）；
2. 点位如何**显式绑定**实体（参考仓库只按房间关联）；
3. 面板和 CoreBox 要不要有状态（参考仓库里面板只是外观加静态 UI）；
4. 流明和色温放在哪（产品库还是点位）；
5. 人员和安防这类联动从"代码里写死 ID"改成**规则数据**（上游给出触发条件和目标列表），或者 v1 直接不做。
