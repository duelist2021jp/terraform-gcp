---
name: terraform-gcp
description: >
  Use when the user wants to provision Google Cloud infrastructure with Terraform — covers
  module-based code structure, Google Cloud best practices via google-developer-knowledge MCP,
  latest provider versions via terraform MCP, readability through comments, and
  Deployment Manager vs Terraform feature parity awareness.
---

# Terraform × Google Cloud プロビジョニング スキル

このスキルは、Terraform を使って Google Cloud 環境をプロビジョニングする際に、
**Google Cloud ベストプラクティス準拠**・**module 構造**・**可読性**
を考慮したコーディングをガイドするものです。

---

## Step 1 — 要件の明確化

`ask_followup_question` ツールを使って、以下を確認する（未確定の項目のみ質問する）。

1. **対象 Google Cloud サービス** — GCE / VPC / GKE / Cloud SQL / Cloud Storage / Pub/Sub など何を構築するか。
2. **プロジェクト構成** — 単一プロジェクトか、複数プロジェクト（dev/stg/prod）を想定するか。
3. **リージョン** — デプロイ先リージョン（例: asia-northeast1）。
4. **既存リソース** — `data` ソースで参照すべき既存インフラがあるか。
5. **状態管理** — tfstate を Cloud Storage などのリモートバックエンドで管理するか。

---

## Step 2 — 最新プロバイダー・モジュールバージョンの確認

### 2-1. Google Cloud プロバイダー最新バージョンを取得

`mcp__terraform__get_latest_provider_version` ツールを呼び出す。

```
namespace: "hashicorp"
name:      "google"
```

取得したバージョンを `required_providers` ブロックに記載する。

### 2-2. 使用モジュールの最新バージョンを取得

構築対象サービスごとに `mcp__terraform__search_modules` → `mcp__terraform__get_module_details`
の順で公式モジュールを調査し、最新の `version` を確認する。

代表的な公式モジュールの例：

| Google Cloud サービス | モジュール検索キーワード |
|---|---|
| VPC / ネットワーク | `network google terraform-google-modules` |
| GKE | `kubernetes-engine google terraform-google-modules` |
| Cloud SQL | `sql-db google terraform-google-modules` |
| Cloud Storage | `cloud-storage google terraform-google-modules` |
| IAM | `iam google terraform-google-modules` |
| Project Factory | `project-factory google terraform-google-modules` |
| KMS | `kms google terraform-google-modules` |

---

## Step 3 — Google Cloud ベストプラクティスの確認

`mcp__google-developer-knowledge_24c2__search_documents` ツールを使い、構築対象サービスのベストプラクティスを調査する。

```
query: "<サービス名> best practices Terraform Google Cloud"
```

調査結果をもとに、以下の観点をコードに反映する。

- **最小権限の原則** — IAM ロールは事前定義済みロールまたはカスタムロールを使用し、基本ロール（`roles/owner`, `roles/editor`）はプロダクション環境で付与しない。
- **サービスアカウントの分離** — コンポーネントごとに専用のサービスアカウントを作成する。
- **暗号化** — Cloud KMS 管理キー（CMEK）の使用を検討する。保存データ・転送データの暗号化を有効化する。
- **ラベル付け** — `environment`, `project`, `team`, `managed-by = terraform` などのラベルを統一する（ラベルキー・値は小文字英数字・アンダースコア・ハイフンのみ）。
- **ログ・監査** — Cloud Audit Logs, VPC Flow Logs, Cloud Logging エクスポートなど監査ログを有効化する。
- **パブリックアクセス制限** — Cloud Storage の `public_access_prevention`, GCE インスタンスの外部 IP 割り当て制限など。
- **リモート State** — tfstate は Cloud Storage バケットに保存し、バージョニング・暗号化を有効化する。シークレットを State に保存しない。
- **ステートフルリソース保護** — Cloud SQL などステートフルリソースには `lifecycle { prevent_destroy = true }` を設定する。

---

## Step 4 — Deployment Manager との機能差異の考慮

以下の差異が関係する場合は、コード内コメントで注意書きを追記し、代替手段を実装する。

