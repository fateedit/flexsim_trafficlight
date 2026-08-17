# FlexSim 红绿灯系统（Traffic Light System for FlexSim）

一个基于 FlexSim 的**最简单的红绿灯模型**：一个十字路口、东西（EW）和南北（NS）两个方向**交替放行**——东西方向绿灯时南北方向红灯，南北方向绿灯时东西方向红灯。用 TaskExecuter + DrawSurrogate 搭建三色灯，按设定时长循环切换红/黄/绿，并通过 **Locker / Group / ControlArea** 控制 AGV 的通行与停车。不绑定任何真实地名，按自己的路网即可复用。

> **本仓库包含：搭建文档 + 模型文件**（`models/红绿灯.fsm`，一个最简单的十字路口红绿灯模型示例）。模型直接打开即可运行；文档讲解**通用逻辑与搭建方法**，可复现到你自己的路网。
> 打开模型需要 **FlexSim 2026**（商业软件，可用学生版/试用版），建议对 FlexSim 的标签、触发器、AGV 模块有一定了解。

## 演示

![红绿灯运行演示](docs/demo-traffic-light.gif)

> 动态图演示（红绿灯轮转 + AGV 通行/停车效果）。[下载 35 秒完整演示视频](docs/demo-traffic-light.mp4)

## 仓库结构

```
flexsim_trafficlight/
├── README.md                      ← 本文件：模型说明 + 基本原理 + 从零搭建指南
├── models/
│   ├── 红绿灯.fsm                ← 最简单的十字路口红绿灯模型（FlexSim 2026 打开）
│   ├── trafficlight.fsl           ← 红绿灯用户库文件（注册后可从库面板直接拖出）
│   └── fbx/
│       ├── 杆横向.fbx             ← 灯柱（横臂）3D 模型
│       └── 灯0.fbx                ← 灯体 3D 模型（三个灯复用，着色区分）
└── docs/
    ├── demo-traffic-light.gif      ← 运行演示动态图（README 首页直接播放）
    ├── demo-traffic-light.mp4      ← 35 秒完整演示录像（可下载）
    ├── 配置详解.md                 ← 先干嘛后干嘛的完整配置顺序（ControlArea 控制点/Group/Locker/标签/脚本）⭐
    ├── 占用模块.md                ← 红绿灯占用/放行模块的通用搭建教程
    ├── 配时逻辑与标签详解.md        ← 配时逻辑（偏移/周期/红灯计算）+ 标签与 lockerName/groupName 详解 ⭐
    ├── 控制区域与占用逻辑详解.md     ← ControlArea 占用逻辑深度解析（位置/三级结构/时序/坑）⭐
    ├── 模型3D外观与用户库.md        ← 3D 外观构建（灯柱+3灯/FBX/用户库）与工作原理 ⭐
    ├── 模型blender.md             ← 用 Blender 脚本生成红绿灯 3D 模型的另一种做法
    └── 踩坑记录.md                ← 部署过程中踩过的坑与修复方法
```

## 模型文件与运行要求

- **模型**：`models/红绿灯.fsm` —— 一个最简单的十字路口红绿灯模型示例（东西/南北两个方向交替放行、AGV 路网与占用/放行逻辑），打开即可运行。
- **打开方式**：FlexSim 2026（中文版）→ File → Open → 选择 `models/红绿灯.fsm` → 点击 Run。
- **版本要求**：模型为 FlexSim 2026 格式（文件头 `flexsimtree 26.0`），低版本可能无法打开或显示异常。
- **3D 外观依赖（路径要求）**：模型外观使用外部 FBX 模型（`杆横向.fbx` 灯柱 + `灯0.fbx` 灯体），构建时保存的是本机绝对路径。仓库已在 `models/fbx/` 附带这两个文件，换电脑打开后若 3D 显示不全，需在 FlexSim 里重新指定外观路径（选中灯实体 → 外观 → 重新选择 FBX），或把 FBX 复制到原路径。完整说明见 [模型3D外观与用户库](docs/模型3D外观与用户库.md)。
- **用户库**：`models/trafficlight.fsl` 是红绿灯用户库文件——**别人可以直接下载它装进自己的 FlexSim 使用**（见下节"快速上手"）。
- 搭建步骤与逻辑细节见下文及 `docs/` 目录。

