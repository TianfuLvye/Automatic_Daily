# 渔网计划 · Fishnet Lab

> **一份结果导向的爬虫 / 信息聚合实验手册**
> 目标产物:一套每天 07:30 与 18:30 自动把「当日情报」排版成 PDF 推送给你的个人报纸系统。
> 适用对象:大三升大四 CS 学生,有 Python 基础,会用 Git,能看懂 Docker 命令。
> 预计投入:4–6 周,每天 2–3 小时(暑假节奏)。

---

## 0. 这份 Lab 的设计哲学

普通教程的做法是「一个项目讲一章」,学完你会得到七个互不相干的玩具。
这份手册反过来:**先定义最终产物,再倒推每个开源项目在其中扮演什么零件**。

所以每个 Lab 都遵守同一条铁律:

> **做完这个 Lab,你的仓库里必须多出一个能被最终系统直接 import 的模块。**
> 不产出可复用零件的练习,一律砍掉。

另外一条:**你不是在学"怎么用 TrendRadar",你是在学"一个信息采集系统的哪些部分不可外包"**。

- 可外包的:平台适配、反爬对抗、热榜接口维护 → 用开源项目,别自己写
- 不可外包的:数据契约、去重策略、个性化排序、排版渲染、调度可靠性 → 这是你的系统的灵魂,也是这份 Lab 的重点

看清楚这条分界线,是这次实验最值钱的收获。

### 关于 vibe coding 的一句提醒

你打算用 AI 辅助写这个系统,这完全合理。但 vibe coding 有个已知的塌方点:**当 AI 不知道你的数据长什么样时,它会每个文件都发明一套自己的格式**,三天后你的 12 个采集脚本会输出 12 种 JSON,然后你就再也整合不起来了。

所以 **Lab 0 的数据契约是整份手册最重要的一节**,重要性超过后面所有爬虫项目的总和。把它写死、写进 README、每次开新对话都先贴给 AI。

---

## 1. 目标系统架构

### 1.1 渔网隐喻 → 工程组件映射

| 渔民的活 | 系统组件 | 关键词 |
|---|---|---|
| 撒网(常年泡在水里) | **Collector** 采集器 | 高频、只管抓、不管好坏 |
| 鱼舱(先囫囵倒进去) | **Raw Store** 原始层 | 幂等入库、保留原始 payload |
| 分拣台(去杂物、去重复) | **Normalizer + Deduper** | 统一 schema、content hash |
| 挑好鱼(哪些今天上桌) | **Ranker** 排序器 | 个性化打分、LLM 评审 |
| 下厨(切配、调味) | **Enricher** 加工层 | 正文抽取、摘要、聚类 |
| 摆盘上菜 | **Renderer** 渲染层 | Markdown → PDF |
| 送到餐桌 | **Notifier** 推送层 | 邮件 / Telegram / 飞书 |
| 记账本(今天捞了多少) | **Observability** | 日志、健康检查、告警 |

### 1.2 架构图

```text
┌─────────────────────── 采集层 (常驻 / 高频, 全天候) ───────────────────────┐
│                                                                          │
│  collectors/hotlist_*.py    ← DailyHotApi / NewsNow    微博知乎抖音B站热榜  │
│  collectors/rss_*.py        ← RSSHub 自建             公众号/UP主/博主/新番 │
│  collectors/targeted_*.py   ← MediaCrawler            指定关键词/账号/帖子   │
│  collectors/finance_*.py    ← akshare + 财经 RSS       自选股 / 宏观         │
│                                                                          │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │  统一写入 Item(唯一数据契约)
                                   ▼
                    ┌──────────────────────────────┐
                    │   Raw Store   SQLite / PG     │
                    │   items / raw_payloads        │
                    │   主键: content_hash (幂等)    │
                    └──────────────┬───────────────┘
                                   │
        ┌──────────────────────────┴───────────────────────────┐
        │           加工层 (收网时触发, 07:00 / 18:00)           │
        │                                                       │
        │  1) Dedup      SimHash / MinHash 近似去重              │
        │  2) Enrich     NewsCrawler 抽正文 → 全文入库           │
        │  3) Score      个性化打分(收藏夹向量 + 长度 + LLM)      │
        │  4) Summarize  LLM 摘要 / 事件聚类 / 观点对撞           │
        └──────────────────────────┬────────────────────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │  Renderer                     │
                    │  section_*.md  →  digest.md   │
                    │  digest.md     →  digest.pdf  │
                    └──────────────┬───────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │  Notifier: 邮件附件 / TG / 飞书 │
                    └──────────────────────────────┘

           ▲                                                    ▲
           └──────────  调度层 Scheduler (cron / APScheduler)  ──┘
```

### 1.3 五条设计原则(违反了后面一定会痛)

1. **采集与加工彻底解耦。** Collector 只做一件事:把数据塞进 Raw Store。它**不许**做去重、不许做摘要、不许决定"这条要不要上报纸"。否则你改一次排版就要动 12 个爬虫。
2. **一切幂等。** 同一条内容重复采 100 次,库里只能有 1 条。靠 `content_hash` 做主键 + `INSERT OR IGNORE`。
3. **原始 payload 必须落盘。** 你一定会改 schema。到时候能不能重放历史数据,取决于你有没有存原始响应。
4. **失败隔离。** 小红书今天挂了,报纸照常出,那一栏写"本栏目今日无数据"。**任何单点故障都不能阻断出报**——这是整个系统的可用性底线。
5. **时间是一等公民。** 每条数据必须有 `published_at`(内容发布时间)和 `fetched_at`(你抓到的时间)。两者混用是这类系统最常见的 bug 来源。

---

## 2. 数据契约(全手册最重要的一节)

你说的"分包到不同的小文件里面",工程上叫 **plugin-based collector architecture**。它能成立的唯一前提是:**所有小文件说同一种话**。

### 2.1 Item 模型

```python
# core/schema.py
from __future__ import annotations
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from enum import Enum
import hashlib, json


class Source(str, Enum):
    """数据来源平台"""
    WEIBO      = "weibo"
    ZHIHU      = "zhihu"
    BILIBILI   = "bilibili"
    DOUYIN     = "douyin"
    XHS        = "xiaohongshu"
    WECHAT_MP  = "wechat_mp"
    NEWS       = "news"
    FINANCE    = "finance"
    RSS        = "rss"
    OTHER      = "other"


class Kind(str, Enum):
    """内容形态 —— 决定它在报纸里进哪个版面"""
    HOTLIST    = "hotlist"     # 热榜条目: 有排名, 生命周期短
    ARTICLE    = "article"     # 长文: 知乎回答 / 公众号 / 深度报道
    POST       = "post"        # 短帖: 微博 / 小红书笔记
    VIDEO      = "video"       # 视频 / 新番更新
    QUOTE      = "quote"       # 行情 / 指标类结构化数据


@dataclass
class Item:
    # ---- 身份 ----
    source: Source
    kind: Kind
    title: str
    url: str

    # ---- 内容 ----
    summary: str | None = None          # 平台给的摘要 / 首段
    content: str | None = None          # 正文全文, 由 Enricher 回填
    author: str | None = None
    author_id: str | None = None

    # ---- 时间 ----
    published_at: datetime | None = None
    fetched_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

    # ---- 信号 ----
    rank: int | None = None             # 热榜名次
    heat: float | None = None           # 热度值 / 阅读量 / 点赞
    tags: list[str] = field(default_factory=list)

    # ---- 溯源 ----
    collector: str = ""                 # 哪个采集器产出的, 如 "hotlist_weibo"
    raw: dict = field(default_factory=dict)   # 原始响应, 用于重放

    # ---- 派生 ----
    content_hash: str = ""

    def compute_hash(self) -> str:
        """去重主键。注意: 只用稳定字段, 不要把 rank / heat 算进去,
        否则同一条热搜排名一变就成了新条目。"""
        basis = f"{self.source.value}|{self.normalized_url()}|{self.title.strip()}"
        return hashlib.sha256(basis.encode("utf-8")).hexdigest()[:32]

    def normalized_url(self) -> str:
        """剥掉 utm_* / share_token 等追踪参数, 否则同一篇文章会入库 N 次。
        Lab 0 的必做练习。"""
        raise NotImplementedError("Lab 0 任务: 自己实现")

    def __post_init__(self):
        if not self.content_hash:
            self.content_hash = self.compute_hash()
```