| 観点 | Deployment Manager | Terraform での対処 |
|---|---|---|
| ドリフト検出 | 組み込みの preview / update で検出 | `terraform plan` / `terraform refresh` で代替。`lifecycle { ignore_changes }` を意図的に使う場合は明示コメントを入れる |
| カスタムリソース | Python/Jinja2 テンプレートの type プロバイダー | `null_resource` + `local-exec` / `terraform-google-modules` で代替 |
| 依存関係の自動解決 | テンプレート内参照で暗黙解決 | `depends_on` を明示的に記述する。循環参照が起きやすいリソースは構造で分離する |
| ロールバック | 前回デプロイへの再適用 | `-target` + state 操作で部分ロールバック。`prevent_destroy = true` で誤削除を防ぐ |
| マルチプロジェクト展開 | DM は単一プロジェクトが基本 | `providers` マップ + `for_each` でマルチプロジェクト対応する |
| Secret Manager 参照 | テンプレート内変数で管理 | `data "google_secret_manager_secret_version"` で参照し、`sensitive = true` を付与する |
| API 有効化 | DM は既存 API 前提 | `google_project_service` リソース（または project-factory モジュール）で必要な API を明示的に有効化する |
| Parameter ファイル | YAML/JSON の変数ファイル | `terraform.tfvars` または `environments/<env>/terraform.tfvars` で代替する |
| 条件付きデプロイ | YAML の条件分岐 | `count = var.enabled ? 1 : 0` または `for_each` で代替する |

---

## Step 5 — module 構造でコードを生成する

### ディレクトリ構成（標準テンプレート）

Google Cloud の公式スタイルガイドに則り、`modules/` と `environments/` を分離する。

```
<project-root>/
├── main.tf               # ルートモジュール：module 呼び出しのエントリポイント
├── variables.tf          # ルートの入力変数定義
├── outputs.tf            # ルートの出力値定義
├── versions.tf           # terraform / required_providers バージョン固定
├── terraform.tfvars      # 変数の実値（gitignore 推奨）
│
├── modules/
│   ├── networking/       # VPC, Subnet, Cloud Router, Cloud NAT, Firewall
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/          # GCE Instance, Instance Template, MIG
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/         # Cloud SQL, Memorystore など
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── security/         # IAM, Service Account, KMS, Secret Manager
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
└── environments/
    ├── dev/
    │   ├── main.tf       # 環境固有のモジュール呼び出し
    │   ├── backend.tf    # GCS バックエンド設定
    │   └── terraform.tfvars
    ├── stg/
    └── prod/
```

> **複数環境の注意点**
> Terraform Workspaces よりも `environments/` ディレクトリ分割が推奨される。
> Workspace は state の分離のみでコードの差異を管理しにくいため。

### コーディングルール

1. **module 単位の分割**
   - 1 モジュール = 1 論理コンポーネント（例: `networking`, `compute`, `database`）
   - モジュール間の依存は `output` → `variable` で明示的に渡す。

2. **命名規則（Google Cloud スタイルガイド準拠）**
   - Terraform リソースラベルは `snake_case` で統一する（例: `web_server`, `main`）。
   - ハイフン（`-`）ではなくアンダースコア（`_`）を使用する。
   - 同一タイプで唯一のリソースは `main` と命名する（例: `google_compute_network.main`）。
   - リソースタイプをリソース名に繰り返さない（例: `google_compute_global_address.main` は NG: `main_global_address`）。
   - Google Cloud リソースの `name` 引数にはハイフン区切り（ケバブケース）を使用する（例: `name = "web-server"`）。

3. **Google Cloud ベストプラクティスの適用**
   - IAM バインディングは `google_project_iam_member`（additive）または `google_project_iam_binding`（authoritative）を使い分ける。ポリシー全体を上書きする `google_project_iam_policy` は原則使用しない。
   - ラベル（`labels`）は変数化し、`merge(local.common_labels, var.extra_labels)` パターンで全リソースに統一付与する。
   - 機密情報は `data "google_secret_manager_secret_version"` 参照を使用し、ハードコーディングしない。`sensitive = true` を付与する。
   - API 有効化（`google_project_service`）は `disable_on_destroy = false` を設定する（モジュール破棄時に意図しない API 無効化を防ぐ）。

