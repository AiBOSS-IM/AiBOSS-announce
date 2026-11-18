# 账本链修复披露（Repair Log）

本文件按追加方式记录对历史账本链的修复。本仓此前公示过的 root 文件**未被修改**，只在此追加说明差异原因，便于任何人用 `verify_economy.py` 复算比对。

## 修复 1 — 测试种子行的占位哈希（2026-11-19）

**现象**：M6 测试期间有一条测试种子行（`ledger.id = 64`，类型 `ADMIN_ADJUST`，`ref_type='m6test'`，`ref_id='seed'`）把 `row_hash` 写成占位串 `TEST-SEED-uid_…`，其后的 `ledger.id = 65` 又把 `prev_hash` 接在这个占位串上，导致 ID-8 链式校验在 id=64 断裂。

**判定**：该行的金额、余额快照与时间戳与相邻真实行自洽（ID-9 通过、无负余额），链断裂只来自占位哈希本身；它是 M6 测试残留，不是业务数据错误。

**处理**：只重算 `prev_hash` / `row_hash` 两列，金额、余额快照、`ts`、`ref_type`、`ref_id` 一律未改（可用重算前后逐字段比对验证）。

- 受影响行：`ledger.id = 64`, `ledger.id = 65`
- 修复前后累计 root（ids 1–65）：`340b2fa9d1544775e553bc63598552186858832f196d1d1a92a067cb5c82b4ff` → `a33e615e0797f09f7f4ce5a379e3a3d540a937dcfba5b46ff996d0d56091e767`

## 修复 2 — 兑换码入账行的 ref_id 指向错误（2026-11-19）

**现象**：`ledger.id = 3` 是一条 `GIFT_REDEEM`（+10 S），但 `ref_id` 被写成测试批次号 `test-2026-11-17`，而不是被核销兑换码的 `code_hash`。跨表核对（兑换码 ↔ 账本）因此报出「1 个已核销码没有对应账本行」与「1 条孤儿账本行」。

**判定依据**（四项独立证据一致）：

1. 兑换码 `ea7b360108cc51e4d15319e7022e0bc0d9e700a851ff9eae9125768c9c1fcda4` 的 `batch_id` 正是 `test-2026-11-17`——写错的值来自批次号而非码哈希；
2. 该码 `status='redeemed'`、`redeemed_by = uid_b45kXvTMvmRUDRGC`、`redeemed_at = 2026-11-17T11:44:51.647Z`，与账本行 3 的用户、金额（10 S）、时间（`11:44:51.712Z`，相差 65ms）完全对应；
3. 该用户全库只有这一条 `GIFT_REDEEM`；
4. 账本行 3 的 `ref_type='redeem'`，与兑换码核销路径一致。

**处理**：把 `ledger.id = 3` 的 `ref_id` 更正为该兑换码的 `code_hash`，随后按 id 顺序重算整条链（`prev_hash`/`row_hash`）。金额、余额快照、`ts`、`user_id`、`type` 均未改动。

**影响**：ids 1–65 的哈希全部随之变化，因此 2026-11-17 已公示的 root（覆盖 ids 1–4）不再与当前链一致：

- ids 1–4 旧 root：`f385a1deb1831abd3a138e497e8e2fbdfeb234fc586b706b5cb3646a3cd43695`（本仓 `roots/2026-11-17.txt`，保持原样未改）
- ids 1–4 更正后 root：`91429113adcac39ca9687a407ea1b37794a77d6d97b53fb78996e3f1dd8fdb4b`

## 复算方式

```bash
# 导出当前账本（主仓 apps/api 目录）
wrangler d1 execute aiboss-db --remote --json \
  --command "SELECT id,user_id,type,amount_s,amount_g,amount_p,ref_type,ref_id,\
balance_after_s,balance_after_g,balance_after_p,prev_hash,row_hash,ts FROM ledger ORDER BY id" > ledger.json

# 校验链、余额、币种恒等式，并输出累计 merkle root
python3 tools/verify_economy.py --ledger ledger.json
```

修复仅涉及测试期数据；上线后账本已加不可变触发器（`ledger_no_update` / `ledger_no_delete`），历史行不再允许被 UPDATE/DELETE。

—— 记录时间：2026-11-19（UTC）
