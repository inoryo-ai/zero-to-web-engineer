# 第14章 フレームワークを自作する

> **この章のゴール**：素の Rack アプリのつらさを体験し、それを解決する部品をテスト駆動で自作します。作るのは Router・Controller・render/redirect・View・Request/Params・before_action です。Rails の「中で何が起きているか」を、自分のコードとの対応で説明できるようになります。
> **目安時間**：読む 6時間 ＋ 演習 30時間
> **前提**：第13章まで

## この章で学ぶこと

- 素の Rack だけでアプリを書くと何がつらいかを、具体的に説明できる
- RED → GREEN → REFACTOR のリズムで、小さな振る舞いを1つずつ積み上げられる
- Router・Controller・render/redirect の役割と責務の境界を説明し、自分で実装できる
- ERB テンプレートで View を分離し、エスケープで XSS を防ぐテストを書ける
- query・path・form の3種類のパラメータを `params` にまとめ、`request` との違いを説明できる
- before_action のような共通処理の仕組みを作り、Rack ミドルウェアとの違いを説明できる

## この章の地図

この章は長い章です。元の教材では7つのフェーズに分かれていた内容を、1つにまとめています。進む順番は次のとおりです。

1. **素の Rack の限界**：あえて Rack だけでアプリを書き、つらさを集める
2. **Router**：URL とメソッドで行き先を振り分ける
3. **Controller / Action**：関連する処理をクラスにまとめる
4. **render / redirect**：Action が Rack の配列を直接書かなくて済むようにする
5. **View と XSS 対策**：HTML をテンプレートに追い出し、危険な文字を無害化する
6. **Request / Params**：バラバラな入力を1つの窓口にまとめる
7. **before_action**：Action の前に共通処理を差し込む

どの段階も、「つらさを感じる → テストで欲しい振る舞いを書く → 最小の実装で通す → 整える」の繰り返しです。完成形を先に作ろうとしないでください。完成形は、テストを積み上げた結果として自然にできあがります。

この章を終えると、Rails の `routes.rb`、`ApplicationController`、`render`、`redirect_to`、`params`、`before_action` が、魔法ではなく「自分も書いたもの」に見えるはずです。これが元教材の言う「フレームワークを読む側になる」ということです。

ここは長くて難しい章です。1日で終わらせようとせず、1段階ずつ区切って進めてください。各段階の終わりにコミットしておくと、迷ったときに戻れます。

## 準備：作業フォルダとテストの土台

作業フォルダは `~/workspace/ch14-framework/` です。

```bash
mkdir -p ~/workspace/ch14-framework/{lib,test,views}
cd ~/workspace/ch14-framework
git init
```

`Gemfile` は第13章と同じ内容で構いません（`rack`、`rackup`、`webrick`、`rake`、`minitest`）。`Rakefile` も第13章と同じです。

テストの土台 `test/test_helper.rb` を次のように用意します。これは演習の解答ではなく「テストを書くための道具」なので、そのまま使って構いません。

```ruby
# test/test_helper.rb
require "minitest/autorun"
require "rack"
require "rack/mock"

$LOAD_PATH.unshift(File.expand_path("../lib", __dir__))

module EnvHelper
  # テスト用の env を作る。サーバを起動せずにアプリを呼び出せる
  def env_for(path, method: "GET", params: nil)
    Rack::MockRequest.env_for(path, method: method, params: params)
  end
end
```

`env_for("/users?page=2")` と書けば、`GET /users?page=2` のリクエストを受けたときと同じ `env` ができます。`method: "POST", params: { "name" => "Alice" }` を付ければ、フォームから POST したときの `env` になります。

この章のテストは、すべてサーバを起動せずに動きます。ブラウザで確かめたいときだけ、第13章と同じく `rackup -o 127.0.0.1` で起動し、確認後すぐに止めます。

## 素の Rack アプリの限界

### 作るもの

最初に、フレームワークを使わずに「ユーザー一覧アプリ」を作ります。Router も ERB も使いません。使ってよいのは Rack と Ruby の標準機能だけです。

| メソッドとパス | 動作 |
|---|---|
| `GET /` | トップページの HTML を返す |
| `GET /users` | ユーザーの一覧を返す（データは配列でよい） |
| `GET /users/:id` | 指定した ID のユーザーを返す |
| `POST /users` | フォームの値を受け取り、配列に追加する |

`:id` は「ここに何かの値が入る」という意味の書き方です。`/users/1` や `/users/42` などに当てはまります。

### 書き始めるとこうなる

最初はこんな形から始まるでしょう。

```ruby
app = Proc.new do |env|
  case env["PATH_INFO"]
  when "/"
    [200, { "content-type" => "text/html" }, ["<h1>Home</h1>"]]
  when "/users"
    # 一覧の HTML を組み立てる……
  end
end
```

ここに `/users/:id` を足そうとすると、`when` では書けないことに気づきます。正規表現を使うことになります。`POST /users` を足すと、今度はメソッドでも分岐が必要です。HTML は文字列の連結だらけになります。

### 観察するポイント

この段階の目的は、きれいに作ることではありません。**つらさを集めること**です。書きながら、次の点をメモしてください。

- **分岐が増え続ける**：パスとメソッドの組み合わせごとに `if` や `when` が増え、見通しが悪くなる
- **HTML がコードに埋まる**：文字列連結で HTML を作るので、見た目の変更がつらい
- **同じ処理があちこちにある**：`content-type` の指定や、配列の組み立てが毎回出てくる
- **どこに何を書くか決まっていない**：新しい機能を足すとき、どこに書けばよいか迷う

このメモが、この後の段階すべての「動機」になります。Router は1つ目を、View は2つ目を、render は3つ目を、Controller は4つ目を解決します。

フレームワークとは、こうした「誰が書いても同じように出てくるつらさ」をまとめて引き受けてくれる仕組みです。そして同時に、「アプリのコードをどこに、どう書くか」を決めてくれるものです。

## TDD のリズムをもう一度

この先は、すべて第6章で学んだ TDD で進めます。

- **RED**：まだ存在しない振る舞いのテストを書き、**失敗することを確かめる**
- **GREEN**：テストを通す**最小の**コードを書く。きれいさより、まず通すこと
- **REFACTOR**：テストを通したまま、名前や重複を整える

「失敗を確かめる」を飛ばしてはいけません。最初から通るテストは、何も確かめていない可能性があるからです。

もう1つ大事なルールがあります。**テストを書き換えて通すのは禁止**です。テストは仕様です。仕様を変えるなら、それは意図した判断として行い、理由をメモに残します。

## Router：行き先を振り分ける

