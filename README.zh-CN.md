# CoreClaw Agent Skill

[English](README.md) | [简体中文](README.zh-CN.md)

这是一个面向 AI Agent 的 CoreClaw 工作流仓库。它不是 SDK，而是一套给 Agent 使用的操作说明和接口契约，帮助 Agent 稳定完成 CoreClaw 的完整使用链路：找到合适的 scraper、读取 live schema、发起运行、跟踪状态、读取结果、导出文件、查看日志、重跑任务，以及中止任务。

## 这个仓库是做什么的

一句话概括：

**让 Agent 可以稳定地使用 CoreClaw，把“找 scraper → 跑任务 → 取结果 → 处理异常”整条链路串起来。**

你可以把它理解为：

- 给 Agent 的 CoreClaw 使用手册
- 给 Agent 的接口调用参考
- 给用户看的使用说明和真实案例集合

## 仓库内容

```text
.
├── CHANGELOG.md
├── LICENSE
├── README.md
├── README.zh-CN.md
├── REGRESSION.md
├── SKILL.md
├── SUPPORT.md
└── openapi.json
```

这些文件分别用于：

- `README.md` / `README.zh-CN.md`：给人的说明文档和案例
- `SKILL.md`：告诉 Agent 应该按什么工作流使用 CoreClaw
- `openapi.json`：告诉 Agent 可以调用哪些接口、参数怎么组织
- `REGRESSION.md`：给维护者使用的冒烟检查和发布前回归清单
- `SUPPORT.md`：说明支持边界和问题归类方式
- `CHANGELOG.md`：记录文档与契约的版本变化
- `LICENSE`：仓库许可协议

## 它适合哪些事情

这个 skill 适合让 Agent 处理结构化采集任务，例如：

- 电商商品列表采集
- 电商商品详情采集
- 新闻、博客、公告信息采集
- 地图、商家、目录类信息收集
- 批量运行 scraper 并导出结果文件
- 采集失败后继续查看日志和运行详情
- 对历史任务重跑、复查、导出和状态跟踪

## 怎么使用

### 1. 放进支持 skills 的 Agent 环境

如果你的宿主环境支持 skills 目录，把整个仓库放进去即可。

例如：

```bash
~/.codex/skills/coreclaw
```

核心入口文件是：

```bash
SKILL.md
```

如果你的宿主环境不支持自动加载 skills，也可以把下面两份文件手动提供给 Agent：

- `SKILL.md`
- `openapi.json`

### 2. 准备运行条件

你需要：

- `curl`
- `jq`
- `CORECLAW_API_KEY`
- 能访问 `https://openapi.coreclaw.com`

设置 API Key：

```bash
export CORECLAW_API_KEY="your_api_key"
```

### 3. 直接把任务交给 Agent

下面这些话都可以直接对 Agent 说：

- “帮我找一个抓 Amazon 商品列表的 CoreClaw scraper。”
- “读取这个 scraper 的 live 参数，然后帮我跑一次。”
- “用异步模式启动这个 scraper，保存 `run_slug` 并持续轮询状态。”
- “先展示前 20 条结果，如果结果很多就导出 CSV。”
- “帮我分析这次 run 为什么失败，并提取错误日志。”
- “把这个历史 run 重新跑一次。”

## 推荐使用流程

一个典型的使用流程通常是：

1. 找到合适的 scraper
2. 读取 scraper 的 live detail
3. 按 live schema 组织输入参数
4. 发起 sync 或 async 运行
5. 用 `run_slug` 跟踪状态
6. 读取结果或导出文件
7. 有异常时继续查看 detail 和日志

## live `custom` 结构怎么理解

对每个 scraper 来说，都要明确区分两层东西：

- `GET /api/scraper?slug=...` 返回的 `data.parameters.custom`，它是 live schema / 参数描述结构
- `POST /api/v1/scraper/run` 里要传的 `input.parameters.custom`，它是你根据上面 schema 组织出来的实际请求值

这点非常关键：

- 不要把 `data.parameters.custom` 原样塞进 `scraper/run`
- 不要把 `startURLs`、`keyword`、`url` 之类字段写死
- 每次都应该先读取当前 scraper 的 detail，再按 live schema 组织 payload

你可以直接通过公开接口 `GET /api/scraper` 查看这个结构。**仅为了获取 live `custom` 结构，不需要 API Key，也不需要控制台登录。**

