# Decidim コンポーネント開発チェックリスト

コンポーネントの開発・拡張時に確認すべき項目の包括的なチェックリストです。

## 目次

1. [新規コンポーネント作成チェックリスト](#新規コンポーネント作成チェックリスト)
2. [既存コンポーネント拡張チェックリスト](#既存コンポーネント拡張チェックリスト)
3. [設定追加チェックリスト](#設定追加チェックリスト)
4. [パフォーマンスチェックリスト](#パフォーマンスチェックリスト)
5. [セキュリティチェックリスト](#セキュリティチェックリスト)
6. [テストチェックリスト](#テストチェックリスト)
7. [デプロイメントチェックリスト](#デプロイメントチェックリスト)
8. [トラブルシューティングガイド](#トラブルシューティングガイド)

## 新規コンポーネント作成チェックリスト

### 基本構造

- [ ] **Gem構造の作成**
  ```bash
  bundle gem decidim-your_component
  cd decidim-your_component
  ```

- [ ] **必須ディレクトリの作成**
  ```
  ├── app/
  │   ├── controllers/
  │   ├── models/
  │   ├── views/
  │   ├── commands/
  │   ├── forms/
  │   └── permissions/
  ├── config/
  │   ├── locales/
  │   └── routes.rb
  ├── db/migrate/
  └── lib/decidim/your_component/
  ```

- [ ] **gemspecファイルの設定**
  ```ruby
  # decidim-your_component.gemspec
  spec.add_dependency "decidim-core", Decidim.version
  ```

### コンポーネント定義

- [ ] **component.rbの作成**
  ```ruby
  # lib/decidim/your_component/component.rb
  Decidim.register_component(:your_component) do |component|
    component.engine = Decidim::YourComponent::Engine
    component.admin_engine = Decidim::YourComponent::AdminEngine
    component.icon = "media/images/icon.svg"
    component.icon_key = "your-component"
    
    # 基本設定
    component.settings(:global) do |settings|
      settings.attribute :enabled, type: :boolean, default: true
    end
  end
  ```

- [ ] **エンジンファイルの作成**
  - [ ] `engine.rb` - メインエンジン
  - [ ] `admin_engine.rb` - 管理画面エンジン

- [ ] **ルーティングの設定**
  ```ruby
  # config/routes.rb
  Decidim::YourComponent::Engine.routes.draw do
    root to: "your_resources#index"
  end
  ```

### 基本機能

- [ ] **モデルの作成**
  - [ ] ActiveRecordモデル
  - [ ] マイグレーション
  - [ ] バリデーション
  - [ ] 関連定義

- [ ] **コントローラーの作成**
  - [ ] ApplicationController継承
  - [ ] 適切な権限チェック
  - [ ] エラーハンドリング

- [ ] **ビューの作成**
  - [ ] レイアウトの適用
  - [ ] レスポンシブデザイン
  - [ ] 多言語対応

### 管理機能

- [ ] **管理画面コントローラー**
- [ ] **管理画面ビュー**
- [ ] **権限クラスの実装**
- [ ] **CSVエクスポート機能**

### 国際化

- [ ] **翻訳ファイルの作成**
  - [ ] `en.yml` - 英語
  - [ ] `ja.yml` - 日本語
  - [ ] その他必要な言語

- [ ] **翻訳キーの一貫性確認**

## 既存コンポーネント拡張チェックリスト

### 準備

- [ ] **拡張方法の決定**
  - [ ] デコレーターパターン
  - [ ] モンキーパッチ
  - [ ] イニシャライザー

- [ ] **影響範囲の確認**
  - [ ] 既存機能への影響
  - [ ] 他のコンポーネントとの依存関係
  - [ ] アップグレード時の互換性

### 実装

- [ ] **拡張コードの配置**
  ```ruby
  # app/decorators/decidim/component_name/model_decorator.rb
  # config/initializers/decidim_extensions.rb
  ```

- [ ] **既存設定の保持**
  ```ruby
  # 既存の設定を上書きしない
  Rails.application.config.to_prepare do
    manifest = Decidim.find_component_manifest(:existing_component)
    # 設定の追加のみ行う
  end
  ```

- [ ] **フックの利用**
  ```ruby
  component.on(:create) do |component|
    # カスタム処理
  end
  ```

### テスト

- [ ] **既存テストの確認**
- [ ] **拡張機能のテスト追加**
- [ ] **統合テストの実行**

## 設定追加チェックリスト

### 設計

- [ ] **設定の必要性確認**
  - [ ] ユースケースの明確化
  - [ ] デフォルト値の妥当性
  - [ ] 既存設定との重複確認

- [ ] **設定タイプの選択**
  - [ ] 適切な型（boolean, integer, text等）
  - [ ] グローバル vs ステップ設定
  - [ ] 必須 vs オプション

### 実装

- [ ] **マニフェストへの追加**
  ```ruby
  settings.attribute :new_setting,
    type: :boolean,
    default: false,
    # 適切なオプション
  ```

- [ ] **翻訳の追加**
  ```yaml
  # config/locales/en.yml
  en:
    decidim:
      components:
        your_component:
          settings:
            global:
              new_setting: "New Setting Label"
              new_setting_help: "Help text explaining the setting"
  ```

- [ ] **管理画面UIの確認**
  - [ ] フィールドが正しく表示される
  - [ ] ヘルプテキストが表示される
  - [ ] バリデーションエラーが適切

### 利用

- [ ] **設定値へのアクセス方法の文書化**
- [ ] **デフォルト値のフォールバック**
- [ ] **キャッシュ戦略の検討**

## パフォーマンスチェックリスト

### データベース

- [ ] **N+1問題の確認**
  ```ruby
  # includesを使用
  components = space.components.includes(:settings)
  ```

- [ ] **インデックスの追加**
  ```ruby
  add_index :table_name, :column_name
  ```

- [ ] **不要なクエリの削減**

### キャッシング

- [ ] **設定値のキャッシュ**
  ```ruby
  def cached_settings
    Rails.cache.fetch("component_#{id}_settings", expires_in: 1.hour) do
      settings
    end
  end
  ```

- [ ] **ビューのフラグメントキャッシュ**
- [ ] **APIレスポンスのキャッシュ**

### アセット最適化

- [ ] **JavaScript/CSSの最小化**
- [ ] **画像の最適化**
- [ ] **遅延読み込みの実装**

## セキュリティチェックリスト

### 入力検証

- [ ] **すべての入力値の検証**
  ```ruby
  validates :setting_value, presence: true, inclusion: { in: ALLOWED_VALUES }
  ```

- [ ] **SQLインジェクション対策**
  ```ruby
  # 危険
  where("name = '#{params[:name]}'")
  
  # 安全
  where(name: params[:name])
  ```

- [ ] **XSS対策**
  - [ ] ユーザー入力のエスケープ
  - [ ] `html_safe`の適切な使用

### 権限管理

- [ ] **すべてのアクションで権限チェック**
  ```ruby
  before_action :authenticate_user!
  before_action :authorize_action!
  ```

- [ ] **管理者権限の適切な分離**
- [ ] **リソースレベルの権限設定**

### データ保護

- [ ] **機密情報の暗号化**
- [ ] **個人情報の適切な取り扱い**
- [ ] **ログへの機密情報出力防止**

## テストチェックリスト

### 単体テスト

- [ ] **モデルテスト**
  - [ ] バリデーション
  - [ ] メソッド
  - [ ] スコープ

- [ ] **コマンドテスト**
  - [ ] 成功ケース
  - [ ] 失敗ケース
  - [ ] エッジケース

### 統合テスト

- [ ] **コントローラーテスト**
- [ ] **システムテスト**
  ```ruby
  # spec/system/component_spec.rb
  it "allows users to interact with component" do
    visit component_path
    expect(page).to have_content("Expected content")
  end
  ```

### 設定テスト

- [ ] **設定の保存と読み込み**
- [ ] **デフォルト値の適用**
- [ ] **バリデーションの動作**

## デプロイメントチェックリスト

### 準備

- [ ] **データベースマイグレーション**
  ```bash
  rails db:migrate
  ```

- [ ] **アセットのプリコンパイル**
  ```bash
  rails assets:precompile
  ```

- [ ] **環境変数の設定**

### デプロイ

- [ ] **gemのインストール**
  ```ruby
  # Gemfile
  gem "decidim-your_component", path: "decidim-your_component"
  ```

- [ ] **初期化コードの確認**
- [ ] **キャッシュのクリア**

### 確認

- [ ] **コンポーネントの登録確認**
  ```ruby
  Decidim.component_manifests.map(&:name)
  ```

- [ ] **管理画面での表示確認**
- [ ] **エラーログの確認**

## トラブルシューティングガイド

### よくある問題と解決方法

#### 1. コンポーネントが表示されない

**症状**: 管理画面でコンポーネントが選択できない

**確認事項**:
- [ ] `Decidim.register_component`が実行されているか
- [ ] エンジンが正しく読み込まれているか
- [ ] gemがGemfileに追加されているか

**解決方法**:
```ruby
# コンソールで確認
Decidim.component_manifests.map(&:name)
```

#### 2. 設定が保存されない

**症状**: 管理画面で設定を変更しても反映されない

**確認事項**:
- [ ] フォームのstrong parametersが正しいか
- [ ] 設定の型が正しいか
- [ ] readonly設定になっていないか

**解決方法**:
```ruby
# ログで確認
Rails.logger.debug component_params
```

#### 3. 翻訳が表示されない

**症状**: 翻訳キーがそのまま表示される

**確認事項**:
- [ ] 翻訳ファイルのパスが正しいか
- [ ] YAMLの構文エラーがないか
- [ ] 翻訳キーの階層が正しいか

**解決方法**:
```ruby
# コンソールで確認
I18n.t("decidim.components.your_component.name")
```

#### 4. マイグレーションエラー

**症状**: マイグレーション実行時にエラー

**確認事項**:
- [ ] マイグレーションファイルの命名規則
- [ ] 依存関係の順序
- [ ] 既存データとの整合性

**解決方法**:
```bash
# ロールバック後に再実行
rails db:rollback STEP=1
rails db:migrate
```

#### 5. 権限エラー

**症状**: アクションが実行できない

**確認事項**:
- [ ] Permissionsクラスの実装
- [ ] アクションの登録
- [ ] ユーザーの権限設定

**解決方法**:
```ruby
# 権限の確認
permission = Decidim::Permission.new(user, :create, :proposal, component: component)
permissions_class.new(permission).permissions
```

### デバッグのヒント

1. **ログレベルを上げる**
   ```ruby
   Rails.logger.level = :debug
   ```

2. **Railsコンソールでの確認**
   ```ruby
   component = Decidim::Component.find(id)
   component.settings
   component.manifest
   ```

3. **ブラウザの開発者ツール**
   - ネットワークタブでリクエスト確認
   - コンソールでJavaScriptエラー確認

4. **テスト環境での再現**
   ```bash
   RAILS_ENV=test rails console
   ```

### サポートリソース

- Decidim公式ドキュメント
- Decidimコミュニティフォーラム
- GitHub Issues
- Gitterチャット