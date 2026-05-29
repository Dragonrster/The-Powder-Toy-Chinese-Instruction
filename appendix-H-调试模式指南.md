# 附录H：调试模式指南

按 `D` 键开启调试模式。再按一次切换到不同调试视图。

---

## 调试视图模式

| 按键 | 模式 | 显示内容 | 使用场景 |
|------|------|---------|---------|
| D(1次) | DEBUG_PARTS | 粒子Type和Life值 | 快速识别粒子类型和生命值 |
| D(2次) | DEBUG_ELEMENTPOP | 每类元素数量统计 | 统计各元素粒子分布 |
| D(3次) | DEBUG_LINES | WIFI频道连线(同频道WiFi间连线) | 追踪WIFI信号路径 |
| D(4次) | DEBUG_PARTICLE | 粒子详细信息(Type/Temp/Life/Ctype/Tmp等) | 深入调试单个粒子 |
| D(5次) | DEBUG_SIMHUD | 模拟器FPS和性能信息 | 性能优化分析 |
| D(6次) | 关闭调试 | 返回正常视图 | — |

### 各调试视图详细解读

#### DEBUG_PARTS（第1次）
每个粒子显示两行信息：
- **上行**：粒子Type编号和英文名缩写
- **下行**：life值（如适用）
- **颜色编码**：液体=蓝、气体=绿、固体=灰、粉末=黄、能量=红

#### DEBUG_ELEMENTPOP（第2次）
- 屏幕左侧列出所有元素的实时粒子计数
- 可快速定位意外增多的粒子（例如无限繁殖bug）
- 显示格式：`元素名: 数量`，按数量降序排列

#### DEBUG_LINES（第3次）
- WIFI粒子间绘制彩色连线
- **同频道**=相同颜色连线
- **不同频道**=不连线
- 可直观看到WIFI网络的拓扑结构
- ARAY天线也会显示信号辐射方向

#### DEBUG_PARTICLE（第4次）
鼠标悬停粒子显示完整信息面板：
```
Type: [类型名](编号)    Ctype: [携带类型]
X/Y: [坐标]            Life: [生命值]
Temp: [温度K/℃]       Tmp: [临时值1]
Tmp2: [临时值2]        Flags: [标志位十六进制]
Vx/Vy: [速度矢量]      Pressure: [压力值]
```

#### DEBUG_SIMHUD（第5次）
性能统计显示：
```
FPS: [帧率]            Draw: [绘制耗时ms]
Frame: [帧号]          Sim: [模拟耗时ms]
Parts: [粒子数]/[最大]  Air: [空气更新耗时ms]
Wall: [墙壁粒子数]      Tick: [总帧耗时ms]
```

---

## HUD显示

按 `H` 键切换HUD：

| 显示 | 说明 | 精度/单位 |
|------|------|-----------|
| FPS | 帧率(60为正常速度) | 整数 |
| 粒子数 | 当前活动粒子数/最大粒子数 | 整数/整数 |
| 压力 | 鼠标位置压力值(pv) | 浮点, 0.01精度 |
| 温度 | 鼠标位置粒子温度(K/℃) | 浮点, 2位小数 |
| 速度 | 鼠标位置粒子速度矢量(Vx,Vy) | 浮点 |
| 空气速度 | 鼠标位置空气流速 | 浮点 |
| 环境温度 | 鼠标位置环境热(hv) | 浮点 |
| 墙壁类型 | 鼠标位置墙壁类型(WL_xxx) | 字符串 |
| 元素类型 | 鼠标位置元素Type和英文名 | 字符串 |
| 重力 | 鼠标位置重力矢量(Gx,Gy) | 浮点 |

### HUD高级使用技巧
- **HUD + 暂停**：暂停模拟后再查看HUD，数据不会变化
- **HUD + 印章保存**：查看特定位置数值后用印章保存该区域
- **多次按H**：H键循环切换不同HUD组合（无HUD→基本→详细→完整）

---

## 调试渲染

