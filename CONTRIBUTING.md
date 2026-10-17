# 参与 Lithe Store

| 我想… | 怎么做 |
| --- | --- |
| 收录自己的插件 | 读 [docs/publishing.md](docs/publishing.md)，提 PR 或开“插件收录申请” issue |
| 发布插件新版本 | 在自己仓库发布 Release 即可，bot 会自动开 PR（见 [发布新版本](docs/publishing.md#4-发布新版本)） |
| 修改插件信息 | 提 PR 修改 `lithe-plugin-index.json` 中自己的条目 |
| 报告有问题的插件 | 开“报告插件问题” issue；安全问题请勾选“安全问题”，维护者会优先处理 |
| 改进脚本、CI 或文档 | 直接提 PR，并运行 `node --test scripts/store.test.mjs` |

规则：

- 一个 PR 只改一个插件，便于审查和回退。
- `plugins` 按 `id` 字母序排列；每个插件的 `versions` 新版本在前。
- 不修改已发布版本的 `downloadURL` 与 `sha256`；不删除历史版本（撤回问题版本除外）。
- 插件的 bug 与功能请求请提到插件自己的仓库，本仓库只处理收录相关事务。

第一次开发语言插件，先读[LSP 插件完整生命周期](docs/lsp-plugin-lifecycle.md)，
再按[构建、运行与测试接入指南](docs/build-tool-integration.md)扩展能力，
最后按[发布文档](docs/publishing.md)提交收录。

审查标准见 [docs/maintainers.md](docs/maintainers.md#审查清单)。
