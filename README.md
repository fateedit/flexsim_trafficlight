# FlexSim 红绿灯系统（Traffic Light System for FlexSim）

一个基于 FlexSim 的**多路口红绿灯控制模型**：用 TaskExecuter + DrawSurrogate 搭建三色灯，通过标签驱动状态机控制灯的轮转，并结合 **Locker / Group / ControlArea** 对 AGV 路径做占用与放行控制。

> **本仓库包含：搭建文档 + 模型文件**（`models/cs 灯光.fsm`，三个路口完整模型）。模型直接打开即可运行；文档用于理解逻辑、按需修改和从零复现。
> 打开模型需要 **FlexSim 2026**（商业软件，可用学生版/试用版），建议对 FlexSim 的标签、触发器、AGV 模块有一定了解。

## 仓库结构

```
flexsim_trafficlight/
├── README.md                      ← 本文件：模型说明 + 模型原理 + 从零搭建指南
├── models/
│   ├── cs 灯光.fsm                ← 三个路口红绿灯完整模型（FlexSim 2026 打开）
│   ├── trafficlight.fsl           ← 红绿灯用户库文件（注册后可从库面板直接拖出）
│   └── fbx/
│       ├── 杆横向.fbx             ← 灯柱（横臂）3D 模型
│       └── 灯0.fbx                ← 灯体 3D 模型（三个灯复用，着色区分）
└── docs/
    ├── 1号路口-两相位方案.md       ← 1 号路口详细方案（三态循环 94s，两两配对）
    ├── 2号路口-四相位独立灯方案.md  ← 2 号路口详细方案（五态循环 117s，四相位独立灯）
    ├── 3号路口-三相位方向灯方案.md  ← 3 号路口详细方案（五态循环 108s，三相位方向灯）
    ├── 路口配时参数.md             ← 三个路口配时值的权威计算与校验
    ├── 占用模块.md                ← 红绿灯占用/放行模块的通用搭建教程
    ├── 模型3D外观与用户库.md        ← 3D 外观构建（灯柱+3灯/FBX/用户库）与工作原理 ⭐
    ├── 模型blender.md             ← 用 Blender 脚本生成红绿灯 3D 模型的另一种做法
    └── 踩坑记录.md                ← 部署过程中踩过的坑与修复方法
```

## 模型文件与运行要求

