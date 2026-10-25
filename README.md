# GitOps 實戰教材：從 Issue 到 Production

這份教材用一個完整的開發情境貫穿：平台組要為訂單服務加上「優惠券折抵」功能，從開 Issue、開 Draft PR 列工項，一路走到 Production 上線與回滾。每一章都接續同一個情境。

第一階段以 GitHub、GitHub Actions、Argo CD、AKS 示範；最後一章對照 Azure DevOps，作為下一階段整合的基礎。

## 目錄

| 章節 | 檔案 |
| --- | --- |
| 課程簡介與學習目標 | [docs/00-intro.md](docs/00-intro.md) |
| GitOps 核心概念 | [docs/01-concepts.md](docs/01-concepts.md) |
| 貫穿全書的開發情境 | [docs/02-scenario.md](docs/02-scenario.md) |
| 第 1 章 Repo 結構與分支策略 | [docs/ch01-repo-structure.md](docs/ch01-repo-structure.md) |
| 第 2 章 規劃：Issue、Draft PR 與工項 Checklist | [docs/ch02-planning-draft-pr.md](docs/ch02-planning-draft-pr.md) |
| 第 3 章 開發：Commit 規範與小步推送 | [docs/ch03-development.md](docs/ch03-development.md) |
| 第 4 章 CI：PR 驗證 Pipeline | docs/ch04-ci-pr-check.md（撰寫中） |
| 第 5 章 Review 與合併 | docs/ch05-review-merge.md（撰寫中） |
| 第 6 章 交付：Image 發布與 Config Repo PR | docs/ch06-release.md（撰寫中） |
| 第 7 章 CD：Argo CD 自動 Reconcile | docs/ch07-cd-argocd.md（撰寫中） |
| 第 8 章 環境晉升與回滾 | docs/ch08-promotion-rollback.md（撰寫中） |
| 第 9 章 Secrets 與安全 | docs/ch09-secrets-security.md（撰寫中） |
| 第 10 章 稽核與可觀測性 | docs/ch10-audit-observability.md（撰寫中） |
| Azure DevOps 整合預告 | docs/11-azure-devops-preview.md（撰寫中） |
| 實作練習與檢核表 | docs/12-labs.md（撰寫中） |
| 附錄：範本與名詞對照 | docs/13-appendix.md（撰寫中） |
