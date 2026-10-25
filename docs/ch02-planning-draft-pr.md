# 第 2 章 規劃：Issue、Draft PR 與工項 Checklist

開發前先把工項列在 Draft PR 上，讓需求、設計與程式碼從第一天就綁在同一個地方。此時 Issue 負責「做什麼、怎樣算完成」，Draft PR 負責「怎麼做、做到哪」。

## 步驟 1：Issue 寫需求與驗收條件

老陳用 Issue Template 開出 Issue #42。驗收條件是整個功能的合約，後面的 PR、測試、驗收都往回對齊。

```markdown
# 結帳支援優惠券折抵

## 背景
雙十一活動需要優惠券，行銷已在 coupon-service 建好資料。

## 驗收條件
- [ ] POST /orders 接受 couponCode 欄位
- [ ] 代碼無效時回 422，訊息為 COUPON_INVALID
- [ ] 折抵後金額不得低於 0
- [ ] coupon-service 超時 500ms 內必須回退為「不折抵」且不阻擋結帳
- [ ] 功能受 feature flag COUPON_ENABLED 控制，預設關閉

## 非功能需求
- p95 結帳延遲增加不超過 100ms
```

## 步驟 2：開 Draft PR 並列工項

小林切分支、推一個幾乎空的 commit，立刻開 Draft PR #45。這一步在寫任何業務邏輯之前完成。

```bash
git switch -c feat/42-coupon-discount
git commit --allow-empty -m "chore: start coupon discount (#42)"
git push -u origin feat/42-coupon-discount
gh pr create --draft --title "feat: coupon discount at checkout" --body-file .github/pull_request_template.md
```

PR 描述依 PR Template 填寫，核心是工項 Checklist：

```markdown
Closes #42

## 工項
- [ ] 新增 CouponClient，封裝 coupon-service HTTP 呼叫（含 timeout 500ms）
- [ ] Order 領域模型加入 discount 欄位與計算
- [ ] POST /orders 接受 couponCode，錯誤回 422
- [ ] feature flag COUPON_ENABLED 接線
- [ ] 單元測試：有效券 / 無效券 / 超時回退 / 金額下限
- [ ] 整合測試：用 WireMock 模擬 coupon-service
- [ ] 更新 OpenAPI 文件
- [ ] 更新 platform-gitops base 的 ConfigMap（新增 COUPON_SERVICE_URL）

## 設計要點
超時回退採 fail-open：coupon-service 不可用時照原價結帳。

## 測試方式
本地 `make test`；staging 用 TEST-NOV11 券驗證。

## 部署注意
需先在 prod 的 Secret 加入 COUPON_SERVICE_TOKEN。
```

## 為什麼要在開發前就開 PR

- Reviewer 在程式碼寫出來前就能對設計方向給意見。阿華看到「fail-open」後留言要求加上 metrics，小林在 Checklist 補了一項。
- 工項清單與程式碼在同一頁，不用在 Jira 與 GitHub 之間來回。GitHub 會在 PR 列表顯示「3 of 9 tasks」。
- CI 從第一次 push 就開始跑，建置環境的問題第一天就暴露。
- PR 描述在 merge 後成為交付文件，半年後追「為什麼 fail-open」有完整脈絡。

## Issue 與 Draft PR 的分工

| 內容 | 放 Issue | 放 Draft PR |
| --- | --- | --- |
| 業務需求、驗收條件 | 是 | 否，用 Closes #42 連回去 |
| 實作工項清單 | 否 | 是 |
| 設計決策與取捨 | 大方向 | 具體做法 |
| 跨多個 PR 的規劃 | 是，或用 Epic / Milestone | 否 |
| 測試與部署注意事項 | 否 | 是 |

## PR Template 設計原則

- 最多四個小節，超過沒人填。
- 第一行就要有 Closes #編號，讓 Issue 與 PR 雙向連結，merge 時自動關 Issue。
- 工項用 `- [ ]` 語法，不用普通列點，否則沒有進度顯示。
- 「部署注意」小節是平台工程師在晉升 prod 前必看的地方，沒有就寫「無」。
- 完整範本見附錄。

## 常見問題

- Draft PR 開太久：工項超過兩週的量，要拆成多個 PR，每個都能獨立合併。小林的 Checklist 有 9 項，預估 4 天，在合理範圍。
- 把 Draft PR 當 Jira：工項只列「這個 PR 要做的事」，跨 PR 的規劃回到 Issue 或 Milestone。
- CI 資源有限：可讓 Draft 狀態跳過耗時的步驟，第 4 章示範。

## 本章在情境中的位置

第 1 天結束：Issue #42 有五條驗收條件，Draft PR #45 有九項工項，阿華已留下第一則設計回饋。還沒有一行業務程式碼。
