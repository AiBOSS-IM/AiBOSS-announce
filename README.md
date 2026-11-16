# AiBOSS-announce — aiboss.im 公开账本公示仓

**公开可复算 · 信任工程（AFR 理念）**

本仓由 [aiboss.im](https://aiboss.im) 自动推送，内容为每日账本 **Merkle root** 快照。

## 公示内容
- `roots/YYYY-MM-DD.txt` — 当日全部账本流水构建的 Merkle root（UTC 日界）
- 校验方法见主仓 `docs/SSOT-v1.1.txt` §11 信任工程与 `verify_economy.py` 公开只读版

## 原则
- 平台账本 append-only，行级链式 hash（ID-8），每日 root 可独立重算
- 任何人均可复算任意区间并比对本仓记录（ID-1 账本守恒）
- 本仓只追加、不修改；历史记录不可变更

---
本仓不含任何密钥、凭据或用户数据。
