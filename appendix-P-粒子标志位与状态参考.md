# 附录P：粒子标志位与状态常量参考

## 元素类型标志 (PROP_xxx)

在元素`Properties`字段中按位组合。一个元素可以同时拥有多个属性标志。

### 全部属性标志清单

| 标志 | 位值 | 说明 | 典型元素 |
|------|------|------|---------|
| TYPE_PART | 1 | 粉末型——受重力下落，可堆积 | DUST, SAND, SALT |
| TYPE_LIQUID | 2 | 液体型——流动，受重力 | WATR, OIL, LAVA |
| TYPE_SOLID | 4 | 固体型——静止不动 | METL, BRCK, WOOD |
| TYPE_GAS | 8 | 气体型——高扩散，无重力 | GAS, SMKE, O2 |
| TYPE_ENERGY | 16 | 能量型——无重力，特殊移动 | PHOT, NEUT, PROT |
| PROP_CONDUCTS | 32 | 导电——可被SPRK激活 | METL, PSCN, NSCN |
| PROP_PHOTPASS | 2048 | 光子穿透——PHOT可穿过 | GLAS, WATR, INSL |
| PROP_NEUTPENETRATE | 4096 | 中子穿透——NEUT穿过并触发反应 | DEUT, PLUT, URAN |
| PROP_NEUTABSORB | 8192 | 中子吸收——NEUT被吸收 | TTAN, WATR |
| PROP_NEUTPASS | 16384 | 中子通过——NEUT穿过不触发反应 | GLAS, INSL |
| PROP_DEADLY | 32768 | 致命——对火柴人造成伤害 | ACID, LAVA, PLSM |
| PROP_HOT_GLOW | 65536 | 高温发光——高温时产生辉光 | METL, LAVA, FIRE |
| PROP_LIFE | 131072 | 有生命值——life字段有意义 | LIFE, DEUT, TRON |
| PROP_RADIOACTIVE | 262144 | 放射性——对火柴人持续辐射 | URAN, PLUT, POLO |
| PROP_LIFE_DEC | 524288 | 生命衰减——每帧life递减 | DEUT(部分衰变) |
| PROP_LIFE_KILL | 1048576 | 生命耗尽自杀——life≤0时死亡 | TRON(消失) |
| PROP_LIFE_KILL_DEC | 2097152 | 生命耗尽衰减——life递减至≤0死 | FIRE(燃烧后消失) |
| PROP_SPARKSETTLE | 4194304 | 火花沉降——作为爆炸碎片 | EMBR, FIRE |
| PROP_NOAMBHEAT | 8388608 | 无环境热交换——不与环境换热 | CFLM, TESC |
| PROP_NOCTYPEDRAW | 16777216 | 不绘制Ctype——使用自定义绘制 | LAVA(显示熔岩) |

### 类型标志组合示例

| 元素 | Properties值 | 组合说明 |
|------|-------------|---------|
| WATR | TYPE_LIQUID + PROP_CONDUCTS + PROP_NEUTABSORB | 液体+导电+吸中子 |
| METL | TYPE_SOLID + PROP_CONDUCTS + PROP_HOT_GLOW + PROP_LIFE + PROP_LIFE_KILL_DEC | 固体+导电+辉光+有生命+烧尽 |
| FIRE | TYPE_GAS + PROP_HOT_GLOW + PROP_LIFE + PROP_LIFE_KILL_DEC + PROP_SPARKSETTLE | 气体+辉光+生命+烧尽+沉降 |
| PHOT | TYPE_ENERGY + PROP_PHOTPASS | 能量+透光 |
| NEUT | TYPE_ENERGY + PROP_NEUTPASS | 能量+透中子 |
| URAN | TYPE_SOLID + PROP_RADIOACTIVE + PROP_HOT_GLOW | 固体+放射+辉光 |
| ACID | TYPE_LIQUID + PROP_DEADLY | 液体+致命 |
| LAVA | TYPE_LIQUID + PROP_HOT_GLOW + PROP_DEADLY + PROP_NOCTYPEDRAW | 液体+辉光+致命+自定义绘制 |
| INSL | TYPE_SOLID + PROP_NEUTPASS + PROP_PHOTPASS | 固体+透中子+透光 |
| VOID | TYPE_SOLID | 固体(特殊行为) |

---

## 粒子标志位 (FLAG_xxx)

在粒子的`flags`字段中按位存储。这些标志控制粒子在每帧模拟中的行为。