### 2.2 建库 DDL

```sql
-- core/schema.sql
CREATE TABLE IF NOT EXISTS items (
    content_hash  TEXT PRIMARY KEY,
    source        TEXT NOT NULL,
    kind          TEXT NOT NULL,
    title         TEXT NOT NULL,
    url           TEXT NOT NULL,
    summary       TEXT,
    content       TEXT,
    author        TEXT,
    author_id     TEXT,
    published_at  TEXT,          -- ISO8601 UTC
    fetched_at    TEXT NOT NULL,
    rank          INTEGER,
    heat          REAL,
    tags          TEXT,          -- JSON array
    collector     TEXT NOT NULL,
    -- 加工层回填字段
    score         REAL,
    cluster_id    TEXT,
    llm_summary   TEXT,
    used_in       TEXT           -- 已被哪期报纸使用, 如 "2026-08-05-am"
);

CREATE INDEX IF NOT EXISTS idx_items_fetched  ON items(fetched_at);
CREATE INDEX IF NOT EXISTS idx_items_source   ON items(source, kind);
CREATE INDEX IF NOT EXISTS idx_items_used     ON items(used_in);

CREATE TABLE IF NOT EXISTS raw_payloads (
    content_hash  TEXT PRIMARY KEY,
    collector     TEXT NOT NULL,
    fetched_at    TEXT NOT NULL,
    payload       TEXT NOT NULL  -- 原始 JSON, 压缩可选
);

-- 采集健康度: 用来做"今天哪张网破了"的告警
CREATE TABLE IF NOT EXISTS collector_runs (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    collector     TEXT NOT NULL,
    started_at    TEXT NOT NULL,
    finished_at   TEXT,
    status        TEXT NOT NULL,   -- ok / partial / failed
    item_count    INTEGER DEFAULT 0,
    error         TEXT
);
```

`used_in` 字段是个容易被忽略但很关键的设计:它保证**早报出现过的内容不会在晚报重复出现**。没有它,你的报纸会天天自我重复。

### 2.3 Collector 接口

```python
# core/base.py
from abc import ABC, abstractmethod
from collections.abc import Iterable
from core.schema import Item


class BaseCollector(ABC):
    name: str                # 唯一标识, 如 "hotlist_weibo"
    interval_minutes: int    # 建议采集间隔
    enabled: bool = True

    @abstractmethod
    def collect(self) -> Iterable[Item]:
        """抓取并 yield Item。
        约定:
          - 只抓不存, 存库由 runner 统一负责
          - 抛异常没关系, runner 会捕获并记入 collector_runs
          - 严禁在这里做去重 / 过滤 / 摘要
        """
        ...
```

你之后写的每一张"渔网",都只是这个类的一个子类。**这就是"分包到小文件"的正确形态。**

---

## 3. Lab 总览

| Lab | 主题 | 涉及项目 | 交付零件 | 时长 |
|---|---|---|---|---|
| 0 | 地基:数据契约与骨架 | — | `core/` + runner + CLI | 2 天 |
| 1 | 热榜聚合(只读 API) | DailyHotApi / NewsNow | `collectors/hotlist_*` | 2 天 |
| 2 | 现成监控系统拆解 | TrendRadar | 关键词配置 + 增量检测逻辑 | 2 天 |
| 3 | 万能适配器 | RSSHub | `collectors/rss_*` + 自写路由 | 4 天 |
| 4 | 定向深度采集 | MediaCrawler | `collectors/targeted_*` | 4 天 |
| 5 | 正文抽取 | NewsCrawler / trafilatura | `enrich/extract.py` | 2 天 |
| 6 | 调度与可靠性 | APScheduler / cron | `scheduler/` + 健康告警 | 3 天 |
| 7 | 个性化排序与 LLM 加工 | embedding + LLM | `pipeline/score.py` | 5 天 |
| 8 | 排版:Markdown → PDF | Typst / WeasyPrint | `render/` + 报纸模板 | 3 天 |
| 9 | 推送与部署 | Docker Compose | `notify/` + 全套编排 | 2 天 |
| ★ | Capstone:整机联调 | 全部 | 每天自动出报 | 5 天 |

---

## Lab 0 · 地基

> **🎯 目标**:在写任何爬虫之前,先把「数据往哪放、怎么放」定死。

### 步骤

**0.1 环境**

```bash
# 用 uv, 比 pip/conda 快一个数量级, 且 MediaCrawler 官方就用它
curl -LsSf https://astral.sh/uv/install.sh | sh

uv init fishnet && cd fishnet
uv add httpx pydantic python-dateutil feedparser loguru apscheduler
uv add --dev pytest ruff
```

**0.2 建立仓库骨架**

```text
fishnet/
├── core/
│   ├── schema.py        # Item 定义(§2.1)
│   ├── schema.sql       # DDL(§2.2)
│   ├── base.py          # BaseCollector(§2.3)
│   ├── store.py         # 入库 / 查询
│   └── registry.py      # 采集器自动发现
├── collectors/          # ← 你的每一张渔网, 一网一文件
│   └── __init__.py
├── enrich/              # 正文抽取、清洗
├── pipeline/            # 去重、打分、摘要
├── render/              # Markdown / PDF
├── notify/              # 推送
├── scheduler/           # 调度
├── config/
│   ├── settings.toml
│   ├── keywords.yaml    # 关注的关键词
│   └── sources.yaml     # 订阅的博主 / 公众号 / 自选股
├── data/                # SQLite + 产出的 md/pdf(加进 .gitignore)
└── main.py              # 统一 CLI 入口
```

**0.3 实现 `store.py` 的三个函数**

```python
def init_db(path: str) -> None: ...
def upsert_items(items: Iterable[Item]) -> tuple[int, int]:
    """返回 (新增数, 重复数)。用 INSERT OR IGNORE 实现幂等。"""
def query_items(since: datetime, kinds=None, unused_only=True) -> list[Item]: ...
```

**0.4 实现 `normalize_url`**

这是 Lab 0 的核心练习。要求剥离 `utm_source` / `utm_medium` / `spm` / `share_token` / `from` 等追踪参数,统一 scheme 与末尾斜杠。测一测:同一篇知乎回答从 App 分享、从网页复制、从 RSS 拿到,三个 URL 能不能归一成同一个 hash。

**0.5 写一个假的 collector 跑通全链路**

```python
# collectors/dummy.py
class DummyCollector(BaseCollector):
    name = "dummy"
    interval_minutes = 60
    def collect(self):
        yield Item(source=Source.OTHER, kind=Kind.ARTICLE,
                   title="Hello Fishnet", url="https://example.com/?utm_source=x",
                   collector=self.name)
```

```bash
uv run main.py collect --only dummy
uv run main.py stats          # 应输出: 1 items, 1 new, 0 dup
uv run main.py collect --only dummy
uv run main.py stats          # 应输出: 1 items, 0 new, 1 dup   ← 幂等验证通过
```

### ✅ 验收标准

- [ ] 连跑三次 dummy collector,库里始终只有 1 条
- [ ] `normalize_url` 通过至少 8 个 case 的 pytest
- [ ] `collector_runs` 表里有三条 `status=ok` 记录
- [ ] `main.py --help` 能列出 `collect / stats / render / push` 四个子命令(后三个先留空实现)

