# 第 10 章 稽核與可觀測性

GitOps 的稽核軌跡不用額外建：platform-gitops 的 git log 就是「誰、何時、改了哪個環境的什麼、誰批准」的完整紀錄。可觀測性則要回答另一個問題：叢集現在與 Git 一致嗎，不一致多久了。

## Git 作為稽核軌跡

第 8 天結束時，platform-gitops 的歷史長這樣：

```
$ git log --oneline --since="8 days ago" -- apps/order-service/
9f2c1a4 revert: disable COUPON_ENABLED in prod (#88)
7be0d33 deploy(prod): enable COUPON_ENABLED (#86)
4a1e9c0 deploy(prod): order-service 1.8.0 (#85)
2d8f7b1 deploy(staging): order-service 1.8.0 (#83)
c31a0e7 deploy(dev): order-service 1.8.0 (#81)
8e4b2f9 feat(order-service): add ExternalSecret for coupon token (#79)
5c7d1a2 feat(order-service): add COUPON_SERVICE_URL to base ConfigMap (#76)
```

每一行都能點進 PR 看到：誰開的、誰批的、argocd-diff 印出的變更、討論紀錄。審計要「prod 在 11 月的所有變更」，一行指令：

```bash
git log --since=2026-11-01 --until=2026-12-01 --format="%h %ad %an %s" --date=short -- apps/*/overlays/prod/
```

要讓這件事成立，前面章節的幾個約定缺一不可：環境用目錄（第 1 章）、只能透過 PR 改 main（第 5 章）、Squash 讓一個 PR 一個 commit（第 5 章）、統一的 commit 標題格式（第 8 章）、叢集禁止人類寫入（第 9 章）。

## 從叢集回推到程式碼

每個部署的 Pod 都能一路追回需求：

1. Pod 的 image 是 acrplatform.azurecr.io/order-service:1.8.0。
2. ACR 裡 1.8.0 的 sha- 標籤指向 order-service 的 commit a1b2c3d。
3. 那個 commit 是 PR #45 的 Squash，描述寫 Closes #42。
4. Issue #42 是老陳寫的驗收條件。

Argo CD UI 也在每個 Application 顯示目前同步的 commit 與作者，點下去就是 GitHub 的 commit 頁。

## 通知

Argo CD Notifications 依事件發到不同地方：

| 事件 | 去哪 | 誰要看 |
| --- | --- | --- |
| on-deployed（Synced + Healthy） | Slack #platform-deploys | 全團隊，知道什麼上了 |
| on-sync-failed | Slack #platform-alerts，PagerDuty（prod） | 平台工程師 |
| on-health-degraded | Slack #platform-alerts | 平台工程師與該服務的開發者 |
| on-sync-status-unknown | Slack #platform-alerts | 平台工程師，通常是 Repo 連不到 |

通知訊息帶 commit 標題與作者，所以第 8 天早上的 Slack 顯示「order-service-prod synced: revert: disable COUPON_ENABLED in prod (小靈)」，團隊不用問發生什麼事。

## 要監控的指標

Argo CD 暴露 Prometheus 指標。這五個要有儀表板與告警：

| 指標 | 意思 | 告警條件 |
| --- | --- | --- |
| argocd_app_info{sync_status="OutOfSync"} | 有 Application 不同步 | prod 超過 15 分鐘 |
| argocd_app_info{health_status="Degraded"} | 有 Application 不健康 | 任何環境超過 5 分鐘 |
| argocd_app_sync_total{phase="Failed"} | 同步失敗次數 | 10 分鐘內增加 |
| argocd_git_request_duration_seconds | 拉 Git 的延遲 | p95 超過 10 秒 |
| 自訂：部署頻率與從 PR 合併到 Healthy 的時間 | DORA 的兩個指標，從 Git 與 Argo CD 事件算 | 趨勢觀察，不告警 |

## Drift 的三種來源與處理

| 來源 | 例子 | 處理 |
| --- | --- | --- |
| 人手動改叢集 | kubectl scale | selfHeal 改回，RBAC 從源頭擋 |
| 另一個控制器改資源 | HPA 改 replicas | 在 Application 設 ignoreDifferences，或 Repo 裡不寫 replicas |
| Repo 改了但同步失敗 | YAML 合法但叢集拒收 | on-sync-failed 告警，回到 PR 修 |

HPA 的例子在情境中真的發生：prod 的 order-service 用 HPA，小靈一開始在 overlay 寫了 replicas 6，HPA 調成 8 後 Argo CD 一直顯示 OutOfSync。解法是從 overlay 拿掉 replicas，讓 HPA 獨自管。

## 本章在情境中的位置

第 8 天的事故報告直接引用 git log 的七行與 Slack 的通知時間，沒有人需要重建時間線。從報警到恢復 5 分鐘，從 Issue #42 建立到 prod 上線 7 天，兩個數字都從 Git 與 Argo CD 的事件算出來。
