# 洛克王国工具集（Roco Toolkit）

《洛克王国：世界》的非官方同人工具站，纯前端静态页面。主打五件事：看远行商人今天卖什么、翻精灵图鉴、按蛋的尺寸重量反查会孵出谁、查属性克制、查物品怎么用。

- 在线地址：<https://luoke.bangzhume.top/>
- 技术栈：React 18 + React Router 7 + Vite 6，无后端、无数据库
- 数据：图鉴和物品是构建期爬好的静态 JSON；商人是运行时直连官方公开接口

## 附加文档

| 文件 | 内容 |
|---|---|
| `docs/shortcut-alert.md` | 用 iPhone 快捷指令定时查「炫彩精灵蛋」，到货弹通知，零部署 |
| `docs/bark-alert.md` | 用 Bark 做真推送：服务器 cron 盯接口，命中即推，含完整脚本 |
| `docs/bark-selfhost.md` | 自建 Bark 服务端的评估与方案（Mac / 云服务器对比，含风险分析） |

---

## 功能页

| 路由 | 页面 | 说明 |
|---|---|---|
| `/` `/merchant` | 远行商人 | 当前轮次、在售商品与价格、下轮倒计时。每 60 秒自动刷新，本机会把每天每一轮的商品记账到 localStorage，沉淀出「最近常出现」榜单 |
| `/pokedex` | 精灵图鉴 | 644 只精灵，支持按名字/编号/英文名搜索、18 属性筛选、按编号或总种族值排序。点卡片出详情抽屉：种族值、特性、特长（含百分比）、技能表（按等级）、进化链、攻防克制 |
| `/egg` | 孵蛋反查 | 填蛋的尺寸（米）和重量（千克），反查所有可能孵出的精灵及概率，并给一句「最可能是谁」的判断 |
| `/type` | 属性克制 | 选 1–2 个属性，列出进攻能压制的、打不动的，以及防守怕的和扛得住的。双系按倍率叠乘，显示 ×2 / ×½ / ×¼ |
| `/items` | 物品图鉴 | 764 件物品，按分类筛选，可搜名字、用途、获取途径。商人页的商品名能直接跳到这里对应的条目 |

顶栏是路由切换（不是组件内 tab），所以每个页面都有独立 URL，可以直接分享某一页。

---

## 本地跑起来

```bash
npm install
npm run dev      # 默认 http://localhost:5175
```

构建与本地预览构建产物：

```bash
npm run build
npm run preview
```

Node 建议 18 及以上。`dev` 端口在 `vite.config.js` 里写死为 5175。

> macOS 上如果 `localhost:5175` 打不开，试 `http://[::1]:5175` —— Vite 默认只监听 IPv6 的 loopback。

---

## 目录结构

```
洛克王国/
├── index.html
├── vite.config.js          # 含 /egg-api 开发代理
├── scripts/
│   └── update-data.py      # 图鉴 / 物品爬虫，输出到 public/data
├── public/
│   ├── data/
│   │   ├── pokedex.json    # 644 只精灵 + 580 技能 + 特长表（约 1.06 MB）
│   │   └── items.json      # 764 件物品（约 274 KB）
│   └── img/                # avatar.png / mascot.png
└── src/
    ├── main.jsx            # BrowserRouter 挂载
    ├── App.jsx             # 顶部导航 + 路由表
    ├── styles.css          # 全站样式与主题变量
    ├── data/types.json     # 18 属性克制表（唯一一份，全局共用）
    ├── lib/
    │   ├── merchant.js     # 商人接口、时区换算、倒计时、本地记账
    │   ├── egg.js          # 孵蛋查询与 HTML 解析
    │   ├── types.js        # 克制倍率计算
    │   └── rocoData.js     # 图鉴/物品加载、搜索匹配、精灵数据展开
    └── tools/
        ├── Merchant.jsx
        ├── Pokedex.jsx
        ├── MonDetail.jsx   # 精灵详情抽屉（被 Pokedex 引用）
        ├── Egg.jsx
        ├── TypeChart.jsx
        └── Items.jsx
```

---

## 数据从哪来

### 1. 远行商人（运行时接口）

```
https://rocokingdomworld.org/api/merchant/live      ← 主源，CORS 全开，可浏览器直连
https://rocokingdomworld.org/data/merchant.json     ← 兜底静态快照
```

返回里用到 `status`（`open` / 其他）、`round`（1–4）、`items`（`name` / `priceRaw` / `category` / `rounds`）、`startedAtBeijing`、`nextRefreshBeijing`。

几个要注意的点：

- 服务器返回的是**北京墙钟时间**字符串，不带时区。`bjTime()` 先按原样拼成 UTC 再减 8 小时，别用 `new Date(str)` 直接转，会因为本机时区错位。
- 轮次固定四段：08:00–12:00 / 12:00–16:00 / 16:00–20:00 / 20:00–24:00。
- 价格单位看 `category`：带「活动」字样的用蜂窝计价，其余是洛克贝。
- 两个源都拉不到时页面会给出手查链接，不白屏。

### 2. 精灵图鉴 / 物品图鉴（构建期静态数据）

源站是 SSR 的 HTML，只能爬。脚本只用 Python 标准库，全量 1408 个页面、8 并发，约 12 分钟：