### 🤔 思考题

1. 如果一条微博热搜的标题被平台微调了一个字,`content_hash` 就变了,你会得到两条记录。这是 bug 还是 feature?什么场景下你希望它是 feature?
2. 为什么 `rank` 和 `heat` 不能参与 hash 计算?如果你想记录"某条热搜的排名变化曲线",表结构该怎么改?
3. `published_at` 拿不到的时候(很多平台不给),用 `fetched_at` 兜底会带来什么后果?

### ⚠️ 坑

- 时区。**所有时间统一存 UTC,只在渲染时转北京时间。**混用 naive datetime 是这类项目的头号 bug。
- SQLite 的写并发。多个 collector 并行写会 `database is locked`,开启 WAL 模式:`PRAGMA journal_mode=WAL;`

---

## Lab 1 · 热榜聚合:最便宜的第一桶鱼

> **🎯 目标**:用 DailyHotApi 在一天内让报纸有第一个真实版面。
> **对应愿景**:微博热点趋势、B 站、知乎热榜。

### 为什么先做这个

DailyHotApi 把各平台热榜统一成了 API,你**完全不用写爬虫**,只需要 HTTP GET。这是投入产出比最高的一步,先让系统跑起来产生正反馈。

### 步骤

**1.1 自部署**

```bash
docker run -d --name dailyhot -p 6688:6688 \
  -e ALLOWED_DOMAIN='*' -e ALLOWED_HOST=0.0.0.0 \
  imsyy/dailyhot-api:latest

curl http://localhost:6688/all          # 看有哪些榜单
curl http://localhost:6688/weibo
```

> 为什么自部署而不用公共实例?因为公共实例随时会挂/限流,而你的系统要跑一年。**任何你依赖的外部服务都应该假设它明天就消失。**

**1.2 写第一个真 collector**

```python
# collectors/hotlist_generic.py
class HotlistCollector(BaseCollector):
    """一个类适配 N 个榜单 —— 注意这里的抽象层次"""
    interval_minutes = 30

    def __init__(self, board: str, source: Source):
        self.board, self.source = board, source
        self.name = f"hotlist_{board}"

    def collect(self):
        r = httpx.get(f"{settings.dailyhot_url}/{self.board}", timeout=20)
        r.raise_for_status()
        data = r.json()
        for i, row in enumerate(data.get("data", []), start=1):
            yield Item(
                source=self.source, kind=Kind.HOTLIST,
                title=row["title"], url=row.get("url") or row.get("mobileUrl", ""),
                summary=row.get("desc"), heat=_to_float(row.get("hot")),
                rank=i, collector=self.name, raw=row,
                published_at=_parse_ts(row.get("timestamp")),
            )
```

然后在 `config/sources.yaml` 里声明要开哪几张网:

```yaml
hotlists:
  - {board: weibo,    source: weibo}
  - {board: zhihu,    source: zhihu}
  - {board: bilibili, source: bilibili}
  - {board: douyin,   source: douyin}
  - {board: toutiao,  source: news}
```

**1.3 加上"增量检测"**

这是热榜数据的核心玩法:你关心的不是"现在榜上有什么",而是**"什么是新上榜的"**和**"什么在快速蹿升"**。

新建一张表记录每次快照的 `(content_hash, rank, fetched_at)`,然后实现:

```python
def newly_entered(board: str, window_hours: int = 6) -> list[Item]:
    """窗口内首次出现的条目"""

def fast_rising(board: str, min_delta: int = 15) -> list[Item]:
    """排名蹿升超过阈值的条目"""
```

排名蹿升的幅度可以简单地用相邻两次快照的名次差,更稳一点的做法是用窗口内的最优名次与首次名次之差:

$$ \Delta r = r_{\text{first}} - \min_{t \in W} r_t $$

$\Delta r$ 越大,说明这条内容在窗口 $W$ 内爬升越猛,通常也就越值得进你的报纸。

**1.4 顺便看一眼 NewsNow**

不必深入部署,但**读它的源码目录 `server/sources/`**。它是"如何优雅地组织 60+ 个数据源适配器"的一份优秀范本,而这正是你接下来要面对的问题。看完问自己:我的 `collectors/` 目录该不该学它这么组织?

### ✅ 验收标准

- [ ] 至少 5 个平台的热榜稳定入库,连跑 6 小时无崩溃
- [ ] `newly_entered()` 能正确输出"过去 6 小时新上榜"的条目
- [ ] 手动 `sqlite3 data/fishnet.db "SELECT source, count(*) FROM items GROUP BY source"` 数据合理
- [ ] **产出第一个 Markdown 片段**:`render/sections/hotlist.md`,内容是"今日新上榜 Top 20"

### 🤔 思考题

1. 30 分钟采一次和 5 分钟采一次,分别会漏掉什么?对"发现新上榜"这个目标,采样间隔的下界由什么决定?
2. 热榜是典型的**幸存者偏差**数据源:你只看得到已经火的。这对你的"批判性思考内容"目标是帮助还是伤害?
3. DailyHotApi 默认有约 60 分钟的缓存。如果你每 5 分钟请求一次,实际拿到的数据新鲜度是多少?

---

## Lab 2 · 拆解 TrendRadar:向成品系统偷设计

> **🎯 目标**:不是"部署一个 TrendRadar 来用",而是**把它当作参考实现来解剖**,偷走它的关键词过滤与增量监控设计。

### 心态调整

到这里你要做一个重要判断。TrendRadar 是个完成度很高的成品,但它的定位是「热榜里的指定信息」——而你要的是一份**包含知乎长文、公众号、B 站新番、自选股、个人订阅博主的个性化报纸**。它的输出形态(推送卡片)和你的目标(PDF 报纸)也不一致。

所以合理的用法是:**读它、抄它的设计、不依赖它。**

### 步骤

**2.1 跑起来**

```bash
git clone https://github.com/sansan0/TrendRadar
cd TrendRadar
# 按 README 配置 config/config.yaml 与 frequency_words.txt
docker compose up -d
```

配一组你自己的关键词跑一天,感受它的推送节奏。

**2.2 精读三处源码**

| 读什么 | 你要回答的问题 |
|---|---|
| 关键词匹配逻辑 | 它怎么处理「必须词 / 可选词 / 排除词」?权重怎么算?**你的 `keywords.yaml` 该长什么样?** |
| 增量/新增热点检测 | 它靠什么判断"这条是新的"?和你 Lab 1 自己写的 `newly_entered` 有何异同? |
| 推送与调度配置 | 它的 GitHub Actions cron 与统一调度配置怎么组织?哪些参数是你也需要的? |

**2.3 设计你自己的关键词 DSL**

```yaml
# config/keywords.yaml
groups:
  - name: 我的自选股
    must:    ["宁德时代"]
    any:     ["财报", "定增", "订单", "产能", "调研"]
    exclude: ["股吧", "荐股", "牛股"]
    weight:  3.0
    sections: [finance]        # 决定进报纸哪个版面

  - name: AI 进展
    any:     ["大模型", "OpenAI", "Anthropic", "推理模型", "开源模型"]
    exclude: ["割韭菜", "培训班", "变现"]
    weight:  2.0
    sections: [tech]
```

`exclude` 列表是个被严重低估的功能。中文互联网的信息噪音大量来自营销号,一份好的 exclude 词表能让报纸质量提升一个档次。**从第一天就开始积累它。**

**2.4 做出取舍决策并写进 ADR**

在 `docs/adr/001-why-not-trendradar.md` 里写清楚:你最终是「直接用 TrendRadar 作为一个 collector 数据源」,还是「只借鉴设计、自己实现」?理由是什么?