### 最初の失敗テスト

Router は「メソッドとパスの組み合わせ」と「呼び出す処理」の対応表です。まずは一番小さな振る舞いをテストにします。

```ruby
# test/router_test.rb
require_relative "test_helper"
require "router"

class RouterTest < Minitest::Test
  include EnvHelper

  def test_get_root_returns_200
    router = Router.new
    router.get "/" do |_req|
      [200, {}, ["OK"]]
    end

    status, _headers, body = router.call(env_for("/"))

    assert_equal 200, status
    assert_equal ["OK"], body
  end
end
```

このテストは、作りたい Router の使い方を表しています。

- `Router.new` で作る
- `get "/"` とブロックで、ルートを登録する
- Router 自身が `call(env)` を持つ。つまり **Router は Rack アプリである**

`lib/router.rb` を空のファイルで用意し、実行します。

```bash
bundle exec ruby test/router_test.rb
```

```
  1) Error:
RouterTest#test_get_root_returns_200:
NameError: uninitialized constant RouterTest::Router
    test/router_test.rb:8:in `test_get_root_returns_200'

1 runs, 0 assertions, 0 failures, 1 errors, 0 skips
```

`Router` がまだないので失敗しました。これが RED です。この失敗は「予想どおりの理由で失敗している」ことが大切です。違う理由で失敗していたら、テストの書き方を見直します。

### 最小の実装で通す

GREEN の段階では、汎用的に作ろうとしないでください。ルートが1つしかないなら、1つだけ覚えておけば通ります。不格好でも構いません。「テストが要求していないことは書かない」が原則です。

### テストを1つずつ増やす

次のテストを順番に1つずつ足し、そのたびに RED → GREEN を回します。

```ruby
def test_get_users
  router = Router.new
  router.get("/") { |_req| [200, {}, ["root"]] }
  router.get("/users") { |_req| [200, {}, ["users"]] }

  _status, _headers, body = router.call(env_for("/users"))

  assert_equal ["users"], body
end
```

ルートが2つになったので、「1つだけ覚える」実装では通りません。複数を覚える仕組みが必要になります。ここで初めて、配列や Hash でルートを持つ形に変えます。

```ruby
def test_post_users
  router = Router.new
  router.get("/users") { |_req| [200, {}, ["list"]] }
  router.post("/users") { |_req| [200, {}, ["created"]] }

  _status, _headers, body = router.call(env_for("/users", method: "POST"))

  assert_equal ["created"], body
end
```

同じパスでも、メソッドが違えば別の行き先です。パスだけで探す実装は、このテストで失敗します。

```ruby
def test_returns_404_when_not_found
  router = Router.new

  status, _headers, _body = router.call(env_for("/unknown"))

  assert_equal 404, status
end
```

どのルートにも当たらないときの振る舞いを決めます。何も返さないと、Rack の約束を破ってしまいます。

### 動的なルート

```ruby
def test_dynamic_route
  router = Router.new
  router.get "/users/:id" do |req|
    [200, {}, [req.path_params["id"]]]
  end

  _status, _headers, body = router.call(env_for("/users/42"))

  assert_equal ["42"], body
end
```

`/users/:id` というパターンが `/users/42` に当たり、`id` が `"42"` として取り出せることを求めています。ここで決めているのは、ブロックに渡す `req` が `path_params` というメソッドを持つ、という使い方です。

実装の考え方としては、パターンを正規表現に変換するのが定番です。正規表現は「文字列のパターン」を表す小さな言語です。Ruby ではこう書けます。

```ruby
m = "/users/42".match(%r{\A/users/(?<id>[^/]+)\z})
p m[:id]             # => "42"
p m.named_captures   # => {"id"=>"42"}
```

- `\A` と `\z`：文字列の先頭と末尾
- `[^/]+`：スラッシュ以外の文字が1文字以上
- `(?<id>...)`：当たった部分に `id` という名前を付ける

`"/users/:id"` という文字列から、この正規表現をどう組み立てるかは、自分で考えてください。もう1つ、テストを足すと安心です。`/users/42/edit` は `/users/:id` に**当たってはいけない**、というテストです。先頭と末尾の指定を忘れると、ここで気づけます。

### REFACTOR

ここまで通ったら整理します。たとえば次のような分け方が考えられます。

- ルートを登録する処理（`get` と `post` の共通部分）
- パターンを正規表現に変換する処理
- リクエストに当たるルートを探す処理

テストを1行も変えずに、中身だけを整えます。整えるたびにテストを実行してください。テストが緑のまま形を変えられるのが、TDD の一番の恩恵です。

## Controller と Action：処理をクラスにまとめる

### なぜ Proc のままではいけないのか

Router の行き先は、今はブロック（Proc）です。小さいうちはこれで十分です。しかし数十のルートができると、困ったことが起きます。

- ユーザー関係の処理（一覧・詳細・作成）が、ばらばらのブロックに散らばる
- 共通の処理（パラメータの取り出しなど）を置く場所がない
- どこにアプリのコードを書くか、人によって変わる

そこで「関連する処理をクラスにまとめ、1つ1つの処理をメソッドにする」という形を導入します。このクラスを **Controller**、メソッドを **Action** と呼びます。

### Controller の Action を呼ぶテスト

```ruby
def test_routes_to_controller_action
  router = Router.new
  router.get "/users", to: UsersController.action(:index)

  status, _, body = router.call(env_for("/users"))

  assert_equal 200, status
  assert_equal ["Users Index"], body
end
```

テストのファイルに、試験用の Controller も書いておきます。

```ruby
class UsersController
  def index
    [200, {}, ["Users Index"]]
  end
end
```

`UsersController.action(:index)` は、「Router から呼べる形の何か」を返すクラスメソッドです。Router はブロックを呼んでいたので、同じように `call` できるものを返せば、Router をほとんど変えずに済みます。

### "users#index" という書き方

Rails の `routes.rb` では、`to: "users#index"` と書きます。これもテストにします。

```ruby
def test_routes_with_controller_action_string
  router = Router.new
  router.get "/users", to: "users#index"

  status, _, body = router.call(env_for("/users"))

  assert_equal 200, status
  assert_equal ["Users Index"], body
