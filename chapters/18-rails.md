# 第18章 Railsで作り直す：自作との対応で理解する

> **この章のゴール**：第17章で自作したCRUDアプリを Rails で作り直し、Rails の各機能が自作のどの部品に当たり、何を自動でしてくれているのかを説明できるようになります。
> **目安時間**：読む 5時間 ＋ 演習 20時間
> **前提**：第17章まで

## この章で学ぶこと

- `rails new` で作られるフォルダとファイルを、自作フレームワークの部品と対応づけて説明できる
- ルーティング・マイグレーション・ActiveRecord（関連・バリデーション・スコープ）を使って CRUD を作れる
- ログを読んで N+1 を見つけ、`includes` で直せる
- Strong Parameters・CSRF対策・XSS対策が Rails でどう自動化されているかを、自作版と対比して説明できる
- Rails のテスト（Minitest・結合テスト・system test）を書いて実行でき、現場で RSpec が多いことを知っている
- `rails console` とログを、調査の道具として使える

## なぜここで Rails なのか

ここまで、ソケットからHTTPサーバを作り、フレームワークを作り、ORMを作り、それらを組み合わせてアプリを作りました。いよいよ Rails です。

元教材の目的は「Railsを使えるようになること」ではなく「Railsを理解すること」でした。最初から Rails を学ぶと、多くのことが魔法に見えます。`resources :posts` と1行書くだけでURLが7つできる。フォームに勝手に謎の隠しフィールドが入る。`<%= %>` に書いた `<script>` が勝手に無害になる。

でも、あなたはもう、その中身を自分で作りました。この章では、Rails の機能1つ1つに対して「これは自作のあれだ」と対応づけていきます。対応づけられると、Rails は魔法ではなく「よくできた設計の集まり」に見えてきます。そして、魔法に見えなくなると、問題が起きたときに自分で調べられるようになります。

### この章が想定するバージョン

この章は **Rails 8系（執筆時点で 8.0 / 8.1）** を想定しています。Rails 7.1 以降なら大部分は同じですが、違う箇所はその都度明記します。Rails 8.1 は Ruby 3.2 以上が必要です。この教材の前提（Ruby 3.3 以降）を満たしていれば動きます。

Rails は毎年のように変わります。この章の出力例と、手元の出力が少し違うことは普通にあります。そのときは、自分のバージョンの Rails ガイド（https://guides.rubyonrails.org/ 、日本語版は https://railsguides.jp/ ）を確認してください。バージョンは `bin/rails -v` で確かめられます。

## インストールと rails new

### Rails を入れる

Rails は gem として配布されています。第1章で入れた Ruby（macOS 標準の古い Ruby ではないもの）が使われていることを確かめてから、インストールします。

```bash
ruby -v
gem install rails
rails -v
```

`ruby -v` が 3.3 以上を示していることを確認してください。`rails -v` で `Rails 8.1.x` のように表示されれば成功です。

### アプリを作る

この章の作業フォルダは `~/workspace/ch18-rails/` です。その中に、投稿（Post）を管理するアプリ `post_app` を作ります。第17章で別のテーマを選んだ人は、名前を読み替えてください。

```bash
mkdir -p ~/workspace/ch18-rails
cd ~/workspace/ch18-rails
rails new post_app
cd post_app
```

`rails new` は数分かかります。フォルダとファイルを大量に作り、必要な gem をインストールし、Git のリポジトリも初期化します。既定のDBは SQLite です。第15〜17章と同じDBなので、安心して比べられます。

### できたフォルダを自作と対応づける

主なフォルダを、自作の部品と対応づけます。

| Rails のフォルダ・ファイル | 何が入るか | 自作では |
|---|---|---|
| `config/routes.rb` | URLとアクションの対応 | Router への登録（第14章） |
| `app/controllers/` | Controller | ControllerBase を継承したクラス（第14章） |
| `app/models/` | Model | BaseModel を継承したクラス（第16章） |
| `app/views/` | ERB のテンプレート | View（第14章） |
| `db/migrate/` | テーブルの変更履歴 | 手書きの `db/schema.sql`（第17章） |
| `db/schema.rb` | 現在のテーブル定義（自動生成） | 同上 |
| `config.ru` | Rack アプリの入り口 | `config.ru`（第13章） |
| `config/database.yml` | DB接続の設定 | `Database.connection` の設定（第16章） |
| `test/` | テスト | `test/`（第6章以降） |
| `log/` | ログファイル | ターミナルへの出力 |
| `storage/` | SQLite のDBファイルなど | `db/development.sqlite3` |
| `bin/rails` | 各種コマンドの入り口 | `bin/console` など |

`.gitignore` も最初から作られています。開いて、`/log/*`、`/tmp/*`、`/storage/*`（SQLite のDBファイルを含む）、`/.env*` などが無視されていることを確かめてください。`config/master.key` という秘密の鍵ファイルも、Git に入らないようになっています。秘密情報の扱いは第19章・第20章で詳しく扱います。

`config.ru` を開いてみてください。第13章で見た Rack の形そのものです。Rails アプリも、正体は1つの Rack アプリです。

## 起動と安全確認

開発用サーバを起動します。待ち受けアドレスは、必ず明示的に `127.0.0.1` を指定します。

```bash
bin/rails server -b 127.0.0.1
```

別のターミナルで、待ち受け先を確かめます。

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

`127.0.0.1:3000` と表示されればOKです。ブラウザで `http://127.0.0.1:3000` を開くと、Rails の初期画面が表示されます。止めるときは、サーバのターミナルで Ctrl+C を押します。

なぜ毎回 `-b 127.0.0.1` を付けるのでしょう。Rails の `server` コマンドは、開発環境（development）では既定で localhost に待ち受けます。ところが、それ以外の環境では既定が `0.0.0.0`（すべてのネットワーク）です。環境変数 `HOST` や `BINDING` でも変わります。ネットで見かける `bin/rails server -b 0.0.0.0` という書き方は、Docker などの特別な事情があるときのものです。この教材では使いません。「既定値に任せず、明示して、確かめる」を習慣にしてください。