- **模型**：`models/cs 灯光.fsm` —— 包含 1/2/3 号三个路口的 11 个红绿灯、AGV 路网与占用/放行逻辑，打开即可运行。
- **打开方式**：FlexSim 2026（中文版）→ File → Open → 选择 `models/cs 灯光.fsm` → 点击 Run。
- **版本要求**：模型为 FlexSim 2026 格式（文件头 `flexsimtree 26.0`），低版本可能无法打开或显示异常。
- **3D 外观依赖（路径要求）**：模型外观使用外部 FBX 模型（`杆横向.fbx` 灯柱 + `灯0.fbx` 灯体），构建时保存的是本机绝对路径。仓库已在 `models/fbx/` 附带这两个文件，换电脑打开后若 3D 显示不全，需在 FlexSim 里重新指定外观路径（选中灯实体 → 外观 → 重新选择 FBX），或把 FBX 复制到原路径。完整说明见 [模型3D外观与用户库](docs/模型3D外观与用户库.md)。
- **用户库**：`models/trafficlight.fsl` 是红绿灯用户库文件——放到 FlexSim 的 libraries 目录（如 `D:\Common\flexsim\libraries\TerminalSim\`），或通过 **工具 → 全局设置 → 用户库 → 添加** 注册后，即可从库面板直接拖出红绿灯实体。
- 三个路口的搭建步骤与逻辑细节见下文及 `docs/` 目录。

## 方案总览

| 路口 | 灯数 | 逻辑 | 周期 | 状态循环 | 锁/放行 |
|------|:---:|------|:---:|------|------|
| 1号 | 4 | 两两配对（NS组 + EW组） | 94s | GREEN→YELLOW→RED | OnMessage + ProcessFlow |
| 2号 | 4 | 四相位独立 | 117s | GREEN→YELLOW→ALLRED→RED→YELLOW2 | ProcessFlow |
| 3号 | 3 | 三相位方向灯 | 108s | 同2号 | ProcessFlow |

每个灯 = 1 个 TaskExecuter + 3 个 DrawSurrogate 子实体（`red` / `green` / `yellow`）。

---

## 模型原理

### 1. 灯 = 状态机

每个灯是一个独立的状态机，用 `phase` 标签记录当前状态，靠 `senddelayedmessage` 定时推进：

- **三态**（1号路口）：`GREEN → YELLOW → RED → GREEN`
- **五态**（2/3号路口）：`GREEN → YELLOW → ALLRED → RED → YELLOW2 → GREEN`

黄灯后的 `ALLRED`（全红）用于清空路口内已进入的车辆，避免下一个方向的车辆提前驶入，防止路口交叉冲突。

### 2. 相位错开 = 偏移时间

同一路口所有灯的**周期必须相等**，各灯用 `偏移时间` 错开绿灯起点，保证任意时刻只有 1 个方向组为 GREEN：

```
灯 N 的偏移 = 前面所有灯的（绿灯 + 黄闪 + 全红）之和
```

模型启动时 OnModelStart 用 `fmod(偏移, 周期)` 算出当前落在哪个状态，所有灯瞬间就处于正确相位，无需等待对时。

三个路口配时值的完整推导与逐灯校验（绿灯窗口互不重叠）见 [路口配时参数](docs/路口配时参数.md)。

### 3. AGV 通行控制 = 占用/放行

灯本身不拦截 AGV，通行控制靠 **Locker + Group + ControlArea** 三级配合：

| 组件 | 作用 |
|------|------|
| `ControlArea` | 铺在 AGV 路径上的禁行/停车区 |
| `Group` | 把同一方向的所有 ControlArea 归为一组 |
| `Locker` | 对一组 ControlArea 统一加锁/解锁 |

**绿灯 = 解除锁定（放行），红灯 = 重新锁定（AGV 在 ControlArea 前停车）**。锁定逻辑有两条路径，结果一致：

- **1号路口**：灯实体 OnMessage 直接锁/放（通过 `lockerName`/`groupName` 标签），另有 ProcessFlow 双保险
- **2/3号路口**：ProcessFlow 子流程轮询 `phase` 标签控制

通用实现细节见 [占用模块](docs/占用模块.md)。

---

## 第一步：Global Code（一个通用函数，三个路口共用）

**位置**：Model → Scripts → Global Code，粘贴一次即可。五态里的 `ALLRED` 显示为红、`YELLOW2` 显示为黄，所以三态/五态可共用：

```cpp
void setLightColor(Object light, string state) {
    treenode redLight = light.find("red");
    treenode greenLight = light.find("green");
    treenode yellowLight = light.find("yellow");

    if (state == "GREEN") {
        if (greenLight) greenLight.color = Color(0, 1, 0, 1);
        if (yellowLight) yellowLight.color = Color(1, 1, 0, 0.3);
        if (redLight) redLight.color = Color(1, 0, 0, 0.3);
    } else if (state == "YELLOW" || state == "YELLOW2") {
        if (greenLight) greenLight.color = Color(0, 1, 0, 0.3);
        if (yellowLight) yellowLight.color = Color(1, 1, 0, 1);
        if (redLight) redLight.color = Color(1, 0, 0, 0.3);
    } else {  // RED 或 ALLRED
        if (greenLight) greenLight.color = Color(0, 1, 0, 0.3);
        if (yellowLight) yellowLight.color = Color(1, 1, 0, 0.3);
        if (redLight) redLight.color = Color(1, 0, 0, 1);
    }
}
```

---

## 第二步：创建灯实体

从 Library 拖 **TaskExecuter** 到场景，放到路口对应位置，按下表命名（共 11 个灯）：

**1号路口（4 个，两两配对）**：

| 名称 | 配对组 | 方向 |
|------|:---:|------|
| `TrafficLight_1_NS_N2S` | NS | 南向北 |
| `TrafficLight_1_NS_S2N` | NS | 北向南 |
| `TrafficLight_1_EW_W2E` | EW | 西向东 |
| `TrafficLight_1_EW_E2W` | EW | 东向西 |

**2号路口（4 个，四相位独立）**：

| 名称 | 方向 |
|------|------|
| `TrafficLight_A` | 南港八路 南向北 |
| `TrafficLight_C` | 南港八路 北向南 |
| `TrafficLight_B` | 自由港大道 西向东 |
| `TrafficLight_D` | 自由港大道 东向西 |

**3号路口（3 个，相位方向灯）**：

| 名称 | 通行方向 |
|------|---------|
| `Phase_EW` | 东西左转 + 直行 |
| `Phase_NS_S` | 南北直行 |
| `Phase_NS_L` | 南北左转 |

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

每个灯 → Properties → Labels，添加 **6 个标签**（三路口统一一套，1号路口 `全红时长` 填 0 即可）：

| 标签名 | 类型 |
|--------|------|
| `绿灯时长` | number |
| `黄闪时长` | number |
| `红灯时长` | number |
| `偏移时间` | number |
| `全红时长` | number |
| `phase` | string |

> 1号路口每个灯**额外**加 2 个**对象类型**标签：`lockerName`、`groupName`，分别指向第六步中该灯对应的 Locker 与 Group（2号、3号路口不加）。

### 配时数值

**1号路口**：

| 标签 | NS_N2S | NS_S2N | EW_W2E | EW_E2W |
|------|:---:|:---:|:---:|:---:|
| 绿灯时长 | 23 | 23 | 20 | 20 |
| 黄闪时长 | 3 | 3 | 3 | 3 |
| 红灯时长 | 68 | 68 | 71 | 71 |
| 偏移时间 | 0 | 0 | 26 | 26 |
| 全红时长 | 0 | 0 | 0 | 0 |
| phase | GREEN | GREEN | GREEN | GREEN |

**2号路口**：

| 标签 | A | C | B | D |
|------|:---:|:---:|:---:|:---:|
| 绿灯时长 | 30 | 18 | 35 | 18 |
| 黄闪时长 | 2 | 2 | 2 | 2 |
| 全红时长 | 2 | 2 | 2 | 2 |
| 红灯时长 | 81 | 93 | 76 | 93 |
| 偏移时间 | 0 | 34 | 56 | 95 |
| phase | GREEN | GREEN | GREEN | GREEN |

**3号路口**：

| 标签 | Phase_EW | Phase_NS_S | Phase_NS_L |
|------|:---:|:---:|:---:|
| 绿灯时长 | 50 | 20 | 20 |
| 黄闪时长 | 3 | 3 | 3 |
| 全红时长 | 3 | 3 | 3 |
| 红灯时长 | 49 | 79 | 79 |
| 偏移时间 | 0 | 56 | 82 |
| phase | GREEN | GREEN | GREEN |

---

## 第四步：通用脚本（每盏灯粘贴三处，一套代码通吃三个路口）

### 4.1 OnModelStart（自动识别三态/五态）

```cpp
/**Custom Code*/
Object light = ownerobject(c);