## 别人怎么用这个仓库（快速上手）

**方式一：直接运行完整模型**

下载 `models/红绿灯.fsm`，用 FlexSim 2026 打开 → 点击 Run 即可看到效果。

**方式二：只装用户库，在自己的模型里直接拖出红绿灯（推荐复用）**

1. **下载**：`models/trafficlight.fsl`（建议连同 `models/fbx/` 两个文件一起下载）；
2. **安装**（二选一）：
   - 把 `trafficlight.fsl` 放到 FlexSim 的 libraries 目录，如 `D:\Common\flexsim\libraries\TerminalSim\`；
   - 或：FlexSim 菜单 **工具 → 全局设置 → 用户库 → 添加**，选择 `trafficlight.fsl`；
3. **使用**：打开你自己的模型，在库面板搜索 "trafficlight"，把红绿灯实体**直接拖进场景**，再按 [配置详解](docs/配置详解.md) 配置方向、标签和 ControlArea，运行即可。

> 💡 **如果拖出来的灯没有 3D 外观**：灯的模型外观引用的是外部 FBX 文件。把 `models/fbx/` 里的 `杆横向.fbx`（灯柱）和 `灯0.fbx`（灯体）放到你电脑上，然后在灯实体的外观属性里重新选择这两个文件即可（详见 [模型3D外观与用户库](docs/模型3D外观与用户库.md)）。

> 逻辑（变色、锁/放）全部在标签和脚本里，跟着用户库一起走，不需要另外配置。

## 最基本的红绿灯是怎么做的

最简单的情况就是：**东西方向绿灯时南北方向红灯，南北方向绿灯时东西方向红灯**——两个方向交替放行，永远只有一个方向在走。

实现它只需要三样东西：

| 组成 | 说明 |
|------|------|
| 灯实体 | 每个方向 1 个灯 = 1 个 TaskExecuter + 3 个灯色子实体（`red` / `green` / `yellow`） |
| 状态循环 | 每个灯按 绿灯 → 黄灯 → 红灯 → 绿灯 循环；两个灯相位正好**相反**（一个绿灯时另一个红灯） |
| 通行控制 | ControlArea（停车区）+ Locker（锁）+ Group（分组）：**绿灯放行、红灯禁行** |

> 下文以一个路口、两个方向组（NS 南北 / EW 东西）为例；灯的数量、配时按你的路网自定。

## 基本原理

### 1. 灯的颜色循环

每个灯用 `phase` 标签记录当前状态，靠 `senddelayedmessage` 定时切换：

```
GREEN（绿灯） → YELLOW（黄灯） → RED（红灯） → GREEN（绿灯） → ...
```

- 绿灯：放行本方向（解除 ControlArea 锁定）；
- 黄灯：警示，同时开始锁定本方向（新来的车不再进路口）；
- 红灯：禁行（AGV 在 ControlArea 前停车等待）。

### 2. 两个灯怎么交替（偏移时间）

两个灯的周期相等（周期 = 绿灯 + 黄闪 + 红灯），相位正好**相反**：灯 1 绿灯时灯 2 红灯，灯 1 变红时灯 2 变绿。用 `偏移时间` 实现：

```
灯 2 的偏移 = 灯 1 的（绿灯 + 黄闪）之和
```

模型启动时 OnModelStart 用 `fmod(偏移, 周期)` 算出当前落在哪个状态，两个灯瞬间就处于正确的相位。

### 3. AGV 通行控制 = 占用/放行

灯本身不拦截 AGV，通行控制靠 **Locker + Group + ControlArea** 三级配合：

| 组件 | 作用 |
|------|------|
| `ControlArea` | 铺在 AGV 路径上的禁行/停车区 |
| `Group` | 把同一方向的所有 ControlArea 归为一组 |
| `Locker` | 对一组 ControlArea 统一加锁/解锁 |

**绿灯 = 解除锁定（放行），红灯 = 重新锁定（AGV 在 ControlArea 前停车）**。锁定逻辑有两条实现路径，可任选其一或双保险：

- **灯实体 OnMessage 直控**：通过 `lockerName`/`groupName` 标签直接锁/放（适合灯少的路口）；
- **ProcessFlow 子流程轮询**：轮询 `phase` 标签控制（独立运行、容错强）。

通用实现细节见 [占用模块](docs/占用模块.md) 与 [控制区域与占用逻辑详解](docs/控制区域与占用逻辑详解.md)。

---

## 第一步：Global Code（一个通用函数，所有灯共用）

**位置**：Model → Scripts → Global Code，粘贴一次即可：

```cpp
void setLightColor(Object light, string state) {
    treenode redLight = light.find("red");
    treenode greenLight = light.find("green");
    treenode yellowLight = light.find("yellow");

    if (state == "GREEN") {
        if (greenLight) greenLight.color = Color(0, 1, 0, 1);
        if (yellowLight) yellowLight.color = Color(1, 1, 0, 0.3);
        if (redLight) redLight.color = Color(1, 0, 0, 0.3);
    } else if (state == "YELLOW") {
        if (greenLight) greenLight.color = Color(0, 1, 0, 0.3);
        if (yellowLight) yellowLight.color = Color(1, 1, 0, 1);
        if (redLight) redLight.color = Color(1, 0, 0, 0.3);
    } else {  // RED
        if (greenLight) greenLight.color = Color(0, 1, 0, 0.3);
        if (yellowLight) yellowLight.color = Color(1, 1, 0, 0.3);
        if (redLight) redLight.color = Color(1, 0, 0, 1);
    }
}
```

---

## 第二步：创建灯实体

从 Library 拖 **TaskExecuter** 到场景，放到路口各方向对应位置。命名建议体现"方向 + 灯"（示例，名称按你的路网自定）：

| 灯 | 方向 |
|------|------|
| `TrafficLight_NS` | 南北方向 |
| `TrafficLight_EW` | 东西方向 |

> 一个方向可能有多个灯（如南北左转/直行分开），命名时把方向写清楚即可。

### 子实体

每个灯下创建 3 个 **DrawSurrogate**：

```
灯实体
├── red      (DrawSurrogate)
├── green    (DrawSurrogate)
└── yellow   (DrawSurrogate)
```

给每个 DrawSurrogate 一个可见形状（如 Sphere / Cylinder），初始颜色设灰色。

---

## 第三步：添加标签

每个灯 → Properties → Labels，添加以下标签：

| 标签名 | 类型 | 说明 |
|--------|------|------|
| `绿灯时长` | number | 绿灯秒数（按交通方案定） |
| `黄闪时长` | number | 黄灯秒数 |
| `红灯时长` | number | **公式算出**（见下） |
| `偏移时间` | number | **公式算出**（相位错开） |
| `phase` | string | 当前状态，脚本自动维护 |

> 若用 OnMessage 直控占用逻辑，每个灯**额外**加 2 个 **Pointer** 标签：`lockerName`（指向该方向 Locker）、`groupName`（指向该方向 Group）；若用 ProcessFlow 轮询则不需要。

### 配时数值怎么填

配时值**按公式计算**，不是拍脑袋填：

```
周期 = 绿灯时长 + 黄闪时长 + 红灯时长
红灯时长 = 周期 - 绿灯时长 - 黄闪时长
偏移时间 = 前面所有灯的（绿灯时长 + 黄闪时长）之和
```

完整公式、通用示例与校验方法见 [配时逻辑与标签详解](docs/配时逻辑与标签详解.md)。模型 `models/红绿灯.fsm` 里各灯的标签值可作为一组现成例子对照。

---

## 第四步：脚本（每盏灯粘贴三处）

### 4.1 OnModelStart（按偏移时间定位初始状态，启动循环）

```cpp
/**Custom Code*/
Object light = ownerobject(c);