### Rails もミドルウェアの積み重ね

次のコマンドを実行してみてください。

```bash
bin/rails middleware
```

`use ActionDispatch::HostAuthorization`、`use Rack::MethodOverride`、`use ActionDispatch::Cookies` など、たくさんの行が並びます。最後は `run PostApp::Application.routes` です。第13章で学んだ、Rack のミドルウェアの積み重ねそのものです。第17章で迷った `_method` の読み替えは、`Rack::MethodOverride` が担当しています。

## MVC とリクエストの流れ

Rails は MVC（Model・View・Controller）という構造を採用しています。第17章で描いたリクエストの流れを、Rails の言葉で書き直すと次のようになります。

```text
ブラウザ
  → Puma（Webサーバ。第11〜13章の自作サーバに当たる）
  → Rack ミドルウェアの列（Cookie・セッション・CSRFの準備など）
  → config/routes.rb（Router）
  → PostsController の before_action（共通処理）
  → PostsController#index などのアクション
  → Post モデル（ActiveRecord） → SQLite
  → app/views/posts/index.html.erb（View）
  → HTTPレスポンス
```

自作したものと、ほぼ同じ形をしています。違うのは、各部品がずっと多くの状況に対応していることです。

### 「設定より規約」

Rails の大きな特徴は「設定より規約（Convention over Configuration）」です。名前の付け方の約束を守れば、設定を書かなくて済みます。

- `Post` モデルは、自動的に `posts` テーブルを使う（第16章で悩んだテーブル名の推測です。`Person` なら `people` になります）
- `PostsController#index` は、自動的に `app/views/posts/index.html.erb` を表示する
- `resources :posts` は、7つの決まったURLとアクションを作る

規約を知らないと、どこで何が決まっているのか分からず、魔法に見えます。規約を知っていれば、設定ファイルを探し回らずに済みます。

## ルーティング

`config/routes.rb` に次の1行を書きます。

```ruby
Rails.application.routes.draw do
  resources :posts
  root "posts#index"
end
```

`resources :posts` が作るルートは、次のコマンドで確認できます。

```bash
bin/rails routes -c posts
```

出力例です（Rails 8.1 の場合）。

```text
   Prefix Verb   URI Pattern               Controller#Action
    posts GET    /posts(.:format)          posts#index
          POST   /posts(.:format)          posts#create
 new_post GET    /posts/new(.:format)      posts#new
edit_post GET    /posts/:id/edit(.:format) posts#edit
     post GET    /posts/:id(.:format)      posts#show
          PATCH  /posts/:id(.:format)      posts#update
          PUT    /posts/:id(.:format)      posts#update
          DELETE /posts/:id(.:format)      posts#destroy
```

第17章で設計したルート表と、ほぼ同じです。PATCH と PUT の両方が update に向いているのは、歴史的な理由です。左端の Prefix は、URLを作るメソッドの名前になります。`posts_path` は `/posts`、`post_path(@post)` は `/posts/1` を返します。URLを文字列で手書きせず、このメソッドを使うのが Rails の書き方です。URLを変えたときに、書き換え漏れが起きないからです。

`(.:format)` は、`/posts.json` のように拡張子を付けられることを表しています。API については第21章で扱います。

## マイグレーションとモデル

### モデルを生成する

Rails では、テーブルとモデルを `generate` コマンドで作るのが普通です。

```bash
bin/rails generate model Post title:string body:text
```

このコマンドは、モデル `app/models/post.rb`、テストの雛形、そしてマイグレーションファイルを作ります。

### マイグレーションとは

マイグレーションは「テーブルをどう変えるか」を Ruby で書いた、変更の履歴です。`db/migrate/` に、日時付きのファイル名で保存されます。中身は次のような形です（数字の部分は作った日時で変わります）。

```ruby
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body

      t.timestamps
    end
  end
end
```

`Migration[8.1]` の数字は、このファイルを作った Rails のバージョンです。Rails 8.0 なら `[8.0]` になります。`t.timestamps` は、`created_at` と `updated_at` の2つのカラムを作ります。値は Rails が自動で入れます。`id` カラムは書かなくても自動で作られます。

第15章で学んだとおり、空にできないカラムには DB 側でも制約を付けます。生成されたファイルを編集して、`t.string :title, null: false` のように `null: false` を足してください。マイグレーションを適用する前なら、このように編集してかまいません。

```bash
bin/rails db:migrate
```

これでテーブルが作られ、`db/schema.rb` が更新されます。`schema.rb` は「今のテーブル定義」を表すファイルで、自動生成されます。手で編集してはいけません。

### なぜ変更履歴で管理するのか

第17章では、`db/schema.sql` を手で書きました。1人で作るならそれで足ります。でもチームで開発すると、Aさんはカラムを足し、Bさんはインデックスを足し、本番DBにも同じ変更を反映する必要があります。「どの変更が、どのDBに、どこまで適用されたか」を管理しなければなりません。

マイグレーションは、Git のコミットのように変更を1つずつ記録します。Rails はDBの中に「どのマイグレーションまで適用したか」を記録していて、`db:migrate` は未適用のものだけを実行します。間違えたときは `bin/rails db:rollback` で1つ戻せます。

注意点が1つあります。**チームに共有した（push した）マイグレーションは、後から書き換えてはいけません。** 他の人のDBや本番DBには、もう古い内容で適用されているからです。直したいときは、修正のための新しいマイグレーションを追加します。

## ActiveRecord

ActiveRecord は Rails の ORM です。第16章で作ったものの、ずっと大きな版です。`rails console` で触ってみましょう。

```bash
bin/rails console
```

