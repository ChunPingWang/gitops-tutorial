# 附錄：範本與名詞對照

## Issue Template（.github/ISSUE_TEMPLATE/feature.md）

```markdown
---
name: 功能需求
about: 新功能或變更
labels: feature
---

## 背景
<!-- 為什麼要做，誰會用 -->

## 驗收條件
<!-- 每條都要能被測試驗證，用 - [ ] -->
- [ ] 

## 非功能需求
<!-- 效能、安全、相容性；沒有就寫無 -->

## 相關資源
<!-- 設計文件、其他 Issue、外部服務 -->
```

## PR Template（.github/pull_request_template.md）

```markdown
Closes #

## 工項
<!-- 這個 PR 要做的事，每項對應一到幾個 commit；用 - [ ] 才有進度顯示 -->
- [ ] 

## 設計要點
<!-- 做了哪些取捨，為什麼；Reviewer 要先看哪裡 -->

## 測試方式
<!-- 本地怎麼測，staging 怎麼驗收 -->

## 部署注意
<!-- 需要先做的設定、Secret、資料庫變更；沒有就寫無。平台工程師晉升 prod 前會看這裡 -->
```

## Config Repo 的 PR Template（platform-gitops）

```markdown
## 變更類型
<!-- deploy / config / rollback / infra，選一個 -->

## 源頭
<!-- App Repo 的 PR 連結，或 Issue -->

## 影響環境
<!-- dev / staging / prod -->

## 驗收
<!-- 前一個環境的驗收紀錄連結；dev 可寫無 -->

## 回滾方式
<!-- 預設是 revert 這個 PR；若不是，說明 -->
```

## Commit 訊息速查

```
feat(scope): 新功能              → minor
fix(scope): 修錯                  → patch
feat(scope)!: 不相容變更          → major
refactor / test / docs / chore    → 不跳版本
deploy(env): service version      → Config Repo 晉升專用
revert: <原標題>                  → Config Repo 回滾專用
```

## 名詞對照表

| 本教材用詞 | 英文 | 說明 |
| --- | --- | --- |
| 期望狀態 | Desired state | Git 裡描述的系統應該長的樣子 |
| 實際狀態 | Live state / Actual state | 叢集現在真正的樣子 |
| 調和 | Reconcile / Reconciliation | 代理人把實際狀態拉向期望狀態的動作 |
| 偏移 | Drift | 實際與期望不一致 |
| 同步 | Sync | Argo CD 執行一次調和 |
| 自我修復 | Self-heal | 偵測到偏移時自動同步 |
| 修剪 | Prune | 刪除 Git 裡已不存在但叢集裡還在的資源 |
| 晉升 | Promotion | 把同一個版本從低環境推到高環境 |
| App Repo | Application repository | 放程式碼的 Repo |
| Config Repo | Configuration / Deployment repository | 放 manifests 的 Repo |
| 覆層 | Overlay | Kustomize 裡環境專屬的差異設定 |
| 草稿 PR | Draft PR | 尚未準備好合併、不能被 merge 的 PR |
| 工項 | Task / Checklist item | PR 描述裡的 `- [ ]` 項目 |
| 驗收條件 | Acceptance criteria | Issue 裡定義「怎樣算完成」的條件 |
| 分支保護 | Branch protection / Branch policy | 對 main 強制 PR、批准、status checks 的設定 |
| 壓縮合併 | Squash merge | 把 PR 所有 commit 合成一個 |
| 特徵開關 | Feature flag | 用設定控制功能開關，讓部署與發布分離 |
| 失效開放 | Fail-open | 依賴不可用時繼續服務，而非拒絕 |
| 工作負載身分 | Workload Identity | AKS Pod 用 Azure AD 身分存取 Azure 資源，不用密碼 |

## 參考資源

- [OpenGitOps 原則](https://opengitops.dev/)
- [Argo CD 文件](https://argo-cd.readthedocs.io/)
- [Kustomize 文件](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [External Secrets Operator](https://external-secrets.io/)
- [Kyverno](https://kyverno.io/)
- [Sigstore cosign](https://docs.sigstore.dev/cosign/overview/)
- [GitHub Docs: About pull request templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/creating-a-pull-request-template-for-your-repository)
- [Azure DevOps: Branch policies](https://learn.microsoft.com/azure/devops/repos/git/branch-policies)
- [Azure DevOps: Link work items to pull requests](https://learn.microsoft.com/azure/devops/boards/backlogs/connect-work-items-to-git-dev-ops)
