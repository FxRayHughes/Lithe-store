## 改动类型

- [ ] 收录新插件
- [ ] 新增版本
- [ ] 修改插件信息
- [ ] 撤回版本 / 下架插件（请写明原因）
- [ ] 脚本、CI 或文档

## 插件

- 插件 ID：
- 来源仓库：
- 版本 / Release 链接：

## 自检

- [ ] 我是该插件的作者或维护者（或已获得授权）
- [ ] `plugin.json` 的 `vendor.signatureRequirement` 为 `"none"`，ID 不在 `dev.lithe.*`
- [ ] 已在本地运行 `node scripts/validate-index.mjs --verify-archives`
- [ ] 插件要运行的程序都已在 `plugin.json` 中声明，并在仓库 README 中说明
- [ ] 没有修改已发布版本的 `downloadURL` / `sha256`，没有删除历史版本（撤回除外）