double greenDur = light.labels["绿灯时长"].value;
double flashDur = light.labels["黄闪时长"].value;
double redDur = light.labels["红灯时长"].value;
double offset = light.labels["偏移时间"].value;
double allRedDur = light.labels["全红时长"].value;   // 1号路口为 0

// 状态序列：全红时长 > 0 → 五态；否则三态
string seq[5]; double dur[5]; int n;
if (allRedDur > 0) {
    n = 5;
    seq[1] = "GREEN";   dur[1] = greenDur;
    seq[2] = "YELLOW";  dur[2] = flashDur;
    seq[3] = "ALLRED";  dur[3] = allRedDur;
    seq[4] = "RED";     dur[4] = redDur;
    seq[5] = "YELLOW2"; dur[5] = flashDur;
} else {
    n = 3;
    seq[1] = "GREEN";   dur[1] = greenDur;
    seq[2] = "YELLOW";  dur[2] = flashDur;
    seq[3] = "RED";     dur[3] = redDur;
}

double totalCycle = 0;
for (int i = 1; i <= n; i++) totalCycle += dur[i];

// 按偏移时间定位当前落在哪个状态
double t = fmod(offset, totalCycle);
int i = 1;
while (i < n && t >= dur[i]) { t -= dur[i]; i++; }