end
```

文字列からクラスを見つけるには、`Object.const_get` が使えます。

```ruby
name, action = "users#index".split("#")
p "#{name.capitalize}Controller"                 # => "UsersController"
p Object.const_get("#{name.capitalize}Controller") # => UsersController
```

存在しないクラス名を渡すと、`NameError: uninitialized constant PostsController` のような例外になります。この場合に例外のままにするか、404 にするかも、テストで決めてください。また `"user_profiles".capitalize` は `"User_profiles"` になります。複数語の名前への対応は、必要になってから考えれば十分です。

### params を Action から使いたい

```ruby
def test_controller_can_access_path_params
  router = Router.new
  router.get "/users/:id", to: "users#show"

  status, _, body = router.call(env_for("/users/42"))

  assert_equal 200, status
  assert_equal ["42"], body
end
```

```ruby
class UsersController
  def show
    [200, {}, [params["id"]]]
  end
end
```

Action の中で `params` を使いたい。そのためには、Controller を作るときにリクエストの情報を渡す必要があります。`initialize(env)` にするか、Router のリクエストオブジェクトを渡すか、自分で決めてください。

### ControllerBase が欲しくなる

ここで、`PostsController` など2つ目の Controller を作ってみてください。`initialize` と `params` を、また同じように書くことになるはずです。

この重複が見えたら、**共通の親クラス** `ControllerBase` を作ります。各 Controller は `class UsersController < ControllerBase` と継承するだけで、`params` などが使えるようにします。

重複が見える前に親クラスを作らないでください。「なぜ必要か」が分からないまま作った抽象は、たいてい間違った形になります。

ここで1つ、よくあるエラーに注意してください。前のテストで `class UsersController`（親クラスなし）と書いたまま、別の場所で `class UsersController < ControllerBase` と書くと、`superclass mismatch for class UsersController` というエラーになります。Ruby では、一度決まったクラスの親を後から変えられないからです。ControllerBase を導入したら、テスト用の Controller の定義を1か所にまとめ、すべて `< ControllerBase` に書き換えます。これはテストの「仕様」ではなく「道具」の変更なので、書き換えて構いません。

### Router と Controller の責務

整理の目安は、**Router は Controller の中身を知りすぎない**ことです。Router の仕事は「どの Controller のどの Action か」を決めて呼ぶところまでです。Controller の作り方や Action の呼び方の細かい手順は、Controller 側（たとえば `action` クラスメソッドや `dispatch` メソッド）に閉じ込めます。

## render と redirect：Action に HTTP を書かせない

### Action が配列を返すつらさ

今の Action は、`[200, { "content-type" => "text/html" }, ["..."]]` のような配列を直接返しています。これには問題があります。

- どの Action でも同じ形の配列を書く
- 「何を返したいか」より「HTTP でどう返すか」に気を取られる
- リダイレクトのように、HTTP の知識が必要な表現が書きにくい

そこで、Action は `render_text "Hello"` のように**意味**で書き、HTTP の配列は ControllerBase が組み立てるようにします。

### render_text のテスト

```ruby
class PagesController < ControllerBase
  def hello
    render_text "Hello"
    nil # 戻り値を使わないことを確かめるため、わざと nil を返す
  end
end
```

```ruby
def test_action_does_not_need_to_return_rack_response
  controller = PagesController.new(env_for("/hello"))

  status, headers, body = controller.dispatch(:hello)

  assert_equal 200, status
  assert_equal "text/plain", headers["content-type"]
  assert_equal ["Hello"], body
end
```

Action の戻り値は `nil` です。それでも `dispatch` はレスポンスを返します。つまり `render_text` は、レスポンスを「Controller の中に覚えておく」働きをし、`dispatch` が Action の実行後にそれを取り出す、という形になります。ヘッダ名は第13章と同じく小文字の `content-type` です。

同じ流れで、次の振る舞いを1つずつテストにします。

- `render_html "<h1>Hello</h1>"` で `content-type` が `text/html` になる
- `render_text "Not Found", status: 404` でステータスを指定できる。指定がなければ 200

### redirect_to のテスト

```ruby
class UsersController < ControllerBase
  def create
    redirect_to "/users"
  end
end
```

```ruby
def test_action_can_redirect
  router = Router.new
  router.post "/users", to: "users#create"

  status, headers, body = router.call(env_for("/users", method: "POST"))

  assert_equal 302, status
  assert_equal "/users", headers["location"]
  assert_equal [], body
end
```

リダイレクトは「本文を返す」のではなく、「次はこの URL に行ってください」と**移動先を通知する**レスポンスです。中身はステータス 302 と `location` ヘッダだけです。ブラウザはこれを受け取ると、自動で `location` の URL に GET でアクセスし直します。

フォームを POST した後にリダイレクトするのには理由があります。POST の結果を直接表示すると、ブラウザの再読み込みで同じ POST がもう一度送られ、データが二重に登録されることがあります。リダイレクトで GET の画面に移れば、再読み込みしても GET が繰り返されるだけです。この形は PRG（Post/Redirect/Get）と呼ばれます。

`redirect_to "/new_users", status: 301` のように、ステータスを変えられるようにもします。301 は「恒久的に移動した」、302 は「一時的に別の場所を見て」という意味です。

### 二重 render を防ぐ

```ruby
class DoubleRenderController < ControllerBase
  def index
    render_text "A"
    render_text "B"
  end
end
```

1つの Action で2回 render したら、どちらを返すべきでしょう。どちらを選んでも、書いた人の意図と違う可能性があります。そこで「2回目は例外にする」と決め、テストで固定します。

```ruby
def test_raises_error_when_render_called_twice
  controller = DoubleRenderController.new(env_for("/double"))

  assert_raises(RuntimeError) { controller.dispatch(:index) }
end
```

専用の例外クラスを作るなら、`RuntimeError` の代わりにそのクラスを書きます。Rails にも `AbstractController::DoubleRenderError` という同じ目的の例外があります。

### 何も render しなかったら

Action が render も redirect もしなかったときの振る舞いも、先に決めます。候補は「例外にする」「空のレスポンスにする」「204 No Content にする」などです。**正解は1つではありません**。大事なのは、選んだ理由を説明でき、テストで固定することです。

### REFACTOR

ここまで来ると、`render_text`、`render_html`、`redirect_to` の中に同じ処理が見えてきます。ステータスの設定、ヘッダの設定、二重 render の判定などです。共通部分を private メソッドにまとめましょう。目標は次の状態です。

- Action は「何を返したいか」だけを書く
- ControllerBase は「HTTP のレスポンスをどう組み立てるか」を引き受ける

## View：HTML をテンプレートに追い出す

### HTML を Action に書くつらさ

`render_html "<h1>Users</h1><ul><li>Alice</li><li>Bob</li></ul>"` のような Action は、すぐに読めなくなります。見た目を変えたいだけなのに、Ruby のコードを触ることになります。そこで HTML を別ファイル（**テンプレート**）に分け、Controller は「表示に必要なデータを用意する」ことに専念させます。

進め方は2段階です。いきなり ERB を使わず、まず**静的な HTML ファイルを読み込んで返す**ところから始めます。`render_file "pages/home.html"` のようなメソッドをテストから作ります。その次に ERB に進みます。一度に2つの新しいことをしないのが、TDD で迷わないコツです。

テストの中でテンプレートファイルを作るときは、テスト専用の一時フォルダを使うと、テスト同士が影響しません。Ruby 標準の `Dir.mktmpdir` が使えます。テンプレートを探すフォルダを ControllerBase の設定として差し替えられるようにすると、テストしやすくなります。

### ERB の基本

**ERB** は、Ruby 標準のテンプレートエンジンです。HTML の中に Ruby を埋め込めます。

```ruby
require "erb"

