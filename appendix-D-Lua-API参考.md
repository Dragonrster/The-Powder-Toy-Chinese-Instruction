# 附录D：Lua API 参考

TPT 内置完整的 Lua 脚本引擎，可通过脚本控制模拟、创建自定义UI、编写自动化脚本等。

---

## 快速入口

| 表名 | 别名 | 用途 |
|------|------|------|
| `sim` | `simulation` | 粒子创建/修改、模拟控制、地图数据 |
| `tpt` | — | 版本信息、截图、录制、日志 |
| `elements` | `elem` | 元素属性读写、自定义元素 |
| `event` | `evt` | 键盘/鼠标事件注册 |
| `graphics` | `gfx` | 屏幕绘制文字、线条、图形 |
| `renderer` | `ren` | 渲染模式、显示设置 |
| `interface` | `ui` | 对话框、UI组件(按钮/滑块/文本框等) |
| `platform` | `plat` | 系统平台、剪贴板、打开链接 |
| `fileSystem` | `fs` | 文件/目录操作 |
| `http` | — | HTTP请求 |
| `socket` | — | 网络socket、时间 |
| `bz2` | — | bzip2压缩/解压 |
| `bit` | — | 位运算(Lua BitOp) |
| `tools` | — | 自定义工具 |

---

## sim / simulation -- 模拟控制

### 粒子操作

#### sim.partCreate(newID, x, y, type [, v])
创建粒子。
- **newID**: -1自动分配ID，或指定ID(该ID不存在时)
- **x, y**: 像素坐标(不是CELL坐标)
- **type**: 数字ID或字符串名(如"WATR"或数字3)
- **v**: 可选，初始速度(0=默认)
- **返回**: 粒子ID，失败返回-1

```lua
-- 创建粒子示例
local id = sim.partCreate(-1, 100, 100, "WATR")  -- 用水名字符串
local id2 = sim.partCreate(-1, 200, 150, 3)      -- 用数字ID(WATR=3)
local id3 = sim.partCreate(-1, 300, 200, "FIRE", 0) -- 带速度参数

-- 错误处理
if id == -1 then
    tpt.log("粒子创建失败：坐标超出范围或达到粒子上限")
end
```

#### sim.partKill(partID [, deathType])
删除粒子。
- **partID**: 粒子ID
- **deathType**: 可选，死亡类型(影响死亡时产生的副产品)
- **返回**: 无

```lua
sim.partKill(id)           -- 默认死亡
sim.partKill(id, 1)        -- 带死亡类型1(可能产生不同效果)
```

#### sim.partChangeType(partID, newType)
改变粒子类型。保留粒子的坐标和部分属性。
- **partID**: 粒子ID
- **newType**: 数字ID或字符串名
- **返回**: 无

```lua
sim.partChangeType(id, "FIRE")  -- 将粒子转为火焰
sim.partChangeType(id, 3)       -- 转为WATR(ID=3)
```

#### sim.partProperty(partID, fieldID [, value])
读/写粒子属性。2个参数=读取，3个参数=写入。
- **partID**: 粒子ID
- **fieldID**: 字段索引(0-13)或字符串名
- **value**: 可选，要设置的值
- **返回**: (读取模式)当前值；(写入模式)无

```lua
-- 读取属性
local temp = sim.partProperty(id, 7)      -- 用数字索引读温度
local life = sim.partProperty(id, "life") -- 用字符串名读生命值

-- 写入属性
sim.partProperty(id, 7, 500)              -- 设温度为500K
sim.partProperty(id, "temp", 500)         -- 同上，用字符串名
sim.partProperty(id, 2, 3)                -- 设ctype为3(WATR)
```

#### sim.partPosition(partID [, x, y])
读/写粒子位置。
- **返回**: (读取)x, y；(写入)无

```lua
local x, y = sim.partPosition(id)     -- 读取位置
sim.partPosition(id, 200, 150)        -- 设置新位置
```

#### sim.partID(x, y)
返回(x,y)处粒子ID。
- **返回**: 粒子ID，无粒子则nil

```lua
local id = sim.partID(100, 100)
if id then
    tpt.log("该处粒子类型: " .. sim.partProperty(id, 0))
else
    tpt.log("该处无粒子")
end
```

#### sim.partExists(partID)
检查粒子是否存在。
- **返回**: boolean

```lua
if sim.partExists(id) then
    -- 粒子仍然存在
else
    tpt.log("粒子已消失")
end
```

#### sim.partNeighbors(x, y, r [, type])
返回半径r内粒子ID列表。
- **x, y**: 中心坐标
- **r**: 半径(格数)
- **type**: 可选，仅返回此类型的粒子
- **返回**: 粒子ID列表(table)

```lua
local neighbors = sim.partNeighbors(100, 100, 5, "FIRE")
tpt.log("5格内有 " .. #neighbors .. " 个火焰粒子")
for _, id in ipairs(neighbors) do
    sim.partKill(id)  -- 清除所有附近的火焰
end
```

#### sim.pmap(x, y)
返回(x,y)处pmap编码值。用于获取粒子ID（不包含光子）。
- **返回**: pmap编码值(包含Type和Index)

```lua
local pmapValue = sim.pmap(100, 100)
if pmapValue then
    local typeVal = bit.rshift(pmapValue, 16)
    tpt.log("元素类型: " .. typeVal)
end
```

#### sim.photons(x, y)
返回(x,y)处光子ID。
- **返回**: 光子ID，无光子则nil

---

### 粒子字段索引

| 索引 | 字段名(字符串) | 说明 | Lua类型 |
|------|---------------|------|---------|
| 0 | "type" | 粒子类型 | number |
| 1 | "life" | 生命值 | number |
| 2 | "ctype" | 携带类型 | number |
| 3 | "x" | X坐标(像素) | number |
| 4 | "y" | Y坐标(像素) | number |
| 5 | "vx" | X速度 | number |
| 6 | "vy" | Y速度 | number |
| 7 | "temp" | 温度(K) | number |
| 8 | "flags" | 标志位 | number |
| 9 | "tmp" | 临时数据1 | number |
| 10 | "tmp2" | 临时数据2 | number |
| 11 | "tmp3" | 临时数据3 / pavg0 | number |
| 12 | "tmp4" | 临时数据4 / pavg1 | number |
| 13 | "dcolour" | 装饰颜色(0xRRGGBB) | number |

