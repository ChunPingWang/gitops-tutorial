# 課程簡介與學習目標

這份教材用一個完整的開發情境貫穿：平台組要為訂單服務加上「優惠券折抵」功能，從開 Issue、開 Draft PR 列工項，一路走到 Production 上線與回滾。每一章都接續同一個情境，讀完後你會親手走過一次 GitOps 的完整生命週期。

## 適用對象

- 已會基本 Git 操作（clone、branch、commit、push）的開發者
- 想把部署流程從「手動 kubectl apply」改成以 Git 為唯一真相來源的 DevOps 或 SRE
- 準備把現有流程搬到 Azure DevOps 的團隊（本教材第一階段以 GitHub 示範，最後一章對照 Azure DevOps）

## 學習目標

1. 說明 GitOps 的四大原則，並分辨 Push 與 Pull 兩種部署模式的差異。
2. 建立 App Repo 與 Config Repo 分離的結構，並選擇合適的分支策略。
3. 用 Issue 定義需求與驗收條件，用 Draft PR 的 Checklist 管理工項並與 Reviewer 提早對齊。
4. 設計 PR 驗證 Pipeline，讓每次 push 都自動跑測試、建置、掃描與 Manifest 驗證。
5. 透過 Argo CD 以 Pull 模式同步叢集，完成 dev → staging → prod 的環境晉升與 git revert 回滾。
6. 把 Secrets 與映像檔簽章納入流程，並用 Git 歷史作為稽核軌跡。

## 閱讀方式

- 第 1 章到第 10 章依情境時間順序排列，建議按序閱讀。
- 每章結尾有「本章在情境中的位置」，說明主角團隊此時做到哪裡。
- 「實作練習與檢核表」一節列出每章對應的 Lab，可邊讀邊做。
- 範例指令以 GitHub、GitHub Actions、Argo CD、Azure Kubernetes Service（AKS）為基礎；工具可替換，流程不變。