template = ERB.new("<p>1 + 2 = <%= 1 + 2 %></p><% if true %><p>表示される</p><% end %>")
puts template.result
```

```
<p>1 + 2 = 3</p><p>表示される</p>
```

- `<%= 式 %>`：式の結果を、その場所に出力する
- `<% 文 %>`：Ruby を実行するが、何も出力しない（`if` や `each` に使う）

### View から @変数 が見える理由

Controller で `@title = "Users"` と書き、テンプレートで `<%= @title %>` と書けば表示される。Rails でおなじみのこの動きは、`binding` という仕組みで実現します。

```ruby
require "erb"

class Greeter
  def initialize
    @name = "Alice"
  end

  def html
    ERB.new("<h1>Hello <%= @name %></h1>").result(binding)
  end
end

puts Greeter.new.html
```

```
<h1>Hello Alice</h1>
```

`binding` は「今この場所から見えている変数やメソッドの一式」を表すオブジェクトです。`result(binding)` は「この場所の視点でテンプレートを評価して」という意味になります。`html` メソッドの中から評価したので、`@name`（この Greeter のインスタンス変数）が見えたのです。

つまり、Controller のメソッドの中で `result(binding)` を呼べば、テンプレートからその Controller の `@変数` が見えます。値は Controller のインスタンスに保存されていて、テンプレートはそれを覗いているだけです。

### 改行と trim_mode

ERB の `<% %>` の行は、何も出力しませんが、行末の改行は残ります。

```ruby
@users = ["Alice", "Bob"]
t = "<ul>\n  <% @users.each do |user| %>\n    <li><%= user %></li>\n  <% end %>\n</ul>\n"
p ERB.new(t).result(binding)
```

```
"<ul>\n  \n    <li>Alice</li>\n  \n    <li>Bob</li>\n  \n</ul>\n"
```

空行が混ざりました。`trim_mode: "-"` を指定し、`-%>` で閉じると、その後ろの改行を取り除けます。

```ruby
t2 = "<ul>\n<% @users.each do |user| -%>\n  <li><%= user %></li>\n<% end -%>\n</ul>\n"
p ERB.new(t2, trim_mode: "-").result(binding)
```

```
"<ul>\n  <li>Alice</li>\n  <li>Bob</li>\n</ul>\n"
```

テストで HTML を完全一致で比べると、こうした空白の違いで失敗しやすくなります。`assert_includes` で「含まれていること」を確かめるか、空白を正規化してから比べるとよいでしょう。どちらにしたか、理由を説明できるようにしてください。なお `trim_mode:` のキーワード指定は Ruby 2.6 以降で使えます。Ruby 3.x では古い書き方（2番目以降の引数で指定）は使えません。

### テンプレートを探すルール

`render_view "users/index"` と書いたら、どのファイルを読むでしょう。たとえば「`views/` の下の `users/index.erb`」というルールが考えられます。このルールを1か所に閉じ込めれば、Controller はファイルの場所を知らずに済みます。Rails が `app/views/users/index.html.erb` を自動で探すのも、同じ考え方です。

存在しないテンプレートを指定したときの振る舞いも、テストで決めます。

### XSS：エスケープしないと攻撃が通る

ここはこの章で一番大事な安全の話です。

**Ruby 標準の ERB は、`<%= %>` の値を自動でエスケープしません。** エスケープとは、HTML で特別な意味を持つ文字（`<`、`>`、`&`、`"`、`'`）を、意味を持たない表記に置き換えることです。

まず、攻撃が通ることを自分の目で確かめます。次の実験用アプリは、**わざと脆弱に**作ってあります。

```ruby
# xss.ru（わざと脆弱にした実験用アプリ。127.0.0.1 でだけ動かし、確認したらすぐ止める）
require "erb"

app = Proc.new do |env|
  req = Rack::Request.new(env)
  comment = req.params["comment"].to_s
  html = ERB.new("<p>コメント: <%= comment %></p>").result(binding)
  [200, { "content-type" => "text/html; charset=utf-8" }, [html]]
end

run app
```

```bash
bundle exec rackup -s webrick -o 127.0.0.1 -p 3000 xss.ru
```

別のターミナルで `lsof -nP -iTCP:3000 -sTCP:LISTEN` を実行し、`127.0.0.1:3000` であることを確かめます。それから curl で、`<script>` を含むコメントを送ります（URL の中では `<` を `%3C` のように書きます）。

```bash
curl -s 'http://127.0.0.1:3000/?comment=%3Cscript%3Ealert(%22XSS%22)%3C%2Fscript%3E'
```

```
<p>コメント: <script>alert("XSS")</script></p>
```

入力した文字列が、そのまま HTML の `<script>` タグになって返ってきました。この URL をブラウザで開くと、アラートが表示されます。つまり、**入力した人が書いた JavaScript が、ページを見た人のブラウザで実行された**のです。

これが **XSS（クロスサイトスクリプティング）** です。今回は `alert` だけですが、本物の攻撃では、ログイン中の利用者のクッキーを盗んだり、本人になりすまして操作したりする JavaScript が仕込まれます。掲示板の投稿のように、保存された入力を他人が見る画面では、被害が全員に広がります。

確認したら、すぐ `Ctrl-C` でサーバを止めてください。このファイルは脆弱性の実験用です。演習のアプリにこの書き方を持ち込んではいけません。

### エスケープで防ぐ

対策は、テンプレートに値を埋め込むときにエスケープすることです。Ruby には `ERB::Util.html_escape` があります。

```ruby
require "erb"

comment = "<script>alert('XSS')</script>"

unsafe = ERB.new("<p><%= comment %></p>").result(binding)
safe   = ERB.new("<p><%= ERB::Util.html_escape(comment) %></p>").result(binding)

puts unsafe
puts safe
```