**注意**：`sim.partProperty(id, "tmp3")`和`sim.partProperty(id, 11)`是等效的。

辅助函数：
- `TYP(n)`: 从合并ID提取类型。等价于`bit.rshift(n, 16)`
- `ID(n)`: 从合并ID提取索引。等价于`bit.band(n, 0xFFFF)`

---

### 迭代器

#### for i in sim.parts() do ... end
遍历所有活动粒子。
```lua
-- 统计所有元素
local counts = {}
for i in sim.parts() do
    local t = sim.partProperty(i, 0)
    counts[t] = (counts[t] or 0) + 1
end

-- 将所有高温粒子转为LAVA
for i in sim.parts() do
    if sim.partProperty(i, 7) > 1000 then
        sim.partChangeType(i, "LAVA")
    end
end
```

#### for id,x,y in sim.neighbors(x,y,rx,ry) do ... end
遍历范围内邻居粒子。比`sim.partNeighbors`效率高(迭代器模式)。
```lua
-- 计算区域内平均温度
local totalTemp, count = 0, 0
for id, x, y in sim.neighbors(100, 100, 10, 10) do
    totalTemp = totalTemp + sim.partProperty(id, 7)
    count = count + 1
end
if count > 0 then
    tpt.log("平均温度: " .. (totalTemp / count) .. "K")
end
```

---

### 批量创建

| 函数 | 用途 | 参数 |
|------|------|------|
| `sim.createParts(x, y, rx, ry, c [, flags])` | 区域创建 | 中心坐标,半径x,半径y,元素,标志 |
| `sim.createLine(x1, y1, x2, y2, rx, ry, c)` | 画线创建 | 起点,终点,半宽x,半宽y,元素 |
| `sim.createBox(x1, y1, x2, y2, c)` | 矩形创建 | 左上角,右下角,元素 |
| `sim.floodParts(x, y, c, cm, flags)` | 泛洪填充 | 起点坐标,元素,条件模式,标志 |

```lua
-- 创建5x5水方块
sim.createParts(100, 100, 2, 2, "WATR")

-- 从左到右画一条墙壁线
sim.createLine(0, 192, 612, 192, 1, 1, "WALL")

-- 在左上角创建矩形金属块
sim.createBox(10, 10, 50, 50, "METL")

-- 泛洪填充(从100,100开始，填充所有空位为水)
sim.floodParts(100, 100, "WATR", 0)  -- cm=0:仅填充空位
```

---

### 地图数据 getter/setter

以区域读写用法:`sim.pressure(x, y [, w, h, value])`
- **无value**：读取(x,y)处单格值
- **有value但无w,h**：设置(x,y)处单格值
- **有value且有w,h**：区域赋值(w,h为区域尺寸)

| 函数 | 数据 | 说明 |
|------|------|------|
| `sim.pressure(x,y,[w,h,v])` | 压力场(pv) | 空气压力值 |
| `sim.ambientHeat(x,y,[w,h,v])` | 环境热(hv) | 环境温度场 |
| `sim.velocityX(x,y,[w,h,v])` | X速度(vx) | 空气X方向速度 |
| `sim.velocityY(x,y,[w,h,v])` | Y速度(vy) | 空气Y方向速度 |
| `sim.wallMap(x,y,[w,h,v])` | 墙壁(bmap) | 墙壁类型(WL_xxx) |
| `sim.elecMap(x,y,[w,h,v])` | 电力(emap) | 电信号强度 |
| `sim.gravityMass(x,y,[w,h,v])` | 重力质量 | 用于牛顿重力模式 |
| `sim.gravityMask(x,y,[w,h,v])` | 重力掩码 | 重力影响区域 |
| `sim.fanVelocityX(x,y,[w,h,v])` | 风扇X速 | 风扇产生的X速度 |
| `sim.fanVelocityY(x,y,[w,h,v])` | 风扇Y速 | 风扇产生的Y速度 |

```lua
-- 读取
local p = sim.pressure(150, 100)
local h = sim.ambientHeat(150, 100)

-- 设置单格
sim.pressure(150, 100, 10)           -- 设置此格压力为10

-- 设置区域
sim.ambientHeat(0, 0, 153, 96, 300)  -- 整个画布环境热设为300K

-- 生成墙壁区域
sim.wallMap(50, 50, 20, 10, 1)       -- 20x10的墙壁区域
```

---

### 模拟状态

| 函数 | 用途 | 参数/返回值 |
|------|------|------------|
| `sim.paused([bool])` | 暂停/恢复 | true=暂停, false=恢复, 无参=查询 |
| `sim.clearSim()` | 清空模拟 | 删除所有粒子和地图数据 |
| `sim.clearRect(x,y,w,h)` | 清空矩形区域 | 删除矩形内粒子 |
| `sim.resetTemp()` | 重置温度场 | 恢复到环境温度 |
| `sim.resetPressure()` | 重置压力场 | 恢复到0 |
| `sim.resetVelocity()` | 重置速度场 | 空气速度归零 |
| `sim.resetSpark()` | 重置电力场 | 清除电信号 |
| `sim.partCount()` | 总粒子数 | 返回number |
| `sim.elementCount(type)` | 某类型粒子数 | type可以是数字或字符串 |
| `sim.canMove(moving, into [, setting])` | can_move关系 | 见附录N |
| `sim.frameRender([frames])` | 待渲染帧数 | 返回/设置渲染帧数 |
| `sim.takeSnapshot()` | 历史快照 | 创建可回退的快照点 |
| `sim.historyRestore()` | 历史后退 | 恢复到上一个快照 |
| `sim.historyForward()` | 历史前进 | 前进到下一个快照 |