| 按键 | 功能 | 详细说明 |
|------|------|---------|
| W | 切换重力模式(垂直/关闭/径向/自定义) | 在4种重力模式间循环 |
| Y | 切换空气模式 | 开启/关闭压力/关闭速度/全关/不更新 |
| P | 截图 | 保存到TPT目录，文件名含时间戳 |
| ` | 打开Lua控制台 | 位于~键(ESC下方) |
| Ctrl+P | 性能测试模式 | 显示每帧各阶段耗时 |
| Ctrl+D | 详细诊断 | 显示模拟器内部状态变量 |

---

## PROP工具

选中元素后，右键元素图标进入属性编辑器，可直接修改：
- 元素名称、颜色、菜单分类
- 所有物理参数(Advection/Gravity/Weight等)
- 状态转换阈值
- 特殊属性标志位
- 自定义回调函数(Lua脚本)

### 属性编辑器详细字段

```
基本信息：
├── Name: 元素名称（修改后立即更新菜单显示）
├── Description: 元素描述文本
├── Colour: ARGB颜色值（0xAARRGGBB格式）
├── MenuVisible: 是否在菜单中可见
└── MenuSection: 菜单分类(SC_WALL=0 到 SC_DECO=15)

物理参数：
├── Advection: 惯性系数（粒子保持速度的程度）
├── AirDrag: 空气阻力（×0.04）
├── AirLoss: 空气能量损失率（0-1）
├── Loss: 速度损失率（每帧乘以1-Loss）
├── Collision: 碰撞弹性（1=完全弹性, 0=完全非弹性）
├── Gravity: 重力影响系数（0=不受重力）
├── Diffusion: 扩散系数（0=不扩散）
├── HotAir: 热气上升速率
├── Falldown: 下落行为(0=不下落,1=下落,2=液体下落)
├── Flammable: 可燃性(0=不可燃)
├── Explosive: 爆炸性(0=不爆炸)
├── Meltable: 可熔化(0=不可熔化)
├── Hardness: 硬度(1-1000,用于ACID腐蚀)
├── Weight: 重量(1-100,用于CanMove判断)
├── HeatConduct: 热传导率(0-255)
└── PhotonReflectWavelengths: 光子反射波长掩码

状态转换：
├── LowPressure: 低压阈值
├── LowPressureTransition: 低压转换目标
├── HighPressure: 高压阈值
├── HighPressureTransition: 高压转换目标
├── LowTemperature: 低温阈值
├── LowTemperatureTransition: 低温转换目标
├── HighTemperature: 高温阈值
└── HighTemperatureTransition: 高温转换目标

属性标志：
└── Properties: 按位组合的标志位(见附录P)
```

### 属性修改注意事项
- 修改后**立即生效**(内存中)，重启游戏恢复默认
- 可用`elements.loadDefault(id)`在Lua中重置
- 修改会保留到Lua脚本中（若通过脚本设置）
- 如需永久修改，将属性设置写入autorun.lua

---

## Lua控制台

按 ` 键打开控制台。控制台是调试最强大的工具。

### 命令模式

| 前缀 | 模式 | 示例 |
|------|------|------|
| 无前缀 | Lua语句直接执行 | `sim.partCount()` |
| `!` | 脚本管理器命令 | `!ls` 列出脚本 |
| `=` | 简洁求值表达式 | `=sim.pressure(100,100)` |
| `?` | 帮助查询 | `?sim.partCreate` |

### 常用控制台命令速查

```lua
-- 粒子查询
=sim.partCount()                    -- 总粒子数
=sim.elementCount("FIRE")           -- FIRE元素数量
=sim.partID(100, 100)               -- (100,100)处粒子ID
=sim.partProperty(id, 7)            -- 粒子温度

-- 粒子创建与删除
=sim.partCreate(-1, 200, 150, "WATR")  -- 在水中创建
=sim.partKill(id)                      -- 删除指定粒子
sim.clearSim()                         -- 清空整个模拟

-- 批量操作
sim.floodParts(100, 100, "FIRE", 0)     -- 泛洪填充火焰
sim.createBox(0, 0, 153, 96, "WATR")    -- 全屏水

-- 地图数据查询
=sim.pressure(150, 100)              -- 查看压力
=sim.ambientHeat(150, 100)           -- 查看环境热
=sim.wallMap(150, 100)               -- 查看墙壁类型

-- 模拟控制
sim.paused(true)                     -- 暂停
sim.paused(false)                    -- 恢复
sim.frameRender(10)                  -- 渲染10帧

-- 属性修改
e = elements.getByName("FIRE")
elements.property(e, "HeatConduct", 0)   -- 关闭FIRE导热
elements.property(e, "Flammable", 10000) -- 极高可燃性

-- 元素查询
=elements.getByName("WATR")          -- 获取WATR的ID
=elements.getByName("LIFE")          -- 获取LIFE的ID

-- 工具操作
=ui.brushID()                        -- 当前画刷元素
ui.brushID(elements.getByName("LAVA"))  -- 切换画刷为LAVA
=ui.brushRadius()                    -- 当前画刷半径
ui.brushRadius(10, 10)               -- 设置画刷半径

-- 渲染
=ren.renderMode()                    -- 当前渲染模式
ren.grid(10)                         -- 显示网格
ren.hud(true)                        -- 显示HUD

-- 截图和记录
sim.takeSnapshot()                   -- 历史快照
ren.screenshot()                     -- 截图(等同于按P)
```