```
<p><script>alert('XSS')</script></p>
<p>&lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;</p>
```

エスケープ後の `&lt;` は、ブラウザ上では `<` という**文字**として表示されます。タグとしては解釈されません。だから、利用者には `<script>alert('XSS')</script>` という文字列がそのまま見え、実行はされません。

`ERB::Util` を `include` すると、短い名前 `h` で呼べます。

```ruby
include ERB::Util
puts h(%q{Tom & "Jerry" <b> 'x'})
```

```
Tom &amp; &quot;Jerry&quot; &lt;b&gt; &#39;x&#39;
```

この章では、テンプレートで値を出すときは必ず `<%= h(値) %>` と書く、というルールにします。ControllerBase で `h` を使えるようにするのは、自分で実装します。

### XSS 対策の完了条件

エスケープは「気を付ける」ではなく、**テストで確かめる**ものです。次のようなテストが、この段階の完了条件です。

```ruby
def test_view_escapes_html_in_user_input
  router = Router.new
  router.post "/comments", to: "users#comment"
  env = env_for("/comments", method: "POST",
                params: { "comment" => "<script>alert(1)</script>" })

  _, headers, body = router.call(env)
  html = body.join

  assert_equal "text/html", headers["content-type"]
  refute_includes html, "<script>"
  assert_includes html, "&lt;script&gt;alert(1)&lt;/script&gt;"
end
```

`refute_includes` は「含まれていないこと」を確かめるアサーションです。攻撃の文字列がそのままの形で出ていないこと、エスケープされた形で出ていることを、両方確かめています。

属性値に埋め込む場合も確かめます。`<input value="<%= h(@comment) %>">` のように、**属性値は必ず引用符で囲みます**。そのうえで、引用符を含む入力 `" onmouseover="alert(1)` が属性を抜け出せないことをテストします。エスケープ後は `&quot; onmouseover=&quot;alert(1)` になり、属性の中の文字列のままになります。

注意点が2つあります。

- **HTML エスケープですべてが安全になるわけではありません。** `<script>` の中、`onclick` などのイベント属性の中、`href` の URL では、別の対策が必要です。この章では、入力をそうした場所に埋め込まないことにします
- **保存済みのデータも信頼しません。** 第15章以降でデータベースに保存した値を表示するときも、同じようにエスケープします。「自分のDBから来た値だから安全」ではありません。最初に誰が入力したかは分からないからです

Rails のテンプレートは、`<%= %>` を**自動で**エスケープします。それでも `raw` や `html_safe` を使うとエスケープが外れます。自作した経験があれば、それがどれほど危険な操作か分かるはずです。

### レイアウトが欲しくなる

画面を3つほど作ると、どのテンプレートにも `<html>`、`<head>`、`<body>`、共通のヘッダが出てくることに気づきます。「各画面の外側に共通の枠が欲しい」という気づきが、**レイアウト**という仕組みの種です。この章では必要性に気づくところまでで構いません。挑戦したい人は演習の発展課題に進んでください。

## Request と Params：入力の窓口を整える

### 入力は3か所から来る

Action が受け取りたい入力は、3つの経路から来ます。

| 種類 | 例 | どこにあるか |
|---|---|---|
| query parameter | `/search?keyword=ruby` | URL の `?` の後ろ（`QUERY_STRING`） |
| path parameter | `/users/42` の `42` | URL のパスの一部。Router が取り出す |
| form parameter | フォームの `name=Alice` | POST のリクエスト本文（`rack.input`） |

Action を書く人にとって、値がどこから来たかはたいてい重要ではありません。欲しいのは「`id` の値」や「`name` の値」です。そこで、3つをまとめて `params["id"]` のように読める窓口を作ります。

### request と params を分ける

一方で、入力値以外の情報も必要です。メソッドが GET か POST か、パスは何か、ヘッダに何が入っているか。これらは **request** という窓口にまとめます。

- `request`：リクエスト全体の窓口（メソッド、パス、ヘッダ、クッキー）
- `params`：入力値の窓口（query・path・form をまとめたもの）

Rack には `Rack::Request` という便利なクラスがあり、`env` を包んで読みやすくしてくれます。最初はこれをそのまま `request` として使って構いません。

```ruby
def test_controller_can_access_request
  router = Router.new
  router.post "/info", to: "pages#info"

  _, _, body = router.call(env_for("/info", method: "POST"))

  assert_equal ["POST"], body
end
```

```ruby
class PagesController < ControllerBase
  def info
    render_text request.request_method
  end
end
```

### params をテストで積み上げる

次の順で、1つずつテストにします。

1. query parameter：`env_for("/search?keyword=ruby")` で `params["keyword"]` が `"ruby"`
2. path parameter：`/users/:id` に `/users/42` で `params["id"]` が `"42"`
3. 両方：`/users/42?page=3` で `id` と `page` の両方が取れる
4. form parameter：`env_for("/users", method: "POST", params: { "name" => "Alice" })` で `params["name"]` が `"Alice"`

3番目のテストの例です。

```ruby
def test_params_contains_both_path_and_query_params
  router = Router.new
  router.get "/users/:id", to: "users#show"

  _, _, body = router.call(env_for("/users/42?page=3"))

  assert_equal ["id=42,page=3"], body
end
```

```ruby
class UsersController < ControllerBase
  def show
    render_text "id=#{params["id"]},page=#{params["page"]}"
  end
end
```

### 同じ名前がぶつかったら

path に `id=42`、query に `id=99` が来たら、`params["id"]` はどちらでしょう。`/users/42?id=99` というリクエストです。

これも**正解は1つではありません**。ただし、決めずに放置するのが一番危険です。たとえば「URL のパスで指定した対象を操作する画面」で、query の `id` が勝つと、利用者が見ている画面と違う対象が操作されるかもしれません。ルールを決め、テストで固定してください。Rails では path parameter が優先されます。

### キーの形もテストで決める

`params["keyword"]` と `params[:keyword]` の両方で読めるようにするか、文字列キーだけにするか。これも使い方のルールです。たとえば「文字列キーだけ。シンボルでは `nil`」と決めたなら、それをテストで固定します。

```ruby
class PagesController < ControllerBase
  def search
    render_text "#{params["keyword"]}|#{params[:keyword]}"
  end
end
```

```ruby
def test_params_are_accessed_with_string_keys
  router = Router.new
  router.get "/search", to: "pages#search"

  _, _, body = router.call(env_for("/search?keyword=ruby"))

  assert_equal ["ruby|"], body
end
```

### なぜ ControllerBase に params を置くのか

