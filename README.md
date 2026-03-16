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
├── tekton/                    # Tekton CI/CDパイプライン定義
│   ├── backend-ci-pipeline-cpp.yaml   # CIパイプライン（ビルド・テスト・SonarQube・イメージプッシュ）
│   ├── event-listener-cpp.yaml        # GitHub Webhook EventListener + RBAC
│   ├── trigger-binding-cpp.yaml       # CI用 TriggerBinding
│   ├── trigger-template-cpp.yaml      # CI用 TriggerTemplate
│   ├── e2e-and-pr-pipeline.yaml       # E2Eテスト + PR作成パイプライン
│   ├── e2e-event-listener.yaml        # ArgoCD Sync EventListener
│   ├── e2e-trigger-binding.yaml       # E2E用 TriggerBinding
│   └── e2e-trigger-template.yaml      # E2E用 TriggerTemplate
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

## Tekton CI/CD パイプライン

本リポジトリの `tekton/` ディレクトリには、ソースコードリポジトリ（smart-mobility-dashboard-c）のCI/CDを実現するTektonパイプライン定義が含まれます。

### 前提条件（Tekton）

#### クラスタにインストール済みであること

| コンポーネント | 説明 | 確認コマンド |
|---------------|------|-------------|
| OpenShift Pipelines (Tekton) | パイプライン実行基盤。ClusterTask（`git-clone`, `buildah`, `skopeo-copy`）を提供 | `oc get pods -n openshift-pipelines` |
| Tekton Triggers | EventListener / TriggerBinding / TriggerTemplate の実行基盤。ClusterRole（`tekton-triggers-eventlistener-roles`, `tekton-triggers-eventlistener-clusterroles`）を提供 | `oc get pods -n openshift-pipelines -l app=tekton-triggers` |

#### tekton/ 内で定義済みのリソース

以下のリソースは `tekton/event-listener-cpp.yaml` 内に含まれており、`oc apply -f tekton/` で自動作成されます。

| 種類 | 名前 | 説明 |
|------|------|------|
| ServiceAccount | `tekton-triggers-sa` | EventListener用のサービスアカウント |
| RoleBinding | `tekton-triggers-eventlistener-binding` | EventListenerのNamespaceスコープ権限 |
| ClusterRoleBinding | `tekton-triggers-cpp-clusterbinding` | EventListenerのクラスタスコープ権限 |
| Secret | `github-webhook-secret` | GitHub Webhookの署名検証用シークレット |

#### 事前に手動作成が必要なリソース

以下のリソースはパイプラインから参照されますが、マニフェストには含まれていないため手動で作成してください。

| 種類 | 名前 | Namespace | 用途 | 作成例 |
|------|------|-----------|------|--------|
| Secret | `github-token` | `smart-mobility-dev-cpp` | E2Eテスト成功時のPR自動作成（`gh` CLI用） | 下記参照 |

```bash
# GitHub Personal Access Token（repo権限が必要）
oc create secret generic github-token \
  --from-literal=token=ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX \
  -n smart-mobility-dev-cpp
```

#### pipeline ServiceAccount の権限設定

Tekton PipelineRunはデフォルトで `pipeline` ServiceAccountを使用します。以下の権限が必要です。

**内部レジストリへのプッシュ権限（buildahタスク）:**
```bash
# pipeline SAに内部レジストリへのpush権限を付与
oc policy add-role-to-user system:image-builder system:serviceaccount:smart-mobility-dev-cpp:pipeline -n smart-mobility-dev-cpp
```

**外部レジストリ（Quay.io）の認証情報（skopeo-copyタスク）:**
```bash
# Quay.ioのdockerconfigjsonをpipeline SAにリンク
oc create secret docker-registry quay-auth \
  --docker-server=quay.io \
  --docker-username=<USERNAME> \
  --docker-password=<PASSWORD> \
  -n smart-mobility-dev-cpp

oc secrets link pipeline quay-auth -n smart-mobility-dev-cpp
```

#### SonarQube C++ プラグインのセットアップ

