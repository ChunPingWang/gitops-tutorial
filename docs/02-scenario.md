# 貫穿全書的開發情境

主角是一家電商公司的平台組，要在訂單服務 order-service 加上「優惠券折抵」功能，並在雙十一活動前上線。之後每一章都接續這個情境。

## 團隊與角色

| 角色 | 人 | 在流程中的責任 |
| --- | --- | --- |
| 開發者 | 小林 | 實作功能，開 Issue 與 Draft PR，維護工項 Checklist |
| Reviewer | 阿華 | 審查程式碼與設計，批准合併 |
| 平台工程師 | 小靈 | 維護 Config Repo、Argo CD、Pipeline 範本；批准 prod 晉升 |
| 產品負責人 | 老陳 | 定義需求與驗收條件，在 staging 驗收 |

## 系統與環境

- 服務：order-service，Go 寫的 REST API，已在 AKS 上線。
- 環境：dev、staging、prod 三套，各是一個 AKS 叢集的一個 namespace。
- 映像檔倉庫：Azure Container Registry（ACR），位址 acrplatform.azurecr.io。
- CD 代理人：Argo CD，每個叢集裝一套，都指向同一個 Config Repo。

## 兩個 Repo

| Repo | 內容 | 誰改它 |
| --- | --- | --- |
| order-service（App Repo） | 程式碼、測試、Dockerfile、CI 定義 | 開發者透過 PR |
| platform-gitops（Config Repo） | 所有服務在三個環境的 Kubernetes manifests | CI 機器人與平台工程師透過 PR |

## 需求摘要

結帳時可輸入一組優惠券代碼，系統驗證有效性後折抵金額。優惠券資料由新的 coupon-service 提供，order-service 透過 HTTP 查詢。功能需受 feature flag 控制，以便在 prod 先部署再開啟。

## 情境時間線

1. 第 1 天：老陳開 Issue #42，小林開 Draft PR #45 列工項。
2. 第 2 到 4 天：小林逐步實作、push，CI 每次驗證，阿華提早給設計回饋。
3. 第 5 天：PR 轉 Ready for Review，阿華批准，Squash Merge 到 main。
4. 第 5 天：CI 建置映像檔 1.8.0，自動往 platform-gitops 開 PR 更新 dev 的 tag，自動合併，Argo CD 同步 dev。
5. 第 6 天：小林開 PR 把 tag 晉升到 staging，老陳驗收。
6. 第 7 天：小靈批准 prod 晉升 PR，Argo CD 同步 prod，功能以 feature flag 開啟。
7. 第 8 天：發現 coupon-service 超時導致結帳失敗，git revert prod 的 tag，3 分鐘內回滾。

## 端到端流程

```
 小林 ──push feat/42──▶ order-service (App Repo)
                              │ PR #45
                              ▼
                        GitHub Actions CI
                        測試 / 建置 / 掃描
                              │ merge 到 main
                   ┌──────────┴──────────┐
                   ▼                     ▼
          ACR: order-service:1.8.0   platform-gitops (Config Repo)
                                     overlays/dev → staging → prod
                                             ▲ 輪詢
                                      Argo CD（每個叢集一套）
                                             │ sync
                                 AKS dev / AKS staging / AKS prod
```
