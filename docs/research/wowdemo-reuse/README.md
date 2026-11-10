# wowdemo house-v1 的可复用性

调研对象：[showtime-wowdemo](http://gitlab.teccoibms.com/super-designer/showtime-wowdemo) `feat/house-v1`（commit 394e3f6），另外参考了本地的 `showroom-space-3d-fortis/projects/fortis-double-bay`。分四个方向，各写一份报告，每条结论都标了文件和行号。

| 报告 | 方向 |
|---|---|
| [1-contract.md](1-contract.md) | 住宅契约与数据模型 |
| [2-shell-assets.md](2-shell-assets.md) | 外壳与资产生成：哪些自动，哪些手工 |
| [3-godot-runtime.md](3-godot-runtime.md) | Godot 运行时能力与验收手段 |
| [4-hub-devices.md](4-hub-devices.md) | Hub 与设备行为 |

## 总结论

1. **wowdemo 是骨架，不是成品。** 按工作量估，自动化只占 15–20%，只有外壳生成和灯光骨架。家具、外立面、材质、灯具都是手工用 Blender 做的。另有约 1,900 行补丁脚本和 294 处 Joe 的逐条修补，全部针对这一套房子。
2. **可复用的是机制，不是数据。** 能复用的有：
   - 契约的骨架：楼层、房间、边界图、实体类型、场景；
   - 确定性构建：dist 必须与重新构建的结果一致；
   - 外壳生成器；
   - `lighting.gd` 的通用部分；
   - "生成 JSON 清单，运行时校验通过才应用"的模式；
   - 设备状态模型（HA 风格的 `{state, attributes}`、按类型声明、场景就是一组目标状态）；
   - 产品图钉、抽屉棚拍特写、全屋换系列；
   - 验收工具：perftour、walk-check、闪烁检测、effects、floatcheck 等。
3. **生成路线不变**（见「生成路线：直接组装 Godot 还是经过 Blender」）。wowdemo 本身就是固定的 `main.tscn` 运行时组装，用 SDFGI 加每间房一个反射探针，不做烘焙。Blender 在它的流程里相当于 Twinwise 的上游。
4. **不带 Node Hub。** Hub 存在的理由是 APP、Agent 和大屏共享同一份状态，Twinwise v1 没有这个需求。做法是把状态规则移植进 Godot，作为内部状态存储，接口和 `hub_client.gd` 同名；用一个仿真时钟统一驱动太阳、温度和事件。以后要接真实设备，用启动参数切换到远端。
5. **要补强的地方**：
   - SDFGI 不处理动态自发光，灯带要靠 AreaLight3D、按房间分接收层和回光代理；
   - 白天天花偏暗至今没有解决，要列为验收项；
   - 目前没有 Mac 或 M1 的性能数据（只在 RTX 3060 上测过）。
6. **契约缺 Twinwise 需要的字段**：
   - 点位坐标，以及点位和回路的显式绑定（wowdemo 靠 GLB 命名猜）；
   - 家具；
   - 贴图、灯带、墙板；
   - 灯的流明和色温；
   - 地理位置和正北偏角（wowdemo 写死悉尼）；
   - 门的交互声明；
   - 上游 glb 节点到房间、门的映射。

   fortis 的白模命名约定 `<层>__<部件>__<id>` 可以作为上游 glb 的输入约定草案。
7. **wowdemo 没实现、要新做的**：整套方案（`SCHEMES` 定义了但没被调用）、单个点位换款式（它是全屋同类一起换）、天气、画质档位、可交互的室内门、行走时楼层跟随、真正的重力、macOS 导出、CoreBox，以及面板和温控面板的状态。
