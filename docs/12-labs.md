# 實作練習與檢核表

每章一個 Lab，用自己的 GitHub 帳號與一個本機 kind 或 minikube 叢集就能完成；要接 AKS 與 ACR 的步驟有標註。全部做完約 6 到 8 小時。

## 前置準備

- [ ] 安裝 git、gh、docker、kubectl、kustomize、kind、argocd CLI
- [ ] 建兩個 GitHub Repo：order-service、platform-gitops（可用本教材附帶的範例，或自己的任一服務）
- [ ] kind create cluster，並依官方文件安裝 Argo CD 到 argocd namespace

## Lab 對照表

| Lab | 對應章節 | 任務 | 完成標準 |
| --- | --- | --- | --- |
| 1 | 第 1 章 | 在 platform-gitops 建立 base 與 dev / staging / prod overlay，三個 overlay 只差 namespace、tag、replicas | `kustomize build` 三個 overlay 都成功，diff 只有這三項 |
| 2 | 第 2 章 | 建 Issue Template 與 PR Template；開一個 Issue 寫五條驗收條件；從空 commit 開 Draft PR 列至少六項工項 | PR 列表顯示 0 of 6 tasks，Issue 頁面看到 linked PR |
| 3 | 第 3 章 | 用 Conventional Commits 做至少三個 commit，每個 push 後勾一項工項 | git log 每行都符合 `類型(範圍): 摘要`，PR 顯示 3 of 6 |
| 4 | 第 4 章 | 建 pr-check workflow，含測試、Draft 時跳過的映像檔建置、kubeconform 驗證 | Draft 時 image job 被 skip；故意弄壞 YAML 時 manifests job 紅燈 |
| 5 | 第 5 章 | 設 Branch Protection（需 PR、1 批准、status checks、只允許 Squash）與 CODEOWNERS；轉 Ready 並請同學 review | 自己按 merge 被拒；合併後 main 只多一個 commit，Issue 自動關閉 |
| 6 | 第 6 章 | 建 release workflow：semantic-release 打 tag，建置推到 ghcr.io（或 ACR），往 platform-gitops 開 PR 改 dev tag | 合併後 5 分鐘內 platform-gitops 出現機器人 PR，diff 只有 newTag |
| 7 | 第 7 章 | 建三個 Argo CD Application，dev 開 automated + selfHeal；合併 Lab 6 的 PR；手動 kubectl scale 觀察被改回 | `argocd app get` 顯示 Synced Healthy；scale 後 3 分鐘內 replicas 回到 Repo 的值 |
| 8 | 第 8 章 | 開 PR 晉升 staging、再晉升 prod；用 git revert 回滾 prod | 三個環境的 tag 在 git log 可追；revert 後 Argo CD 自動回舊版，沒用 argocd rollback |
| 9 | 第 9 章 | 裝 External Secrets Operator 接 Azure Key Vault（或本機用 Sealed Secrets 替代）；裝 Kyverno 拒收未簽章映像檔 | Git 裡 grep 不到任何 token；本機 build 的映像檔被 Kyverno 擋下 |
| 10 | 第 10 章 | 設 Argo CD Notifications 到 Slack 或 Discord；用一行 git log 列出 prod 所有變更 | 每次 sync 有通知；git log 的輸出每行都能對到一個 PR |

## 教材整體檢核表

做完十個 Lab 後，對照這張表確認自己的環境符合 GitOps：

- [ ] 所有環境的期望狀態都在 Config Repo，沒有「只在叢集裡」的設定
- [ ] 沒有人或 CI 持有叢集的寫入憑證，只有 Argo CD
- [ ] 每次部署都對應一個已合併的 PR，PR 對應一個 Issue
- [ ] 每個工作在開發前有 Draft PR 與工項 Checklist
- [ ] main 只能透過 PR 改，包含機器人與管理員
- [ ] prod 的變更需要指定角色批准
- [ ] 回滾用 git revert，沒有人用過 argocd app rollback 或 kubectl rollout undo
- [ ] 密碼不在 Git，映像檔有簽章且叢集驗證
- [ ] 有人手動改叢集時，3 分鐘內被改回或被告警
- [ ] 能用一行 git log 回答「某環境在某時間區間改了什麼」
