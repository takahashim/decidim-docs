# Decidim OAuth2 メールアドレス処理に関する調査報告

## 調査概要

調査日: 2025-07-24  
調査目的: DecidimにおけるOAuth2ログイン時のメールアドレスの扱いと、既存アカウントとの紐付け処理の仕組みを明らかにする

### 調査した主な疑問点
1. すでに通常のログイン用に登録されているメールアドレスを使って、OAuth2で同じメールアドレスを使った別のアカウントを作ることが可能か？
2. 同じメールアドレスであればOAuth2のユーザーは自動的に通常ログイン用のアカウントと紐付けられるのか？
3. メールアドレスを持たないOAuth2プロバイダーの場合はどうなるか？

## 調査方法

1. OAuth2/Omniauth関連のコードを検索して実装箇所を特定
2. ユーザー認証処理とメールアドレスの重複チェックの実装を調査
3. 既存アカウントとOAuth2アカウントの紐付け処理を確認
4. 実際の挙動を確認するためテストコードを調査

## 調査結果

### 1. 主要な実装ファイル

#### コアとなる実装
- `/decidim/decidim-core/app/commands/decidim/create_omniauth_registration.rb`
  - OAuth認証後のユーザー作成/紐付けロジックの中核
- `/decidim/decidim-core/app/controllers/decidim/devise/omniauth_registrations_controller.rb`
  - OmniAuthコールバックのコントローラー
- `/decidim/decidim-core/app/forms/decidim/omniauth_registration_form.rb`
  - OmniAuth登録フォーム（メールアドレス必須バリデーション含む）
- `/decidim/decidim-core/app/models/decidim/user.rb`
  - ユーザーモデルのバリデーション定義
- `/decidim/decidim-core/app/models/decidim/identity.rb`
  - OAuthプロバイダーとユーザーの紐付け情報を管理

#### カスタマイズ実装（decidim-cfj）
- `/decidim-cfj/decidim-user_extension/app/commands/concerns/decidim/user_extension/create_omniauth_commands_overrides.rb`
  - 日本版Decidimでのカスタマイズ実装

### 2. メールアドレスの処理フロー

#### 2.1 OAuth認証時の処理（CreateOmniauthRegistration）

```ruby
def create_or_find_user
  @user = User.find_or_initialize_by(
    email: verified_email,
    organization:
  )

  if @user.persisted?
    # 既存ユーザーが存在する場合
    if !@user.confirmed? && @user.email == verified_email
      @user.skip_confirmation!
      @user.after_confirmation
    end
    @user.tos_agreement = "1"
    @user.save!
  else
    # 新規ユーザーの場合
    @user.email = (verified_email || form.email)
    @user.name = form.name.gsub(REGEXP_SANITIZER, "")
    @user.nickname = form.normalized_nickname
    @user.newsletter_notifications_at = nil
    @user.password = SecureRandom.hex
    attach_avatar(form.avatar_url) if form.avatar_url.present?
    @user.skip_confirmation! if verified_email
    @user.tos_agreement = "1"
    @user.save!
  end
end
```

#### 2.2 既存Identityのチェック

```ruby
def existing_identity
  @existing_identity ||= Identity.find_by(
    user: organization.users,
    provider: form.provider,
    uid: form.uid
  )
end
```

### 3. バリデーションルール

#### 3.1 Userモデルのバリデーション

```ruby
# decidim/decidim-core/app/models/decidim/user.rb:46
validates :email, :nickname, uniqueness: { scope: :organization }, 
          unless: -> { deleted? || managed? || nickname.blank? }
```

- 同一organization内でメールアドレスの重複は不可
- 削除済みユーザーやマネージドユーザーは除外

#### 3.2 Identityモデルのバリデーション

```ruby
# decidim/decidim-core/app/models/decidim/identity.rb
validates :uid, presence: true, uniqueness: { scope: [:provider, :organization] }
```

- provider + uid + organizationの組み合わせでユニーク

#### 3.3 OmniauthRegistrationFormのバリデーション

```ruby
# decidim/decidim-core/app/forms/decidim/omniauth_registration_form.rb:18
validates :email, presence: true
```

- フォームレベルでメールアドレスは必須

### 4. 動作パターン

#### パターン1: 新規ユーザー（メールアドレスが未登録）
1. 新規Userレコードが作成される
2. OAuthプロバイダーから取得したメールアドレスが設定される
3. verified_emailがある場合は確認済み状態で作成
4. Identityレコードが作成され、ユーザーと紐付けられる

#### パターン2: 既存ユーザー（同じメールアドレスが登録済み）+ verified_emailあり
1. 既存のUserレコードが検索される（find_or_initialize_by）
2. **新規アカウントは作成されない**
3. 既存ユーザーが未確認の場合は確認済みに更新
4. 新規Identityレコードが作成され、既存ユーザーと紐付けられる

#### パターン3: 既存ユーザー + verified_emailなし
1. form.invalidとなり、エラーが発生（テストコードで確認）
2. 既存アカウントとの紐付けは行われない
3. `broadcast(:error)`でエラー処理

#### パターン4: 既にOAuth認証済み（同じprovider + uid）
1. existing_identityが見つかる
2. 紐付けられているユーザーでログイン
3. 新規作成処理はスキップ

#### パターン5: メールアドレスを持たないOAuth2プロバイダー
1. verified_emailがnullとなる
2. OmniauthRegistrationFormのバリデーションエラー（email必須）
3. `broadcast(:invalid)`が発生
4. メールアドレス入力画面（new.html.erb）が表示される
5. ユーザーが手動でメールアドレスを入力
6. 新規ユーザーとして作成（既存アカウントとの統合なし）