```ruby
post = Post.new(title: "はじめての投稿", body: "テストです")
post.save                       # => true
Post.count                      # => 1
Post.find(post.id)
Post.find_by(title: "はじめての投稿")
Post.where(title: "はじめての投稿").to_a
post.update(title: "タイトル変更")
post.destroy
```

`find` は見つからないと例外 `ActiveRecord::RecordNotFound` を投げ、`find_by` は `nil` を返します。第16章で「どちらにするか」を悩んだ設計判断を、Rails は両方用意することで解決しています。Controller で `find` を使うと、見つからないときに Rails が自動で 404 を返してくれます。

### バリデーション

`app/models/post.rb` にバリデーションを書きます。第16章の「発展」で考えた DSL の形です。

```ruby
class Post < ApplicationRecord
  validates :title, presence: true, length: { maximum: 100 }
  validates :body, presence: true, length: { maximum: 2000 }
end
```

```ruby
post = Post.new(title: "", body: "")
post.save                  # => false
post.errors.full_messages  # => ["Title can't be blank", "Body can't be blank"]
```

`save` が false を返し、理由が `errors` に入る。自作版と同じ考え方です。エラーメッセージは日本語化することもできます（rails-i18n という gem などを使います）。

### 関連（アソシエーション）

投稿にコメントを付けられるようにします。コメントは、どの投稿へのコメントかを表す `post_id` を持ちます。

```bash
bin/rails generate model Comment post:references body:text
bin/rails db:migrate
```

`post:references` は、`post_id` カラムと、外部キー制約（存在しない投稿の id を入れられないようにするDBの制約）を作ります。生成された `app/models/comment.rb` には、`belongs_to :post` が書かれています。投稿側には、自分で次の1行を足します。

```ruby
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
  # validates ...
end
```

これで `post.comments` で投稿のコメント一覧が、`comment.post` でコメントの投稿が取れます。`dependent: :destroy` は「投稿を削除したら、そのコメントも削除する」という意味です。これがないと、行き先のないコメントが残るか、外部キー制約で投稿を削除できなくなります。

`belongs_to` は、既定で「関連先が存在すること」をバリデーションします。`post_id` が空のコメントは保存できません。

### スコープ

よく使う条件に名前を付けられます。これをスコープと呼びます。

```ruby
class Post < ApplicationRecord
  scope :recent, -> { order(created_at: :desc) }
end
```

`Post.recent` で新しい順の一覧が取れます。`Post.where(...).recent.limit(10)` のように、つなげて書けます。第16章で「Query は条件のバリエーション」と気づきました。ActiveRecord では、`where` や `order` が Query オブジェクト（`ActiveRecord::Relation`）を返し、それをつなげていきます。実際にSQLが発行されるのは、結果が必要になったとき（`each` や `to_a` を呼んだとき）です。

発行されるSQLは `to_sql` で見られます。

```ruby
Post.where(title: "はじめての投稿").recent.to_sql
# => "SELECT \"posts\".* FROM \"posts\" WHERE \"posts\".\"title\" = 'はじめての投稿' ORDER BY \"posts\".\"created_at\" DESC"
```

`to_sql` の表示では値が埋め込まれて見えますが、実際の実行時には値はバインドされます。

## N+1 と includes

第16章で、N+1問題を自分の手で起こしました。Rails でも、何も考えずに書くと同じことが起きます。一覧画面で、各投稿のコメントを表示するとします。

```erb
<% @posts.each do |post| %>
  <h2><%= post.title %></h2>
  <% post.comments.each do |comment| %>
    <p><%= comment.body %></p>
  <% end %>
<% end %>
```

この画面を開いて、サーバのターミナル（または `log/development.log`）を見ると、次のようなログが出ます（例。数値や細部は環境とバージョンで変わります）。

```text
  Post Load (0.2ms)  SELECT "posts".* FROM "posts"
  ↳ app/views/posts/index.html.erb:1
  Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 1]]
  ↳ app/views/posts/index.html.erb:3
  Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 2]]
  ↳ app/views/posts/index.html.erb:3
  Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 3]]
  ↳ app/views/posts/index.html.erb:3
```

同じ形の `Comment Load` が、投稿の数だけ並んでいます。N+1 です。`↳` の行は、そのSQLを発行したコードの場所を示しています。ここを見れば、どこを直せばよいか分かります。

直し方は、Controller で関連を先読みすることです。

```ruby
@posts = Post.includes(:comments).recent
```

すると、コメントの取得が1回にまとまります。ログには `WHERE "comments"."post_id" IN (...)` という形のSQLが1行だけ出ます（値の表示のされ方はバージョンで異なります）。第16章で、`IN` を使ってまとめて取る方法を自分で考えました。`includes` は、それを自動で行ってくれます。

N+1 に気づく手段も紹介しておきます。

- ログを読む。同じ形のSQLが並んでいたら疑う
- `strict_loading`：先読みしていない関連を読もうとしたら例外にする機能（Rails 6.1 以降）
- テストで `assert_queries_count` を使い、発行するSQLの回数を固定する（Rails 7.2 以降）
- 現場では bullet という gem もよく使われる

細かい点ですが、`post.comments.count` は先読みしていても毎回 `COUNT` のSQLを発行します。先読みした結果を使いたいときは `size` を使います。こうした違いも、ログを見る習慣があれば気づけます。

## Controller と Strong Parameters

### アクションの基本形

Controller のアクションは、第17章で書いたものとほとんど同じ流れです。作成（create）の典型的な形を見てみます。

```ruby
def create
  @post = Post.new(post_params)

  if @post.save
    redirect_to @post, notice: "投稿を作成しました"
  else
    render :new, status: :unprocessable_entity
  end
end
```

成功したらリダイレクト、失敗したら 422 でフォームを再表示。第17章の PRG パターンそのものです。`:unprocessable_entity` は 422 を表す名前です。Rack 3.1 以降では、新しい名前の `:unprocessable_content` も使えます。Rails 8.1 の雛形は、環境に応じて後者を使います。どちらも 422 です。

