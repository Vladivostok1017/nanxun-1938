# 南浔线 1938 · 一个 Iron Mind 战役模块

> 1938 年 7–10 月，长江南岸。日军要打通南浔铁路，国军要在德安、万家岭一线把它拖住。
> 你是第九战区的长官：只能发电报，具体怎么打是下面那些军长师长的事。

[![license: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![engine](https://img.shields.io/badge/engine-%E2%89%A5%200.12-2f6f4f)

这是一个**内容模块**，不是引擎。引擎在 [iron-mind](https://github.com/Vladivostok1017/iron-mind)，
它只提供框架（tick、实体、补给、战斗、指挥链、LLM 将领），
这一仗"在哪打、谁打谁、想要什么"全部在这里。

三个仓库分开是有意的：

| 仓库 | 负责 | 不该出现的东西 |
|---|---|---|
| [iron-mind](https://github.com/Vladivostok1017/iron-mind) | Kernel / Systems / Rules / 服务端 / 前端 | 任何"1938 年""日本""中国"的硬编码 |
| [iron-mind-sdk](https://github.com/Vladivostok1017/iron-mind-sdk) | 模块脚手架、开发循环、将来的可视化编辑器 | 任何引擎逻辑 |
| **nanxun-1938**（本仓库） | 地图、序列、将领、规则、结局、战役文档 | 任何引擎逻辑 |

> **想自己做一个模块？** 看引擎仓库的 [`content/README.md`](https://github.com/Vladivostok1017/iron-mind/blob/main/content/README.md)，
> 那是模块格式的完整契约（字段表、结局/规则写法、校验码、发布清单）。
> 起步最快的办法：
>
> ```powershell
> cd ..\iron-mind-sdk
> node tools\module-init.js --id org.you.mymod --name "我的战役"   # 生成空模块
> node imdev.mjs --module ..\nanxun-1938\nanxun_1938.imod          # 改完存盘就重载
> ```
>
> 它会生成一个能直接加载、校验零警告的空模块，改 map.json / oob.json 里的占位数据就行。
> 开发工具在 SDK 仓库——引擎出货的二进制不带开发行为，作者工具也不该混进玩家安装包。

引擎不认识"国军""日军"，它只认识 side、unit、province 与内容声明的规则。
所以换一仗不用改引擎；改引擎也不会动这一仗。

---

## 怎么跑

### 只想玩

去 [Releases](https://github.com/Vladivostok1017/nanxun-1938/releases) 下 `nanxun_1938-v0.1.0.imod`，
在游戏里「新的战役 → 导入 .imod」选它就行。（`.imod` 是一个 zip：里面是本目录那五个 json。）

### 想改它 / 调试它

```powershell
cd ..\iron-mind
$env:IRONMIND_MODULES = "F:\dsh\nanxun-1938"   # 告诉引擎去这个目录找模块
cargo run -p sim-server                         # 默认就会捡到本模块
# 或者显式指定（目录、压缩包都吃）：
cargo run -p sim-server -- --module ../nanxun-1938/nanxun_1938.imod
```

浏览器打开 http://127.0.0.1:8899/ 。模型密钥放在引擎那边的 `.env`：
`IRONMIND_API_KEY=...`、`IRONMIND_MODEL=deepseek-chat`。

改完存盘就重载的开发循环在 [iron-mind-sdk](https://github.com/Vladivostok1017/iron-mind-sdk)：
`node imdev.mjs --module ..\nanxun-1938\nanxun_1938.imod`。

> **源文件是目录，不是压缩包。** 本仓库里 `nanxun_1938.imod/` 是文件夹——那样才能 diff、
> 才能热重载；`.imod` 压缩包只在发布时由引擎仓库的 `tools/package.ps1` 打出来。

## 目录

```
nanxun_1938.imod/
  module.json         元信息：名字、可玩方、尺度、敌方统帅部（岡村寧次）
  map.json            87 个省：真实经纬度、地形（DEM 实测）、铁路、邻接
  provinces_geo.json  真实县界多边形（DataV），前端据此画地图
  oob.json            26 个部队：番号、兵力、将领性格、编制、上级
  scenario.json       77 天战役：通信条件、模块规则、6 条结局
docs/
  01-武汉会战场景设计.md   五条战线、战斗序列、派系、MVP 裁剪
  02-视频精读-战斗记录.md  47 分钟战役视频逐帧精读（870 行）
  03-史料核对-战斗序列.md  中方编制树到师级+师长、派系、时间线矛盾
  06-战役数据底稿.md       三源合并的兵力与时间线
```

## 这一仗的设定

- **规模**：87 个省，约 10 公里一个（`scale: "front"`，正面/方面军级）；
  师/军级单位，1 tick = 1 天。这个粒度是为了能表达"罗店式"的逐阵地争夺——
  罗店/德安/万家岭各自都有独立的得失节奏。
- **时间**：1938-07-26 起 77 天。
- **兵力**：国军 227.8 千（18 个番号，其中 2 个迟到）vs 日军 96.5 千（8 个）。
  史实上中国军队人数远多于日军，靠装备与补给劣势才打成苦战；
  单兵差距由 `effectiveness`（国军 0.9–1.15 / 日军 1.26–1.53）表达，
  后勤差距由 `supply_reach`（0.1–0.16 / 0.45–0.5）表达。
- **连续正面模型**（`scale: "front"` 自动启用）：占领只看本省是否清空，
  围歼看邻省有没有我方部队，窄阵地按软上限算战力。这一档的平衡单独标过：
  77 日实测 **国军 107.8 千 / 46 省，日军 61.7 千 / 41 省，结局「两败俱伤」，
  德安失守、南昌还在手里**——与史实（德安 1938-10 失守、南昌 1939-03 才陷）一致。
  为了让密模型站得住，本模块比初版多了 8 个守备师（历史口径：南浔正面二十余师），
  并把日军 `effectiveness` 下调约 10%（原来那套是按 35 公里/省的粗口径标的）。
- **地形**：不是手写的。由 `tools/mapgen/terrain.js`（在引擎仓库）从
  AWS Terrain Tiles 的 DEM 采样判定，33 个省改过判。
- **地势与原稿的差异**：真实县界取自 DataV（现代县界，作为 1938 年的近似），
  1938 年的县界数据不可得。
- **敌方统帅部**：`module.json` 里的 `hq` 是岡村寧次（第11军司令官），
  他每隔几天开一次会、给自己的师团下电报，师团长可以抗命。

## 数据可信度

`oob.json` 里 `estimated: true` 表示**推断值**，不是档案数字。
兵力口径：一个国军师约 7–9 千（1938 年实编、缺额严重），一个军 2–3 师；
日军一个常备师团 20–25 千。凡是查得到出处的写在 `note` 里，
查不到的标 `estimated`——**不要为了让数值好看而删掉这个标记**。

## 改这个模块

改数据即可，不用碰 Rust：

```powershell
cd ..\iron-mind
node tools/mapgen/terrain.js stat    # 地形判定表（只读）
node tools/mapgen/terrain.js build   # 从 DEM 重算地形，写回 map.json
```

改完重开服务（浏览器里也能直接切模块：左下角下拉框）。

## 校验

引擎加载时会做 20 多条静态校验（邻接对称、补给源可达、编制连续、
兵力对比是否反常……）。警告会打在服务端启动日志里，例如：

```
balance.lopsided  可玩方总兵力 89 千，敌方 96 千（93%）…
```

那条警告就是本模块早期写错兵力的直接证据。
