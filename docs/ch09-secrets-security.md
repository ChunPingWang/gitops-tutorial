# 第 9 章 Secrets 與安全

Git 是真相來源，但密碼不能明文進 Git。做法是在 Git 裡放「去哪裡拿密碼」的宣告，密碼本體留在 Azure Key Vault，由叢集內的控制器在執行期取回。

## 三種做法的比較

| 做法 | Git 裡放什麼 | 密碼在哪 | 輪替密碼時 | 適合 |
| --- | --- | --- | --- | --- |
| External Secrets Operator | ExternalSecret，指向 Key Vault 的 key 名稱 | Azure Key Vault | 改 Key Vault，控制器自動同步，Git 不動 | 已有中央密碼管理的團隊，本教材選這個 |
| Sealed Secrets | 用叢集公鑰加密的 SealedSecret | Git，加密形式 | 重新加密並開 PR | 沒有外部密碼管理的小團隊 |
| SOPS + Age 或 Key Vault | 用 SOPS 加密的 YAML | Git，加密形式 | 重新加密並開 PR | Flux 使用者，或要在 Git 裡看到 diff |

## ExternalSecret 範例（小林在第 6 天開的 PR）

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: order-service-coupon
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: order-service-coupon        # 產生的 Kubernetes Secret 名稱
  data:
    - secretKey: COUPON_SERVICE_TOKEN
      remoteRef:
        key: order-service-coupon-token   # Key Vault 裡的 secret 名稱
```

這個 YAML 可以安心進 Git：它只說「到 Key Vault 拿叫 order-service-coupon-token 的那個」。平台組在三個環境的 Key Vault 都建了同名的 secret，所以 ExternalSecret 放在 base 就好。

ClusterSecretStore 用 AKS 的 Workload Identity 連 Key Vault，沒有任何長期憑證存在叢集。

## 映像檔簽章與驗證

第 6 章 CI 用 cosign 簽了映像檔。叢集端用 Kyverno 政策拒絕沒簽章的映像檔：

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-acr-images
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences: ["acrplatform.azurecr.io/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/platform-org/order-service/.github/workflows/release.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
```

效果：只有 order-service 的 release workflow 在 main 建置的映像檔能跑。有人在本機 build 一個推到 ACR，或從 feature branch 建置，都被拒。這是供應鏈安全的最小實作。

## 最小權限

| 誰 | 能做 | 不能做 |
| --- | --- | --- |
| 開發者 | 合併 App Repo；在 Config Repo 改 dev 與 staging | 批准 prod 目錄；在叢集執行 kubectl 寫入 |
| CI（GitHub Actions） | 推映像檔到 ACR；在 Config Repo 開 PR | 讀叢集；合併 Config Repo（由 auto-merge 機制依標籤合併） |
| Argo CD | 讀 Config Repo；寫自己叢集的指定 namespace | 讀其他叢集；寫 Git |
| 平台工程師 | 批准 prod；緊急時自審 incident PR | 繞過 PR 直接 push main |
| 人類對叢集 | kubectl get、logs、port-forward | kubectl apply、edit、delete（RBAC 擋下） |

CI 用 OIDC（第 6 章的 id-token: write 與 azure/login）登入 Azure，沒有存 service principal 密碼。Argo CD 讀 GitHub 用 GitHub App，不用人的 token。

## Config Repo 的安全掃描

pr-check 多一步 gitleaks，任何看起來像密碼的字串進 PR 就紅燈。小林第 2 天曾把本機測試用的 token 寫進 ConfigMap，gitleaks 擋下，改成 ExternalSecret。

## 本章在情境中的位置

ExternalSecret 在第 6 天合併，第 7 天 prod 晉升前 Key Vault 已有 token。整個情境中沒有任何密碼出現在 Git、PR 留言或 CI 日誌。