なぜ 422 が大事なのでしょう。Rails 7 以降は、Turbo という仕組みがフォームの送信を JavaScript で行います。Turbo は、フォーム送信の応答が 200 の HTML だと、画面を正しく更新しません。失敗時に 422 を返すことで、エラー付きのフォームが表示されます。「失敗したのにエラーが表示されない」という相談は、ここが原因のことがよくあります。

同様に、更新や削除の後のリダイレクトには `status: :see_other`（303）を付けるのが Rails の雛形の書き方です。DELETE の後に、ブラウザが次も DELETE で行き先を取りに行かないようにするためです。

### Strong Parameters とは

`post_params` の中身が、Strong Parameters と呼ばれる仕組みです。

```ruby
private

def post_params
  params.expect(post: [:title, :body])   # Rails 8.0 以降
  # Rails 7.x では: params.require(:post).permit(:title, :body)
end
```

これは「フォームから受け取ってよい項目は title と body だけ」という許可リストです。

なぜ必要なのでしょう。もし `Post.new(params[:post])` と、送られてきた値をそのまま渡せたらどうなるか考えます。第19章で、投稿に持ち主（`user_id`）を付けます。攻撃者が、開発者ツールでフォームに `post[user_id]=2` という項目を書き足して送信したら、他人の投稿として保存できてしまいます。管理者フラグがあれば、自分を管理者にできるかもしれません。これを「一括代入（Mass Assignment）」の脆弱性と呼びます。

第16章で、カラム名を許可リストで制限しました。Strong Parameters は、同じ「許可リスト」の考え方を、フォームの入力に適用したものです。Rails は、許可していないパラメータをモデルにまとめて渡そうとすると、例外を投げて止めます。

`params.expect` は Rails 8.0 で追加されました。`require` と `permit` の組み合わせより、不正な形のパラメータ（ハッシュのはずが文字列になっている、など）を 400 エラーとして扱いやすくなっています。Rails 7 系のコードを読むときは、`require(...).permit(...)` の形が出てきます。意味は同じと考えてかまいません。

## View と XSS

### 自動エスケープ

Rails の ERB では、`<%= %>` で出力した値は**自動的にエスケープ**されます。第14章で、出力するたびにエスケープする仕組みを自分で作りました。Rails ではそれが既定の動きです。

```erb
<p><%= @post.body %></p>
```

本文が `<script>alert(1)</script>` でも、HTMLには `&lt;script&gt;alert(1)&lt;/script&gt;` と出力されます。

### 自動エスケープを外す書き方に注意

逆に、エスケープを外す書き方もあります。

- `<%= raw @post.body %>`
- `<%= @post.body.html_safe %>`
- `<%== @post.body %>`

どれも「この文字列は安全なHTMLだ」と Rails に宣言するものです。利用者が入力した値に使うと、XSS の穴になります。`html_safe` は「安全にする」メソッドではありません。「安全だと信じる」メソッドです。名前に騙されないでください。

どうしても一部のHTMLタグ（太字や改行など）を許したいときは、`sanitize` ヘルパーを使います。許可したタグと属性だけを残し、それ以外を取り除きます。これも許可リストの考え方です。

もう1つ、見落としやすい穴があります。利用者が入力したURLを、そのままリンク先にする場合です。`<%= link_to "サイト", @profile.website %>` で、website が `javascript:alert(1)` だと、クリックしたときにスクリプトが実行されます。エスケープでは防げません。URLは保存するときに `http://` か `https://` で始まるものだけを許可するバリデーションを付けます。

## CSRF 対策の自動化

第17章で苦労して作った CSRF 対策は、Rails では既定で有効になっています。

### どこで何が起きているか

まず、Controller の親クラス `ApplicationController` で、CSRF の検証が有効になっています。`protect_from_forgery` という設定です。新しいアプリでは、明示的に書かなくても有効になる設定（`default_protect_from_forgery`）が効いています。検証に失敗すると例外 `ActionController::InvalidAuthenticityToken` が発生し、リクエストは処理されません。

次に、`form_with` でフォームを作ると、隠しフィールドが自動で入ります。ブラウザでフォームのページを開き、HTMLのソースを見てみてください。

```html
<input type="hidden" name="authenticity_token" value="（長いランダムな文字列）" autocomplete="off">
```

これが第17章の CSRFトークンです。トークンはセッションに結び付けられていて、送信時に照合されます。削除ボタンを `button_to "削除", @post, method: :delete` で作ると、小さなフォームが作られ、同じくトークンが入ります。

さらに、レイアウトファイル `app/views/layouts/application.html.erb` には `<%= csrf_meta_tags %>` が書かれています。JavaScript（Turbo など）からリクエストを送るとき、ここに置かれたトークンを読み取って、ヘッダに付けて送ります。

### テスト環境では無効になっている

ここに大きな落とし穴があります。`config/environments/test.rb` を開くと、次の行があります。

```ruby
config.action_controller.allow_forgery_protection = false
```

テスト環境では、CSRF の検証が**無効**になっています。テストのたびにトークンを用意するのは面倒なので、こうなっています。つまり、普通に書いた結合テストは、CSRF 対策が効いているかどうかを確かめていません。

CSRF 対策をテストで確かめたいときは、そのテストの中だけ一時的に有効にします。

```ruby
test "トークンのない投稿作成は拒否される" do
  ActionController::Base.allow_forgery_protection = true

  assert_no_difference("Post.count") do
    post posts_url, params: { post: { title: "攻撃", body: "x" } }
  end
  assert_response 422
ensure
  ActionController::Base.allow_forgery_protection = false
end
```

`ensure` で必ず元に戻します。戻し忘れると、他のテストに影響します。検証に失敗したときの例外は、テスト環境の設定（`show_exceptions = :rescuable`、Rails 7.1 以降の既定）により、422 のレスポンスに変換されます。正しいトークンを付けた場合に成功することも確かめましょう。フォームのページを `get` で取得し、レスポンスの HTML から `authenticity_token` の値を取り出して送ります。取り出し方は `css_select` というメソッドを調べてみてください。

