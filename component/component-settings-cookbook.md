# Decidim コンポーネント設定クックブック

実際のユースケースに基づいたコンポーネント設定の実装例集です。

## 目次

1. [コメント機能の拡張](#コメント機能の拡張)
2. [条件付き設定の実装](#条件付き設定の実装)
3. [動的選択肢の実装](#動的選択肢の実装)
4. [カスタム検証の追加](#カスタム検証の追加)
5. [既存コンポーネントのカスタマイズ](#既存コンポーネントのカスタマイズ)
6. [設定のグループ化とUI改善](#設定のグループ化とui改善)

## コメント機能の拡張

### ユースケース
コンポーネントごとにコメント機能の詳細な設定を可能にしたい。

### 実装例

#### 1. コンポーネントマニフェストに設定を追加

```ruby
# lib/decidim/proposals/component.rb
Decidim.register_component(:proposals) do |component|
  # 既存の設定...
  
  component.settings(:global) do |settings|
    # 基本的なコメント設定（既存）
    settings.attribute :comments_enabled, type: :boolean, default: true
    settings.attribute :comments_max_length, type: :integer, default: 1000, required: true
    
    # 拡張コメント設定（新規追加）
    settings.attribute :comments_voting_enabled, 
      type: :boolean, 
      default: true
    
    settings.attribute :comments_nesting_depth, 
      type: :integer, 
      default: 3,
      validates: { inclusion: { in: 1..5 } }
    
    settings.attribute :comments_edit_time_limit, 
      type: :integer, 
      default: 15,
      readonly: ->(context) { 
        # 既にコメントがある場合は変更不可
        Decidim::Comments::Comment.where(
          root_commentable: context[:component].resources
        ).any?
      }
    
    settings.attribute :comments_moderation_enabled, 
      type: :boolean, 
      default: false
    
    settings.attribute :comments_sorting_method,
      type: :enum,
      default: "best",
      choices: -> { %w(best older recent most_voted) }
  end
end
```

#### 2. ヘルパーメソッドの実装

```ruby
# app/helpers/decidim/comments/comments_helper.rb
module Decidim
  module Comments
    module CommentsHelper
      def comments_settings_for(resource)
        component = resource.component
        return default_comments_settings unless component
        
        {
          enabled: component.settings.comments_enabled,
          max_length: component.settings.comments_max_length,
          voting_enabled: component.settings.comments_voting_enabled,
          nesting_depth: component.settings.comments_nesting_depth,
          edit_time_limit: component.settings.comments_edit_time_limit,
          moderation_enabled: component.settings.comments_moderation_enabled,
          sorting_method: component.settings.comments_sorting_method
        }
      end
      
      private
      
      def default_comments_settings
        {
          enabled: true,
          max_length: 1000,
          voting_enabled: true,
          nesting_depth: 3,
          edit_time_limit: 15,
          moderation_enabled: false,
          sorting_method: "best"
        }
      end
    end
  end
end
```

#### 3. ビューでの利用

```erb
<%# app/views/decidim/comments/_comments.html.erb %>
<% settings = comments_settings_for(resource) %>

<% if settings[:enabled] %>
  <div class="comments" 
       data-max-length="<%= settings[:max_length] %>"
       data-nesting-depth="<%= settings[:nesting_depth] %>"
       data-voting-enabled="<%= settings[:voting_enabled] %>"
       data-edit-time-limit="<%= settings[:edit_time_limit] %>">
    
    <% if settings[:moderation_enabled] %>
      <div class="callout warning">
        <%= t("decidim.comments.moderation_notice") %>
      </div>
    <% end %>
    
    <%= render partial: "decidim/comments/comment_form", 
               locals: { max_length: settings[:max_length] } %>
    
    <div class="comments-list" data-sort="<%= settings[:sorting_method] %>">
      <%= render partial: "decidim/comments/comment", 
                 collection: sorted_comments(comments, settings[:sorting_method]) %>
    </div>
  </div>
<% end %>
```

#### 4. 管理画面での設定表示の改善

```erb
<%# app/views/decidim/admin/components/_comment_settings.html.erb %>
<div class="card" id="comment-settings">
  <div class="card-divider">
    <h2 class="card-title">
      <%= t("decidim.admin.components.comment_settings.title") %>
    </h2>
  </div>
  
  <div class="card-section">
    <div class="row column">
      <%= form.check_box :comments_enabled %>
    </div>
    
    <div class="comments-advanced-settings" 
         data-toggler=".hide" 
         class="<%= "hide" unless form.object.comments_enabled %>">
      
      <div class="row column">
        <%= form.number_field :comments_max_length, min: 50, max: 10000 %>
      </div>
      
      <div class="row column">
        <%= form.check_box :comments_voting_enabled %>
      </div>
      
      <div class="row column">
        <%= form.number_field :comments_nesting_depth, min: 1, max: 5 %>
        <p class="help-text">
          <%= t("decidim.admin.components.comment_settings.nesting_depth_help") %>
        </p>
      </div>
      
      <% if form.object.settings(:comments_edit_time_limit).readonly?(component: @component) %>
        <div class="row column">
          <label><%= t("decidim.admin.components.comment_settings.edit_time_limit") %></label>
          <p class="readonly-setting">
            <%= form.object.comments_edit_time_limit %> 
            <%= t("decidim.admin.components.comment_settings.minutes") %>
          </p>
          <p class="help-text callout warning">
            <%= t("decidim.admin.components.comment_settings.edit_time_limit_readonly") %>
          </p>
        </div>
      <% else %>
        <div class="row column">
          <%= form.number_field :comments_edit_time_limit, min: 0, max: 60 %>
          <p class="help-text">
            <%= t("decidim.admin.components.comment_settings.edit_time_limit_help") %>
          </p>
        </div>
      <% end %>
      
      <div class="row column">
        <%= form.check_box :comments_moderation_enabled %>
      </div>
      
      <div class="row column">
        <%= form.collection_radio_buttons :comments_sorting_method,
                                         %w(best older recent most_voted),
                                         :to_s,
                                         ->(val) { t("decidim.comments.sorting.#{val}") } %>
      </div>
    </div>
  </div>
</div>

<script>
  // comments_enabledの変更時に詳細設定の表示/非表示を切り替え
  $('#component_settings_comments_enabled').on('change', function() {
    $('.comments-advanced-settings').toggleClass('hide', !this.checked);
  });
</script>
```

## 条件付き設定の実装

### ユースケース
特定の条件下で設定を読み取り専用にしたい。

### 実装パターン

#### 1. データ存在チェック

```ruby
settings.attribute :participatory_texts_enabled,
  type: :boolean,
  default: false,
  readonly: ->(context) {
    # 既に提案が存在する場合は変更不可
    Decidim::Proposals::Proposal
      .where(component: context[:component])
      .any?
  }
```

#### 2. 時間ベースの制御

```ruby
settings.attribute :registration_enabled,
  type: :boolean,
  default: true,
  readonly: ->(context) {
    component = context[:component]
    # イベント開始後は登録設定を変更不可
    component.meetings.any? { |m| m.start_time < Time.current }
  }
```

#### 3. 権限ベースの制御

```ruby
settings.attribute :budget_limit,
  type: :integer,
  default: 100000,
  readonly: ->(context) {
    user = context[:current_user]
    component = context[:component]
    # スーパー管理者以外は予算上限を変更不可
    !user&.admin? || component.orders.any?
  }
```

## 動的選択肢の実装

### ユースケース
選択肢を動的に生成したい。

### 実装例

#### 1. 組織の設定に基づく選択肢

```ruby
settings.attribute :available_languages,
  type: :select,
  choices: ->(context) {
    organization = context[:component].organization
    organization.available_locales.map do |locale|
      [locale, I18n.with_locale(locale) { I18n.t("locale.name") }]
    end
  },
  include_blank: true
```

#### 2. 他のコンポーネントからの選択肢

```ruby
settings.attribute :linked_component,
  type: :select,
  choices: ->(context) {
    space = context[:component].participatory_space
    space.components
         .where.not(id: context[:component].id)
         .map { |c| [c.id, c.name[I18n.locale.to_s]] }
  }
```

#### 3. 外部サービスからの選択肢

```ruby
settings.attribute :map_provider,
  type: :enum,
  default: "osm",
  choices: -> {
    # 利用可能な地図プロバイダーを動的に取得
    Decidim::Map.available_providers
  }
```

## カスタム検証の追加

### ユースケース
設定値に複雑な検証を追加したい。

### 実装例

#### 1. カスタムフォームクラスの作成

```ruby
# app/forms/decidim/proposals/admin/component_form.rb
module Decidim
  module Proposals
    module Admin
      class ComponentForm < Decidim::Admin::ComponentForm
        validate :validate_voting_settings
        validate :validate_proposal_length
        
        private
        
        def validate_voting_settings
          return unless settings.vote_limit.present? && settings.minimum_votes_per_user.present?
          
          if settings.minimum_votes_per_user > settings.vote_limit
            errors.add(:minimum_votes_per_user, :invalid)
          end
        end
        
        def validate_proposal_length
          return unless settings.proposal_limit.present? && settings.proposal_length.present?
          
          if settings.proposal_length < 50
            errors.add(:proposal_length, :too_short)
          end
        end
      end
    end
  end
end
```

#### 2. マニフェストでカスタムフォームを指定

```ruby
Decidim.register_component(:proposals) do |component|
  component.component_form_class_name = "Decidim::Proposals::Admin::ComponentForm"
  # ...
end
```

## 既存コンポーネントのカスタマイズ

### ユースケース
既存のコンポーネントに新しい設定を追加したい。

### 実装例

#### 1. デコレーターパターンを使用

```ruby
# app/decorators/decidim/proposals/component_decorator.rb
Decidim::Proposals::ComponentManifest.class_eval do
  settings(:global) do |settings|
    # 既存の設定は保持される
    
    # 新しい設定を追加
    settings.attribute :require_author_verification,
      type: :boolean,
      default: false
    
    settings.attribute :auto_publish_proposals,
      type: :boolean,
      default: false,
      readonly: ->(context) {
        # 承認が必要な場合は自動公開を無効化
        context[:component].settings.proposal_answering_enabled
      }
  end
end
```

#### 2. イニシャライザーでの拡張

```ruby
# config/initializers/decidim_proposals_extensions.rb
Rails.application.config.to_prepare do
  Decidim.find_component_manifest(:proposals).tap do |manifest|
    manifest.settings(:global) do |settings|
      settings.attribute :custom_field_1, type: :string
      settings.attribute :custom_field_2, type: :boolean, default: true
    end
  end
end
```

## 設定のグループ化とUI改善

### ユースケース
関連する設定をグループ化して管理画面を整理したい。

### 実装例

#### 1. 設定の論理的グループ化

```ruby
# lib/decidim/custom_component/component.rb
component.settings(:global) do |settings|
  # 基本設定
  settings.attribute :enabled, type: :boolean, default: true
  settings.attribute :announcement, type: :text, translated: true
  
  # 参加設定グループ
  settings.attribute :participation_enabled, type: :boolean, default: true
  settings.attribute :participation_limit, type: :integer, default: 0
  settings.attribute :participation_requires_verification, type: :boolean
  
  # 通知設定グループ  
  settings.attribute :notifications_enabled, type: :boolean, default: true
  settings.attribute :notification_email_footer, type: :text, translated: true
  settings.attribute :notification_frequency, 
    type: :enum, 
    default: "daily",
    choices: %w(immediately daily weekly)
end
```

#### 2. カスタムビューでのグループ表示

```erb
<%# app/views/decidim/admin/components/_custom_settings.html.erb %>
<div class="card-section">
  <fieldset>
    <legend><%= t("decidim.admin.components.settings.basic") %></legend>
    <%= render "basic_settings", form: form %>
  </fieldset>
  
  <fieldset class="collapsible collapsed">
    <legend><%= t("decidim.admin.components.settings.participation") %></legend>
    <div class="fieldset-content">
      <%= render "participation_settings", form: form %>
    </div>
  </fieldset>
  
  <fieldset class="collapsible collapsed">
    <legend><%= t("decidim.admin.components.settings.notifications") %></legend>
    <div class="fieldset-content">
      <%= render "notification_settings", form: form %>
    </div>
  </fieldset>
</div>

<script>
  // 折りたたみ可能なフィールドセット
  $('.collapsible legend').on('click', function() {
    $(this).parent().toggleClass('collapsed');
  });
</script>
```

## ベストプラクティス

### 1. 設定の命名規則

- 明確で一貫性のある名前を使用
- boolean型は`_enabled`や`_allowed`で終わる
- 数値制限は`_limit`や`max_`/`min_`を使用

### 2. デフォルト値の設定

- 最も一般的な使用ケースに合わせる
- 破壊的でない安全な値を選択
- ゼロや無制限より適切な制限を設定

### 3. ヘルプテキストの提供

- 各設定の目的と影響を明確に説明
- 制限値の理由を説明
- 関連する設定との関係を明示

### 4. パフォーマンスの考慮

- 頻繁にアクセスされる設定はキャッシュ
- 重い処理を伴うreadonlyチェックは避ける
- 動的choices生成は軽量に保つ

### 5. 後方互換性

- 既存の設定名は変更しない
- 新しい設定にはデフォルト値を必ず設定
- 削除より非推奨化を優先