| 标志 | 位值 | 说明 | 设置时机 | 影响 |
|------|------|------|---------|------|
| FLAG_STAGNANT | 1 | 停滞——粒子当前帧未移动 | 每帧更新后 | 下帧可能跳过移动 |
| FLAG_SKIPMOVE | 2 | 跳过移动——本帧不参与移动计算 | 部分更新函数 | 粒子本帧原地不动 |
| FLAG_MOVABLE | 4 | 可移动标记 | 内部使用 | 标记粒子可以参与移动 |
| FLAG_PHOTDECO | 8 | 光子装饰——光子带有装饰颜色 | PHOT装饰模式 | 渲染使用装饰颜色而非正常波长 |

### 标志位生命周期

```
[帧开始] → 清除FLAG_STAGNANT → 移动计算 → 
    ├── 成功移动 → FLAG_STAGNANT=0
    └── 移动失败 → FLAG_STAGNANT=1
→ 更新计算 → 渲染 → [帧结束]
```

### 标志位使用代码示例

```lua
-- 检查标志位
local flags = sim.partProperty(partId, 8)
local isStagnant = bit.band(flags, 1) ~= 0   -- FLAG_STAGNANT
local skipMove = bit.band(flags, 2) ~= 0     -- FLAG_SKIPMOVE

-- 设置标志
flags = bit.bor(flags, 2)  -- 设置FLAG_SKIPMOVE
sim.partProperty(partId, 8, flags)

-- 清除标志
flags = bit.band(flags, bit.bnot(2))  -- 清除FLAG_SKIPMOVE
sim.partProperty(partId, 8, flags)
```

---

## PMAP编码

`pmap[y][x]`存储的是编码值：高16位=Type，低16位=Index。

### 编码结构
```
│<───── 高16位 ─────>│<───── 低16位 ─────>│
│       Type          │       Index         │
│   (元素类型ID)       │   (粒子索引)         │
```

### 提取宏

| 宏 | 功能 | 等价Lua |
|---|---|------|
| TYP(pmap[y][x]) | 提取类型 | `bit.rshift(val, 16)` 或 `math.floor(val / 65536)` |
| ID(pmap[y][x]) | 提取索引 | `bit.band(val, 0xFFFF)` |
| PMAPID(v) | 编码到PMAP | (不需要在Lua中使用) |

### Lua中使用
```lua
local val = sim.pmap(100, 100)
if val then
    local typeId = bit.rshift(val, 16)
    local index = bit.band(val, 0xFFFF)
    tpt.log(string.format("Type=%d Index=%d", typeId, index))
end
```

---

## 模拟常量速查（完整版）

### 画布常量

| 常量 | 值 | 说明 |
|------|-----|------|
| XRES | 612 | 画布X方向像素分辨率 |
| YRES | 384 | 画布Y方向像素分辨率 |
| CELL | 4 | 每CELL的像素数(每个格子4x4像素) |
| XCELLS | 153 | X方向CELL数(XRES/CELL) |
| YCELLS | 96 | Y方向CELL数(YRES/CELL) |
| NPART | — | 最大粒子数(65536或更多) |
| MAX_PARTS | NPART | 最大粒子数别名 |

### 温度常量

| 常量 | 值(K) | 说明 |
|------|-------|------|
| R_TEMP | 295.15 | 室温(22℃) |
| MAX_TEMP | 9999 | 最高温度 |
| MIN_TEMP | 0 | 最低温度(绝对零度) |
| R_TEMP_C | 22 | 室温(摄氏度) |
| R_TEMP_F | 71.6 | 室温(华氏度) |

### 压力常量

| 常量 | 值 | 说明 |
|------|-----|------|
| MAX_PRESSURE | 256 | 最高压力 |
| MIN_PRESSURE | -256 | 最低压力 |

### 转换标记

| 常量 | 值 | 说明 |
|------|-----|------|
| NT | -1 | 无转换(No Transition) |
| ST | PT_NUM | 特殊转换(Special Transition)，用于LAVA的Ctype |
| ITH | — | 极高温度标记(IntegerMax-1) |
| ITL | — | 极低温度标记(IntegerMin) |
| IPH | — | 极高压力标记(IntegerMax-1) |
| IPL | — | 极低压力标记(IntegerMin) |

