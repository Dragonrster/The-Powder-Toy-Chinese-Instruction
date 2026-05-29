# 附录Q：Subframe 子帧技术详解

Subframe（子帧技术）是 TPT 中最强大的构建技术。它利用粒子执行顺序（Particle Order）在每一帧内编排多步逻辑操作，突破 60Hz 单帧限制，使 TPT 能够构建高速计算机、FILT 内存、图形显示器等超大规模数字系统。

本文档整合了 mark2222 的 The Subframe Lessons 系列、Schmolendevice 的 A3SFTT 原始理论、以及社区多年的实践经验。

---

## 目录

1. [SPRK 深度解析](#1-sprk-深度解析)
2. [粒子执行顺序](#2-粒子执行顺序)
3. [三步激活周期](#3-三步激活周期)
4. [CONV 导体重置详解](#4-conv-导体重置详解)
5. [DRAY 粒子复制系统](#5-dray-粒子复制系统)
6. [BRAY 光束逻辑](#6-bray-光束逻辑)
7. [ARAY 角度射线](#7-aray-角度射线)
8. [CRAY 物质射线](#8-cray-物质射线)
9. [堆叠技术完整指南](#9-堆叠技术完整指南)
10. [第一个 Subframe 电路：从零开始](#10-第一个-subframe-电路从零开始)
11. [逻辑门完整实现](#11-逻辑门完整实现)
12. [时序与时钟系统](#12-时序与时钟系统)
13. [FILT RAM 内存系统](#13-filt-ram-内存系统)
14. [数据总线与多比特传输](#14-数据总线与多比特传输)
15. [高级电路模式](#15-高级电路模式)
16. [Subframe Mod 完整参考](#16-subframe-mod-完整参考)
17. [调试与排错](#17-调试与排错)
18. [性能优化](#18-性能优化)
19. [学习路线与社区资源](#19-学习路线与社区资源)

---

## 1. SPRK 深度解析

SPRK（电脉冲）是所有子帧电路的基础。理解它的精确生命周期是入门第一步。

### 1.1 SPRK 生命周期

SPRK 不是一个"独立"粒子——它在创建时替换原导体粒子，life 递减耗尽后还原为 ctype 存储的导体类型。

```
时间线（帧）：
  帧 0：BTRY/CONV 创建 SPRK（替换导体）
         SPRK.life = 4（大多数导体上）
         SPRK.ctype = 原导体类型（如 METL）
  
  帧 1：SPRK.life = 3
         → 🔑 ARAY/DRAY/CRAY/LDTC 等检测到 life==3
         → 这是"激活窗口"！元件在此帧执行操作
  
  帧 2：SPRK.life = 2
         → 传导到相邻导体（产生新 SPRK）
  
  帧 3：SPRK.life = 1
         → 继续传导
  
  帧 4：SPRK.life = 0
         → 粒子还原为 ctype 存储的导体类型
         → 周期结束
```

### 1.2 不同导体上的 life 值

| 导体 | 初始 life | 激活帧 | 总周期 | 特殊说明 |
|------|----------|--------|--------|---------|
| METL | 4 | 帧1(life=3) | 4帧 | 自加热+10℃/传导 |
| PSCN | 4 | 帧1(life=3) | 4帧 | 向任何导体传导 |
| NSCN | 4 | 帧1(life=3) | 4帧 | 不向PSCN传导 |
| WATR | 6 | 帧3(life=3) | 6帧 | 水中传导慢 |
| SLTW | 5 | 帧2(life=3) | 5帧 | 比纯水快 |
| SWCH(ON) | 14 | 帧11(life=3) | 14帧 | 开关通电时间长 |
| RSST | 5 | 帧2(life=3) | 5帧 | 耗尽后自身销毁 |
| INST | 4 | 帧1(life=3) | 4帧 | 整条线路一帧传导 |
| INWR | 4 | 帧1(life=3) | 4帧 | 只连PSCN/NSCN |
| TUNG | 4 | 帧1(life=3) | 4帧 | 高温耐受3422℃ |
| GOLD | 4 | 帧1(life=3) | 4帧 | 十字4格远程导电 |

### 1.3 传导规则

```
SPRK 传导检测顺序（每帧）：
  1. 扫描周围 8 个邻域
  2. 对每个邻域粒子：
     a. 检查是否为导体（PROP_CONDUCTS）
     b. 检查该导体是否可接收 SPRK（sender/receiver 规则）
     c. 两粒子中点是否有 INSL/RSSS 绝缘体 → 有则阻断
     d. 检查对方是否已在冷却期（life != 0） → 是则跳过
  3. 传导成功 → 对方变为 SPRK(life=4)
```

### 1.4 SPRK 冷却期问题

**这是 Subframe 存在的根本原因**：

```
普通导体周期：
  创建SPRK(life=4) → 3帧激活 → 1帧恢复 → 共4帧冷却
  
  60 FPS 下 = 每秒最多 15 次激活
  对于计算来说太慢了！
```

**Subframe 的解决方案**：
```
CONV 重置周期：
  创建SPRK(life=4) → 下一帧 CONV 重置 → 同时 BTRY 重新通电 → 循环
  
  60 FPS 下 = 每秒 60 次激活（4倍提升！）
```

---

## 2. 粒子执行顺序

### 2.1 ID 分配算法

TPT 在**首次加载存档**时扫描画布并分配粒子 ID：

```
算法（伪代码）：
  for y = 0 to YRES-1:        // 从上到下逐行
    for x = 0 to XRES-1:      // 从左到右逐列
      for each particle at (x,y):  // 该像素的堆叠粒子
        assign next_id++           // 按堆叠顺序分配
```

**实际示例**：

```
画布上有以下粒子布局：
  位置(10,5)：METL(先放的) → ID=15
  位置(10,5)：CONV(后放的) → ID=16  // 同像素，堆叠上层
  位置(10,6)：BTRY           → ID=17  // 下一行
  位置(5,7) ：ARAY           → ID=14  // 左边列，扫得早！

更新顺序：ARAY(14) → METL(15) → CONV(16) → BTRY(17)
```

### 2.2 顺序约束规则表

| 元件对 | 必须的顺序 | 原因 |
|--------|----------|------|
| ARAY → CONV | ARAY 在 CONV 之前 | ARAY 需要检测旧 SPRK，CONV 重置会销毁它 |
| CONV → BTRY | CONV 在 BTRY 之前 | CONV 先清空导体，BTRY 才能产生新 SPRK |
| DRAY → 目标 | DRAY 在目标之前 | DRAY 复制时源必须还是旧状态 |
| DTEC → 逻辑 | DTEC 在逻辑处理之前 | 先检测再处理 |

### 2.3 存档后顺序丢失

**这是最常见的 Subframe 故障**：

- 放置粒子时，顺序取决于放置先后（后放=大ID=后更新）
- 保存存档时，只保存粒子数据不保存放置顺序
- **重新加载时**，按 2.1 的扫描算法重新分配 ID

**修复方法**：按扫描顺序（左→右，上→下）重新排列粒子物理位置，确保关键粒子在扫描时先被读到。

---

## 3. 三步激活周期

### 3.1 完整周期时序图

```
帧 N 开始
│
├─ [粒子更新开始，按 ID 顺序]
│
├─ ID=100 ARAY：检测相邻 SPRK.life == 3？
│   ├─ 是 → 沿 SPRK 对角方向发射 BRAY
│   └─ 否 → 无操作
│
├─ ID=101 METL(old)：SPRK.life == 3，有源元件已处理
│
├─ ID=102 CONV：检测相邻 METL(old) 是否通电？
│   ├─ 是 → 用 create_part(ID=101, METL) 替换
│   │       新 METL.life=0, 保留 ID=101
│   └─ 否 → 无操作
│
├─ ID=103 BTRY：检测相邻 METL(new).life == 0？
│   ├─ 是 → 产生 SPRK(life=4) 覆盖 METL(new)
│   └─ 否 → 无操作
│
├─ [其他粒子继续更新...]
│
└─ 帧 N 结束 → 帧 N+1 开始（ARAY 再次检测到 life==3）
```

### 3.2 时序关键窗口

```
           帧 N                   帧 N+1
    ┌──────┼──────┐        ┌──────┼──────┐
    │  更新阶段   │        │  更新阶段   │
    └──────┼──────┘        └──────┼──────┘
           ↓                       ↓
    ARAY检测(SPRK.life=3)    ARAY检测(新SPRK.life=3)
    CONV重置(→life=0)       CONV重置
    BTRY通电(→life=4)       BTRY通电
    
    每帧一次完整循环 = 60Hz 运算速度
```

### 3.3 多级级联

单帧内可以有多级 Subframe 级联：

```
帧内的多个"虚拟时钟周期"：
  
  ID=100: ARAY_1 检测 SPRK_1.life=3 → 发射 BRAY_1
  ID=101: CONV_1 重置 SPRK_1
  ID=102: BTRY_1 重新通电 SPRK_1
  ID=103: FILT_1 被 BRAY_1 击中 → ctype 改变（存储数据）
  ID=200: ARAY_2 检测 SPRK_2.life=3 → 发射 BRAY_2
  ID=201: CONV_2 重置 SPRK_2
  ID=202: BTRY_2 重新通电 SPRK_2
  ID=203: DTEC   检测 FILT_1 的新 ctype → 输出 SPRK_3
  
  一帧内完成了两轮完整操作 + 一次检测！
```

**关键**：级联数 = 画布宽度可容纳的独立 Subframe 单元数。理论上，1920 像素宽可容纳数百级。

---

## 4. CONV 导体重置详解

### 4.1 CONV 更新函数执行流程

```
CONV.Update() 每帧执行步骤：
  1. 读取自身 Ctype → 目标元素类型
  2. 如果 Ctype == 0（未设置）：
     a. 扫描 3×3 邻域
     b. 学习第一个有效粒子类型 → 写入 Ctype
  3. 如果 Ctype > 0（已设置）：
     a. 扫描 3×3 邻域
     b. 对每个邻域粒子：
        - 排除 DMND（不可转换）
        - 排除自身类型（已是目标类型）
        - 排除特殊元素（CLNE/PCLN/BCLN/STKM/CONV）
        - 检查 Tmp 过滤（如果设置了类型过滤）
        - 检查 Tmp2 反转（如果为1则排除Tmp类型）
     c. 对符合条件的粒子：
        - create_part(ID=原粒子ID, type=Ctype)
        - 新粒子在原位置创建
        - 保留原 ID（关键！）
        - 新粒子默认属性（life=0, tmp=0, tmp2=0）
```

### 4.2 导体重置的精确时序

```
CONV 重置单个导体的每帧过程：

帧 N，按 ID 顺序：
  1. METL(old) 被 BTRY 通电 → SPRK(life=4)
  2. SPRK(life=4) 存在，但 ARAY 不在这一帧检测它
     
帧 N+1：
  1. ARAY 检测到 SPRK.life == 3 → 发射 BRAY ✓
  2. CONV 检测到 SPRK 存在 → 替换为新 METL(life=0)
  3. BTRY 检测到新 METL(life=0) → 产生 SPRK(life=4)
  
帧 N+2：
  1. ARAY 检测到新 SPRK.life == 3 → 再次发射 BRAY ✓
  2. 循环继续...

结论：CONV + BTRY 实现了每帧一次激活（60Hz），而非普通 SPRK 的每 4 帧一次（15Hz）
```

### 4.3 多导体并行重置

```
单个 CONV 可以重置多个导体：

  CONV(ctype=METL) 放在中心位置
  周围 8 个方向各有 METL 导体
  → CONV 每帧扫描全部 8 个方向
  → 每个方向的 METL 只要通电就会被重置
  → 8 个导体独立工作，共享 CONV

限制：
  - 8 个导体不能同时通电（否则 CONV 只重置第一个扫描到的）
  - 需确保各导体通电时间错开（通过粒子顺序）
```

### 4.4 CONV 常见配置

```
配置 1：单导体重置（最简单）
  [METL] [CONV(ctype=METL)] [BTRY]
  粒子顺序：METL(far left) → CONV → BTRY(far right)

配置 2：双导体交替
  [METL_A] [CONV] [BTRY_A]
  [METL_B] [CONV] [BTRY_B]
  两个导体不同行，独立工作

配置 3：链式重置
  METL_1 → CONV_1 → BTRY_1
     ↓ 信号传到 ↓
  METL_2 → CONV_2 → BTRY_2
  每级独立时序，信号接力传输
```

---

## 5. DRAY 粒子复制系统

### 5.1 DRAY 工作原理

DRAY（D-Type Ray Emitter）可以复制任意粒子——它是 Subframe 的"通用数据总线"。

```
DRAY 激活流程：
  1. 检测 SPRK（life==3）在邻域
  2. 确定发射方向（SPRK 的对角方向）
  3. 从自身位置沿方向扫描
  4. 对扫描路径上的每个空位：
     - 创建 DRAY.ctype 类型的粒子
     - 复制 DRAY 存储的属性（tmp/tmp2/life/temp/dcolour）
  5. 直到碰到障碍或达到射程
```

### 5.2 DRAY Wires（DRAY 导线）

DRAY 最强大的用法是构建"导线"——通过复制 SPRK 实现远距离信号传输：

```
DRAY Wire 原理：
  DRAY 检测到相邻 METL 上的 SPRK(life=3)
  → DRAY 沿发射方向复制 SPRK 粒子
  → 路径上每个空位产生一个 SPRK
  → 远端的 ARAY/CRAY 检测到这些 SPRK
  
优势：
  - 无延迟（光速传播，同一帧内到达）
  - 可跨越大距离
  - 不会像普通导线那样受传导距离限制
  
代价：
  - 需要精确的粒子顺序
  - DRAY 的 SPRK 只能存活一帧（下一帧 CONV 会重置源头）
```

### 5.3 DRAY 属性继承

```
DRAY 复制粒子时继承的属性：

  源粒子属性        DRAY 行为
  ─────────────────────────────
  ctype = SPRK    → 复制 SPRK（保留 ctype=导体类型）
  life = X        → 新粒子 life = X（可精确控制life值）
  tmp/tmp2        → 新粒子继承 tmp/tmp2
  temp            → 新粒子继承温度
  dcolour         → 新粒子继承装饰色
  
特殊用法：
  DRAY(life=3) → 创建的 SPRK 下一帧直接激活（跳过 life=4 等待）
  DRAY(life=0) → 创建未通电的导体（用于"冷"复制）
```

### 5.4 DRAY vs CONV 对比

| 特性 | CONV | DRAY |
|------|------|------|
| 操作方式 | 原地转换（保留 ID） | 复制到远处（新 ID） |
| 作用范围 | 3×3 邻域 | 直线射线（可远超 3 格） |
| 粒子 ID | 保留原 ID | 分配新 ID |
| 适用场景 | 导体重置 | 信号传输、数据复制 |
| 时序要求 | 必须在 BTRY 之前 | 必须在信号接收器之前 |
| 多目标 | 8 方向同时 | 单方向线性 |

---

## 6. BRAY 光束逻辑

### 6.1 BRAY 基本特性

BRAY（B-Type Ray）由 ARAY 创建，沿直线高速飞行。

```
BRAY 物理属性：
  - TYPE_ENERGY（能量型）
  - 速度：极快（每帧沿方向移动）
  - Life：自动递减，耗尽消失
  - 碰撞：击中固体/液体时可能触发反应
  - 温度：极高（可引燃可燃物）
```

### 6.2 BRAY Annihilation（对撞湮灭）

这是 Subframe 逻辑的基础——用 BRAY 实现布尔运算：

```
原理：
  两个 BRAY 从相反方向对撞 → 互相湮灭消失
  只有一个 BRAY 到达 → 触发目标

实现 AND（两路都通才输出）：
  输入 A ─→ BRAY_A ─┐
                      ├→ 对撞点 ← 目标
  输入 B ─→ BRAY_B ─┘
  
  结果：
    A=1, B=1 → BRAY_A 和 BRAY_B 都对撞湮灭 → 目标不被击中 = 0？不对...

实际实现（通过 FILT 中转）：
  输入 A ─→ BRAY_A → FILT_A(ctype 被修改)
  输入 B ─→ BRAY_B → FILT_B(ctype 被修改)
  DTEC 检测 FILT_A  and FILT_B 的 ctype → 输出
  
  或用 NOR-NOR 组合实现 AND（更可靠）
```

### 6.3 BRAY 路径设计

```
BRAY 路径规划要点：

  1. 直线传播——不能拐弯（除非用 FILT 散射模式）
  2. 途中障碍会触发反应——路径上必须是空位或透明粒子
  3. 多束 BRAY 可交叉（不互相干扰）
  4. 用 INSL/墙类在路径两侧建"管道"
  
路径布局示例：
  ARAY → ═══════════════ → FILT（目标）
          INSL 管道壁
  
  两束对撞布局：
  ARAY_1 → ════╗
                ║ 对撞区（需精确对准）
  ARAY_2 → ════╝
```

---

## 7. ARAY 角度射线

### 7.1 ARAY 工作方式

ARAY 检测 SPRK 并根据 SPRK 位置决定 BRAY 发射方向：

```
ARAY 扫描邻域 SPRK → 确定 SPRK 所在方向 → 发射 BRAY

方向映射：
  SPRK 在 ARAY 的右上方 → BRAY 向右上方发射（↗）
  SPRK 在 ARAY 的左下方 → BRAY 向左下方发射（↙）
  以此类推（8 方向映射到 8 方向）

多 SPRK 处理：
  如果 ARAY 周围有多个 SPRK：
  → 选择最先扫描到的 SPRK（通常是最左上的）
  → 只沿该方向发射一束 BRAY
```

### 7.2 多方向 ARAY

```
单个 ARAY 一帧只能发射一束 BRAY（一个方向）。

需要多方向时：
  - 使用多个 ARAY，每个对应一个方向
  - 各 ARAY 独立检测不同的 SPRK 源
  - 堆叠多个 ARAY 在同一像素（每层独立工作）

多方向 ARAY 阵列：
     SPRK_N   SPRK_NE
        ↓        ↓
     [ARAY_N] [ARAY_NE]
  
  SPRK_W → [ARAY_W]  [Central]  [ARAY_E] ← SPRK_E
  
     [ARAY_SW] [ARAY_S]
        ↓        ↓
     SPRK_SW   SPRK_S
```

### 7.3 ARAY 与 CONV 的时序配合

```
精确时序（同一帧内）：

  ID=100: SPRK(life=3) 在 ARAY 右侧
  ID=101: ARAY 检测到 → 向右发射 BRAY ✓
  ID=102: CONV 重置 SPRK → life=0
  ID=103: BTRY 重新通电 → 新 SPRK(life=4)
  
  下一帧：
  ID=100: 新 SPRK(life=3) → ARAY 再次检测 → 再次发射 ✓

关键：ARAY 必须在 CONV 之前（ID 更小），否则 ARAY 看到的是已重置的 life=0
```

---

## 8. CRAY 物质射线

### 8.1 CRAY 与 ARAY/BRAY 的区别

| 特性 | ARAY+BRAY | CRAY |
|------|-----------|------|
| 发射物 | BRAY（能量光束） | ctype 指定的任意粒子 |
| 射程控制 | BRAY.life 自然衰减 | tmp 参数精确控制 |
| 属性继承 | BRAY 自带高温 | 继承 CRAY 的温度/life/颜色 |
| 用途 | 信号传输、逻辑 | 物质创建、导体生成 |
| FILT 交互 | 不直接交互 | 可通过 FILT 过滤颜色 |

### 8.2 CRAY 作为"物质创建器"

```
CRAY 在 Subframe 中的独特价值：

  1. 直接创建 SPRK
     CRAY(ctype=SPRK) → 发射 SPRK 粒子
     → 可用于远距离"投放"通电导体
     → 绕过传导距离限制

  2. 创建特定 life 值的粒子
     CRAY.life = X → 发射的粒子 life = X
     → CRAY(life=3) 创建的 SPRK 下一帧直接激活
     
  3. 批量创建
     CRAY.tmp = 射程 → 一整条线上的粒子
     → 可用于初始化大规模电路

  4. FILT 颜色过滤
     CRAY 射线穿过 FILT → 创建的粒子继承 FILT 颜色
     → 数据编码在粒子颜色中传输
```

### 8.3 CRAY 发射模式（通过 SPRK.ctype 控制）

```
检测到的 SPRK 的 ctype 决定了 CRAY 的行为模式：

  SPRK.ctype = PSCN → 销毁模式
    射线路径上的非 DMND 粒子被销毁
    用于"清除"路径
    
  SPRK.ctype = INST → 穿透模式
    射线可穿过 CRAY 粒子不停止
    用于跨越障碍
    
  SPRK.ctype = INWR → 火花创建模式
    在路径上创建 SPRK 而非 ctype 粒子
    用于构建"火花雨"
    
  其他 → 普通发射模式
    创建 CRAY.ctype 指定的粒子类型
```

---

## 9. 堆叠技术完整指南

### 9.1 TPT 的三层粒子系统

```
每个像素同时容纳三层粒子：

  ┌─────────────────────┐
  │  装饰层（始终存在）   │  ← dcolour 字段
  ├─────────────────────┤
  │  pmap 主粒子层       │  ← 固体/液体/气体/粉末（1个可见粒子）
  │  (ID 最大=可见粒子)  │
  ├─────────────────────┤
  │  photons 光子层      │  ← 能量粒子（PHOT/NEUT/ELEC/PROT等）
  │  (0~N 个能量粒子)   │     多个能量粒子可共存
  └─────────────────────┘
```

### 9.2 堆叠的物理实现

```
堆叠 = 同一像素 pmap 中有多个不同类型粒子

堆叠规则：
  1. 每个像素最多 5 个堆叠粒子（正常情况）
  2. EHOLE 内最多 1500 个（特殊容器）
  3. 最顶层粒子 = 最后放置的 = 图形渲染的
  4. 更新顺序按 ID（不管堆叠层级）
  5. 某些元素互斥（如两种固体不能同时占据 pmap）
```

### 9.3 Subframe 中的典型堆叠结构

```
结构 1：基础 ARAY-CONV-BTRY 堆叠
  像素 (10, 5)：
    层 3（顶）：BTRY    — 持续电源
    层 2（中）：CONV    — 导体重置
    层 1（底）：METL    — 导体
    结果：3 个粒子共享一个像素，紧凑！
    
结构 2：带检测的堆叠
  像素 (10, 5)：
    层 4（顶）：PCLN(PHOT) — 光子克隆器（信号输出）
    层 3：LDTC            — 线性探测器（信号输入）
    层 2：CONV            — 重置
    层 1：METL            — 导体

结构 3：多层内存单元
  像素 (10, 5)：
    层 3：FILT(ctype=数据1)
    层 2：FILT(ctype=数据2)
    层 1：METL（地址选择器）
  两 bit 数据存储在两层 FILT 中，同像素！
```

### 9.4 堆叠的放置顺序

```
手动堆叠步骤：
  1. 用 METL 画第一层（底层）
  2. 暂停游戏（空格键）
  3. 用 Shift-S 进入堆叠模式
  4. 在 METL 上方画 CONV（自动叠加上去）
  5. 再画 BTRY（叠在更上层）
  6. 用 X 键查看/编辑各层

粒子 ID 分配（按放置顺序）：
  先放 METL → ID=100（小）
  再放 CONV → ID=101（中）
  后放 BTRY → ID=102（大）
```

### 9.5 堆叠与粒子顺序的交互

```
堆叠粒子在同一像素时的更新顺序：

像素 (10,5) 有 3 个粒子，ID 为 100, 101, 102
更新顺序：100 → 101 → 102（按 ID，不按层级）

但堆叠层级影响粒子间的相互作用：
  - 导电：BTRY(层3) → 向下接触 METL(层1) → 通电 ✓
  - 重置：CONV(层2) → 邻域检测 METL(层1) → 重置 ✓
  - 检测：ARAY(邻域) → 扫描邻域 → 看到 METL(层1)的 SPRK ✓
```

---

## 10. 第一个 Subframe 电路：从零开始

### 10.1 目标：构建一个持续运行的 BRAY 发射器

这是最简单的 Subframe 电路——每帧发射一束 BRAY。

### 10.2 物料清单

```
所需元素：
  1× METL（导体）
  1× CONV(ctype=METL)（导体复位器）
  1× BTRY（电源）
  1× ARAY（BRAY 发射器）
  
可选：
  1× INSL（绝缘体，隔离信号）
  1× DMND（阻挡 BRAY，安全考虑）
```

### 10.3 逐步构建

**步骤 1：放置基础导体**

```
在画布上从左到右放置：
  
  (5,10)  (6,10)  (7,10)  (8,10)
  [ARAY]  [METL]  [CONV]  [BTRY]
  
  注意顺序：ARAY 最左 → 先扫描 → 小 ID → 先更新 ✓
```

现在暂停，检查：METL 紧贴 ARAY（右下或正右），CONV 紧贴 METL，BTRY 紧贴 CONV。

**步骤 2：设置 CONV 的 Ctype**

```
  1. 用 METL 画一笔在 CONV 上（让 CONV "学习"METL 类型）
  2. 确认 CONV 的颜色变成 METL 的颜色
  3. 或者用 PROP 工具直接设置 CONV.ctype = METL 的 Type 值(14)
```

**步骤 3：构建 BRAY 管道**

```
在 ARAY 的发射方向上放置管道：

  ARAY(5,10) → 向右上方发射 BRAY
  在(6,9)到(15,4)放置 INSL 作为管道壁：
  
  (5,10)ARAY → ═══════════ INSL 管道 ═══════ → (16,3) DMND 挡板
                  BRAY 飞行路径（空位）
```

**步骤 4：启动电路**

```
  1. 取消暂停（空格键）
  2. BTRY 立即在 METL 上产生 SPRK
  3. 下一帧：SPRK.life=3 → ARAY 检测到 → 发射 BRAY
  4. CONV 重置 METL → BTRY 重新通电
  5. 每帧循环！
```

**步骤 5：验证时序**

```
  按 D 键进入调试模式（DEBUG_PARTS 视图）
  观察 METL 粒子：
    - 应该每帧闪烁（SPRK→METL→SPRK→METL...）
    - life 值在 4→0→4→0 之间切换
    - 如果停在某个值不动 → 顺序错误！
```

### 10.4 常见构建错误

| 错误现象 | 原因 | 修复 |
|---------|------|------|
| ARAY 不发射 | ARAY 在 CONV 之后更新（ID 太大） | 把 ARAY 往左/上移动 |
| 发射一帧就停 | CONV 在 BTRY 之后（导体来不及重新通电） | BTRY 往右/下移动 |
| SPRK 卡在 life=4 | BTRY 在 CONV 之前（重置被跳过） | 交换 BTRY 和 CONV 位置 |
| BRAY 击中 CONV | ARAY 方向偏了 | 调整 ARAY 和 SPRK 的相对位置 |
| 存档后失效 | 粒子顺序在加载时改变 | 按扫描顺序重排粒子位置 |

### 10.5 进阶变体

```
变体 1：双 ARAY 交替发射
  [ARAY_A] [METL_A] [CONV] [BTRY_A]
  [ARAY_B] [METL_B] [CONV] [BTRY_B]
  → 两束 BRAY 交替发射（每帧各一束）

变体 2：带使能控制的发射器
  [SWCH] → [PSCN] →
  [ARAY] [METL] [CONV] [BTRY]
  → SWCH 控制是否通电，实现"开关"

变体 3：DRAY 代替 ARAY
  [DRAY(ctype=PHOT)] [METL] [CONV] [BTRY]
  → 发射光子而非 BRAY（用于 FILT 交互）
```

---

## 11. 逻辑门完整实现

### 11.1 NOR 门（通用逻辑基础）

```
原理：
  任一输入为 1 → CONV 重置输出导体 → 输出 = 0
  所有输入为 0 → 输出导体保持通电 → 输出 = 1

电路布局：
  输入 A ─→ PSCN ─→ METL_A ─→ CONV_1(重置输出)
  输入 B ─→ PSCN ─→ METL_B ─→ CONV_2(重置输出)
  
  BTRY ─→ METL_OUT ─→ ARAY_OUT ─→ BRAY（输出信号）
                    ↑              ↑
              CONV_1, CONV_2 可重置此 METL

真值表：
  A  B  | 输出
  0  0  |  1   ← BTRY 通电正常
  0  1  |  0   ← CONV_B 重置输出
  1  0  |  0   ← CONV_A 重置输出
  1  1  |  0   ← 两个 CONV 都重置
```

### 11.2 从 NOR 构建所有逻辑门

```
NOT（非门）= NOR(A, A)
  输入 A ─┬─→ CONV_1 ─→ 重置输出
          └─→ CONV_2 ─→ 重置输出
  输出 = NOR(A, A)
  A=1 → 输出=0, A=0 → 输出=1 ✓

OR（或门）= NOT(NOR(A, B))
  NOR(A, B) → NOT → 输出
  = 两个 NOR 串联

AND（与门）= NOR(NOT(A), NOT(B))
  NOT(A) → NOR ← NOT(B)
  输出 = NOR(NOT(A), NOT(B))
  = 三个 NOR

XOR（异或门）= OR(AND(A, NOT(B)), AND(NOT(A), B))
  需要 5 个 NOR
```

### 11.3 SR 锁存器（1 比特内存）

```
电路：
  SET ─→ CONV_S ─→ 重置 METL_OUT
  RESET ─→ CONV_R ─→ 设置 METL_OUT
  BTRY ─→ METL_OUT ─→ ARAY ─→ 输出

行为：
  SET 通电  → 输出 = 0（重置后 BTRY 来不及通电）→ 需设计
  RESET 通电 → 输出 = 1
  都没通电 → 保持上一状态

时序必须极其精确——SET/RESET 的 CONV 必须在 BTRY 之后更新
```

### 11.4 D 触发器（边沿触发）

```
结构：
  CLK ─→ DLAY(延时1帧) ─→ 
  DATA ─→ CONV_D ─→ 重置/设置 METL_STORE
  BTRY ─→ METL_STORE（存储单元）
  
行为：
  CLK 上升沿时采样 DATA
  其他时间保持存储值不变
  
这是构建寄存器和计数器的基本单元
```

---

## 12. 时序与时钟系统

### 12.1 时钟发生器

```
方案 1：BTRY 连续时钟（最快 = 60Hz）
  BTRY ─→ METL ─→ [每帧一次 SPRK]
  优点：最快
  缺点：不可调速

方案 2：DLAY 可调时钟
  BTRY ─→ DLAY(temp=N) ─→ PSCN ─→ METL ─→ ARAY ─→ 反馈回 DLAY
  DLAY.temp = N → 延时 N 帧
  频率 = 60/N Hz
  HEAT/COOL 调节 DLAY 温度 = 调节频率

方案 3：外部触发时钟
  手动放置 SPRK → 触发一次
  或利用环境（LAVA 热胀冷缩、WATR 流动）产生不规则时钟
```

### 12.2 多相时钟

```
两相不重叠时钟（常用于同步电路）：

  Phase 1: BTRY_1 → DLAY(t=2) → METL_1 → 系统 A 部分
  Phase 2: BTRY_2 → DLAY(t=2) → METL_2 → 系统 B 部分
  
  Phase 1 和 Phase 2 错开 2 帧 → 不会同时激活
```

### 12.3 时钟分布网络

```
单一时钟源 → 分配给多个 Subframe 单元

  CLK_SRC ─┬─→ DRAY_Wire_1 ─→ 单元 1
            ├─→ DRAY_Wire_2 ─→ 单元 2
            ├─→ DRAY_Wire_3 ─→ 单元 3
            └─→ ...

  问题：DRAY 复制会消耗粒子 ID，远端单元收到信号有传播延迟
  解决：使用 BRAY 光束（光速无延迟）或 CONV 链接力
```

---

## 13. FILT RAM 内存系统

### 13.1 FILT 作为 30 位存储器

```
FILT 内存单元：
  ctype = 30 位数据（bit0~bit29）
  tmp   = 工作模式（0~9）
  
  每个 FILT 粒子可存储 30 bit = 约 3.75 字节
  100 个 FILT = 375 字节
  1000 个 FILT = 3.75 KB（对于 TPT 来说相当可观！）
```

### 13.2 FILT ctype 数据编码

```
ctype 位布局（30 位有效）：

  bit 0-9:   红色分量（10 位，每 2 位一级 = 5 级）
  bit 10-19: 绿色分量（10 位，每 2 位一级 = 5 级）
  bit 20-29: 蓝色分量（10 位，每 2 位一级 = 5 级）
  
  总共 2^30 ≈ 10 亿种颜色组合（实际受限于 5 级色深）
  
  二进制存储用法：
    只使用 bit0 = 存储 1 bit 二进制
    使用 bit0~3 = 存储 4 bit（16 值）
    全部 30 bit = 存储完整颜色/波长数据
```

### 13.3 写入 FILT

```
方法 1：DRAY 复制写入
  DRAY(ctype=新数据的FILT) → 发射射线 → 目标位置
  → 复制一个带新 ctype 的 FILT 覆盖旧 FILT
  
  需要：DRAY 的 ctype 设置为"目标 FILT 类型"
  注意：DRAY 复制的 FILT 是全新粒子（新 ID）

方法 2：CRAY 创建写入
  CRAY(ctype=FILT) + CRAY.dcolour = 新数据
  → 发射 FILT 粒子到目标位置

方法 3：PHOT 波长写入
  PHOT 击中 FILT → FILT.tmp 模式决定如何修改 ctype
  模式 0-6 各有不同颜色操作
```

### 13.4 读取 FILT

```
方法 1：DTEC 检测
  DTEC(ctype=目标波长范围) 检测 FILT 的 ctype
  → ctype 匹配 → 输出 SPRK
  → 用于二进制判断（等于/不等于）

方法 2：LDTC 线性扫描
  LDTC 扫描 FILT 行 → 逐行读取 ctype
  → 可用于顺序读取多个 FILT

方法 3：颜色可视化
  直接看 FILT 的颜色（人眼读取，调试用）
  或通过 ARAY+BRAY 照射 FILT 观察透射光颜色
```

### 13.5 地址解码

```
一维地址线（N 个 FILT 单元）：

  地址 A[0]  A[1]  A[2]
    ↓       ↓       ↓
  FILT_0  FILT_1  FILT_2  ...  FILT_N
  
  地址选择：用 ARAY 向对应 FILT 发射 BRAY
  或：用 CONV 链逐个激活"字线"

二维地址（行列解码）：
        列0    列1    列2
  行0 [FILT] [FILT] [FILT]
  行1 [FILT] [FILT] [FILT]
  行2 [FILT] [FILT] [FILT]
  
  行选择：ARAY 水平发射 BRAY 选中整行
  列选择：ARAY 垂直发射 BRAY 选中整列
  交叉点 = 目标 FILT
```

### 13.6 FILT RAM 完整读写周期

```
写周期（3 步）：
  1. 地址解码 → 选中目标 FILT 单元
  2. 数据准备 → DRAY/CRAY 携带新数据
  3. 写入执行 → 替换目标 FILT
     （需要 >1 帧完成，除非在同一帧内安排好时序）

读周期（2 步）：
  1. 地址解码 → 选中目标 FILT
  2. DTEC/LDTC 检测 ctype → 输出 SPRK 或数据
```

---

## 14. 数据总线与多比特传输

### 14.1 并行总线

```
4 位并行总线（同时传输 4 bit）：

  Bit3 ─→ [DRAY_3] ─→ [FILT_3]
  Bit2 ─→ [DRAY_2] ─→ [FILT_2]
  Bit1 ─→ [DRAY_1] ─→ [FILT_1]
  Bit0 ─→ [DRAY_0] ─→ [FILT_0]
  
  每 bit 独立的 DRAY + FILT 通道
  同一时钟边沿同时传输全部 4 bit
  
  优势：速度快（一帧传输全部位）
  代价：占用空间大（N bit = N 条独立通道）
```

### 14.2 串行总线

```
串行传输（逐 bit 传输）：

  CLK ─→ [移位寄存器] → Bit_Out
                           ↓
         Bit3 → Bit2 → Bit1 → Bit0（逐个输出）
  
  每帧传输 1 bit，N bit 需要 N 帧
  
  优势：占用空间小（一条通道）
  代价：速度慢（需要多帧）
```

### 14.3 多路复用

```
2 路复用器（2-to-1 MUX）：

  SEL ─→ [选择器]
  IN0 ─→ [DTEC_0] ─→ [MUX逻辑] ─→ OUT
  IN1 ─→ [DTEC_1] ─→ [MUX逻辑] ─→ OUT
  
  SEL=0 → OUT = IN0
  SEL=1 → OUT = IN1
  
  在 Subframe 中：SEL 用 SPRK 的有无表示
```

---

## 15. 高级电路模式

### 15.1 移位寄存器

```
4 位移位寄存器：

  CLK ─→ [DLAY(t=1)] ─→ 触发所有 DFF
  
  DATA_IN ─→ [DFF_0] ─→ [DFF_1] ─→ [DFF_2] ─→ [DFF_3] ─→ DATA_OUT
  
  每个 CLK 边沿：
    DFF_0 采样 DATA_IN
    DFF_1 采样 DFF_0 的旧值
    DFF_2 采样 DFF_1 的旧值
    DFF_3 采样 DFF_2 的旧值
  
  → 数据右移一位

应用：
  - 串行→并行转换
  - 并行→串行转换
  - 延迟线
  - 序列发生器
```

### 15.2 计数器

```
2 位二进制计数器（计数 0→1→2→3→0→...）：

  状态存储：2 个 SR 锁存器 = 2 bit
  CLK ─→ 递增逻辑（半加器 + 进位链）
  
  电路：
    第 1 帧：00
    第 2 帧：01（bit0 翻转）
    第 3 帧：10（bit0 翻转 + 进位到 bit1）
    第 4 帧：11（bit0 翻转）
    第 5 帧：00（溢出归零）
```

### 15.3 状态机

```
简单状态机（3 态循环）：

  ┌───┐  condition_1  ┌───┐  condition_2  ┌───┐
  │ S0 │ ───────────→ │ S1 │ ───────────→ │ S2 │
  └───┘               └───┘               └───┘
    ↑                                         │
    └───────────── condition_0 ───────────────┘

在 Subframe 中：
  每个状态 = 一个 METL + CONV + BTRY 单元
  状态转换 = DTEC 检测条件 → SPRK → 激活下一状态
  同一时刻只有一个状态"活跃"（对应的 SPRK 存在）
```

### 15.4 小规模 CPU 架构

```
一个最小 CPU 包含：

  1. PC（程序计数器）= 计数器
  2. IR（指令寄存器）= FILT 存储
  3. 指令解码 = DTEC + 逻辑门阵列
  4. ALU（算术逻辑单元）= 加法器 + 逻辑门
  5. 寄存器文件 = 一组 FILT 或 SR 锁存器
  6. 控制单元 = 状态机

指令执行周期：
  1. FETCH：从 FILT RAM 读取指令（PC→地址→读出）
  2. DECODE：DTEC 解码指令类型
  3. EXECUTE：ALU 执行操作
  4. WRITEBACK：结果写回寄存器
  5. PC++：计数器递增
  
  每步至少 1 帧 → 最简 CPU 至少 5 帧/指令 ≈ 12 IPS
  优化后可达到接近 60 IPS（每帧一条指令）
```

---

## 16. Subframe Mod 完整参考

### 16.1 安装与配置

```bash
# 克隆 Mod 仓库
git clone https://github.com/wxwisiasdf/TPT-Subframe-Mod
cd TPT-Subframe-Mod

# 构建（参考 TPT 标准构建流程）
meson setup build
cd build
ninja

# 或使用预编译版本（从 GitHub Releases 下载）
```

### 16.2 完整快捷键表

| 快捷键 | 功能 | 使用场景 |
|--------|------|---------|
| **Shift-F5** | 重载粒子顺序 | 存档后修复电路 |
| **Shift-Space** | 子帧动画（逐粒子单步） | 调试时序 |
| **Shift-S** | 堆叠工具开关 | 多层构建 |
| **Shift-D** | 堆叠绘制模式 | 在现有粒子上叠加 |
| **C** | 配置工具 | 快速设置元素属性 |
| **X** | 堆叠编辑（增加深度） | 查看/修改下层粒子 |
| **Shift-X** | 反向堆叠编辑（减少深度） | 查看/修改上层粒子 |
| **Alt-F** | 单粒子更新 | 精确步进调试 |
| **Shift-F** | 更新到光标位置 | 批量更新测试 |
| **Alt-Z** | 子帧录制开始/停止 | 录制时序动画 |
| **Alt-R** | 子帧回放 | 回放录制的时序 |

### 16.3 Lua 调试 API

```lua
-- === 粒子顺序管理 ===

-- 启用自动粒子顺序修复（存档加载时自动执行）
tpt.autoreload_enable(1)

-- 手动重载粒子顺序
tpt.autoreload()

-- === 调试工具 ===

-- 启用子帧调试标志（0x8 = 显示粒子ID和life）
tpt.setdebug(0x8)

-- 设置调试标志组合
-- 0x1 = DEBUG_PARTS（类型+life）
-- 0x2 = DEBUG_ELEMENTPOP（元素统计）
-- 0x4 = DEBUG_LINES（WIFI连线）
-- 0x8 = 子帧专用（ID + order info）

-- === 粒子检查 ===

-- 打印指定位置所有堆叠粒子的信息
local function inspect_stack(x, y)
    local i = sim.pmap(x, y)
    while i do
        local p = sim.partProperty(i, "type")
        local life = sim.partProperty(i, "life")
        local ctype = sim.partProperty(i, "ctype")
        print(string.format("ID=%d type=%d life=%d ctype=%d", i, p, life, ctype))
        -- 获取堆叠中的下一个粒子
        i = sim.partProperty(i, "stack_next") -- 注意：实际API可能不同
    end
end

-- === 时序录制 ===

-- 开始录制
tpt.record_start("my_subframe_timelapse")

-- 录制第 N 帧
for i = 1, 100 do
    sim.frame()  -- 单帧步进
end

-- 停止录制
tpt.record_stop()

-- === 批量操作 ===

-- 自动构建重复结构
local function build_dff_array(start_x, start_y, count)
    for i = 0, count - 1 do
        local x = start_x + i * 4  -- 每单元间隔 4 像素
        sim.partCreate(-1, x, start_y, "METL")
        sim.partCreate(-1, x + 1, start_y, "CONV")
        sim.partCreate(-1, x + 2, start_y, "BTRY")
        -- 设置 CONV ctype
        -- ... (需要找到刚创建的 CONV 粒子 ID)
    end
end
```

---

## 17. 调试与排错

### 17.1 系统化调试方法

```
调试 Subframe 电路的标准流程：

1. 隔离测试
   → 只保留最简电路（1个ARAY+1个METL+1个CONV+1个BTRY）
   → 验证基本循环是否工作
   → 逐步添加功能

2. 逐帧检查（Alt-F）
   → 按 Alt-F 单步更新
   → 观察每个粒子的 life 值变化
   → 确认 ARAY 在 life==3 时触发

3. 粒子顺序验证（Shift-F5）
   → 重载粒子顺序
   → 在调试模式下查看 ID（0x8标志）
   → 确认 ID 大小关系正确

4. 信号追踪
   → 从源头（BTRY）开始
   → 沿信号路径逐个检查粒子状态
   → 找出信号在哪里"丢失"
```

### 17.2 常见故障模式

| 症状 | 诊断 | 解决 |
|------|------|------|
| ARAY 从不发射 | SPRK life 未达到 3（在到达前被重置） | ARAY 必须在 CONV 之前，确认 ID 顺序 |
| 发射不稳定（有时有有时无） | 粒子顺序边界情况，ID 恰好临界 | 增加 ARAY 和 CONV 之间的物理距离 |
| 第一帧正常之后停止 | CONV 重置在 BTRY 之前，BTRY 产生 SPRK 后 CONV 又重置 | 交换 CONV 和 BTRY 位置 |
| 信号串扰（相邻电路互相干扰） | SPRK 通过共用导体泄漏到相邻电路 | 用 INSL 隔离各电路的导体 |
| 堆叠粒子意外反应 | 不同层粒子物理接触 | 调整堆叠元素选择（避免互克元素） |
| BRAY 路径被阻挡 | 路径上有非透明粒子 | 清理 BRAY 路径，或使用 INSL 管道 |

### 17.3 使用 Shift-Space 调试时序

```
Shift-Space 子帧动画模式：
  1. 按 Shift-Space 进入
  2. 模拟器暂停
  3. 按 [→] 键 → 更新下一个粒子
  4. 观察每个粒子更新时的状态变化
  5. 屏幕高亮当前更新的粒子
  6. 按 Shift-Space 退出
  
这是理解粒子顺序和时序的最直观方式！
```

---

## 18. 性能优化

### 18.1 粒子数量控制

```
每个 Subframe 单元需要的粒子：
  基础（ARAY/METL/CONV/BTRY）：4 个
  带 DRAY：+1 个
  带 FILT：+1 个
  带 DTEC：+1 个

大型电路优化：
  - 共享 CONV（一个 CONV 可重置多个相邻导体）
  - 共享 BTRY（一个 BTRY 可供电多个导体）
  - 减少不必要的堆叠层数
  
TPT 粒子限制：~100 万粒子（取决于硬件）
Subframe 电路通常 1K~100K 粒子
```

### 18.2 空间布局优化

```
紧凑布局 vs 可调试布局：

紧凑布局（同一像素堆叠）：
  [+] 节省空间
  [+] 更多电路可放入画布
  [-] 难以调试
  [-] 放置容易出错
  
可调试布局（展开排列）：
  [+] 容易观察每个粒子
  [+] 容易修改
  [-] 占用更多空间
  [-] 信号路径更长

建议：开发阶段用展开布局，完成后压缩为紧凑布局
```

### 18.3 级联深度优化

```
级联深度 = 一帧内串联的 Subframe 单元数

深度过大导致的问题：
  - 帧时间过长（TPT 可能跳帧）
  - 后面的级联可能被截断（帧结束时未更新完）
  - 增加调试复杂度

建议：
  - 单帧级联深度 < 画布宽度的 1/4（1920px → < 480 级）
  - 将超长逻辑链拆分到多帧完成
  - 使用流水线（pipeline）设计
```

---

## 19. 学习路线与社区资源

### 19.1 学习路线

#### 第一阶段：基础（1-3 天）
1. 理解 SPRK 生命周期（life=4→3→2→1→0）
2. 理解粒子执行顺序（左→右，上→下）
3. 手动构建第一个 ARAY-CONV-BTRY 循环
4. 使用调试模式观察粒子 life 变化

#### 第二阶段：核心技巧（1-2 周）
5. 掌握 CONV 重置的精确时序
6. 堆叠技术——在同一像素放置多层粒子
7. DRAY Wires——用 DRAY 传输信号
8. NOR 逻辑——构建第一个逻辑门
9. 学习使用 Subframe Mod

#### 第三阶段：逻辑设计（2-4 周）
10. 构建完整逻辑门集（NOT/AND/OR/XOR）
11. SR 锁存器和 D 触发器
12. FILT 读写（单 bit 存储）
13. 时钟发生器与分频器
14. 简单状态机

#### 第四阶段：数字系统（1-2 月）
15. 移位寄存器
16. 计数器
17. FILT RAM（多 bit 存储阵列）
18. 地址解码器
19. 数据总线（并行/串行）

#### 第五阶段：计算机架构（2-6 月）
20. ALU 设计
21. 指令集设计
22. 控制单元
23. 完整 CPU
24. 图形输出（显示器）

### 19.2 社区资源

| 资源 | 位置 |
|------|------|
| **The Subframe Lessons** (mark2222) | powdertoy.co.uk 论坛 Thread #22405 |
| **A3SFTT 原始论文** (Schmolendevice) | powdertoy.co.uk 论坛 Thread #19943 |
| **Subframe Chipmaker Mod** | github.com/wxwisiasdf/TPT-Subframe-Mod |
| **Subframe 社区 Discord** | discord.gg/fjF24Hc |
| **Subframe Tutorial (fixed)** | powdertoy.co.uk Thread #27116 |
| **DISP56S 高分辨率彩色屏幕** | powdertoy.co.uk Thread #22849 |
| **FILT RAM 讨论** | powdertoy.co.uk Thread #23125 |

### 19.3 推荐学习存档

```
入门存档（在 TPT 存档浏览器中搜索）：
  - "Subframe tutorial" — 基础教程存档
  - "SL101 Preliminaries" — mark2222 的第一课
  - "Subframe Plotter" (ID: 2200079) — 绘图演示
  - "SL113 BRAY Annihilation" (ID: 2305646) — BRAY 湮灭示例
  
进阶存档：
  - A3SFTT 计算机演示
  - DISP56S 显示器
  - 各种 FILT RAM 实现
```

---

## 相关附录

- [附录G：电路构建指南](appendix-G-电路构建指南.md) — 基础电路入门（SR锁存器、时钟、PWM等）
- [附录D：Lua API 参考](appendix-D-Lua-API参考.md) — Subframe 自动化脚本 API
- [附录O：FILT波长模式参考](appendix-O-FILT波长模式与LAVA-Ctype参考.md) — FILT RAM 数据编码和 10 种 tmp 模式
- [附录H：调试模式指南](appendix-H-调试模式指南.md) — DEBUG_PARTS/DEBUG_LINES 视图用法
- [附录P：粒子标志位参考](appendix-P-粒子标志位与状态常量参考.md) — PROP_xxx 标志位完整说明
- [附录N：CanMove交互规则](appendix-N-CanMove交互规则.md) — 粒子移动限制对电路布局的影响

---

*本文档整合了 mark2222 的 The Subframe Lessons (2018)、Schmolendevice 的 A3SFTT、wxwisiasdf 的 Subframe Chipmaker Mod 文档、以及社区多年的实践经验。建议配合 Subframe Mod 和调试模式实践每个概念。*
