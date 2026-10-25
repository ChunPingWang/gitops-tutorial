# 第 7 章 CD：Argo CD 自動 Reconcile

Argo CD 裝在每個叢集裡，每 3 分鐘拉一次 platform-gitops，把每個 Application 指向的目錄渲染成 manifests，與叢集實際狀態比對。有差異就標記 OutOfSync，設了自動同步就直接修正。

## Application 定義（argocd/applications/order-service-dev.yaml）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-dev
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
spec:
  project: platform
  source:
    repoURL: https://github.com/platform-org/platform-gitops.git
    targetRevision: main
    path: apps/order-service/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: order-dev
  syncPolicy:
    automated:
      prune: true        # Repo 刪了的資源，叢集也刪
      selfHeal: true     # 有人手動改叢集，改回來
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 3
      backoff: { duration: 30s, factor: 2 }
```

三個環境的差異只在 path、namespace、與 syncPolicy：

| 環境 | automated | selfHeal | prune | 誰按同步 |
| --- | --- | --- | --- | --- |
| dev | 是 | 是 | 是 | 沒人，全自動 |
| staging | 是 | 是 | 是 | 沒人，全自動 |
| prod | 是 | 是 | 否 | 沒人，但 prune 需手動確認 |

prod 關 prune 的原因：誤刪 Repo 裡的一個 PVC 定義，不該在 3 分鐘內變成 prod 的資料丟失。走到這步要有人在 Argo CD UI 按下 Prune。

## 同步的五個狀態

| 狀態 | 意思 | 何時看到 |
| --- | --- | --- |
| Synced | 叢集等於 Repo | 平時 |
| OutOfSync | Repo 改了叢集還沒跟上，或有人手動改叢集 | PR 合併後的幾秒到 3 分鐘 |
| Progressing | 正在 rollout，Pod 還沒全部 Ready | 同步中 |
| Healthy | 所有資源的 health check 通過 | 同步完成 |
| Degraded | Pod CrashLoop、readiness 失敗或 rollout 超時 | 部署壞了 |

Sync 與 Health 是兩個獨立軸：Synced 但 Degraded 代表 Repo 裡的設定本身有問題，不是 Argo CD 沒做事。

## 第 5 天下午在 dev 發生的事

1. 12:05 platform-gitops PR 合併，main 的 dev overlay 指向 1.8.0。
2. 12:07 Argo CD 輪詢到新 commit，order-service-dev 變成 OutOfSync。
3. 12:07 automated sync 觸發，kubectl apply 等效的變更送進叢集，Deployment 開始 rolling update。
4. 12:08 新 Pod Ready，舊 Pod 結束，狀態 Synced + Healthy。
5. 12:08 Argo CD Notifications 往 Slack #platform-deploys 發一則：order-service-dev synced to 1.8.0。

小林全程沒有碰 kubectl。他用的指令只有這兩個：

```bash
argocd app get order-service-dev
argocd app history order-service-dev
```

## 演示：持續調和

小林在 dev 手動把 replicas 改成 3 做壓測：

```bash
kubectl -n order-dev scale deploy/order-service --replicas=3
```

3 分鐘內 Argo CD 標記 OutOfSync，selfHeal 把 replicas 改回 1。要壓測就開 PR 改 overlay，或暫時在 Argo CD UI 停止 auto-sync。這是 GitOps 的約定：叢集不是編輯器，Git 才是。

## Webhook 加速

3 分鐘輪詢對 dev 有點慢。在 GitHub 設一個 webhook 指向 Argo CD 的 /api/webhook，push 後幾秒內就觸發 refresh。輪詢保留作為 webhook 掉了的後備。

## App of Apps

argocd/applications 目錄本身也由一個 Argo CD Application 管理（叫 root）。新增一個服務就是在這個目錄加一個 YAML 並開 PR，Argo CD 會自己長出新的 Application。Argo CD 的設定也在 Git 裡，沒有例外。

## 本章在情境中的位置

第 5 天下午：dev 跑的是 1.8.0，staging 與 prod 仍是 1.7.2。小林在 dev 用 TEST-NOV11 券下了一筆訂單，折抵正確。