```lua
-- 暂停/恢复切换
sim.paused(not sim.paused())

-- 检查粒子数
local total = sim.partCount()
local fireCount = sim.elementCount("FIRE")
tpt.log(string.format("总粒子:%d 火焰:%d", total, fireCount))

-- 帧控制
sim.frameRender(100)  -- 渲染100帧后自动暂停

-- 快照
sim.takeSnapshot()     -- 保存当前状态
-- ... 做变化 ...
sim.historyRestore()   -- 回退
```

---

### 模拟设置

| 函数 | 用途 | 参数 |
|------|------|------|
| `sim.edgeMode([mode])` | 边界模式 | EDGE_VOID/SOLID/LOOP |
| `sim.gravityMode([mode])` | 重力模式 | GRAV_VERTICAL/OFF/RADIAL/CUSTOM |
| `sim.customGravity([x, y])` | 自定义重力 | x,y方向矢量 |
| `sim.airMode([mode])` | 空气模式 | AIR_ON/PRESSUREOFF/VELOCITYOFF/OFF/NOUPDATE |
| `sim.ambientAirTemp([temp])` | 环境温度 | 温度(K) |
| `sim.waterEqualization([bool])` | 水平衡 | true/false |
| `sim.newtonianGravity([bool])` | 牛顿重力 | true/false |
| `sim.convectionMode([mode])` | 对流模式 | 0/1 |

```lua
-- 零重力环境
sim.gravityMode(GRAV_OFF)

-- 循环边界
sim.edgeMode(EDGE_LOOP)

-- 自定义重力(向右下方)
sim.gravityMode(GRAV_CUSTOM)
sim.customGravity(0.5, 1.0)

-- 无空气模拟
sim.airMode(AIR_OFF)

-- 调整环境温度
sim.ambientAirTemp(200)  -- -73℃ 寒冷环境
```

---

### 印章

| 函数 | 用途 |
|------|------|
| `sim.saveStamp(x,y,w,h,includePressure)` | 保存印章，返回名称 |
| `sim.loadStamp(name,x,y [, hflip, rotation, pressure])` | 加载印章 |
| `sim.deleteStamp(name)` | 删除印章 |
| `sim.listStamps()` | 列出所有印章名称 |

```lua
-- 保存
local name = sim.saveStamp(50, 50, 100, 50, true)
tpt.log("印章已保存: " .. name)

-- 加载(可水平翻转、旋转、保持压力)
sim.loadStamp(name, 200, 100, false, 0, false)

-- 水平翻转加载
sim.loadStamp(name, 200, 100, true, 0, false)
```

---

## elements / elem -- 元素管理

### 元素生命周期

| 函数 | 用途 | 参数/返回 |
|------|------|----------|
| `elements.allocate(group, name)` | 分配新元素 | 组名,元素名, 返回ID或-1 |
| `elements.free(id)` | 释放自定义元素 | 元素ID |
| `elements.exists(id)` | 检查元素是否存在 | 元素ID, 返回boolean |

### 元素属性

| 函数 | 用途 |
|------|------|
| `elements.element(id [, table])` | 读/写全部属性(通过table) |
| `elements.property(id, propName [, value])` | 读/写单个属性 |
| `elements.loadDefault([id])` | 重置为默认值(无id=重置全部) |
| `elements.getByName(identifier)` | 通过名称查找元素ID |

```lua
-- 获取元素ID
local watrId = elements.getByName("WATR")
local fireId = elements.getByName("FIRE")

-- 读取单个属性
local weight = elements.property(watrId, "Weight")
tpt.log("水的重量: " .. weight)

-- 修改单个属性
elements.property(fireId, "HeatConduct", 200)  -- 提高导热性

-- 修改多个属性(通过table)
elements.element(fireId, {
    HeatConduct = 250,
    Diffusion = 1.5,
    HotAir = 0.001
})

-- 重置
elements.loadDefault(fireId)  -- 重置FIRE
elements.loadDefault()        -- 重置所有元素

-- 分配新元素
local newId = elements.allocate("POWDERS", "MYELEM")
if newId == -1 then
    tpt.log("元素分配失败")
else
    -- 设置属性
    elements.element(newId, {
        Name = "MYEL",
        Colour = 0xFF8040,
        MenuVisible = true,
        MenuSection = 8,  -- SC_POWDERS
        Advection = 0.7,
        Gravity = 0.3,
        Weight = 10
    })
end
```

---

### 可读写元素属性完整列表

| 属性名 | 类型 | 说明 |
|--------|------|------|
| Name | string | 元素名称(4字母限制) |
| Colour | number | ARGB颜色(0xAARRGGBB) |
| MenuVisible | boolean | 菜单中可见 |
| MenuSection | number | 菜单分类(0-15,见附录P) |
| Enabled | boolean | 是否启用 |
| Description | string | 描述文本 |
| Advection | number | 惯性系数(0-1) |
| AirDrag | number | 空气阻力(×0.04) |
| AirLoss | number | 空气能量损失(0-1) |
| Loss | number | 速度损失(每帧×1-Loss) |
| Collision | number | 碰撞弹性(0-1) |
| Gravity | number | 重力系数(0=无, 1=正常) |
| Diffusion | number | 扩散系数 |
| HotAir | number | 热气上升速率 |
| Falldown | number | 下落行为(0/1/2) |
| Flammable | number | 可燃性(0=不可燃) |
| Explosive | number | 爆炸性(0=不爆炸) |
| Meltable | number | 可熔化(0=不可) |
| Hardness | number | 硬度(1-1000) |
| Weight | number | 重量(1-100) |
| HeatConduct | number | 热传导率(0-255) |
| PhotonReflectWavelengths | number | 光子反射掩码 |
| Properties | number | 属性标志位(按位OR,见附录P) |
| LowPressure | number | 低压阈值 |
| LowPressureTransition | number | 低压转换目标 |
| HighPressure | number | 高压阈值 |
| HighPressureTransition | number | 高压转换目标 |
| LowTemperature | number | 低温阈值(K) |
| LowTemperatureTransition | number | 低温转换目标 |
| HighTemperature | number | 高温阈值(K) |
| HighTemperatureTransition | number | 高温转换目标 |
| CarriesTypeIn | number | 携带类型(少数元素使用) |

---

### 元素属性常量

#### 元素类型(TYPE_xxx)