string state = seq[i];
double remaining = dur[i] - t;

light.labels["phase"].value = state;
setLightColor(light, state);
senddelayedmessage(light, remaining, current, state);
```

### 4.2 OnMessage（状态转移 + 可选占用逻辑）

```cpp
/**Custom Code*/
Object light = ownerobject(c);
string state = msgparam(1);   // 由 OnModelStart / 上一次 OnMessage 传入

double greenDur = light.labels["绿灯时长"].value;
double flashDur = light.labels["黄闪时长"].value;
double redDur = light.labels["红灯时长"].value;
double allRedDur = light.labels["全红时长"].value;

// 状态转移：五态含 全红/黄闪2，三态自动跳过
string nextState; double delay;
if (state == "GREEN") {
    nextState = "YELLOW"; delay = flashDur;
} else if (state == "YELLOW") {
    nextState = (allRedDur > 0) ? "ALLRED" : "RED";
    delay = (allRedDur > 0) ? allRedDur : redDur;
} else if (state == "ALLRED") {
    nextState = "RED"; delay = redDur;
} else if (state == "RED") {
    nextState = (allRedDur > 0) ? "YELLOW2" : "GREEN";
    delay = (allRedDur > 0) ? flashDur : greenDur;
} else {  // YELLOW2
    nextState = "GREEN"; delay = greenDur;
}

light.labels["phase"].value = nextState;
setLightColor(light, nextState);

// ===== 占用逻辑（可选）：仅 1号路口配了 lockerName/groupName 标签时生效 =====
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

- 1号路口：NS 两灯同步变、EW 两灯同步变
- 2号路口：A→C→B→D 依次 GREEN
- 3号路口：EW→NS_S→NS_L 依次 GREEN

如不对，依次检查：① 子实体名是否全小写 `red`/`green`/`yellow` ② 标签类型是否 number（不是 string） ③ Global Code 是否保存。

---

## 第六步：Locker 与 Group

| 路口 | Locker ↔ Group |
|------|------|
| 1号 | `TrafficLightLocker_NS` ↔ `NS_Stop_CA_Group`（南北路径全部 ControlArea）；`TrafficLightLocker_EW` ↔ `EW_Stop_CA_Group`（东西路径全部） |
| 2号 | 推荐与1号同构：A、C 共用 NS 套，B、D 共用 EW 套；也可拆 4 套：`TrafficLightLocker_A`↔`A_Stop_CA_Group`、`C`↔`C_Stop_CA_Group`、`B`↔`B_Stop_CA_Group`、`D`↔`D_Stop_CA_Group` |
| 3号 | `Locker_EW`↔`Group_EW`、`Locker_NS_S`↔`Group_NS_S`、`Locker_NS_L`↔`Group_NS_L` |

> 3号路口南北右转路径**不加** ControlArea，右转始终自由。

---

## 第七步：路面加 ControlArea 并归入 Group

在路口各方向的 AGV 路径上添加 **ControlArea** 对象，分别拖入第六步对应的 Group。

---

## 第八步：初始化锁定（一套通用代码，只改三个数组）

放在 Model 的 OnModelStart 或一个独立触发器（在灯启动之后执行）：