最安全的理解方式是：

1. 先用 `/api/store` 找 scraper
2. 再用 `/api/scraper` 读取 live contract
3. 检查 `data.parameters.custom`
4. 按它定义出来的字段去组织 `input.parameters.custom`
5. 再发起运行

### 代表性的 live schema 形态

下面这些公开 scraper 契约能直接说明，为什么 `custom` 不能被当成固定结构。

#### 形态 A：扁平关键词表单

代表 scraper：

- `01KPAFF816MDMRD5MSCH2SBT68` — `Google Search By KeyWord`

live schema 特征：

- `custom` 是一个扁平对象
- 必填字段是 `keyword`
- 可选字段包括 `start`、`domain`、`gl`、`hl`、`location`、`safe` 等

按 live schema 组织出来的最小 payload 可以是：

```json
{
  "keyword": "openai"
}
```

#### 形态 B：数组型请求项，每项包含多字段

代表 scraper：

- `01KPD6M5YQADCQKGVKPDZVYC63` — `Google Map Details By Keyword`

live schema 特征：

- `custom.url` 是一个数组
- 数组里的每一项都是结构化对象
- 必填项字段包括 `base_location` 和 `keyword`
- 可选项字段包括 `lang`、`max_results`、`fetch_reviews`、`max_reviews_per_place`

按 live schema 组织出来的最小 payload 可以是：

```json
{
  "url": [
    {
      "base_location": "Singapore",
      "keyword": "coffee shop"
    }
  ]
}
```

#### 形态 C：明细 URL 数组

代表 scraper：

- `01KPD6M5YVHWCNQCRK32BD02TP` — `Google Map Detail By Detail URL`

live schema 特征：

- `custom.url` 同样是数组
- 每一项代表一个明细页请求
- 必填项字段是 `detail_url`
- 可选项字段包括 `lang`、`max_reviews_per_place`

按 live schema 组织出来的最小 payload 可以是：

```json
{
  "url": [
    {
      "detail_url": "https://www.google.com/maps/place/..."
    }
  ]
}
```

关键点不在于上面这些字段名本身，而在于：这些字段名来自当次 live `/api/scraper` 响应，不同 scraper 可能会暴露完全不同的结构。

## 真实使用案例

下面这些案例都可以直接作为你的使用参考，也可以直接复制成提示词交给 Agent。

### 案例 1：抓 Amazon 商品列表

目标：按关键词搜索 Amazon 商品，并返回结构化商品列表。

你可以这样对 Agent 说：

```text
帮我找一个适合抓 Amazon 商品列表的 CoreClaw scraper，读取 live 参数后，用关键词“wireless mouse”跑一次，完成后先展示前 20 条结果。
```

典型输出：

- 商品标题
- 价格
- 品牌
- 评分
- 评论数
- 商品链接
- 图片链接

### 案例 2：抓 Amazon 商品详情

目标：输入商品 URL，获取单个商品的结构化详情。

你可以这样对 Agent 说：

```text
帮我找一个按 URL 抓 Amazon 商品详情的 CoreClaw scraper，读取 live 参数后，用这个商品链接跑一次，并把结果导出成 CSV。
```

典型输出：

- 标题
- 卖家
- 品牌
- 描述
- 价格
- 类目
- ASIN
- 图片
- 评分
- 商品详情字段

### 案例 3：抓 Shopify 店铺商品目录

目标：抓取某个 Shopify 店铺某个 collection 下的商品列表。

你可以这样对 Agent 说：

```text
帮我找一个适合抓 Shopify collection 商品列表的 CoreClaw scraper，读取 live 参数后，抓取这个 collection 页面，并返回商品标题、价格、商品链接和图片链接。
```

适合场景：

- 商品库整理
- 竞品分析
- 价格对比
- 店铺目录归档

### 案例 4：抓 Google Maps 商家信息

目标：按关键词和地区抓商家列表。

你可以这样对 Agent 说：

```text
帮我找一个抓 Google Maps 商家的 CoreClaw scraper，按关键词“coffee shop”和地区“Singapore”采集商家名称、评分、地址、电话和网站。
```

典型输出：

- 商家名称
- 评分
- 评论数
- 地址
- 电话
- 官网
- 地图链接

### 案例 5：抓新闻列表并做监测

