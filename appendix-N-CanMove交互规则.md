# 附录N：CanMove 粒子交互规则

TPT用`can_move[moving][destination]`矩阵控制两个粒子相遇时的行为。

---

## 基本模式

| 模式 | 值 | 名称 | 行为 |
|------|-----|------|------|
| CANMOVE_BOUNCE | 0 | 弹开 | 移动粒子不可进入目标格，被反弹 |
| CANMOVE_SWAP | 1 | 交换 | 两个粒子交换位置 |
| CANMOVE_ENTER | 2 | 穿透 | 两粒子占据同一空间（共存） |
| CANMOVE_BUILTIN | 3 | 特殊规则 | 调用元素特定的CanMove逻辑 |

---

## 通用规则

### 重量系统
- **轻粒子不能排开重粒子**：`moving.Weight ≤ destination.Weight → can_move=0`
- **例外**：GEL(凝胶)可被任何重量粒子排开
- **重量值参考**：

| 重量 | 代表元素 |
|------|---------|
| 1 | GEL(凝胶,最轻固体) |
| 10 | 轻粉末(SUGR/SALT/FSEP等) |
| 20 | 标准粉末(DUST/STNE/SAND等) |
| 30 | 较重粉末(CNCT/IRON/BMTL等) |
| 50 | 重固体(BRCK/METL/GLAS等) |
| 70 | 极重(TTAN等) |
| 90 | 超重(GOLD/PLUT/TUNG等) |
| 100 | DMND(钻石,最重固体) |

### 类型交互规则
- **同种粒子**：默认不可互排(can_move=0)
- **能量粒子间**：默认互相穿透(can_move=2)
- **液体/气体间**：默认可交换(can_move=1)

---

## 特殊粒子完整规则

### PHOT(光子)

| 目标属性 | 行为 | 模式 |
|---------|------|------|
| PROP_PHOTPASS | 穿透 | 2 |
| 无PROP_PHOTPASS | 交换位置 | 1 |
| INVS | 特殊判断 | 3(内置) |
| FILT | 穿过并修改波长 | 2 |
| GLAS | 穿过(透光) | 2 |
| WATR | 穿过(透明) | 2 |

光子遇到FILT时使用内置规则（波长修改），遇到INVS时根据压力判断。

### NEUT(中子)

| 目标属性 | 行为 | 模式 | 典型元素 |
|---------|------|------|---------|
| PROP_NEUTPASS | 穿透(无反应) | 2 | GLAS, INSL |
| PROP_NEUTABSORB | 交换(吸收) | 1 | TTAN, WATR |
| PROP_NEUTPENETRATE | 交换(穿透+反应) | 1 | DEUT, PLUT, URAN |
| INVS | 穿透 | 2 | INVS |
| 无特殊属性 | 交换(一般反应) | 1 | 大多数 |

**NEUT交互流程**：
```
NEUT ──遇到粒子──▶
    ├── PROP_NEUTPASS     → 穿过（不触发任何反应）
    ├── PROP_NEUTABSORB   → 被吸收（NEUT消失,目标可能转变）
    ├── PROP_NEUTPENETRATE → 穿过并触发反应（目标转变,NEUT继续）
    └── 无特殊属性        → 根据目标类型有不同反应
```

### STKM/STK2/FIGH(火柴人/战斗单位)

| 目标类型 | 行为 | 模式 |
|---------|------|------|
| 液体(TYPE_LIQUID) | 穿透 | 2 |
| 气体(TYPE_GAS) | 穿透 | 2 |
| 能量(TYPE_ENERGY) | 穿透 | 2 |
| PRTO, SPAWN | 穿透 | 2 |
| 固体/粉末 | 不可移动 | 0 |
| 其他火柴人 | 战斗判定 | 3(内置) |

### SPRK(电脉冲)
**始终不可移动(can_move=0)**——电脉冲必须固定在导电元素上，无法独立移动。

### VOID/PVOD(虚空/压力虚空)

