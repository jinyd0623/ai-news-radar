# 阶段 0 执行记录

- 更新时间：2026-08-31（北京时间）
- 状态：已完成
- 对应计划：`PROJECT_PLAN.md` 的“阶段 0：Fork 和工作流”（该文件在仓库之外单独维护，当前位于
  `C:\Users\jyd64\Desktop\PROJECT_PLAN.md`）

## 完成结果

| 项目 | 结果 |
| --- | --- |
| GitHub Fork | 已创建：[jinyd0623/ai-news-radar](https://github.com/jinyd0623/ai-news-radar) |
| 本地目录 | 当前为 `D:\workspace\ai-news-radar`（阶段 0 当时在 `E:\workspace\my\news\ai-news-radar`） |
| 默认分支 | 按用户确认继续使用上游已有的 `master`，未另建 `main` |
| `origin` | `https://github.com/jinyd0623/ai-news-radar.git` |
| `upstream` | `https://github.com/LearnPrompt/ai-news-radar.git` |
| GitHub Actions | 已启用；首次手动运行 [#1](https://github.com/jinyd0623/ai-news-radar/actions/runs/33376476977) 成功，用时 6 分 7 秒 |
| GitHub Pages | 已从 `master` 分支根目录发布：[在线页面](https://jinyd0623.github.io/ai-news-radar/)；使用默认域名并强制 HTTPS |
| 自定义域名 | 未配置；已删除上游 `CNAME`，不会占用 `news.learnprompt.pro` |

## 本地环境与验证

- Python：3.11.9，虚拟环境目录为 `.venv`。
- Windows 时区数据已通过条件依赖 `tzdata` 补齐。
- 完整自动化测试：`234 passed`。
- 数据格式检查：12 个 JSON 文件、RSS XML 和 GitHub Actions YAML 均可正常解析。
- 本地网页检查：页面可正常加载最新数据、栏目、热点和中英双语标题，仓库入口已指向个人 Fork。
- 线上检查：Pages 返回 HTTP 200；当前 143 条精选数据中，英文标题漏译数为 0。

## 免费来源与安全约束

- 工作流只使用公开免费来源。
- AgentMail、X API、SocialData 和 TikHub 在工作流中固定关闭。
- DeepSeek 相关环境变量和密钥入口已从工作流移除；未引入 DSH。
- 仓库没有写入任何 API Key、Token 或其他密钥。
- `paid-source-state.json` 的来源状态为空，最新精选、广泛列表和日报中付费来源条目数均为 0。
- 已删除上游的 `data/persona-cache.json`，避免继续沿用外部模型评分缓存；无模型密钥时人物评分按项目自身的无密钥降级逻辑运行。

## 中文标题处理

- 修复了“来源只提供英文标题时提前返回”的问题。
- 短产品名和仓库名也可以命中中文翻译缓存。
- 当前缺失标题已写入翻译缓存，因此后续定时任务可复用，不需要付费翻译服务。
- GitHub Actions 首次刷新后新增的 10 条英文标题已补译并重新发布。

## 提交与发布记录

| 提交 | 说明 |
| --- | --- |
| `f30a669` | 完成阶段 0 初始化：Pages 地址、免费源工作流、本地兼容、翻译修复及首批公开数据 |
| `a330868` | GitHub Actions 首次成功运行后自动提交的数据快照 |
| `d6bbd32` | 补全首次自动刷新新增的英文标题翻译 |

Pages 对 `d6bbd32` 的发布任务 [#3](https://github.com/jinyd0623/ai-news-radar/actions/runs/33377446951) 已成功完成，用时 46 秒。

## 已知情况

- Pages 的 GitHub 托管构建显示一条 Node.js 20 弃用警告，来源是 GitHub 管理的 `actions/upload-artifact@v4`；本仓库自己的 `update-news.yml` 只用 `actions/checkout@v6` 和 `actions/setup-python@v6`，无从处理也不需要处理。

## 遗留项处置（2026-08-31）

原「已知情况」里的三条已全部收口。

### 两个失败源

依据是 `data/source-status.json`（`generated_at` 为 `2026-08-31T09:12:15Z`，即失败那一轮）里的原始报错。

| 源 | 报错 | 处置 |
| --- | --- | --- |
| `iris` | origin 连续 502，重试耗尽，单源耗时 183 秒（整轮约 6 分钟） | 下线 |
| `aibreakfast` | `403 Client Error` on `https://r.jina.ai/...`，253 毫秒被拒 | 改双路径 |

`iris.findtruman.io` 已转型为无关产品（FindTruman，AI 应用搭建），`/web/info_flow` 页面永久消失，
该 fetcher 依赖抓取那个页面里的 `const feeds = [...]`，无法修复。它在
`reports/source-quality/v0.8-audit.md` 里也是最差的源：11563 条、AI 保留率 16.9%、进精选率 0.2%。
已从 `collect_all` 注册表移除并删除 `fetch_iris`，但**保留** `SOURCE_TIER_BY_SITE["iris"]`——
未登记的 `site_id` 会回落成 `("other", "其他来源", 9)`，删掉会让 `archive.json` 里的历史条目在页面上掉级。

`aibreakfast` 原先只有 Jina Reader 一条路，而 `docs/SOURCE_COVERAGE.md` 记载走 Jina 的**理由**
正是 beehiiv 原生 feed 会被 GitHub Actions 屏蔽，即两条路同时不通。现改为
beehiiv 原生 `/feed` 优先、Jina Reader 回落，两条都拿不到条目才算失败，报错信息合并两边原因。
**这个修法本地无法验证**：国内网络下 beehiiv 返回 Cloudflare `Just a moment` 挑战，`r.jina.ai` 完全连不上，
能否救活只有 CI 跑一轮才知道。若 Actions 里两条路都不通，该源应直接删除（审计里它仅贡献 6 条，
但 AI 相关度 100%、独家率 83.3%）。

### 本地仓库

原浅克隆记录已不适用。当前工作副本是 `D:\workspace\ai-news-radar`（**不再是**阶段 0 记录的
`E:\workspace\my\news\ai-news-radar`，该路径在当前机器上不存在），完整历史 2713 条提交，
无 `.git/shallow`，`origin` 与 `upstream` 双远程已配置。

### 验证

`./.venv/Scripts/pytest.exe -q` → `235 passed`（阶段 0 基线 234 项，本次新增
`test_parse_ai_breakfast_feed_entries` 覆盖正常条目、重复链接与缺链接）。

## 阶段 0 验收结论

Fork、克隆、双远程配置、本地运行、Actions、Pages、公开免费数据生成和安全限制均已验证。阶段 0 可以关闭，后续工作可从来源清单、人物库和个人化配置继续。
