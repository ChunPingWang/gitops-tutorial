# Azure DevOps 整合預告

> 本頁是概念對照。實際在 Azure DevOps 組織上逐步操作、附截圖的版本見[第 11 章 Azure DevOps 整合：Step by Step 實作](ch11-azure-devops.md)。

整套流程搬到 Azure DevOps 時，概念不變，改的是工具名稱與幾個設定位置。這一節先給對照表與需要特別注意的差異，下一階段的教材會把每一章的範例改成 Azure DevOps 版。

## 概念對照

| 本教材用的 | Azure DevOps 對應 | 差異要點 |
| --- | --- | --- |
| GitHub Repository | Azure Repos | Git 完全相同，URL 格式為 dev.azure.com/組織/專案/_git/repo |
| GitHub Issue | Azure Boards Work Item（User Story 或 Product Backlog Item） | 有正式的階層：Epic → Feature → Story → Task |
| Issue Template | Work Item Template，或 Process 裡的自訂欄位 | 驗收條件有專屬欄位 Acceptance Criteria |
| Closes #42 | AB#42 在 commit 或 PR 描述，或 PR 的 Work Items 頁籤連結 | 批准後自動將 Work Item 設為 Resolved 或 Closed，可在專案設定關閉 |
| Draft PR | Draft PR | 功能相同，轉正式按「Publish」 |
| PR Checklist `- [ ]` | 同樣語法，可勾選 | PR 列表不顯示完成進度，可改用 PR 連結的 Task Work Items 追蹤 |
| pull_request_template.md | .azuredevops/pull_request_template.md，或依分支的 pull_request_template/branches/main.md | 位置與檔名不同，支援依目標分支套用不同範本 |
| GitHub Actions | Azure Pipelines（YAML） | 檔案是 azure-pipelines.yml，語法不同但結構相似 |
| Branch Protection | Branch Policies | 最少審查者、必須連結 Work Item、Build Validation、Comment resolution、Merge type 限制 |
| CODEOWNERS | Branch Policy 的 Automatically included reviewers，依路徑指定 | 設在 Policy 裡而不是 Repo 檔案 |
| Required status checks | Build Validation policy | 一條 Pipeline 一條 policy |
| GitHub Environments | Azure Pipelines Environments | 有 Approvals and checks，可要求人工批准 |
| OIDC 登入 Azure | Service Connection（Workload identity federation） | 不用存密碼，建立後 Pipeline 直接引用 |
| GITOPS_BOT_TOKEN | Build Service 帳號或 PAT，或 Pipeline 跨 Repo 的 Contribute 權限 | 跨 Repo 開 PR 需給 Build Service 帳號 Contribute 與 Create branch |
| Argo CD 連 GitHub App | Argo CD 連 Azure Repos，用 PAT 或 Azure Managed Identity | Argo CD 2.x 支援 Azure DevOps 的 webhook 與 Managed Identity |
| GitHub webhook 到 Argo CD | Azure Repos Service Hook（Code pushed）到 Argo CD /api/webhook | 設在 Project Settings 的 Service Hooks |

## 要特別注意的差異

1. Work Item 連結是硬性要求。Branch Policy 可以設「每個 PR 必須連結 Work Item」，這比 GitHub 的 Closes 約定強。建議開啟，第 2 章的「Issue 與 PR 雙向連結」就成為系統保證。
2. 工項 Checklist 有兩條路。繼續在 PR 描述用 `- [ ]`，或把工項建成 Story 底下的 Task Work Items 並連結 PR。後者能在 Boards 看進度，前者跟程式碼更貼近。下一階段建議兩者並用：Task 列大項，PR Checklist 列細項。
3. Squash merge 的 commit 訊息預設是 PR 標題加描述，與 GitHub 相同，但 Azure Repos 會在描述後自動附上 Related work items 的清單。semantic-release 要能讀懂，標題仍須守 Conventional Commits。
4. Pipeline 觸發模型不同。Azure Pipelines 的 PR 觸發不是寫在 YAML 的 pr: 區塊（Azure Repos 忽略它），而是透過 Build Validation policy 設在分支上。第 4 章的 Draft 跳過步驟無法用預定義變數判斷：第 11 章實測 PR 觸發的 run 沒有 Draft 狀態，改由 Pipeline 用 System.AccessToken 查 PR API 的 isDraft（見第 11 章 Step 4）。
5. Argo CD 連 Azure Repos 的認證。建議用 Managed Identity（AKS Workload Identity）授權 Azure DevOps，避免 PAT 到期。這是第 9 章最小權限的延伸。
6. Environments 的 Approvals 可以取代部分 CODEOWNERS 的角色，但在 GitOps 裡批准點應該留在 Config Repo 的 PR，而非 Pipeline。下一階段會具體討論怎麼選。

## Azure Pipelines 的 PR 驗證骨架（第 4 章的對應）

```yaml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

stages:
  - stage: test
    jobs:
      - job: test
        steps:
          - task: GoTool@0
            inputs: { version: '1.22' }
          - script: golangci-lint run ./... && go test ./... -race

  - stage: image
    # isDraft 由 test stage 的 meta job 查 PR API 取得，完整寫法見第 11 章 Step 4
    condition: and(succeeded(), ne(dependencies.test.outputs['meta.pr.isDraft'], 'true'))
    jobs:
      - job: build
        steps:
          - task: Docker@2
            inputs: { command: build, repository: order-service, tags: pr-$(System.PullRequest.PullRequestId) }
          - script: trivy image --severity CRITICAL --exit-code 1 order-service:pr-$(System.PullRequest.PullRequestId)
```

Branch Policy 對 main 設 Build Validation 指向這條 Pipeline，相當於 GitHub 的 required status check。

## 下一階段範圍

- 每一章加一節「Azure DevOps 版」，範例改成 Azure Repos 與 Azure Pipelines。
- 第 2 章補 Azure Boards 的 Epic / Feature / Story / Task 層級與 PR 連結。
- 第 6 章補 Pipeline 跨 Repo 開 PR 的 Build Service 權限設定。（已於第 11 章 Step 10 完成）
- 第 7 章補 Argo CD 連 Azure Repos 的 Managed Identity 設定與 Service Hook。（已於第 11 章 Step 11 完成）
- 第 9 章補 Service Connection 的 Workload identity federation。（已於第 11 章 Step 10 完成）
- 新增一章：Azure DevOps Wiki 與 Dashboard 展示部署狀態與 DORA 指標。
