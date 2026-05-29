# 附录R：Lua脚本入门指南

TPT 内置完整的 Lua 脚本引擎。本指南面向零基础用户，从打开控制台到写出实用脚本，逐步讲解。完成后可阅读**附录D**获取完整 API 参考。

---

## 目录

1. [什么是Lua脚本](#1-什么是lua脚本)
2. [打开控制台与执行第一条命令](#2-打开控制台与执行第一条命令)
3. [脚本管理器](#3-脚本管理器)
4. [Lua语法速成](#4-lua语法速成)
5. [TPT核心API实战](#5-tpt核心api实战)
6. [常用脚本配方（复制即用）](#6-常用脚本配方复制即用)
7. [安装和运行社区脚本](#7-安装和运行社区脚本)
8. [调试技巧与常见错误](#8-调试技巧与常见错误)
9. [进阶学习路线](#9-进阶学习路线)

---

## 1. 什么是Lua脚本

### 概述

Lua 是一种轻量、快速的脚本语言，被广泛嵌入游戏和模拟器中。TPT 使用 Lua 5.2 引擎（LuaJIT 编译），让你可以用代码操控模拟的每一个方面。

### 为什么要用Lua脚本？

| 场景 | 手工操作 | 用脚本 |
|------|---------|--------|
| 清空所有 FIRE 粒子 | 按查找键逐个清除 | 一行 `for` 循环 |
| 每30秒自动保存 | 手动按保存，容易忘 | TICK 事件定时触发 |
| 创建自定义元素 | 只能修改存档文件 | `elements.allocate` 直接定义 |
| 批量生成图案 | 逐格绘制，耗时巨大 | `sim.createParts` 区域生成 |
| 实时显示数据 | 只能开HUD看几个值 | `graphics.drawText` 任意显示 |
| 自动化实验 | 手工重复劳动 | 脚本一键执行 |

### Lua能做和不能做的事

**能做：**
- 创建、删除、修改粒子属性
- 读取和修改地图数据（压力、温度、墙壁等）
- 注册键盘/鼠标事件实现自定义交互
- 在屏幕上绘制文字、线条、图形
- 自定义元素（全部属性 + 每帧回调函数）
- 读写文件、发送HTTP请求
- 控制模拟状态（暂停/恢复/清空等）

**不能做：**
- 直接修改TPT引擎C++源码逻辑
- 突破粒子上限（MAX_PARTS）
- 访问操作系统底层（受沙箱限制）

---

## 2. 打开控制台与执行第一条命令

### 打开Lua控制台

```
按键盘左上角的 ` 键（反引号，位于ESC下方、Tab上方、数字1左边）
```

```
┌──────────────────────────────────────┐
│  ESC                                 │
│   `  (反引号 ~ 键 ── 按这个打开控制台) │
│  Tab                                 │
├──────────────────────────────────────┤
│                                      │
│          TPT 模拟画布                  │
│                                      │
│  ┌──────────────────────────────────┐│
│  │ > _                              ││  ← 这是Lua控制台窗口
│  │                                  ││
│  └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

- 按一次 `` ` `` 打开控制台
- 再按一次 `` ` `` 关闭控制台
- 打开后出现 `>` 提示符，可直接输入 Lua 命令

### 你的第一条命令

在控制台中输入以下内容，然后按回车：

```lua
tpt.log("Hello, TPT!")
```

**结果：** 控制台输出 `Hello, TPT!`。恭喜，你已经运行了第一条 Lua 脚本！

### 尝试更多单行命令

```lua
-- 查看版本
tpt.log(tpt.version().tpt)

-- 查看当前粒子数
tpt.log(sim.partCount())

-- 查看绘制区的宽高
tpt.log("画面大小: " .. XRES .. " x " .. YRES)

-- 在画布中央创建一个水粒子
sim.partCreate(-1, 306, 192, "WATR")
```

**小提示：**
- `--` 开头的是注释，不会被执行
- `..` 用于拼接字符串
- 每条命令按回车后立即执行
- 按 `` ` `` 关闭控制台回到画布

### 控制台的多行输入

如果代码跨多行（如 `if` 语句），控制台会自动进入续行模式。也可以一次性粘贴多行代码：

```lua
for i = 1, 10 do
    sim.partCreate(-1, 100 + i * 4, 100, "WATR")
end
```

粘贴整段代码到控制台，按回车即可执行。

---

## 3. 脚本管理器

单行命令虽然方便，但复杂脚本需要保存和复用。TPT 内置了脚本管理器。

### 打开脚本管理器

```
方式1：菜单栏 -> Scripts -> Script Manager
方式2：控制台中按键 Ctrl+Shift+S（视版本而定）
```

### 脚本管理器界面说明

```
┌─────────────────────────────────────────┐
│  Script Manager                    [X]  │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────┐    │
│  │ scripts/                        │    │
│  │   ├── my_script.lua             │    │
│  │   ├── auto_save.lua             │    │
│  │   └── particle_count.lua        │    │
│  └─────────────────────────────────┘    │
│                                         │
│  [ New ]  [ Open ]  [ Save ]  [ Run ]  │
│  [ Stop ] [ Reload ]                    │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────┐    │
│  │ -- 脚本编辑区                     │    │
│  │ tpt.log("Hello from script!")   │    │
│  │                                 │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

| 按钮 | 功能 |
|------|------|
| **New** | 创建新脚本文件 |
| **Open** | 打开已有 `.lua` 文件 |
| **Save** | 保存当前脚本 |
| **Run** | 运行当前脚本 |
| **Stop** | 停止正在运行的脚本 |
| **Reload** | 重新加载脚本（修改后重新运行前使用） |

### 创建你的第一个脚本文件

**步骤：**

1. 打开 Script Manager
2. 点击 **New**
3. 在编辑区输入以下代码：

```lua
-- my_first_script.lua
tpt.log("========== 我的第一个脚本 ==========")
tpt.log("当前TPT版本: " .. tpt.version().tpt)
tpt.log("粒子总数: " .. sim.partCount())
tpt.log("画布大小: " .. XRES .. "x" .. YRES)
tpt.log("====================================")
```

4. 点击 **Save**，命名为 `my_first_script.lua`
5. 点击 **Run** 运行
6. 在控制台中查看输出

### 脚本文件保存位置

脚本默认保存在 TPT 目录下的 `scripts/` 文件夹中。也可以放在 TPT 可访问的任意路径下：

```
Windows: %APPDATA%/The Powder Toy/scripts/
Linux:   ~/.local/share/The Powder Toy/scripts/
MacOS:   ~/Library/Application Support/The Powder Toy/scripts/
```

### 自动运行脚本 (autorun)

如果你希望某个脚本在 TPT 启动时自动运行，将其命名为 `autorun.lua` 并放在 `scripts/` 目录中。

```
scripts/
  ├── autorun.lua      ← TPT启动时自动执行
  ├── my_tools.lua     ← 手动加载的脚本
  └── experiments.lua  ← 手动加载的脚本
```

---

## 4. Lua语法速成

以下是最常用的 Lua 语法。即使零编程基础，跟着示例敲一遍就能掌握。

### 4.1 变量与数据类型

```lua
-- 变量定义（Lua是动态类型，不需要声明类型）
local a = 10           -- 数字
local b = 3.14         -- 数字（小数）
local name = "WATR"    -- 字符串
local isHot = true     -- 布尔值
local nothing = nil    -- 空值/不存在

-- 推荐使用 local（局部变量），避免污染全局空间
-- 但有时需要全局变量（如跨事件共享状态）
global_counter = 0     -- 全局变量（不加 local）

-- 多重赋值
local x, y = 100, 200
local vx, vy = 1.5, -2.0

-- 字符串拼接用 ..
tpt.log("x坐标: " .. x .. ", y坐标: " .. y)
```

### 4.2 运算符

```lua
-- 算术运算
local sum = 3 + 4      -- 7（加）
local diff = 10 - 3    -- 7（减）
local prod = 5 * 6     -- 30（乘）
local quot = 20 / 4    -- 5（除）
local mod = 17 % 5     -- 2（取余）
local pow = 2 ^ 8      -- 256（幂运算）

-- 比较运算（返回 true/false）
local eq = (a == b)    -- 等于
local ne = (a ~= b)    -- 不等于
local lt = (a < b)     -- 小于
local le = (a <= b)    -- 小于等于
local gt = (a > b)     -- 大于
local ge = (a >= b)    -- 大于等于

-- 逻辑运算
local andResult = (a > 5) and (b < 10)   -- 与
local orResult  = (a < 0) or (b > 0)     -- 或
local notResult = not (a == 10)          -- 非
```

### 4.3 条件判断

```lua
-- if 语句
local temp = 500

if temp > 1000 then
    tpt.log("高温！")
elseif temp > 500 then
    tpt.log("中温")
else
    tpt.log("低温")
end

-- 单行写法（适合简单判断）
if temp > 1000 then tpt.log("很热") end

-- 三元表达式的Lua替代
local state = (temp > 500) and "hot" or "cold"
```

### 4.4 循环

```lua
-- for 循环（数字范围）
for i = 1, 10 do
    tpt.log("第 " .. i .. " 次")
end

-- for 循环（带步长）
for i = 0, 100, 10 do   -- 0, 10, 20, ..., 100
    tpt.log(i)
end

-- while 循环
local count = 0
while count < 5 do
    tpt.log("count = " .. count)
    count = count + 1
end

-- repeat...until 循环（至少执行一次）
local n = 1
repeat
    n = n * 2
until n > 100
tpt.log("n = " .. n)  -- 128

-- 提前跳出循环
for i = 1, 1000 do
    if i > 5 then
        break    -- 跳出循环
    end
    tpt.log(i)
end
```

### 4.5 函数

```lua
-- 定义函数
local function add(a, b)
    return a + b
end

local result = add(3, 5)   -- result = 8

-- 多返回值
local function getPosition()
    return 306, 192     -- 返回画布中心
end
local cx, cy = getPosition()

-- 匿名函数（常用于事件注册）
event.register(evt.TICK, function()
    tpt.log("每帧都执行")
end)

-- 可变参数
local function sum(...)
    local total = 0
    for _, v in ipairs({...}) do
        total = total + v
    end
    return total
end
tpt.log(sum(1, 2, 3, 4, 5))   -- 15
```

### 4.6 表（Table）

Lua 的表是唯一的数据结构，同时充当数组、字典和对象。

```lua
-- 数组（索引从1开始，注意不是0！）
local fruits = {"apple", "banana", "orange"}
tpt.log(fruits[1])     -- "apple"
tpt.log(#fruits)       -- 3（数组长度）

-- 遍历数组
for i, v in ipairs(fruits) do
    tpt.log(i .. ": " .. v)
end

-- 字典（键值对）
local props = {
    name = "WATR",
    temp = 373,
    id = 3
}
tpt.log(props.name)    -- "WATR"
tpt.log(props["temp"]) -- 373（两种访问方式等价）

-- 遍历字典
for key, value in pairs(props) do
    tpt.log(key .. " = " .. tostring(value))
end

-- 混合使用
local particle = {
    type = "WATR",
    x = 100,
    y = 200,
    temp = 373
}

-- 添加新键
particle.vx = 0.5

-- 删除键
particle.vx = nil

-- 嵌套表
local grid = {}
for x = 1, 10 do
    grid[x] = {}
    for y = 1, 10 do
        grid[x][y] = 0
    end
end
```

### 4.7 字符串处理

```lua
-- string 函数
local s = "  Hello TPT World  "
tpt.log(string.upper(s))     -- "  HELLO TPT WORLD  "
tpt.log(string.lower(s))     -- "  hello tpt world  "
tpt.log(string.sub(s, 3, 7)) -- "Hello"
tpt.log(string.len(s))       -- 19

-- string.format（格式化，类似C的printf）
local name = "FIRE"
local count = 42
local temp = 973.15
tpt.log(string.format("元素: %s, 数量: %d, 温度: %.1fK", name, count, temp))
-- 输出: 元素: FIRE, 数量: 42, 温度: 973.2K

-- 模式匹配（简单查找）
local found = string.find(s, "TPT")  -- 返回起始位置
tpt.log("找到TPT在位置: " .. found)

-- 字节与字符互转
local code = string.byte("A")   -- 65
local char = string.char(65)    -- "A"
```

### 4.8 数学函数

```lua
-- 常用数学函数
local abs = math.abs(-10)         -- 10（绝对值）
local ceil = math.ceil(3.14)     -- 4（向上取整）
local floor = math.floor(3.14)   -- 3（向下取整）
local max = math.max(5, 10, 3)   -- 10（取最大值）
local min = math.min(5, 10, 3)   -- 3（取最小值）
local sqrt = math.sqrt(16)       -- 4（平方根）
local sin = math.sin(math.pi/2)  -- 1.0（正弦，弧度制）
local cos = math.cos(0)          -- 1.0（余弦，弧度制）
local rnd = math.random()        -- [0,1) 随机小数
local rndInt = math.random(1, 10)-- [1,10] 随机整数

-- 限制值在范围内
local function clamp(val, low, high)
    return math.max(low, math.min(high, val))
end
local safe = clamp(1500, 0, 1000)  -- 1000

-- 温度单位换算
local function kelvinToCelsius(k)
    return k - 273.15
end
local function celsiusToKelvin(c)
    return c + 273.15
end
```

### 4.9 注释

```lua
-- 单行注释

--[[
多行注释
可以跨越多行
--]]

--[[
多行注释的快速关闭技巧：
在前面的 --[[ 后加一个等号变成 --[=[
然后添加 ]=] 来关闭，这样可以在注释内自由使用 ]]
-- 现在这里 --]] 不会提前终止注释
--]=]
```

---

## 5. TPT核心API实战

以下所有代码均可直接在控制台中运行，或粘贴到脚本管理器中执行。

### 5.1 粒子操作

#### sim.partCreate -- 创建粒子

```lua
-- 基本创建
local id = sim.partCreate(-1, 300, 200, "WATR")
--  参数1: -1 = 自动分配ID
--  参数2,3: x, y 像素坐标（注意不是CELL坐标！）
--  参数4: 元素名(字符串) 或 数字Type值

-- 用数字ID创建
local id2 = sim.partCreate(-1, 310, 200, 3)    -- 3 = WATR

-- 指定初始速度
local id3 = sim.partCreate(-1, 320, 200, "FIRE", 0)  -- v=0表示用默认速度

-- 检查是否创建成功
if id == -1 then
    tpt.log("创建失败！可能坐标越界或达到粒子上限")
else
    tpt.log("粒子创建成功，ID: " .. id)
end

-- 批量创建（一行水珠）
for i = 0, 20 do
    sim.partCreate(-1, 100 + i * 3, 100, "WATR")
end
```

使用 `-1` 让系统自动分配ID是最安全的方式。只有当你知道该ID未被占用时，才手动指定ID。

```
坐标系说明：
  X: 0 → 612   (左 → 右)
  Y: 0 → 384   (上 → 下)

画布中心: (306, 192)

┌───────────────────────── (612, 0)
│                          │
│         (306, 192)       │
│          ★ 中心           │
│                          │
└───────────────────────── (612, 384)
(0, 384)
```

#### sim.partKill -- 删除粒子

```lua
-- 创建后马上删除（演示用）
local id = sim.partCreate(-1, 300, 200, "WATR")
sim.partKill(id)              -- 默认死亡类型

-- 不同死亡类型（影响死亡时产生的副产品）
sim.partKill(id, 1)           -- 死亡类型1

-- 删除鼠标位置粒子
local mx, my = ui.mousePosition()
local id = sim.partID(mx, my)
if id then
    local t = sim.partProperty(id, 0)
    sim.partKill(id)
    tpt.log("删除了一个 " .. t .. " 粒子")
end
```

#### sim.partProperty -- 读/写粒子属性

```lua
-- 先创建一个粒子来操作
local id = sim.partCreate(-1, 300, 200, "WATR")

-- === 读取属性 ===
local typeVal = sim.partProperty(id, 0)       -- 用数字索引读类型
local life    = sim.partProperty(id, "life")   -- 用字符串名读生命
local temp    = sim.partProperty(id, "temp")   -- 温度
local ctype   = sim.partProperty(id, "ctype")  -- 携带类型

tpt.log(string.format("类型:%d 生命:%d 温度:%.1fK", typeVal, life, temp))

-- === 写入属性 ===
sim.partProperty(id, "temp", 500)              -- 设为500K
sim.partProperty(id, "life", 100)              -- 设生命值为100
sim.partProperty(id, "ctype", 3)              -- 设ctype为3(WATR)

-- 将水改为沸腾温度
sim.partProperty(id, 7, 373.15)                -- 数字7 = temp字段
```

**粒子属性字段索引速查：**

| 索引 | 字段名 | 说明 |
|------|--------|------|
| 0 | type | 粒子类型 |
| 1 | life | 生命值 |
| 2 | ctype | 携带类型 |
| 3 | x | X坐标（像素） |
| 4 | y | Y坐标（像素） |
| 5 | vx | X速度 |
| 6 | vy | Y速度 |
| 7 | temp | 温度（K） |
| 8 | flags | 标志位 |
| 9 | tmp | 临时数据1 |
| 10 | tmp2 | 临时数据2 |
| 11 | tmp3 | 临时数据3 / pavg0 |
| 12 | tmp4 | 临时数据4 / pavg1 |
| 13 | dcolour | 装饰颜色 (0xRRGGBB) |

#### sim.partPosition -- 读/写粒子位置

```lua
local id = sim.partCreate(-1, 300, 200, "WATR")

-- 读取位置
local x, y = sim.partPosition(id)
tpt.log("粒子坐标: (" .. x .. ", " .. y .. ")")

-- 瞬移粒子到新位置
sim.partPosition(id, 500, 300)
tpt.log("已移动到 (500, 300)")
```

#### sim.partChangeType -- 改变粒子类型

```lua
local id = sim.partCreate(-1, 300, 200, "WATR")
tpt.log("创建了水粒子")

sim.partChangeType(id, "FIRE")
tpt.log("水变成火了！")
-- 注意：类型改变后原粒子ID不变，坐标和其他属性保留
```

#### sim.partID -- 查询坐标上的粒子

```lua
local id = sim.partID(306, 192)   -- 查询画布中心的粒子
if id then
    local typeVal = sim.partProperty(id, 0)
    local temp = sim.partProperty(id, 7)
    tpt.log(string.format("中心有粒子: type=%d, temp=%.1fK", typeVal, temp))
else
    tpt.log("中心没有粒子")
end
```

### 5.2 地图查询

#### sim.pmap / sim.photons -- 底层粒子查询

```lua
-- sim.pmap(x, y) 返回 pmap 编码值（不包含光子）
local pmapVal = sim.pmap(300, 200)
if pmapVal then
    -- pmap编码: 高16位=类型, 低16位=粒子索引
    local typeVal = math.floor(pmapVal / 65536)  -- 或 bit.rshift(pmapVal, 16)
    local index = pmapVal % 65536                 -- 或 bit.band(pmapVal, 0xFFFF)
    tpt.log(string.format("pmap: type=%d, index=%d", typeVal, index))
end

-- sim.photons(x, y) 单独查询光子
local photonId = sim.photons(300, 200)
if photonId then
    tpt.log("此处有光子, ID: " .. photonId)
end
```

#### sim.partNeighbors -- 获取周围粒子

```lua
-- 获取(300, 200)周围半径5格内的所有粒子
local neighbors = sim.partNeighbors(300, 200, 5)
tpt.log("半径5格内有 " .. #neighbors .. " 个粒子")

-- 只获取火焰粒子
local fires = sim.partNeighbors(300, 200, 10, "FIRE")
tpt.log("半径10格内有 " .. #fires .. " 个火焰")

-- 遍历并处理邻居
for _, neighborId in ipairs(neighbors) do
    local t = sim.partProperty(neighborId, 0)
    local temp = sim.partProperty(neighborId, 7)
    if temp > 500 then
        sim.partKill(neighborId)  -- 清除所有高温粒子
    end
end
```

#### sim.parts() -- 遍历所有粒子

```lua
-- 统计所有元素类型
local counts = {}
for i in sim.parts() do
    local t = sim.partProperty(i, 0)
    counts[t] = (counts[t] or 0) + 1
end

for t, count in pairs(counts) do
    tpt.log("Type " .. t .. ": " .. count .. " 个")
end

-- 将所有温度超1000K的粒子转为LAVA
for i in sim.parts() do
    if sim.partProperty(i, 7) > 1000 then
        sim.partChangeType(i, "LAVA")
    end
end
tpt.log("高温粒子已转为熔岩")
```

### 5.3 地图场数据

#### 压力场 -- sim.pressure

```lua
-- 读取某点压力
local p = sim.pressure(300, 200)
tpt.log("(300,200) 压力: " .. p)

-- 设置某点压力
sim.pressure(300, 200, 10)         -- 设该点压力为10

-- 区域设置（左上角100x50区域压力归零）
sim.pressure(0, 0, 100, 50, 0)
```

#### 环境热场 -- sim.ambientHeat

```lua
-- 读取环境温度
local h = sim.ambientHeat(150, 100)
tpt.log("环境热: " .. h)

-- 整个画布环境热设为300K（温暖环境）
sim.ambientHeat(0, 0, XRES, YRES, 300)

-- 创建"冷区"——左上角区域环境热200K
sim.ambientHeat(0, 0, 50, 50, 200)
```

#### 空气速度场

```lua
-- 读取空气速度
local vx = sim.velocityX(150, 100)
local vy = sim.velocityY(150, 100)
tpt.log(string.format("空气速度: (%.2f, %.2f)", vx, vy))

-- 在区域产生向右的风
sim.velocityX(0, 0, 612, 100, 5)    -- 上部区域X速度=5

-- 产生向上的风
sim.velocityY(0, 0, 612, 384, -2)    -- 全域Y速度=-2（上升）
```

#### 墙壁场 -- sim.wallMap

```lua
-- 创建一段墙壁
sim.wallMap(100, 100, 30, 5, 1)     -- 30宽x5高的墙壁

-- 使用常量（WL_xxx系列）
local WL_WALLELEC = 3                -- 导电墙
sim.wallMap(200, 50, 10, 10, WL_WALLELEC)

-- 查询某处墙类型
local w = sim.wallMap(150, 102)
if w then
    tpt.log("此处有墙壁，类型: " .. w)
else
    tpt.log("此处无墙壁")
end
```

### 5.4 事件系统

#### 键盘事件

```lua
-- 空格键暂停/恢复
event.register(evt.KEYPRESS, function(key, scan, repeat, shift, ctrl, alt)
    if key == 32 and not repeat then    -- 32 = 空格键, non-repeat
        sim.paused(not sim.paused())
        tpt.log(sim.paused() and "已暂停" or "已恢复")
    end
end)

-- Ctrl+R 重置模拟
event.register(evt.KEYPRESS, function(key, scan, repeat, shift, ctrl, alt)
    if key == 114 and ctrl then         -- 114 = R键, Ctrl按下
        sim.clearSim()
        tpt.log("模拟已重置")
    end
end)

-- S键保存快照
event.register(evt.KEYPRESS, function(key, scan, repeat, shift, ctrl, alt)
    if key == 115 and not repeat then   -- 115 = S键
        sim.takeSnapshot()
        tpt.log("快照已保存")
    end
end)
```

**常用按键码：**

| 键 | 码 | 键 | 码 |
|----|-----|----|-----|
| 空格 | 32 | A-Z | 97-122 |
| 回车 | 13 | 0-9 | 48-57 |
| ESC | 27 | 方向键 | 上273/下274/左276/右275 |
| Tab | 9 | Shift/Ctrl/Alt | 参数中的布尔值 |

#### 鼠标事件

```lua
-- 右键点击检测粒子信息
event.register(evt.MOUSEDOWN, function(x, y, button)
    if button == 3 then    -- 1=左键 2=中键 3=右键
        local id = sim.partID(x, y)
        if id then
            local t = sim.partProperty(id, 0)
            local temp = sim.partProperty(id, 7)
            tpt.log(string.format("类型:%d 温度:%.1fK", t, temp))
        end
    end
end)

-- 左键点击创建水粒子
event.register(evt.MOUSEDOWN, function(x, y, button)
    if button == 1 then
        sim.partCreate(-1, x, y, "WATR")
    end
end)

-- 中键滚轮缩放
event.register(evt.MOUSEWHEEL, function(x, y, d)
    -- d > 0 = 向上滚（放大），d < 0 = 向下滚（缩小）
    ren.zoomWindow(x, y, 1 + d * 0.1)
end)
```

#### TICK事件（每帧执行）

```lua
-- 每秒报告一次粒子数
local frameCount = 0

event.register(evt.TICK, function()
    frameCount = frameCount + 1
    if frameCount % 60 == 0 then   -- TPT默认60FPS, 60帧=1秒
        tpt.log("粒子总数: " .. sim.partCount())
    end
end)
```

#### 模拟前后事件

```lua
-- 模拟前准备
event.register(evt.BEFORESIM, function()
    -- 在这里修改模拟参数
end)

-- 模拟后处理
event.register(evt.AFTERSIM, function()
    -- 分析模拟结果
end)

-- 渲染前绘制
event.register(evt.BEFORESIMDRAW, function()
    -- 在粒子绘制前叠加图形
end)

-- 渲染后绘制（覆盖在粒子上方）
event.register(evt.AFTERSIMDRAW, function()
    -- 在粒子绘制后叠加图形
end)
```

**重要**：使用 `evt.TICK` 和 `evt.AFTERSIMDRAW` 绘制屏幕覆盖层。

### 5.5 屏幕绘制

必须在事件回调中使用（如 `evt.TICK` 或 `evt.AFTERSIMDRAW`）。

```lua
-- 在屏幕左上角显示信息
event.register(evt.AFTERSIMDRAW, function()
    -- 半透明黑色背景
    graphics.fillRect(5, 5, 250, 75, 0, 0, 0, 180)

    -- 白色边框
    graphics.drawRect(5, 5, 250, 75, 255, 255, 255, 255)

    -- 文字内容
    graphics.drawText(15, 10,
        "粒子总数: " .. sim.partCount(),
        255, 255, 255, 255)

    -- 温度读数（鼠标位置）
    local mx, my = ui.mousePosition()
    if mx and my then
        local temp = sim.ambientHeat(mx, my)
        graphics.drawText(15, 30,
            string.format("环境温度: %.1fK (%.1f℃)", temp, temp - 273.15),
            255, 200, 100, 255)
    end

    -- 分隔线
    graphics.drawLine(15, 55, 240, 55, 255, 255, 255, 100)

    -- FPS
    local fps = sim.frameRender()
    graphics.drawText(15, 60,
        "FPS: " .. fps,
        0, 255, 0, 255)
end)
```

**绘制函数一览：**

| 函数 | 用途 |
|------|------|
| `graphics.drawText(x, y, text, r, g, b, a)` | 绘制文字 |
| `graphics.drawLine(x1, y1, x2, y2, r, g, b, a)` | 绘制线条 |
| `graphics.drawRect(x, y, w, h, r, g, b, a)` | 绘制矩形边框 |
| `graphics.fillRect(x, y, w, h, r, g, b, a)` | 填充矩形 |
| `graphics.drawCircle(x, y, rx, ry, r, g, b, a)` | 绘制圆边框 |
| `graphics.fillCircle(x, y, rx, ry, r, g, b, a)` | 填充圆 |
| `graphics.textSize(text)` | 获取文字宽高（返回 w, h） |

**颜色参数说明：**
- r, g, b: 0-255
- a: 0-255（透明度，0=完全透明，255=完全不透明）

### 5.6 输出与交互

```lua
-- 日志输出
tpt.log("普通日志消息")
tpt.log(string.format("格式化: %s %d", "hello", 42))

-- 系统消息（显示在屏幕中央）
tpt.sysmsg("模拟已完成！")

-- 控制台信息（同tpt.log，不同实现）
tpt.log("这条消息出现在控制台")

-- 获取鼠标位置
local mx, my = ui.mousePosition()
if mx then
    -- mx, my 是相对于绘制区的坐标
    tpt.log(string.format("鼠标: (%d, %d)", mx, my))
end
```

**注意：** TPT 中没有 `tpt.alert` 函数。如需弹出对话框，使用 UI 组件：

```lua
-- 消息框
interface.beginMessageBox("标题", "这是消息内容", false, function()
    tpt.log("用户关闭了消息框")
end)

-- 确认框
interface.beginConfirm("确认操作", "确定要清空模拟吗？", "确定", function(result)
    if result then
        sim.clearSim()
        tpt.log("模拟已清空")
    end
end)
```

### 5.7 自定义元素

#### 分配新元素

```lua
-- 分配一个新元素槽位
local myElem = elements.allocate("POWDERS", "MYEL")
--  参数1: 组名（决定菜单分类）: POWDERS/LIQUIDS/GASES/SOLIDS等
--  参数2: 唯一标识符（内部分配用）
--  返回: 新元素ID，失败返回-1

if myElem == -1 then
    tpt.log("元素分配失败！可能已满")
    return
end

-- 设置元素属性
elements.element(myElem, {
    Name = "MYEL",          -- 菜单显示名（最多4个字母）
    Colour = 0xFFFF8040,    -- ARGB颜色
    MenuVisible = true,     -- 在菜单中可见
    MenuSection = SC_POWDERS,  -- 分类: SC_POWDERS = 粉末栏
    Enabled = true,
    Description = "自定义粉末元素",

    -- 物理属性
    Advection = 0.7,        -- 惯性系数
    AirDrag = 0.04,         -- 空气阻力
    AirLoss = 0.95,         -- 空气能量损失
    Loss = 0.95,            -- 速度损失
    Collision = 0.0,        -- 碰撞弹性
    Gravity = 0.3,          -- 重力
    Diffusion = 0.0,        -- 扩散
    HotAir = 0.0,           -- 热气上升
    Falldown = 1,           -- 下落行为
    Flammable = 0,          -- 可燃性
    Explosive = 0,          -- 爆炸性
    Meltable = 0,           -- 可熔化
    Hardness = 1,           -- 硬度
    Weight = 15,            -- 重量
    HeatConduct = 50,       -- 热导率

    -- 状态转换
    HighTemperature = 1000,
    HighTemperatureTransition = 11,   -- 超1000K转为FIRE (Type 11)

    Properties = TYPE_PART,           -- 类型：粉末
})

tpt.log("自定义元素 MYEL 创建成功！ID: " .. myElem)
```

#### 添加每帧更新逻辑

```lua
local myElem = elements.allocate("POWDERS", "GLWP")
elements.element(myElem, {
    Name = "GLWP",
    Colour = 0xFFFFAA00,
    MenuVisible = true,
    MenuSection = SC_POWDERS,
    Enabled = true,
    Description = "随时间发光的粉末",
    Advection = 0.7, Gravity = 0.3,
    Weight = 15, HeatConduct = 50,
    HighTemperature = 1000,
    HighTemperatureTransition = 11,   -- FIRE
    Properties = TYPE_PART or PROP_LIFE or PROP_LIFE_DEC,

    -- Update回调（每帧调用）
    Update = function(i, x, y, surround_space, nt)
        local life = sim.partProperty(i, "life")
        sim.partProperty(i, "life", life + 1)    -- 生命递增

        if life > 300 then
            sim.partChangeType(i, "PHOT")          -- 300帧后变光子
            return 1                               -- 返回1表示有变化
        end
        return 0                                   -- 0表示无变化
    end,

    -- Graphics回调（控制渲染外观）
    Graphics = function(i, colr, colg, colb)
        local life = sim.partProperty(i, "life")
        local brightness = math.min(255, life)
        return 0, PMODE_GLOW, 255,
            brightness, brightness * 0.7, 0,  -- 颜色RGB
            100, brightness * 0.5, brightness * 0.3, 0  -- 发光效果
    end
})

tpt.log("自定义发光粉末 GLWP 创建成功！")
```

#### 元素属性读写

```lua
-- 读取已有元素属性
local watrId = elements.getByName("WATR")
local weight = elements.property(watrId, "Weight")
local heatCond = elements.property(watrId, "HeatConduct")
tpt.log(string.format("WATR: 重量=%d, 导热=%d", weight, heatCond))

-- 修改已有元素属性（注意：影响所有此类型粒子）
elements.property(watrId, "HeatConduct", 200)   -- 提高水的导热性
elements.property(watrId, "Diffusion", 1.0)     -- 增加扩散

-- 重置为默认
elements.loadDefault(watrId)                     -- 恢复WATR默认值
elements.loadDefault()                           -- 恢复所有元素默认值
```

---

## 6. 常用脚本配方（复制即用）

以下脚本可直接复制粘贴到脚本管理器中运行。每个脚本都有注释说明如何修改。

### 6.1 自动保存定时器

```lua
-- ============================================================
-- 自动保存印章：每N秒自动保存整个画布
-- 修改 SAVE_INTERVAL 参数调整间隔（秒）
-- ============================================================

local SAVE_INTERVAL = 30    -- 自动保存间隔（秒），改为60=每1分钟
local ENABLE_LOG = true     -- 是否输出日志

local lastSave = 0

event.register(evt.TICK, function()
    local now = socket.getTime()
    if now - lastSave >= SAVE_INTERVAL then
        local stampName = sim.saveStamp(0, 0, XRES, YRES, true)
        if ENABLE_LOG then
            tpt.log("[自动保存] " .. stampName .. "  时间: " ..
                os.date("%H:%M:%S", math.floor(now)))
        end
        lastSave = now
    end
end)

tpt.log("自动保存已启动，间隔: " .. SAVE_INTERVAL .. " 秒")
tpt.log("保存文件位于 stamps/ 目录")
```

### 6.2 清除所有指定类型粒子

```lua
-- ============================================================
-- 清除所有指定类型的粒子
-- 修改 TARGET_TYPE 为目标元素名
-- 按 Ctrl+K 执行清除
-- ============================================================

local TARGET_TYPE = "FIRE"    -- 改为你想要清除的元素名

local function clearAllTarget()
    local count = 0
    for i in sim.parts() do
        local t = sim.partProperty(i, 0)
        if t == elements.getByName(TARGET_TYPE) then
            sim.partKill(i)
            count = count + 1
        end
    end
    tpt.log(string.format("已清除 %d 个 %s 粒子", count, TARGET_TYPE))
    tpt.sysmsg(string.format("清除了 %d 个 %s", count, TARGET_TYPE))
end

-- 按 Ctrl+K 执行清除
event.register(evt.KEYPRESS, function(key, scan, repeat, shift, ctrl, alt)
    if key == 107 and ctrl and not repeat then   -- K键
        clearAllTarget()
    end
end)

-- 也可以直接在控制台中输入 clearAllTarget() 来手动调用
tpt.log("按 Ctrl+K 清除所有 " .. TARGET_TYPE .. " 粒子")
```

### 6.3 温度显示叠加层

```lua
-- ============================================================
-- 实时温度叠加层
-- 在屏幕上显示画布中心区域的粒子温度和热力图指示
-- ============================================================

event.register(evt.AFTERSIMDRAW, function()
    local mx, my = ui.mousePosition()
    if not mx then return end

    -- 鼠标位置粒子温度
    local id = sim.partID(mx, my)
    local tempK = 0
    local hasParticle = false

    if id then
        tempK = sim.partProperty(id, 7)
        hasParticle = true
    else
        tempK = sim.ambientHeat(mx, my)
    end

    local tempC = tempK - 273.15

    -- 温度颜色映射
    local r, g, b = 255, 255, 255
    if tempC < 0 then
        r, g, b = 100, 150, 255     -- 冷色（蓝色）
    elseif tempC < 100 then
        r, g, b = 255, 255, 255     -- 常温（白色）
    elseif tempC < 500 then
        r, g, b = 255, 200, 50      -- 温热（橙色）
    elseif tempC < 1000 then
        r, g, b = 255, 100, 30      -- 高温（红橙）
    else
        r, g, b = 255, 255, 100     -- 极高温（亮黄）
    end

    -- 信息面板背景
    local panelW, panelH = 220, 55
    local px, py = mx + 15, my - 60
    if px + panelW > graphics.WIDTH then px = mx - panelW - 15 end
    if py < 0 then py = my + 15 end

    graphics.fillRect(px, py, panelW, panelH, 0, 0, 0, 200)
    graphics.drawRect(px, py, panelW, panelH, r, g, b, 200)

    graphics.drawText(px + 8, py + 5,
        string.format("温度: %.1fK / %.1f℃", tempK, tempC),
        r, g, b, 255)

    graphics.drawText(px + 8, py + 25,
        string.format("环境: %.1fK | 压力: %.1f",
            sim.ambientHeat(mx, my),
            sim.pressure(mx, my)),
        200, 200, 200, 255)
end)
```

### 6.4 简单粒子动画

```lua
-- ============================================================
-- 粒子动画：在画布上绘制一个旋转的水粒子环
-- ============================================================

local centerX, centerY = 306, 192     -- 旋转中心
local radius = 40                      -- 旋转半径
local speed = 0.05                     -- 旋转速度（弧度/帧）
local angle = 0                        -- 当前角度
local particleCount = 12               -- 环上粒子数

event.register(evt.TICK, function()
    -- 清除上一次位置的粒子
    local oldNeighbors = sim.partNeighbors(centerX, centerY, radius + 10, "WATR")
    for _, id in ipairs(oldNeighbors) do
        sim.partKill(id)
    end

    -- 在新位置创建粒子
    angle = angle + speed
    for i = 0, particleCount - 1 do
        local a = angle + (i / particleCount) * math.pi * 2
        local x = centerX + math.cos(a) * radius
        local y = centerY + math.sin(a) * radius
        sim.partCreate(-1, x, y, "WATR")
    end

    -- 在中心放一个不同颜色的粒子作为标记
    sim.partCreate(-1, centerX, centerY, "FIRE")
end)

tpt.log("粒子环动画已启动！")
```

### 6.5 自动连点器（Auto-Clicker Tool）

```lua
-- ============================================================
-- 自动画笔：按住左键时持续在当前画笔位置创建粒子
-- 适用条件：当前选中任意画笔工具（非特殊工具）
-- 按 Ctrl+A 切换自动模式开关
-- ============================================================

local autoEnabled = false
local mouseDown = false

-- 切换开关
event.register(evt.KEYPRESS, function(key, scan, repeat, shift, ctrl, alt)
    if key == 97 and ctrl and not repeat then   -- Ctrl+A
        autoEnabled = not autoEnabled
        tpt.log("自动画笔: " .. (autoEnabled and "开启" or "关闭"))
        tpt.sysmsg(autoEnabled and "自动画笔 ON" or "自动画笔 OFF")
    end
end)

-- 检测鼠标按下/释放
event.register(evt.MOUSEDOWN, function(x, y, button)
    if button == 1 then mouseDown = true end
end)

event.register(evt.MOUSEUP, function(x, y, button, reason)
    if button == 1 then mouseDown = false end
end)

-- 每帧自动创建粒子
event.register(evt.TICK, function()
    if autoEnabled and mouseDown then
        local mx, my = ui.mousePosition()
        if mx and my then
            local brushId = ui.brushID()
            if brushId then
                sim.partCreate(-1, mx, my, brushId)
            end
        end
    end
end)

tpt.log("自动画笔已加载，按 Ctrl+A 开关")
tpt.log("使用方法：开自动模式 -> 选画笔元素 -> 按住左键拖动")
```

### 6.6 粒子统计面板

```lua
-- ============================================================
-- 粒子统计面板：在屏幕右上角显示元素数量排名
-- ============================================================

event.register(evt.AFTERSIMDRAW, function()
    -- 每60帧更新一次以减少性能开销
    if sim.frameRender() % 60 ~= 0 then return end

    local counts = {}
    for i in sim.parts() do
        local t = sim.partProperty(i, 0)
        counts[t] = (counts[t] or 0) + 1
    end

    -- 排序取前10
    local sorted = {}
    for t, count in pairs(counts) do
        table.insert(sorted, {type = t, count = count})
    end
    table.sort(sorted, function(a, b) return a.count > b.count end)

    local topN = math.min(10, #sorted)

    -- 绘制面板
    local px, py = graphics.WIDTH - 200, 5
    local panelH = topN * 16 + 25
    graphics.fillRect(px, py, 195, panelH, 0, 0, 0, 180)
    graphics.drawRect(px, py, 195, panelH, 100, 100, 100, 200)

    graphics.drawText(px + 5, py + 5,
        "元素排名 (总数: " .. sim.partCount() .. ")",
        255, 255, 200, 255)

    for i = 1, topN do
        local color = elements.property(sorted[i].type, "Colour")
        -- 从ARGB提取RGB
        local r = math.floor(color / 65536) % 256
        local g = math.floor(color / 256) % 256
        local b = color % 256

        graphics.drawText(px + 10, py + 10 + i * 15,
            string.format("#%d  Type %d: %d", i, sorted[i].type, sorted[i].count),
            r, g, b, 255)
    end
end)
```

---

## 7. 安装和运行社区脚本

### 获取社区脚本

社区Lua脚本主要来源：

- **TPT官方论坛**：https://powdertoy.co.uk/Discussions/
- **TPT存档分享站**：https://powdertoy.co.uk/Browse/
- **GitHub仓库**：搜索 "tpt-lua" 或 "powdertoy scripts"
- **Discord社区**：TPT官方Discord的 `#scripts` 频道

### 安装脚本

**方法一：放入scripts目录**

1. 下载 `.lua` 脚本文件
2. 将其复制到TPT的 `scripts/` 目录
3. 打开TPT -> Script Manager -> Open -> 选择该文件 -> Run

```
TPT安装目录/
  └── scripts/
        └── 你的脚本.lua   ← 放这里
```

**方法二：拖拽运行**

部分TPT版本支持直接将 `.lua` 文件拖到TPT窗口上自动运行。

**方法三：控制台加载**

```lua
-- 在控制台中运行指定的Lua文件
dofile("scripts/community_script.lua")
```

### 脚本兼容性注意事项

- 检查脚本要求的TPT版本（`tpt.version()` 可查询）
- 部分旧脚本使用了已废弃的API，需要手动修改
- 如果脚本报错，对比附录D检查函数名是否变更
- 主要兼容性陷阱：
  - `tpt.alert` 已被 `tpt.log` 替代
  - API别名（如 `simulation` vs `sim`）可能在某些版本不可用
  - Window API在较新版本中才有完整支持

### 推荐入门脚本

以下是一些适合初学者的社区脚本类型（按难度递增）：

| 难度 | 脚本类型 | 学习内容 |
|------|---------|---------|
| 容易 | 粒子计数器 | `sim.parts()` 迭代器、表操作 |
| 容易 | 定时保存器 | `evt.TICK`、`socket.getTime` |
| 中等 | 屏幕HUD叠加 | `graphics.*` 绘制函数 |
| 中等 | 按键绑定工具 | `evt.KEYPRESS`、`sim.*` 粒子操作 |
| 较难 | 自定义元素 | `elements.allocate`、Update/Graphics回调 |
| 困难 | 完整UI窗口 | `ui.*` 组件系统、Window回调 |

---

## 8. 调试技巧与常见错误

### 8.1 调试技巧

#### 使用 tpt.log 进行print调试

```lua
-- 最基础的调试方法：在关键位置打印信息
local function debugExample(id)
    tpt.log("=== 开始调试 ===")
    if not sim.partExists(id) then
        tpt.log("错误：粒子 " .. id .. " 不存在")
        return
    end
    local temp = sim.partProperty(id, 7)
    tpt.log("粒子温度: " .. temp .. "K")
    tpt.log("=== 调试结束 ===")
end
```

#### 检查变量类型

```lua
-- 使用 type() 函数检查变量类型
local val = sim.partID(300, 200)
tpt.log("type = " .. type(val))     -- "number" 或 "nil"

val = {1, 2, 3}
tpt.log("type = " .. type(val))     -- "table"

val = sim.parts
tpt.log("type = " .. type(val))     -- "function"
```

#### 使用 pcall 捕获错误

```lua
-- pcall 可以在出错时不让脚本崩溃
local function riskyOperation()
    -- 可能会出错的代码
    local id = sim.partCreate(-1, -100, -200, "WATR")  -- 越界坐标
    return id
end

local ok, result = pcall(riskyOperation)
if ok then
    tpt.log("操作成功: " .. tostring(result))
else
    tpt.log("操作失败: " .. tostring(result))
    -- result 包含错误信息
end
```

#### 条件断点

```lua
-- 在特定条件满足时才输出调试信息
local debugEnabled = true
local frameCount = 0

event.register(evt.TICK, function()
    frameCount = frameCount + 1

    -- 仅当粒子数超过阈值时报告
    if debugEnabled and frameCount % 300 == 0 then
        local count = sim.partCount()
        if count > 10000 then
            tpt.log("警告：粒子数过高 " .. count)
        end
    end
end)
```

### 8.2 常见错误及解决

#### 错误1：attempt to call a nil value

```lua
-- 原因：函数不存在或拼写错误
-- 错误示例：
tpt.alert("hello")    -- 错误！没有 tpt.alert 函数

-- 正确写法：
tpt.log("hello")      -- 用 tpt.log
```

#### 错误2：attempt to index a nil value

```lua
-- 原因：尝试访问不存在的对象
-- 错误示例：
local id = sim.partID(100, 100)
local temp = sim.partProperty(id, 7)   -- 如果 id 是 nil，这里崩溃

-- 正确写法：先检查
local id = sim.partID(100, 100)
if id then
    local temp = sim.partProperty(id, 7)
    tpt.log("温度: " .. temp)
else
    tpt.log("此处无粒子")
end
```

#### 错误3：坐标越界

```lua
-- 原因：坐标超出画布范围
-- X: 0-612, Y: 0-384
-- CELL坐标: X: 0-153, Y: 0-96

-- 错误示例：
sim.partCreate(-1, 700, 500, "WATR")  -- 坐标超出范围！

-- 正确写法：先限制坐标
local function safeCreate(x, y, elem)
    x = math.max(0, math.min(XRES - 1, x))
    y = math.max(0, math.min(YRES - 1, y))
    return sim.partCreate(-1, x, y, elem)
end
```

#### 错误4：混淆像素坐标与CELL坐标

```lua
-- 重要概念：
-- 像素坐标（pixel）: sim.partCreate 使用此坐标，范围 0-612, 0-384
-- CELL坐标:         一些内部函数使用，范围 0-153, 0-96
-- 换算: 1 CELL = 4 像素

-- 错误示例：把CELL坐标传给需要像素坐标的函数
local cx, cy = 50, 30     -- CELL坐标
sim.partCreate(-1, cx, cy, "WATR")    -- 错误！应该 × 4

-- 正确示例：
sim.partCreate(-1, cx * 4, cy * 4, "WATR")  -- CELL转像素
-- 或者直接用像素坐标：
sim.partCreate(-1, 200, 120, "WATR")
```

#### 错误5：事件重复注册

```lua
-- 原因：多次运行脚本导致同一个事件被注册多次

-- 问题示例：每次Run都会叠加一个新的事件处理器
event.register(evt.KEYPRESS, myHandler)

-- 解决方案1：检查脚本是否已加载（用全局变量标记）
if not scriptLoaded_myScript then
    scriptLoaded_myScript = true
    event.register(evt.KEYPRESS, myHandler)
end

-- 解决方案2：使用Reload代替重复Run
-- 在脚本管理器中点击 Reload 按钮

-- 解决方案3：使用 Stop 停止旧脚本再 Run 新脚本
```

#### 错误6：忘记 local 导致全局污染

```lua
-- 错误示例：函数内忘记 local
function badFunc()
    count = 0          -- 忘记local，变成全局变量
    for i = 1, 10 do
        count = count + i
    end
end

-- 正确写法：
function goodFunc()
    local count = 0    -- 局部变量
    for i = 1, 10 do
        count = count + i
    end
    return count
end
```

#### 错误7：数组索引从1开始

```lua
-- Lua 的表数组索引从 1 开始（不是 0！）
local arr = {"a", "b", "c"}
tpt.log(arr[1])    -- "a"（不是 arr[0]）
tpt.log(arr[0])    -- nil

-- for 循环遍历
for i = 1, #arr do
    tpt.log(i .. ": " .. arr[i])
end
```

#### 错误8：性能问题——每帧操作不当

```lua
-- 错误示例：每帧遍历所有粒子并做昂贵操作
event.register(evt.TICK, function()
    for i in sim.parts() do
        -- 如果粒子数上万，这个循环会导致严重掉帧
        if sim.partProperty(i, 7) > 500 then
            -- 每帧都创建粒子会严重影响性能
            sim.partCreate(-1, 100, 100, "WATR")
        end
    end
end)

-- 正确做法：降低频率
local frameCounter = 0
event.register(evt.TICK, function()
    frameCounter = frameCounter + 1
    if frameCounter % 300 == 0 then  -- 每5秒执行一次
        local count = 0
        for i in sim.parts() do
            count = count + 1
            if count > 1000 then break end  -- 限制遍历数量
        end
        tpt.log("抽样统计: " .. count)
    end
end)
```

### 8.3 错误排查清单

遇到问题时按以下顺序排查：

1. **拼写错误**：检查函数名和变量名是否与API文档一致
2. **nil检查**：对可能返回nil的函数调用后先检查结果
3. **坐标范围**：X在0-611，Y在0-383（粒子函数）；CELL在0-152/0-95
4. **类型匹配**：该传数字的地方不要传字符串，反之亦然
5. **事件清理**：重启TPT清除所有已注册事件，避免残留
6. **语法检查**：检查 `end` 是否配对，`then` 后是否有代码
7. **查看日志**：错误信息通常很明确，仔细阅读报错
8. **逐步注释**：注释掉部分代码，逐步缩小错误范围

---

## 9. 进阶学习路线

完成本指南后，建议按以下顺序深入学习：

### 第一步：巩固基础
- 运行本章第6节的所有示例脚本
- 修改参数观察效果变化
- 尝试组合两个脚本来实现新功能

### 第二步：学习API参考
- **附录D**：完整的 Lua API 参考（所有函数、参数、返回值）
- **附录P**：粒子标志位与状态常量（位运算操作粒子标志）

### 第三步：系统级操作
- **附录I**：管道系统详解（用脚本控制PIPE/PPIP）
- **附录J**：活塞框架系统（用脚本控制PSTN/FRME）
- **附录G**：电路构建指南（理解模拟逻辑后再用脚本增强）

### 第四步：自定义元素
- 参考社区中的复杂自定义元素脚本
- 学习 Update/Graphics/ChangeType 等回调的配合使用
- 尝试创建有独特行为的完整自定义元素

### 第五步：完整项目
- 用 ui.Window 构建完整的交互式工具面板
- 实现自动化实验流程（创建→记录→分析）
- 将多个脚本整合为一个功能齐全的工具集

### 补充资源

| 资源 | 链接 | 说明 |
|------|------|------|
| TPT官方Wiki（Lua） | https://powdertoy.co.uk/Wiki/Lua.html | 官方Lua文档 |
| Lua 5.2参考手册 | https://www.lua.org/manual/5.2/ | Lua语言本身 |
| TPT API参考（本套文档） | 附录D | 完整API列表 |
| TPT官方论坛 | https://powdertoy.co.uk/Discussions/ | 提问和查找脚本 |
| LuaJIT扩展 | https://luajit.org/extensions.html | TPT使用的Lua变体 |

---

## 交叉参考

| 附录 | 内容 | 与本指南的关系 |
|------|------|---------------|
| **附录D** | Lua API 完整参考 | 本指南的补充——所有函数详细签名和参数 |
| **附录G** | 电路构建指南 | Lua 脚本控制电路的接口 |
| **附录H** | 调试模式指南 | 配合 D 键调试模式排查脚本效果 |
| **附录I** | 管道系统详解 | ``sim.partProperty`` 操作 PIPE 元素的 tmp 字段 |
| **附录J** | 活塞框架系统 | Lua 控制 PSTN/FRME 的参数 |
| **附录K** | 种子遗传系统 | Lua 操作 SEED/PLNT 基因数据 |
| **附录M** | 温度与压力阈值 | Lua 脚本中温度和压力操作的参考范围 |
| **附录N** | CanMove 交互规则 | ``sim.canMove()`` 函数的使用上下文 |
| **附录O** | FILT/LAVA 参考 | Lua 中操作 FILT 波长和 LAVA 属性 |
| **附录P** | 标志位与状态常量 | Lua 位运算的常量定义（PROP_xxx, FLAG_xxx 等） |
| **附录Q** | Subframe 子帧技术 | Lua 脚本在子帧级别的高级控制 |