4. **バージョン固定**
   - `versions.tf` に `required_providers` と `required_version` を記述し、バージョンを固定する。
   - プロバイダーバージョンは Step 2 で取得した最新バージョンを使用する。

5. **可読性のためのコメント**
   - 各 `main.tf` の先頭にファイルヘッダーコメントを記述する。
   - 各リソースブロックの直前に役割・設定意図をコメントする。Deployment Manager との差異が関係する箇所は `[DM 差異]` プレフィックスで明示する。
   - 変数には `description` と `type` を必ず記述する。
   - 複雑な `locals` や `for_each` ロジックには説明コメントを付ける。

### ファイルヘッダーコメント

各 `main.tf` の先頭に記述する：

```hcl
##############################################################################
# モジュール名: <module-name>
# 概要        : <何を作るか一行で>
# 依存モジュール: <呼び出し元や依存するモジュール>
# 最終更新    : <YYYY-MM-DD>
##############################################################################
```

---

## Step 6 — コード生成

以下の順序でファイルを生成する。`write_file` または `apply_diff` ツールを使用する。

1. `versions.tf` — terraform / Google Cloud プロバイダーのバージョン固定（Step 2 の値を使用）
2. `variables.tf` — プロジェクト・リージョン・共通ラベル変数
3. `modules/<service>/main.tf` — 各サービスのリソース定義（Step 4 の差異コメントを含む）
4. `modules/<service>/variables.tf` / `outputs.tf`
5. ルート `main.tf` — モジュール呼び出し
6. `outputs.tf` — 上位層への出力

---

## Step 7 — コードレビューチェックリスト

生成後、以下を確認してからユーザーに提示する。

- [ ] `versions.tf` に最新の `google` プロバイダーバージョンが固定されているか
- [ ] すべてのリソースに `labels` 変数が付与されているか（ラベルキー・値は小文字英数字・アンダースコア・ハイフンのみ）
- [ ] ラベル値に使用する変数（`project_name`, `team` 等）に `validation` ブロックが設定されているか（`terraform plan` 段階での文字制限違反を検出するため。落とし穴 #3 参照）
- [ ] ハードコードされた機密情報（パスワード・サービスアカウントキー等）がないか
- [ ] `variable` ブロックに `description` と適切な `type` が設定されているか
- [ ] `output` ブロックに `description` が設定されているか
- [ ] module 呼び出しが `main.tf` に集約されているか
- [ ] Google Cloud ベストプラクティス（Step 3 で取得した内容）が反映されているか
- [ ] IAM ロールに基本ロール（`roles/owner`, `roles/editor`, `roles/viewer`）がプロダクション環境で使用されていないか
- [ ] `google_project_iam_policy`（ポリシー全体上書き）が使われていないか（additive/authoritative のいずれかを意図的に選択しているか）
- [ ] `google_project_service` に `disable_on_destroy = false` が設定されているか
- [ ] Cloud SQL などステートフルリソースに `lifecycle { prevent_destroy = true }` が設定されているか
- [ ] Deployment Manager との差異が関係する箇所に `[DM 差異]` コメントが入っているか
- [ ] `lifecycle { ignore_changes }` / `depends_on` を使った箇所に理由コメントがあるか
- [ ] module 間に依存関係がある場合、出力参照に加えて `depends_on = [module.<name>]` が明示されているか（落とし穴 #6 参照）
- [ ] Secret Manager を参照している変数に `sensitive = true` が付与されているか
- [ ] tfstate バックエンドに Cloud Storage が設定されているか（本番環境）

---

## 参考: versions.tf テンプレート