`request.params["keyword"]` と書けば、Rack::Request だけで query と form は読めます。それでも `params` を ControllerBase に置くのは、次の理由からです。

- path parameter は Router が取り出したもので、Rack::Request は知らない。統合する場所が要る
- Action からよく使うものは、短い名前で近くに置きたい
- 将来、独自の Request クラスに替えても、Action のコードを変えずに済む

## before_action：Action の前に共通処理を差し込む

### 共通処理が Action に混ざる

たとえば「ログインしていなければログイン画面にリダイレクトする」処理を考えます。これを各 Action の先頭に書くと、次の問題が起きます。

- 同じ処理を何度も書く
- 書き忘れた Action だけ、ログインなしで使えてしまう
- Action の本来の仕事が読みにくくなる

特に2つ目は、セキュリティの穴になります。共通処理は「書き忘れられない場所」に置くべきです。そこで、Action の前に決まった処理を走らせる **before_action** を作ります。

### テストの書き方の注意

before_action は、**クラスに**登録します。`UsersController.before_action { ... }` と書くと、その設定はクラスに残ります。テストの中で `UsersController` に登録すると、**他のテストにもその設定が残ってしまいます**。テストの実行順によって結果が変わる、不安定なテストになります。

これを避けるには、テストごとに新しい Controller クラスを作ります。Ruby では `Class.new(親クラス) do ... end` で、名前のないクラスをその場で作れます。

```ruby
class BeforeActionTest < Minitest::Test
  include EnvHelper

  def build_controller_class
    Class.new(ControllerBase) do
      def index
        render_text "index"
      end

      def show
        render_text "show"
      end
    end
  end

  def test_before_action_runs_before_action_method
    calls = []
    klass = build_controller_class
    klass.before_action { calls << "before" }

    klass.new(env_for("/")).dispatch(:index)

    assert_equal ["before"], calls
  end
end
```

`calls` という配列に記録し、「本当に呼ばれたか」を確かめています。

### 積み上げる振る舞い

次の順でテストを足していきます。

1. **複数の登録**：`before_action { calls << 1 }` と `before_action { calls << 2 }` を登録すると、`[1, 2]` の順で呼ばれる
2. **only**：`before_action(only: [:index]) { ... }` は、`index` のときだけ呼ばれ、`show` では呼ばれない
3. **except**：`before_action(except: [:show]) { ... }` は、`show` 以外で呼ばれる
4. **中断**：before_action の中で render や redirect をしたら、Action 本体は実行されない
5. **メソッド名で指定**：`before_action :require_login` のように、シンボルで private メソッドを指定できる
6. **継承**：親クラスで登録した before_action は、子クラスでも動く。子で登録したものは親には影響しない

4番目の中断のテストは、たとえばこう書けます。

```ruby
def test_before_action_can_halt_execution
  klass = build_controller_class
  klass.before_action { render_text "blocked", status: 403 }

  status, _, body = klass.new(env_for("/")).dispatch(:index)

  assert_equal 403, status
  assert_equal ["blocked"], body
end
```

ブロックの中で `render_text` を呼べるということは、ブロックが **Controller のインスタンスの中で**実行される必要があります。Ruby の `instance_exec` を調べてみてください。

中断の仕組みは、ログイン確認のような処理の要です。「ログインしていなければリダイレクトして、Action は実行しない」。これがテストで保証されていれば、Action の書き忘れによる穴を防げます。

6番目の継承は、Rails の `ApplicationController` に書いた before_action が全 Controller で動くのと同じ仕組みです。子クラスに登録したものが親に漏れないことも、忘れずにテストしてください。クラスのインスタンス変数と継承の関係は、ここで一番つまずきやすいところです。

### Rack ミドルウェアとの違い

第13章のミドルウェアも「前後に処理を挟む」仕組みでした。何が違うのでしょう。

| | Rack ミドルウェア | before_action |
|---|---|---|
| 位置 | Router より手前。アプリ全体を包む | Controller の中。Action の直前 |
| 知っていること | env（リクエストの生の情報）だけ | どの Controller の、どの Action か。params も使える |
| 向いている処理 | 全リクエスト共通のもの（ログ、計測、圧縮） | Action によって変わるもの（ログイン確認、対象データの読み込み） |

どちらに書くべきかは、「その処理は、どの Action かを知る必要があるか」で判断するとよいでしょう。

## 自作と Rails の対応表

ここまで作ったものと、Rails の部品を並べます。第18章で Rails に移るとき、この表に戻ってきてください。

| 自作したもの | Rails での名前 |
|---|---|
| `Router#get "/users/:id", to: "users#show"` | `config/routes.rb` の `get "/users/:id", to: "users#show"` |
| `ControllerBase` | `ActionController::Base`（と `ApplicationController`） |
| `dispatch(:index)` | Action を呼び出す内部の仕組み |
| `render_text` / `render_html` | `render plain:` / `render html:` |
| `render_view "users/index"` | `render :index`（省略すると自動で探す） |
| `redirect_to "/users"` | `redirect_to users_path` |
| 二重 render の例外 | `AbstractController::DoubleRenderError` |
| `h(値)` | 自動エスケープ（`<%= %>`）。外すのは `raw` / `html_safe` |
| `params` | `params`（`ActionController::Parameters`） |
| `before_action` | `before_action` |

## 実務ではこうなる

**Rails のコードを「読める」ようになります。** 実務では、Rails の挙動で困ったとき、Rails 自体のソースコードを読んで原因を探すことがあります。`before_action` が動かない、params の値が想像と違う、render が2回呼ばれたと言われる。どれも、この章で自分が作った仕組みと同じ形の問題です。

**XSS の穴は「自動エスケープを外したところ」に開きます。** Rails は自動でエスケープするので、普通に書けば安全です。事故が起きるのは、`html_safe` や `raw` を使ったとき、JavaScript の中に値を埋め込んだとき、`href` にユーザー入力の URL を入れたときです。コードレビューでは、これらを見つけたら必ず理由を確認します。

**params の優先順位や型で事故が起きます。** 「URL の ID と、フォームの hidden の ID が違う」ときにどちらが使われるか。実務のバグ調査でよく出てくる問いです。

**ジュニアがやりがちな失敗：**

- Controller に処理を詰め込みすぎる（いわゆる Fat Controller）。HTML の組み立てや計算まで Action に書いてしまう
- ログイン確認を各 Action に手で書き、1か所だけ書き忘れる
- テストで、クラスに登録した設定（before_action など）を後片付けせず、他のテストを壊す
- 「表示が崩れるから」という理由で `html_safe` を付けて、XSS の穴を開ける
- POST の後にリダイレクトせず、再読み込みで二重登録が起きる