## SQLインジェクション対策

ActiveRecord を使えば安全、というわけではありません。安全な書き方と危険な書き方を並べます。

```ruby
# 安全：ハッシュで条件を渡す（値はバインドされる）
Post.where(title: params[:q])

# 安全：プレースホルダを使う
Post.where("title LIKE ?", "%#{Post.sanitize_sql_like(params[:q])}%")

# 危険：文字列に埋め込む
Post.where("title = '#{params[:q]}'")
```

3つ目は、第16章で見た文字列埋め込みと同じです。ActiveRecord を経由しても、SQLインジェクションが起きます。2つ目では `#{}` を使っていますが、埋め込んでいるのは `LIKE` に渡す値の文字列を作る部分で、それ自体は `?` でバインドされています。`sanitize_sql_like` は、`%` や `_` を検索文字として扱うためのものです。

並び替えも注意が必要です。`Post.order(params[:sort])` のように、利用者の入力をそのまま `order` に渡してはいけません。Rails 6.1 以降は、カラム名として解釈できない文字列を `order` に渡すと例外にする仕組みを持っています。それでも、第16章と同じく許可リスト（`%w[title created_at]` に含まれるか）で絞るのが原則です。

## 自作版と Rails の対応表

ここまでの内容を、表にまとめます。演習18-4で、この表を自分の言葉で埋め直します。

| 自作したもの（章） | Rails で対応するもの | Rails が追加で引き受けていること |
|---|---|---|
| ソケットのHTTPサーバ（11・13） | Puma | 多数の接続・スレッド管理・タイムアウト |
| Rack 対応（13） | Rails アプリ自体が Rack アプリ。`config.ru` | ミドルウェアの標準セット |
| Router（14） | `config/routes.rb`、`resources` | URLを作るヘルパー、制約、入れ子 |
| ControllerBase・before_action（14） | `ApplicationController`、`before_action` | パラメータ解析、フィルタ、例外から 404 などへの変換 |
| View とエスケープ（14） | ERB の自動エスケープ、`sanitize` | 既定でエスケープ。外すには明示が必要 |
| Model・BaseModel（16） | `ApplicationRecord`（ActiveRecord） | 型変換、関連、変更検知、コールバック |
| `all` / `find` / `where`（16） | 同名のメソッドと `Relation` | OR・JOIN・集計・ページ分けなど |
| 許可リストによるカラム名検査（16） | ハッシュ条件、`order` の検査 | 文字列で条件を書けば、やはり危険 |
| `save` / `errors`（16） | `save` / `save!` / `errors` | 多数のバリデーション、トランザクション |
| Console（16） | `bin/rails console` | sandbox モード、`reload!` |
| `db/schema.sql`（17） | マイグレーション、`schema.rb` | 変更履歴、ロールバック |
| CSRF トークンとセッション（17） | `protect_from_forgery`、`form_with`、`csrf_meta_tags` | 既定で有効。ただしテスト環境では無効 |
| 一括代入の対策（なし） | Strong Parameters | 許可していないパラメータを止める |
| エラーページ（17） | `public/404.html` など、本番での例外の変換 | 本番では詳細を出さない |

この表で大事なのは、右端の列です。Rails が引き受けていることを知っていれば、それを外したとき（`html_safe` や CSRF の無効化）に何が失われるかが分かります。

## Rails のテスト

### Minitest が標準

Rails の標準のテストフレームワークは、第6章から使ってきた Minitest です。テストは `test/` フォルダにあり、次のコマンドで実行します。

```bash
bin/rails test
```

テストは、開発用とは別のテスト用DB（`storage/test.sqlite3`）で実行されます。テストのたびに、トランザクションでデータが巻き戻されます。第16章で悩んだ「テストごとにDBをきれいにする」問題を、Rails はこうして解決しています。

### フィクスチャ

テスト用のデータは `test/fixtures/` の YAML ファイルに書きます。これをフィクスチャと呼びます。

```yaml
# test/fixtures/posts.yml
first:
  title: 最初の投稿
  body: テスト用の本文です
```

テストの中では `posts(:first)` で、このデータを取り出せます。

### テストの種類

Rails のテストには、主に次の種類があります。

| 種類 | 置き場所 | 何を確かめるか |
|---|---|---|
| モデルのテスト | `test/models/` | バリデーション、スコープ、メソッド |
| 結合テスト（コントローラのテスト） | `test/controllers/`、`test/integration/` | リクエストを送り、ステータス・リダイレクト・DBの変化を確かめる |
| system test | `test/system/` | 本物のブラウザを自動操作して、画面の動きを確かめる |

モデルのテストは、たとえば次の形です。

```ruby
require "test_helper"

class PostTest < ActiveSupport::TestCase
  test "タイトルが空なら保存できない" do
    post = Post.new(title: "", body: "本文")
    assert_not post.save
    assert_includes post.errors[:title], "can't be blank"
  end
end
```

結合テストでは、`post posts_url, params: { ... }` のようにリクエストを送り、`assert_difference("Post.count")` で件数の変化を確かめます。第17章で自作した `post_form` 補助メソッドに当たるものが、最初から用意されています。

system test は、Chrome などのブラウザを自動で動かし、ボタンを押したり文字を入力したりします。JavaScript を含めた画面全体の動きを確かめられる反面、実行が遅く、不安定になりやすい面もあります。Rails 8.1 では、scaffold（雛形を自動生成するコマンド）で system test が既定では作られなくなりました。バージョンによって扱いが違うので、自分の環境を確かめてください。この教材では、モデルのテストと結合テストを中心にします。

### 現場では RSpec が多い

日本の Rails の現場では、Minitest ではなく **RSpec** というテストフレームワークを使っていることが多いです。`rspec-rails` という gem で導入し、`bundle exec rspec` で実行します。書き方は次のように、英語の文章に近い形です。