```hcl
##############################################################################
# バージョン制約
# Google Cloud プロバイダーは terraform MCP で確認した最新バージョンに固定する。
##############################################################################
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> <LATEST_VERSION>"  # Step 2 で取得した最新バージョン
    }
  }

  # リモートバックエンド（本番環境では必須）
  # [DM 差異] Deployment Manager は Google 側で状態管理するが、
  # Terraform は tfstate を明示的に管理する必要がある。
  # backend "gcs" {
  #   bucket = "<tfstate-bucket-name>"
  #   prefix = "<project>/<env>"
  # }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

---

## 参考: 共通ラベルテンプレート

```hcl
##############################################################################
# 共通ラベル
# Google Cloud のラベルベストプラクティスに従い、環境・コンポーネント・
# 管理ツールを統一ラベルとして全リソースに付与する。
# ラベルキー・値は小文字、数字、アンダースコア、ハイフンのみ使用可能。
##############################################################################
locals {
  common_labels = {
    managed_by  = "terraform"
    environment = var.environment
    project     = var.project_name
    team        = var.team
  }
}

# 使用例（各リソースで merge して追加ラベルを受け付ける）
# labels = merge(local.common_labels, { component = "database" })
```

---

## 参考: IAM ベストプラクティスコメント例

```hcl
# [IAM ベストプラクティス] 基本ロール (roles/editor 等) はプロダクション環境で使用しない。
# コンポーネントごとに専用サービスアカウントを作成し、必要最小限の事前定義済みロールを付与する。
resource "google_service_account" "app" {
  account_id   = "app-service-account"
  display_name = "Application Service Account"
  description  = "Service account for the application component"
}

# [IAM ベストプラクティス] google_project_iam_member (additive) を使用する。
# google_project_iam_policy は既存のバインディングを上書きするため原則使用しない。
resource "google_project_iam_member" "app_storage_reader" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

---

## 参考: API 有効化コメント例

```hcl
# [DM 差異] Deployment Manager は既存の有効 API を前提とするが、
# Terraform では google_project_service で必要な API を明示的に有効化する。
# disable_on_destroy = false: モジュール破棄時に他のサービスに影響しないよう API は無効化しない。
resource "google_project_service" "compute" {
  project            = var.project_id
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}
```

---

## 付録: Google Cloud Terraform での既知の落とし穴

### 1. `google_project_iam_policy` による既存バインディングの意図しない削除

| 項目 | 内容 |
|---|---|
| 変更種別 | **既知の落とし穴** |
| 原因 | `google_project_iam_policy` はプロジェクトの IAM ポリシー**全体**を上書きする。Terraform 管理外のバインディングが削除される |

**対処**: `google_project_iam_member`（追加）または `google_project_iam_binding`（ロール単位での管理）を使用する。

```hcl
# 非推奨: ポリシー全体を上書きする（既存バインディングが失われる）
# resource "google_project_iam_policy" "project" { ... }

# 推奨: additive バインディング（既存バインディングへの影響なし）
resource "google_project_iam_member" "example" {
  project = var.project_id
  role    = "roles/compute.viewer"
  member  = "serviceAccount:${google_service_account.main.email}"
}
```

### 2. `google_project_service` の `disable_on_destroy` デフォルト値

| 項目 | 内容 |
|---|---|
| 変更種別 | **既知の落とし穴** |
| 原因 | `disable_on_destroy` のデフォルト値は `true`。モジュール削除時に API が無効化され、他のサービスが影響を受ける |

**対処**: 常に `disable_on_destroy = false` を明示的に設定する。

### 3. ラベルキー・値の文字制限

| 項目 | 内容 |
|---|---|
| 変更種別 | **既知の落とし穴** |
| 原因 | Google Cloud のラベルキー・値は小文字英字・数字・アンダースコア（`_`）・ハイフン（`-`）のみ。大文字・全角文字・スペースはエラーになる |

**エラー症状**:
```
Error: googleapi: Error 400: Invalid user label: project with value My First Project.
  Reason: value "My First Project" does not conform to regular expression
  "[\p{Ll}\p{Lo}\p{N}_-]{0,63}"; character "M" at position 0 is not a
  non-uppercased letter, digit, hyphen, or underscore.
```

**対処**: ラベルキー・値はすべて小文字で定義する。`managed_by = "terraform"` のように統一する。

