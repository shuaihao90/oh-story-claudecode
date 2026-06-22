# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这个仓库是什么

这是一个面向中文网文写作（网文写作）的 **Claude Code / OpenClaw skills 插件包**。它 *不是* 运行时应用——没有构建步骤、没有服务端、没有测试框架。交付物是：

- **`skills/`** —— 13 个 skill，每个由一个 `SKILL.md` 入口（带 YAML frontmatter）+ 按需加载的 `references/` 知识库 + 可选的 `scripts/` 组成。
- **`scripts/`** —— Bash 校验/回归脚本（即「测试套件」）和 Node.js 采集/工具脚本。
- **`.claude-plugin/marketplace.json`** —— 插件清单，每个 skill 对应一个 plugin 条目。发版时在此 bump `metadata.version`。
- **`demo/`** —— 样例产出（拆文报告、导入工程、封面图），同时充当 fixture。

在这里「开发」= 编辑 Markdown 的 skill/references、bash 校验脚本、Node 采集脚本，然后跑下面那套校验。

### 本 `CLAUDE.md` 与被部署的那个 CLAUDE.md

有两个文件都叫 "CLAUDE.md"，别混淆：

- **本文件**（`./CLAUDE.md`）指导**本插件包自身**的开发。（它被本地 `.gitignore` 忽略，不入库。）
- **`skills/story-setup/references/templates/CLAUDE.md.tmpl`** 是 `story-setup` skill **部署进用户写作项目**的模板。那个被部署的文件以及 `.claude/`、`AGENTS.md` 等同样被 `.gitignore` 忽略——它们是部署产物，不属于本仓库。

## 开发命令

没有构建。校验靠一组 Bash 脚本（在仓库根目录运行）。CI 工作流 `.github/workflows/cross-platform.yml` 在 ubuntu/windows/macos 上跑的就是这些脚本，因此它们在合并前必须全过。要跑**单个检查**，直接执行该脚本：

```bash
bash scripts/static-check.sh                 # frontmatter、引用路径、死文件、交叉引用、agent 引用、反引号引用
bash scripts/check-shared-files.sh           # 跨 skill 同名 reference/script 的字节一致性
bash scripts/check-hook-regex-sync.sh        # hook 伏笔状态检测行为
bash scripts/check-story-setup-deployment.sh # story-setup 部署完整性（模板 → 目标）
bash scripts/test-ai-patterns.sh             # deslop AI 痕迹检测器回归
bash scripts/check-python-invocation.sh      # 禁止裸 `python3`（Windows Store stub bug）
bash scripts/check-hook-locale-safety.sh     # hooks 在 LC_ALL=C 下运行，禁止全角字符类
bash scripts/test-hook-encoding-portable.sh  # cp936/GBK 路径编码可移植性
bash scripts/test-charcount-portable.sh      # 可移植 CJK 字数统计（真实解释器）
bash scripts/test-charcount-portable.sh --stub  # ……并针对模拟的 Windows Store stub
```

Node 脚本（采集器、`cdp-utils.js`、`check-ai-patterns.js`、`normalize-punctuation.js`）在 CI 做语法校验：

```bash
node --check skills/story-long-scan/scripts/fanqie-rank-scraper.js   # 单个文件
for f in $(find skills -name '*-scraper.js' -o -name 'cdp-utils.js' -o -name 'setup-cdp-chrome.js'); do node --check "$f"; done
```

以插件形式安装/校验本包（见 CONTRIBUTING）：

```bash
npx skills validate                  # 校验全部 skill（新增 skill 必跑）
npx skills add worldwonderer/oh-story-claudecode -y -g   # 全局安装（用户路径）
```

## 架构

### Skill 解剖与路由

每个 skill 是 `skills/<name>/SKILL.md`（frontmatter：`name`、`version`、`description` 以 `触发方式:` 触发词结尾，可选 `metadata.openclaw`）+ `references/`（知识库，按需加载，绝不整包塞进上下文）+ 可选 `scripts/`。`skills/story/SKILL.md` 是**路由器**：一张意图→skill 分发表，把用户请求（`/story`、`/网文`、"帮我开书"）模糊匹配到具体 skill（通过 `Skill("skill-name")`），对走 agent 的查询有优雅降级。

### 共享文件同步系统（CI 最容易挂的地方）

很多 reference 文件被有意复制到 2–6 个 skill 中，且**必须逐字节一致**（如 `banned-words.md`、`anti-ai-writing.md`、`hooks-suspense.md`、`genre-*.md` 家族、`format-and-structure.md`）。每个受管副本都带一行 `sync-source:` frontmatter，指向 `skills/story-setup/references/agent-references/` 下的规范副本。

- **改一个共享文件 ⇒ 同步所有副本。** 跑 `bash scripts/check-shared-files.sh` 查漂移；有任何字节不一致就 fail。
- 少数文件是**有意**分叉、被豁免的——列在 `check-shared-files.sh` 顶部的 `IGNORE_NAMES` / `ANALYST_DIVERGENT_NAMES`（如 `output-templates.md`、`story-short-analyze` 里带 analyst-lens 前缀的 genre 副本）。看到不一致先读这份白名单，再判断是不是 bug。