double greenDur = light.labels["绿灯时长"].value;
double flashDur = light.labels["黄闪时长"].value;
double redDur = light.labels["红灯时长"].value;
double offset = light.labels["偏移时间"].value;

double totalCycle = greenDur + flashDur + redDur;
double t = fmod(offset, totalCycle);

string state;
double remaining;
if (t < greenDur) {
    state = "GREEN";
    remaining = greenDur - t;
} else if (t < greenDur + flashDur) {
    state = "YELLOW";
    remaining = (greenDur + flashDur) - t;
} else {
    state = "RED";
    remaining = totalCycle - t;
}

light.labels["phase"].value = state;
setLightColor(light, state);
senddelayedmessage(light, remaining, current, state);
```

### 4.2 OnMessage（状态切换 + 可选占用逻辑）

```cpp
/**Custom Code*/
Object light = ownerobject(c);
string state = msgparam(1);   // 由 OnModelStart / 上一次 OnMessage 传入

double greenDur = light.labels["绿灯时长"].value;
double flashDur = light.labels["黄闪时长"].value;
double redDur = light.labels["红灯时长"].value;

string nextState; double delay;
if (state == "GREEN") {
    nextState = "YELLOW"; delay = flashDur;
} else if (state == "YELLOW") {
    nextState = "RED"; delay = redDur;
} else {  // RED
    nextState = "GREEN"; delay = greenDur;
}