> ADR(Architecture Decision Record)是工程师的基本功。三个月后你会忘记为什么这么选,而这份文档会救你。顺便,这也是求职时能拿出来讲的东西。

### ✅ 验收标准

- [ ] TrendRadar 成功推送过至少一次到你的 Telegram / 飞书 / 邮箱
- [ ] `config/keywords.yaml` 写好至少 5 个关键词组,覆盖你愿景里的财经、政治经济、AI
- [ ] `pipeline/keyword.py` 实现 must/any/exclude/weight 匹配,并有单元测试
- [ ] 写完 ADR-001

### 🤔 思考题

1. 关键词匹配是最朴素的过滤方式,它的召回率问题在哪里?(提示:「宇树」vs「Unitree」vs「四足机器人」)Lab 7 会用向量检索补上这一课,先想想为什么关键词不够。
2. TrendRadar 依赖 NewsNow 作为数据源。这种「项目依赖项目」的链条对可靠性意味着什么?你的系统里有几条这样的链?

### ⚠️ 坑

- 别在这个 Lab 上花超过两天。它的价值是**参考**,不是让你成为它的重度用户。

---

## Lab 3 · RSSHub:你愿景里最关键的一块拼图

> **🎯 目标**:用 RSSHub 一次性解决「指定公众号 / 指定 UP 主 / B 站新番 / 指定博主 / 知乎专栏」这一大类需求,并**亲手写一个 RSSHub 路由**。
> **对应愿景**:公众号更新、B 站新番、指定博主的帖子。

### 为什么这个 Lab 权重最高

回看你的愿景清单,里面「我指定的 X」占了一半以上:指定公众号、指定 UP 主、指定博主、指定自选股。这类需求的共同点是:**目标固定、更新频率低、不需要对抗搜索/推荐系统**。

对付这类需求,RSSHub 的性价比碾压一切自写爬虫。它已经把上千个网站变成了统一的 RSS 格式,你只需要 `feedparser.parse(url)`,一行搞定。

### 步骤

**3.1 自部署 RSSHub**

```yaml
# docker-compose.yml 片段
services:
  rsshub:
    image: diygod/rsshub:chromium-bundled
    ports: ["1200:1200"]
    environment:
      NODE_ENV: production
      CACHE_TYPE: redis
      REDIS_URL: "redis://redis:6379/"
    depends_on: [redis]
  redis:
    image: redis:alpine
```

同样,**必须自部署**。公共实例对热门路由限流严重,且随时可能下线。

**3.2 建立你的订阅清单**

```yaml
# config/sources.yaml
feeds:
  - {name: "某UP主投稿", url: "http://rsshub:1200/bilibili/user/video/{uid}", source: bilibili, kind: video, weight: 2.0}
  - {name: "本季新番",   url: "http://rsshub:1200/bilibili/bangumi/...",       source: bilibili, kind: video}
  - {name: "知乎某人回答", url: "http://rsshub:1200/zhihu/people/activities/{id}", source: zhihu, kind: article}
  - {name: "华尔街见闻", url: "http://rsshub:1200/wallstreetcn/news/global",   source: finance,  kind: article, weight: 2.5}
```

**3.3 写通用 RSS Collector**

```python
# collectors/rss_generic.py
class RSSCollector(BaseCollector):
    interval_minutes = 60

    def __init__(self, cfg: dict):
        self.cfg = cfg
        self.name = f"rss_{slugify(cfg['name'])}"

    def collect(self):
        feed = feedparser.parse(self.cfg["url"])
        if feed.bozo and not feed.entries:
            raise RuntimeError(f"feed broken: {feed.bozo_exception}")
        for e in feed.entries:
            yield Item(
                source=Source(self.cfg["source"]),
                kind=Kind(self.cfg.get("kind", "article")),
                title=e.title,
                url=e.link,
                summary=_strip_html(e.get("summary", ""))[:500],
                content=_extract_content(e),   # RSS 有时直接带全文, 白捡
                author=e.get("author") or self.cfg["name"],
                published_at=_parse_struct_time(e.get("published_parsed")),
                collector=self.name,
                tags=[t.term for t in e.get("tags", [])],
                raw=dict(e),
            )
```

注意这里有个便宜可以捡:**部分 RSS 源直接在 `content:encoded` 里给全文**,那你连 Lab 5 的正文抽取都省了。先检查再决定要不要请求原页面。

**3.4 微信公众号:单独说**

公众号是你愿景里技术上最麻烦的一项,因为微信没有公开的订阅接口。目前几条路,按可靠性排序:

| 方案 | 原理 | 可靠性 | 成本 |
|---|---|---|---|
| 商业 RSS 服务(wechat2rss 类) | 第三方代抓 | 较高 | 少量订阅费 |
| 自建中转(如 werss / wewe-rss 类项目) | 用读书/开放接口拿更新 | 中,需维护 | 时间 |
| 搜狗微信搜索 | 网页抓取 | 低,反爬强、内容不全 | 高 |
| 手动 + 半自动 | 你自己转发到某个入口 | 高但不自动 | 人力 |

**建议:先用 RSSHub 试,不通就上第三方服务,不要在这上面死磕两周。**这是一个明确的「可外包」问题,不是你的系统的核心竞争力。把时间留给 Lab 7。

**3.5 ★ 本 Lab 的核心练习:自己写一个 RSSHub 路由**

找一个 RSSHub 还不支持、但你需要的信息源(你们学校教务处通知、某个小众博客、某券商研报列表都行),按 RSSHub 的文档写一个路由,本地跑通。

```typescript
// lib/routes/yoursite/notice.ts
export const route: Route = {
    path: '/notice',
    categories: ['university'],
    example: '/yoursite/notice',
    handler,
};

async function handler() {
    const response = await ofetch(baseUrl);
    const $ = load(response);
    const items = $('.notice-list li').toArray().map((el) => ({
        title: $(el).find('a').text().trim(),
        link: new URL($(el).find('a').attr('href'), baseUrl).href,
        pubDate: parseDate($(el).find('.date').text()),
    }));
    return { title: '通知公告', link: baseUrl, item: items };
}
```

**为什么这一步不能跳过**:它逼你面对真实网页解析的全部细节——相对 URL、日期格式、编码、分页、反爬。这些经验在你后面调试任何爬虫时都会用上。而且这是一个可以提 PR 的真实开源贡献。

### ✅ 验收标准

- [ ] 自建 RSSHub 跑通,至少 10 个订阅源稳定出数据
- [ ] B 站新番 + 至少 2 个指定 UP 主 + 至少 1 个知乎源入库
- [ ] 公众号方案确定并至少通了 1 个号(或明确记录为「已知限制」写进 ADR)
- [ ] **自写路由 1 个,本地能返回合法 RSS XML**
- [ ] 产出 `render/sections/subscriptions.md`

### 🤔 思考题

1. RSS 只给你最新 N 条。如果某个 UP 主一天发 20 条而你 1 小时轮询一次,会不会漏?怎么设计轮询间隔与 feed 长度的关系?
2. RSSHub 的路由本质上是「网页 → 结构化数据」的适配器。这和 Lab 5 的 NewsCrawler 有什么本质区别?什么时候该用哪个?
3. 如果 RSSHub 某天彻底不可用,你的系统会损失哪几个版面?这个风险你接受吗?

### ⚠️ 坑

- RSSHub 部分路由需要登录 Cookie(知乎、微博等),配在环境变量里,**别提交进 Git**。用 `.env` + `.gitignore`。
- `feedparser` 对不规范 XML 很宽容,`feed.bozo=1` 但仍有 `entries` 是常态,别一见 bozo 就报错。

---

## Lab 4 · MediaCrawler:面对真实的反爬