### `story-setup` 部署管线

`story-setup` 是基础设施 skill：用户在其写作项目里跑 `/story-setup` 时，它把模板从 `skills/story-setup/references/templates/` 复制进目标项目。映射关系是 `story-setup/SKILL.md`（Phase 2.0）里一张机械可检查的表：

| 源（本仓库内） | 目标（用户项目） | 合并方式 |
|---|---|---|
| `templates/hooks/`（含 `lib/common.sh`、`lib/sentinel.sh`） | `.claude/hooks/` | 递归替换 |
| `templates/rules/*.md` | `.claude/rules/*.md` | 替换 |
| `templates/agents/*.md` | `.claude/agents/*.md`（7 个 agent） | 替换 |
| `agent-references/*.md` | `.claude/skills/story-setup/references/agent-references/` | 替换 |
| `templates/CLAUDE.md.tmpl` | `CLAUDE.md` | **按 marker/section 合并——绝不覆盖用户 section** |
| `templates/settings-hooks.json` | `.claude/settings.local.json` | **按 hook 命令合并、去重** |
| `templates/上下文.md.tmpl` | `{书名}/追踪/上下文.md` | **仅在不存在时创建** |
| 生成的 sentinel | `.story-deployed` | 替换 |

铁律：**合并，绝不覆盖用户配置。** 文件分类（可安全覆盖 / 需合并 / 不碰）与基于 `.story-deployed` 的版本检测见 `skills/story-setup/UPGRADING.md`。改模板时遵守此分类。`check-story-setup-deployment.sh` 校验模板→目标集合的完整性与内部可解析性。

### Agent 体系与降级

7 个专业 agent 定义在 `templates/agents/`，被写作/审查 skill 消费：`story-architect`（Opus）、`character-designer` / `narrative-writer` / `story-researcher`（Sonnet）、`consistency-checker` / `story-explorer` / `chapter-extractor`（Haiku）。它们作为 subagent 被 spawn，且在 agent 不可用/缺失/过期/spawn 失败时**必须降级为 solo**——每个 spawn agent 的 skill 都有显式 fallback 路径（如 `story-review` → 带内置 rubric 的 solo；`story` 路由器 → 直接 Read/Grep 查询）。注意 Claude Code 只在会话启动时注册 custom agent，因此部署后的 agent 需要新开会话才能生效（这在部署报告里会告知用户）。

### 小说项目文件模型 + 拆文→写作数据流

skill 在用户的书目录下读写一套固定目录结构——大多数 skill 改动都需要先理解它：

- `设定/`（世界观 · 角色 · 势力 · 关系.md · 题材定位）、`大纲/`（大纲.md · 卷纲_*.md · 细纲_第NNN章.md = 逐章"章节蓝图"）、`正文/`、`对标/`、`参考资料/`
- `追踪/` —— 分层连续性：`上下文.md`（compact 恢复用）、`伏笔.md`、`时间线.md`、`角色状态.md`。这是长篇扛过上下文限制的方式；pre/post-compact hook 会快照并恢复它。
- `.active-book`（仓库相对路径指向当前书）为 hook 和 skill 定位当前活跃项目。

**拆文 → 写作管线：** 拆文 skill（`story-long-analyze`、`story-short-analyze`）把结构化拆解结果输出到 `拆文库/{书名}/`（角色/剧情/设定/章节，长篇另有 `文风.md` · `节奏.md` · `情绪模块.md`）——这是**唯一事实源**。写作 skill 通过项目级视图 `对标/{书名}/剧情/` 消费它们，找不到时回退直接读 `拆文库/`。`story-import` 把已有小说逆向工程成同样的结构以便续写。

### 跨平台可移植性是一等公民

近期大量工作针对 Windows 中文（cp936/GBK）和 macOS 边界情况。改 hook 或字数统计代码时，守住这些不变量：

- Hooks 在 `LC_ALL=C` 下运行；正则里禁止用全角字符类。
- 禁止裸 `python3` 调用（Windows Store stub 返回误导性退出码）——`check-python-invocation.sh` 守这条。
- 路径/编码代码必须在 Git Bash on Windows + cp936 stdout 下存活（issue #164 的那个 bug）。
- 字数统计必须在真实解释器和 Windows Store stub 之间都可移植——见 `test-charcount-portable.sh` 及其 `--stub` 模式。
- Bash 脚本面向 **bash 3+**（macOS 默认）；避开 bash-4 专属特性。

## 贡献约定（来自 CONTRIBUTING.md）

- **所有 skill/reference 内容用中文。** 多用表格和模板，少写散文；内容必须能让 agent 直接执行，而不是教程。
- commit message 用中文，格式 `类型: 描述`，`类型` ∈ `feat` / `fix` / `docs` / `refactor`。
- 一个 PR = 一个聚焦改动。从 `main` 切分支。
- 同一 skill 内不要有重复 reference 文件；跨 skill 用路径引用共享没问题（但那样它就成了受管同步副本——见上文）。
