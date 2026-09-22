# Career Agent 架构说明

## 系统上下文

```mermaid
flowchart LR
    Scheduler[GitHub Actions 定时任务] --> Pipeline[Career Agent 流水线]
    Sources[企业招聘站] --> Crawlers[平台与企业爬虫]
    Crawlers --> Normalize[岗位标准化与届别识别]
    Normalize --> Store[(SQLite 岗位库)]
    Store --> Filter[规则过滤与增量判断]
    Profile[个人画像 profile.yaml] --> Scoring[匹配分析]
    Filter --> Scoring
    Scoring -->|高价值候选| LLM[DeepSeek API]
    LLM --> Store
    Store --> Reporter[静态报告生成器]
    Reporter --> Pages[GitHub Pages / Cloudflare Pages]
    Reporter --> Feishu[飞书机器人]
    Store --> WebApp[本地投递管理器]
    WebApp --> Applications[data/applications.json]
```

## 每日流水线

```mermaid
sequenceDiagram
    participant A as GitHub Actions
    participant C as 招聘站爬虫
    participant D as SQLite
    participant R as 规则引擎
    participant L as DeepSeek
    participant P as 报告与通知

    A->>C: 定时抓取岗位
    C->>D: 标准化并增量入库
    D->>R: 读取新增或已变化岗位
    R->>R: 届别、类型、关键词粗筛
    alt 高匹配候选
        R->>L: 请求结构化匹配分析
        L-->>R: 分数、理由和风险提示
    end
    R->>D: 保存分析与版本信息
    D->>P: 生成脱敏静态报告
    P-->>A: 发布页面并按需推送飞书
```

## 关键设计

- **增量优先**：岗位 JD、画像、模型和评分版本未变化时复用已有结果，减少运行时间与模型费用。
- **分层分析**：先用本地规则完成届别和岗位方向过滤，只对高价值候选调用更昂贵的模型。
- **失败隔离**：单个招聘站失败不会阻断整个抓取流程，也不会直接把历史岗位判定为下架。
- **发布解耦**：主流程产出静态报告，可发布到 GitHub Pages 或 Cloudflare Pages；本地管理器单独维护投递状态。
- **隐私边界**：API Key 和 Webhook 通过 Secrets 注入；公开部署前应清除个人投递记录和敏感分析。

## 主要数据

| 数据 | 作用 | 建议存储方式 |
| --- | --- | --- |
| `config.yaml` | 公司、入口与抓取策略 | Git 跟踪 |
| `profile.yaml` | 个人画像和评分规则 | 私有部署或脱敏示例 |
| `data/jobs.db` | 岗位、分析和增量状态 | 运行时存储或制品，不建议长期提交大文件 |
| `data/applications.json` | 投递进度 | 私有存储，不应公开 |
| `outputs/` | 静态报告 | `gh-pages` 或部署制品 |