### 调试技巧：诊断性能问题

```lua
-- 在控制台中检查性能瓶颈
local start = socket.getTime()
for i = 1, 1000 do
    sim.floodParts(50, 50, "WATR", 0)
end
tpt.log("1000 floodParts耗时: " .. (socket.getTime() - start) .. "秒")

-- 找出最多的元素类型
local counts = {}
for i in sim.parts() do
    local t = sim.partProperty(i, 0)
    counts[t] = (counts[t] or 0) + 1
end
for t, c in pairs(counts) do
    tpt.log(t .. ": " .. c)
end
```

---

## 高级调试技术

### 热力图可视化
通过Lua创建热力图脚本，在`TICK`事件中绘制：
```lua
-- 温度热力图（放到autorun.lua中）
local function drawHeatmap()
    for x = 0, XRES, CELL do
        for y = 0, YRES, CELL do
            local temp = sim.ambientHeat(x, y)
            local r = math.min(255, math.max(0, (temp - 273.15) * 2))
            local b = math.min(255, math.max(0, (573.15 - temp)))
            graphics.fillRect(x, y, CELL, CELL, r, 0, b, 100)
        end
    end
end
event.register(evt.TICK, drawHeatmap)
```

### 粒子追踪
追踪特定粒子的移动轨迹：
```lua
local tracked = -1
local points = {}

-- 设置追踪粒子
function trackParticle(id) tracked = id; points = {} end

-- 在TICK中调用
local function updateTrack()
    if tracked > 0 and sim.partExists(tracked) then
        local x, y = sim.partPosition(tracked)
        table.insert(points, {x, y, socket.getTime()})
        if #points > 500 then table.remove(points, 1) end
    end
end

-- 绘制轨迹
local function drawTrack()
    for i = 2, #points do
        graphics.drawLine(points[i-1][1], points[i-1][2],
            points[i][1], points[i][2], 255, 0, 0, 255)
    end
end
```

### 墙壁与电力可视化
```lua
-- 显示墙壁热图
function showWalls()
    local wallColors = {
        [0] = {0,0,0,0},       -- WL_NONE
        [1] = {255,100,100,100}, -- WL_WALL
        [2] = {100,100,255,100}, -- WL_FAN
        [3] = {255,255,100,100}, -- WL_DETECT
        [4] = {100,255,100,100}, -- WL_EWALL
        -- ... 其他墙壁类型
    }
    -- 遍历并绘制
end

-- 显示电力场
function showElectric()
    for x = 0, XRES, CELL do
        for y = 0, YRES, CELL do
            local emap = sim.elecMap(x, y)
            if emap > 0 then
                graphics.fillRect(x, y, CELL, CELL, 255, 255, 0, emap * 10)
            end
        end
    end
end
```

---

## 常见调试场景

### 场景1：排查内存/性能问题
1. 按D两次进入DEBUG_ELEMENTPOP视图
2. 观察是否有元素粒子数异常增长
3. 用控制台`=sim.elementCount("xxx")`精确定位
4. 用`sim.clearRect()`删除问题区域

### 场景2：追踪WIFI信号故障
1. 按D三次进入DEBUG_LINES视图
2. 检查同频道WiFi是否有连线中断
3. 确认频道号是否正确（按H看温度→频道转换）
4. 检查是否有INVS阻挡了信号路径

### 场景3：调试电信号电路
1. 按D一次识别所有PSCN/NSCN/导体粒子
2. 用控制台设置断点`sim.paused(true)`步进观察
3. 检查SPRK粒子的life值（电信号持续时间）
4. 验证INST/PTCT/NTCT的开关状态

### 场景4：温度/压力异常
1. 按H启用HUD查看温度和压力
2. 暂停后逐帧观察温度扩散
3. 检查是否有意外的HEAC/COOL元素干扰
4. 用PROP工具查看元素的HeatConduct设置

---

## 交互参考

- 参考**附录D**：Lua API完整参考（控制台使用的所有函数）
- 参考**附录M**：温度与压力阈值（理解HUD读数含义）
- 参考**附录P**：粒子标志位与状态（理解DEBUG_PARTICLE中的Flags字段）
