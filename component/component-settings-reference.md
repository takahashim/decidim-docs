# Decidim コンポーネント設定リファレンス

Decidimの全コンポーネントの設定項目と仕様の詳細なリファレンスです。

## 目次

1. [設定型の詳細仕様](#設定型の詳細仕様)
2. [コンポーネント別設定一覧](#コンポーネント別設定一覧)
3. [共通設定パターン](#共通設定パターン)
4. [バリデーションルール](#バリデーションルール)
5. [セキュリティ考慮事項](#セキュリティ考慮事項)

## 設定型の詳細仕様

### 基本型

#### `:boolean`
- **説明**: 真偽値（true/false）
- **デフォルト**: `false`
- **フォーム**: チェックボックス
- **使用例**: 機能の有効/無効切り替え

```ruby
settings.attribute :feature_enabled, type: :boolean, default: true
```

#### `:integer`
- **説明**: 整数値
- **デフォルト**: `0`
- **フォーム**: 数値入力フィールド
- **検証**: 数値であること、範囲指定可能
- **使用例**: 制限値、カウント

```ruby
settings.attribute :max_items, type: :integer, default: 10, required: true
```

#### `:float`
- **説明**: 浮動小数点数
- **デフォルト**: `nil`
- **フォーム**: 数値入力フィールド
- **使用例**: パーセンテージ、比率

```ruby
settings.attribute :threshold_percentage, type: :float, default: 75.0
```

#### `:string`
- **説明**: 短い文字列
- **デフォルト**: `nil`
- **フォーム**: テキスト入力フィールド
- **使用例**: 識別子、短いラベル

```ruby
settings.attribute :identifier, type: :string, required: true
```

#### `:text`
- **説明**: 長文テキスト
- **デフォルト**: `nil`
- **フォーム**: テキストエリア（エディタオプション有）
- **オプション**: `translated`, `editor`
- **使用例**: お知らせ、説明文

```ruby
settings.attribute :description, type: :text, translated: true, editor: true
```

### 選択型

#### `:enum`
- **説明**: 事前定義された選択肢から1つ選択
- **デフォルト**: `nil`
- **フォーム**: ラジオボタン
- **必須オプション**: `choices`

```ruby
settings.attribute :layout,
  type: :enum,
  default: "grid",
  choices: %w(grid list tiles)
```

#### `:select`
- **説明**: ドロップダウンから選択
- **デフォルト**: `nil`
- **フォーム**: セレクトボックス
- **オプション**: `include_blank`

```ruby
settings.attribute :category,
  type: :select,
  choices: -> { Category.pluck(:name, :id) },
  include_blank: true
```

### 特殊型

#### `:array`
- **説明**: 複数の値の配列
- **デフォルト**: `[]`
- **フォーム**: カスタム実装が必要
- **使用例**: タグ、複数選択

```ruby
settings.attribute :allowed_tags, type: :array, default: %w(important urgent)
```

#### `:scope`
- **説明**: 組織のスコープID
- **デフォルト**: `nil`
- **フォーム**: スコープ選択ドロップダウン
- **使用例**: 地域や部門の指定

```ruby
settings.attribute :scope_id, type: :scope
```

#### `:time`
- **説明**: タイムゾーン付き日時
- **デフォルト**: `nil`
- **フォーム**: 日時選択フィールド
- **使用例**: 開始/終了時刻

```ruby
settings.attribute :starts_at, type: :time
```

## コンポーネント別設定一覧

### Proposals（提案）

#### グローバル設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| scopes_enabled | boolean | false | スコープ機能の有効化 |
| scope_id | scope | nil | デフォルトスコープ |
| vote_limit | integer | 0 | ユーザーあたりの投票数上限（0=無制限） |
| minimum_votes_per_user | integer | 0 | 最小投票数 |
| proposal_limit | integer | 0 | ユーザーあたりの提案数上限 |
| proposal_length | integer | 500 | 提案本文の最大文字数 |
| proposal_edit_time | enum | "limited" | 編集可能時間（limited/infinite） |
| proposal_edit_before_minutes | integer | 5 | 編集可能時間（分） |
| threshold_per_proposal | integer | 0 | 提案あたりの承認閾値 |
| can_accumulate_votes_beyond_threshold | boolean | false | 閾値超過後も投票可能 |
| proposal_answering_enabled | boolean | true | 管理者による回答機能 |
| official_proposals_enabled | boolean | true | 公式提案の有効化 |
| comments_enabled | boolean | true | コメント機能 |
| comments_max_length | integer | 1000 | コメント最大文字数 |
| geocoding_enabled | boolean | false | 位置情報機能 |
| attachments_allowed | boolean | true | 添付ファイル許可 |
| resources_permissions_enabled | boolean | true | リソース権限管理 |
| collaborative_drafts_enabled | boolean | false | 協働草案機能 |
| participatory_texts_enabled | boolean | false | 参加型テキスト機能 |
| amendments_enabled | boolean | false | 修正案機能 |

#### ステップ設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| votes_enabled | boolean | true | 投票の有効化 |
| votes_blocked | boolean | false | 投票のブロック |
| votes_hidden | boolean | false | 投票数の非表示 |
| comments_blocked | boolean | false | コメントのブロック |
| creation_enabled | boolean | true | 提案作成の有効化 |
| proposal_answering_enabled | boolean | true | 回答機能の有効化 |
| publish_answers_immediately | boolean | true | 回答の即時公開 |
| answers_with_costs | boolean | false | コスト付き回答 |
| amendments_enabled | boolean | true | 修正案の有効化 |
| amendments_visibility | enum | "all" | 修正案の表示範囲 |
| announcement | text | nil | ステップ別お知らせ |

### Meetings（ミーティング）

#### グローバル設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| scopes_enabled | boolean | false | スコープ機能 |
| scope_id | scope | nil | デフォルトスコープ |
| announcement | text | nil | お知らせ |
| default_registration_terms | text | nil | デフォルト参加規約 |
| comments_enabled | boolean | true | コメント機能 |
| comments_max_length | integer | 1000 | コメント最大文字数 |
| registration_code_enabled | boolean | true | 登録コード機能 |
| resources_permissions_enabled | boolean | true | リソース権限管理 |
| enable_pads_creation | boolean | false | パッド作成機能 |
| creation_enabled_for_participants | boolean | false | 参加者による作成許可 |

### Budgets（予算）

#### グローバル設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| scopes_enabled | boolean | false | スコープ機能 |
| scope_id | scope | nil | デフォルトスコープ |
| workflow | enum | "one" | ワークフロータイプ |
| projects_per_page | integer | 12 | ページあたりのプロジェクト数 |
| vote_rule_threshold_percent_enabled | boolean | true | 閾値パーセント有効化 |
| vote_threshold_percent | integer | 70 | 投票閾値（%） |
| vote_rule_minimum_budget_projects_enabled | boolean | false | 最小プロジェクト数ルール |
| vote_minimum_budget_projects_number | integer | 1 | 最小プロジェクト数 |
| vote_rule_selected_projects_enabled | boolean | false | 選択プロジェクトルール |
| vote_selected_projects_minimum | integer | 0 | 最小選択数 |
| vote_selected_projects_maximum | integer | 0 | 最大選択数 |
| comments_enabled | boolean | true | コメント機能 |
| comments_max_length | integer | 1000 | コメント最大文字数 |
| geocoding_enabled | boolean | false | 位置情報機能 |

### Debates（討論）

#### グローバル設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| scopes_enabled | boolean | false | スコープ機能 |
| scope_id | scope | nil | デフォルトスコープ |
| comments_enabled | boolean | true | コメント機能 |
| comments_max_length | integer | 1000 | コメント最大文字数 |
| announcement | text | nil | お知らせ |

#### ステップ設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| endorsements_enabled | boolean | true | 支持機能 |
| endorsements_blocked | boolean | false | 支持のブロック |
| announcement | text | nil | ステップ別お知らせ |

### Surveys（アンケート）

#### グローバル設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| scopes_enabled | boolean | false | スコープ機能 |
| scope_id | scope | nil | デフォルトスコープ |
| starts_at | time | nil | 開始日時 |
| ends_at | time | nil | 終了日時 |
| announcement | text | nil | お知らせ |
| clean_after_publish | boolean | true | 公開後のクリーンアップ |

### Blogs（ブログ）

#### グローバル設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| announcement | text | nil | お知らせ |
| comments_enabled | boolean | true | コメント機能 |
| comments_max_length | integer | 1000 | コメント最大文字数 |

### Pages（ページ）

#### グローバル設定

| 設定名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| announcement | text | nil | お知らせ |

## 共通設定パターン

### 基本機能設定

```ruby
# ほぼすべてのコンポーネントで使用される基本設定
settings.attribute :announcement, type: :text, translated: true, editor: true
settings.attribute :comments_enabled, type: :boolean, default: true
settings.attribute :comments_max_length, type: :integer, default: 1000, required: true
```

### スコープ設定

```ruby
# 地理的・組織的な範囲を指定する設定
settings.attribute :scopes_enabled, type: :boolean, default: false
settings.attribute :scope_id, type: :scope
```

### 権限とセキュリティ設定

```ruby
# アクセス制御に関する設定
settings.attribute :resources_permissions_enabled, type: :boolean, default: true
settings.attribute :creation_enabled_for_participants, type: :boolean, default: false
```

### 制限値設定

```ruby
# 数量制限に関する設定
settings.attribute :vote_limit, type: :integer, default: 0
settings.attribute :proposal_limit, type: :integer, default: 0
settings.attribute :max_length, type: :integer, required: true
```

## バリデーションルール

### 必須フィールド

```ruby
settings.attribute :required_field, type: :string, required: true
```

### 数値範囲

```ruby
settings.attribute :percentage, 
  type: :integer, 
  default: 50,
  validates: { 
    numericality: { 
      greater_than_or_equal_to: 0, 
      less_than_or_equal_to: 100 
    } 
  }
```

### 文字列長

```ruby
settings.attribute :short_text,
  type: :string,
  validates: { length: { maximum: 100 } }
```

### カスタムバリデーション

```ruby
# ComponentFormクラスで実装
validate :custom_validation

private

def custom_validation
  if settings.end_date < settings.start_date
    errors.add(:end_date, :invalid)
  end
end
```

## セキュリティ考慮事項

### 1. 入力値のサニタイズ

すべての設定値は自動的にサニタイズされますが、特に注意が必要な項目：

- HTMLコンテンツ（`editor: true`の場合）
- URLやパス
- 実行可能なコード片

### 2. 権限チェック

設定変更時の権限確認：

```ruby
# UpdateComponentコマンド内で自動的に実行
authorize! :update, component
```

### 3. 読み取り専用設定の保護

```ruby
# 既存データがある場合の設定変更を防ぐ
settings.attribute :critical_setting,
  type: :boolean,
  readonly: ->(context) {
    Model.where(component: context[:component]).any?
  }
```

### 4. デフォルト値の安全性

- 破壊的でない値を選択
- 最小権限の原則に従う
- 明示的な有効化を要求

### 5. 機密情報の取り扱い

- パスワードやトークンを設定に保存しない
- 環境変数や専用の暗号化ストレージを使用
- ログに機密情報を出力しない

## パフォーマンス最適化

### 1. 設定値のキャッシング

```ruby
def cached_settings
  @cached_settings ||= component.settings
end
```

### 2. 重い処理の回避

```ruby
# 避けるべき例
settings.attribute :data,
  choices: -> { ExpensiveQuery.all.map(&:name) }

# 推奨例
settings.attribute :data_id,
  type: :select,
  choices: -> { Rails.cache.fetch("data_choices", expires_in: 1.hour) { 
    ExpensiveQuery.pluck(:name, :id) 
  }}
```

### 3. N+1問題の防止

```ruby
# コンポーネントと設定を一括読み込み
components = space.components.includes(:settings)
```