### 5. テストコードによる動作確認

`decidim/decidim-core/spec/commands/decidim/create_omniauth_registration_spec.rb`より：

#### 既存ユーザーとのリンク（line 216-229）
```ruby
context "with a verified email" do
  let(:verified_email) { email }

  it "links a previously existing user" do
    user = create(:user, email:, organization:)
    expect { command.call }.not_to change(User, :count)
    expect(user.identities.length).to eq(1)
  end

  it "confirms a previously existing user" do
    create(:user, email:, organization:)
    expect { command.call }.not_to change(User, :count)

    user = User.find_by(email:)
    expect(user).to be_confirmed
  end
end
```

#### verified_emailがない場合の動作（line 231-248）
```ruby
context "with an unverified email" do
  let(:verified_email) { nil }

  it "does not link a previously existing user" do
    user = create(:user, email:, organization:)
    expect { command.call }.to broadcast(:error)

    expect(user.identities.length).to eq(0)
  end
end
```

### 6. メールアドレスを持たないOAuth2プロバイダーの処理詳細

#### 6.1 処理フロー

1. **OmniAuthコールバック時**
   ```ruby
   # OmniauthRegistrationsController#create
   @form.email ||= verified_email  # verified_emailがnullならform.emailもnull
   ```

2. **フォームバリデーション**
   ```ruby
   # OmniauthRegistrationForm
   validates :email, presence: true  # メールアドレス必須
   ```

3. **エラー時の画面表示**
   ```ruby
   on(:invalid) do
     set_flash_message :notice, :success, kind: @form.provider.capitalize
     render :new  # メールアドレス入力画面を表示
   end
   ```

4. **メールアドレス入力画面**
   - ファイル: `/decidim/decidim-core/app/views/decidim/devise/omniauth_registrations/new.html.erb`
   - 入力項目：
     - 名前（name）
     - ニックネーム（nickname）
     - メールアドレス（email） - 手動入力必須

5. **手動入力後の処理**
   ```ruby
   # CreateOmniauthRegistration#create_or_find_user
   @user.email = (verified_email || form.email)  # 手動入力されたメールアドレスを使用
   @user.skip_confirmation! if verified_email    # verified_emailがnullなら確認プロセスが必要
   ```

## 結論

### 質問への回答

1. **すでに通常のログイン用に登録されているメールアドレスを使って、OAuth2で同じメールアドレスを使った別のアカウントを作ることが可能か？**
   - **不可能です**。同じメールアドレスで別アカウントは作成できません。

2. **同じメールアドレスであればOAuth2のユーザーは自動的に通常ログイン用のアカウントと紐付けられるか？**
   - **verified_emailがある場合のみ自動的に紐付けられます**。
   - OAuthプロバイダーがメールアドレスを検証済みとして提供する場合のみ、既存アカウントと統合されます。

3. **メールアドレスを持たないOAuth2プロバイダーの場合はどうなるか？**
   - **追加のメールアドレス入力画面が表示されます**。
   - ユーザーは手動でメールアドレスを入力する必要があります。
   - 入力されたメールアドレスで新規アカウントが作成されます（既存アカウントとの自動統合なし）。
   - メールアドレスの確認プロセスが必要になります。

### セキュリティ上の考慮点

1. **メールアドレスの検証**
   - OAuthプロバイダーからverified_emailが提供される場合のみ自動紐付けを行う
   - これによりメールアドレスのなりすましを防止

2. **重複アカウントの防止**
   - データベースレベルでメールアドレスのユニーク制約
   - アプリケーションレベルでも重複チェック

3. **複数プロバイダー対応**
   - 一人のユーザーが複数のOAuthプロバイダー（Facebook、Google、LINE等）を使用可能
   - 各プロバイダーはIdentityレコードで管理

4. **メールアドレスなしプロバイダーへの対応**
   - 手動入力を要求することで、必ずメールアドレスを持つユーザーアカウントを作成
   - 既存アカウントとの自動統合を行わないことで、なりすましを防止

### 推奨事項

1. **OAuthプロバイダーの設定**
   - 可能な限りメールアドレスの取得と検証状態の確認を有効にする
   - メールアドレスを提供しないプロバイダーを使用する場合は、ユーザーへの説明を充実させる

2. **ユーザーへの説明**
   - 既存アカウントと同じメールアドレスでOAuth認証すると自動的に統合されることを明記
   - メールアドレスを持たないプロバイダーの場合、追加入力が必要なことを説明

3. **既存アカウントとの統合**
   - メールアドレスなしプロバイダーで既存アカウントとの統合を希望する場合：
     - 先に既存アカウントでログイン後、プロフィール設定からOAuthプロバイダーを追加
     - または管理者による手動統合

## 参考情報

### 関連ファイル一覧
- コア実装: `/decidim/decidim-core/app/commands/decidim/create_omniauth_registration.rb`
- コントローラー: `/decidim/decidim-core/app/controllers/decidim/devise/omniauth_registrations_controller.rb`
- フォーム: `/decidim/decidim-core/app/forms/decidim/omniauth_registration_form.rb`
- ビュー: `/decidim/decidim-core/app/views/decidim/devise/omniauth_registrations/new.html.erb`
- モデル: `/decidim/decidim-core/app/models/decidim/user.rb`, `/decidim/decidim-core/app/models/decidim/identity.rb`
- テスト: `/decidim/decidim-core/spec/commands/decidim/create_omniauth_registration_spec.rb`
- カスタマイズ: `/decidim-cfj/decidim-user_extension/app/commands/concerns/decidim/user_extension/create_omniauth_commands_overrides.rb`