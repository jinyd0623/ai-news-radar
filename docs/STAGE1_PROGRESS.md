# 阶段 1 执行记录

- 更新时间：2026-08-31（北京时间）
- 状态：本轮范围已完成，待 CI 验证
- 对应计划：`PROJECT_PLAN.md` 第 5 节「1. 免费 MVP」与第 7 节（该文件在仓库之外单独维护）

## 本轮定下的四个边界

按用户决定收口，不在本轮扩张：

| 决定 | 取舍 |
| --- | --- |
| 不引入 LLM 密钥 | 推荐理由与标题改写留空，改用规则手段补摘要与标签 |
| 不改名 | 展示名仍为 AI News Radar，只清理指错地方的上游引用 |
| 低质源限流不删 | buzzing / techurls / newsnow 保留，但真正接上限流 |
| 本轮不接 arXiv | 实测 cs.AI 每天 302 条、每条约 1900 字符英文摘要，需独立栏位，且无密钥时全英文反而拖累中文阅读 |

## 改前的实测差距

依据 `data/source-status.json` 与 `data/daily-brief.json`（2026-08-31T09:12Z）。

计划要求日报每条包含七项，实际只有三项达标：

| 要求 | 改前 | 处置 |
| --- | ---: | --- |
| 中文标题 | 20/20 | 已达标 |
| 一句话摘要 | 3/20 | 本轮修复 |
| 推荐理由 | 0/20 | 需密钥，本轮不做 |
| 来源等级 | 0/20 | 本轮修复（数据早已存在，只是没进日报） |
| 主题标签 | 无 | 本轮新增 |
| 原始链接 | 20/20 | 已达标 |
| 关联人物或事件 | 0/20 | 属阶段 2 人物库 |

两个根因都不需要付费：

1. `RawItem` 没有 `summary` 字段，RSS 的 `<description>` 从官方源、精选媒体、OPML 源一路被丢弃。
   下游其实早已预留位置（story 与日报的构建都在写 `summary`），只是上游从来没填过。
   摘要实际走的是 `meta` 通道（`PUBLIC_RAW_META_FIELDS` 里本来就列了 `summary`），
   所以本轮**没有改动 dataclass**，按既有写法接入即可。
2. `DISCUSSION_FETCH_CAP` 只在 `fetch_buzzing` 一处生效。techurls 单轮抓 405 条、newsnow 抓 129 条，
   都没截断，最终占了精选 143 条里的 66 条（46%），而两者历史进精选率都在 0.4% 以下。

## 本轮改动

| 方面 | 改动 |
| --- | --- |
| 摘要 | 新增 `clean_feed_summary` 与 `feed_entry_summary`，接入官方源、精选媒体、OPML 两个分支 |
| 摘要副作用 | 收紧 `apply_public_raw_meta`：空字符串不再写入记录，避免两千多条各多一个空字段；`False` 这类合法假值仍保留 |
| 主题标签 | 新增 `TOPIC_RULES` / `assign_topics` / `add_topic_fields`，纯关键词零依赖 |
| 来源等级 | `build_story_record` 的 `primary_item` 补 `source_tier` / `source_tier_label` / `topics` |
| 限流 | 新增 `cap_discussion_items`，接入 `fetch_techurls` 与 `fetch_newsnow` |
| 前端 | 主视图 meta-row 加主题 chip；来源等级进来源 chip 的 tooltip |
| 上游残留 | 清理 legacy 页 canonical/og:url、伯乐 Skill README 的示例地址、marketplace 的 owner/homepage/失实描述、GPT_HANDOFF 的路径 |

几处刻意的取舍：

- **限流按新鲜度截断，不在循环里 break**。techurls 的页面按 publisher 分组，
  中途截断会整块丢掉后面的 publisher，而不是丢最旧的条目。
- **主题标签不设 products 兜底桶**。落在所有条目上的标签不携带信息。
- **前端不为来源等级再加一个可见 chip**。等级已由来源 chip 的配色表达，meta-row 再多一个标签只会更挤。
- **`SOURCE_TIER_BY_SITE` 保留已下线的 `iris`**。未登记的 `site_id` 会回落成「其他来源」，
  删掉会让 `archive.json` 里的历史条目在页面上掉级。
- **marketplace.json 的 `author` 保留 LearnPrompt**。那是真实作者，不是上游残留；
  `LICENSE`、README 的 fork 署名、fork 指引里指向上游的链接同理保留。

## 验证结果与它的边界

`./.venv/Scripts/pytest.exe -q` → **249 passed**（阶段 0 基线 234；本轮累计新增 15 项）。

实测数据（拿真实 `data/latest-24h.json` 跑）：

- 主题标签命中 73/143 条（51%）；models 41 / ai-coding 22 / agents 20 / open-source-models 10 / research 6 / evaluation 2
- trap 词 video / guide / provide / management 均未误命中 `ide`
- story 层 40/40 带上来源等级标签（改前 0/20），topics 命中 24/40
- `clean_feed_summary` 的中英文截断路径输出长度全部落在 160 以内

**两处没能本地验证，必须留意：**

1. **摘要覆盖率的真实提升量不出来。** 这台机器上 `huggingface.co`、`ai.meta.com`、
   `youtube.com`、`r.jina.ai` 全部返回 000——是网络屏蔽而非源失效（`huggingface.co`
   本身就是仓库里正常工作的官方源）。能连通的只有 arXiv 和 about.fb.com。
   端到端覆盖率要等 GitHub Actions 跑一轮后看 `data/daily-brief.json`。
2. **前端 `itemTopicLabels` 未实际执行。** `node --check` 通过，但逻辑 harness 被本地
   权限策略拦下，JS 行为仅经人工推演。下一轮 Pages 发布后需人工打开页面确认 chip 渲染正常。

## 遗留与下一步

- **`aibreakfast` 双路径修复仍未经 CI 验证**（阶段 0 遗留项）。下一轮 Actions 跑完后看
  `data/source-status.json`：若 beehiiv 与 Jina 两条路都失败，该源应直接删除。
- **计划第 7 节的 Meta AI / Mistral / YouTube 官方源尚未接入。** 本地网络无法验证可用性，
  盲加违背「每次引入新来源时记录可靠性与降级方案」，需要能在 CI 里探测后再做。
- **arXiv 留待下一轮**，需要独立的论文栏位与选题规则。
- **观察项（本轮未动）**：这一轮 `official_ai` 抓了 178 条原始条目，但进 24 小时精选视图的是 0 条。
  官方源按 45 天窗口抓取，而视图是 24 小时窗口，官方博客并非每天更新。结果是权重最高的一层
  （importance 1.0）当天缺席，权重最低的热议层（0.32）贡献 66 条。这是窗口设计的固有结果，
  改它属于产品决策，先记录不处理。
