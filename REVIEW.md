# CloudFormation テンプレート レビュー

対象: マルチスタック版テンプレート一式（`01network.yaml` / `02date.yaml` / `03app.yaml` / `04cicd.yaml`）

構成は **ネットワーク層 → データ層 → アプリ層 → CI/CD 層** の4スタックに分割され、
各スタックが `Outputs` / `Export` と `Fn::ImportValue` で疎結合に連携する設計。
単一スタック版（`network.yaml`）から比べて、VPC Flow Logs・NAT冗長化・WAF・Secrets Manager・
分離サブネット・CI/CD パイプラインが追加され、全体的に本番を意識した良い構成になっている。

以下、**重大 → 要注意 → 軽微 → 良い点** の順にまとめる。

---

## 🔴 重大（このままだとデプロイ／運用が失敗する）

### 1. ECR が IMMUTABLE なのに毎回 `latest` タグで push → 2回目以降のビルドが必ず失敗
- 該当: `04cicd.yaml`
  - `EcrRepository.ImageTagMutability: IMMUTABLE`（L26）
  - `CodeBuildProject` の環境変数 `IMAGE_TAG: "latest"`（L132-133）
- IMMUTABLE は「同じタグを上書き不可」という設定。初回は成功するが、2回目のビルドで
  `latest` を再 push しようとして `ImageTagAlreadyExistsException` で失敗する。
- **修正案（いずれか）**
  - タグをコミットハッシュ等の一意値にする（推奨）。buildspec 側で
    `IMAGE_TAG=${CODEBUILD_RESOLVED_SOURCE_VERSION:0:7}` のように生成し、
    `imagedefinitions.json` にもそのタグを書き込む。
  - どうしても `latest` を使い続けるなら `ImageTagMutability: MUTABLE` にする。

### 2. CI/CD アーティファクト用 S3 バケットにバージョニングが無い
- 該当: `04cicd.yaml` `ArtifactBucket`（L33-45）
- CodePipeline のアーティファクトストアは **バージョニング有効が要件**。
  実際 `PipelineRole` / `CodeBuildRole` は `s3:GetObjectVersion` を前提にしている（L91, L155）。
- **修正案**: `BookImagesBucket` と同様に以下を追加。
  ```yaml
  VersioningConfiguration:
    Status: Enabled
  ```

---

## 🟠 要注意（環境・タイミングによって問題になる）

### 3. ソースが CodeCommit
- 該当: `04cicd.yaml` `CodeCommitRepository`（L15-19）、`CodePipeline` Source ステージ（L206-220）
- AWS は 2024年7月以降、**新規アカウントでの CodeCommit 新規利用を停止**している。
  既存で有効化済みのアカウント以外ではリポジトリ作成自体ができない。
- **修正案**: GitHub 等を CodeStar Connections（`AWS::CodeStarConnections::Connection` +
  `CodePipeline` の Source を `CodeStarSourceConnection`）に切り替える。

### 4. RDS `MultiAZ: true` が dev/prd 共通でハードコード
- 該当: `02date.yaml` `RdsInstance.MultiAZ`（L57）
- dev でも Multi-AZ が立ち上がりコストが約2倍になる。
- **修正案**: `Conditions` で `IsProd: !Equals [!Ref Environment, prd]` を定義し、
  `MultiAZ: !If [IsProd, true, false]` にする。

### 5. RDS の再現性・保護設定が不足
- 該当: `02date.yaml` `RdsInstance`（L40-57）
- `EngineVersion` 未指定 → AWS 側のデフォルトが変わると再デプロイで挙動が変わる。明示的に固定推奨。
- `BackupRetentionPeriod` 未指定（デフォルト1日）→ prd では 7〜30 日を推奨。
- `DeletionProtection` 未設定 → prd では `true` を推奨（`DeletionPolicy: Snapshot` はあるが二重で守る）。
- （任意）`MaxAllocatedStorage` を付けるとストレージ自動拡張が効く。

### 6. TargetGroup のヘルスチェックパスが `/`
- 該当: `03app.yaml` `TargetGroup.HealthCheckPath: /`（L51）
- Spring Boot アプリだとトップが 302 リダイレクトやログイン画面になり、ヘルスチェックが
  意図せず unhealthy 判定になることがある。
- **修正案**: `/actuator/health`（spring-boot-starter-actuator）等の専用エンドポイントにする。
  併せて `Matcher: { HttpCode: "200" }` を明示するとより堅牢。

