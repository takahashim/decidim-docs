# 既存コンポーネントへの設定追加ガイド

既存のDecidimコンポーネントに新しい設定を追加し、管理画面で編集可能にする方法を解説します。

## 目次

1. [概要](#概要)
2. [基本的な実装手順](#基本的な実装手順)
3. [実装例：提案コンポーネントの拡張](#実装例提案コンポーネントの拡張)
4. [高度なカスタマイズ](#高度なカスタマイズ)
5. [注意事項とベストプラクティス](#注意事項とベストプラクティス)

## 概要

Decidimでは、既存コンポーネントを直接修正することなく、新しい設定を追加できます。これにより、Decidimのアップグレード時にカスタマイズを維持しやすくなります。

### 主な手法

1. **イニシャライザーを使用した拡張**（推奨）
2. **カスタムエンジンからの拡張**
3. **モジュールプリペンドによる拡張**

## 基本的な実装手順

### ステップ1: イニシャライザーの作成

```ruby
# config/initializers/decidim_component_extensions.rb
Rails.application.config.to_prepare do
  # 拡張したいコンポーネントのマニフェストを取得
  manifest = Decidim.find_component_manifest(:proposals)
  
  # 新しい設定を追加
  manifest.settings(:global) do |settings|
    settings.attribute :custom_setting,
      type: :boolean,
      default: false
  end
end
```

### ステップ2: 翻訳の追加

```yaml
# config/locales/decidim_extensions.ja.yml
ja:
  decidim:
    components:
      proposals:
        settings:
          global:
            custom_setting: "カスタム設定"
            custom_setting_help: "この設定を有効にすると..."
```

### ステップ3: 設定の利用

```ruby
# アプリケーション内で設定を使用
if current_component.settings.custom_setting
  # カスタム設定が有効な場合の処理
end
```

## 実装例：提案コンポーネントの拡張

### 完全な実装例

```ruby
# config/initializers/decidim_proposals_advanced_settings.rb
Rails.application.config.to_prepare do
  manifest = Decidim.find_component_manifest(:proposals)
  
  # グローバル設定の追加
  manifest.settings(:global) do |settings|
    # 自動承認機能
    settings.attribute :auto_approval_enabled,
      type: :boolean,
      default: false
    
    settings.attribute :auto_approval_threshold,
      type: :integer,
      default: 10,
      required: true
    
    # 通知設定
    settings.attribute :notify_authors_on_vote,
      type: :boolean,
      default: true
    
    settings.attribute :notification_email_template,
      type: :text,
      translated: true,
      editor: true
    
    # AI統合設定
    settings.attribute :ai_moderation_enabled,
      type: :boolean,
      default: false,
      readonly: ->(context) {
        # AI APIキーが設定されていない場合は読み取り専用
        ENV['AI_API_KEY'].blank?
      }
    
    settings.attribute :ai_confidence_threshold,
      type: :float,
      default: 0.8
  end
  
  # ステップ設定の追加
  manifest.settings(:step) do |settings|
    settings.attribute :require_author_verification,
      type: :boolean,
      default: false
  end
  
  # 新しい統計情報の追加
  manifest.register_stat :verified_proposals do |components, start_at, end_at|
    Decidim::Proposals::Proposal
      .where(component: components)
      .where(author_verified: true)
      .count
  end
end
```

### 対応する翻訳ファイル

```yaml
# config/locales/decidim_proposals_advanced.ja.yml
ja:
  decidim:
    components:
      proposals:
        settings:
          global:
            auto_approval_enabled: "自動承認を有効化"
            auto_approval_enabled_help: "指定した条件を満たした提案を自動的に承認します"
            auto_approval_threshold: "自動承認閾値"
            auto_approval_threshold_help: "この数以上の支持を得た提案が自動承認されます"
            notify_authors_on_vote: "投票時に作成者に通知"
            notify_authors_on_vote_help: "提案に投票があった際に作成者にメール通知を送信します"
            notification_email_template: "通知メールテンプレート"
            notification_email_template_help: "投票通知メールのカスタムテンプレート"
            ai_moderation_enabled: "AIモデレーションを有効化"
            ai_moderation_enabled_help: "AIを使用して不適切な内容を自動検出します（APIキーが必要）"
            ai_confidence_threshold: "AI信頼度閾値"
            ai_confidence_threshold_help: "この値以上の信頼度でAIが不適切と判断した内容をフラグします"
          step:
            require_author_verification: "作成者の本人確認を必須にする"
            require_author_verification_help: "有効にすると、本人確認済みのユーザーのみが提案を作成できます"

# config/locales/decidim_proposals_advanced.en.yml
en:
  decidim:
    components:
      proposals:
        settings:
          global:
            auto_approval_enabled: "Enable auto-approval"
            auto_approval_enabled_help: "Automatically approve proposals that meet specified criteria"
            # ... 他の英語翻訳
```

## 高度なカスタマイズ

### カスタムバリデーションの追加

```ruby
# app/forms/decidim/proposals/admin/component_form.rb
module Decidim
  module Proposals
    module Admin
      class ComponentForm < Decidim::Admin::ComponentForm
        validate :validate_auto_approval_settings
        
        private
        
        def validate_auto_approval_settings
          return unless settings.auto_approval_enabled
          
          if settings.auto_approval_threshold < 1
            errors.add(:auto_approval_threshold, "must be at least 1")
          end
          
          if settings.auto_approval_threshold > 1000
            errors.add(:auto_approval_threshold, "is too high (maximum 1000)")
          end
        end
      end
    end
  end
end

# イニシャライザーでフォームクラスを指定
Rails.application.config.to_prepare do
  manifest = Decidim.find_component_manifest(:proposals)
  manifest.component_form_class_name = "Decidim::Proposals::Admin::ComponentForm"
end
```

### 設定に基づく動的な振る舞い

```ruby
# app/commands/decidim/proposals/create_proposal.rb のオーバーライド
module ProposalExtensions
  def call
    super.tap do
      if form.current_component.settings.notify_authors_on_vote
        # 通知設定を有効化
        enable_vote_notifications
      end
      
      if should_auto_approve?
        # 自動承認ロジック
        auto_approve_proposal
      end
    end
  end
  
  private
  
  def should_auto_approve?
    component = form.current_component
    component.settings.auto_approval_enabled &&
      @proposal.endorsements_count >= component.settings.auto_approval_threshold
  end
end

# 適用
Rails.application.config.to_prepare do
  Decidim::Proposals::CreateProposal.prepend(ProposalExtensions)
end
```

### カスタムビューの追加

```erb
<%# app/views/decidim/admin/components/_proposals_advanced_settings.html.erb %>
<% if Decidim.find_component_manifest(:proposals) %>
  <div class="card" id="proposals-advanced-settings">
    <div class="card-divider">
      <h2 class="card-title">
        <%= t("decidim.admin.components.proposals.advanced_settings.title") %>
      </h2>
    </div>
    
    <div class="card-section">
      <!-- 自動承認設定 -->
      <div class="row column">
        <%= form.check_box :settings_auto_approval_enabled %>
      </div>
      
      <div id="auto-approval-options" 
           style="<%= 'display:none' unless form.object.settings.auto_approval_enabled %>">
        <div class="row column">
          <%= form.number_field :settings_auto_approval_threshold, 
                               min: 1, 
                               max: 1000 %>
        </div>
      </div>
      
      <!-- AI設定 -->
      <% unless form.object.settings(:ai_moderation_enabled).readonly?(component: @component) %>
        <div class="row column">
          <%= form.check_box :settings_ai_moderation_enabled %>
        </div>
        
        <div id="ai-options" 
             style="<%= 'display:none' unless form.object.settings.ai_moderation_enabled %>">
          <div class="row column">
            <%= form.number_field :settings_ai_confidence_threshold, 
                                 step: 0.1, 
                                 min: 0.0, 
                                 max: 1.0 %>
          </div>
        </div>
      <% else %>
        <div class="callout warning">
          <%= t("decidim.admin.components.proposals.ai_settings_unavailable") %>
        </div>
      <% end %>
    </div>
  </div>
  
  <script>
    // 動的な表示/非表示
    $('#component_settings_auto_approval_enabled').on('change', function() {
      $('#auto-approval-options').toggle(this.checked);
    });
    
    $('#component_settings_ai_moderation_enabled').on('change', function() {
      $('#ai-options').toggle(this.checked);
    });
  </script>
<% end %>
```

## 注意事項とベストプラクティス

### 1. Rails.application.config.to_prepareの使用

```ruby
# 正しい
Rails.application.config.to_prepare do
  # コンポーネント拡張
end

# 避けるべき（開発環境でリロードされない）
manifest = Decidim.find_component_manifest(:proposals)
manifest.settings(:global) do |settings|
  # ...
end
```

### 2. 既存設定との競合を避ける

```ruby
# 設定追加前に既存設定を確認
Rails.application.config.to_prepare do
  manifest = Decidim.find_component_manifest(:proposals)
  
  # 既存の設定名を確認
  existing_settings = manifest.settings(:global).attributes.keys
  
  # 新しい設定名が競合しないことを確認
  unless existing_settings.include?(:my_custom_setting)
    manifest.settings(:global) do |settings|
      settings.attribute :my_custom_setting, type: :boolean
    end
  end
end
```

### 3. アップグレード対応

```ruby
# バージョンチェックを含める
Rails.application.config.to_prepare do
  if Decidim.version >= "0.28.0"
    # 新しいバージョン用の設定
  else
    # 古いバージョン用の設定
  end
end
```

### 4. パフォーマンスの考慮

```ruby
# 重い処理は遅延評価
settings.attribute :expensive_options,
  type: :select,
  choices: -> {
    Rails.cache.fetch("expensive_options", expires_in: 1.hour) do
      # 重い処理
    end
  }
```

### 5. テストの作成

```ruby
# spec/lib/decidim_extensions_spec.rb
require "rails_helper"

RSpec.describe "Decidim Extensions" do
  it "adds custom settings to proposals component" do
    manifest = Decidim.find_component_manifest(:proposals)
    settings = manifest.settings(:global).attributes
    
    expect(settings).to have_key(:auto_approval_enabled)
    expect(settings[:auto_approval_enabled].type).to eq(:boolean)
  end
end
```

## まとめ

Decidimの既存コンポーネントへの設定追加は、`Rails.application.config.to_prepare`ブロック内で`Decidim.find_component_manifest`を使用することで安全に実装できます。この方法により：

- Decidimのアップグレード時にカスタマイズが保持される
- 開発環境でのコードリロードが適切に動作する
- 他のカスタマイズとの競合を最小限に抑えられる

設定を追加する際は、必ず翻訳ファイルも用意し、必要に応じてカスタムバリデーションやビューも実装しましょう。