# 乌托邦 3D战略游戏 - 上下文恢复文档
最后更新：2026-06-26

## 项目概述
基于 Three.js 的 3D 地球战略游戏，模仿 AOH3 的**UI风格、游戏机制和玩法**。
单文件 HTML（`index.html`，~2500行），包含所有 CSS/JS/HTML。
GitHub仓库：https://github.com/mr2820404807-art/3d-Earth-game

## ⚠️ 本次会话的惨痛教训

### 绝对不能做的事
1. **永远不要用 `git checkout` 覆盖工作目录**——除非你100%确认不需要恢复
2. **永远不要用 `git stash` 后做 `Copy-Item` 备份**——stash会把修改版藏起来，Copy-Item复制的是已还原的干净版
3. **永远不要用正则 `[\s\S]*?` 做大范围替换**——它会吃掉多个函数定义，把中间的代码全部删掉
4. **每次编辑前先 `Copy-Item` 备份当前版本到一个带时间戳的文件**
5. **edit工具的 old_string 匹配会吃掉相邻代码**——必须精确定位到唯一行

### 正确的备份流程
```powershell
Copy-Item "index.html" "index_backup_$(Get-Date -Format 'yyyyMMdd_HHmmss').html"
```

## 文件版本管理
- **index_backup_v2.html**（133KB）：本次会话最后的可用版本，包含所有功能
- **index.html**：当前工作版本，基于v2
- **GitHub main分支**：已push，包含v2版本

## 游戏名称
- 标题：**乌托邦**（原名"历史时代三·虚革"，后改为"乌托邦"）
- 英文：Utopia · 3D Strategy

## 已实现的功能