```ruby
RSpec.describe Post, type: :model do
  it "タイトルが空なら無効" do
    expect(Post.new(title: "", body: "本文")).not_to be_valid
  end
end
```

テストデータは、フィクスチャではなく FactoryBot という gem で作ることが多いです。書き方は違っても、「準備・実行・確認」の考え方と TDD の進め方は同じです。Minitest で考え方を身につけておけば、RSpec にはすぐに移れます。配属先のプロジェクトがどちらを使っているかを、最初に確かめてください。

## rails console

第16章で自作した Console の本物です。

```bash
bin/rails console
```

`Post.count` や `Post.last` で、すぐにデータを確かめられます。コードを書き換えたら、`reload!` で読み込み直せます。

便利な使い方をいくつか挙げます。

- `bin/rails console --sandbox`：終了時に、コンソールで行った変更をすべて巻き戻す。データを壊さずに試したいときに使う
- `Post.where(...).to_sql`：発行されるSQLを確かめる
- `Post.new.valid?` と `errors.full_messages`：バリデーションの動きを確かめる

注意点が1つあります。現場では、本番環境のコンソールに入れる権限を持つ人がいます。本番のコンソールでの操作は、本物の利用者のデータを直接変えます。`destroy_all` を1回打つだけで、取り返しがつきません。ジュニアのうちは、本番のコンソールでデータを変える操作をしない、と決めておくのが安全です。必要なときは、手順を書いて、先輩と一緒に行います。

## ログの読み方

開発中、サーバのターミナルと `log/development.log` には、リクエストごとのログが出ます。別のターミナルで `tail -f log/development.log` とすると、流れてくるログを見続けられます。投稿を作成したときのログの例を、1行ずつ読んでみます（数値や細部は環境とバージョンで変わります）。

```text
Started POST "/posts" for 127.0.0.1 at 2026-10-01 10:00:00 +0900
Processing by PostsController#create as TURBO_STREAM
  Parameters: {"authenticity_token" => "[FILTERED]", "post" => {"title" => "はじめての投稿", "body" => "テストです"}, "commit" => "Create Post"}
  TRANSACTION (0.1ms)  BEGIN immediate TRANSACTION
  Post Create (0.4ms)  INSERT INTO "posts" ("title", "body", "created_at", "updated_at") VALUES (?, ?, ?, ?) RETURNING "id"  [["title", "はじめての投稿"], ["body", "テストです"], ["created_at", "2026-10-01 01:00:00.000000"], ["updated_at", "2026-10-01 01:00:00.000000"]]
  ↳ app/controllers/posts_controller.rb:25:in 'PostsController#create'
  TRANSACTION (0.2ms)  COMMIT TRANSACTION
Redirected to http://127.0.0.1:3000/posts/1
Completed 302 Found in 12ms (ActiveRecord: 0.7ms (1 query, 0 cached) | GC: 0.0ms)
```

- `Started POST "/posts" for 127.0.0.1`：どのメソッドで、どのURLに、どこからリクエストが来たか
- `Processing by PostsController#create`：Router が選んだ Controller とアクション
- `Parameters`：受け取ったパラメータ。`authenticity_token` が `[FILTERED]` になっている
- `TRANSACTION` と `Post Create`：`save` の中でトランザクションが張られ、INSERT が実行された。値は `?` でバインドされている
- `↳`：そのSQLを発行したコードの場所
- `Redirected to` と `Completed 302 Found`：リダイレクトした。かかった時間と、DBの処理時間と、SQLの回数

`Parameters` の `[FILTERED]` に注目してください。`config/initializers/filter_parameter_logging.rb` に、ログに出さないパラメータの名前が並んでいます。`passw`、`email`、`token` などです。名前の一部が一致すれば隠されるので、`password` も `authenticity_token` も隠れます。パスワードがログファイルに残らないための仕組みです。

ログを読むときは、上から順に「どのリクエストが」「どのアクションに届き」「どんなSQLを何回発行し」「どんなステータスで終わったか」を追います。エラーが起きたら、`Completed` の行のステータスと、その直前の例外のメッセージを見ます。この読み方は、第23章の運用でも使います。Hash の表示形式（`=>` の前後の空白など）は Ruby のバージョンで少し違います。

## 実務ではこうなる

現場の Rails アプリは、何年も前から育ってきた大きなコードであることがほとんどです。新しく `rails new` する機会は多くありません。最初の仕事は「既存のコードを読んで、小さな変更をする」ことです。この章で身につけた「Rails の機能と仕組みを対応づける力」は、そのまま既存コードを読む力になります。

ジュニアがやりがちな失敗を挙げます。

- 画面の一覧で関連を表示し、N+1 を作る。開発環境ではデータが少ないので気づかず、本番で遅くなる。ログの `↳` を見る習慣で防げる
- Strong Parameters に `permit!`（すべて許可）を書いたり、`user_id` や `admin` を許可リストに入れたりする
- 表示を整えるために `html_safe` や `raw` を使い、XSS を作る
- 共有済みのマイグレーションを書き換え、他の人のDBと食い違わせる
- 結合テストが通ったので CSRF も大丈夫だと思い込む。テスト環境では無効になっている
- エラーの原因を調べずに、ネットの記事の設定を貼り付ける。`protect_from_forgery` を外す記事もある

また、Rails 7.2 以降の `rails new` には、Brakeman（Rails のコードからセキュリティの問題を探すツール）や、GitHub Actions の CI の設定が最初から含まれています。第20章と第22章で使います。

## 演習

すべて `~/workspace/ch18-rails/post_app/` で行います。サーバは `bin/rails server -b 127.0.0.1` で起動し、データはダミーだけを使います。scaffold（雛形の自動生成）は使わず、手で書いてください。scaffold で作ったコードは、演習18-2の最後で比較用に使います。