| 常量 | 值 | 说明 |
|------|-----|------|
| TYPE_PART | 1 | 粉末 |
| TYPE_LIQUID | 2 | 液体 |
| TYPE_SOLID | 4 | 固体 |
| TYPE_GAS | 8 | 气体 |
| TYPE_ENERGY | 16 | 能量 |

#### 属性位(PROP_xxx)

| 常量 | 位值 | 说明 |
|------|------|------|
| PROP_CONDUCTS | 32 | 导电 |
| PROP_PHOTPASS | 2048 | 光子穿透 |
| PROP_NEUTPENETRATE | 4096 | 中子穿透+反应 |
| PROP_NEUTABSORB | 8192 | 中子吸收 |
| PROP_NEUTPASS | 16384 | 中子通过 |
| PROP_DEADLY | 32768 | 致命 |
| PROP_HOT_GLOW | 65536 | 高温发光 |
| PROP_LIFE | 131072 | 有生命值 |
| PROP_RADIOACTIVE | 262144 | 放射性 |
| PROP_LIFE_DEC | 524288 | 生命衰减 |
| PROP_LIFE_KILL | 1048576 | 生命耗尽自杀 |
| PROP_LIFE_KILL_DEC | 2097152 | 生命耗尽衰减 |
| PROP_SPARKSETTLE | 4194304 | 火花沉降 |
| PROP_NOAMBHEAT | 8388608 | 无环境热交换 |
| PROP_NOCTYPEDRAW | 16777216 | 不绘制Ctype |

#### 菜单分类(SC_xxx)

| 常量 | 值 | 名称 |
|------|-----|------|
| SC_WALL | 0 | 墙壁 |
| SC_ELEC | 1 | 电子元件 |
| SC_POWERED | 2 | 电动元件 |
| SC_SENSOR | 3 | 传感器 |
| SC_FORCE | 4 | 力场 |
| SC_EXPLOSIVE | 5 | 爆炸物 |
| SC_GAS | 6 | 气体 |
| SC_LIQUID | 7 | 液体 |
| SC_POWDERS | 8 | 粉末 |
| SC_SOLIDS | 9 | 固体 |
| SC_NUCLEAR | 10 | 核材料 |
| SC_SPECIAL | 11 | 特殊 |
| SC_LIFE | 12 | 生命 |
| SC_TOOL | 13 | 工具 |
| SC_DECO | 15 | 装饰 |

---

### 元素回调 (Lua函数)

自定义元素可以设置以下回调函数。回调通过`elements.element(id, {Update = func, ...})`设置。

#### Update 回调
```lua
-- function(i, x, y, surround_space, nt) return changed end
-- i: 粒子索引, x,y: 坐标, surround_space: 周围状态, nt: 转换类型
-- 返回: 0(无变化) 或 1(有变化,需要重新渲染)

elements.element(myId, {
    Update = function(i, x, y, surround_space, nt)
        -- 每帧更新逻辑
        local life = sim.partProperty(i, 1)
        if life > 100 then
            sim.partChangeType(i, "FIRE")
            return 1
        end
        return 0
    end
})
```

#### Graphics 回调
```lua
-- function(i, colr, colg, colb) return cache, pixel_mode, cola, colr, colg, colb, firea, firer, fireg, fireb end

elements.element(myId, {
    Graphics = function(i, colr, colg, colb)
        -- 自定义渲染
        local temp = sim.partProperty(i, 7)
        local r = math.min(255, (temp - 273) * 2)
        return 0, PMODE_GLOW, 255, r, 50, 255, 0, 0, 0, 0
    end
})
```

#### Create 回调
```lua
-- function(i, x, y, t, v)
-- 粒子被创建时调用
elements.element(myId, {
    Create = function(i, x, y, t, v)
        tpt.log(string.format("粒子创建于(%d,%d), 类型:%d", x, y, t))
    end
})
```

#### CreateAllowed 回调
```lua
-- function(i, x, y, t) return allowed end
-- 检查是否允许创建，返回true允许/false阻止
elements.element(myId, {
    CreateAllowed = function(i, x, y, t)
        -- 禁止在屏幕边缘创建
        if x < 10 or x > XRES - 10 then return false end
        return true
    end
})
```

#### ChangeType 回调
```lua
-- function(i, x, y, from, to)
-- 粒子类型改变时调用
elements.element(myId, {
    ChangeType = function(i, x, y, from, to)
        tpt.log(string.format("类型变化: %d → %d", from, to))
    end
})
```

#### CtypeDraw 回调
```lua
-- function(i, t, v) return draw end
-- 控制是否绘制Ctype(如LAVA在Ctype为ROCK时的美观渲染)
elements.element(myId, {
    CtypeDraw = function(i, t, v)
        return true  -- 允许Ctype绘制
    end
})
```

---

### 自定义元素完整示例

```lua
-- 创建一个"发光粉末"元素
local glowPowder = elements.allocate("POWDERS", "GLWP")
if glowPowder == -1 then
    tpt.log("元素分配失败")
    return
end

elements.element(glowPowder, {
    Name = "GLWP",
    Colour = 0xFFFFAA00,
    MenuVisible = true,
    MenuSection = SC_POWDERS,
    Enabled = true,
    Description = "Glowing powder that gets brighter over time",

    -- 物理属性
    Advection = 0.7,
    AirDrag = 0.04,
    AirLoss = 0.95,
    Loss = 0.95,
    Collision = 0.0,
    Gravity = 0.3,
    Diffusion = 0.0,
    HotAir = 0.0,
    Falldown = 1,
    Flammable = 0,
    Explosive = 0,
    Meltable = 0,
    Hardness = 1,
    Weight = 15,
    HeatConduct = 50,
    PhotonReflectWavelengths = 0,
    Properties = TYPE_PART,

    -- 状态转换
    LowPressure = IPL,
    LowPressureTransition = NT,
    HighPressure = IPH,
    HighPressureTransition = NT,
    LowTemperature = ITL,
    LowTemperatureTransition = NT,
    HighTemperature = 1000,
    HighTemperatureTransition = 11,  -- 转FIRE

    -- 自定义Update
    Update = function(i, x, y, surround_space, nt)
        local life = sim.partProperty(i, 1)
        sim.partProperty(i, 1, life + 1)  -- 生命递增
        if life > 200 then
            sim.partChangeType(i, "PHOT") -- 变光子
            return 1
        end
        return 0
    end,

    -- 自定义Graphics
    Graphics = function(i, colr, colg, colb)
        local life = sim.partProperty(i, 1)
        local brightness = math.min(255, life)
        return 0, PMODE_GLOW, 255, brightness, brightness * 0.7, 0,
            100, brightness * 0.5, brightness * 0.3, 0
    end
})
```

