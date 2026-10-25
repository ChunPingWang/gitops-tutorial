# 第 4 章 CI：PR 驗證 Pipeline

PR 驗證 Pipeline 的工作是在合併前回答一個問題：這個分支合進 main 後，main 是否仍然可部署。它在每次 push 時跑，Draft 階段也跑，但不碰任何叢集。

## 驗證階段

| 階段 | 工具 | 失敗時代表 | Draft 是否跑 |
| --- | --- | --- | --- |
| Lint 與格式 | golangci-lint | 程式碼品質或風格不符 | 是 |
| 單元測試 | go test，覆蓋率門檻 80% | 邏輯錯誤 | 是 |
| 整合測試 | WireMock 模擬 coupon-service | 外部介面處理錯誤 | 是 |
| 映像檔建置 | docker buildx，只建不推 | Dockerfile 壞了 | 否 |
| 漏洞掃描 | Trivy，Critical 等級就失敗 | 基底映像檔或依賴有已知漏洞 | 否 |
| Manifest 驗證 | kustomize build + kubeconform | Config Repo 的 YAML 無法渲染或不符 schema | 是 |
| Policy 檢查 | Kyverno CLI 或 Conftest | 違反團隊政策，如沒設 resource limits | 是 |

## GitHub Actions 範例（.github/workflows/pr-check.yml）

```yaml
name: pr-check
on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: go.mod }
      - run: golangci-lint run ./...
      - run: go test ./... -race -coverprofile=cover.out
      - run: go tool cover -func=cover.out | tail -1 | awk '{ if ($3+0 < 80) exit 1 }'

  image:
    if: github.event.pull_request.draft == false
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v6
        with: { push: false, tags: order-service:pr-${{ github.event.number }}, load: true }
      - uses: aquasecurity/trivy-action@0.28.0
        with: { image-ref: order-service:pr-${{ github.event.number }}, severity: CRITICAL, exit-code: 1 }

  manifests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { repository: platform-org/platform-gitops, path: gitops }
      - run: |
          for env in dev staging prod; do
            kustomize build gitops/apps/order-service/overlays/$env \
              | kubeconform -strict -summary
          done
```

## 幾個設計決定

- Draft 階段跳過映像檔建置與掃描：這兩步最耗時，且 Draft 時程式碼天天變。轉 Ready for Review 時 GitHub 會重新觸發，就會跑。
- PR 階段的映像檔不推到 ACR：沒合併的程式碼不該在 Registry 裡留下沒人清的映像檔。
- Manifest 驗證跑在 App Repo 的 PR：因為小林這次要改 ConfigMap，他在 platform-gitops 也開了 PR。兩邊都驗證，避免合併後才發現 YAML 壞了。
- Pipeline 定義放在 Repo 裡跟程式碼一起版控：改 Pipeline 也要開 PR。

## Config Repo 自己的 PR 驗證

platform-gitops 也有 pr-check，但只做三件事：kustomize build 每個 overlay、kubeconform 驗證 schema、Kyverno 檢查政策。另外有一個 argocd-diff 步驟，用 Argo CD CLI 在 PR 留言印出「合併後叢集會改什麼」，Reviewer 不用自己在腦中渲染 YAML。

```bash
argocd app diff order-service-prod --local apps/order-service/overlays/prod --server-side-generate
```

## CI 紅燈的處理原則

- 紅燈是訊息不是懲罰，當天處理。
- 不用 retry 挖過去：不穩定的測試要修或標記為 flaky 並開 Issue。
- 不在 PR 裡關掉檢查步驟來讓它綠：改 Pipeline 要在獨立 PR 討論。

## 本章在情境中的位置

小林的 PR #45 在第 2 天被 Trivy 擋下一次：基底映像檔 golang:1.22 有 Critical 漏洞，改成 1.22.8 後通過。第 4 天結束時，App Repo 與 Config Repo 的 PR 都是綠燈。