```cpp
/**Custom Code*/
// ===== 只需改这三个数组（名称与第二/六步一致）=====
string lightNames[]  = {"灯1", "灯2", "灯3", "灯4"};       // 该路口所有灯
string lockerNames[] = {"Locker_A", "Locker_B"};          // 与 groupNames 一一对应
string groupNames[]  = {"Group_A", "Group_B"};

// 1) 所有灯先置为全红，防止初始时误放行
for (int i = 1; i <= lightNames.length; i++) {
    Object light = Model.find(lightNames[i]);
    if (light) light.labels["phase"].value = "ALLRED";
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

三个路口只需填（照抄即可）：

| 路口 | lightNames | lockerNames / groupNames |
|------|-----------|--------------------------|
| 1号 | `TrafficLight_1_NS_N2S`、`TrafficLight_1_NS_S2N`、`TrafficLight_1_EW_W2E`、`TrafficLight_1_EW_E2W` | `TrafficLightLocker_NS` / `NS_Stop_CA_Group`、`TrafficLightLocker_EW` / `EW_Stop_CA_Group` |
| 2号 | `TrafficLight_A`、`TrafficLight_C`、`TrafficLight_B`、`TrafficLight_D` | 同上（2 套）；或 4 套 A/C/B/D |
| 3号 | `Phase_EW`、`Phase_NS_S`、`Phase_NS_L` | `Locker_EW` / `Group_EW`、`Locker_NS_S` / `Group_NS_S`、`Locker_NS_L` / `Group_NS_L` |

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

按表创建子流程（每行 = 一个子流程）：

| 路口 | 子流程 | 灯 | Locker | Group |
|------|------|-----|--------|-------|
| 1号 | NS | `TrafficLight_1_NS_N2S` + `TrafficLight_1_NS_S2N` | `TrafficLightLocker_NS` | `NS_Stop_CA_Group` |
| 1号 | EW | `TrafficLight_1_EW_W2E` + `TrafficLight_1_EW_E2W` | `TrafficLightLocker_EW` | `EW_Stop_CA_Group` |
| 2号 | NS | `TrafficLight_A` + `TrafficLight_C` | `TrafficLightLocker_NS` | `NS_Stop_CA_Group` |
| 2号 | EW | `TrafficLight_B` + `TrafficLight_D` | `TrafficLightLocker_EW` | `EW_Stop_CA_Group` |
| 3号 | EW | `Phase_EW` | `Locker_EW` | `Group_EW` |
| 3号 | NS_S | `Phase_NS_S` | `Locker_NS_S` | `Group_NS_S` |
| 3号 | NS_L | `Phase_NS_L` | `Locker_NS_L` | `Group_NS_L` |

---

## 第十步：联调验证

- [ ] 灯颜色按预期轮转
- [ ] 同一路口同一时刻只有 1 个方向组是 GREEN
- [ ] GREEN 方向 AGV 能通行
- [ ] 非 GREEN 方向 AGV 在 ControlArea 前停车
- [ ] Output Console 有正确的放行/锁定日志

---

## 常见问题速查

| 问题 | 原因 / 解决 |
|------|------|
| 灯不变色 | ① Global Code 没保存 ② 子实体名不是 `red`/`green`/`yellow` ③ 标签类型写成 string（应 number） |
| 多个灯同时 GREEN | 偏移值算错，重新验算 |
| AGV 被永久锁住 | PF 子流程没启动；或 phase 标签读取异常、deallocate 未执行 |
| 时序漂移 | senddelayedmessage 浮点误差，可定期 `fmod(time(), totalCycle)` 校准 |

---

## 相关文档

- [1号路口-两相位方案](docs/1号路口-两相位方案.md) — 1 号路口的详细配时、脚本与占用逻辑
- [2号路口-四相位独立灯方案](docs/2号路口-四相位独立灯方案.md) — 2 号路口的四相位独立灯详细方案
- [3号路口-三相位方向灯方案](docs/3号路口-三相位方向灯方案.md) — 3 号路口的三相位方向灯详细方案
- [路口配时参数](docs/路口配时参数.md) — 三个路口配时值的权威计算与校验
- [占用模块](docs/占用模块.md) — 红绿灯占用/放行模块的通用搭建教程
- [模型3D外观与用户库](docs/模型3D外观与用户库.md) — 3D 外观构建（灯柱+3灯子模型、FBX、用户库注册）与工作原理（控制区域位置、占用逻辑）
- [模型blender](docs/模型blender.md) — 红绿灯 3D 视觉模型脚本
- [踩坑记录](docs/踩坑记录.md) — 部署过程中常见错误的排查与修复方法

## License

[MIT](LICENSE)。FlexSim 是 FlexSim Software Products, Inc. 的商业软件，本仓库仅包含文档与 FlexScript 脚本示例，不含 FlexSim 本体。