---

## event / evt -- 事件系统

### 事件注册

```lua
event.register(eventType, func)        -- 注册
event.unregister(eventType, func)      -- 注销
```

### 完整事件列表

| 事件常量 | 说明 | 回调参数 | 触发时机 |
|----------|------|---------|---------|
| `KEYPRESS` | 按键按下 | (key, scan, repeat, shift, ctrl, alt) | 键盘按下 |
| `KEYRELEASE` | 按键释放 | (key, scan, repeat, shift, ctrl, alt) | 键盘释放 |
| `MOUSEDOWN` | 鼠标按下 | (x, y, button) | 鼠标按钮按下 |
| `MOUSEUP` | 鼠标释放 | (x, y, button, reason) | 鼠标按钮释放 |
| `MOUSEMOVE` | 鼠标移动 | (x, y, dx, dy) | 鼠标移动 |
| `MOUSEWHEEL` | 鼠标滚轮 | (x, y, d) | 滚轮滚动 |
| `TICK` | 每帧触发 | 无参数 | 每帧 |
| `BEFORESIM` | 模拟前 | 无参数 | 模拟计算前 |
| `AFTERSIM` | 模拟后 | 无参数 | 模拟计算后 |
| `BEFORESIMDRAW` | 模拟绘制前 | 无参数 | 模拟渲染前 |
| `AFTERSIMDRAW` | 模拟绘制后 | 无参数 | 模拟渲染后 |
| `TEXTINPUT` | 文本输入 | (text) | 文本输入 |
| `BLUR` | 失焦 | 无参数 | 窗口失焦 |
| `CLOSE` | 关闭 | 无参数 | 窗口关闭 |

### 常用事件示例

```lua
-- TICK事件(每帧执行)
local tickCount = 0
event.register(evt.TICK, function()
    tickCount = tickCount + 1
    if tickCount % 60 == 0 then  -- 每60帧(1秒)
        tpt.log("粒子数: " .. sim.partCount())
    end
end)

-- 按键事件
event.register(evt.KEYPRESS, function(key, scan, repeat, shift, ctrl, alt)
    if key == 32 and not repeat then  -- 空格键(非重复)
        sim.paused(not sim.paused())
    end
    if key == 114 and ctrl then  -- Ctrl+R
        sim.clearSim()
        tpt.log("模拟已重置")
    end
end)

-- 鼠标事件
event.register(evt.MOUSEDOWN, function(x, y, button)
    if button == 3 then  -- 右键
        local id = sim.partID(x, y)
        if id then
            local t = sim.partProperty(id, 0)
            tpt.log("右键点击了元素: " .. t)
        end
    end
end)

-- 鼠标滚轮缩放
event.register(evt.MOUSEWHEEL, function(x, y, d)
    ren.zoomWindow(x, y, 1 + d * 0.1)
end)
```

---

## graphics / gfx -- 绘制

### 绘制函数

| 函数 | 用途 | 参数 |
|------|------|------|
| `gfx.drawText(x, y, text [, r, g, b, a])` | 绘制文字 | 坐标,文字,颜色(0-255) |
| `gfx.drawLine(x1, y1, x2, y2 [, r, g, b, a])` | 绘制线条 | 起点,终点,颜色 |
| `gfx.drawRect(x, y, w, h [, r, g, b, a])` | 绘制矩形边框 | 左上角,宽,高,颜色 |
| `gfx.fillRect(x, y, w, h [, r, g, b, a])` | 填充矩形 | 左上角,宽,高,颜色 |
| `gfx.drawCircle(x, y, rx, ry [, r, g, b, a])` | 绘制圆边框 | 圆心,半径x,半径y,颜色 |
| `gfx.fillCircle(x, y, rx, ry [, r, g, b, a])` | 填充圆 | 圆心,半径x,半径y,颜色 |
| `gfx.getColors(hexColor)` | 获取RGBA | 返回r,g,b,a |
| `gfx.getHexColor(r, g, b [, a])` | 获取Hex | 返回hex颜色值 |
| `gfx.textSize(text)` | 文字尺寸 | 返回w, h |

### 绘制API常量

- `gfx.WIDTH` -- 窗口宽度
- `gfx.HEIGHT` -- 窗口高度

```lua
-- 完整绘制示例(在TICK或自定义Window的onDraw中使用)
local function drawUI()
    -- 绘制半透明背景面板
    graphics.fillRect(10, 10, 200, 80, 0, 0, 0, 180)

    -- 绘制边框
    graphics.drawRect(10, 10, 200, 80, 255, 255, 255, 255)

    -- 绘制文字
    local fps = sim.frameRender()
    graphics.drawText(20, 20, "FPS: " .. fps, 255, 255, 0, 255)
    graphics.drawText(20, 40, "Particles: " .. sim.partCount(),
        255, 255, 255, 255)

    -- 绘制分隔线
    graphics.drawLine(20, 65, 195, 65, 255, 255, 255, 100)

    -- 绘制温度指示条
    local temp = sim.ambientHeat(100, 100)
    local barWidth = math.min(180, (temp - 273) * 2)
    graphics.fillRect(20, 75, barWidth, 8, 255, 100, 0, 200)
end
event.register(evt.TICK, drawUI)
```

---

## renderer / ren -- 渲染

### 渲染函数