```bash
python3 scripts/update-data.py            # 全量（精灵 + 物品）
python3 scripts/update-data.py --pets     # 只更新精灵图鉴
python3 scripts/update-data.py --items    # 只更新物品图鉴
```

- 抓取结果缓存在 `.data-cache/`（已 gitignore），重复跑会走缓存，很快。
- 输出直接覆盖 `public/data/*.json`，格式压成单行并打印 gzip 体积。
- 脚本里手动关掉了系统代理（`ProxyHandler({})`），走系统代理环境时不会卡死。
- 源站改版就要改 `parse_mon` / `parse_item` 里的正则，没有更稳的办法。

### 3. 属性克制（本地静态）

`src/data/types.json` 是 18 属性的 `{强克, 抵抗}` 表，抄自 B 站 wiki 的 `Widget:RestrainCalc.js`。图鉴里不存每只精灵的克制关系，而是用 `computeMatchup()` 按属性现算，双系自动叠乘——这样属性表改了不用重爬图鉴。

### 4. 孵蛋反查（需要代理）

数据源 `https://luokewangguofudan.wiki/?size=&weight=` **没有 CORS 头**，浏览器直连必被拦。

- 开发环境：`vite.config.js` 里 `/egg-api` 代理到该站，顺手补了 UA 和 `Accept`。
- 线上环境：需要在 nginx 里加一条同样的反代，前端请求的是相对路径 `/egg-api/?size=..&weight=..`，不用改代码。
- 解析用的是「把标签替换成空格再跑正则」，不是 `textContent`。因为相邻标签会被 `textContent` 拼成一串（`No.007火花`），正则里 `\s+` 直接匹配失败，之前在这里踩过坑。

---

## 数据文件格式

`pokedex.json`：

```jsonc
{
  "generated": "2026-11-05",
  "source": "rocokingdomworld.org",
  "pets": [
    {
      "slug": "dimo", "id": "NO001", "name": "迪莫", "nameEn": "Dimo",
      "types": ["光"], "total": 582,
      "stats": { "hp": 120, "atk": 80, "matk": 80, "df": 105, "mdf": 105, "spd": 92 },
      "ability": { "name": "最好的伙伴", "desc": "…" },
      "specs": [["无", 64.0], ["奇袭", 6.91]],   // [特长名, 百分比]
      "sk":    [["1", "猛烈撞击"]],              // [学习等级, 技能名]
      "evo":   ["迪莫", "圣光迪莫"]              // 进化链
    }
  ],
  "skills": { "猛烈撞击": { "type": "普通", "cat": "物理", "power": 40, "pp": 35, "effect": "…" } },
  "specialties": { "奇袭": "…" }
}
```

刻意做了两处冗余压缩：精灵只存技能**名字**，技能详情放全局字典；特长同理。这样 644 只精灵的文件能压到 1 MB 出头。组件里用 `expandPet(pet, skills, specialties)` 展开成能直接渲染的结构。

`items.json` 是扁平数组：`{ slug, id, name, img, cat, sub, use, obtain }`。

物品图鉴有一个已知缺口：源站没收录限时活动道具（比如捣蛋糖果），商人在卖但库里搜不到。

---

## 加一个新工具页

1. `src/tools/Xxx.jsx` 写组件，取数放 `src/lib/` 下；
2. `src/App.jsx` 的 `TABS` 加一项（`to` / `label` / `icon`）；
3. `Routes` 里加一条 `<Route path="/xxx" element={<Xxx />} />`。

样式统一用 `styles.css` 里的现成类：`panel` / `panel-title` / `chip` / `notice` / `dex-grid` / `dex-card` 等，主题色走 CSS 变量，别在组件里写死颜色。

---

## 部署

当前线上是 nginx 托管静态文件 + Node 构建产物：

```bash
npm run build                      # 产物在 dist/
```

发布结构用 releases 目录切换，站点的 root 指向 `current` 软链：

```
/home/web/roco-toolkit/
├── current -> releases/20261105-3
└── releases/
    ├── 20261105-1
    ├── 20261105-2
    └── 20261105-3
```

nginx 站点配置 `/etc/nginx/sites-available/luoke.bangzhume.top.conf`，两个必须注意的点：

1. **SPA 回退**：`try_files $uri $uri/ /index.html;`，否则刷新 `/pokedex` 之类的子路由会 404。
2. **gzip**：`gzip_types` 要显式包含 `application/javascript`。只写 `text/*` 的话 JS 不压缩，主包 200 KB 会原样传，加了之后约 77 KB。

证书走 ZeroSSL ECC，到期 2027-02-03。

---

## 已知限制

- 3 只精灵源站没有属性、10 只没有技能记录（是源站缺失，不是解析丢的）。
- 商人历史记账只存在当前浏览器的 localStorage，换设备/清缓存就没了。
- 图鉴首次加载 1 MB（gzip 后 250 KB），弱网下会有明显等待；没做分页预取，只在列表底部「显示更多」。
- 精灵图片全部热链源站，源站挂图就白块（组件里 `onError` 会直接隐藏）。
- 没有活动日历。源站有 37 个活动，是后续可以补的一块。

---

## 免责声明

非官方同人作品，与腾讯及《洛克王国：世界》官方无关。图鉴与物品数据整理自 rocokingdomworld.org，版权归原权利方所有。仅供玩家查阅，不做商业用途。