目标：定期采集某个新闻栏目，获取最新文章标题和发布时间。

你可以这样对 Agent 说：

```text
帮我找一个适合抓新闻列表页的 CoreClaw scraper，读取 live 参数后，抓取这个栏目最近一页内容，并返回标题、发布时间、分类和文章链接。
```

适合场景：

- 舆情监测
- 行业动态追踪
- 内容归档
- 定时更新摘要

### 案例 6：抓企业目录或黄页数据

目标：收集某个目录站中的公司名称、行业和联系方式。

你可以这样对 Agent 说：

```text
帮我找一个适合抓企业目录的 CoreClaw scraper，采集这个目录页中的公司名称、行业、所在地、官网和联系方式。
```

适合场景：

- 销售线索收集
- 行业名录整理
- B2B 数据归档

### 案例 7：异步批量采集并持续跟踪

目标：让 Agent 用 async 模式发起采集，并持续跟踪任务状态。

你可以这样对 Agent 说：

```text
用异步模式启动这个 scraper，保存 run_slug，每 10 秒查询一次状态，完成后先告诉我结果条数，再决定是直接展示还是导出文件。
```

适合场景：

- 数据量较大
- 需要后台运行
- 需要后续自动处理结果

### 案例 8：结果很多时自动导出文件

目标：当结果过多时，不直接在对话中展示，而是导出 CSV。

你可以这样对 Agent 说：

```text
如果这次采集结果超过 50 条，就不要直接全贴出来，导出成 CSV 并返回下载链接。
```

适合场景：

- 商品目录
- 新闻列表
- 商家清单
- 大批量结构化数据

### 案例 9：采集失败后继续查看原因

目标：任务失败时，不只告诉你失败，而是继续把原因找出来。

你可以这样对 Agent 说：

```text
如果这次 run 失败，先查 run/detail，再查 run/last/log，把导致失败的核心原因总结给我。
```

适合场景：

- 参数组织错误
- Worker 执行失败
- 页面抓取异常
- 运行超时或环境问题

### 案例 10：对历史任务做 rerun

目标：复用历史任务参数重新执行一次。

你可以这样对 Agent 说：

```text
对这个 run_slug 做 rerun，启动后继续跟踪新 run 的状态，并在完成后展示前 10 条结果。
```

适合场景：

- 定期重复采集
- 修复问题后重跑
- 对比不同时间点的数据变化

### 案例 11：先预览，再决定是否导出

目标：先快速看一小部分结果，再决定最终交付方式。

你可以这样对 Agent 说：

```text
先读取这次采集结果的前 20 条，如果数据量很大就导出文件，否则直接展示在对话里。
```

适合场景：

- 临时查看采集效果
- 核对字段是否正确
- 控制对话输出长度

### 案例 12：按业务字段重新整理结果

目标：把原始采集结果进一步整理成更适合业务使用的格式。

你可以这样对 Agent 说：

```text
把这次采集结果按“标题、价格、评分、链接”整理出来，如果有缺失字段也保留原始值，不要丢字段。
```

适合场景：

- 采购清单整理
- 竞品表格整理
- 商业情报归档
- 内容运营素材汇总

## 最小上手方式

如果你想最快开始，可以直接这样用：

### 第一步：找 scraper

```text
帮我找一个适合抓 Amazon 商品列表的 CoreClaw scraper。
```

### 第二步：读取参数并运行

```text
读取这个 scraper 的 live 参数，用最小必要参数跑一次。
```

### 第三步：读取结果

```text
完成后先展示前 20 条结果，如果结果很多就导出 CSV。
```

## 这个仓库适合谁

适合：

- 做 Agent 产品的人
- 需要把 CoreClaw 接进自动化流程的人
- 需要结构化采集能力的 AI 应用
- 想让 Agent 而不是人工来管理 scraper 运行的人

## 支持与发布说明

如果你准备公开使用或团队内复用，也建议同时查看：

- `SUPPORT.md`：问题归类方式和支持边界
- `REGRESSION.md`：维护者发布前的冒烟检查清单
- `CHANGELOG.md`：文档与契约的变更记录
- `LICENSE`：仓库许可协议

## 参考

- CoreClaw Store: https://coreclaw.com/store
- Skill 入口：`SKILL.md`
- OpenAPI 契约：`openapi.json`