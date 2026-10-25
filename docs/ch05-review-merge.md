# 第 5 章 Review 與合併

從 Draft 轉成 Ready for Review 是一個明確的交接點：作者宣告「我認為這可以合了」，Reviewer 從看方向轉為逐行審查。合併的條件由 Branch Protection 強制，不靠人記得。

## 轉 Ready for Review 前的自我檢查

- [ ] Checklist 全部勾完，或未勾的項目已說明為什麼移出
- [ ] CI 綠燈，包含轉 Ready 後才會跑的映像檔建置與掃描
- [ ] 已 rebase 到最新 main，沒有衝突
- [ ] PR 描述的「測試方式」與「部署注意」已更新成最終版
- [ ] 自己先在 GitHub 上看一次 diff，移除 debug 程式碼與無關改動

```bash
gh pr ready 45
gh pr edit 45 --add-reviewer ahua
```

## Branch Protection（main 分支）

| 設定 | order-service | platform-gitops |
| --- | --- | --- |
| 需要 PR 才能合併 | 是 | 是 |
| 最少批准人數 | 1 | 1，prod 目錄需 CODEOWNERS 批准 |
| 必須通過的 status checks | test、image、manifests | validate、argocd-diff |
| 分支必須是最新 | 是 | 是 |
| 合併方式 | 只允許 Squash | 只允許 Squash |
| 禁止 force push | 是 | 是 |
| 管理員也受限 | 是 | 是 |

platform-gitops 的 CODEOWNERS：

```
# 預設所有人都能審 dev 與 staging
*                                   @platform-org/developers
# prod 目錄與 Argo CD 設定只有平台組能批
/apps/*/overlays/prod/              @platform-org/platform-engineers
/argocd/                            @platform-org/platform-engineers
```

## Review 準則

Reviewer 的責任是回答「這能不能合」，不是重寫作者的程式碼。阿華依這個順序看 PR #45：

1. 先讀 PR 描述與 Issue #42 的驗收條件，確認每一條都有對應的程式碼與測試。
2. 再看測試：測試名稱能不能讀出行為，超時回退那條有沒有被測到。
3. 最後看實作：介面、錯誤處理、並發、資源釋放。
4. 留言分級：「blocking」不改不合，「suggestion」作者自己決定，「nit」風格小事。
5. 每則留言說明原因，不只寫「這樣不好」。

阿華留了一則 blocking：CouponClient 在 timeout 後沒有關閉 response body，會漏連線。小林補一個 commit 修正，CI 重跑，阿華批准。

## 合併方式：Squash Merge

小林的六個 commit 在 main 上變成一個。Squash 的 commit 訊息用 PR 標題，內文用 PR 描述。

```
feat: coupon discount at checkout (#45)

Closes #42

- CouponClient with 500ms timeout and fail-open fallback
- discount applied with zero floor
- POST /orders accepts couponCode, 422 on invalid
- gated behind COUPON_ENABLED
```

| 方式 | main 上的歷史 | 適合 |
| --- | --- | --- |
| Squash | 一個 PR 一個 commit，歷史等於 PR 列表 | Trunk-based，main 用來自動算版本號 |
| Merge commit | 保留全部 commit 加一個 merge 節點 | 需要保留細部歷史的大型重構 |
| Rebase | 全部 commit 線性接上 | 作者已把 commit 整理得很乾淨 |

選 Squash 的原因：第 6 章的版本號工具讀 main 的 commit 類型，一個 PR 一個 feat 最乾淨；回滾時也只要 revert 一個 commit。

## 合併後自動發生的事

- Issue #42 因 Closes 關鍵字自動關閉。
- 分支 `feat/42-coupon-discount` 自動刪除（Repo 設定「Automatically delete head branches」）。
- release workflow 觸發，進入第 6 章。

## 本章在情境中的位置

第 5 天上午：PR #45 以 Squash 合進 main，Issue #42 關閉。main 的最新 commit 是 `feat: coupon discount at checkout (#45)`。
