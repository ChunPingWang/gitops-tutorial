# 第 3 章 開發：Commit 規範與小步推送

開發階段的節奏是「小 commit、天天 push、勾一項工項」。每次 push 都讓 CI 跑一輪，也讓 Reviewer 看到最新進度。

## Commit 訊息：Conventional Commits

格式為 `類型(範圍): 摘要`，摘要用現在式、不超過 72 字元。類型決定版本號怎麼跳（第 6 章）。

| 類型 | 用途 | 版本影響 |
| --- | --- | --- |
| feat | 新功能 | minor 加一 |
| fix | 修錯 | patch 加一 |
| refactor | 重構，行為不變 | 無 |
| test | 只動測試 | 無 |
| docs | 只動文件 | 無 |
| chore | 工具、依賴、CI 設定 | 無 |
| feat! 或 footer 寫 BREAKING CHANGE | 不相容變更 | major 加一 |

小林在這四天的 commit：

```
feat(coupon): add CouponClient with 500ms timeout (#42)
feat(order): apply discount and enforce zero floor (#42)
feat(api): accept couponCode on POST /orders (#42)
test(coupon): cover timeout fallback with WireMock (#42)
feat(flag): gate coupon flow behind COUPON_ENABLED (#42)
docs(api): document couponCode in OpenAPI (#42)
```

## 小步推送的節奏

1. 一個 commit 對應 Checklist 的一項或更小，每個 commit 都能編譯、測試通過。
2. 每天至少 push 一次，不讓工作只存在本機。
3. push 後回 PR 頁面勾掉完成的工項，必要時在 PR 描述補一行註記。
4. CI 紅燈當天處理，不累積。
5. 每天或每兩天 `git rebase origin/main`，讓分支跟上 main，減少最後合併的衝突。

## 工項改變時怎麼辦

- 發現新工項：直接加到 Checklist，並在 PR 留言說明原因。小林第 2 天加了「coupon_lookup_duration_seconds metric」，回應阿華的回饋。
- 工項超出範圍：刪掉並開新 Issue。「優惠券使用次數上限」被移到 Issue #47。
- Checklist 是活的文件，但驗收條件不該在 PR 裡改；要改就回 Issue 與老陳討論。

## 早期回饋的做法

- Draft 階段的 review 是「看方向」，不是「挑錯字」。阿華只看 CouponClient 的介面與錯誤處理策略。
- 想要針對性回饋時，在 PR 留言指名並說明要看哪一段，例如「請先看 internal/coupon/client.go 的 timeout 處理」。
- 不要在 Draft 階段指派正式 Reviewer，避免 GitHub 一直提醒。

## 本章在情境中的位置

第 4 天結束：Checklist 九項全部勾完，外加一項 metric。分支有六個 commit，CI 綠燈。小林準備把 PR 轉成 Ready for Review。