### 演習18-1：rails new と対応づけ

- やること
  1. `rails new post_app` でアプリを作り、`bin/rails server -b 127.0.0.1` で起動する
  2. `lsof -nP -iTCP:3000 -sTCP:LISTEN` で待ち受け先を確かめる
  3. `bin/rails middleware` の出力を見て、第13章で学んだ Rack ミドルウェアとの関係を確かめる
  4. `docs/rails-mapping.md` を作り、本文の「できたフォルダを自作と対応づける」の表を、自分の第17章のフォルダ名を使って書き直す
  5. 最初のコミットを作る。その前に `git status` で、`storage/` のDBファイルや `config/master.key` が含まれていないことを確かめる
- **完了条件**
  - `lsof` の出力が `127.0.0.1:3000` を示している（出力を `docs/rails-mapping.md` に貼る）
  - `git ls-files -- config/master.key 'storage/*.sqlite3'` の出力が空である
  - `docs/rails-mapping.md` に、10個以上のフォルダ・ファイルについて自作との対応が書かれている
- ヒント
  - `bin/rails middleware` の中から、`Rack::MethodOverride` と `ActionDispatch::Cookies` を探してみましょう。第17章で何を自作したかを思い出せます

### 演習18-2：第17章のアプリを TDD で作り直す

- やること
  1. `Post` モデルを `bin/rails generate model` で作り、マイグレーションに `null: false` を足してから `db:migrate` する
  2. バリデーションのテスト（モデルのテスト）を先に書いて RED を確かめてから、`validates` を書く
  3. `resources :posts` を設定し、`bin/rails routes -c posts` で第17章のルート表と比べる
  4. 結合テスト（`test/controllers/posts_controller_test.rb`）で、一覧・詳細・作成・更新・削除・存在しない id・バリデーション失敗を先に書き、Controller とビューを作る
  5. 別のブランチで `bin/rails generate scaffold` を使って同じものを生成し、自分のコードと見比べる。違いを `docs/rails-mapping.md` に書いたら、そのブランチは捨てる
- **完了条件**
  - `bin/rails test` がすべて成功する
  - 作成の失敗時に、件数が変わらず、ステータスが 422 になるテストがある
  - 存在しない id の詳細ページで 404 になるテストがある
  - `docs/rails-mapping.md` に、scaffold との違いが3つ以上書かれている
- ヒント
  - 存在しない id のテストは、`get post_url(id: 0)` の後に `assert_response :not_found` で確かめられます。テスト環境では `RecordNotFound` が 404 のレスポンスに変換されます
  - scaffold を生成するブランチは、`git switch -c try-scaffold` で作り、終わったら `git switch main` で戻ってから削除します

### 演習18-3：関連・スコープ・N+1

- やること
  1. `Comment` モデルを `post:references` で作り、`Post` に `has_many :comments, dependent: :destroy` を足す
  2. 投稿の詳細ページで、コメントの一覧と追加フォームを表示する（ルートは `resources :posts do resources :comments, only: [:create] end` のような入れ子にする）
  3. 一覧ページに各投稿のコメントを表示し、ログで N+1 を観察する。ログの該当部分を `docs/n-plus-one.md` に貼る
  4. `includes` で直し、直した後のログも貼る
  5. 新しい順のスコープ `recent` を作り、テストを書く
  6. 投稿を削除するとコメントも消えるテストを書く
- **完了条件**
  - `bin/rails test` がすべて成功する
  - `assert_queries_count`（Rails 7.2 以降）などで、投稿が3件のときに一覧ページのSQLの回数が投稿数に比例しないことを確かめるテストがある（Rails 7.1 の場合は、直す前後のログを `docs/n-plus-one.md` に貼ることで代える）
  - `docs/n-plus-one.md` に、直す前と後のログと、何が変わったかの説明がある
- ヒント
  - SQLの回数を数えるテストを先に書くと、直す前は RED、直した後は GREEN になります。これも TDD です
  - 一覧で `post.comments.count` を使うと、先読みしても SQL が出ます。`size` との違いを確かめてみましょう

### 演習18-4：Rails のセキュリティ機能をテストで確かめる（最後の項目は発展）

- やること
  1. `test/integration/security_test.rb` を作る
  2. CSRF：テストの中だけ `allow_forgery_protection` を有効にし、トークンなしの作成・更新・削除がすべて拒否され、件数と内容が変わらないテストを書く。正しいトークンでは成功するテストも書く
  3. XSS：本文に `<script>alert(1)</script>` を含む投稿の詳細・一覧ページで、生の `<script>alert(1)` が含まれず、`&lt;script&gt;` が含まれるテストを書く
  4. SQLインジェクション：タイトルの検索機能（`/posts?q=...`）を作り、`' OR '1'='1` で全件が返らないテストを書く
  5. Strong Parameters：許可していない項目（例：`id` や `created_at`）を送っても、その値が保存されないテストを書く
  6. `docs/rails-mapping.md` に、本文の「自作版と Rails の対応表」を自分の言葉で書き直す
  7. 発展（任意）：`bundle info activerecord` で ActiveRecord のソースの場所を調べ、`where` の定義を探して読む。第16章の自分の `where` と比べた感想を書く
- **完了条件**
  - `bin/rails test test/integration/security_test.rb` がすべて成功する
  - CSRF のテストで `ensure` を使い、他のテストに影響しないことを `bin/rails test` 全体の成功で確かめている
  - 検索の実装をわざと `where("title = '#{params[:q]}'")` に変えると SQLインジェクションのテストが失敗することを確かめ、元に戻した（その結果を `docs/rails-mapping.md` に記録する）
  - `docs/rails-mapping.md` の対応表に、自作の部品10個以上について「Rails で対応するもの」と「Rails が追加で引き受けていること」が書かれている
