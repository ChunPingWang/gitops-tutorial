# 第 8 章 環境晉升與回滾

晉升就是「把 staging 目錄的 tag 改成 dev 現在的 tag」的一個 PR。回滾就是「git revert 那個 PR」。兩者都不需要重新建置，都留下紀錄，都由 Argo CD 執行。

## 晉升到 staging（第 6 天）

小林在 platform-gitops 開 PR：

```bash
git switch -c promote/order-service-staging-1.8.0
cd apps/order-service/overlays/staging
kustomize edit set image acrplatform.azurecr.io/order-service=acrplatform.azurecr.io/order-service:1.8.0
git commit -am "deploy(staging): order-service 1.8.0"
git push -u origin HEAD
gh pr create --title "deploy(staging): order-service 1.8.0" --body "晉升自 dev。源頭：order-service PR #45。驗收：老陳用 TEST-NOV11 券測結帳。"
```

PR 跑驗證，argocd-diff 印出只有 image 改變。阿華批准，合併，Argo CD 同步 staging。老陳在 staging 驗收五條驗收條件，在 Issue #42 留言「staging 驗收通過」。

## 晉升到 prod（第 7 天）

同樣的 PR，差別在批准者與注意事項：

- CODEOWNERS 要求 prod 目錄的變更由 platform-engineers 批准，所以是小靈審。
- 小靈先看小林原 PR #45 的「部署注意」：需先在 prod 的 Secret 加 COUPON_SERVICE_TOKEN。確認第 9 章的 ExternalSecret 已在前一天合併。
- PR 描述要附上 staging 驗收的連結。
- 合併時間避開活動峰時，選在第 7 天上午 10:00。

合併後 Argo CD 同步 prod，6 個 Pod 依 rolling update 策略（maxSurge 1、maxUnavailable 0）逐一替換，約 4 分鐘完成。此時 COUPON_ENABLED 仍是 false，程式碼在 prod 但功能沒開。

## 開啟 feature flag

再一個 PR，改 prod overlay 的 ConfigMap patch：

```diff
 # apps/order-service/overlays/prod/configmap-patch.yaml
 data:
-  COUPON_ENABLED: "false"
+  COUPON_ENABLED: "true"
```

部署與開啟分兩個 PR 的理由：部署出問題是基礎設施問題，功能開啟出問題是業務邏輯問題，分開就能分開回滾。

## 回滾（第 8 天）

早上 9:20 監控報警：結帳 5xx 率從 0.1% 升到 4%。coupon_lookup_duration_seconds 的 p99 超過 500ms，因 fail-open 應該不影響結帳，但 5xx 仍在升。小靈決定先回滾再查。

- 選項一，關 flag：revert 開啟 flag 的 PR。最小變動，程式碼留在 prod。
- 選項二，回版本：revert 晉升 prod 到 1.8.0 的 PR，回到 1.7.2。

小靈選一，因為還不確定問題是否在 flag 後面。

```bash
git revert --no-edit <開啟 flag 的 squash commit>
git push origin HEAD:refs/heads/rollback/coupon-flag
gh pr create --title "revert: disable COUPON_ENABLED in prod" --label incident
```

小靈批准自己的 PR（團隊約定：標籤 incident 的 PR 允許平台工程師自審，事後補審）。9:23 Argo CD 同步，ConfigMap 更新，Pod 依 Reloader 重啟。9:25 5xx 回到 0.1%。從報警到恢復 5 分鐘。

事後查出原因：coupon-service 超時時，CouponClient 的 fail-open 正確回退，但金額計算步驟在 discount 為 nil 時 panic。整合測試模擬的超時回的是空折抵，不是 nil。修正走一次完整流程：Issue #49、PR、CI、1.8.1、dev、staging、prod。

## 回滾的原則

| 原則 | 原因 |
| --- | --- |
| 回滾也是 PR，不是 argocd app rollback | Argo CD 的 rollback 會讓叢集與 Git 不一致，auto-sync 會再次滾回去；且 Git 裡沒紀錄 |
| 用 git revert 不用 git reset | 保留歷史，main 禁止 force push |
| 先回滾再查原因 | 恢復服務比找出原因急 |
| 回滾最小範圍 | flag 能解就關 flag，不要連帶回掉其他已驗證的變更 |
| 修正走正常流程 | 不在 prod 目錄直接改，不跳過 staging |

## 晉升的共通規則

- 每次晉升一個環境，不跳級。
- 晉升的是 tag，不重新建置：prod 跑的映像檔與 dev 測的是同一個 digest。
- PR 標題統一 `deploy(環境): 服務 版本`，讓 git log 能 grep。
- staging 與 prod 的晉升 PR 不自動合併，要有人按。

## 本章在情境中的位置

第 8 天下午：prod 跑 1.8.0 但 flag 關閉。Issue #49 追蹤 nil panic，小林已開 Draft PR #50。platform-gitops 的 git log 完整紀錄了晉升、開啟、回滾三個動作。
