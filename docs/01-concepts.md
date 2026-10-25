# GitOps 核心概念

GitOps 的一句話定義：系統的期望狀態全部寫在 Git 裡，由軟體代理人持續把實際狀態拉向那個期望。要改系統，就改 Git；不再有人直接對叢集下指令。

## 四大原則

整理自 [OpenGitOps v1.0](https://opengitops.dev/)。

| 原則 | 意思 | 在本教材情境中的對應 |
| --- | --- | --- |
| 宣告式 Declarative | 描述「要什麼」，不描述「怎麼做」 | Kubernetes YAML 與 Kustomize overlay |
| 版本化且不可變 Versioned and immutable | 期望狀態存在 Git，每次變更都有 commit | Config Repo 的每一次 merge |
| 自動拉取 Pulled automatically | 代理人從 Git 拉變更，而非 CI 推進叢集 | Argo CD 每 3 分鐘輪詢 Config Repo |
| 持續調和 Continuously reconciled | 代理人不斷比對實際與期望，有差異就修正 | 有人手動 kubectl scale，Argo CD 會改回來 |

## 與傳統 CI/CD 的差異

傳統模式是 Push：CI pipeline 跑完測試後直接拿著叢集憑證執行 kubectl apply 或 helm upgrade。GitOps 是 Pull：CI 只負責產出映像檔與更新 Config Repo，叢集內的代理人自己去拉。

| 面向 | Push（傳統 CI/CD） | Pull（GitOps） |
| --- | --- | --- |
| 誰持有叢集憑證 | CI 系統，憑證外洩面大 | 叢集內的代理人，憑證不出叢集 |
| 部署觸發 | Pipeline 步驟 | Git 的變更被代理人偵測 |
| 有人手動改叢集 | 沒人發現，直到下次部署覆蓋 | 代理人標記 OutOfSync，可自動改回 |
| 回滾 | 重跑舊版 pipeline | git revert，代理人自動同步 |
| 稽核 | 散在 CI 日誌與 kubectl 歷史 | git log 就是完整紀錄 |
| 多叢集 | 每個叢集一組憑證與步驟 | 每個叢集裝一個代理人，指向同一 Repo |

```
Push 模式
  開發者 ──push──▶ App Repo ──▶ CI（測試、建置）──kubectl apply──▶ 叢集
                                    │ 持有叢集憑證

Pull 模式
  開發者 ──push──▶ App Repo ──▶ CI（測試、建置）──▶ Registry
                                    └──PR 更新 tag──▶ Config Repo ◀──輪詢── Argo CD（在叢集內）──▶ 叢集
```

## CI 與 CD 的分工

- CI（持續整合）負責「驗證與產出」：測試、建置映像檔、推到 Registry、往 Config Repo 開一個更新 tag 的 PR。
- CD（持續部署）負責「調和」：Argo CD 或 Flux 監看 Config Repo，把叢集變成 Repo 描述的樣子。
- 兩者之間的交接點永遠是 Git commit，所以每一次部署都有人負責 review、有 PR 可追。

## 常見誤解

- 「把 YAML 放進 Git 就是 GitOps」：不是。沒有自動拉取與持續調和，只是版控。
- 「GitOps 只適用 Kubernetes」：常見但不必然。Terraform 搭配 Atlantis 或 Terraform Cloud 也是同一精神。
- 「GitOps 取代 CI」：不會。CI 仍要跑測試與建置，只是不再直接碰叢集。
