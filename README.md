# The Powder Toy 中文元素说明

基于 [The-Powder-Toy-Chinese](https://github.com/Dragonrster/The-Powder-Toy-Chinese) 最新源码 + [官方英文 Wiki](https://powdertoy.co.uk/Wiki.html) 编译的完整中文元素参考文档。

> 旧版《TPT元素说明 Version 97》（Ike / 91.5 版）已移至 [OLD/](OLD/) 目录存档。

## 快速开始

- **首次使用？** 从 [index.md](index.md) 开始——含参数速查和完整目录
- **找某个元素？** 打开 [appendix-L-完整元素索引.md](appendix-L-完整元素索引.md) 按字母查找
- **想学电路？** 读 [appendix-G-电路构建指南.md](appendix-G-电路构建指南.md) 入门，[appendix-Q-Subframe子帧技术.md](appendix-Q-Subframe子帧技术.md) 进阶
- **想学 Lua？** [appendix-R-Lua脚本入门指南.md](appendix-R-Lua脚本入门指南.md) 零基础开始
- **想建反应堆？** [appendix-S-核反应堆构建指南.md](appendix-S-核反应堆构建指南.md)
- **想造计算机？** [appendix-T-计算机构建指南.md](appendix-T-计算机构建指南.md)

## 文档结构

### 元素分类

| 分类 | 文件 | 元素数 |
|------|------|--------|
| 墙类 | [墙类.md](墙类.md) | 19 种墙 |
| 电子类 | [电子类.md](电子类.md) | 21 个 |
| 可控材料 | [可控材料.md](可控材料.md) | 10 个 |
| 传感器 | [传感器.md](传感器.md) | 7 个 |
| 力 | [力.md](力.md) | 9 个 |
| 爆炸物 | [爆炸物.md](爆炸物.md) | 21 个 |
| 气体 | [气体.md](气体.md) | 14 个 |
| 液体 | [液体.md](液体.md) | 22 个 |
| 粉末 | [粉末.md](粉末.md) | 20 个 |
| 固体 | [固体.md](固体.md) | 33 个 |
| 核能 | [核能.md](核能.md) | 17 个 |
| 特殊 | [特殊.md](特殊.md) | 20 个 |
| 生命 | [生命.md](生命.md) | 1 + 24 种 GOL 规则 |
| 工具 | [工具.md](工具.md) | 13 种 |
| 装饰 | [装饰.md](装饰.md) | 7 种 |

### 数据参考

| 附录 | 内容 |
|------|------|
| [A - Type 值表](appendix-A-Type值表.md) | 195 元素唯一编码 |
| [B - 导热速度表](appendix-B-导热速度表.md) | 热导率排名 |
| [C - PHOT 反射值表](appendix-C-PHOT反射值表.md) | 波长掩码详解 |
| [D - Lua API](appendix-D-Lua-API参考.md) | 完整 API 函数签名 |
| [E - GOL 规则](appendix-E-GOL生命游戏规则.md) | 24 种内置生命游戏规则 |
| [F - 元素配方](appendix-F-元素制取配方.md) | 全部合成配方 |
| [L - 元素索引](appendix-L-完整元素索引.md) | 按字母快速查找 |

### Lua 与脚本

| 附录 | 内容 |
|------|------|
| [D - Lua API 参考](appendix-D-Lua-API参考.md) | 完整函数签名和参数 |
| [R - Lua 脚本入门](appendix-R-Lua脚本入门指南.md) | 零基础到实用脚本 |

### 系统构建指南

| 附录 | 内容 |
|------|------|
| [G - 电路指南](appendix-G-电路构建指南.md) | 基础电路到高级模式 |
| [H - 调试指南](appendix-H-调试模式指南.md) | 调试视图和工具 |
| [I - 管道系统](appendix-I-管道系统详解.md) | PIPE/PPIP 详解 |
| [J - 活塞框架](appendix-J-活塞框架系统.md) | PSTN/FRME 联动 |
| [K - 种子遗传](appendix-K-种子遗传系统.md) | SEED/PLNT 育种 |

### 进阶参考

| 附录 | 内容 |
|------|------|
| [M - 温度压力阈值](appendix-M-温度与压力阈值参考.md) | 关键转换点速查 |
| [N - CanMove 规则](appendix-N-CanMove交互规则.md) | 粒子移动条件 |
| [O - FILT 波长](appendix-O-FILT波长模式与LAVA-Ctype参考.md) | 10 种 tmp 模式 |
| [P - 粒子标志位](appendix-P-粒子标志位与状态常量参考.md) | PROP_xxx 详解 |

### 高级教程

| 附录 | 内容 |
|------|------|
| [Q - Subframe 子帧技术](appendix-Q-Subframe子帧技术.md) | 60Hz 高速计算 |
| [R - Lua 脚本入门](appendix-R-Lua脚本入门指南.md) | 零基础脚本教程 |
| [S - 核反应堆](appendix-S-核反应堆构建指南.md) | 4 种反应堆设计 |
| [T - 计算机构建](appendix-T-计算机构建指南.md) | 4 位可编程计算机 |
| [U - 自定义元素](appendix-U-自定义元素开发教程.md) | C++ + Lua 双方案 |

### 其他

- [STANDARD.md](STANDARD.md) — 文档撰写标准
- [OLD/](OLD/) — 旧版 V97 参考文档（存档）

## 数据说明

- 元素总数：**195**
- 数据来源：`The-Powder-Toy-Chinese/src/simulation/elements/*.cpp`
- 中文描述：取自源码中的 `Description` 字段
- 图片资源：来自 [官方 Wiki](https://powdertoy.co.uk/Wiki.html)（195 个元素图标 + 56 个演示 GIF）
- 更新日期：2026-05

## 撰写标准

每个元素文档包含：
1. 基本属性表（内部标识、颜色、类型、重量、硬度、热导率）
2. 核心工作机制（Update/Graphics 函数逻辑）
3. 参数控制系统（tmp/tmp2/life/ctype/temp 含义和默认值）
4. 特殊交互与反应链
5. 应用场景（至少 2-3 个具体用法）

详见 [STANDARD.md](STANDARD.md)。

## 相关项目

- [The-Powder-Toy-Chinese](https://github.com/Dragonrster/The-Powder-Toy-Chinese) — 游戏源码
- [官方 Wiki (英文)](https://powdertoy.co.uk/Wiki.html)
- [TPT 元素说明 Wiki.js 版](https://github.com/Dragonrster/Wikijs/tree/main/TPTWIKI)
