# モバイル版ロゴ機能追加の実装仕様

## 概要
Decidim::Organizationにモバイル版専用のロゴ画像を追加し、現在faviconが使用されている箇所で新しいモバイルロゴを表示できるようにする。

## 1. 現状分析

### 既存の画像管理機能
- **Organizationモデル**: `logo`、`favicon`、`official_img_footer`など複数の画像をActive Storageで管理
- **モバイル版表示**: `_logo_mobile.html.erb`でfaviconの`:medium`バリアントを使用
- **管理画面**: OrganizationAppearanceControllerとFormで画像アップロード機能を提供

### 技術スタック
- Active Storage for file uploads
- ImageUploader基底クラスによる画像処理
- Decidim::Admin::UpdateOrganizationAppearanceコマンドパターン

## 2. 実装方針

### A. データベース拡張
Active Storageを使用するため、マイグレーションでのカラム追加は不要。

```ruby
# マイグレーション: db/migrate/[timestamp]_add_mobile_logo_to_decidim_organizations.rb
class AddMobileLogoToDecidimOrganizations < ActiveRecord::Migration[6.1]
  def change
    # Active Storageを使用するため、カラム追加は不要
    # このマイグレーションは実行記録のためのみ
  end
end
```

### B. モデル拡張
Organizationモデルにmobile_logoアタッチメントを追加。

```ruby
# decidim-cfj/app/decorators/models/decidim/organization_decorator.rb
module Decidim
  Organization.class_eval do
    has_one_attached :mobile_logo
    validates_upload :mobile_logo, uploader: Decidim::OrganizationMobileLogoUploader
  end
end
```

### C. アップローダー作成
モバイルロゴ専用のアップローダーを定義。適切なサイズバリアントを設定。

```ruby
# decidim-cfj/app/uploaders/decidim/organization_mobile_logo_uploader.rb
module Decidim
  class OrganizationMobileLogoUploader < ImageUploader
    set_variants do
      {
        small: { resize_to_fit: [180, 60] },
        medium: { resize_to_fit: [360, 120] }
      }
    end

    def dimensions_info
      {
        small: { processor: :resize_to_fit, dimensions: [180, 60] },
        medium: { processor: :resize_to_fit, dimensions: [360, 120] }
      }
    end
  end
end
```

### D. 管理画面フォーム拡張
OrganizationAppearanceFormにモバイルロゴフィールドを追加。

```ruby
# decidim-cfj/app/decorators/forms/decidim/admin/organization_appearance_form_decorator.rb
Decidim::Admin::OrganizationAppearanceForm.class_eval do
  attribute :mobile_logo
  attribute :remove_mobile_logo, Boolean, default: false

  validates :mobile_logo, passthru: { to: Decidim::Organization }
end
```

### E. コマンド拡張
UpdateOrganizationAppearanceコマンドでmobile_logoを処理できるように拡張。

```ruby
# decidim-cfj/app/decorators/commands/decidim/admin/update_organization_appearance_decorator.rb
Decidim::Admin::UpdateOrganizationAppearance.class_eval do
  # mobile_logoをファイル属性として追加
  def self.fetch_file_attributes(*attrs)
    @file_attributes ||= []
    @file_attributes += attrs
    @file_attributes << :mobile_logo unless @file_attributes.include?(:mobile_logo)
    @file_attributes
  end
end
```

### F. ビュー修正
モバイルロゴの表示ロジックを実装。優先順位: mobile_logo > favicon > テキスト。

```erb
# decidim-cfj/app/views/layouts/decidim/_logo_mobile.html.erb（override）
<% if organization %>
  <%= link_to decidim.root_url(host: organization.host), "aria-label": t("front_page_link", scope: "decidim.accessibility") do %>
    <% if organization.mobile_logo.attached? %>
      <%= image_tag organization.attached_uploader(:mobile_logo).variant_path(:medium),
          alt: t("logo", scope: "decidim.accessibility", organization: current_organization_name) %>
    <% elsif organization.favicon.attached? %>
      <%= image_tag organization.attached_uploader(:favicon).variant_path(:medium),
          alt: t("logo", scope: "decidim.accessibility", organization: current_organization_name) %>
    <% else %>
      <span><%= current_organization_name %></span>
    <% end %>
  <% end %>
<% else %>
  <%= Decidim.application_name %>
<% end %>
```

### G. 管理画面ビュー拡張
Defaceを使用して管理画面にモバイルロゴアップロードフィールドを追加。

```ruby
# decidim-cfj/app/overrides/add_mobile_logo_to_organization_appearance.rb
Deface::Override.new(
  virtual_path: "decidim/admin/organization_appearance/form/_images",
  name: "add_mobile_logo_field",
  insert_after: "[data-fieldset-for='favicon']",
  text: %q{
    <div>
      <%= form.upload(
        :mobile_logo,
        dimensions_info: current_organization.attached_uploader(:mobile_logo).dimensions_info,
        extension_allowlist: current_organization.attached_uploader(:mobile_logo).extension_allowlist,
        help_i18n_scope: "decidim.forms.file_help.image",
        label: t("mobile_logo", scope: "activemodel.attributes.organization"),
        button_class: "button button__sm button__transparent-secondary"
      ) %>
    </div>
  }
)
```

### H. 言語ファイル
日本語と英語の翻訳を追加。

```yaml
# config/locales/ja.yml
ja:
  activemodel:
    attributes:
      organization:
        mobile_logo: "モバイル版ロゴ"
  decidim:
    admin:
      organization_appearance:
        form:
          mobile_logo_help: "モバイル版で表示されるロゴ画像です。推奨サイズ: 360x120px"

# config/locales/en.yml
en:
  activemodel:
    attributes:
      organization:
        mobile_logo: "Mobile logo"
  decidim:
    admin:
      organization_appearance:
        form:
          mobile_logo_help: "Logo image displayed on mobile version. Recommended size: 360x120px"
```

## 3. 実装の利点

1. **既存機能の再利用**: Active StorageとImageUploaderの仕組みをそのまま活用
2. **後方互換性**: faviconへのフォールバック機能により、既存の動作を維持
3. **管理の容易性**: 管理画面から簡単に設定・変更可能
4. **拡張性**: Decidimコアを変更せず、デコレーターパターンで拡張
5. **保守性**: Decidimのアップデートに対して影響を最小限に抑制

## 4. テスト項目

1. モバイルロゴのアップロード機能
2. モバイルロゴの表示（モバイルビュー）
3. フォールバック機能（mobile_logo → favicon → テキスト）
4. 画像の削除機能
5. バリデーション（ファイルサイズ、形式）

## 5. 注意事項

- 画像の推奨サイズは360x120ピクセル
- 対応フォーマット: PNG, JPG, WEBP
- ファイルサイズ上限は組織設定に依存
- レスポンシブデザインを考慮したサイズ設定が必要