> **🎯 目标**:理解「搜索式采集」与「订阅式采集」的根本差异,并建立对合规边界的判断力。
> **对应愿景**:指定关键词的全网帖子、指定博主的小红书/微博内容。

### ⚠️ 先读这段,再动手

MediaCrawler 的许可证**明确限制在学习与研究用途**,并要求不进行大规模抓取、不用于商业用途。这不是形式主义:

- 你的个人报纸系统 → 自用、低频、不再分发 → 合理
- 把抓来的内容做成公开网站/服务 → 越界
- 高并发批量拉取 → 既违反许可,也可能违反《数据安全法》与平台服务条款,还会让你的账号被封

**这个 Lab 的正确心态:学习采集技术与反爬原理,而不是建立一个大规模抓取管道。**把并发调到最低,把频率调到最慢,只抓你真正需要的少量目标。

### 步骤

**4.1 装起来跑通一次**

```bash
git clone https://github.com/NanmiCoder/MediaCrawler
cd MediaCrawler && uv sync
uv run playwright install chromium

# 从最温和的开始: 抓指定创作者
uv run main.py --platform xhs --type creator
```

**4.2 观察反爬机制(这才是本 Lab 的知识点)**

一边跑一边开浏览器 DevTools,回答:

1. **登录态怎么维持的?** Cookie 存在哪?过期了会怎样?
2. **请求签名。** 小红书/抖音的请求里那串看不懂的参数是什么?项目是怎么生成它的?(提示:找找它是不是在浏览器里执行了平台自己的 JS)
3. **为什么用 Playwright 而不是纯 requests?** 这个选择的代价是什么?(内存、速度、可部署性)
4. **限速在哪一层做的?** 如果你把并发调到 10 会发生什么?(**别真试**,想清楚就行)

把答案写进 `docs/notes/anti-crawling.md`。这份笔记是你以后面试聊爬虫时的弹药。

**4.3 接入你的系统:进程隔离**

MediaCrawler 依赖重(Playwright + Chromium),不适合直接 import 进你的主进程。用**子进程 + 文件交换**的方式集成:

```python
# collectors/targeted_xhs.py
class XHSCreatorCollector(BaseCollector):
    interval_minutes = 360     # 6 小时一次, 已经足够, 别更快

    def collect(self):
        out = Path("data/mc_out") / f"{self.name}_{ts()}.json"
        subprocess.run(
            ["uv", "run", "main.py", "--platform", "xhs",
             "--type", "creator", "--save_data_option", "json"],
            cwd=MC_HOME, timeout=600, check=True,
        )
        for row in _read_latest_json(MC_HOME / "data"):
            yield Item(
                source=Source.XHS, kind=Kind.POST,
                title=row["title"], url=row["note_url"],
                summary=row.get("desc"), author=row.get("nickname"),
                author_id=row.get("user_id"),
                heat=float(row.get("liked_count", 0) or 0),
                published_at=_ms_to_dt(row.get("time")),
                collector=self.name, raw=row,
            )
```

> 这个「重依赖跑在子进程 / 独立容器里,通过文件或消息队列交换数据」的模式,是集成异构爬虫的标准做法。学会它比学会 MediaCrawler 本身更有价值。

**4.4 决策:你真的需要它吗?**

诚实评估:你的愿景里,有哪几项是**只能**靠 MediaCrawler 完成的?

- 「指定博主发的帖子」→ 如果博主在 B 站/知乎/微博,RSSHub 就够了。只有小红书/抖音必须用它。
- 「知乎优质内容」→ RSSHub + 你自己的排序,比搜索式抓取更合适。

结论可能是:**MediaCrawler 在你的系统里只负责 1–2 个小众版面。**那就把它放在低优先级,别让它拖累整体进度。把这个判断写进 ADR-002。

### ✅ 验收标准

- [ ] 成功抓取 1 个指定创作者的内容并转成 Item 入库
- [ ] `docs/notes/anti-crawling.md` 回答完 4.2 的四个问题
- [ ] 采集频率配置 ≥ 6 小时,并发数 = 1
- [ ] ADR-002 写清 MediaCrawler 在你系统中的定位与边界

### 🤔 思考题

1. 「订阅式采集」(RSS,目标已知)与「搜索式采集」(关键词,目标未知)在**风险、成本、数据质量**三个维度上各有什么差异?
2. 平台的 robots.txt、服务条款、《数据安全法》《个人信息保护法》,这几者的约束力有什么不同?个人自用与对外服务的界线在哪?
3. 如果某个平台明确禁止抓取,而你的报纸又很需要它——你的替代方案是什么?(手动、官方 API、放弃?)

---

## Lab 5 · 正文抽取:从标题到内容

> **🎯 目标**:把热榜/RSS 里的一条链接变成可供 LLM 阅读的干净正文。
> **对应愿景**:当天新闻、财经新闻的**实质内容**,而不只是标题。

### 为什么必须有这一层

热榜给你的是标题 + 链接。但「一份报纸」如果只有标题,那就是个书签列表。LLM 也没法只凭标题写出有价值的摘要。**正文抽取是从「信息聚合」升级到「情报产品」的分水岭。**

### 步骤

**5.1 三条路线对比,自己测**

| 方案 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| `trafilatura` | 通用启发式 + DOM 密度 | 一行调用、无需适配、覆盖广 | 特殊站点会翻车 |
| `NewsCrawler` | 平台专用适配器 | 微信/头条/网易等国内站准确 | 只覆盖已适配平台 |
| 自写规则 | CSS 选择器 | 完全可控 | 网站改版就废 |

建一个 20 条 URL 的测试集(覆盖新闻站、公众号、知乎、财经站),三种方案各跑一遍,人工评分。**这是你第一次做「工程选型的量化对比」,是很好的练习。**

```python
# enrich/extract.py
def extract(url: str, html: str | None = None) -> ExtractResult:
    """按站点路由到不同抽取器, 通用兜底 trafilatura"""
    for matcher, extractor in _REGISTRY:
        if matcher(url):
            return extractor(url, html)
    return _trafilatura_extract(url, html)
```

**5.2 加缓存与礼貌**

```python
# 必须有的三件事
# 1. 本地 HTML 缓存: 同一 URL 24h 内不重复请求
# 2. 请求间隔: 同域名至少 sleep 1~2s
# 3. 合理 UA + 遵守 robots.txt
```

**5.3 质量兜底**

抽取失败(正文 < 200 字、全是导航文字)时,**降级为只用标题 + summary**,不要让报纸出现空白条目。写一个 `content_quality_score()` 打分,低于阈值就标记 `content=None`。

### ✅ 验收标准

- [ ] 20 条测试 URL 的抽取成功率 ≥ 80%,失败的能正确降级
- [ ] 缓存生效:重复调用同 URL 不发起网络请求
- [ ] `enrich/extract.py` 可以被 pipeline 直接调用,回填 `items.content`

### 🤔 思考题

1. 付费墙、需要登录、纯图片的公众号文章——各该怎么处理?
2. 抽取正文后,你实际上在本地存储了一份他人的完整作品。个人自用和公开分发在版权上的区别是什么?你的 PDF 里应该放全文还是摘要 + 链接?**(这个问题请认真想,它会影响你的渲染层设计。)**

---

## Lab 6 · 调度与可靠性:让渔网真的挂在海里

> **🎯 目标**:实现你说的「持续不断运行 + 早晚收网」,并让它在你不看的时候也不会悄悄死掉。

### 核心设计:两种节奏

```text
撒网(持续)                          收网(定时)
─────────────                        ─────────────
每 30 分钟  热榜采集                  07:00  → 早报 pipeline
每 60 分钟  RSS 采集                  07:30  → PDF 推送
每 6 小时   定向采集                  18:00  → 晚报 pipeline
每 6 小时   正文抽取(补齐 content)    18:30  → PDF 推送
```

### 步骤

