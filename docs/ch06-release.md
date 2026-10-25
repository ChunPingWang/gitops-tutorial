# 第 6 章 交付：Image 發布與 Config Repo 的 PR

合併到 main 後，CI 做三件事：算出版本號、建置並推送映像檔、往 Config Repo 開一個更新 dev tag 的 PR。CI 到此為止，不碰叢集。

## release workflow（.github/workflows/release.yml）

```yaml
name: release
on:
  push:
    branches: [main]

permissions:
  contents: write
  id-token: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      # 1. 從 Conventional Commits 算版本號，打 tag，產生 CHANGELOG
      - id: semver
        uses: cycjimmy/semantic-release-action@v4
        env: { GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} }

      # 2. 登入 ACR，建置並推送，簽章
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az acr login --name acrplatform
      - id: build
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            acrplatform.azurecr.io/order-service:${{ steps.semver.outputs.new_release_version }}
            acrplatform.azurecr.io/order-service:sha-${{ github.sha }}
      - run: cosign sign --yes acrplatform.azurecr.io/order-service@${{ steps.build.outputs.digest }}

      # 3. 往 Config Repo 開 PR 更新 dev
      - uses: actions/checkout@v4
        with:
          repository: platform-org/platform-gitops
          token: ${{ secrets.GITOPS_BOT_TOKEN }}
          path: gitops
      - run: |
          cd gitops/apps/order-service/overlays/dev
          kustomize edit set image acrplatform.azurecr.io/order-service=acrplatform.azurecr.io/order-service:${{ steps.semver.outputs.new_release_version }}
      - uses: peter-evans/create-pull-request@v7
        with:
          path: gitops
          token: ${{ secrets.GITOPS_BOT_TOKEN }}
          branch: bot/order-service-dev-${{ steps.semver.outputs.new_release_version }}
          title: "deploy(dev): order-service ${{ steps.semver.outputs.new_release_version }}"
          body: |
            自動產生。源頭：order-service PR #45
            Image digest: ${{ steps.build.outputs.digest }}
          labels: auto-merge
```

## 版本號從哪裡來

semantic-release 讀 main 上自上次 tag 以來的 commit。小林的 Squash commit 是 feat，所以 1.7.2 變成 1.8.0。沒有人手動決定版本號，這是第 3 章堅持 Conventional Commits 的回收。

## 為什麼是 PR 而不是直接 push 到 Config Repo

- Config Repo 的 main 有 Branch Protection，機器人也不例外。
- PR 跑 Config Repo 自己的驗證（kustomize build、kubeconform、argocd-diff），機器人生的 YAML 也可能壞。
- dev 環境的 PR 打上 auto-merge 標籤，驗證通過就自動合併，效果等同直接 push，但多了一層驗證與一筆紀錄。
- staging 與 prod 不自動合併，第 8 章說明。

## 映像檔 tag 的選擇

| Tag | 用途 |
| --- | --- |
| 1.8.0 | 人讀的版本，Config Repo 裡寫這個 |
| sha-a1b2c3d | 對應回程式碼 commit，除錯用 |
| digest sha256:... | cosign 簽章與 Kyverno 驗證用，不可變 |

不使用 latest：Argo CD 看不出 latest 有沒有變，回滾時也不知道要回到哪。

## Config Repo 收到的 PR

```diff
 # apps/order-service/overlays/dev/kustomization.yaml
 images:
   - name: acrplatform.azurecr.io/order-service
-    newTag: "1.7.2"
+    newTag: "1.8.0"
```

argocd-diff 在 PR 留言：

```
===== apps/Deployment order-dev/order-service ======
-        image: acrplatform.azurecr.io/order-service:1.7.2
+        image: acrplatform.azurecr.io/order-service:1.8.0
```

小林在第 4 章開的 ConfigMap PR（新增 COUPON_SERVICE_URL）已在第 4 天合併，所以這次只改 tag。設定先行、程式碼後到，是需要新設定的功能的標準順序。

## 本章在情境中的位置

第 5 天中午：ACR 有了 order-service:1.8.0 且已簽章。platform-gitops 的 PR「deploy(dev): order-service 1.8.0」自動合併。叢集還沒有任何變化，下一章 Argo CD 接手。