light.labels["phase"].value = nextState;
setLightColor(light, nextState);

// ===== 占用逻辑（可选）：仅当配了 lockerName/groupName 标签时生效 =====
Object locker = getlabel(light, "lockerName");
Object grp = getlabel(light, "groupName");
if (locker && grp) {
    Group theGroup = Group(grp.name);

    // 离开 GREEN → 锁定该方向
    if (state == "GREEN") {
        for (int i = 1; i <= theGroup.length; i++) {
            Object caObj = theGroup[i];
            if (!caObj) continue;
            AGV.AllocatableObject ca = caObj.as(AGV.AllocatableObject);
            int alreadyLocked = 0;
            for (int j = 1; j <= ca.allocations.length; j++)
                if (string(ca.allocations[j].allocator.name) == locker.name) { alreadyLocked = 1; break; }
            if (!alreadyLocked)
                for (int j = 1; j <= ca.requests.length; j++)
                    if (string(ca.requests[j].allocator.name) == locker.name) { alreadyLocked = 1; break; }
            if (!alreadyLocked) ca.allocate(locker, 1);
        }
    }

    // 进入 GREEN → 释放该方向
    if (nextState == "GREEN") {
        for (int i = 1; i <= theGroup.length; i++) {
            Object caObj = theGroup[i];
            if (!caObj) continue;
            AGV.AllocatableObject ca = caObj.as(AGV.AllocatableObject);
            for (int j = ca.allocations.length; j >= 1; j--) {
                AGV.AllocationPoint ap = ca.allocations[j];
                if (string(ap.allocator.name) == locker.name) ap.deallocate();
            }
            for (int j = ca.requests.length; j >= 1; j--) {
                AGV.AllocationPoint ap = ca.requests[j];
                if (string(ap.allocator.name) == locker.name) ap.deallocate();
            }
        }
    }
}

senddelayedmessage(light, delay, current, nextState);
```

### 4.3 OnReset

```cpp
/**Custom Code*/
Object light = ownerobject(c);
light.labels["phase"].value = "GREEN";