特に `project_name` や `team` などラベル値に使用する変数には、`terraform plan` の段階でエラーを検出できるよう `validation` ブロックを必ず設定する。

```hcl
variable "project_name" {
  description = "ラベル付けに使用するプロジェクト略称（小文字英数字・ハイフン・アンダースコアのみ / 大文字・スペース不可）"
  type        = string

  # [既知の落とし穴] Google Cloud ラベル値は小文字英数字・アンダースコア・ハイフンのみ。
  # 大文字・スペース・全角文字を含む値を渡すと apply 時に Error 400 になるため、
  # terraform plan の段階で検出できるよう validation を設定する。
  validation {
    condition     = can(regex("^[a-z0-9_-]+$", var.project_name))
    error_message = "project_name はラベル値として使用されるため、小文字英数字・アンダースコア・ハイフンのみ使用できます（大文字・スペース・全角文字は不可）。例: my-first-project"
  }
}
```

> **実例**: `project_name = "My First Project"` と設定した場合、`terraform apply` 時に Error 400 が発生する。
> `project_name = "my-first-project"` のように修正すること。

### 4. Cloud SQL の `deletion_protection` デフォルト

| 項目 | 内容 |
|---|---|
| 変更種別 | **既知の落とし穴** |
| 原因 | `google_sql_database_instance` の `deletion_protection` はデフォルト `true`。`terraform destroy` で削除しようとするとエラーになる |

**対処**: 意図的な削除には `deletion_protection = false` を設定してから `terraform apply` し、その後 `terraform destroy` を実行する。本番環境では `true` のままにし `lifecycle { prevent_destroy = true }` も合わせて設定する。

```hcl
resource "google_sql_database_instance" "main" {
  name             = "db-${var.environment}"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier = "db-f1-micro"
  }

  # [ベストプラクティス] 本番環境では削除保護を有効化する。
  deletion_protection = var.environment == "prod" ? true : false

  lifecycle {
    # [ベストプラクティス] ステートフルリソースの誤削除防止。
    prevent_destroy = true
  }
}
```

### 5. GKE ノードプール変更時の再作成


| 項目 | 内容 |
|---|---|
| 変更種別 | **既知の落とし穴** |
| 原因 | `google_container_node_pool` の一部パラメータ変更は in-place 更新不可でノードプール再作成が発生する |

**対処**: ノードプール変更時はブルーグリーン更新を使用するか、`lifecycle { create_before_destroy = true }` を設定する。

```hcl
resource "google_container_node_pool" "main" {
  name       = "main-node-pool"
  cluster    = google_container_cluster.main.name
  node_count = var.node_count

  lifecycle {
    # ノードプール再作成が必要な変更時は新規作成してから古いものを削除する
    create_before_destroy = true
  }
}
```

### 6. module 間の `depends_on` 未設定による依存関係の曖昧化

| 項目 | 内容 |
|---|---|
| 変更種別 | **既知の落とし穴** |
| 原因 | module の出力参照（`module.A.output_value`）は参照先の**単一リソース**への依存のみ Terraform が推論する。呼び出し先 module 内の IAM バインディングやオブジェクト等、他リソースの完了は保証されない |

**エラー症状**: apply は成功するが、依存先 module の一部リソース（IAM 設定など）が完了する前に依存元 module のリソース作成が始まり、権限エラーや 404 が発生することがある。

**対処**: module 間に依存関係がある場合は `depends_on = [module.<name>]` を明示的に設定し、依存先 module 全体の完了を待つ。`depends_on` を設定した箇所には理由コメントを必ず付ける。

```hcl
module "cdn" {
  source      = "./modules/cdn"
  bucket_name = module.storage.bucket_name  # 出力参照だけでは module 全体の完了を保証しない

  # storage モジュール全体（バケット・IAM・オブジェクト）の作成完了後に
  # cdn モジュールを開始することを明示する。
  depends_on = [module.storage]
}
```

> **[DM 差異]** Deployment Manager はテンプレート内参照で依存関係を暗黙解決するが、
> Terraform では module 間の依存は `depends_on` で明示的に記述する必要がある。