```
can_move = 3(内置规则)

VOID行为 = f(Ctype, Tmp, 目标粒子类型)
├── Ctype=0(白名单模式): 仅吸收Tmp列表中的元素
│   Tmp编码：位图，每位对应一种元素
├── Ctype!=0(黑名单模式): 吸收除Ctype外的所有元素
│   吸收顺序：先吸收tmp1低字节类型，再吸收高字节类型
└── 压力虚空(PVOD): 还需检查压力阈值
```

**VOID白名单/黑名单编码**：
- `Tmp` 的每个位对应一种元素类型ID
- 例：`tmp=0b00000111` → 仅吸收类型0,1,2(=吸收WATR/OIL/LAVA等)
- `Ctype` 存储例外的目标类型（不被吸收的）

### INVS(隐形)

```
can_move = 3(内置规则)

根据压力判断是否阻挡：
├── 低压力(<阈值): INVS"固体化"→阻挡粒子进入
├── 高压力(>阈值): INVS"透明化"→允许粒子穿透
└── PHOT(光子): 特殊处理,始终可穿透(2)
```

### 黑洞(BHOL/NBHL)

| 目标类型 | 行为 | 模式 |
|---------|------|------|
| 所有粒子 | 交换位置 | 1 |

黑洞的can_move始终设置为1(交换)，因为吸入效果是通过其特殊更新函数实现的——不是通过can_move，而是在update函数中将接触的粒子传送到黑洞内部。

### EMBR(火花)

```
所有粒子不能穿过EMBR: can_move = 0
例外：能量粒子(TYPE_ENERGY)可穿透(2)
```

EMBR作为不可穿透屏障，常用于保护/隔离。

### DEST(高爆炸药)

| 目标类型 | 行为 | 模式 | 原因 |
|---------|------|------|------|
| DMND | 不可移动 | 0 | 钻石不可破坏 |
| CLNE/PCLN/BCLN | 不可移动 | 0 | 克隆器不可破坏 |
| PBCN | 不可移动 | 0 | 可破坏克隆器 |
| ROCK | 不可移动 | 0 | 岩石直接转化 |
| 其他 | 交换 | 1 | 正常爆炸 |

### PROT/GRVT(质子/万有引力)

```
穿透除以下外的所有粒子(can_move=2)：
├── DMND (钻石 → 0, 不可穿透)
├── INSL (绝缘体 → 0)
├── VOID/PVOD (虚空 → 0)
├── VIBR/BVBR (振动器 → 0)
├── PRTI/PRTO (传送门 → 0)
└── 其他所有元素 → 2(穿透)
```

### SAWD(锯末)

```
其他粉末不能排开SAWD: can_move=0
→ SAWD"浮"在粉末层之上
```

### CNCT(混凝土)

```
其他粒子不能排开CNCT: can_move=0
→ 混凝土不可被推动（类似墙壁）
```

---

## 完整类型交互矩阵

### 固体vs各类型

| 移动\目标 | 固体 | 液体 | 气体 | 粉末 | 能量 |
|----------|------|------|------|------|------|
| 固体 | 0(不可互排) | 1(交换) | 1(交换) | 1(交换) | 2(穿透) |
| 液体 | 0(不可) | 1(交换) | 1(交换) | 1(交换) | 2(穿透) |
| 气体 | 0(不可) | 1(交换) | 1(交换) | 1(交换) | 2(穿透) |
| 粉末 | 0(不可) | 1(交换) | 1(交换) | 0(同种)/1(异种) | 2(穿透) |
| 能量 | 1(交换) | 1(交换) | 1(交换) | 1(交换) | 2(穿透) |

### 特殊元素交互矩阵

| 移动\目标 | PHOT | NEUT | SPRK | STKM | VOID | BHOL | INVS |
|----------|------|------|------|------|------|------|------|
| 普通粒子 | 1* | 1* | 0 | 0/2* | 3* | 1 | 3* |
| PHOT | 2 | 1 | 0 | 2 | 3 | 1 | 2 |
| NEUT | 1 | 2 | 0 | 2 | 3 | 1 | 2 |
| SPRK | 0 | 0 | 0 | 0 | 3 | 1 | 3 |
| STKM | 2 | 2 | 0 | 3(战斗) | 3 | 1 | 3 |
| 能量粒子 | 2 | 2 | 0 | 2 | 3 | 1 | 2 |