Object red = light.find("red");
Object green = light.find("green");
Object yellow = light.find("yellow");
if (red) red.color = Color(0.5, 0.5, 0.5, 1);
if (green) green.color = Color(0.5, 0.5, 0.5, 1);
if (yellow) yellow.color = Color(0.5, 0.5, 0.5, 1);
```

---

## 第五步：测试灯变色

点击 **Run**，观察：

- 每个灯按 绿 → 黄 → 红 循环轮转；
- **同一方向组**的灯同步变；各方向按偏移时间**依次变绿**，任意时刻只有一个方向组为绿灯。

如不对，依次检查：① 子实体名是否全小写 `red`/`green`/`yellow` ② 标签类型是否 number（不是 string） ③ Global Code 是否保存。

---

## 第六步：Locker 与 Group

一个方向一套（示例：两个方向组）：

| 方向组 | Locker | Group | 成员 |
|--------|--------|-------|------|
| 南北 | `TrafficLightLocker_NS` | `NS_Stop_CA_Group` | 南北路径上的全部 ControlArea |
| 东西 | `TrafficLightLocker_EW` | `EW_Stop_CA_Group` | 东西路径上的全部 ControlArea |

> 右转不受控方向**不加** ControlArea，右转始终自由。

---

## 第七步：路面加 ControlArea 并归入 Group

在路口各方向的 AGV 路径上添加 **ControlArea** 对象（放在停车线处、冲突区之前），分别拖入第六步对应的 Group。位置规则与图示见 [控制区域与占用逻辑详解](docs/控制区域与占用逻辑详解.md)。

---

## 第八步：初始化锁定（一套通用代码，只改三个数组）

放在 Model 的 OnModelStart 或一个独立触发器（在灯启动之后执行）：

```cpp
/**Custom Code*/
// ===== 只需改这三个数组（名称与第二/六步一致）=====
string lightNames[]  = {"灯1", "灯2", "灯3", "灯4"};       // 该路口所有灯
string lockerNames[] = {"Locker_A", "Locker_B"};          // 与 groupNames 一一对应
string groupNames[]  = {"Group_A", "Group_B"};

// 1) 所有灯先置为红灯，防止初始时误放行
for (int i = 1; i <= lightNames.length; i++) {
    Object light = Model.find(lightNames[i]);
    if (light) light.labels["phase"].value = "RED";
}

// 2) 每组 Locker 预锁其 Group 内所有 ControlArea
for (int k = 1; k <= lockerNames.length; k++) {
    Object locker = Model.find(lockerNames[k]);
    Group grp = Group(groupNames[k]);
    if (!locker) continue;

    for (int i = 1; i <= grp.length; i++) {
        Object caObj = grp[i];
        if (!caObj) continue;
        AGV.AllocatableObject ca = caObj.as(AGV.AllocatableObject);

        int locked = 0;
        for (int j = 1; j <= ca.allocations.length; j++)
            if (ca.allocations[j].allocator == locker) { locked = 1; break; }
        if (!locked)
            for (int j = 1; j <= ca.requests.length; j++)
                if (ca.requests[j].allocator == locker) { locked = 1; break; }
        if (!locked) ca.allocate(locker, 1);
    }
}
```

只需把三个数组填成你自己的名字（示例：两个方向组）：

| lightNames | lockerNames | groupNames |
|-----------|-------------|------------|
| `TrafficLight_NS`、`TrafficLight_EW` | `TrafficLightLocker_NS`、`TrafficLightLocker_EW` | `NS_Stop_CA_Group`、`EW_Stop_CA_Group` |

---

## 第九步：ProcessFlow（一个通用子流程模板）

新增一个主流程，内部放并行子流程。每个子流程 = 下方模板 + 改前三行：

```cpp
/**Custom Code*/
// ===== 每个子流程只改这里：灯（可多个）、Locker、Group =====
Object light1 = Model.find("【灯1】");
Object light2 = Model.find("【灯2】");   // 单灯子流程删掉这一行
Object locker = Model.find("【Locker名】");
Group group = Group("【Group名】");