SonarQube Community Edition は C++ を標準サポートしていないため、[sonar-cxx](https://github.com/SonarOpenCommunity/sonar-cxx) コミュニティプラグインのインストールが必要です。
プラグインがない場合、SonarQube はソースコードを認識できず、カバレッジデータが `Imported coverage data for 0 files` となります。

**1. プラグイン永続化用の PVC を作成:**
```bash
oc apply -f - -n sonarqube <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-plugins
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF
```

**2. SonarQube Deployment にプラグインボリュームをマウント:**
```bash
# SonarQubeのDeployment名を確認
oc get deployment -n sonarqube

# ボリュームとマウントを追加（Deployment名は環境に合わせて変更）
oc patch deployment docker-sonarqube-git -n sonarqube --type=json -p '[
  {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"plugins","persistentVolumeClaim":{"claimName":"sonarqube-plugins"}}},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"plugins","mountPath":"/opt/sonarqube/extensions/plugins"}}
]'

# Pod が再起動するまで待機
oc rollout status deployment/docker-sonarqube-git -n sonarqube --timeout=120s
```

**3. sonar-cxx プラグイン JAR をコピー:**
```bash
# sonar-cxx の最新リリースを取得（SonarQubeバージョンに合ったものを選択）
# https://github.com/SonarOpenCommunity/sonar-cxx/releases
curl -L -o /tmp/sonar-cxx-plugin.jar \
  "https://github.com/SonarOpenCommunity/sonar-cxx/releases/download/cxx-2.3.0/sonar-cxx-plugin-2.3.0.1445.jar"

# SonarQube Pod にコピー
SONAR_POD=$(oc get pods -n sonarqube -l app=docker-sonarqube-git -o jsonpath='{.items[0].metadata.name}')
oc cp /tmp/sonar-cxx-plugin.jar ${SONAR_POD}:/opt/sonarqube/extensions/plugins/ -n sonarqube

# SonarQube を再起動してプラグインを読み込み
oc rollout restart deployment/docker-sonarqube-git -n sonarqube
oc rollout status deployment/docker-sonarqube-git -n sonarqube --timeout=120s
```

**4. プラグインのインストール確認:**
```bash
# インストール済みプラグイン一覧を確認
curl -s -u admin:<SONAR_PASSWORD> \
  "http://docker-sonarqube-git.sonarqube.svc.cluster.local:9000/api/plugins/installed" | \
  python3 -c "import json,sys; [print(f'  {p[\"name\"]} v{p[\"version\"]}') for p in json.load(sys.stdin).get('plugins',[])]"

# 「C++ (Community)」が表示されれば成功
```

> **注意:** PVC を使わずに `oc cp` だけでプラグインを配置した場合、Pod の再起動でプラグインが消失します。必ず PVC をマウントしてください。

#### C++ カバレッジの仕組み

CIパイプラインでは `gcov` / `gcovr` を使って C++ のテストカバレッジを計測し、SonarQube に連携しています。

```
cmake (Debug + Coverage) → テスト実行 → gcovr → coverage.xml → SonarQube
```

| 項目 | 値 |
|------|-----|
| カバレッジツール | gcov (GCC内蔵) + gcovr (レポート生成) |
| CMakeフラグ | `-DENABLE_COVERAGE=ON` (`--coverage -fprofile-arcs -ftest-coverage`) |
| レポート形式 | SonarQube Generic Coverage Format (`--sonarqube`) |
| SonarQube パラメータ | `sonar.coverageReportPaths=coverage.xml` |

> **C++ 固有の考慮事項:** GCC の gcov はコンパイラが生成する例外処理パス（RAII/スタック巻き戻し）をブランチとしてカウントするため、通常のテストではカバー不可能なブランチが多数報告されます。パイプラインでは `--exclude-throw-branches` フラグの使用と、ブランチデータの除外処理により、行カバレッジのみを SonarQube に報告しています。

#### その他の前提

| 項目 | 説明 |
|------|------|
| ビルド用イメージ | `image-registry.openshift-image-registry.svc:5000/cluster-admin-devspaces/udi-cpp-dev:latest` が利用可能であること |
| SonarQube | `http://docker-sonarqube-git.sonarqube.svc.cluster.local:9000` でアクセス可能であること（C++プラグイン導入済み） |
| GitHub Webhook | ソースリポジトリのWebhook設定で、EventListenerのRouteURLとシークレットを登録すること |

### パイプライン構成

| パイプライン | 説明 |
|-------------|------|
| `backend-ci-pipeline-cpp` | CMakeビルド、単体テスト、カバレッジ取得、SonarQube静的解析、コンテナイメージビルド・プッシュ |
| `e2e-and-pr-pipeline` | ArgoCD sync後のE2Eテスト実行、成功時にdevelop→mainへのPR自動作成 |

### CI パイプライン（GitHub Push トリガー）

GitHub Webhookからのpushイベントで自動起動します。

```
GitHub Push → EventListener → TriggerBinding → TriggerTemplate → PipelineRun
                                                                    ↓
                                                              git-clone
                                                                    ↓
                                                          cmake-build-and-test
                                                                    ↓
                                                            sonarqube-scan
                                                                    ↓
                                                              buildah (イメージビルド)
                                                                    ↓
                                                            skopeo-copy (外部レジストリ)
```

セットアップ:
```bash
oc apply -f tekton/event-listener-cpp.yaml
oc apply -f tekton/trigger-binding-cpp.yaml
oc apply -f tekton/trigger-template-cpp.yaml
oc apply -f tekton/backend-ci-pipeline-cpp.yaml
```

### E2E + PR パイプライン（ArgoCD Sync トリガー）

ArgoCD syncの完了通知で自動起動し、E2Eテスト成功後にdevelop→mainへのPRを作成します。

```
ArgoCD Sync通知 → EventListener → TriggerBinding → TriggerTemplate → PipelineRun
                                                                        ↓
                                                                  wait-for-app
                                                                        ↓
                                                                  git-clone-e2e
                                                                        ↓
                                                                  run-e2e-tests
                                                                        ↓
                                                                  create-pr (develop→main)
```

セットアップ:
```bash
oc apply -f tekton/e2e-event-listener.yaml
oc apply -f tekton/e2e-trigger-binding.yaml
oc apply -f tekton/e2e-trigger-template.yaml
oc apply -f tekton/e2e-and-pr-pipeline.yaml
```

### 全Tektonリソースの一括適用

```bash
oc apply -f tekton/
```

---

## イメージレジストリ

デフォルトでは以下のイメージを使用します:
- `image-registry.openshift-image-registry.svc:5000/smart-mobility-dev-cpp/smart-mobility-backend-cpp`

外部レジストリ（Quay.io）を使用する場合:
- `quay.io/example_kmotojim/smart-mobility-backend-cpp`

本番環境で外部レジストリを使用する場合、`overlays/prod/kustomization.yaml` を編集してください。
