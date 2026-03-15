# Smart Mobility Dashboard (C++) - Kubernetes Manifests

Smart Mobility Dashboard C++アプリケーションをOpenShift上にデプロイするためのKubernetesマニフェストリポジトリです。
ArgoCDによるGitOpsデプロイをサポートしています。

ソースコードは [smart-mobility-dashboard-c](https://github.com/kmotojim/smart-mobility-dashboard-c) リポジトリのC++実装です。
C++版はフロントエンド（静的HTML/CSS/JS）とバックエンド（REST API）を単一プロセスで提供します。

## ディレクトリ構成

```
smart-mobility-dashboard-cpp-manifests/
├── argocd/                    # ArgoCD Application定義
│   ├── appproject.yaml        # AppProject定義
│   ├── application-dev.yaml   # 開発環境用Application
│   └── application-prod.yaml  # 本番環境用Application
├── base/                      # 基本のKubernetesマニフェスト
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── backend-deployment.yaml   # C++アプリ（ポート8080、UI + API）
│   ├── backend-service.yaml
│   ├── backend-route.yaml
│   └── e2e-test-job.yaml      # E2Eテスト（ArgoCD Post-Sync Hook）
└── overlays/                  # 環境別オーバーレイ
    ├── dev/                   # 開発環境
    │   └── kustomization.yaml
    └── prod/                  # 本番環境
        └── kustomization.yaml
```

## 前提条件

### 必須
- OpenShiftクラスタ
- ArgoCD がインストール済み

### 確認コマンド
```bash
# ArgoCDの確認
oc get pods -n openshift-gitops
```

---

## ArgoCD セットアップ

### 1. ArgoCDへのApplicationの登録

開発環境:
```bash
oc apply -f argocd/application-dev.yaml
```

本番環境:
```bash
oc apply -f argocd/application-prod.yaml
```

### 2. 手動でのデプロイ確認（Kustomize）

マニフェストをプレビュー:
```bash
# 開発環境
kustomize build overlays/dev

# 本番環境
kustomize build overlays/prod
```

直接適用する場合:
```bash
oc apply -k overlays/dev
```

### 環境別設定

| 設定項目 | 開発環境 (dev) | 本番環境 (prod) |
|---------|---------------|----------------|
| Namespace | smart-mobility-dev-cpp | smart-mobility-prod-cpp |
| Replicas | 1 | 2 |
| Image Tag | dev | latest |

---

## E2Eテスト（ArgoCD Post-Sync Hook）

ArgoCDがリソースをSyncするたびに、Post-Sync HookとしてE2Eテストが自動実行されます。

### 動作フロー

```
ArgoCD Sync完了
    ↓
Post-Sync Hook起動
    ↓
[init] wait-for-app: サービス起動待機（/health + /）
    ↓
[main] e2e-test: Cucumber + Playwright実行
    ↓
テスト結果（成功/失敗）
```

### E2Eテスト結果の確認

```bash
# E2EテストJobの一覧
oc get jobs -n smart-mobility-dev-cpp -l app.kubernetes.io/name=e2e-test

# E2EテストPodのログ確認
oc logs -n smart-mobility-dev-cpp -l app.kubernetes.io/name=e2e-test --tail=100

# 最新のE2EテストPodのログを全て表示
oc logs -n smart-mobility-dev-cpp $(oc get pods -n smart-mobility-dev-cpp -l app.kubernetes.io/name=e2e-test --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
```

### E2Eテストの手動再実行

```bash
# 既存のJobを削除して再作成（ArgoCD経由）
oc delete job e2e-test-job -n smart-mobility-dev-cpp
argocd app sync smart-mobility-dashboard-cpp-dev

# または直接Jobを実行
oc create -f base/e2e-test-job.yaml -n smart-mobility-dev-cpp
```

---

## リセット手順

デモ環境を初期状態に戻すための手順です。

### クイックリセット（アプリケーションのみ）

```bash
# ArgoCDアプリケーションを再同期
argocd app sync smart-mobility-dashboard-cpp-dev --force

# または、Deploymentを再起動
oc rollout restart deployment/smart-mobility-backend-cpp -n smart-mobility-dev-cpp
```

### E2Eテスト結果の削除

```bash
# E2EテストJobを削除
oc delete job -l app.kubernetes.io/name=e2e-test -n smart-mobility-dev-cpp
```

### フルリセット（Namespace削除）

```bash
# 1. ArgoCDアプリケーションを削除
oc delete -f argocd/application-dev.yaml

# 2. Namespaceの削除を確認
oc delete namespace smart-mobility-dev-cpp

# 3. Namespaceが削除されるまで待機
oc wait --for=delete namespace/smart-mobility-dev-cpp --timeout=120s

# 4. ArgoCDアプリケーションを再登録
oc apply -f argocd/application-dev.yaml
```

### 本番環境のリセット

```bash
# 1. ArgoCDアプリケーションを削除
oc delete -f argocd/application-prod.yaml

# 2. Namespaceの削除を確認
oc delete namespace smart-mobility-prod-cpp

# 3. Namespaceが削除されるまで待機
oc wait --for=delete namespace/smart-mobility-prod-cpp --timeout=120s

# 4. ArgoCDアプリケーションを再登録
oc apply -f argocd/application-prod.yaml
```

### 全環境の一括リセット

```bash
# 全てのArgoCDアプリケーションを削除
oc delete -f argocd/

# 両方のNamespaceを削除
oc delete namespace smart-mobility-dev-cpp smart-mobility-prod-cpp

# 削除完了を待機
oc wait --for=delete namespace/smart-mobility-dev-cpp namespace/smart-mobility-prod-cpp --timeout=180s

# アプリケーションを再デプロイ
oc apply -f argocd/
```

---

## イメージレジストリ

デフォルトでは以下のイメージを使用します:
- `image-registry.openshift-image-registry.svc:5000/smart-mobility-dev-cpp/smart-mobility-backend-cpp`

外部レジストリ（Quay.io）を使用する場合:
- `quay.io/rh_ee_kmotojim/smart-mobility-backend-cpp`

本番環境で外部レジストリを使用する場合、`overlays/prod/kustomization.yaml` を編集してください。