**6.1 选调度器**

| 方案 | 适合 | 不适合 |
|---|---|---|
| **APScheduler**(推荐起步) | 单机、Python 原生、代码内配置 | 多机、复杂依赖 |
| cron + CLI | 极简、系统级可靠 | 任务间依赖难表达 |
| n8n | 可视化、易接通知 | 逻辑复杂后难维护 |
| Airflow / Prefect | 有 DAG 依赖、要重跑历史 | 单人项目过重 |

**给你的建议:APScheduler 起步,`main.py` 保持可单独调用**。这样你既有常驻进程,又能手动 `uv run main.py render --edition am` 调试,不用等到明天早上。

```python
# scheduler/run.py
sched = BlockingScheduler(timezone="Asia/Shanghai")

for c in registry.all_collectors():
    sched.add_job(run_collector, "interval", minutes=c.interval_minutes,
                  args=[c], id=c.name, max_instances=1,
                  jitter=120,          # 打散, 避免整点齐发
                  coalesce=True)       # 错过的任务只补一次, 不堆积

sched.add_job(produce_edition, "cron", hour=7,  minute=0, args=["am"])
sched.add_job(produce_edition, "cron", hour=18, minute=0, args=["pm"])
```

`jitter` 和 `coalesce` 这两个参数是新手最常漏的:没有 jitter,你的所有采集器会在整点同时发请求,像一次小型 DDoS;没有 coalesce,机器休眠一晚醒来会瞬间补跑几十个任务。

**6.2 每个 collector 包一层「安全壳」**

```python
def run_collector(c: BaseCollector):
    run_id = store.start_run(c.name)
    try:
        items = list(_with_timeout(c.collect, seconds=300))
        new, dup = store.upsert_items(items)
        store.finish_run(run_id, "ok", len(items))
    except Exception as e:
        store.finish_run(run_id, "failed", 0, error=repr(e))
        logger.exception(f"[{c.name}] failed")
        # ★ 关键: 绝不向上抛。一张网破了, 其他网继续捞。
```

**6.3 健康监控(这个 Lab 的高分项)**

写一个 `pipeline/health.py`,在每次出报时检查:

- 哪些 collector 在过去 24h 内**从未成功**?
- 哪些 collector 的产出量比 7 日均值低 80% 以上?(说明可能被限流或页面改版了)
- 数据库大小、最旧未处理数据的年龄

把结果作为**报纸的最后一页「系统体检」**打印出来。这样你每天读报时顺便就完成了运维巡检——非常优雅,也是这个项目区别于玩具的地方。

### ✅ 验收标准

- [ ] 常驻进程连续跑 72 小时不崩,期间故意 kill 掉 RSSHub 容器,系统仍能出报
- [ ] 早晚两次 pipeline 按时触发,`used_in` 正确标记,早报内容不在晚报重复
- [ ] 「系统体检」页能正确报出你手动制造的故障
- [ ] `main.py` 每个子命令都能独立手动执行

### 🤔 思考题

1. 你的笔记本会关机。这套系统最终该跑在哪?(树莓派 / 云服务器 / GitHub Actions?)三者在成本、可靠性、IP 环境上各有什么取舍?
2. 如果早上 7 点 pipeline 跑到一半崩了,你希望发生什么?重试?出一份残缺的报?还是不出报只告警?
3. GitHub Actions 免费额度能跑这个系统吗?用它跑爬虫有什么隐患?

---

## Lab 7 · 个性化排序:把「信息堆」变成「我的报纸」★ 核心 Lab

> **🎯 目标**:实现你愿景里技术含量最高、也最独特的两条——「和我收藏夹风格/长度相似的知乎内容」「引起批判性思考的内容」。
> **这是整个项目里唯一没有现成开源方案的部分,也是你真正的作品。**

前面六个 Lab 你都在组装别人的零件。这个 Lab 开始,你在造别人没有的东西。**如果时间不够,砍掉 Lab 4,也不要砍这个。**

### 7.1 问题拆解

「每天早上给我一份报纸」的真正难点不是采集,是**每天 3000 条候选,只能上 30 条,选哪 30 条?**

打分函数是系统的大脑:

$$ S(x) = w_1 \cdot S_{\text{sim}}(x) + w_2 \cdot S_{\text{len}}(x) + w_3 \cdot S_{\text{llm}}(x) + w_4 \cdot S_{\text{hot}}(x) + w_5 \cdot S_{\text{kw}}(x) - w_6 \cdot P_{\text{dup}}(x) $$

逐项来做。

### 7.2 $S_{\text{sim}}$:收藏夹相似度

这是你需求里最有意思的一条。做法:

**Step 1 · 导出你的知乎收藏夹**,拿到 N 篇你亲自认可的文章正文(用 RSSHub 的收藏夹路由,或手动导出)。这是你的**黄金标注集**,越大越准,50 篇起步。

**Step 2 · 向量化。** 用中文 embedding 模型(`bge-large-zh-v1.5` / `text2vec` 本地跑,或调用 API):

$$ \mathbf{e}_i = \text{Embed}(\text{title}_i \,\|\, \text{content}_i[:2000]) $$

**Step 3 · 别只算一个质心。** 最朴素的做法是取平均:

$$ \mathbf{c} = \frac{1}{N}\sum_{i=1}^{N}\mathbf{e}_i $$

但你的兴趣不是一个点,是**好几簇**(比如「技术深度」「社会观察」「历史」)。平均之后会落在几簇中间的空地上,谁都不像。正确做法是先对收藏夹做 K-means 聚类得到 $k$ 个兴趣簇心 $\mathbf{c}_1,\dots,\mathbf{c}_k$,然后取**最近簇**的相似度:

$$ S_{\text{sim}}(x) = \max_{j \in [1,k]} \frac{\mathbf{e}_x \cdot \mathbf{c}_j}{\|\mathbf{e}_x\| \, \|\mathbf{c}_j\|} $$

> 这一步的思路差异会直接体现在推荐质量上,值得你自己做实验对比一下 `mean` 和 `max-over-clusters` 两种方案的效果。

**Step 4 · 落地检索。** 用 `sqlite-vec` 或 `chromadb` 存向量,别自己撸暴力检索。

### 7.3 $S_{\text{len}}$:长度偏好

你说「和收藏夹里文章长度相似」。统计你收藏夹正文长度的对数分布,取均值 $\mu$ 和标准差 $\sigma$,然后用对数正态的形状打分:

$$ S_{\text{len}}(x) = \exp\left(-\frac{(\ln L_x - \mu)^2}{2\sigma^2}\right) $$

$L_x$ 是候选文章字数。这样长度正好落在你舒适区的得 1 分,过短的水贴和过长的裹脚布都自然衰减。用对数而不是原始长度,是因为文章长度天然是重尾分布。

### 7.4 $S_{\text{llm}}$:批判性思考评分

「引起批判性思考的内容」没法用关键词匹配,只能让 LLM 当评委。关键是**写好 rubric**:

```python
CRITICAL_RUBRIC = """
你是一位严格的内容评审。请对下面这篇文章在「激发批判性思考」维度上打分(0-10)。

高分特征:
- 提出了与主流叙事相左的观点, 并给出可检验的论据
- 揭示了一个被普遍忽略的前提假设或概念混淆
- 呈现了同一事实的多种解释, 而非单一结论
- 包含具体数据、原始文献或一手观察, 而非二手转述
- 作者明确标注了自己论证的边界与不确定性

低分特征:
- 情绪煽动、结论先行、诉诸群体认同
- 营销软文、标题党、伪科普
- 纯资讯播报, 无分析增量
- 观点正确但论证空洞, 只是重复常识

只输出 JSON: {"score": <0-10>, "reason": "<40字以内>", "angle": "<这篇挑战了什么假设>"}
"""
```

