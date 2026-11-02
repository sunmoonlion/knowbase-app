

## 2026-11-02 luna 第二轨开发（未发布）

最新增量：后端 `971747286c2b2329c8738f5735e8d76321651cc9` 已推
`feat/reading-receipts`。清单追加/修改/重解析在既有修订锁内核基版，拒绝过期写入，
返回 409 resolution_changed；provider README 同步，无 HTTP 字段或 schema 变更。
完整 733 passed / 5 skipped、ruff/pyright 通过，三种并发覆盖在原版复现后修复。
详见后端 CHECKPOINT。没有自动恢复丢失回包，也没有替用户确认采集。本分支仍含下述
未发布 WP-3/逐读法开发，不改变 qwen 发布候选或部署源。

后端 fe184e4 增加 WP-3 公共原件精确核验；5bfb60f 修复既有类型和部分交付边界。全量 723 passed/8 skipped、最后边界 9 passed、ruff/pyright 通过。代码保留在 feat/finance-track-two 开发分支，不更新部署 sources.yaml，不进入 #46 巨潮保底。

非公开交付逐读法完整性与部分加工恢复仍需生产者/消费者联合修复和契约测试；未声明 Wind 接入或整条数据链验收通过。完整继续说明见 knowbase-backend/CHECKPOINT.md。

2026-11-02 白天更新：feat/reading-receipts 的后端 a69436b 补 collected-originals 的逐读法种类与最新 ready 版本引用，保留旧布尔字段语义。完整 730 passed / 5 skipped，ruff/pyright 通过；新增 HTTP/真实库验证，所有已有 schema 定义不变，新增 CollectedOriginalRead/CollectedRenditionRead 并补 knowfeed 为契约使用方。此父仓保留早前 WP-3 未发布开发，不能误认为只含本次字段修改；发布评审需核完整基线差异。未修改部署源或 qwen 正在发布的候选。


## 2026-11-02 同步本人重新核对路由

后端 020a420 补齐 user-data/v2 用户侧 refresh，复用原服务重解析，保留 expected_revision CAS。契约 README 已记身份、修订、错误与无授权/无采集副作用。开发分支真实跨应用/数据库定向 25 passed，ruff/pyright 通过；未发布。