### 7. `DesiredCount: 0` と AutoScaling `MinCapacity: 1` の食い違い
- 該当: `03app.yaml` `DesiredCount` デフォルト `0`（L24）／`EcsAutoScalingTarget.MinCapacity: 1`（L261）
- サービスは 0 で作られるが、ScalableTarget 登録により最小1へ引き上げられる。
  「0で起動したつもりが1になる」ため意図が読みづらい。デフォルトを `1` に揃えるか、
  0 で止めたい意図ならコメントで明示する。

### 8. ECR の KMS 暗号化と CodeBuild 権限
- 該当: `04cicd.yaml` `EcrRepository.EncryptionType: KMS`（L29-30）、`CodeBuildRole`（L66-108）
- AWS マネージドキー（`aws/ecr`）利用なら概ね問題ないが、カスタム KMS キーに変えた場合は
  CodeBuild ロールに `kms:GenerateDataKey` / `kms:Decrypt` が必要になる点に注意。

---

## 🟢 軽微 / スタイル

- **絵文字コメントの除去**: `04cicd.yaml` L291 `# 🌟 電波を検知したとき…` は本番テンプレートとしては
  通常の説明コメントに整えるのが望ましい。
- **セキュリティグループの Egress**: 各 SG は明示的な `SecurityGroupEgress` が無く、デフォルトの
  全許可（0.0.0.0/0）になっている。最小権限にこだわるなら RDS SG などは egress も絞れる。
- **リポジトリ内の旧 `network.yaml`（単一スタック版）との重複**: 新旧が混在すると、どちらが正か
  分かりにくい。分割版に移行するなら旧ファイルは `legacy/` へ退避 or 削除を検討。
- **スタック間デプロイ順序**: `04cicd` の Deploy ステージはアプリ層の cluster/service 実在が前提。
  スタックの適用順（network → data → app → cicd）を README に明記しておくと事故が減る。
- **`AllowedValues: [dev, prd]`**: `prd`（`prod` ではない）表記で統一されている点は一貫していてOK。
  buildspec 等アプリ側と表記を合わせること。

---

## ✅ 良い点（このまま活かしたい）

- サブネットを **public / private(NAT) / isolated(RDS専用)** の3層に分け、RDS を完全分離サブネットに
  配置している（`01network.yaml` PrivateSubnet3/4 + IsolatedRouteTable）。
- **SG のチェーン設計**（ALB ← ECS ← RDS）で、各層が一つ内側からしか到達できない多層防御。
- **NAT ゲートウェイの AZ 冗長化**（NatGateway1/2）で片系障害に強い。
- **VPC Flow Logs** を IAM ロール付きで有効化し、監査ログを確保。
- **Secrets Manager でDBパスワード自動生成**し、平文パラメータを排除（旧版の `DBPassword` 手入力を改善）。
- **S3 の SSL 強制ポリシー**（`aws:SecureTransport=false` を Deny）・BlockPublicAccess 全部 true・
  デフォルト暗号化・バージョニング（画像バケット）と、S3 まわりのハードニングが丁寧。
- **WAF（AWSManagedRulesCommonRuleSet）を ALB に関連付け**、L7 の基本防御を導入。
- ALB リスナーが **証明書ARNの有無で HTTP→HTTPS リダイレクトを自動切替**（`HasCertificate` 条件）する
  作りは実用的で良い。
- **EventBridge によるパイプライントリガーを IaC 化**（`PollForSourceChanges: false` + Rule）できている。

---

## 対応優先度まとめ

| 優先 | 項目 | ファイル | 状態 |
|------|------|----------|------|
| 🔴 | ECR IMMUTABLE × `latest` タグ衝突 → IMMUTABLE維持＋buildspecでコミットハッシュ一意タグ化 | 04cicd.yaml / buildspec.yml | ✅ 対応済み |
| 🔴 | ArtifactBucket のバージョニング欠如 → Enabled 追加 | 04cicd.yaml | ✅ 対応済み |
| 🟠 | CodeCommit 依存（新規アカウント不可） | 04cicd.yaml | 未対応 |
| 🟠 | RDS MultiAZ ハードコード / 保護・バージョン設定不足 | 02date.yaml | 未対応 |
| 🟠 | ヘルスチェックパス `/` | 03app.yaml | 未対応 |
| 🟠 | DesiredCount 0 と MinCapacity 1 の不整合 | 03app.yaml | 未対応 |
| 🟢 | 絵文字コメント・旧ファイル重複・egress 等 | 全般 | 未対応 |