**成本控制很重要**:别对 3000 条全跑 LLM。做**两阶段召回**——先用便宜的信号(关键词 + 相似度 + 热度)粗排出 Top 150,再对这 150 条跑 LLM 精排。这是工业推荐系统的标准架构(召回 → 粗排 → 精排),你在这里会亲手实现一遍。

### 7.5 $P_{\text{dup}}$:去重惩罚

同一件事会有 20 家媒体报道。你的报纸只该出现一次。

- **精确去重**:`content_hash`(Lab 0 已做)
- **近似去重**:SimHash 汉明距离 < 3,或标题 embedding 余弦 > 0.92
- **事件聚类**:对当日候选做层次聚类,每个簇只出**分数最高的一条**作为主稿,其余作为「相关报道」折叠展示

事件聚类做出来之后,你的报纸会突然从「链接列表」变成「有编辑思路的版面」,观感提升极大。

### 7.6 反馈闭环(加分项)

在生成的 PDF 里给每条内容一个短链或编号,你读完点一下「有用/无用」,写回数据库。累积两周后,用这些标注重新训练权重 $w_i$(逻辑回归就够)或扩充你的黄金集。

**这个闭环一旦跑起来,你的系统就从「规则引擎」变成了「会学习的推荐系统」**,而且是一个只服务你一个人、数据完全私有的推荐系统。这也是这个项目最值得写进简历的部分。

### ✅ 验收标准

- [ ] 收藏夹黄金集 ≥ 50 篇并完成向量化
- [ ] 实现两阶段召回,LLM 调用量每期 ≤ 150 次
- [ ] 事件聚类生效:同一新闻的多家报道被折叠
- [ ] 做一次 **A/B 自评**:纯热度排序 vs 你的打分函数,各出一期报纸,人工盲评哪期更想读
- [ ] 反馈按钮闭环打通(至少能记录)

### 🤔 思考题

1. 你的黄金集来自过去的收藏,系统会不断推给你「过去喜欢的东西」。这就是**信息茧房的形成机制**。你打算怎么在打分函数里留一个「探索」的口子?(提示:$\epsilon$-greedy,或强制留 3 个版位给低相似度但高 LLM 分的内容)
2. LLM 评委本身有偏见——它可能系统性地偏爱某种文风。怎么检测?
3. 权重 $w_i$ 怎么定?拍脑袋、网格搜索、还是从反馈数据学?第一版你会选哪个,为什么?

---

## Lab 8 · 渲染:Markdown → 一份像样的报纸

> **🎯 目标**:产出你愿景的最终形态——一个**排版舒服、愿意在早餐时读完**的 PDF。

### 8.1 中文 PDF 排版方案选型

这是国内开发者的经典坑,先看清楚再动手:

| 方案 | 链路 | 中文支持 | 排版能力 | 上手难度 |
|---|---|---|---|---|
| **Typst**(推荐) | md → typ → pdf | 好,字体配置简单 | 强,支持多栏、脚注 | 中,但语法比 LaTeX 友好太多 |
| Pandoc + XeLaTeX | md → tex → pdf | 好,但字体配置繁琐 | 最强 | 高,报错难懂 |
| **WeasyPrint**(推荐) | md → html+css → pdf | 好 | 会 CSS 就会排版 | 低 |
| Playwright 打印 | html → pdf | 好 | 同 CSS,但分页控制弱 | 低 |

**建议**:你会 CSS 就选 **WeasyPrint**(能直接用 `@page`、`column-count`、`break-inside: avoid` 做真正的报纸多栏排版);想学点新东西、追求排版质量就选 **Typst**。

### 8.2 分层渲染架构

```text
pipeline 输出结构化数据
        │
        ▼
render/sections/*.j2      ← 每个版面一个 Jinja2 模板
        │
        ├── 01_headline.md      头版: 今日最重要 3 条 + LLM 综述
        ├── 02_hotlist.md       热榜速览: 新上榜 / 蹿升
        ├── 03_deepread.md      深度阅读: 知乎/公众号长文(打分 Top N)
        ├── 04_finance.md       财经: 自选股 + 宏观 + 市场
        ├── 05_tech.md          科技/AI
        ├── 06_subscribe.md     订阅更新: UP主 / 新番 / 博主
        ├── 07_critical.md      「今日一问」: LLM 分最高的批判性内容
        └── 99_health.md        系统体检
        │
        ▼
digest_2026-08-05_am.md   ← 合并, 加页眉页脚元信息
        │
        ▼
digest_2026-08-05_am.pdf
```

**为什么中间必须有 Markdown 这一层**(而不是直接生成 PDF):

1. Markdown 可以直接进你的笔记软件(Obsidian / Logseq),PDF 不行
2. 调试排版时不用重跑 pipeline
3. 出错时你能一眼看出是数据问题还是排版问题

**你在愿景里说的这个设计是对的,坚持它。**

### 8.3 报纸设计要点

- **头版必须有「今日综述」**:让 LLM 读完当日 Top 20,写 200 字总述。这是把「信息列表」变成「报纸」的关键一笔。
- **早报晚报差异化**:早报重「昨夜今晨发生了什么」(时效);晚报重「深度阅读 + 收藏级长文」(适合饭后慢读)。这个差异应该体现在打分权重上,不只是内容不同。
- **信息密度**:每条给标题 + 2 行摘要 + 来源 + 二维码/短链。全文不要塞进 PDF(见 Lab 5 思考题 2)。
- **可扫描性**:早餐时你只有 5 分钟。用加粗、分隔、固定版序,让眼睛能跳读。
- **元信息页脚**:期号、生成时间、候选总数 → 入选数。这个数字每天看能让你直观感受系统状态。

### ✅ 验收标准

- [ ] 生成的 PDF 中文无乱码、无豆腐块,标点挤压正常
- [ ] 至少 5 个版面,早晚报模板有实质差异
- [ ] 单页排版不出现孤行/断表,长标题正确换行
- [ ] 从 `digest.md` 到 PDF 的渲染在 30 秒内完成
- [ ] **把连续 3 天的 PDF 打出来给同学看,问他愿不愿意订阅**(这是最真实的验收)

### 🤔 思考题

1. PDF 的优点是排版固定、适合打印和长期归档;缺点是不能点击跳转体验差、手机上阅读不友好。要不要同时输出一个 EPUB 或响应式 HTML?
2. 如果某天只采到 5 条内容,报纸该怎么呈现?(空版面比错误更伤害体验)

---

## Lab 9 · 推送与部署

> **🎯 目标**:让它真正每天出现在你面前,并且能长期无人值守运行。

### 步骤

**9.1 推送通道选型**

| 通道 | 附件 PDF | 移动端体验 | 配置成本 |
|---|---|---|---|
| **邮件(SMTP)** | ✅ 原生 | 好,可归档可搜索 | 低,推荐主通道 |
| Telegram Bot | ✅ | 很好 | 低,但国内需网络条件 |
| 飞书/企业微信机器人 | ⚠️ 需上传接口 | 好 | 中 |
| 自建静态站 + RSS | 链接 | 一般 | 中,但历史归档最好 |

**建议组合:邮件发 PDF 附件(主) + Telegram/飞书发一条摘要卡片(提醒)**。

**9.2 Docker Compose 全家桶**

```yaml
services:
  fishnet:                    # 主程序: 调度 + 采集 + pipeline + 渲染
    build: .
    volumes: ["./data:/app/data", "./config:/app/config"]
    env_file: .env
    depends_on: [rsshub, dailyhot]
    restart: unless-stopped
  rsshub:      { image: diygod/rsshub:chromium-bundled, restart: unless-stopped }
  dailyhot:    { image: imsyy/dailyhot-api:latest,      restart: unless-stopped }
  redis:       { image: redis:alpine,                    restart: unless-stopped }
```

