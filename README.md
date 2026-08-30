# Terraform GCP Skill

Google Cloud 環境を Terraform でプロビジョニングする際に、Google Cloud のベストプラクティス、再利用可能なモジュール構成、Deployment Manager との機能差異、既知の落とし穴を踏まえたコード生成を支援するスキルです。

## 提供するガイド

- Google Cloud サービスごとのベストプラクティスをコードに反映
- Google Cloud プロバイダー（`hashicorp/google`）の最新バージョンを確認し、`versions.tf` に固定
- Terraform Registry の公式モジュール（`terraform-google-modules`）を評価して再利用
- ネットワーク、コンピュート、データベース、セキュリティを論理単位でモジュール化
- ラベル、命名規則、機密情報、IAM、Secret Manager などの設計方針を統一
- Deployment Manager と Terraform の機能差異をコメントと代替実装で明示
- ラベル文字制限、`disable_on_destroy`、`deletion_protection` など既知の落とし穴を回避

## 前提条件

- Terraform 1.6.0 以降
- gcloud CLI で Google Cloud に認証済みであること
- Google Cloud プロバイダーを実行する ID に、対象リソースを作成・参照するための最小限の権限があること
- Google Cloud ベストプラクティスを取得する google-developer-knowledge MCP
- プロバイダーバージョンおよびモジュールを検索する Terraform MCP

> MCP が利用できない場合も、標準構成、セキュリティ方針、レビュー項目は適用できます。プロバイダーバージョンは Terraform Registry で確認してください。

## 利用フロー

1. 対象 Google Cloud サービスや構成（プロジェクト・リージョン・環境数・状態管理）を確認します。
2. Terraform MCP で `hashicorp/google` の最新バージョンを確認します。
3. 必要に応じて `terraform-google-modules` の公式モジュールを検索・評価します。
4. google-developer-knowledge MCP で対象サービスのベストプラクティスを調査します。
5. Deployment Manager との差異が関係する設計を特定し、代替実装とコメントを追加します。
6. ルートモジュールとサービス別モジュールに分割して Terraform コードを生成します。
7. レビュー項目を確認して `terraform fmt`、`terraform validate`、`terraform plan` を実行します。

## 推奨ディレクトリ構成

```text
<project-root>/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── terraform.tfvars
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── database/
│   └── security/
└── environments/
    ├── dev/
    ├── stg/
    └── prod/
```

環境差分は Terraform Workspaces よりも `environments/` ディレクトリで分割します。Workspace は state を分離できますが、環境ごとのコード差分を管理しにくいためです。

## コーディング規約

- 1 モジュールは 1 論理コンポーネントとし、依存関係は `output` から `variable` へ明示的に渡します。
- Terraform のリソースラベルは `snake_case` に統一し、同一タイプで唯一のリソースは `main` と命名します（例: `google_compute_network.main`）。
- リソースタイプをリソース名に繰り返さず、Google Cloud リソースの `name` 引数にはケバブケースを使用します（例: `name = "web-server"`）。
- 変数と出力には `description` と適切な `type` を記述します。
- 共通ラベルは `locals` に定義し、すべてのリソースへ `merge()` で付与します。ラベルキー・値は小文字英数字・アンダースコア・ハイフンのみ使用できます。
- IAM バインディングは `google_project_iam_member`（additive）または `google_project_iam_binding` を使い分け、`google_project_iam_policy`（ポリシー全体上書き）は原則使用しません。
- 機密情報はハードコードせず、`data "google_secret_manager_secret_version"` と `sensitive = true` を使用します。
- `depends_on`、`ignore_changes`、`prevent_destroy` を使う場合は、その理由をコメントで残します。
- Deployment Manager との差異が関係する箇所には `[DM 差異]` プレフィックスでコメントを明示します。

### 共通ラベル

```hcl
locals {
  common_labels = {
    managed_by  = "terraform"
    environment = var.environment
    project     = var.project_name
    team        = var.team
  }
}

# labels = merge(local.common_labels, { component = "database" })
```