## 演習

演習はすべて `~/workspace/ch14-framework/` で行います。各演習の終わりにコミットしてください。すべての演習で、RED（失敗の確認）→ GREEN → REFACTOR の順を守ります。テストを書く前に実装を書いた場合は、その実装を消してやり直してください。

### 演習14-1：素の Rack でユーザー一覧アプリを作る

- Router や ERB を使わず、`config.ru` と Rack だけで、本文の表にある4つの機能を実装する
- データは配列で持つ。名前は `Alice` `Bob` のようなダミーにする
- `bundle exec rackup -s webrick -o 127.0.0.1 -p 3000 config.ru` で起動し、`lsof` で待受先を確かめてから、curl で4つの機能を試す。終わったら止める
- `notes/raw-rack-pain.md` に「書いていてつらかった点」を5つ以上、「将来問題になりそうな点」を3つ以上書く
- **完了条件**：
  - `curl -i http://127.0.0.1:3000/users/1` が 200 と1人分のユーザー情報を返し、`curl -i http://127.0.0.1:3000/users/999` が 404 を返す
  - `curl -i -X POST -d 'name=Carol' http://127.0.0.1:3000/users` の後、`/users` に Carol が表示される
  - メモに「なぜフレームワークが必要か」を、自分のつらさを根拠に3行以上で書いている
- ヒント：POST の本文は `Rack::Request.new(env).params` で読めます。ここではテストを書かなくても構いません。この演習は「つらさ」を集めることが目的です

### 演習14-2：Router をTDDで作る

- 本文の Router のテストを、1つずつ書いては RED → GREEN → REFACTOR を回す
- 必ず含めるテスト：静的なルート／複数のルート／メソッドによる分岐／404／動的なルート（`/users/:id`）／深いパスに当たらないこと（`/users/42/edit`）
- 各テストを足すたびに、RED のときの失敗メッセージを `notes/router-tdd.md` に1行ずつ記録する
- **完了条件**：
  - `bundle exec rake test` で Router のテストが6個以上あり、失敗0・エラー0
  - `notes/router-tdd.md` に、6つのテストそれぞれの RED の理由と、REFACTOR で何を変えたかが書かれている
- ヒント：正規表現の組み立てには `String#gsub` が使えます。`:id` のような部分を見つけて、名前付きのグループに置き換えることを考えてください

### 演習14-3：Controller と render / redirect をTDDで作る

- `UsersController.action(:index)` を Router から呼べるようにする
- `to: "users#index"` の文字列形式を扱えるようにする
- 重複が見えてから `ControllerBase` を作り、`params`（この時点では path parameter だけでよい）を共通化する
- `render_text`、`render_html`、`status:` オプション、`redirect_to`（`status:` オプション付き）、二重 render の例外、何も render しない場合の振る舞いをテストから作る
- **完了条件**：
  - 次のテストがすべてあり、通る：Controller の Action を呼べる／同じ Controller の複数 Action／文字列形式／path parameter を `params` で読める／`render_text`／`render_html`／`status:`／`redirect_to` が 302 と `location` を返す／`redirect_to` の `status:`／二重 render で例外／何も render しない場合
  - レスポンスヘッダ名がすべて小文字（テストを `Rack::Lint` で包んでも通る）
  - `notes/controller.md` に「Router と Controller の責務の違い」「何も render しない場合の仕様と、それを選んだ理由」が書かれている
- ヒント：Router に Controller の作り方を書き込みすぎていないか、REFACTOR のたびに見直してください。Router は「どこに行くか」を決めるだけです

### 演習14-4：View・XSS 対策・Request/Params をTDDで作る

- 静的な HTML ファイルを返す `render_file` を作り、次に ERB を評価する `render_view` を作る
- Controller の `@変数` をテンプレートから参照できるようにし、配列を `each` で表示できることを確かめる
- テンプレートが見つからない場合の振る舞いをテストで決める
- `h` ヘルパーを使えるようにする
- 本文の脆弱な `xss.ru` を `127.0.0.1` で起動し、ブラウザでアラートが出ることを確認してから、すぐ止める。何が起きたかをメモする
- `request` と `params` を整理する：query／path／form を `params` で読める、`request` からメソッドとパスを読める、同じキーがぶつかったときの優先順位、キーの形（文字列かシンボルか）
- **完了条件**：
  - **XSS のテストが通る**：`<script>alert(1)</script>` を含む入力が、テンプレートの出力で `&lt;script&gt;` にエスケープされ、`<script>` がそのまま含まれない
  - **属性値のテストが通る**：`" onmouseover="alert(1)` を含む入力が、`value="..."` の属性を抜け出さない
  - `&` や `'` を含む値（たとえば `Tom & 'Jerry'`）が、エスケープされた形で表示されるテストが通る
  - params のテスト（query、path、両方、form、優先順位、キーの形）がすべて通る
  - `notes/view-params.md` に「なぜ View から `@変数` が見えるのか」「XSS の実験で何が起きたか」「params の優先順位とその理由」が書かれている
- ヒント：`h` を使い忘れたテンプレートがあっても気づけるように、XSS のテストは「実際に使うテンプレート」を通して確かめてください。エスケープ関数だけを単体でテストしても、テンプレート側で使い忘れたら意味がありません

### 演習14-5：before_action をTDDで作り、アプリを組み立て直す

- 本文の6つの振る舞い（複数・only・except・中断・メソッド名指定・継承）を、この順にテストから作る
- テストごとに `Class.new(ControllerBase)` で新しいクラスを作り、テスト同士が影響しないようにする
- 演習14-1のユーザー一覧アプリを、自作フレームワークで作り直す。`config.ru` で Router を `run` し、127.0.0.1 で起動して確認する
- 「ダミーのトークン `?token=dummy` がなければ `/` にリダイレクトする」before_action を、`create` にだけ適用する
- **完了条件**：
  - before_action の6つの振る舞いのテストが通る。子クラスの登録が親に漏れないテストを含む
  - `bundle exec rake test` を `--seed` を変えて3回実行し、毎回通る（テストの順番が変わっても通る）
  - 作り直したアプリで、`curl -i -X POST -d 'name=Carol' http://127.0.0.1:3000/users` が 302 と `location: /` を返し、`?token=dummy` 付きなら登録される
  - 演習14-1の `config.ru` と比べ、つらさがどう解消したかを `notes/before-after.md` に書いている