while (true) {
    bool isGreen = light1.labels["phase"].value == "GREEN"
                || (light2 && light2.labels["phase"].value == "GREEN");
    if (isGreen) {
        // 放行：解除该组所有 ControlArea 的锁定
        for (int i = 1; i <= group.length; i++) {
            Object caObj = group[i];
            if (caObj) caObj.as(AGV.AllocatableObject).deallocate(locker);
        }
        // 等待灯变红
        while (light1.labels["phase"].value == "GREEN"
            || (light2 && light2.labels["phase"].value == "GREEN")) sleep(0.05);
        // 重新锁定
        for (int i = 1; i <= group.length; i++) {
            Object caObj = group[i];
            if (!caObj) continue;
            AGV.AllocatableObject ca = caObj.as(AGV.AllocatableObject);
            int locked = 0;
            for (int j = 1; j <= ca.allocations.length; j++)
                if (ca.allocations[j].allocator == locker) { locked = 1; break; }
            if (!locked)
                for (int j = 1; j <= ca.requests.length; j++)
                    if (ca.requests[j].allocator == locker) { locked = 1; break; }
            if (!locked) ca.allocate(locker, 1);
        }
    }
    sleep(0.05);
}
```

按表创建子流程（每行 = 一个子流程，示例：两个方向组）：

| 子流程 | 灯 | Locker | Group |
|--------|-----|--------|-------|
| NS | `TrafficLight_NS` | `TrafficLightLocker_NS` | `NS_Stop_CA_Group` |
| EW | `TrafficLight_EW` | `TrafficLightLocker_EW` | `EW_Stop_CA_Group` |

---

## 第十步：联调验证

- [ ] 灯颜色按 绿 → 黄 → 红 轮转
- [ ] 同一路口同一时刻只有 1 个方向组是绿灯
- [ ] 绿灯方向 AGV 能通行
- [ ] 红灯方向 AGV 在 ControlArea 前停车
- [ ] Output Console 有正确的放行/锁定日志

---

## 常见问题速查

| 问题 | 原因 / 解决 |
|------|------|
| 灯不变色 | ① Global Code 没保存 ② 子实体名不是 `red`/`green`/`yellow` ③ 标签类型写成 string（应 number） |
| 多个灯同时绿灯 | 偏移值算错，重新验算 |
| AGV 被永久锁住 | PF 子流程没启动；或 phase 标签读取异常、deallocate 未执行 |
| 时序漂移 | senddelayedmessage 浮点误差，可定期 `fmod(time(), totalCycle)` 校准 |

---

## 相关文档

- [配置详解](docs/配置详解.md) — 先干嘛后干嘛的完整配置顺序（ControlArea 控制点/Group/Locker/标签/脚本）⭐
- [配时逻辑与标签详解](docs/配时逻辑与标签详解.md) — 偏移时间计算、周期/红灯公式、标签与 lockerName/groupName 详解
- [占用模块](docs/占用模块.md) — 红绿灯占用/放行模块的通用搭建教程
- [控制区域与占用逻辑详解](docs/控制区域与占用逻辑详解.md) — ControlArea 位置、Locker/Group/ControlArea 三级机制、占用时序与常见坑 ⭐
- [模型3D外观与用户库](docs/模型3D外观与用户库.md) — 3D 外观构建（灯柱+3灯子模型、FBX、用户库注册）与工作原理（控制区域位置、占用逻辑）
- [模型blender](docs/模型blender.md) — 红绿灯 3D 视觉模型脚本
- [踩坑记录](docs/踩坑记录.md) — 部署过程中常见错误的排查与修复方法

## License

[MIT](LICENSE)。FlexSim 是 FlexSim Software Products, Inc. 的商业软件，本仓库仅包含文档与 FlexScript 脚本示例，不含 FlexSim 本体。