## Deployment Manager との差異

| Deployment Manager | Terraform での対応 |
| --- | --- |
| 組み込みの preview / update でドリフト検出 | `terraform plan` / `terraform refresh` |
| Python/Jinja2 テンプレートのカスタムリソース | `null_resource` + `local-exec` または `terraform-google-modules` |
| テンプレート内参照による暗黙の依存解決 | `depends_on` を明示的に記述 |
| 前回デプロイへの再適用でロールバック | `-target` + state 操作、`prevent_destroy = true` |
| 単一プロジェクトが基本 | `providers` マップ + `for_each` でマルチプロジェクト対応 |
| テンプレート内変数での Secret 管理 | `data "google_secret_manager_secret_version"` と `sensitive = true` |
| 既存 API を前提 | `google_project_service` で API を明示的に有効化 |
| YAML/JSON のパラメータファイル | `terraform.tfvars` または環境別の tfvars |
| YAML の条件分岐 | `count` または `for_each` |

差異が実装に影響する箇所には、次のようなコメントを記述します。

```hcl
# [DM 差異] Deployment Manager は既存の有効 API を前提とするが、
# Terraform では google_project_service で必要な API を明示的に有効化する。
resource "google_project_service" "compute" {
  project            = var.project_id
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}
```

## 既知の落とし穴

- `google_project_iam_policy` はプロジェクトの IAM ポリシー全体を上書きし、既存バインディングを削除する。`google_project_iam_member` / `google_project_iam_binding` を使用する。
- `google_project_service` の `disable_on_destroy` はデフォルト `true`。他サービスへの影響を防ぐため `false` を明示する。
- ラベルキー・値は小文字英数字・アンダースコア・ハイフンのみ有効。大文字や全角文字を含むと `apply` 時に Error 400 になるため、ラベル値に使う変数には `validation` ブロックを設定する。
- `google_sql_database_instance` の `deletion_protection` はデフォルト `true`。誤削除防止のため本番環境では維持し、`lifecycle { prevent_destroy = true }` も併用する。
- `google_container_node_pool` の一部パラメータ変更は再作成を伴う。`lifecycle { create_before_destroy = true }` を検討する。
- module の出力参照は参照先の単一リソースへの依存のみ推論される。module 間に依存関係がある場合は `depends_on = [module.<name>]` を明示する。

## レビュー チェックリスト

- [ ] `versions.tf` に Terraform と Google Cloud プロバイダーのバージョン制約がある
- [ ] すべてのリソースに共通ラベルが付与されている（小文字英数字・アンダースコア・ハイフンのみ）
- [ ] ラベル値に使用する変数に `validation` ブロックが設定されている
- [ ] 機密情報がハードコードされていない
- [ ] 変数と出力に `description` と適切な `type` がある
- [ ] モジュール呼び出しがルートの `main.tf` に集約されている
- [ ] IAM ロールに基本ロール（`roles/owner`, `roles/editor`, `roles/viewer`）が本番環境で使用されていない
- [ ] `google_project_iam_policy`（ポリシー全体上書き）が使われていない
- [ ] `google_project_service` に `disable_on_destroy = false` が設定されている
- [ ] ステートフルリソースに `lifecycle { prevent_destroy = true }` が設定されている
- [ ] Deployment Manager の差異に関わる箇所に `[DM 差異]` コメントがある
- [ ] `depends_on`、`ignore_changes`、`prevent_destroy` に理由コメントがある
- [ ] module 間の依存関係がある場合、`depends_on = [module.<name>]` が明示されている
- [ ] tfstate バックエンドに Cloud Storage が設定されている（本番環境）

## 検証コマンド

```powershell
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan
```

## スキル定義

エージェント向けの詳細な実行手順、テンプレート、コード生成順序は [SKILL.md](SKILL.md) を参照してください。
