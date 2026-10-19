# LinguaGacha 贡献指南

本文约定贡献流程纪律：分支模型、提交组织与 PR 说明要求。仓库行动规则见 [AGENTS.md](AGENTS.md)，冲突时以其为准；开发命令与验证入口见 [docs/WORKFLOW.md](docs/WORKFLOW.md)。

## 分支模型

|分支|角色|
|---|---|
|`main`|发布线。版本提交（`v0.x.y`）与发布标签落在这里，不接受直接的功能开发。|
|`dev`|集成分支。功能分支从这里拉出，PR 合回 `dev`；发布时由 `dev` 收敛到 `main`。|

- 功能分支命名 `<type>/<主题>`，与提交的 type 一致，如 `feat/built-in-proxy-settings`、`fix/catalog-refresh`。
- 功能分支从**上游** `dev` 最新提交拉出（`git fetch upstream && git rebase upstream/dev`）；本地 `origin` 指向个人 fork，不作为分支基点。
- 生命周期以单次贡献为限，合入后删除；长周期分支定期 rebase 上游，保持 diff 可审。

## 原子化提交

- 一个提交解决一个可独立理解的改动：commit 信息不读 diff 即可回答"这个提交做了什么、为什么"。
- 每个提交独立构建、独立通过所选验证；实现、配套测试与文档说明进同一提交，不出现"实现先行、测试后补"的序列。
- 无关改动不顺手携带：格式化、依赖升级、文档重组各自单独提交。
- 提交序列应能讲述实现故事，推荐按依赖方向排序：领域与契约 → 传输与后端 → 界面与文案 → 长期文档。
- 不遗留 `fixup`/`wip`/注释掉的代码；推送前自行整理历史。

## 提交信息

遵循 Conventional Commits，格式 `type(scope): 中文摘要 #issue[如有]`：

- `type`：`feat`、`fix`、`refactor`、`perf`、`docs`、`test`、`chore`。
- `scope` 用领域词（如 `backend`、`frontend`、`agent`、`translation`、`toolbox`），全仓性质改动可省略。
- 摘要用中文动宾短句，说清行为结果而非动作过程（`feat(backend): 支持手动代理配置`，而非"添加代码"）。
- 只有确实关联 issue 时才写编号，不编造占位编号。

## PR 说明

- PR 描述按「动机 / 改动 / 行为影响 / 验证 / 限制与未决」组织，打开前按 [WORKFLOW](docs/WORKFLOW.md) 的触发条件完成验证并把结果写进描述。
- PR 描述要能让 reviewer 不读 diff 就回答五个问题：为什么改、改了什么、影响什么、如何验证、还有什么未决。
- 一个 PR 只承载一个主题；发现范围蔓延时拆分 PR。
- 本地草稿放 `.github/pr-drafts/<主题>.md`，PR 打开后将内容转为 GitHub 描述并删除草稿文件。

## 本地脚手架

本仓库允许在贡献分支上引入仅服务于本地开发的脚手架，用于对齐贡献流程与评审上下文：

- 典型内容：贡献纪律、PR 草稿、领域词汇表等背景材料。
- 这些提交与产品改动分离，**不回合上游**。
- 脚手架不得改变产品行为，也不作为产品改动的验证依赖。
- 每个脚手架提交在正文末尾带 trailer `Scaffold: drop-before-merge`，用 `git log --grep="Scaffold: drop-before-merge"` 可完整列出。

### 交付前移除

1. 确认产品提交都在脚手架之后，且工作区干净。
2. 列出全部脚手架提交：`git log --grep="Scaffold: drop-before-merge" --oneline upstream/dev..HEAD`。
3. 建本地备份分支：`git branch scaffold-backup/<主题>`，**不推送**。
4. 交互式 rebase 丢弃脚手架提交：`git rebase -i upstream/dev`，对脚手架行标记 `drop`。
5. 复核 `git log upstream/dev..HEAD` 只剩产品提交，再推送到 fork。
6. PR 合入后删除备份分支。

不使用 tag 标记待丢弃提交：tag 是长期引用，rebase 后会悬空，且容易被误推。需要找回时用本地备份分支。

## Reviewer friendly 检查清单

- [ ] 提交原子化，序列无 fixup 残留，信息符合上一节约定。
- [ ] 测试与文档随实现同步，不单独欠账。
- [ ] 已按 [WORKFLOW](docs/WORKFLOW.md) 选择静态检查与行为测试，结果写入 PR 描述。
- [ ] 默认行为与旧配置的兼容性有明确说明（尤其涉及默认值变化时）。
- [ ] 交付前回看 diff，确认命名、注释、边界与声明一致。
- [ ] 前端视觉改动附设计输入与真机/等价验证说明。
