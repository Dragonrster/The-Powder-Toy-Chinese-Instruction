# 附录U：自定义元素开发教程

本教程将指导你完成为 The Powder Toy 添加全新元素的完整流程。从环境搭建到编译运行，从数据结构到回调函数，从 C++ 源码修改到 Lua 脚本方案，每一步都有详细说明。

更新日期：2026-05-22 | 适用版本：The-Powder-Toy-Chinese（基于最新源码）

---

## 目录

1. [两种开发路径概述](#1-两种开发路径概述)
2. [前置准备](#2-前置准备)
3. [元素数据结构详解](#3-元素数据结构详解)
4. [回调函数详解](#4-回调函数详解)
5. [完整实战：创建"MAGIC"元素](#5-完整实战创建magic元素)
6. [编译与测试](#6-编译与测试)
7. [元素注册流程](#7-元素注册流程)
8. [Lua 脚本自定义元素](#8-lua-脚本自定义元素)
9. [最佳实践与常见陷阱](#9-最佳实践与常见陷阱)
10. [参考资源](#10-参考资源)

---

## 1. 两种开发路径概述

为 TPT 添加自定义元素有两条路。它们适合不同场景，互补而非对立。

### C++ 源码魔改（本教程重点）

直接在 TPT 的 C++ 源码中编写元素逻辑，重新编译后成为游戏的一部分。

| 优点 | 缺点 |
|------|------|
| 性能最优（编译为机器码） | 需要搭建编译环境 |
| 可访问所有内部 API | 修改后必须重新编译 |
| 能实现任意复杂的行为逻辑 | 分发需要提供整个可执行文件 |
| 自定义渲染完全自由 | 学习曲线较陡 |

### Lua 脚本方案（参见第 8 节）

在游戏内通过 Lua 脚本动态创建元素。不需要编译，实时可调。

| 优点 | 缺点 |
|------|------|
| 即时生效，无需编译 | 性能低于原生 C++ |
| 可在存档中打包分发 | 无法使用全部内部 API |
| 学习门槛低 | 渲染能力有限 |
| 上手快，适合原型验证 | 复杂互动行为实现困难 |

**建议路径**：先用 Lua 脚本做原型验证，确认设计合理后再用 C++ 正式实现以获得最佳性能。

---

## 2. 前置准备

### 2.1 C++ 基础知识

你需要了解以下 C++ 概念：

- **基本语法**：变量、函数、条件、循环
- **指针与引用**：TPT 大量使用指针传递仿真状态
- **结构体（struct）**：元素属性存储在 `Particle` 和 `Element` 结构体中
- **位运算**：属性标志位通过按位 OR ( `|` ) 和按位 AND ( `&` ) 组合与检测
- **宏（macro）**：TPT 使用大量预处理器宏简化代码
- **静态函数与回调**：元素行为通过函数指针绑定

如果你对这些不熟悉，建议先阅读一篇 C++ 入门教程（重点看结构体、指针、位运算三章即可）。

### 2.2 编译环境搭建

TPT 使用 **Meson** 构建系统 + **Ninja** 构建后端。

#### Windows

```bash
# 1. 安装 MSYS2 (https://www.msys2.org/)
# 2. 在 MSYS2 终端中安装工具链
pacman -S mingw-w64-x86_64-gcc
pacman -S mingw-w64-x86_64-meson
pacman -S mingw-w64-x86_64-ninja
pacman -S mingw-w64-x86_64-SDL2
pacman -S mingw-w64-x86_64-luajit
pacman -S mingw-w64-x86_64-curl
pacman -S mingw-w64-x86_64-fftw
pacman -S mingw-w64-x86_64-jsoncpp
pacman -S mingw-w64-x86_64-libpng
pacman -S mingw-w64-x86_64-bzip2

# 3. 配置并编译
cd The-Powder-Toy-Chinese
meson setup build
meson compile -C build
```

#### Linux

```bash
# Ubuntu/Debian
sudo apt install g++ meson ninja-build libsdl2-dev libluajit-5.1-dev \
  libcurl4-openssl-dev libfftw3-dev libjsoncpp-dev libpng-dev libbz2-dev

# 配置并编译
cd The-Powder-Toy-Chinese
meson setup build
meson compile -C build
```

### 2.3 源码结构速览

```
The-Powder-Toy-Chinese/
├── src/
│   ├── simulation/
│   │   ├── Element.h              ← 元素类定义（数据结构核心）
│   │   ├── Element.cpp            ← Element 构造函数
│   │   ├── ElementClasses.h       ← 元素编号宏定义 (PT_NONE=0, PT_DUST=1...)
│   │   ├── ElementClasses.cpp     ← 元素实例化（调用所有 Element_XXX() 函数）
│   │   ├── ElementCommon.h        ← 所有元素文件必须包含的头
│   │   ├── ElementDefs.h          ← 类型常量、属性标志位、回调宏定义
│   │   ├── ElementGraphics.h      ← 渲染上下文 GraphicsFuncContext
│   │   ├── Particle.h             ← 粒子数据结构
│   │   ├── Simulation.h           ← Simulation 类（part_create 等核心 API）
│   │   ├── SimulationData.h       ← 墙类常量、仿真数据
│   │   ├── MenuSection.h          ← 菜单分类常量 (SC_ELEC, SC_LIQUID...)
│   │   ├── TransitionConstants.h  ← IPL, IPH, ITL, ITH, NT, ST
│   │   ├── elements/
│   │   │   ├── meson.build        ← 元素文件列表（注册新元素的核心位置）
│   │   │   ├── WATR.cpp           ← 示例：水的实现
│   │   │   ├── FIRE.cpp           ← 示例：火的实现（有 Update/Graphics/Create）
│   │   │   ├── GLOW.cpp           ← 示例：荧光液（有复杂的 Graphics）
│   │   │   ├── CRAY.cpp           ← 示例：物质射线（有 CtypeDraw）
│   │   │   ├── BOMB.cpp           ← 示例：炸弹（简单的 Update/Graphics）
│   │   │   ├── NONE.cpp           ← 示例：空元素（有 IconGenerator）
│   │   │   └── ...（195个元素文件）
│   │   └── gravity/               ← 牛顿重力相关
│   └── graphics/
│       └── Pixel.h                ← RGB 类和 _rgb 字面量
```

**关键文件**：你只需要关心 `Element.h`（了解数据结构）和 `elements/` 目录下的文件（编写元素逻辑）。

---

## 3. 元素数据结构详解

每个元素都是一个 `Element` 类的实例。在 `Element::Element_XXX()` 函数中填充所有字段。以下是每个字段的完整说明。

### 3.1 基础标识

```cpp
Identifier = "DEFAULT_PT_MYEL";  // 内部唯一标识字符串，格式 "DEFAULT_PT_XXXX"
Name = "MYEL";                   // 游戏中显示的4字母简称
Colour = 0xFF8040_rgb;           // 菜单图标颜色（ARGB格式，_rgb后缀）
```

**Identifier 规范**：必须为 `"DEFAULT_PT_" + 元素名`，用于 Lua API 查找元素。

**Colour**：使用 `_rgb` 字面量，格式为 `0xRRGGBB_rgb`。Alpha 通道由系统自动设为 0xFF。

### 3.2 菜单与启用

```cpp
MenuVisible = 1;      // 1=在菜单中显示, 0=隐藏（如 SPRK 火花通常不直接绘制）
MenuSection = SC_SOLIDS;  // 所属菜单分类，参见下表
Enabled = 1;          // 1=启用, 0=禁用（禁用后粒子自动死亡）
```

**菜单分类常量 (MenuSection)**：

| 常量 | 值 | 中文名 | 典型元素 |
|------|-----|--------|---------|
| SC_WALL | 0 | 墙壁 | 各类墙体 |
| SC_ELEC | 1 | 电子元件 | METL, PSCN, NSCN, SPRK |
| SC_POWERED | 2 | 电动元件 | SWCH, PUMP, GPMP |
| SC_SENSOR | 3 | 传感器 | TSNS, PSNS, LSNS |
| SC_FORCE | 4 | 力场 | ACEL, DCEL, FRAY |
| SC_EXPLOSIVE | 5 | 爆炸物 | FIRE, BOMB, GUNP |
| SC_GAS | 6 | 气体 | GAS, SMKE, O2 |
| SC_LIQUID | 7 | 液体 | WATR, OIL, LAVA |
| SC_POWDERS | 8 | 粉末 | DUST, SAND, SALT |
| SC_SOLIDS | 9 | 固体 | BRCK, WOOD, GLAS |
| SC_NUCLEAR | 10 | 核材料 | URAN, PLUT, NEUT |
| SC_SPECIAL | 11 | 特殊 | CLNE, VOID, BHOL |
| SC_LIFE | 12 | 生命 | LIFE (GOL) |
| SC_TOOL | 13 | 工具 | HEAT, COOL, PROP |
| SC_FAVORITES | 14 | 收藏 | 用户收藏 |
| SC_DECO | 15 | 装饰 | 装饰工具 |

**MenuSort**：同菜单中排序位置，默认为 0。数值越大排序越靠后。大多数元素不设置此字段（即 0）。

### 3.3 物理属性

```cpp
Advection = 0.0f;        // 惯性系数 (0~1)，越大粒子越"滑"，保持运动趋势越强
AirDrag = 0.04f * CFDS;  // 空气阻力，乘以 CFDS(=4.0/CELL) 进行单位转换
AirLoss = 0.90f;         // 与空气交互时的能量损失率 (0~1)，1=无损失
Loss = 0.00f;            // 每帧速度损失率 (0~1)，速度乘以 (1-Loss)
Collision = 0.0f;        // 碰撞弹性 (0~1)，1=完全弹性碰撞
Gravity = 0.0f;          // 重力系数，0=无重力, 0.1=较轻, 0.3=正常粉末
NewtonianGravity = 0.0f; // 牛顿重力系数，仅在牛顿重力模式下生效
Diffusion = 0.00f;       // 扩散系数，越大随机运动越强（气体通常较高）
HotAir = 0.000f * CFDS;  // 热气上升速率，乘以 CFDS
Falldown = 0;            // 下落行为: 0=不下落, 1=常规下落, 2=液体式下落
```

**物理属性速查表**（参考典型元素）：

| 元素类型 | Advection | AirDrag | Gravity | Diffusion | Falldown |
|---------|-----------|---------|---------|-----------|----------|
| 粉末 (DUST) | 0.7 | 0.04×CFDS | 0.3 | 0.00 | 1 |
| 液体 (WATR) | 0.6 | 0.01×CFDS | 0.1 | 0.00 | 2 |
| 固体 (METL) | 0.0 | 0.00 | 0.0 | 0.00 | 0 |
| 气体 (GAS) | 1.5 | 0.01×CFDS | 0.0 | 0.75 | 0 |
| 能量 (PHOT) | 0.0 | 0.00 | 0.0 | 0.00 | 0 |
| 火焰 (FIRE) | 0.9 | 0.04×CFDS | -0.1 | 0.00 | 1 |

**注意**：
- 所有含 `CFDS` 的字段必须乘以 `CFDS` 常量进行归一化。直接写 `0.04f * CFDS` 不要省略。
- `Gravity` 为负数时粒子会向上升（如火焰的 -0.1）。
- `Falldown=2` 时粒子有液体般的水平扩散行为。

### 3.4 反应属性

```cpp
Flammable = 0;    // 可燃性，0=不可燃, 数值越大越容易被火焰/PLSM/LAVA点燃
Explosive = 0;    // 爆炸性，0=不爆炸, 1=可燃爆炸, >1=更剧烈
Meltable = 0;     // 可熔化性，0=不可熔化, 数值越大越容易被高温熔化
Hardness = 1;     // 硬度，1=最软(受酸腐蚀等), 数值越大越耐腐蚀/破坏
```

**Flammable** 非零值说明：实际点燃概率取决于 `Flammable + 压力×10`。数值越大越易点燃，示例：
- WOOD（木头）：Flammable=20，极易点燃
- OIL（油）：Flammable=10，较易点燃
- COAL（煤）：Flammable=0 但在 FIRE/PLSM 接触时有特殊燃烧逻辑

**Explosive** 的数值仅决定被点燃时的强度（影响压力增加量），布尔含义由是否 >0 决定。

**Meltable** 非零值表示可被高温/LAVA熔化。数值影响熔化概率（越大越易熔化）。

**Hardness** 范围 0~1000：
- 1：最软（大多数普通元素）
- 20：中等（BOMB 炸弹的硬度，避免被轻易压碎）
- 0：特殊（FIGH 打手硬度为 0，表示不适用此系统）

### 3.5 重量与热导率

```cpp
Weight = 100;        // 重量 (1~100)，决定密度排序和重力影响
HeatConduct = 0;     // 热传导率 (0~255)，0=绝热, 255=瞬时导热
HeatCapacity = 1.0f; // 体积热容（每像素），必须非零，默认 1.0
```

**Weight 参考**：
- 轻气体：1~5（GAS=1, SMKE=1）
- 液体：20~40（WATR=35, OIL=10, LAVA=100）
- 固体：50~100（METL=100, WOOD=50, GLAS=50）
- 重元素：100（DMND=100, BRCK=100）

**HeatConduct 参考**（取值 0~255）：
- 0：绝热体（INSL, VOID, VACUUM-like elements）
- 29：较差导体（BOMB）
- 44：中等导体（GLOW）
- 88：良好导体（FIRE）
- 251：优秀导体（METL, CLNE, 金属类）
- 255：完美导体（WATR 等液体）
- 参见 [附录B：导热速度表](appendix-B-导热速度表.md)

### 3.6 属性标志位 (Properties)

`Properties` 是 32 位无符号整数，通过按位 OR (`|`) 组合多个标志：

```cpp
Properties = TYPE_SOLID | PROP_CONDUCTS | PROP_HOT_GLOW | PROP_LIFE;
```

**完整标志位清单**：

| 标志 | 位值 | 说明 | 使用场景 |
|------|------|------|---------|
| TYPE_PART | 0x01 | 粉末型：受重力下落，可堆积 | DUST, SAND, SALT |
| TYPE_LIQUID | 0x02 | 液体型：流动，受重力 | WATR, OIL, LAVA |
| TYPE_SOLID | 0x04 | 固体型：静止不动 | METL, BRCK, WOOD |
| TYPE_GAS | 0x08 | 气体型：高扩散，无重力 | GAS, SMKE, O2 |
| TYPE_ENERGY | 0x10 | 能量型：无重力，特殊移动 | PHOT, NEUT, PROT |
| PROP_CONDUCTS | 0x20 | 导电：可被 SPRK 激活 | METL, PSCN, NSCN |
| PROP_PHOTPASS | 0x40 | **光子穿透**：PHOT 可穿过，可能在内部折射 | GLAS, WATR, INSL |
| PROP_NEUTPENETRATE | 0x80 | **中子穿透+反应**：NEUT 穿过时触发反应 | DEUT, PLUT, URAN |
| PROP_NEUTABSORB | 0x100 | **中子吸收**：NEUT 被吸收（默认是反射） | TTAN, WATR |
| PROP_NEUTPASS | 0x200 | **中子通过**：NEUT 穿过但不触发反应 | GLAS, INSL |
| PROP_DEADLY | 0x400 | **致命**：对火柴人 (STKM/STKM2) 造成伤害 | ACID, LAVA, PLSM |
| PROP_HOT_GLOW | 0x800 | **高温发光**：高温时自动产生辉光效果 | METL, LAVA, FIRE |
| PROP_LIFE | 0x1000 | **有生命值**：life 字段有意义，会在调试模式显示 | LIFE, DEUT, TRON |
| PROP_RADIOACTIVE | 0x2000 | **放射性**：对火柴人持续辐射伤害 | URAN, PLUT, POLO |
| PROP_LIFE_DEC | 0x4000 | **生命衰减**：每帧 life 递减 1 | DEUT（部分衰变） |
| PROP_LIFE_KILL | 0x8000 | **生命耗尽自杀**：life ≤ 0 时死亡 | TRON（消失） |
| PROP_LIFE_KILL_DEC | 0x10000 | **生命耗尽衰减**：life 递减至 ≤ 0 时死亡 | FIRE（燃烧后消失） |
| PROP_SPARKSETTLE | 0x20000 | **火花沉降**：作为 EMBR 爆炸碎片的行为 | EMBR, FIRE |
| PROP_NOAMBHEAT | 0x40000 | **无环境热交换**：不与环境进行热交换 | CFLM, TESC |
| PROP_NOCTYPEDRAW | 0x100000 | **不绘制 Ctype**：使用自定义绘制而非 ctype 外观 | LAVA（显示熔岩自身外观） |

**注意**：
- 五种类型标志（`TYPE_PART`, `TYPE_LIQUID`, `TYPE_SOLID`, `TYPE_GAS`, `TYPE_ENERGY`）必须且只能选一个。它们是互斥的。
- `PROP_PHOTPASS`(0x40) 不是附录P中的 0x800(2048)。0x40 是源码实际值。历史原因：旧文档中按十进制列出，这里以源码 `ElementDefs.h` 为准。
- `PROP_LIFE_DEC` 和 `PROP_LIFE_KILL_DEC` 的区别：前者仅递减但不一定自杀（需要额外的 kill 逻辑），后者在递减到 ≤0 时自动杀死粒子。

**组合示例**：
```cpp
// 导电固体
Properties = TYPE_SOLID | PROP_CONDUCTS | PROP_HOT_GLOW;

// 致命液体
Properties = TYPE_LIQUID | PROP_DEADLY | PROP_PHOTPASS;

// 可燃气体（火焰）
Properties = TYPE_GAS | PROP_LIFE_DEC | PROP_LIFE_KILL | PROP_SPARKSETTLE;

// 高质量固体（有生命值做特殊用途）
Properties = TYPE_SOLID | PROP_LIFE;
```

### 3.7 CarriesTypeIn -- 类型携带字段

`CarriesTypeIn` 指定元素通过哪个粒子字段存储其"携带的元素类型"（如 CLNE 通过 ctype 记录它要复制什么）。

```cpp
CarriesTypeIn = 1U << FIELD_CTYPE;  // 通过 ctype 携带类型
CarriesTypeIn = 1U << FIELD_TMP;    // 通过 tmp 携带类型
```

`FIELD_xxx` 常量（在 `Particle.h` 中定义）：

| 常量 | 值 | 对应字段 |
|------|-----|---------|
| FIELD_TYPE | 0 | type（类型本身） |
| FIELD_LIFE | 1 | life（生命值） |
| FIELD_CTYPE | 2 | ctype（携带类型——最常用） |
| FIELD_TMP | 9 | tmp（临时数据 1） |
| FIELD_TMP2 | 10 | tmp2（临时数据 2） |
| FIELD_TMP3 | 11 | tmp3（临时数据 3） |
| FIELD_TMP4 | 12 | tmp4（临时数据 4） |

**大多数元素不设置 CarriesTypeIn**（默认为 0，不携带任何类型）。典型用例：
- CLNE（复制体）：`1U << FIELD_CTYPE`，将其 ctype 作为要复制的目标类型
- FIRE（火焰）：`1U << FIELD_CTYPE`，将 ctype 用于记录特殊火焰状态
- CRAY（物质射线）：`1U << FIELD_CTYPE`，将 ctype 作为要发射的元素类型

### 3.8 状态转换

每个元素有四种自动状态转换检测（由引擎自动处理，不需要手动写代码）：

```cpp
LowPressure = IPL;                  // 低压阈值（低于此值触发转换）
LowPressureTransition = NT;         // 低压转换目标（NT=无转换）
HighPressure = IPH;                 // 高压阈值（高于此值触发转换）
HighPressureTransition = NT;        // 高压转换目标
LowTemperature = ITL;               // 低温阈值 (K)
LowTemperatureTransition = NT;      // 低温转换目标
HighTemperature = ITH;              // 高温阈值 (K)
HighTemperatureTransition = NT;     // 高温转换目标
```

**特殊常量**（定义在 `TransitionConstants.h`）：

| 常量 | 值 | 含义 |
|------|-----|------|
| IPL | MIN_PRESSURE - 1 | "永不触发的低压"，即禁用低压转换 |
| IPH | MAX_PRESSURE + 1 | "永不触发的高压"，即禁用高压转换 |
| ITL | MIN_TEMP - 1 | "永不触发的低温"，即禁用低温转换 |
| ITH | MAX_TEMP + 1 | "永不触发的高温"，即禁用高温转换 |
| NT | -1 | "无转换"（实际效果：杀死粒子 — PT_NONE） |
| ST | PT_NUM | "特殊转换"（仅在特定代码块中处理，如 LAVA） |

**带转换的实例**：
```cpp
// 水：高温蒸发为蒸汽
HighTemperature = 373.0f;           // 100℃ = 373.15K
HighTemperatureTransition = PT_WTRV;

// 火焰：高温转为等离子体
HighTemperature = 2773.0f;
HighTemperatureTransition = PT_PLSM;

// 冰：高温融化
HighTemperature = 273.15f;          // 0℃
HighTemperatureTransition = PT_WATR;

// 打手：高温着火
HighTemperature = 620.0f;
HighTemperatureTransition = PT_FIRE;
```

**注意**：
- 温度单位为开尔文 (K)，室温为 `R_TEMP + 273.15f = 295.15K`（22℃）。
- 这四个转换由引擎在 `Simulation::part_change_type` 和每帧更新中自动检测，不需要在 Update 回调中手动处理。
- 转换目标必须是一个合法的元素 ID（如 `PT_WATR`），不能填 0（`PT_NONE`）。填 `NT`(-1) 相当于杀死粒子。

### 3.9 DefaultProperties -- 默认粒子属性

新创建的粒子会从 `DefaultProperties` 获得初始值：

```cpp
DefaultProperties.temp = R_TEMP + 273.15f;   // 默认室温 (295.15K)
DefaultProperties.life = 0;                   // 默认生命值
DefaultProperties.tmp = 0;                    // 默认临时值 1
DefaultProperties.tmp2 = 0;                   // 默认临时值 2
DefaultProperties.ctype = 0;                  // 默认携带类型
```

**常用预设**：
```cpp
// 火焰：初始高温
DefaultProperties.temp = R_TEMP + 400.0f + 273.15f;  // 约 695K (422℃)

// 冰：初始低温
DefaultProperties.temp = 273.15f - 28.0f;            // 约 245K (-28℃)

// 火柴人：初始生命值 100
DefaultProperties.life = 100;

// CLNE：携带类型为 WATR（被其他元素碰到后改变）
DefaultProperties.ctype = PT_WATR;
```

### 3.10 PhotonReflectWavelengths

光子反射波长掩码。当 PHOT 击中此元素时，PHOT 的波长与此值进行 AND 运算，只有两者共有的波长成分会被反射。

```cpp
PhotonReflectWavelengths = 0xFFFFFFFF;  // 默认：反射所有波长（白光照单全收）
PhotonReflectWavelengths = 0x00000000;  // 不反射任何光（光子被吸收）
```

多数元素使用默认值 `0xFFFFFFFF`，只有 FILT（滤光片）等特殊元素修改此值。一般新元素可直接省略此字段。

### 3.11 描述文本

```cpp
Description = ByteString("你元素的中文描述").FromUtf8();
```

**规范**：
- 描述应当简洁，不超过一行。
- 使用 `ByteString(...).FromUtf8()` 以支持中文 UTF-8 编码。
- 描述会显示在游戏菜单中鼠标悬停元素的 tooltip 上。

---

## 4. 回调函数详解

回调函数是元素行为的核心。它们通过函数指针绑定到 `Element` 实例上。

### 4.1 Update() -- 每帧行为

```cpp
// 函数签名（由宏 UPDATE_FUNC_ARGS 展开）
static int update(Simulation* sim, int i, int x, int y,
                  int surround_space, int nt,
                  Parts &parts, int pmap[YRES][XRES]);

// 绑定
Update = &update;
```

**参数说明**：

| 参数 | 类型 | 说明 |
|------|------|------|
| sim | Simulation* | 仿真器指针，调用 `sim->create_part()`, `sim->kill_part()` 等 |
| i | int | 当前粒子在 parts 数组中的索引 |
| x, y | int | 当前粒子的像素坐标 |
| surround_space | int | 周围空间状态（1=有空位） |
| nt | int | 转换类型（内部使用，通常忽略） |
| parts | Parts& | 粒子数组引用，通过 `parts[i].xxx` 读写 |
| pmap | int\[YRES\]\[XRES\] | 粒子占用图，通过 `pmap[y][x]` 检测某个位置是否有粒子 |

**返回值**：
- `0`：无变化（标准返回）
- `1`：粒子已死亡或发生重大变化，需要通知引擎跳过后续默认行为

**完整代码示例**（加热周围粒子）：

```cpp
#include "simulation/ElementCommon.h"

// 前向声明
static int update(UPDATE_FUNC_ARGS);

void Element::Element_MYEL()
{
    // ... 基础属性设置 ...
    Update = &update;
}

static int update(UPDATE_FUNC_ARGS)
{
    // 获取仿真数据引用（全局只读配置）
    auto &sd = SimulationData::CRef();
    auto &elements = sd.elements;

    // 遍历周围 3×3 区域（包括自己的位置）
    for (auto rx = -1; rx <= 1; rx++)
    {
        for (auto ry = -1; ry <= 1; ry++)
        {
            // 跳过自己
            if (rx == 0 && ry == 0)
                continue;

            // 获取邻居位置的粒子
            auto r = pmap[y+ry][x+rx];
            if (!r)
                continue;  // 该位置无粒子

            // 提取邻居的类型和索引
            int rt = TYP(r);   // 类型（使用 TYP 宏从 pmap 编码值中提取）
            int ri = ID(r);    // 粒子索引（使用 ID 宏从 pmap 编码值中提取）

            // 排除自己和其他特定类型
            if (rt == PT_MYEL)
                continue;

            // 加热邻居粒子
            parts[ri].temp += 0.5f;  // 每帧升温 0.5K

            // 限制最高温度
            if (parts[ri].temp > MAX_TEMP)
                parts[ri].temp = MAX_TEMP;
        }
    }

    // 自身缓慢升温
    parts[i].temp += 0.1f;

    return 0;  // 无重大变化
}
```

**关键宏**：
- `TYP(r)`：从 pmap 编码值中提取元素类型
- `ID(r)`：从 pmap 编码值中提取粒子数组索引
- `PMAP(id, typ)`：组合 id 和 type 为 pmap 编码值
- `TYP(parts[i].ctype)`：用于提取 ctype 字段的高位类型信息

**重要**：`surround_space` 参数是由引擎传入的，表示该粒子周围是否有空位。这在编写粉末/液体类元素下落逻辑时有用（当 `surround_space` 为 0 时，周围完全被填满，可跳过下落尝试）。

### 4.2 Graphics() -- 自定义渲染

```cpp
// 函数签名（由宏 GRAPHICS_FUNC_ARGS 展开）
static int graphics(GraphicsFuncContext &gfctx, const Particle *cpart,
                    int nx, int ny, int *pixel_mode,
                    int* cola, int *colr, int *colg, int *colb,
                    int *firea, int *firer, int *fireg, int *fireb);

// 绑定
Graphics = &graphics;
```

**参数说明**：

| 参数 | 类型 | 说明 |
|------|------|------|
| gfctx | GraphicsFuncContext& | 渲染上下文（含随机数生成器 rng 等） |
| cpart | const Particle* | 当前粒子数据（只读） |
| nx, ny | int | 渲染坐标 |
| pixel_mode | int* | 像素模式标志位（通过 `|=` 设置） |
| cola/colr/colg/colb | int* | 主颜色（ARGB），修改它们改变像素外观 |
| firea/firer/fireg/fireb | int* | 火焰叠加颜色，用于发光/辉光效果 |

**像素模式标志位 (pixel_mode)**：

| 标志 | 说明 |
|------|------|
| PMODE_NONE | 清除所有标志 |
| PMODE_FLAT | 平面渲染（无阴影等效果） |
| PMODE_FLARE | 镜头光晕（用于爆炸等强光源） |
| PMODE_GLOW | 辉光效果（向外渗透的柔和光晕） |
| PMODE_BLUR | 模糊效果 |
| PMODE_ADD | 叠加混合（颜色叠加而非替换，适合发光效果） |
| FIRE_ADD | 火焰叠加混合 |

返回值：`0`=动态颜色（不缓存，每帧重新计算），`1`=静态颜色（可以缓存）。

**完整代码示例**（基于温度变化颜色 + 辉光效果）：

```cpp
static int graphics(GRAPHICS_FUNC_ARGS)
{
    // 基于生命值计算 RGB 颜色（用于动态色彩循环）
    int life = cpart->life;
    *colr = (life * 3) % 255;      // 红色通道随 life 循环
    *colg = (life * 5) % 255;      // 绿色通道随 life 循环（不同频率）
    *colb = (life * 7) % 255;      // 蓝色通道随 life 循环
    *cola = 255;                    // 完全不透明

    // 基于温度添加辉光效果
    float tempC = cpart->temp - 273.15f;  // 转为摄氏度
    if (tempC > 50.0f)
    {
        // 温度超过 50℃ 时产生火焰光晕
        int glowStrength = (int)((tempC - 50.0f) / 2.0f);
        if (glowStrength > 128)
            glowStrength = 128;

        *firea = glowStrength;       // 火焰 Alpha
        *firer = *colr;              // 火焰颜色跟随主色
        *fireg = *colg;
        *fireb = *colb;
        *pixel_mode |= FIRE_ADD;     // 启用火焰叠加
    }

    *pixel_mode |= PMODE_GLOW;       // 始终有辉光效果
    return 0;  // 动态颜色，不缓存
}
```

**实际参考**：
- FIRE 元素的 Graphics：使用 `Renderer::flameTableAt()` 查找火焰颜色表，并设置 `FIRE_ADD` 和 `PMODE_NONE` 来产生火焰叠加效果。
- GLOW 元素的 Graphics：根据温度和压力动态计算 RGB + 辉光，在亮度高时使用 `FIRE_ADD + PMODE_GLOW + PMODE_ADD`，亮度低时使用 `PMODE_BLUR`。

### 4.3 Create() -- 创建时回调

```cpp
// 函数签名
static void create(ELEMENT_CREATE_FUNC_ARGS);
// 展开为: Simulation *sim, int i, int x, int y, int t, int v

// 绑定
Create = &create;
```

在粒子被创建后（已被添加到粒子数组）立即调用。用于设置初始状态。

```cpp
static void create(ELEMENT_CREATE_FUNC_ARGS)
{
    // 设置随机生命值
    sim->parts[i].life = sim->rng.between(120, 169);

    // 设置初始温度
    sim->parts[i].temp = R_TEMP + 200.0f + 273.15f;  // 约 495K

    // 设置初始颜色
    sim->parts[i].dcolour = 0xFF8040FF;  // ARGB
}
```

**注意**：
- `t` 参数是创建此粒子的原因类型（创建者），`v` 是创建速度参数。
- 此回调在 `sim->create_part()` 内部自动调用。
- 不要在 Create 回调中调用 `sim->create_part()` 创建同类型粒子（会无限递归）。

### 4.4 CreateAllowed() -- 放置限制

```cpp
// 函数签名
static bool createAllowed(ELEMENT_CREATE_ALLOWED_FUNC_ARGS);
// 展开为: Simulation *sim, int i, int x, int y, int t

// 绑定
CreateAllowed = &createAllowed;
```

返回 `true` 允许创建粒子，返回 `false` 阻止创建。用于限制元素放置条件。

```cpp
static bool createAllowed(ELEMENT_CREATE_ALLOWED_FUNC_ARGS)
{
    // 只允许在固体表面上创建
    int r = sim->pmap[y+1][x];  // 检查下方
    if (!r)
        return false;  // 下方为空，不允许创建

    int rt = TYP(r);
    // 只能放在固体上
    if (sim->elements[rt].Properties & TYPE_SOLID)
        return true;

    return false;
}
```

**实际用例**：
- STKM/STKM2（火柴人）：限制每张地图最多 1 个，通过检查已有的 STKM 计数实现。
- SPAWN/SPAWN2（重生点）：限制地图中同时存在的 SPAWN 数量。
- FIGH（打手）：限制数量并检查是否已有同类。

### 4.5 ChangeType() -- 类型变更处理器

```cpp
// 函数签名
static void changeType(ELEMENT_CHANGETYPE_FUNC_ARGS);
// 展开为: Simulation *sim, int i, int x, int y, int from, int to

// 绑定
ChangeType = &changeType;
```

当粒子被 `sim->part_change_type()` 转换为其他类型时触发。`from` 是原始类型，`to` 是目标类型。

```cpp
static void changeType(ELEMENT_CHANGETYPE_FUNC_ARGS)
{
    // 当 STKM 被转换为其他类型时，释放占用的玩家槽位
    if (to != PT_STKM && to != PT_STKM2 && to != PT_FIGH)
    {
        // 清理资源（例如释放分配的内存/索引）
        Free(sim, sim->parts[i].tmp);
    }
}
```

**实际用例**：
- STKM/STKM2：当火柴人被杀死或转换时，释放占用的玩家槽位。
- SPAWN/SPAWN2：清理重生点相关数据。
- SOAP：转换时进行清理。

### 4.6 CtypeDraw() -- 自定义 Ctype 渲染

```cpp
// 函数签名
static bool ctypeDraw(CTYPEDRAW_FUNC_ARGS);
// 展开为: Simulation *sim, int i, int t, int v

// 绑定
CtypeDraw = &ctypeDraw;
```

当其他元素绘制（draw on）此元素时，决定此元素的 ctype 外观如何显示。主要用于让元素显示出其 ctype 指向的元素外观。

```cpp
static bool ctypeDraw(CTYPEDRAW_FUNC_ARGS)
{
    // 调用默认的 ctype 渲染（基本框架）
    if (!Element::ctypeDrawVInCtype(CTYPEDRAW_FUNC_SUBCALL_ARGS))
        return false;

    // 特殊处理：如果携带 LIGH，设置特殊标志
    if (t == PT_LIGH)
    {
        sim->parts[i].ctype |= PMAPID(30);
    }

    // 使用 ctype 对应元素的默认温度
    sim->parts[i].temp = elements[t].DefaultProperties.temp;
    return true;
}
```

**内置辅助方法**：
- `Element::basicCtypeDraw(CTYPEDRAW_FUNC_ARGS)`：最基本的 ctype 绘制
- `Element::ctypeDrawVInTmp(CTYPEDRAW_FUNC_ARGS)`：在 tmp 字段中查找 ctype 的变体
- `Element::ctypeDrawVInCtype(CTYPEDRAW_FUNC_ARGS)`：在 ctype 字段中查找 ctype 的变体

**实际用例**：
- CRAY（物质射线）：`CtypeDraw = &Element::ctypeDrawVInCtype`（使用内置方法）
- CLNE（复制体）：`CtypeDraw = &Element::ctypeDrawVInTmp`（使用内置方法）
- NONE（擦除工具）：没有 CtypeDraw（不需要）

### 4.7 IconGenerator() -- 菜单图标生成

```cpp
// 函数签名
static std::unique_ptr<VideoBuffer> iconGen(int wallID, Vec2<int> size);

// 绑定
IconGenerator = &iconGen;
```

生成自定义菜单图标的函数。返回包含图标像素数据的 VideoBuffer。

```cpp
#include "graphics/VideoBuffer.h"

static std::unique_ptr<VideoBuffer> iconGen(int wallID, Vec2<int> size)
{
    // 创建指定尺寸的纹理
    auto texture = std::make_unique<VideoBuffer>(size);

    // 在纹理中心绘制字符
    texture->BlendChar(
        size / 2 - Vec2(4, 2),  // 居中位置
        0xE06C,                  // 字符码（Unicode 私有区域）
        0xFF0000_rgb.WithAlpha(0xFF)  // 颜色
    );

    return texture;
}
```

**实际用例**：NONE 元素使用 IconGenerator 绘制菜单中的擦除图标。绝大多数元素不设置此字段，使用默认的颜色方块图标。

---

## 5. 完整实战：创建"MAGIC"元素

### 5.1 元素设计

我们来创建一个名为 **MAGIC**（魔法水晶）的元素。它的行为如下：

- **类型**：固体（TYPE_SOLID）
- **外观**：颜色随 life 值在彩虹色之间循环变化，带辉光效果
- **行为**：缓慢加热周围粒子，自身温度也随之升高
- **生命周期**：life 值每帧递减，life 耗尽时力竭消失（转为 FIRE 闪烁）
- **创建**：初始 life 在 200~300 之间随机
- **状态转换**：温度超过 1000K 时变为 LAVA

效果预览：
- 放置后为亮色固体，颜色随时间缓慢变化
- 周围粒子逐渐被加热
- 最终自己也会过热熔化（或 life 耗尽后闪烁消失）

### 5.2 文件结构

在 `src/simulation/elements/` 目录下创建两个文件：

```
src/simulation/elements/
├── MAGIC.cpp   ← 元素主要逻辑
└── MAGIC.h     ← 头文件（如有需要）
```

### 5.3 完整代码

#### MAGIC.h

```cpp
// MAGIC.h - 魔法水晶元素头文件
#pragma once
// 如有需要在其他元素中引用的函数，可在此声明
// 本示例中不需要额外导出
```

#### MAGIC.cpp（完整代码 + 中文注释）

```cpp
// MAGIC.cpp - 魔法水晶元素实现
// 一个示例自定义元素，演示 Update、Graphics、Create 回调的完整用法

#include "simulation/ElementCommon.h"

// ==================== 前向声明 ====================
static int  update(UPDATE_FUNC_ARGS);    // 每帧更新回调
static int  graphics(GRAPHICS_FUNC_ARGS); // 渲染回调
static void create(ELEMENT_CREATE_FUNC_ARGS); // 创建回调

// ==================== 元素注册函数 ====================
// 此函数由 ElementClasses.cpp 通过宏自动调用
// 函数名必须遵循格式: Element::Element_XXX()
void Element::Element_MAGIC()
{
    // ---------- 基础标识 ----------
    Identifier = "DEFAULT_PT_MAGIC";  // 内部唯一标识
    Name = "MAGI";                    // 游戏内显示名（4字母限制，但可以用~别名）
    Colour = 0xFF00FF_rgb;           // 菜单图标颜色：品红
    MenuVisible = 1;                 // 在菜单中可见
    MenuSection = SC_SPECIAL;        // 放在"特殊"分类
    Enabled = 1;                     // 启用

    // ---------- 物理属性 ----------
    // 固体不需要 Advection/Gravity/Diffusion 等
    Advection = 0.0f;                // 惯性：0（固体不动）
    AirDrag = 0.00f * CFDS;          // 空气阻力：无
    AirLoss = 1.00f;                 // 空气能量损失：无损失
    Loss = 0.00f;                    // 速度损失：无
    Collision = 0.0f;                // 碰撞弹性：无
    Gravity = 0.0f;                  // 重力：无（固体不受重力）
    Diffusion = 0.00f;               // 扩散：无
    HotAir = 0.000f * CFDS;          // 热气上升：无
    Falldown = 0;                    // 下落行为：0=不下落

    // ---------- 反应属性 ----------
    Flammable = 0;                   // 不可燃
    Explosive = 0;                   // 不爆炸
    Meltable = 0;                    // 不可被 LAVA 等间接熔化（我们有手动转换逻辑）
    Hardness = 10;                   // 中等硬度，不会轻易被破坏

    // ---------- 重量与导热 ----------
    Weight = 100;                    // 重量 100（重固体）
    HeatConduct = 128;               // 中等热传导率（0~255，128 为中等偏上）
    // HeatCapacity 使用默认值 1.0f
    Description = ByteString("魔法水晶,持续发光并加热周围粒子,温度过高时会熔化").FromUtf8();

    // ---------- 属性标志位 ----------
    // TYPE_SOLID: 固体型，静止不动
    // PROP_HOT_GLOW: 高温时自动辉光（与手动 Graphics 辉光叠加）
    // PROP_LIFE: 有生命值，调试模式下显示
    // PROP_LIFE_DEC: 每帧 life 自动递减 1
    // PROP_LIFE_KILL_DEC: life 递减至 ≤0 时自动死亡
    Properties = TYPE_SOLID | PROP_HOT_GLOW | PROP_LIFE | PROP_LIFE_DEC | PROP_LIFE_KILL_DEC;

    // 不携带任何类型（大多数元素不设置 CarriesTypeIn）
    // CarriesTypeIn = 0;  // 默认值，可省略

    // ---------- 状态转换 ----------
    // 禁用所有自动状态转换（除了高温）
    LowPressure = IPL;                      // 永不触发的低压
    LowPressureTransition = NT;             // "无转换" = 不转换
    HighPressure = IPH;                     // 永不触发的高压
    HighPressureTransition = NT;
    LowTemperature = ITL;                   // 永不触发的低温
    LowTemperatureTransition = NT;
    HighTemperature = 1000.0f;              // 高温阈值：1000K (约 727℃)
    HighTemperatureTransition = PT_LAVA;    // 超过阈值变为岩浆

    // ---------- 默认粒子属性 ----------
    // 初始室温
    DefaultProperties.temp = R_TEMP + 273.15f;   // 22℃ + 273.15 = 295.15K
    // life/tmp/tmp2/ctype 在 Create() 回调和引擎中设置默认值 0

    // ---------- 回调函数绑定 ----------
    Update = &update;          // 每帧更新
    Graphics = &graphics;      // 自定义渲染
    Create = &create;          // 粒子创建时初始化
}

// ==================== 创建回调 ====================
// 每当有 MAGIC 粒子被创建时（通过菜单放置、CLNE 复制等），此函数被调用
static void create(ELEMENT_CREATE_FUNC_ARGS)
{
    // 设置随机的初始生命值（决定颜色周期的起始相位）
    // between(a, b) 返回 [a, b] 范围内的随机整数
    sim->parts[i].life = sim->rng.between(200, 300);

    // 设置初始温度略高于室温（模拟魔法能量的初始热量）
    sim->parts[i].temp = R_TEMP + 50.0f + 273.15f;  // 约 345K (72℃)
}

// ==================== 每帧更新回调 ====================
// 引擎每帧为每个 MAGIC 粒子调用此函数
static int update(UPDATE_FUNC_ARGS)
{
    // 获取仿真数据引用（全局只读配置：元素属性表等）
    auto &sd = SimulationData::CRef();
    auto &elements = sd.elements;

    // ---------- 第一步：加热周围粒子 ----------
    // 遍历以自己为中心的 3×3 区域
    for (auto rx = -1; rx <= 1; rx++)
    {
        for (auto ry = -1; ry <= 1; ry++)
        {
            // 不处理自己所在的位置
            if (rx == 0 && ry == 0)
                continue;

            // 获取邻居位置 (x+rx, y+ry) 的粒子
            // pmap[y][x] 存储该位置的粒子编码（包含类型和索引）
            auto r = pmap[y + ry][x + rx];
            if (!r)
                continue;  // 该位置为空，跳过

            // 提取邻居的类型和索引
            int rt = TYP(r);   // 使用 TYP 宏从编码值中提取类型 ID
            int ri = ID(r);    // 使用 ID 宏从编码值中提取粒子数组索引

            // 跳过同类型的 MAGIC 粒子（避免无限自加热）
            // PT_MAGIC 是在 ElementClasses.h 中由宏自动生成的常量
            if (rt == PT_MAGIC)
                continue;

            // 跳过一些特殊元素（DMND 金刚石应该几乎不受影响）
            if (rt == PT_DMND)
                continue;

            // 加热邻居粒子（每帧 0.8K 的升温速率）
            // MAGIC 的魔法能量向周围辐射热量
            parts[ri].temp += 0.8f;

            // 安全钳：不超过最高温度限制
            if (parts[ri].temp > MAX_TEMP)
                parts[ri].temp = MAX_TEMP;
        }
    }

    // ---------- 第二步：自身因辐射热量而缓慢冷却 ----------
    // 每帧散失少量热量（模拟能量向外辐射的代价）
    // 但不低于室温（限制到 R_TEMP + 273.15f 以上）
    parts[i].temp -= 0.05f;
    if (parts[i].temp < R_TEMP + 273.15f)
        parts[i].temp = R_TEMP + 273.15f;

    // ---------- 第三步：life 特殊行为 ----------
    // life 在 50-100 之间时：魔法水晶开始不稳定闪烁
    if (parts[i].life < 100 && parts[i].life > 50)
    {
        // 随机加速加热（魔法能量不稳定释放）
        // chance(1, N) 返回 true 的概率为 1/N
        if (sim->rng.chance(1, 10))  // 10% 概率
        {
            parts[i].temp += 5.0f;   // 快速升温
        }
    }

    // ---------- 第四步：life 即将耗尽时的效果 ----------
    if (parts[i].life <= 10)
    {
        // 魔法水晶在消失前剧烈闪烁
        // 通过 tmp 字段标记闪烁状态（用于 Graphics 渲染）
        parts[i].tmp = 1;  // 1=闪烁中

        // 最后一次猛烈加热周围
        for (auto rx = -2; rx <= 2; rx++)
        {
            for (auto ry = -2; ry <= 2; ry++)
            {
                if (rx == 0 && ry == 0)
                    continue;
                auto r = pmap[y + ry][x + rx];
                if (!r)
                    continue;
                int ri = ID(r);
                if (TYP(r) == PT_MAGIC)
                    continue;
                parts[ri].temp += 2.0f;  // 猛烈的最后热量
                if (parts[ri].temp > MAX_TEMP)
                    parts[ri].temp = MAX_TEMP;
            }
        }
    }

    // 返回 0 表示"无重大变化"（粒子仍然存活且位置不变）
    // 返回 1 表示粒子已死亡或被移动，引擎应跳过后续默认处理
    return 0;
}

// ==================== 渲染回调 ====================
// 决定每个 MAGIC 粒子在屏幕上的外观
static int graphics(GRAPHICS_FUNC_ARGS)
{
    // 获取生命值和温度（只读）
    int life = cpart->life;
    float tempK = cpart->temp;         // 开尔文温度
    float tempC = tempK - 273.15f;     // 转为摄氏度
    int isFlashing = cpart->tmp;       // 读取 Update 中设置的闪烁标记

    // ---------- 计算基础颜色 ----------
    // 颜色随 life 值在色轮上循环（产生彩虹般的渐变效果）
    // 使用正弦函数制造平滑的颜色过渡
    // life 值每帧递减 1，所以颜色会持续缓慢变化

    // 相位偏移：红绿蓝三个通道有不同的偏移角度，形成色轮效果
    float phase = (float)life * 0.1f;  // life 每 62 帧完成一个完整色轮循环

    int r = (int)((sin(phase)             + 1.0f) * 127.0f);   // 红通道
    int g = (int)((sin(phase + 2.094f)    + 1.0f) * 127.0f);  // 绿通道 (偏移 2π/3)
    int b = (int)((sin(phase + 4.188f)    + 1.0f) * 127.0f);  // 蓝通道 (偏移 4π/3)

    // 如果正在闪烁（life <= 10），颜色快速切换（明暗交替）
    if (isFlashing == 1)
    {
        // 使用 gfctx.rng 获取一个每帧都不同的随机值
        // 这样每帧随机明暗，产生"闪烁"效果
        int flicker = gfctx.rng.between(100, 255);
        r = r * flicker / 255;
        g = g * flicker / 255;
        b = b * flicker / 255;
    }

    // 限制颜色在有效范围内
    *colr = r > 255 ? 255 : (r < 0 ? 0 : r);
    *colg = g > 255 ? 255 : (g < 0 ? 0 : g);
    *colb = b > 255 ? 255 : (b < 0 ? 0 : b);
    *cola = 255;  // 完全不透明

    // ---------- 辉光和火焰效果 ----------
    // 温度越高，辉光越强
    if (tempC > 30.0f)
    {
        // 计算辉光强度（基于温度，有上限）
        int glowStrength = (int)((tempC - 30.0f) / 3.0f);
        if (glowStrength > 120)
            glowStrength = 120;

        *firea = glowStrength;     // 火焰 Alpha
        *firer = *colr;            // 火焰颜色跟随主色
        *fireg = *colg;
        *fireb = *colb;
        *pixel_mode |= FIRE_ADD;   // 启用火焰叠加混合模式
    }

    // 始终添加辉光效果（life 值越大，辉光越强）
    int glowBase = life / 4;
    if (glowBase > 60)
        glowBase = 60;
    if (*firea < glowBase)
    {
        *firea = glowBase;
        *firer = *colr;
        *fireg = *colg;
        *fireb = *colb;
    }

    *pixel_mode |= PMODE_GLOW;   // 启用辉光
    *pixel_mode &= ~PMODE_FLAT;  // 不使用平面渲染（保持光照效果）

    // 返回 0 表示颜色是动态的（缓存系统不应缓存此粒子的渲染结果）
    // 因为我们的颜色每帧都在变化
    return 0;
}
```

### 5.4 代码要点讲解

**颜色系统**：使用 `sin()` 函数的三相偏移实现彩虹色轮。三个正弦波各偏移 2π/3（约 2.094 和 4.188 弧度），保证 RGB 三通道始终有一组协调的颜色。

**加热逻辑**：3×3 区域遍历是 TPT 中最常见的空间扫描模式。使用 `pmap[y][x]` 而非直接数组访问，因为 pmap 是结构化的，包含类型和索引的编码信息。

**闪烁效果**：通过 `gfctx.rng.between()` 在渲染回调中使用随机数，每次渲染产生不同的明暗值。这与 Update 中的 `sim->rng.chance()` 不同——Graphics 中的 rng 只影响视觉，不影响模拟。

**返回值**：`return 0` 告诉引擎"不要缓存此粒子颜色"，因为我们的颜色每帧都在变化。如果元素有固定的颜色，可以 `return 1` 以提高渲染性能。

---

## 6. 编译与测试

### 6.1 注册构建文件

在编译之前，必须将新元素添加到构建系统中。编辑以下文件：

#### 第一步：编辑 `meson.build`

打开 `src/simulation/elements/meson.build`，在 `simulation_elem_names` 列表中添加 `'MAGIC'`。

列表按元素 ID 顺序排列（ID = 数组索引）。新元素需要添加到列表末尾附近：

```python
# 在 meson.build 的 simulation_elem_names 列表末尾添加：
simulation_elem_names = [
    'NONE',
    'DUST',
    # ... 省略已有元素 ...
    'SEED',
    'MAGIC',     # ← 新元素名称
]
```

**位置说明**：列表的顺序决定了元素的 Type ID（NONE=0, DUST=1, ... MAGIC=新ID）。将新元素放在末尾可避免影响已有元素的 ID。

**警告**：如果元素列表中间有空缺（某些元素是 disabler），ID 序号可能与列表位置不完全一致。meson.build 中的 `elem_id` 累加逻辑会自动处理。但为了安全，新元素一般添加到列表末尾。

#### 第二步：注册 ElementClasses.h 宏

TPT 使用代码生成。在 `src/simulation/ElementNumbers.template.h` 中有一个占位符，由构建脚本在 configure 阶段自动替换。**你不需要手动编辑此文件**。

但是，为了让 C++ 编译器能找到 `PT_MAGIC` 常量，你需要**重新配置 meson**：

```bash
meson setup build --reconfigure
```

这一步会重新扫描 `meson.build` 中的元素列表，自动生成包含 `PT_MAGIC` 的 `ElementNumbers.h`。

### 6.2 编译

```bash
# Windows (MSYS2)
cd /path/to/The-Powder-Toy-Chinese
meson setup build --reconfigure
meson compile -C build

# Linux
cd /path/to/The-Powder-Toy-Chinese
meson setup build --reconfigure
meson compile -C build
```

**常见编译错误与修复**：

| 错误信息 | 原因 | 解决方法 |
|---------|------|---------|
| `'PT_MAGIC' was not declared in this scope` | 未重新配置 meson | 运行 `meson setup build --reconfigure` |
| `undefined reference to Element::Element_MAGIC()` | MAGIC.cpp 未加入构建 | 确认 `meson.build` 中已添加 `'MAGIC'` |
| `no matching function for call to 'sin(float&)'` | 缺少 `#include <cmath>` | `ElementCommon.h` 已自动包含 `<cmath>` |
| `'CFDS' was not declared` | 缺少 ElementCommon.h 包含 | 确保第一行是 `#include "simulation/ElementCommon.h"` |
| `'SimulationData' has not been declared` | 错误使用引用 | 应在 Update 函数内获取 `auto &sd = SimulationData::CRef();` |
| linker error: 重复定义 | 函数忘记加 `static` 关键字 | 回调函数必须是 `static` |
| `'pmap' was not declared in this scope` | 函数签名错误 | 确保 Update 函数使用 `UPDATE_FUNC_ARGS` 宏 |

### 6.3 在游戏中测试

1. **启动编译好的 TPT**
2. **按 D 键开启调试模式**（第一次按是 DEBUG_PARTS 模式，第四次是 DEBUG_PARTICLE 模式）
3. **在特殊分类 (SC_SPECIAL) 中找到 MAGI 图标**，点击并放置几个粒子
4. **验证功能**：

   - **颜色变化**：观察粒子颜色是否随时间缓慢变化（彩虹色循环）
   - **辉光**：粒子周围应该有辉光
   - **加热周围**：在旁边放一些冷水 (WATR)，观察水温是否上升
   - **过热熔化**：用 HEAT 工具加热 MAGIC 粒子到 1000K 以上，应转变为 LAVA（岩浆）
   - **life 耗尽**：等待足够长时间（200~300 帧 ≈ 3~5 秒），life 耗尽后粒子消失

5. **使用 PROP 工具验证**：
   - 按 `P` 键选择 PROP 工具
   - 点击一个 MAGIC 粒子
   - 查看弹出的属性面板：Type 应为 MAGI, temp 应为 345K 左右, life 应在 200~300 之间

6. **性能测试**：用大面积笔刷放置大量 MAGIC 粒子（如 1000 个），观察 FPS 是否明显下降。按 D 键到 DEBUG_SIMHUD 模式查看帧率。

### 6.4 调试技巧

**添加日志输出**：在 Update 中使用 `tpt.log()`（需要包含对应的头文件）或使用标准输出：

```cpp
#include <cstdio>
// 在 update 中：
printf("MAGIC[%d]: life=%d, temp=%.1fK\n", i, parts[i].life, parts[i].temp);
```

**禁用特定行为以隔离问题**：临时注释掉一部分代码，逐步定位 bug。

**对比参考元素**：遇到问题时，打开 `FIRE.cpp`、`GLOW.cpp`、`CLNE.cpp` 等官方元素源码，对比结构和方法。

---

## 7. 元素注册流程（总结）

完整的新元素注册需要以下 3 步。每次添加新元素都遵循相同的流程。

### 第一步：编写元素源文件

在 `src/simulation/elements/` 下创建 `XXX.cpp`（和可选的 `XXX.h`）：

```cpp
// XXX.cpp
#include "simulation/ElementCommon.h"

static int update(UPDATE_FUNC_ARGS);

void Element::Element_XXX()
{
    Identifier = "DEFAULT_PT_XXX";
    Name = "XXXX";
    Colour = 0xFF0000_rgb;
    MenuVisible = 1;
    MenuSection = SC_SOLIDS;
    Enabled = 1;

    // ... 所有物理属性 ...

    Update = &update;
}

static int update(UPDATE_FUNC_ARGS)
{
    // ... 行为逻辑 ...
    return 0;
}
```

### 第二步：修改 meson.build

在 `src/simulation/elements/meson.build` 的 `simulation_elem_names` 列表中添加元素名：

```python
simulation_elem_names = [
    # ... 已有元素 ...
    'SEED',
    'XXX',     # ← 添加此行
]
```

### 第三步：重新配置并编译

```bash
meson setup build --reconfigure
meson compile -C build
```

**不需要手动编辑的文件**（构建系统自动生成）：
- `ElementNumbers.h`：由 `meson configure` 从 `ElementNumbers.template.h` 和 `meson.build` 自动生成
- `ElementClasses.cpp`：不需要修改（通过宏自动调用所有 `Element_XXX()`）
- `ElementClasses.h`：不需要修改（宏自动生成所有 `PT_XXX` 常量）

**自动注册原理**：

1. `meson.build` 中的元素名列表被构建脚本转换为 `ELEMENT_NUMBERS` 宏的内容
2. 这个宏在 `ElementClasses.h` 中展开，为每个元素生成 `constexpr int PT_XXX = id;` 常量
3. 同一个宏在 `Element.h` 中声明所有 `void Element_XXX();` 函数
4. `ElementClasses.cpp` 中的 `GetElements()` 函数通过宏调用所有 `Element_XXX()` 初始化函数
5. 编译时，编译器将你写的 `MAGIC.cpp` 编译为目标文件，链接器将其连接到最终可执行文件

---

## 8. Lua 脚本自定义元素

如果你不想折腾 C++ 编译环境，Lua 方案提供了即时可用的轻量级替代。

### 8.1 基本流程

Lua 自定义元素通过三个步骤创建：分配 → 设置属性 → 设置回调。

```lua
-- ==================== 第一步：分配新元素 ====================
-- elements.allocate("菜单分类", "元素显示名")
-- 返回值：元素 ID（整数），失败返回 -1
local myElemId = elements.allocate("SPECIAL", "MYELEM")

if myElemId == -1 then
    tpt.log("元素分配失败！可能已达到元素数量上限")
    return
end

-- ==================== 第二步：设置基本属性 ====================
elements.element(myElemId, {
    -- 基础标识
    Name = "MYEL",                   -- 4字母简称
    Colour = 0xFF4488,               -- ARGB 颜色

    -- 菜单配置
    MenuVisible = true,              -- true=可见, false=隐藏
    MenuSection = 11,                -- 菜单分类 (11=SC_SPECIAL, 见上文常量表)
    Enabled = true,                  -- true=启用

    -- 描述
    Description = "我的自定义元素，每帧加热周围粒子",

    -- 物理属性
    Advection = 0.0,                 -- 固体不需要惯性
    AirDrag = 0.0,
    AirLoss = 1.0,
    Loss = 0.0,
    Collision = 0.0,
    Gravity = 0.0,
    Diffusion = 0.0,
    HotAir = 0.0,
    Falldown = 0,

    -- 反应属性
    Flammable = 0,
    Explosive = 0,
    Meltable = 0,
    Hardness = 10,

    -- 重量与导热
    Weight = 100,
    HeatConduct = 128,

    -- 属性标志位（按位 OR 组合）
    Properties = 4 | 0x800,          -- TYPE_SOLID(=4) | PROP_HOT_GLOW(=0x800)

    -- 状态转换
    LowPressure = -257,              -- 永不触发的低压（= IPL）
    LowPressureTransition = -1,      -- 无转换（NT）
    HighPressure = 257,              -- 永不触发的高压（= IPH）
    HighPressureTransition = -1,
    LowTemperature = -1,             -- 永不触发的低温
    LowTemperatureTransition = -1,
    HighTemperature = 1000,          -- 高温阈值 1000K
    HighTemperatureTransition = 6,   -- PT_LAVA = 6（注意：Lua 中只能用数字 ID）
})
```

### 8.2 设置回调函数

```lua
-- ==================== Create 回调 ====================
elements.element(myElemId, {
    Create = function(i, x, y, t, v)
        -- 设置初始生命值 200~300
        sim.partProperty(i, "life", math.random(200, 300))
        -- 设置初始温度
        sim.partProperty(i, "temp", 295.15 + 50)  -- 约 345K
    end
})

-- ==================== Update 回调 ====================
-- 函数签名: function(i, x, y, surround_space, nt)
-- 返回: 0=无变化, 1=有变化
elements.element(myElemId, {
    Update = function(i, x, y, surround_space, nt)
        -- 获取当前温度
        local temp = sim.partProperty(i, "temp")

        -- 遍历周围 3×3 区域并加热邻居
        for rx = -1, 1 do
            for ry = -1, 1 do
                if rx ~= 0 or ry ~= 0 then
                    local r = sim.partID(x + rx, y + ry)
                    if r then
                        local neighborType = sim.partProperty(r, "type")
                        -- 获取该元素的 Properties
                        local neighborProps = elements.property(neighborType, "Properties")
                        -- 跳过同类型和不存在的
                        if neighborType ~= myElemId then
                            local nt = sim.partProperty(r, "temp")
                            sim.partProperty(r, "temp", nt + 0.8)
                        end
                    end
                end
            end
        end

        -- 自身缓慢冷却（向外辐射能量）
        sim.partProperty(i, "temp", temp - 0.05)

        -- life 递减（Lua 中需要通过 PROP_LIFE_DEC 标志或手动递减）
        local life = sim.partProperty(i, "life")
        if life > 0 then
            sim.partProperty(i, "life", life - 1)
        else
            -- life 耗尽，杀死粒子
            sim.partKill(i)
            return 1
        end

        return 0
    end
})

-- ==================== Graphics 回调 ====================
-- 函数签名: function(i, colr, colg, colb)
-- 返回: cache, pixel_mode, cola, colr, colg, colb, firea, firer, fireg, fireb
elements.element(myElemId, {
    Graphics = function(i, colr, colg, colb)
        local life = sim.partProperty(i, "life")
        local temp = sim.partProperty(i, "temp")

        -- 颜色随 life 循环变化（彩虹效果）
        local phase = life * 0.1
        local r = (math.sin(phase) + 1) * 127
        local g = (math.sin(phase + 2.094) + 1) * 127
        local b = (math.sin(phase + 4.188) + 1) * 127

        -- 辉光强度基于温度
        local firea_val = 60
        if temp > 273.15 + 30 then
            firea_val = math.min((temp - 273.15 - 30) / 3, 120)
        end

        -- 返回值的顺序是固定的：
        -- return cache, pixel_mode, cola, colr, colg, colb, firea, firer, fireg, fireb
        -- cache=0: 动态颜色，不缓存
        -- PMODE_GLOW=0x200, FIRE_ADD=0x800（这些值来自 C++ 源码）
        return 0, 0x200 | 0x800, 255, r, g, b, firea_val, r, g, b
    end
})
```

### 8.3 完整 Lua 自定义元素脚本

```lua
-- ============================================================
-- 完整 Lua 自定义元素示例：彩虹加热水晶
-- 将此脚本保存为 .lua 文件，在 TPT 中通过脚本管理器加载
-- ============================================================

-- 第一步：分配元素
local myElemId = elements.allocate("SPECIAL", "RAINBOWCRYSTAL")
if myElemId == -1 then
    tpt.log("错误：无法分配自定义元素")
    return
end

-- 第二步：设置所有基本属性
elements.element(myElemId, {
    Name = "RBCR",
    Colour = 0xFF00FF,
    MenuVisible = true,
    MenuSection = 11,   -- SC_SPECIAL
    Enabled = true,
    Description = "彩虹水晶,加热周围粒子,颜色随生命周期变化",
    Advection = 0.0,
    AirDrag = 0.0,
    AirLoss = 1.0,
    Loss = 0.0,
    Collision = 0.0,
    Gravity = 0.0,
    Diffusion = 0.0,
    HotAir = 0.0,
    Falldown = 0,
    Flammable = 0,
    Explosive = 0,
    Meltable = 0,
    Hardness = 10,
    Weight = 100,
    HeatConduct = 128,
    Properties = 4 | 0x800,  -- TYPE_SOLID | PROP_HOT_GLOW
    LowPressure = -257.0,
    LowPressureTransition = -1,
    HighPressure = 257.0,
    HighPressureTransition = -1,
    LowTemperature = -1.0,
    LowTemperatureTransition = -1,
    HighTemperature = 1000.0,
    HighTemperatureTransition = 6,  -- PT_LAVA
})

-- 第三步：Create 回调
elements.element(myElemId, {
    Create = function(i, x, y, t, v)
        sim.partProperty(i, "life", math.random(200, 300))
        sim.partProperty(i, "temp", 295.15 + 50)  -- ~345K
    end
})

-- 第四步：Update 回调
elements.element(myElemId, {
    Update = function(i, x, y, surround_space, nt)
        local temp = sim.partProperty(i, "temp")
        local life = sim.partProperty(i, "life")

        -- 加热周围 3×3 区域
        for rx = -1, 1 do
            for ry = -1, 1 do
                if rx ~= 0 or ry ~= 0 then
                    local r = sim.partID(x + rx, y + ry)
                    if r then
                        local rt = sim.partProperty(r, "type")
                        if rt ~= myElemId then
                            local nt_val = sim.partProperty(r, "temp")
                            sim.partProperty(r, "temp", math.min(nt_val + 0.8, 9999))
                        end
                    end
                end
            end
        end

        -- 自身缓慢冷却
        sim.partProperty(i, "temp", math.max(temp - 0.05, 295.15))

        -- 生命值管理
        if life <= 0 then
            sim.partKill(i)
            return 1
        else
            sim.partProperty(i, "life", life - 1)
        end

        -- life 低时的不稳定阶段
        if life < 100 and life > 50 then
            if math.random(1, 10) == 1 then
                sim.partProperty(i, "temp", temp + 5)
            end
        end

        return 0
    end
})

-- 第五步：Graphics 回调
elements.element(myElemId, {
    Graphics = function(i, colr, colg, colb)
        local life = sim.partProperty(i, "life")
        local temp = sim.partProperty(i, "temp")

        -- 彩虹色轮
        local phase = life * 0.1
        local r = math.floor((math.sin(phase) + 1) * 127)
        local g = math.floor((math.sin(phase + 2.094) + 1) * 127)
        local b = math.floor((math.sin(phase + 4.188) + 1) * 127)

        -- 温度辉光
        local firea_val = 60
        if temp > 273.15 + 30 then
            firea_val = math.floor(math.min((temp - 273.15 - 30) / 3, 120))
        end

        -- 返回：cache=0(不缓存), pixel_mode=PMODE_GLOW|FIRE_ADD, cola=255, colr/g/b, firea/firer/fireg/fireb
        return 0, 0x200 | 0x800, 255, r, g, b, firea_val, r, g, b
    end
})

tpt.log("自定义元素 RAINBOWCRYSTAL (RBCR) 已加载！")
tpt.log("元素 ID: " .. myElemId)
tpt.log("在特殊分类(SPECIAL)中找到它")
```

### 8.4 Lua 方案的局限性

与 C++ 方案相比，Lua 自定义元素有以下限制：

1. **性能**：Lua 回调每帧被解释执行，大量粒子时性能明显低于 C++
2. **无 IconGenerator**：无法自定义菜单图标
3. **无 CtypeDraw 回调**：不支持自定义的 ctype 渲染
4. **无 ChangeType 回调**：无法在类型转换时做清理
5. **无 CreateAllowed 回调**：无法限制放置条件
6. **受限的渲染能力**：Graphics 回调的参数和模式较少
7. **无法访问内部 API**：不能直接操作 pmap、bmap 等底层数据结构
8. **状态转换 ID 限制**：Lua 中只能用数字 ID 指定转换目标（如 PT_LAVA=6 而不是 "LAVA"），因为 `properties` 表字段接受的是数值

**建议**：先用 Lua 快速验证元素设计，效果满意后再移植到 C++ 以获得最佳完整性和性能。

---

## 9. 最佳实践与常见陷阱

### 9.1 命名规范

| 项目 | 规范 | 示例 |
|------|------|------|
| 元素文件名 | 大写4字母 + .cpp | `MAGIC.cpp` |
| 元素标识符 | "DEFAULT_PT_" + 大写名 | `"DEFAULT_PT_MAGIC"` |
| 元素显示名 | 大写4字母（或用~的别名） | `"MAGI"` |
| 构造函数 | `Element_` + 大写名 | `Element_MAGIC()` |
| 回调函数 | 小写动词 | `update`, `graphics`, `create` |
| 函数声明 | 必须 `static` | `static int update(UPDATE_FUNC_ARGS);` |

### 9.2 性能考虑

1. **减少不必要的遍历**：Update 中只遍历必要的范围。3×3 区域（9个邻居）是大多数元素的标准。避免在每帧遍历整个地图。

2. **使用正确的返回值**：
   - `return 0` 于 Update：元素仍然存活，位置未变
   - `return 1` 于 Update：元素已死亡或被移动，引擎跳过后续处理
   - `return 0` 于 Graphics：颜色动态变化（不缓存，逐帧重绘）
   - `return 1` 于 Graphics：颜色固定（可缓存，提升性能）

3. **避免在 Create 中创建同类型粒子**：会形成无限递归。如果确实需要，确保有条件跳出。

4. **限制随机数使用**：`sim->rng.chance()` 和 `sim->rng.between()` 每次调用都有开销。在高频代码路径中（如遍历 5×5 区域），如果不需要每粒子独立随机，考虑把随机逻辑提前。

5. **温度钳制**：每次修改温度后都应用 `restrict_flt()` 或手动限制到 `[MIN_TEMP, MAX_TEMP]`，防止温度溢出导致未定义行为。

6. **Graphics 返回值**：如果元素颜色永远不变，`return 1` 让引擎缓存渲染结果。如果颜色频繁变化，`return 0`。

### 9.3 向后兼容性

1. **不要修改已有元素的 ID**：修改 `meson.build` 中已有元素的顺序会被坏所有存档。新元素始终添加到列表末尾。

2. **不要删除/禁用已有元素**：如果元素不再需要，改为设置 `Enabled = 0` 而不是移除它。移除会导致其他元素中对它 ID 的引用失效。

3. **DefaultProperties 的保守设置**：确保初始温度、生命值等在合法范围内，避免 Lua 脚本或现有存档因新建元素而崩溃。

4. **CarriesTypeIn 的兼容性**：如果设置了这个字段，确保对应的粒子字段（ctype/tmp 等）初始值是合法的，或者在你的 Update 中处理非法值。

### 9.4 常见陷阱

| 陷阱 | 说明 | 预防方法 |
|------|------|---------|
| 忘记 `static` 关键字 | 回调函数必须是 `static`，否则链接时符号冲突 | 每个回调函数前加 `static` |
| 类型标志互斥 | 同时使用 `TYPE_SOLID \| TYPE_LIQUID` 导致未定义行为 | 五个类型标志只选一个 |
| 越界访问 pmap | 读写 `pmap[y+ry][x+rx]` 前不检查边界 | 添加 `if (x+rx >= 0 && x+rx < XRES && y+ry >= 0 && y+ry < YRES)` |
| 写只读粒子 | Graphics 回调中 `cpart` 是 `const Particle*` | 只读 cpart，需要修改时在 Update 中进行 |
| Graphics 返回负值 | 颜色值超出 [0, 255] 范围 | 始终钳制 r/g/b/a 值 |
| 忽略仿真上下限 | 温度/压力可能溢出 | 使用 `MAX_TEMP`, `MIN_TEMP`, `MAX_PRESSURE`, `MIN_PRESSURE` |
| 死循环粒子创建 | Update 中无条件创建同类型粒子 | 加计数器上限检查 |
| 忘记 re-configure | 修改 meson.build 后不重新配置 | 始终运行 `meson setup build --reconfigure` |

### 9.5 代码模板（快速起步）

以下是最小化的新元素代码模板，复制后填入你的逻辑即可：

```cpp
// TEMPLATE.cpp - 新元素模板
#include "simulation/ElementCommon.h"

// 前向声明（根据需要的回调选择）
static int  update(UPDATE_FUNC_ARGS);
static int  graphics(GRAPHICS_FUNC_ARGS);
static void create(ELEMENT_CREATE_FUNC_ARGS);

void Element::Element_TMPL()
{
    Identifier = "DEFAULT_PT_TMPL";
    Name = "TMPL";
    Colour = 0xFF8000_rgb;
    MenuVisible = 1;
    MenuSection = SC_SPECIAL;
    Enabled = 1;

    // === 物理属性（按元素类型选一套）===
    // 固体模板：
    Advection = 0.0f;     AirDrag = 0.00f * CFDS;  AirLoss = 1.00f;
    Loss = 0.00f;         Collision = 0.0f;        Gravity = 0.0f;
    Diffusion = 0.00f;    HotAir = 0.000f * CFDS;  Falldown = 0;
    // 粉末模板：
    // Advection = 0.7f;  AirDrag = 0.04f * CFDS;  AirLoss = 0.96f;
    // Loss = 0.00f;      Collision = 0.0f;        Gravity = 0.3f;
    // Diffusion = 0.00f; HotAir = 0.000f * CFDS;  Falldown = 1;
    // 液体模板：
    // Advection = 0.6f;  AirDrag = 0.01f * CFDS;  AirLoss = 0.98f;
    // Loss = 0.00f;      Collision = 0.0f;        Gravity = 0.1f;
    // Diffusion = 0.00f; HotAir = 0.000f * CFDS;  Falldown = 2;

    Flammable = 0;    Explosive = 0;    Meltable = 0;    Hardness = 1;
    Weight = 100;     HeatConduct = 0;

    Description = ByteString("元素描述").FromUtf8();

    Properties = TYPE_SOLID;  // 改为你的类型

    LowPressure = IPL;    LowPressureTransition = NT;
    HighPressure = IPH;   HighPressureTransition = NT;
    LowTemperature = ITL; LowTemperatureTransition = NT;
    HighTemperature = ITH; HighTemperatureTransition = NT;

    DefaultProperties.temp = R_TEMP + 273.15f;

    // === 绑定回调（按需取消注释）===
    Update = &update;
    Graphics = &graphics;
    Create = &create;
    // CreateAllowed = &createAllowed;
    // ChangeType = &changeType;
    // CtypeDraw = &ctypeDraw;
    // IconGenerator = &iconGen;
}

static int update(UPDATE_FUNC_ARGS)
{
    auto &sd = SimulationData::CRef();
    auto &elements = sd.elements;
    // TODO: 在此添加每帧行为逻辑
    return 0;
}

static int graphics(GRAPHICS_FUNC_ARGS)
{
    // TODO: 在此添加自定义渲染逻辑
    // 或删除此函数并将 Graphics 绑定设为 nullptr（使用默认渲染）
    *colr = cpart->dcolour ? (cpart->dcolour >> 16) & 0xFF : 0x80;
    *colg = cpart->dcolour ? (cpart->dcolour >> 8)  & 0xFF : 0x80;
    *colb = cpart->dcolour ? (cpart->dcolour)       & 0xFF : 0x80;
    *cola = 255;
    return 1;  // 静态颜色，可缓存
}

static void create(ELEMENT_CREATE_FUNC_ARGS)
{
    // TODO: 在此添加创建逻辑
    // sim->parts[i].life = sim->rng.between(100, 200);
}
```

---

## 10. 参考资源

### 10.1 优秀参考元素

学习官方元素的源码是掌握 TPT 元素开发的最佳途径。以下元素实例按复杂度排序，建议按顺序阅读：

| 元素 | 文件 | 学习重点 | 难度 |
|------|------|---------|------|
| BOMB | `bomb.cpp` | Update + 简单的 Graphics + 爆炸创建粒子 | 入门 |
| GLOW | `GLOW.cpp` | Update + 复杂的动态 Graphics + 辉光 | 入门 |
| CLNE | `CLNE.cpp` | Update + CarriesTypeIn + CtypeDraw | 中级 |
| CRAY | `CRAY.cpp` | Update + CtypeDraw + 射线逻辑 | 中级 |
| FIRE | `FIRE.cpp` | Update + 复杂的 Create + Graphics + 元素类型分支 | 高级 |
| FIGH | `FIGH.cpp` | Update + CreateAllowed + ChangeType + 外部资源管理 | 高级 |
| NONE | `NONE.cpp` | IconGenerator 菜单图标生成 | 参考 |

所有元素源码位于：`src/simulation/elements/`

### 10.2 相关附录

| 附录 | 内容 |
|------|------|
| [附录A：Type值完整表](appendix-A-Type值表.md) | 所有元素的 Type ID 对照表 |
| [附录B：元素导热速度表](appendix-B-导热速度表.md) | HeatConduct 取值参考 |
| [附录C：PHOT元素反射值表](appendix-C-PHOT反射值表.md) | PhotonReflectWavelengths 参考 |
| [附录D：Lua API 参考](appendix-D-Lua-API参考.md) | 完整的 Lua API 参考 |
| [附录H：调试模式指南](appendix-H-调试模式指南.md) | 调试工具使用方法 |
| [附录M：温度与压力阈值参考](appendix-M-温度与压力阈值参考.md) | 状态转换阈值设计参考 |
| [附录N：CanMove粒子交互规则](appendix-N-CanMove交互规则.md) | 粒子间可移动性规则 |
| [附录P：粒子标志位与状态常量参考](appendix-P-粒子标志位与状态常量参考.md) | Properties 标志位完整说明 |
| [附录Q：Subframe子帧技术详解](appendix-Q-Subframe子帧技术.md) | 高速计算技术（进阶） |

### 10.3 关键头文件速查

| 头文件 | 路径 | 用途 |
|--------|------|------|
| Element.h | `src/simulation/Element.h` | Element 类定义（所有字段声明） |
| ElementDefs.h | `src/simulation/ElementDefs.h` | 类型常量、属性标志位、回调宏 |
| ElementCommon.h | `src/simulation/ElementCommon.h` | 所有元素 .cpp 的公共包含 |
| Particle.h | `src/simulation/Particle.h` | 粒子数据结构（Particle struct） |
| TransitionConstants.h | `src/simulation/TransitionConstants.h` | IPL, IPH, ITL, ITH, NT, ST 常量 |
| MenuSection.h | `src/simulation/MenuSection.h` | 菜单分类常量 SC_XXX |
| Pixel.h | `src/graphics/Pixel.h` | RGB 类和 _rgb 字面量 |
| Simulation.h | `src/simulation/Simulation.h` | Simulation 类（create_part 等核心 API） |

### 10.4 外部资源

- **TPT 官方 Wiki：Coding Tutorial**：https://powdertoy.co.uk/Wiki/Wiki.html?title=Coding-tutorial.html （英文）
- **TPT 官方 GitHub**：https://github.com/The-Powder-Toy/The-Powder-Toy
- **TPT 中文版 GitHub**：https://github.com/Dragonrster/The-Powder-Toy-Chinese
- **Lua API 官方文档**：https://powdertoy.co.uk/Wiki/Wiki.html?title=Lua （英文）

### 10.5 开发流程检查清单

新元素开发完成后，对照以下清单逐项检查：

- [ ] 元素文件 `XXX.cpp` 位于 `src/simulation/elements/`
- [ ] 第一行 `#include "simulation/ElementCommon.h"`
- [ ] `Element::Element_XXX()` 函数存在且正确设置所有字段
- [ ] `Identifier` 格式为 `"DEFAULT_PT_XXX"`
- [ ] 物理属性参数合理且参考同类元素
- [ ] Properties 五种类型标志选一且仅选一
- [ ] 回调函数声明为 `static` 并使用正确的宏参数
- [ ] Update 中正确处理边界检查（`pmap` 访问安全）
- [ ] 返回值正确（Update: 0/1, Graphics: 0/1）
- [ ] `meson.build` 中已添加 `'XXX'` 到 `simulation_elem_names` 列表末尾
- [ ] 运行 `meson setup build --reconfigure` 重新配置
- [ ] 编译通过无警告
- [ ] 游戏内放置/创建元素正常
- [ ] 每帧行为符合预期
- [ ] 与其他元素的交互正常
- [ ] 状态转换触发正确
- [ ] 无性能问题（大量粒子时帧率稳定）
- [ ] 代码有适当中文注释（对于中文版）
- [ ] 未修改已有元素的 ID 或行为

---

> 本教程基于 The-Powder-Toy-Chinese 最新源码编写。元素数据结构和回调宏定义以 `Element.h`、`ElementDefs.h` 和 `ElementCommon.h` 为准。所有代码示例均可在正确配置的 TPT 编译环境中编译通过。