### UI布局（AOH3风格）
- **顶栏(#aoh-top-bar)**：国旗+国名 | 金币/经济/人力/外交/威望/人口/增长率/战争 | 时间控制(-/速度/+/暂停)
  - 悬浮资源图标显示 .tip 提示框
  - 点击国旗弹出暂停菜单
  - 点击外交资源弹出外交关系弹窗
- **左侧边栏(#aoh-left-sidebar)**：10个地图模式按钮（政治/地形/经济/人口/外交/宗教/文化/贸易/军事/意识形态，目前只弹通知）
  - 每个按钮有 .tip 悬浮提示
- **底部军事栏(#aoh-bottom-panel)**：军官/重组/运输/要塞/殖民 + 军团数/上限/军官数
  - 所有按钮有 title 属性提示
- **右侧信息面板(#aoh-right-panel)**：点击省份时显示
  - 省份信息：国旗+省名、省份经济、建筑等级、人口、建筑列表、征兵按钮
  - 建筑点击用 event delegation
- **右侧功能按钮(#aoh-side-btns)**：科技树(可研究)/任务/法律/宫廷/战争
  - 每个按钮有 .tip 悬浮提示
- **科技树弹窗(#tech-popup)**：20个科技，带研究时间和独特加成
- **军官弹窗(#officer-popup)**：招募/刷新军官
- **军团管理弹窗(#army-popup)**：管理军团
- **外交弹窗(#diplomacy-popup)**：查看外交关系
- **通知系统(#aoh-notifications)**：顶部居中弹出
- **暂停菜单(#pause-overlay)**：继续游戏/返回主菜单

### 游戏系统

#### 经济系统
- `countryEconomyData`：60+国家真实GDP分配(0-100)
- `aohTickEconomy()` 每个时间刻度结算
  - 金币收入 = (经济-30) × 1.5 × (1 + 科技加成)
  - 人力增长 = 经济 × 0.5
  - 威望：经济>60时+1，<40时-1
  - 外交点 = 基础 + 经济×0.04 + 科技加成
  - 经济变化：经济>60时12%概率+1，<40时12%概率-1，中间8%随机波动

#### 人口增长率系统
- `countryPopulationData`：60+国家真实人口（万人单位）
- `countryGrowthRateData`：各国基础年增长率（%）
- `aohCalcGrowthRate(eco, baseRate)`：经济增长率修正
  - 经济≥80：+0.06/经济点
  - 经济50-79：+0.03/经济点
  - 经济30-49：-0.02/经济点差
  - 经济<30：-0.04/经济点差
  - 最终增长率限制在 [-2%, +5%]
- `aohTickPopulation()`：每24个时间单位更新一次人口
  - 密度制约：人口接近50万时增长趋缓（densityFactor = max(0.1, 1 - pop/500000)）
  - 人力上限 = 人口 × 5%

#### 省份建筑系统
- `provinceData`：按 `国家::省份` 存储，每个省份独立经济值（初始用名称hash随机变异±20）
- `buildingDefs`：5种建筑
  | 建筑 | 图标 | 基础费用 | 效果 | 上限 |
  |------|------|----------|------|------|
  | 陆上交通 | 🛤 | 40金 | 军队移速+15%/级 | Lv.3 |
  | 海上运输 | 🚢 | 60金 | 经济+3/级（仅沿海） | Lv.3 |
  | 基础设施 | 🏗 | 50金 | 经济+2/级 | Lv.5 |
  | 农业开发 | 🌾 | 30金 | 经济+1/级 | Lv.5 |
  | 军营 | 🏕 | 50金 | 军团上限+3/级 | Lv.3 |
- 建筑费用递增：`基础费用 × (1 + 当前等级 × 0.5)`
- **沿海判断**：用 `coastalCountries` 国家列表 + `inlandExceptions` 内陆省份排除表
- `aohTickProvinceBuildings()`：每刻度自动提升省份经济
- 军营效果：所有省份军营等级叠加，影响军团上限

#### 科技树系统
- 20个科技，每个有：名称、图标、费用、研究天数、加成描述、apply函数
- 研究机制：同一时间只能研究一个，每刻度推进一天
- 全局加成 `aohTechBonuses`：armyAtk/armyDef/goldIncome/diploTick/popGrowth/infraCost/roadCost/navalMaint/allCost
- 金币收入受 `goldIncome` 加成影响
- 外交点受 `diploTick` 加成影响

| 科技 | 费用 | 天数 | 加成 |
|------|------|------|------|
| 冶金术 | 100 | 24 | 军团攻击+2 |
| 军事纪律 | 120 | 30 | 军团防御+2 |
| 航海术 | 100 | 24 | 海军维护费-10% |
| 骑兵战术 | 130 | 36 | 军团攻击+3 |
| 火药武器 | 150 | 40 | 军团攻击+5 |
| 印刷术 | 80 | 20 | 外交点+1/刻度 |
| 银行制度 | 160 | 36 | 金币收入+20% |
| 大学教育 | 140 | 32 | 经济+3 |
| 蒸汽机 | 180 | 40 | 基建费用-20% |
| 铁路运输 | 160 | 36 | 交通费用-20% |
| 电报通讯 | 150 | 32 | 外交点+2/刻度 |
| 工业化 | 200 | 48 | 经济+5 |
| 现代医学 | 180 | 40 | 人口增长率+1% |
| 核能技术 | 250 | 56 | 军团攻击+10 |
| 信息技术 | 220 | 48 | 所有建筑费用-15% |
| 人工智能 | 280 | 60 | 经济+3 增长率+0.5% |
| 太空探索 | 300 | 64 | 威望+50 |
| 量子计算 | 350 | 72 | 经济+5 增长率+1% |

#### 军官系统
- 5种专长：攻击(⚔)/防御(🛡)/机动(🐎)/围城🏰/指挥(🎖)
- 每个军官有：名称、等级(Lv.1-3)、基础攻击/防御、专长加成、招募费用
- 军团有：baseAttack、baseDefense、officer（可指派/卸任）
- 军官加成叠加到军团上

#### 时间系统
- 每日8个时段（00:00-21:00），每月自动进位
- 速度等级：x0/x1/x2/x4/x8/x16
- 空格暂停/继续，Escape关闭弹窗
- advanceTime/tickTime 被包装，每刻度调用 aohTickEconomy → aohTickPopulation → aohTickProvinceBuildings → aohTickTech

#### 国家选择
- 点击地球→射线检测→省份/国家级选择→预览国旗→确认→资源初始化→进入游戏
- 选择阶段：地球Y轴跟随鼠标视差，X轴自由旋转
- 确认后：地球自动旋转到选中国家位置

#### 框选系统
- 鼠标中键拖拽→蓝色矩形→松开→投影省份中心到屏幕→判断是否在框内→高亮选中
- 显示框选结果面板：省份数量、平均经济、批量征兵

#### 星空跟随
- 星空在 scene 上，不在 earthGroup 内
- 手动拖拽/有惯性时同步：`sg.rotation.y = earthGroup.rotation.y`

### 按键
- **Space**：暂停/继续时间
- **Escape**：取消选择 / 关闭弹窗 / 关闭暂停菜单
- **滚轮缩放**：仅游戏阶段可用
- **鼠标中键**：框选省份
- **左键拖拽**：旋转地球

## 关键架构决策

### 事件监听必须在 document 上
所有 `renderer.domElement.addEventListener` 改为 `document.addEventListener`，handler内检查 `e.target !== renderer.domElement` 过滤。
原因：AOH3面板 `pointer-events: auto` 会拦截 canvas 事件。

### .aoh-popup 默认 pointer-events: none
否则 `opacity:0` 的380px居中不可见div会挡住整个屏幕中间的所有操作。

### advanceTime / setTimeSpeed 被包装
```js
const _origAdvanceTime = advanceTime;
advanceTime = function() { _origAdvanceTime(); aohTickEconomy(); aohTickPopulation(); aohTickProvinceBuildings(); aohTickTech(); };
```
注意：addEventListener 绑定后重新赋值函数不会更新绑定，必须 remove + add。

### 事件委托（event delegation）用于动态内容
建筑列表、框选结果列表等动态生成的内容，点击事件用父容器委托：
```js
document.getElementById('rp-buildings').addEventListener('click', (e) => {
  const btn = e.target.closest('.prov-build-btn');
  if (!btn) return;
  // 处理点击
});
```

### ID命名约定（重要！）
- HTML按钮用 `btn-xxx` 格式（btn-slower, btn-faster, btn-pause）
- JS中引用同一ID，不要混用 `aoh-xxx` 和 `btn-xxx`
- AOH3面板元素用 `aoh-xxx` 格式（aoh-top-bar, aoh-left-sidebar等）
- 值显示用 `val-xxx` 格式（val-gold, val-economy等）

## 编辑注意事项（踩过的坑）

1. **earthGroup 不能丢**：编辑后必须保留 `const earthGroup = new THREE.Group(); scene.add(earthGroup);`
2. **mouseNormY / mouseScreenX / mouseScreenY 不能删**：pointermove handler 中使用
3. **`opacity: 0` 不等于 `pointer-events: none`**：隐藏元素如果没设 pointer-events: none 会拦截事件
4. **不要在 animate() 开头加 try-catch 做调试**
5. **单文件2500行，必须外科手术式编辑**：每次改完验证地球动画、贴图是否正常
6. **edit工具的正则替换极度危险**：`[\s\S]*?` 会跨函数吃掉代码，永远用精确字符串匹配
7. **编辑前必须备份**：`Copy-Item "index.html" "index_backup_日期时间.html"`
8. **构建大段新代码时用 node 脚本做字符串拼接插入**，不要在 edit 工具里写多行模板

## 本次会话丢失过什么
- 整个修改版被 `git checkout HEAD -- index.html` 覆盖
- 原因：尝试修复bug时直接checkout了，没有意识到修改版从未commit
- 教训：修改版必须及时push到GitHub或备份到独立文件

## 文件路径
- 项目主文件：`c:\Users\28204\MIMO code\3d-earth\index.html`
- 备份文件：`c:\Users\28204\MIMO code\3d-earth\index_backup_v2.html`
- 地球贴图：`c:\Users\28204\MIMO code\3d-earth\earth_hd.jpg`
- AOH3游戏参考：`d:\DOWNLOAD\aoh\历史时代三 虚革 电脑版 1.4.1\`
  - 游戏数据：`game/gameValues/`、`game/buildings/`、`game/laws/`、`game/technologies/`
  - FAQ文档：`game/_FAQ/`（Buildings.txt、Laws.txt、Technology.txt、Units.txt等）
- GitHub仓库：https://github.com/mr2820404807-art/3d-Earth-game

## 待实现功能
- 地图模式实际切换（目前只弹通知）
- 任务/法律/宫廷/战争系统
- 重组/运输/要塞/殖民功能
- 外交关系影响游戏数值
- 军官系统完整效果（攻击/防御/机动加成实际应用到战斗）
- 更多国家完整数据