| 函数 | 说明 |
|------|------|
| `ren.renderMode([mode])` | 渲染模式(PMODE_xxx) |
| `ren.displayMode([mode])` | 显示模式 |
| `ren.colorMode([mode])` | 颜色模式 |
| `ren.decorations([bool])` | 显示装饰 |
| `ren.grid([size])` | 网格大小(0=关闭) |
| `ren.fireSize([size])` | 火焰强度 |
| `ren.hud([bool])` | HUD显示 |
| `ren.debugHud([bool])` | 调试HUD |
| `ren.zoomWindow([x, y, factor])` | 缩放窗口 |
| `ren.zoomScope([x, y, size])` | 设置查看范围 |

```lua
-- 显示网格
ren.grid(10)       -- 10像素网格
ren.grid(0)        -- 关闭网格

-- 显示HUD
ren.hud(true)

-- 缩放窗口
ren.zoomWindow(306, 192, 2.0)   -- 以画布中心放大2倍
ren.zoomWindow(306, 192, 0.5)   -- 缩小到0.5倍
ren.zoomWindow(306, 192, 1.0)   -- 恢复默认
```

---

## interface / ui -- UI组件

### 对话框

| 函数 | 说明 | 回调参数 |
|------|------|---------|
| `ui.beginMessageBox(title, message, large, callback)` | 消息框 | 无 |
| `ui.beginThrowError(errorMessage, callback)` | 错误框 | 无 |
| `ui.beginInput(title, prompt, text, shadow, callback)` | 输入框 | text或nil |
| `ui.beginConfirm(title, message, buttonText, callback)` | 确认框 | boolean |

```lua
-- 消息框
ui.beginMessageBox("提示", "模拟已完成", false, function()
    tpt.log("用户关闭了消息框")
end)

-- 输入框
ui.beginInput("输入", "请输入名称:", "默认值", true, function(text)
    if text then
        tpt.log("用户输入: " .. text)
    else
        tpt.log("用户取消了输入")
    end
end)

-- 确认框
ui.beginConfirm("确认", "确定要清空模拟吗?", "确定", function(result)
    if result then
        sim.clearSim()
    end
end)
```

### 工具/画刷

| 函数 | 说明 |
|------|------|
| `ui.activeTool(index [, identifier])` | 获取/设置活动工具 |
| `ui.brushID([index])` | 获取/设置画刷元素 |
| `ui.brushRadius([x, y])` | 获取/设置画刷半径 |
| `ui.mousePosition()` | 获取鼠标位置(返回x,y) |

### UI组件构造

| 组件 | 构造函数 |
|------|---------|
| Button | `ui.button(x, y, w, h, text, tooltip)` |
| Label | `ui.label(x, y, w, h, text)` |
| Checkbox | `ui.checkbox(x, y, w, h, text)` |
| Textbox | `ui.textbox(x, y, w, h, text, placeholder)` |
| Slider | `ui.slider(x, y, w, h, steps)` |
| ProgressBar | `ui.progressBar(x, y, w, h, value, status)` |
| Window | `ui.window(x, y, w, h)` |

### 组件通用方法

```lua
local btn = ui.button(10, 10, 80, 24, "点击", "这是一个按钮")
btn:position(20, 20)     -- 移动位置
btn:size(100, 30)        -- 改变大小
btn:visible(false)       -- 隐藏
btn:visible(true)        -- 显示
btn:text("新文本")       -- 改变文本
btn:enabled(false)       -- 禁用
btn:enabled(true)        -- 启用
```

### 组件特有方法和事件

```lua
-- Button
local btn = ui.button(10, 10, 80, 24, "按钮", "提示")
btn:onMouseDown(function(x, y, button)
    tpt.log("按钮被按下")
end)

-- Checkbox
local cb = ui.checkbox(10, 10, 80, 24, "选项")
cb:checked(true)           -- 设置为选中
local isChecked = cb:checked()  -- 读取选中状态

-- Textbox
local tb = ui.textbox(10, 10, 200, 24, "默认文本", "占位符")
tb:onTextChanged(function(text)
    tpt.log("文本变为: " .. text)
end)

-- Slider
local sl = ui.slider(10, 10, 200, 24, 100)  -- 100步
sl:onValueChanged(function(value)
    tpt.log("滑块值: " .. value)
end)
sl:value(50)     -- 设置值
local v = sl:value()  -- 读取值

-- ProgressBar
local pb = ui.progressBar(10, 10, 200, 24, 50, "处理中...")
pb:value(75)          -- 更新进度
pb:status("完成")     -- 更新状态文本

-- Window
local win = ui.window(100, 100, 400, 300)
-- Window事件回调(直接赋值):
win.onInitialized = function() end
win.onExit = function() end
win.onTick = function(dt) end
win.onDraw = function()
    -- 在此绘制Window内容
    graphics.drawText(10, 10, "Windows Title", 255, 255, 255, 255)
end
win.onFocus = function() end
win.onBlur = function() end
win.onTryExit = function() return true end  -- 返回true允许退出
win.onTryOkay = function() return true end
win.onMouseMove = function(x, y, dx, dy) end
win.onMouseDown = function(x, y, button) end
win.onMouseUp = function(x, y, button) end
win.onMouseWheel = function(x, y, d) end
win.onKeyPress = function(key, scan, shift, ctrl, alt) end
win.onKeyRelease = function(key, scan, shift, ctrl, alt) end
```

---

## tools -- 自定义工具

### 工具管理

| 函数 | 用途 |
|------|------|
| `tools.allocate(group, name)` | 分配工具，返回索引 |
| `tools.free(index)` | 释放自定义工具 |
| `tools.exists(index)` | 检查工具是否存在 |
| `tools.isCustom(index)` | 是否自定义工具 |
| `tools.property(index, propName [, value])` | 读/写工具属性 |

### 工具回调

| 回调 | 说明 | 参数 |
|------|------|------|
| `Perform` | 执行(单击) | (index, x, y, brushX, brushY, strength) |
| `Click` | 点击 | (index, x, y, button) |
| `Drag` | 拖拽 | (index, x, y, dx, dy) |
| `Draw` | 绘制预览 | (index, x, y) |
| `DrawLine` | 画线 | (index, x1, y1, x2, y2) |
| `DrawRect` | 画矩形 | (index, x1, y1, x2, y2) |
| `DrawFill` | 填充 | (index, x1, y1, x2, y2) |
| `Select` | 选择 | (index, x, y) |