### 转换标记用途
- **NT(-1)**：表示该状态转换条件不存在/永不触发
- **ST(PT_NUM)**：特殊类型转换——Ctype变为元素总数，用于LAVA的"ROCK"状态
- **ITH**：温度设置为整数最大值-1，表示"温度极高"
- **ITL**：温度设置为整数最小值，表示"温度极低"
- 这些标记是C++内部的预设值，Lua中不需要直接使用

---

## 边界模式

| 常量 | 值 | 说明 | 视觉效果 |
|------|-----|------|---------|
| EDGE_VOID | 0 | 虚空边界——粒子到达边界消失 | 画布外全黑 |
| EDGE_SOLID | 1 | 固体边界——粒子被阻挡 | 粒子堆积在边界 |
| EDGE_LOOP | 2 | 循环边界——粒子从对面出现 | 穿过右边界从左边出来 |

### 边界模式效果对比

```
EDGE_VOID:   粒子→|虚空(消失)
EDGE_SOLID:  粒子→|████(被挡住)
EDGE_LOOP:   粒子→|→→→→→→→→(从对面出来)
```

---

## 重力模式

| 常量 | 值 | 说明 | 粒子行为 |
|------|-----|------|---------|
| GRAV_VERTICAL | 0 | 垂直重力(默认向下) | 粒子向下落 |
| GRAV_OFF | 1 | 关闭重力 | 粒子不下落 |
| GRAV_RADIAL | 2 | 径向重力(朝向中心) | 粒子向画布中心汇聚 |
| GRAV_CUSTOM | 3 | 自定义重力方向 | 粒子向自定义矢量方向落 |

### 重力模式选择指南

- **GRAV_VERTICAL**: 标准模式，适合大多数场景
- **GRAV_OFF**: 太空/零重力场景，或测试粒子自身运动
- **GRAV_RADIAL**: 模拟行星中心引力，粒子围绕中心
- **GRAV_CUSTOM**: 通过`sim.customGravity(x,y)`设置任意方向

---

## 空气模式

| 常量 | 值 | 说明 | 使用场景 |
|------|-----|------|---------|
| AIR_ON | 0 | 完整空气模拟(压力+风速) | 默认模式 |
| AIR_PRESSUREOFF | 1 | 关闭压力(仅风速) | 仅需空气流动 |
| AIR_VELOCITYOFF | 2 | 关闭风速(仅压力) | 仅需压力模拟 |
| AIR_OFF | 3 | 关闭空气 | 性能优化/纯净模拟 |
| AIR_NOUPDATE | 4 | 不更新空气(保留当前值) | 冻结空气状态 |

---

## 渲染像素模式

| 常量 | 值 | 视觉效果 | 适用元素 | 性能影响 |
|------|-----|---------|---------|---------|
| PMODE_NONE | 0 | 不绘制 | 隐形元素 | 无 |
| PMODE_FLAT | 1 | 平面正方形(像素格) | 固体、粉末 | 最低 |
| PMODE_BLOB | 2 | 液体圆形 | 液体(WATR, OIL) | 低 |
| PMODE_BLUR | 3 | 模糊效果 | 气体(SMKE) | 中 |
| PMODE_GLOW | 4 | 发光效果 | 能量/高温 | 中 |
| PMODE_SPARK | 5 | 火花闪烁 | 火花(FIRE, EMBR) | 中 |
| PMODE_FLARE | 6 | 强光耀斑(大范围) | 光(PHOT) | 高 |
| PMODE_LFLARE | 7 | 局部耀斑(粒子大小) | 局部光源 | 中 |
| PMODE_ADD | 8 | 叠加混合(照亮) | 火焰(FIRE) | 低 |
| PMODE_BLEND | 9 | 透明度混合 | 玻璃(GLAS) | 低 |
| PSPEC_STICKMAN | — | 火柴人专用渲染 | STKM, STK2 | 特殊 |

---

## 火焰效果

| 常量 | 效果 | 用途 |
|------|------|------|
| FIRE_ADD | 叠加模式火焰 | 照亮背景的火焰 |
| FIRE_BLEND | 混合模式火焰 | 半透明火焰效果 |
| FIRE_SPARK | 火花模式 | 爆炸碎片的火花效果 |

---

## 装饰工具模式

| 常量 | 效果 | 说明 |
|------|------|------|
| DECO_DRAW | 绘制 | 添加装饰色 |
| DECO_CLEAR | 清除 | 移除装饰色 |
| DECO_ADD | 叠加 | 叠加装饰色 |
| DECO_SUBTRACT | 减色 | 减少装饰色分量 |
| DECO_MULTIPLY | 乘色 | 乘算装饰色 |
| DECO_DIVIDE | 除色 | 除算装饰色 |
| DECO_SMUDGE | 模糊 | 模糊装饰色 |