- ヒント
  - 正しいトークンを取り出すには、まず `get new_post_url` でフォームを取得します。その後、レスポンスの HTML から `input[name="authenticity_token"]` の値を読みます
  - 「対策を外すとテストが失敗する」ことを確かめないと、そのテストが本当に攻撃を検出できるのか分かりません

## 確認クイズ

**Q1.** `resources :posts` と書くと、どんなルートができますか。HTTPメソッドとURLとアクションの組を、少なくとも4つ挙げてください。

**Q2.** 結合テストで、トークンを付けずに `post posts_url, params: ...` を送ったら成功しました。CSRF 対策が効いていないのでしょうか。

**Q3.** ビューで `<%= @post.body.html_safe %>` と書きました。何が問題ですか。

**Q4.** 一覧ページが遅いと言われました。ログに `Comment Load ... WHERE "comments"."post_id" = ?` が何十行も並んでいます。原因と、直し方を答えてください。

**Q5.** Strong Parameters で `params.expect(post: [:title, :body])` と書くのはなぜですか。書かずに `Post.new(params[:post])` とできた場合に、何が起きうるかを答えてください。

### 解答と解説

**A1.** `GET /posts`（index）、`GET /posts/new`（new）、`POST /posts`（create）、`GET /posts/:id`（show）、`GET /posts/:id/edit`（edit）、`PATCH` と `PUT /posts/:id`（update）、`DELETE /posts/:id`（destroy）です。`bin/rails routes` で確かめられます。

**A2.** そうとは限りません。テスト環境では、`config/environments/test.rb` で CSRF の検証が無効になっています。CSRF 対策を確かめたいときは、そのテストの中だけ `ActionController::Base.allow_forgery_protection = true` にし、`ensure` で元に戻します。

**A3.** `html_safe` は「この文字列は安全なHTMLだ」と Rails に宣言するだけで、エスケープを外します。利用者が入力した本文に `<script>` が含まれていれば、そのまま実行され、XSS になります。普通は `<%= @post.body %>` と書き、自動エスケープに任せます。一部のタグを許したいなら `sanitize` を使います。

**A4.** N+1 問題です。投稿ごとにコメントを取るSQLが発行されています。Controller で `Post.includes(:comments)` のように関連を先読みすれば、コメントの取得が1回にまとまります。ログの `↳` の行で、SQLを発行したコードの場所を確かめてから直します。

**A5.** フォームから受け取ってよい項目を、許可リストで限定するためです。書かなければ、攻撃者がフォームに項目を書き足して、`user_id` や管理者フラグなど、変えてはいけない値を変えられます（一括代入の脆弱性）。第16章でカラム名を許可リストで制限したのと同じ考え方です。

## AIの使い方（この章）

Rails は情報が多く、AIも大量の知識を持っています。その分、古いバージョンの書き方が混ざりやすいことに注意してください。

推奨する聞き方の例です。

- 「Rails 8.1 を使っています。`params.expect` と `params.require(...).permit(...)` の違いを、初心者向けに説明してください。どのバージョンから使えるかも教えてください」
- 「このログ（貼る）で、N+1 が起きている行はどれですか。どう読めばそう判断できるのかを説明してください。修正コードは書かないでください」
- 「Rails の `protect_from_forgery` がどこでトークンを照合しているか、ソースコードのどのファイルを読めばよいか教えてください」

やってはいけない使い方です。

- 演習のモデル・コントローラ・テストを丸ごと生成させること。特に、テストをAIに書かせると「何を確かめるべきか」を考える練習がなくなります
- AIが勧める設定（`protect_from_forgery` を外す、`permit!` を使う、`html_safe` を付けるなど）を、理由を理解せずに採用すること。エラーを消すための設定変更が、セキュリティの穴になることがよくあります

## まとめ

- Rails の各機能は、自作した部品（サーバ・Router・Controller・View・ORM・CSRF対策）に対応している。対応づけられれば、Rails は魔法ではなく設計の集まりに見える
- Rails は CSRF 対策・XSS のエスケープ・一括代入の防止を既定で有効にしている。`html_safe`・`permit!`・文字列のSQLなどでそれを外せば、自作のときと同じ穴が開く
- テスト環境では CSRF の検証が無効になっている。攻撃が失敗するテストは、自分で検証を有効にして書く
- ログを読めば、リクエストの流れ・発行されたSQL・N+1 が見える。`rails console` と合わせて、調査の基本の道具にする

## 用語

| 用語 | 意味 |
|---|---|
| Ruby on Rails | Ruby で書かれたWebアプリケーションフレームワーク |
| 設定より規約 | 名前の付け方などの約束を守れば、設定を書かずに済むという Rails の考え方 |
| `resources` | REST の7つのルートをまとめて作る、ルーティングの書き方 |
| マイグレーション | テーブルの変更を Ruby で書いた履歴。`db:migrate` で適用する |
| `schema.rb` | 現在のテーブル定義を表す、自動生成されるファイル |
| 関連（アソシエーション） | `has_many` / `belongs_to` など、モデル同士のつながりを表す仕組み |
| スコープ | よく使う条件に名前を付けたもの。`scope :recent, -> { ... }` |
| `includes` | 関連をまとめて先読みし、N+1 を防ぐメソッド |
| Strong Parameters | フォームから受け取ってよい項目を許可リストで限定する仕組み |
| 一括代入（Mass Assignment） | 送られてきた値をまとめてモデルに渡し、変えてはいけない項目まで変えられてしまう脆弱性 |
| `authenticity_token` | Rails が自動でフォームに入れる CSRF トークン |
| Turbo | フォームの送信やページの切り替えを JavaScript で行う、Rails 標準の仕組み |
| フィクスチャ | YAML で書くテスト用のデータ |
| system test | ブラウザを自動操作して画面全体の動きを確かめるテスト |
| RSpec | 日本の Rails の現場でよく使われるテストフレームワーク |

---
[← 前の章](./17-crud-app.md) | [目次](../README.md) | [次の章 →](./19-auth-session.md)