```lua
-- 自定义工具示例：单击创建5x5水方块
local toolIdx = tools.allocate("LIQUID", "WT5X")
tools.property(toolIdx, "Description", "创建5x5水方块")
tools.property(toolIdx, "Perform", function(idx, x, y, bx, by, strength)
    sim.createParts(x, y, 2, 2, "WATR")
end)
```

---

## tpt -- TPT全局

| 函数 | 说明 |
|------|------|
| `tpt.version()` | 返回版本信息(jit, repo, build, id, snapshot, tpt) |
| `tpt.log(message [, ...])` | 输出到日志 |
| `tpt.screenshot([x, y, w, h])` | 截图 |
| `tpt.startScriptRecord()` | 开始录制 |
| `tpt.stopScriptRecord()` | 停止录制 |
| `tpt.record(message)` | 录制中间帧 |
| `tpt.input(text)` | 模拟键盘输入 |
| `tpt.sysmsg(text)` | 系统消息 |
| `tpt.registerStep(handler)` | 注册步骤处理器 |
| `tpt.unregisterStep(handler)` | 注销步骤处理器 |

---

## platform / plat -- 系统平台

| 函数 | 说明 |
|------|------|
| `plat.platform()` | 返回平台名("Windows"/"Linux"/"MacOS") |
| `plat.ident()` | 返回平台标识符 |
| `plat.restart()` | 重启TPT |
| `plat.openLink(uri)` | 在浏览器打开链接 |
| `plat.clipboardCopy()` | 获取剪贴板内容 |
| `plat.clipboardPaste(text)` | 设置剪贴板内容 |

---

## fileSystem / fs -- 文件系统

| 函数 | 说明 |
|------|------|
| `fs.list(dir)` | 列出目录内容 |
| `fs.exists(path)` | 检查路径是否存在 |
| `fs.isFile(path)` | 是否是文件 |
| `fs.isDirectory(path)` | 是否是目录 |
| `fs.makeDirectory(path)` | 创建目录 |
| `fs.removeDirectory(path)` | 删除目录 |
| `fs.removeFile(path)` | 删除文件 |
| `fs.move(old, new, replace)` | 移动/重命名 |
| `fs.copy(old, new)` | 复制 |
| `fs.read(path)` | 读取文件内容(string) |
| `fs.write(path, data)` | 写入文件 |

```lua
-- 读取印章目录
local stamps = fs.list("stamps")
for _, f in ipairs(stamps) do
    tpt.log("印章文件: " .. f)
end

-- 写入配置文件
local config = "difficulty=hard\nparticles=1000\n"
fs.write("myconfig.txt", config)
```

---

## http -- HTTP请求

| 函数 | 说明 | 返回值 |
|------|------|--------|
| `http.get(url [, headers])` | GET请求 | status, body |
| `http.post(url, data [, headers])` | POST请求 | status, body |

```lua
-- GET请求
local status, body = http.get("https://example.com/api/data")
if status == 200 then
    tpt.log("获取成功: " .. #body .. " 字节")
else
    tpt.log("请求失败: " .. (status or "无响应"))
end

-- POST请求
local status, body = http.post("https://example.com/api/submit",
    '{"data":"test"}')
if status == 200 then
    tpt.log("提交成功")
end
```

---

## socket -- 网络和时间

| 函数 | 说明 |
|------|------|
| `socket.getTime()` | 返回Unix时间戳(毫秒精度) |
| `socket.sleep(time)` | 休眠(秒) |
| `socket.tcp()` | 创建TCP socket |
| `socket.select(recvt, sendt [, timeout])` | Socket选择 |

---

## bz2 -- 压缩

| 函数 | 说明 |
|------|------|
| `bz2.compress(data [, maxSize])` | bzip2压缩 |
| `bz2.decompress(data [, maxSize])` | bzip2解压 |

---

## bit -- 位运算

| 函数 | 说明 |
|------|------|
| `bit.band(a, b, ...)` | 位与 |
| `bit.bor(a, b, ...)` | 位或 |
| `bit.bxor(a, b, ...)` | 位异或 |
| `bit.bnot(a)` | 位非 |
| `bit.lshift(a, n)` | 左移 |
| `bit.rshift(a, n)` | 逻辑右移 |
| `bit.arshift(a, n)` | 算术右移 |
| `bit.rol(a, n)` | 循环左移 |
| `bit.ror(a, n)` | 循环右移 |

```lua
-- 提取属性标志位
local props = elements.property(id, "Properties")
local isConductive = bit.band(props, 32) ~= 0  -- PROP_CONDUCTS
local isDeadly = bit.band(props, 32768) ~= 0   -- PROP_DEADLY

-- 组合标志
local flags = bit.bor(bit.lshift(type, 16), bit.band(index, 0xFFFF))
```

---

## 模拟常量速查