---

## 菜单分类

| 常量 | 值 | 中文名 | 包含元素示例 |
|------|-----|--------|-------------|
| SC_WALL | 0 | 墙壁 | 各种墙壁类型 |
| SC_ELEC | 1 | 电子元件 | METL, PSCN, NSCN, INST |
| SC_POWERED | 2 | 电动元件 | SWCH, PTCT, NTCT |
| SC_SENSOR | 3 | 传感器 | DLAY, TSNS, PSNS |
| SC_FORCE | 4 | 力场 | ARAY, WIFI, PROT, GRVT |
| SC_EXPLOSIVE | 5 | 爆炸物 | FIRE, GUN, NITR, TNT, DEST |
| SC_GAS | 6 | 气体 | GAS, SMKE, O2, CO2 |
| SC_LIQUID | 7 | 液体 | WATR, OIL, LAVA, ACID |
| SC_POWDERS | 8 | 粉末 | DUST, SAND, SALT, SNOW |
| SC_SOLIDS | 9 | 固体 | WOOD, METL, GLAS, BRCK |
| SC_NUCLEAR | 10 | 核材料 | URAN, PLUT, DEUT, ISOZ |
| SC_SPECIAL | 11 | 特殊 | VOID, BHOL, NBHL |
| SC_LIFE | 12 | 生命 | LIFE, GOL |
| SC_TOOL | 13 | 工具 | HEAT, COOL, VAC, AIR |
| SC_DECO | 15 | 装饰 | 各种装饰元素 |

---

## 属性交互矩阵

### 属性相互影响的连锁反应

```
属性A                 影响属性B            影响属性C
────────────────────────────────────────────────
PROP_HOT_GLOW     →   渲染模式           → 辉光半径
PROP_LIFE_KILL    →   粒子生命期          → 粒子死亡时间
PROP_LIFE_DEC     →   life值递减速率      → 粒子寿命
PROP_LIFE_KILL_DEC →  life值≤0→死亡      → 粒子自动清除
PROP_RADIOACTIVE  →   对STKM造成伤害      → 周围粒子变异
PROP_NOAMBHEAT    →   不与环境换热        → 温度保持独立
PROP_PHOTPASS     →   光子可穿透          → 不影响光路
PROP_NEUTPENETRATE →  NEUT穿透+反应      → 链式反应
PROP_NEUTABSORB   →   NEUT被吸收消失      → 反应终止
PROP_NEUTPASS     →   NEUT无阻碍通过      → 不触发反应
PROP_CONDUCTS     →   SPRK可传播          → 形成电路
PROP_DEADLY       →   对STKM/STK2伤害    → 生命值减少
PROP_SPARKSETTLE  →   碰固体消失          → 火花短寿命
PROP_NOCTYPEDRAW  →   忽略Ctype颜色       → 自定义渲染
```

### 常见标志组合模式

```
不可破坏固体：
  TYPE_SOLID | Hardness=1000 | Weight=100
  代表：DMND(钻石)

可燃固体：
  TYPE_SOLID | Flammable>0 | Meltable
  代表：WOOD(木材)

导电液体：
  TYPE_LIQUID | PROP_CONDUCTS
  代表：WATR(水), SLTW(盐水)

惰性气体：
  TYPE_GAS | PROP_NEUTPASS | PROP_PHOTPASS
  代表：INLS但非气体(INLS是固体)

放射性固体：
  TYPE_SOLID | PROP_RADIOACTIVE | PROP_HOT_GLOW
  代表：URAN(铀), PLUT(钚)

自动衰减元素：
  PROP_LIFE | PROP_LIFE_DEC | PROP_LIFE_KILL
  代表：DEUT(重水部分衰变)
```

---

## 交叉参考

- **附录N**：CanMove交互规则（PROP_PHOTPASS, PROP_NEUTPASS等直接影响can_move）
- **附录M**：温度压力阈值（PROP_HOT_GLOW触发条件, PROP_NOAMBHEAT影响）
- **附录D**：Lua API中操作properties和flags的函数
- **附录H**：DEBUG_PARTICLE视图显示flags字段
- **主文档**：元素详情中的Properties值和含义