*表示有例外情况，详见上方各元素规则。

---

## 边界情况与陷阱

### 1. 同种粒子堆积
**问题**：相同类型粒子默认can_move=0，导致堆积时顶部粒子不下落。
**解法**：
- 将下方同种粒子临时转为其他类型
- 使用GEL作为中间层（GEL可被任何粒子排开）
- 通过Lua手动移动

### 2. 重量边界条件
**问题**：Weight==50的粒子能否排开Weight==50的粒子？
**答案**：不能。条件是`moving.Weight ≤ destination.Weight → 0`，用<=比较。

### 3. 液体与粉末交互
**问题**：水(WATR,Weight=20)落在沙(SAND,Weight=20)上
**结果**：先到达的粒子占据格子，后来者被弹开。由于重量相同，顺序取决于更新顺序。

### 4. 气压通过粒子
**问题**：气体能否"穿过"固体？
**答案**：不能。气体被固体阻挡(can_move=0)，但气压场独立于粒子，可通过空气传播。

### 5. 穿透粒子的共存
当can_move=2时，两个粒子占据同一空间。后果：
- 两者都能独立move和update
- 渲染取决于元素绘制模式
- 两个粒子可以有不同的温度而互不影响

### 6. SPRK导电信道
SPRK can_move=0但不阻挡电信号传播——电信号沿导电粒子链传播，不受can_move影响。

### 7. VOID兼容性表
VOID的Tmp白名单编码复杂。如果设置错误：
- Ctype=0且Tmp=0→吸收所有粒子（危险的"黑洞"）
- Ctype=0且Tmp!=(你的目标元素位)→不吸收任何粒子（VOID无效）

---

## Lua操作CanMove

```lua
-- 读取can_move关系
local result = sim.canMove("WATR", "OIL")
tpt.log("水移动进入油的can_move: " .. result)

-- 设置自定义can_move
sim.canMove("WATR", "LAVA", 2)  -- WATR现在可以穿透LAVA

-- 重置为默认
sim.canMove("WATR", "LAVA", nil) -- nil表示恢复默认

-- 批量设置
local energyTypes = {"PHOT", "NEUT", "PROT", "GRVT"}
for _, e in ipairs(energyTypes) do
    for _, t in ipairs({"WATR", "DUST", "FIRE"}) do
        sim.canMove(e, t, 2)  -- 这些能量粒子穿透水和粉尘
    end
end

-- 创建完全阻塞
for _, blocker in ipairs({"DMND", "WALL", "BRCK"}) do
    for _, m in ipairs({"WATR", "DUST", "FIRE"}) do
        sim.canMove(m, blocker, 0)  -- 不可进入
    end
end
```

---

## CanMove调试技巧

```lua
-- 打印两个类型的can_move关系
function debugCanMove(typeA, typeB)
    local idA = elements.getByName(typeA)
    local idB = elements.getByName(typeB)
    if not idA or not idB then
        tpt.log("元素不存在")
        return
    end
    local result = sim.canMove(idA, idB)
    local modes = {"弹开", "交换", "穿透", "特殊"}
    tpt.log(string.format("%s → %s: %s(%d)",
        typeA, typeB, modes[result+1] or "未知", result))
end

-- 打印元素完整CanMove矩阵
function printCanMoveMatrix(elementName)
    local id = elements.getByName(elementName)
    if not id then return end
    tpt.log("=== " .. elementName .. " 的CanMove矩阵 ===")
    local types = {"WATR", "DUST", "FIRE", "METL", "PHOT", "NEUT", "SPRK"}
    for _, t in ipairs(types) do
        debugCanMove(elementName, t)
    end
end
```

---

## 交叉参考

- **附录P**：PROP_PHOTPASS, PROP_NEUTPASS, PROP_NEUTABSORB, PROP_NEUTPENETRATE标志位
- **附录D**：`sim.canMove()` API详情
- **主文档-元素章节**：各元素的移动和更新行为
- **附录M**：温度和压力对粒子状态的影响（间接影响can_move，如温度改变粒子类型→can_move改变）