```lua
-- 画布
XRES = 612           -- X像素分辨率
YRES = 384           -- Y像素分辨率
CELL = 4             -- 每CELL像素数
XCELLS = 153         -- X方向CELL数
YCELLS = 96          -- Y方向CELL数
PT_NUM               -- 元素种类总数
MAX_PARTS = NPART    -- 最大粒子数

-- 温度
R_TEMP = 295.15      -- 室温(22℃)
MAX_TEMP = 9999      -- 最高温度
MIN_TEMP = 0         -- 绝对零度

-- 压力
MAX_PRESSURE = 256   -- 最高压力
MIN_PRESSURE = -256  -- 最低压力

-- 转换标记
NT = -1              -- 无转换
ST = PT_NUM          -- 特殊转换
ITH                  -- 极高温度标记
ITL                  -- 极低温度标记
IPH                  -- 极高压力标记
IPL                  -- 极低压力标记

-- 边界
EDGE_VOID = 0        -- 虚空边界
EDGE_SOLID = 1       -- 固体边界
EDGE_LOOP = 2        -- 循环边界

-- 重力
GRAV_VERTICAL = 0    -- 垂直重力
GRAV_OFF = 1         -- 无重力
GRAV_RADIAL = 2      -- 径向重力
GRAV_CUSTOM = 3      -- 自定义重力

-- 空气
AIR_ON = 0           -- 完整空气
AIR_PRESSUREOFF = 1  -- 关压力
AIR_VELOCITYOFF = 2  -- 关风速
AIR_OFF = 3          -- 关空气
AIR_NOUPDATE = 4     -- 不更新

-- 像素模式
PMODE_NONE = 0       -- 不绘制
PMODE_FLAT = 1       -- 平面
PMODE_BLOB = 2       -- 圆形
PMODE_BLUR = 3       -- 模糊
PMODE_GLOW = 4       -- 发光
PMODE_SPARK = 5      -- 火花
PMODE_FLARE = 6      -- 耀斑
PMODE_LFLARE = 7     -- 局部耀斑
PMODE_ADD = 8        -- 叠加
PMODE_BLEND = 9      -- 透明度

-- 火焰
FIRE_ADD = 0         -- 叠加火焰
FIRE_BLEND = 1       -- 混合火焰
FIRE_SPARK = 2       -- 火花火焰

-- 装饰
DECO_DRAW = 0        -- 绘制
DECO_CLEAR = 1       -- 清除
DECO_ADD = 2         -- 叠加
DECO_SUBTRACT = 3    -- 减色
DECO_MULTIPLY = 4    -- 乘色
DECO_DIVIDE = 5      -- 除色
DECO_SMUDGE = 6      -- 模糊

-- 粒子标志
FLAG_STAGNANT = 1    -- 停滞
FLAG_SKIPMOVE = 2    -- 跳过移动
FLAG_MOVABLE = 4     -- 可移动
FLAG_PHOTDECO = 8    -- 光子装饰
```

---

## 实战示例

### 示例1：自动化脚本 -- 自动保存和恢复模拟

```lua
-- 每30秒自动保存印章
local lastSave = 0
event.register(evt.TICK, function()
    local now = socket.getTime()
    if now - lastSave > 30 then
        sim.saveStamp(0, 0, XCELLS, YCELLS, true)
        tpt.log("自动保存完成")
        lastSave = now
    end
end)

-- Ctrl+S手动保存，Ctrl+L加载最近的印章
event.register(evt.KEYPRESS, function(key, scan, repeat, shift, ctrl, alt)
    if key == 115 and ctrl and not repeat then  -- Ctrl+S
        local name = sim.saveStamp(0, 0, XCELLS, YCELLS, true)
        tpt.log("已保存: " .. name)
    end
end)
```

### 示例2：粒子分类计数器

```lua
-- 统计所有元素，显示分类
local function countAllElements()
    local categories = {
        [0] = {name = "墙壁", count = 0},
        [1] = {name = "电子", count = 0},
        [7] = {name = "液体", count = 0},
        [6] = {name = "气体", count = 0},
        [8] = {name = "粉末", count = 0},
        [9] = {name = "固体", count = 0},
        [10] = {name = "核材料", count = 0},
    }

    for i in sim.parts() do
        local t = sim.partProperty(i, 0)
        local elemId = t
        local section = elements.property(elemId, "MenuSection")
        if categories[section] then
            categories[section].count = categories[section].count + 1
        end
    end

    local report = "=== 粒子统计 ===\n"
    for _, cat in pairs(categories) do
        report = report .. string.format("%s: %d\n", cat.name, cat.count)
    end
    tpt.log(report)
end

-- 绑定到BEFORESIMDRAW
event.register(evt.BEFORESIMDRAW, countAllElements)
```

### 示例3：温度PID控制器

```lua
-- PID控制单个粒子的温度保持在目标值
local targetTemp = 500      -- 目标500K
local kp, ki, kd = 2.0, 0.1, 0.5
local integral = 0
local lastError = 0

local function pidControl(partId)
    local currentTemp = sim.partProperty(partId, 7)
    local error = targetTemp - currentTemp
    integral = integral + error
    local derivative = error - lastError
    local output = kp * error + ki * integral + kd * derivative
    lastError = error

    -- 限制输出
    output = math.max(-100, math.min(100, output))

    -- 应用修正
    sim.partProperty(partId, 7, currentTemp + output)
end

-- 控制指定粒子
local controlledId = sim.partCreate(-1, 306, 192, "METL")
event.register(evt.TICK, function()
    if sim.partExists(controlledId) then
        pidControl(controlledId)
    end
end)
```

### 示例4：自定义UI窗口

```lua
local win = ui.window(200, 100, 300, 200)
local slider, chkbox, label

win.onInitialized = function()
    -- 创建控件
    label = ui.label(10, 10, 280, 20, "画笔大小: 5")
    slider = ui.slider(10, 35, 280, 24, 20)  -- 0-20步
    slider:value(5)

    slider:onValueChanged(function(v)
        label:text("画笔大小: " .. v)
        ui.brushRadius(v, v)
    end)

    chkbox = ui.checkbox(10, 70, 280, 24, "启用自动保存")

    local saveBtn = ui.button(10, 105, 130, 24, "保存快照", "保存模拟状态")
    saveBtn:onMouseDown(function()
        sim.takeSnapshot()
        tpt.log("快照已保存")
    end)

    local loadBtn = ui.button(155, 105, 135, 24, "恢复快照", "恢复上一快照")
    loadBtn:onMouseDown(function()
        sim.historyRestore()
        tpt.log("快照已恢复")
    end)
end

win.onTick = function(dt)
    if chkbox:checked() then
        -- 自动保存逻辑
    end
end

win.onTryExit = function()
    return true  -- 允许关闭
end
```

---

## 交叉参考

- **附录E**：GOL规则（Lua操作LIFE粒子）
- **附录H**：调试模式（Lua控制台使用指南）
- **附录I**：管道系统（Lua脚本控制管道）
- **附录J**：活塞系统（Lua脚本控制活塞）
- **附录K**：种子遗传（Lua基因操作示例）
- **附录M**：温度压力阈值（Lua中温度操作参考）
- **附录N**：CanMove（Lua中sim.canMove()用法）
- **附录O**：FILT/LAVA（Lua中FILT操作）
- **附录P**：标志位（Lua中位运算操纵标志）
