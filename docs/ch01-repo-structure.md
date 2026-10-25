# 第 1 章 Repo 結構與分支策略

程式碼與部署設定分在兩個 Repo，是 GitOps 最常見的起點。平台組的 order-service 放程式碼，platform-gitops 放所有環境的 manifests。

## 為什麼要分離

- 改一個 replica 數不該觸發整套單元測試與映像檔建置。
- Argo CD 監看 Config Repo 時，不該因為程式碼 commit 而被弄醒。
- 權限可以分開：開發者能合併 App Repo，prod 目錄的變更則需要平台工程師批准。
- 稽核更清楚：Config Repo 的 git log 就是所有環境的部署歷史。

## App Repo：order-service

```
order-service/
├── cmd/                  # 程式入口
├── internal/             # 業務邏輯（含 coupon/）
├── Dockerfile
├── .github/
│   ├── workflows/
│   │   ├── pr-check.yml  # PR 驗證（第 4 章）
│   │   └── release.yml   # 合併後發布（第 6 章）
│   ├── ISSUE_TEMPLATE/
│   └── pull_request_template.md
└── README.md
```

## Config Repo：platform-gitops

使用 Kustomize 的 base 與 overlay 結構。三個環境是三個目錄，不是三個分支。

```
platform-gitops/
├── apps/
│   └── order-service/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/
│           │   └── kustomization.yaml   # images: tag 1.8.0, replicas 1
│           ├── staging/
│           │   └── kustomization.yaml   # tag 1.7.2, replicas 2
│           └── prod/
│               └── kustomization.yaml   # tag 1.7.2, replicas 6
├── argocd/
│   └── applications/
│       ├── order-service-dev.yaml
│       ├── order-service-staging.yaml
│       └── order-service-prod.yaml
└── CODEOWNERS
```

環境 overlay 範例（overlays/prod/kustomization.yaml）：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: order-prod
images:
  - name: acrplatform.azurecr.io/order-service
    newTag: "1.7.2"
replicas:
  - name: order-service
    count: 6
```

## 環境用目錄而不用分支

| 做法 | 晉升方式 | 問題 |
| --- | --- | --- |
| 每個環境一個分支（dev / staging / prod branch） | merge dev → staging → prod | 環境專屬設定會被 merge 帶過去，cherry-pick 容易出錯，分支長期分歧 |
| 每個環境一個目錄，都在 main | 開 PR 改 prod 目錄的 tag | 差異一目了然，CODEOWNERS 可依目錄設批准者，沒有分支分歧 |

## 分支策略：Trunk-based

兩個 Repo 都只有一條長期分支 main。每個工作開短命的 feature branch，透過 PR 合回 main，存活時間以天計。

- 分支命名：`feat/42-coupon-discount`，前綴表示類型，數字是 Issue 編號。
- main 永遠可部署：每次合併都產生可用的映像檔。
- 未完成的功能用 feature flag 藏起來，而不是用長期分支藏起來。
- 不使用 Git Flow 的 develop / release 分支：這些分支與 Config Repo 的環境目錄角色重複。

## 本章在情境中的位置

小林從 main 切出 `feat/42-coupon-discount`。platform-gitops 三個環境目前都指向 1.7.2，這是整個情境的起點。