**9.3 长期运行的三件小事**

1. **数据库定期归档**:超过 90 天的 items 移到 `items_archive`,否则一年后查询会变慢
2. **磁盘水位告警**:HTML 缓存和 PDF 会一直涨
3. **秘钥管理**:所有 Cookie/API Key 走 `.env`,`.gitignore` 里写死,**别把仓库开源了才发现 Cookie 提交上去了**

### ✅ 验收标准

- [ ] `docker compose up -d` 一条命令拉起全部服务
- [ ] 邮箱连续 7 天收到早晚两报
- [ ] 冷启动测试:删掉容器全部重建,系统能自愈并继续出报
- [ ] README 写清部署步骤,让别人(或三个月后的你)能照着装起来

---

## ★ Capstone · 整机联调与复盘

### 交付清单

- [ ] 系统连续无人值守运行 **14 天**,出报成功率 ≥ 95%
- [ ] 报纸覆盖你愿景里至少 **6 类内容**:新闻 / 知乎深度 / 公众号 / B站 / 微博热点 / 财经自选股
- [ ] 一份 `docs/RETRO.md` 复盘,回答:
  - 14 天里出过哪些故障?根因是什么?
  - 哪个版面你每天都读?哪个版面你从来跳过?(**要敢删掉没人读的版面**)
  - 打分函数的哪一项贡献最大?
  - 如果重做一遍,你会怎么改架构?
- [ ] 一页架构图 + 一份 5 分钟的项目讲解稿

### 为什么要求 14 天

三天能跑通的系统和三十天不出问题的系统,是两个物种。**只有让它真正跑起来一段时间,你才会遇到证书过期、Cookie 失效、页面改版、磁盘写满这些真实的工程问题**——而这些正是把「课程作业」和「工程能力」区分开的东西。

---

## 附录 A · 合规与伦理红线

在你敲第一行代码前请读完,这不是免责声明,是真会影响你的东西。

**技术红线**
- 遵守 `robots.txt`;设置真实可辨识的 User-Agent
- 单域名请求间隔 ≥ 1s,并发 ≤ 2;**你在做个人订阅,不是做搜索引擎**
- 尊重 `429` / `Retry-After`,指数退避
- 不绕过付费墙,不破解加密接口去获取本该付费的内容

**法律红线(中国大陆)**
- 不采集、不存储个人信息(手机号、身份证、住址、精确定位)
- 不抓取明确标注禁止爬取的内容
- 不对外提供服务、不二次分发抓取到的原文——**「个人自用」是你合规的基石,一旦对外分发,性质完全改变**
- 注意《数据安全法》《个人信息保护法》《网络安全法》与各平台服务条款

**给你的一条实践建议**:在 README 顶部写明「本项目仅供个人学习与自用信息聚合,不对外提供服务」。这既是自我约束,也是你日后开源它时的必要声明。

---

## 附录 B · 30 天进度表

| 周 | 天 | 内容 | 里程碑 |
|---|---|---|---|
| W1 | 1–2 | Lab 0 地基 | 幂等入库跑通 |
| W1 | 3–4 | Lab 1 热榜 | **第一个真实版面** |
| W1 | 5–6 | Lab 2 TrendRadar | 关键词体系 |
| W1 | 7 | 缓冲 / 补作业 | |
| W2 | 8–11 | Lab 3 RSSHub ★ | 订阅版面 + 自写路由 |
| W2 | 12–14 | Lab 5 正文抽取 | 内容有正文了 |
| W3 | 15–17 | Lab 6 调度 | **系统开始 7×24 运行** |
| W3 | 18–21 | Lab 8 渲染 | **第一份 PDF 到手** |
| W4 | 22–26 | Lab 7 个性化 ★★ | 报纸开始「懂你」 |
| W4 | 27–28 | Lab 9 部署 | 一键起服务 |
| W5 | 29–30+ | Lab 4 + Capstone | 14 天稳定运行 |

**注意这个顺序不是 Lab 编号顺序**。刻意把 Lab 8(渲染)提到 Lab 7 前面,是为了让你**尽早拿到实物 PDF**——看得见的产出会显著提升你坚持下去的概率。Lab 4(MediaCrawler)排在最后,因为它是最可能被砍掉的一环。

---

## 附录 C · Vibe Coding 提示词模板

### C.1 每次开新对话必贴的上下文

```markdown
我在做一个个人信息聚合系统 fishnet, 架构如下:
- 采集层: 每个 collector 继承 BaseCollector, 只 yield Item, 不做去重/过滤/摘要
- 存储层: SQLite, 主键 content_hash, INSERT OR IGNORE 保证幂等
- 加工层: 去重 → 正文抽取 → 打分 → LLM 摘要
- 渲染层: Jinja2 → Markdown → PDF

Item 定义如下(严格遵守, 不要改字段名):
<粘贴 core/schema.py>

现在请帮我实现: <具体任务>
要求:
1. 只写这一个文件, 不要修改 core/ 下的任何东西
2. 网络请求必须有 timeout 和重试
3. 解析失败时抛异常, 不要静默返回空列表
4. 附一个 pytest 测试用例
```

**最后两条要特别强调。**AI 写爬虫时最爱的操作就是 `try: ... except: return []`——页面改版后你的报纸悄悄少了一个版面,而你三周后才发现。**静默失败是这类系统的头号杀手。**

### C.2 任务切分原则

| ✅ 好的粒度 | ❌ 坏的粒度 |
|---|---|
| 「写一个 collectors/rss_generic.py,输入 feed 配置,yield Item」 | 「帮我做一个新闻聚合系统」 |
| 「给 extract.py 加一个微信公众号专用抽取器」 | 「把爬虫和渲染都写了」 |
| 「实现 max-over-clusters 的相似度打分,输入向量输出分数」 | 「做个推荐算法」 |

**判据:一个任务对应一个文件、一个明确输入输出、能写一个测试。**超过这个粒度,AI 的输出质量会断崖下跌,而且你会失去对代码的理解。

### C.3 你必须亲手写、不要交给 AI 的部分

- `core/schema.py` 的字段设计
- 打分函数的权重与结构
- 关键词表与 exclude 词表
- 报纸的版面结构

**理由**:这些是「你的品味」,不是「代码」。AI 能替你写实现,但替不了你决定什么是好内容——**而这恰恰是这个项目唯一不可替代的价值**。

---

## 附录 D · 项目 GitHub 索引

| 项目 | 地址 | 在本手册的角色 |
|---|---|---|
| TrendRadar | https://github.com/sansan0/TrendRadar | Lab 2 参考实现 |
| MediaCrawler | https://github.com/NanmiCoder/MediaCrawler | Lab 4 定向采集 |
| NewsNow | https://github.com/ourongxing/newsnow | Lab 1 架构参考 |
| RSSHub | https://github.com/DIYgod/RSSHub | Lab 3 核心依赖 |
| DailyHotApi | https://github.com/imsyy/DailyHotApi | Lab 1 热榜数据源 |
| NewsCrawler | https://github.com/NanmiCoder/NewsCrawler | Lab 5 正文抽取 |
| Agent-Reach | https://github.com/Panniantong/Agent-Reach | 扩展方向 |

---

## 最后一句

这个项目做完,你会同时拥有:

1. 一个**每天真的在用**的产品(比 90% 的课程项目更有说服力)
2. 一段能讲清楚的完整工程经历:数据契约、异构系统集成、可靠性、召回排序两阶段架构、合规判断
3. 一份持续增长的个人语料库,以后想做什么信息类应用都能直接接上

面试时,「我写过爬虫」和「我有一个跑了半年的个人情报系统,每天早上给我出一份 PDF,这是它的架构图」——完全是两个量级的东西。

祝顺利。开始撒网吧。