- ヒント：Minitest はテストを毎回ランダムな順番で実行します。`bundle exec rake test TESTOPTS="--seed=1234"` のように seed を指定すると、同じ順番を再現できます。ここで使うトークンは本物の認証ではありません。本物のログインの仕組みは第19章で学びます
- 発展：`after_action` を作る／レイアウト `views/layouts/application.erb` を導入し、`<%= yield %>` の位置に各画面を埋め込む／`render_json` を作る

## 確認クイズ

**Q1.** Router を Rack アプリ（`call(env)` を持つもの）として作ると、何がうれしいですか。

**Q2.** Action が `[200, { "content-type" => "text/html" }, [...]]` を直接返す代わりに、`render_html` を呼ぶ形にしました。この変更で、何が Action から ControllerBase に移りましたか。

**Q3.** `redirect_to "/users"` は、HTTP のレスポンスとしては何を返していますか。また、フォームの POST の後にリダイレクトするのはなぜですか。

**Q4.** View のテンプレートから Controller の `@title` が見えるのはなぜですか。

**Q5.** 「DBに保存された値は、自分のアプリが保存したものだから、表示するときにエスケープしなくてよい」。この意見は正しいですか。

**Q6.** ログイン確認を Rack ミドルウェアではなく before_action に書くのは、どんな場合ですか。

### 解答と解説

**A1.** Router 自体を `config.ru` で `run` したり、ミドルウェアで包んだりできます。テストでも `router.call(env_for(...))` と、他の Rack アプリと同じ方法で呼べます。約束を守る部品は、組み合わせが自由になります。

**A2.** 「HTTP レスポンスの組み立て方」が移りました。ステータスの既定値、`content-type` の値、本文を配列に入れること、二重 render の検査です。Action は「HTML を返したい」という意味だけを書けばよくなります。

**A3.** ステータス 302 と、`location: /users` ヘッダを返しています。本文は空で構いません。ブラウザはこれを見て `/users` に GET し直します。POST の結果を直接表示すると、再読み込みで同じ POST が送られ、二重登録が起きることがあります。リダイレクトで GET の画面に移れば、それを防げます（PRG）。

**A4.** テンプレートを `ERB#result(binding)` で評価しているからです。`binding` は、その場所から見える変数やメソッドの一式です。Controller のインスタンスメソッドの中で評価するので、そのインスタンスの `@title` が見えます。値は Controller のインスタンスに保存されています。

**A5.** 正しくありません。保存された値も、もとは誰かが入力したものです。入力時にチェックをすり抜けたもの、別の経路で入ったもの、仕様変更前に保存されたものもあり得ます。エスケープは「出力するとき」に、出力先（HTML）に合わせて必ず行います。

**A6.** 「どの Action か」によって処理を変えたい場合です。たとえば一覧は誰でも見られるが、作成・削除だけはログインが必要、という場合です。before_action は Controller と Action を知っていて、`only` や `except` で適用範囲を選べます。全リクエスト共通でよい処理なら、ミドルウェアが向いています。

## AIの使い方（この章）

推奨する聞き方の例です。

- 「Router のテストとして、次の5つを考えました。抜けている観点があれば、テストの名前と意図だけ挙げてください。実装は書かないでください」（自分のテスト名の一覧を貼る）
- 「Ruby の `binding` と `instance_exec` の違いを、小さな例で説明してください。私の課題の答えは書かないでください」
- 「XSS のテストとして、`<script>` 以外にどんな入力を試すべきですか。HTML の本文と属性値に分けて教えてください」

この章でやってはいけない使い方です。

- 「Router を作って」「ControllerBase を実装して」「このテストを通るコードを書いて」と頼むこと。この章の価値は、テストから設計が生まれる過程を自分で体験することです。AI に書かせたコードが通っても、Rails を読む力は付きません
- 失敗したテストを AI に貼り、「テストを通るように直して」とだけ頼むこと。代わりに「このエラーメッセージは何を意味していますか」と聞き、直すのは自分で行ってください

## まとめ

素の Rack で集めた「つらさ」を、1つずつ部品にしました。Router は分岐を、Controller は置き場所を、render/redirect は HTTP の細部を引き受けます。View は HTML を、params は入力の経路を、before_action は共通処理を引き受けます。View では、ERB が自動でエスケープしないことを自分で確かめ、エスケープをテストで保証しました。フレームワークとは、こうした仕組みの集まりであり、「アプリのコードをどこにどう書くか」を決める約束でもあります。

さらに深く：[Phase 4.1](https://github.com/speee/training-web-app/blob/main/phase-4-1.md)、[Phase 4.2](https://github.com/speee/training-web-app/blob/main/phase-4-2.md)、[Phase 4.3](https://github.com/speee/training-web-app/blob/main/phase-4-3.md)、[Phase 4.4](https://github.com/speee/training-web-app/blob/main/phase-4-4.md)、[Phase 4.5](https://github.com/speee/training-web-app/blob/main/phase-4-5.md)、[Phase 4.6](https://github.com/speee/training-web-app/blob/main/phase-4-6.md)、[Phase 4.7](https://github.com/speee/training-web-app/blob/main/phase-4-7.md)、[OWASP XSS 対策チートシート](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

## 用語

| 用語 | 意味 |
|---|---|
| フレームワーク | アプリの骨組みと共通処理を引き受け、コードの書き方と置き場所を決める仕組み |
| Router | メソッドとパスの組み合わせから、呼び出す処理を決める部品 |
| path parameter | `/users/42` の `42` のように、URL のパスの一部として渡される値 |
| Controller | 関連する処理をまとめたクラス。画面やリソースの単位で作る |
| Action | Controller の中の1つ1つの処理（メソッド）。`index`、`show` など |
| render | 返したい内容を伝え、HTTP レスポンスの組み立てをフレームワークに任せる操作 |
| redirect | ステータス 3xx と `location` ヘッダで、別の URL への移動を指示するレスポンス |
| PRG | Post/Redirect/Get。POST の後にリダイレクトし、二重送信を防ぐ形 |
| View / テンプレート | 表示を担当する部分。HTML にデータを埋め込むためのひな形 |
| ERB | Ruby 標準のテンプレートエンジン。`<%= %>` で値を出力する |
| binding | ある場所から見える変数・メソッドの一式を表すオブジェクト |
| エスケープ | 特別な意味を持つ文字を、意味を持たない表記に置き換えること |
| XSS | 入力に含まれたスクリプトが、他人のブラウザで実行される脆弱性 |
| params | query・path・form の入力値をまとめた窓口 |
| before_action | Action の前に共通処理を実行し、必要なら中断する仕組み |

---
[← 前の章](./13-rack-concurrency-tls.md) | [目次](../README.md) | [次の章 →](./15-sql-and-database.md)
