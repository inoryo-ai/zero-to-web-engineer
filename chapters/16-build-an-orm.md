# 第16章 ORMを自作する

> **この章のゴール**：生SQLのつらさから出発し、TDDで小さなORM（Model・Finder・永続化・バリデーション・BaseModel）を自作します。そのうえで、ORMの限界と ActiveRecord が何を引き受けているかを説明できるようになります。
> **目安時間**：読む 5時間 ＋ 演習 25時間
> **前提**：第15章まで（特に第6章のTDD、第14章の自作フレームワーク、第15章のSQL）

## この章で学ぶこと

- Controller に生SQLを書くと何がつらいのかを、自分のコードを例に説明できる
- 値をプレースホルダでバインドし、動的なカラム名を許可リストで制限して、SQLインジェクションを防げる
- DBの1行を Ruby のオブジェクトに変換する仕組み（ORMの核心）を実装できる
- `all` / `find` / `where` / `save` / `destroy` / バリデーションを TDD で育てられる
- 共通部分を BaseModel に集め、「どこまで抽象化するか」を判断できる
- N+1問題とトランザクションを例に、自作ORMの限界と ActiveRecord の価値を説明できる

## この章の進め方

この章は長い章です。元教材の Phase 5.1〜5.9 の9つの段階を、1つの章にまとめています。1日で終わらせようとしないでください。1ステップ進むごとにテストを通し、Git にコミットするのが安全な進め方です。

ORM（オーアールエム）とは Object-Relational Mapping の略です。データベースの「表の1行」と、プログラムの「オブジェクト」を対応づける仕組みのことです。たとえるなら、データベースという外国語の書類を、Ruby という母国語に通訳してくれる係です。Rails の ActiveRecord は、世界で最もよく使われている ORM の1つです。

この章では、その通訳係を自分で作ります。ただし、最初から通訳係を作るわけではありません。まず通訳なしで外国語の書類を直接読み書きし、そのつらさを体験します。つらさが十分に分かってから、少しずつ通訳を雇っていきます。この順番が大事です。「便利だから使う」ではなく「必要だから生まれた」と腹落ちさせるためです。

全体の流れは次のとおりです。

| ステップ | やること | 欲しくなったもの | できあがるAPIの例 |
|---|---|---|---|
| 1 | Controller に生SQLを書く | SQLをどこかにまとめたい | （なし） |
| 2 | 1テーブル専用の Model を作る | Model をすぐ試したい | `User.all`, `User.find(1)` |
| 3 | Console を作る | 戻り値を自然にしたい | `bin/console` |
| 4 | 行をオブジェクトにする | 他の Model にも広げたい | `user.name` |
| 5 | Finder を共通化する | 保存もしたい | `User.where(name: "Alice")` |
| 6 | 永続化を作る | 不正な値を止めたい | `user.save`, `user.destroy` |
| 7 | バリデーションを作る | 重複を減らしたい | `user.errors` |
| 8 | BaseModel にまとめる | 本物と比べたい | `class User < BaseModel` |
| 9 | 限界を分析する | （第18章の Rails へ） | レポート |

各ステップで必ず RED → GREEN → REFACTOR の順に進めます。RED は「まだ実装がないので失敗するテスト」を書くこと、GREEN は「そのテストを通す最小の実装」を書くこと、REFACTOR は「テストを通したまま整理する」ことでした（第6章）。

### 作業フォルダを用意する

第14章で作ったフレームワークを土台にします。第14章のフォルダ（例：`~/workspace/ch14-framework/`）を丸ごとコピーして、この章の作業フォルダ `~/workspace/ch16-orm/` を作ります。

```bash
cp -R ~/workspace/ch14-framework ~/workspace/ch16-orm
cd ~/workspace/ch16-orm
git status
```

Ruby から SQLite を使うために、`Gemfile` に `gem "sqlite3"` があることを確かめ、`bundle install` します。ターミナルで使う `sqlite3` コマンド（第15章で使ったもの）と、Ruby の sqlite3 gem は別物です。両方が必要です。

開発用のDBファイルは `db/development.sqlite3` に置きます。中身はダミーデータだけにします。DBファイルは Git に入れないのが普通なので、先に `.gitignore` に `db/*.sqlite3` を足しておきます。テストでは、ファイルを使わずメモリ上のDB（`":memory:"`）を毎回新しく作ります。こうすると、テスト同士が互いのデータに影響しません。

## ステップ1：生SQLのつらさを体験する

最初のステップでは、あえて Controller に直接SQLを書きます。目的は「動かすこと」ではなく「痛みを感じること」です。

### DBの戻り値はどんな形か

まず、Ruby から SQLite を触ってみます。次のスクリプトは、メモリ上にDBを作り、2件入れて取り出します。

```ruby
require "sqlite3"

db = SQLite3::Database.new(":memory:")
db.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)")
db.execute("INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')")
row = db.execute("SELECT id, name, email FROM users").first
p row
p row[1]
db.results_as_hash = true
p db.execute("SELECT id, name, email FROM users").first["name"]
```

```text
[1, "Alice", "alice@example.com"]
"Alice"
"Alice"
```

既定では、1行は「配列」で返ってきます。名前を取り出すには `row[1]` と書きます。`1` が name だと覚えておくのは人間の仕事です。SELECT の列の順番を変えたら、`row[1]` は別の値を指してしまいます。

`results_as_hash = true` にすると、`row["name"]` と書けるようになります。少しましですが、キーが文字列なのか記号（Symbol）なのか、DBライブラリのバージョンによって細かい違いがあります。どちらにしても、「DBライブラリの都合」がアプリのコードに漏れ出しています。

### Controller にSQLを書いてみる

元教材と同じく、`users` テーブル（id, name, email）を題材にします。一覧・詳細・名前検索の3つを、TDDで作ります。最初のテストは、たとえば次のような形です。`seed_users` はテスト用にデータを入れる補助メソッドで、自分で作ります。`env_for` は第14章で作った、リクエスト環境を組み立てる補助メソッドです。

```ruby
def test_users_index_displays_all_users
  seed_users([
    { name: "Alice", email: "alice@example.com" },
    { name: "Bob", email: "bob@example.com" }
  ])

  router = Router.new
  router.get "/users", to: "users#index"

  status, _headers, body = router.call(env_for("/users"))

  assert_equal 200, status
  assert_includes body.join, "Alice"
  assert_includes body.join, "Bob"
end
```

このテストを通すために、`UsersController#index` の中で `db.execute("SELECT ...")` を呼びます。詳細表示（`/users/:id`）、存在しないIDのとき（404にするなど、仕様を先にテストで決めます）、名前検索（`/users/search?name=Bob`）と、テストを1つずつ足していきます。

3つ書き終えたら、Controller を眺めてください。おそらく次のような状態になっているはずです。

- 似たような SELECT 文が3か所にある
- DB接続を取り出す処理があちこちにある
- `.first` を付けたり、0件のときの分岐を書いたりしている
- View が `row[1]` や `row["name"]` を知っている

Controller は本来、「リクエストを受けて、何を表示するか決める」場所です。DBの細かい事情を書く場所ではありません。ここで感じた「もやもや」を、メモに書き残してください。この後のすべてのステップは、このもやもやを1つずつ解消していく作業です。

ここで大きく整理したくなりますが、我慢してください。DB接続を `db` というメソッドに切り出す程度の整理にとどめます。つらさを残しておくことで、次のステップで Model を導入する意味がはっきりします。

### 必須要件：SQLインジェクションを防ぐ

名前検索を作るとき、つい次のように書きたくなります。

```ruby
sql = "SELECT id, name FROM users WHERE name = '#{name}'"
```

これは絶対に書いてはいけない形です。実際に、悪意のある入力を入れて試してみます。

```ruby
require "sqlite3"

db = SQLite3::Database.new(":memory:")
db.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL)")
db.execute("INSERT INTO users (name, email) VALUES (?, ?)", ["Alice", "alice@example.com"])
db.execute("INSERT INTO users (name, email) VALUES (?, ?)", ["Bob", "bob@example.com"])

p db.execute("SELECT id, name, email FROM users ORDER BY id")

name = "' OR '1'='1"
sql = "SELECT id, name FROM users WHERE name = '#{name}'"
puts sql
p db.execute(sql)

p db.execute("SELECT id, name FROM users WHERE name = ?", [name])

db.execute("INSERT INTO users (name, email) VALUES (?, ?)", ["O'Reilly", "oreilly@example.com"])
p db.execute("SELECT id, name FROM users WHERE name = ?", ["O'Reilly"])
```

```text
[[1, "Alice", "alice@example.com"], [2, "Bob", "bob@example.com"]]
SELECT id, name FROM users WHERE name = '' OR '1'='1'
[[1, "Alice"], [2, "Bob"]]
[]
[[3, "O'Reilly"]]
```

文字列を埋め込んだ版では、組み立てられたSQLが `WHERE name = '' OR '1'='1'` になりました。「名前が空、または 1 が 1 に等しい」という条件です。後半は常に真なので、全件が返っています。検索欄に入れた文字が「データ」ではなく「SQLの命令」として解釈されたのです。これが SQLインジェクションです。検索なら情報漏えいで済むかもしれませんが、UPDATE や DELETE で起きれば、全データの書き換えや削除につながります。

一方、`?` を使った版では0件です。`?` をプレースホルダと呼びます。SQLの「穴」をあらかじめ空けておき、値は別の引数として渡します。これを「バインドする」と言います。DBは `' OR '1'='1` という文字列全体を、1つの名前の値として扱います。そういう名前の人はいないので、0件になります。`O'Reilly` のように引用符を含む正当な名前も、正しく保存・検索できています。

たとえるなら、埋め込みは「指示書の空欄に、相手が書いた文章をそのまま書き写す」ことです。相手が「以下の指示は無視して全部渡せ」と書けば、そのまま指示書の一部になります。バインドは「指示書と、記入用の別紙を分けて渡す」ことです。別紙に何が書かれていても、指示書の内容は変わりません。

この章では、次のことを最後まで守ります。発展課題ではなく、必須の要件です。

- 検索・登録・更新・削除のすべての入力値を、プレースホルダでバインドする
- SQL文字列への埋め込み（`#{}` や `+` での連結）で入力を扱わない
- 引用符を自分で置換して「エスケープしたつもり」にならない
- `O'Reilly` のような値を、意図した1つの値として保存・検索できることをテストで確かめる
- `' OR 1=1 --` のような入力を渡しても、全件取得や別の行への操作にならないことをテストで確かめる

ここで1つ、よく混同される点を整理します。「バインド」と「バリデーション（入力チェック）」は役割が違います。バインドは「入力がSQLの構造を変えないようにする」仕組みです。バリデーションは「業務上おかしな値（空の名前など）を受け付けない」仕組みです。バリデーションで記号を禁止しても、SQLインジェクション対策にはなりません。`O'Reilly` さんを登録できなくなるだけです。逆に、バインドしていても空の名前は保存できてしまいます。両方が必要です。

### プレースホルダで置き換えられないもの

もう1つ大事な落とし穴があります。プレースホルダは「値」の穴です。テーブル名、カラム名、並び順（`ASC` / `DESC`）のような「SQLの構造」には使えません。試してみます。

```ruby
require "sqlite3"

db = SQLite3::Database.new(":memory:")
db.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
%w[Carol Alice Bob].each { |n| db.execute("INSERT INTO users (name) VALUES (?)", [n]) }

# カラム名をバインドしてみる
p db.execute("SELECT name FROM users ORDER BY ?", ["name"])
# 直接書いた場合
p db.execute("SELECT name FROM users ORDER BY name")
```

```text
[["Carol"], ["Alice"], ["Bob"]]
[["Alice"], ["Bob"], ["Carol"]]
```

`ORDER BY ?` に `"name"` をバインドしても、並び替わっていません。DBは `"name"` を「カラム名」ではなく「name という文字列の定数」として扱ったからです。すべての行で同じ値なので、並び順に影響しません。エラーにならないので、気づきにくいのが厄介です。

では、カラム名を利用者に選ばせたいときはどうするか。答えは「許可リスト」です。開発者があらかじめ「使ってよいカラム名の一覧」を固定で持っておき、その一覧にある名前だけを受け付けます。一覧にない名前は、SQLを実行する前に拒否します。ステップ5の `where` で、これを実装します。

> さらに深く：[Phase 5.1: 生SQLのつらさをTDDで体験する](https://github.com/speee/training-web-app/blob/main/phase-5-1.md)

## ステップ2：1テーブル専用の Model を導入する

ステップ1で感じた「Controller にSQLを書きたくない」という欲求に、最小限の手で応えます。`users` テーブルへのアクセスを、`User` というクラスに集めます。これが Model です。

### Model とは何か

Model は「データとそのルールを担当する部品」です。レストランでたとえると、Controller はホール係、View は盛り付け、Model は厨房と食材庫の管理係です。ホール係は「Aさんの注文を出して」と頼むだけで、食材庫のどの棚に何があるかは知りません。知っているのは厨房の係だけです。

ここでの Model の役割は、SQLを「隠す」ことではありません。データにアクセスする責任を、Controller から「分けて」1か所に置くことです。SQLは消えずに、Model の中に引っ越します。

### TDDで `User.all` と `User.find` を作る

最初のテストは「`User.all` を呼びたい」という要求を固定するものです。この時点では `User` クラスは存在しないので、テストは失敗します。これが RED です。

次に、Controller に書いていたSQLを、ほぼそのまま `User.all` の中に移して GREEN にします。それから Controller を `@users = User.all` に書き換えます。テストが通ったままなら、REFACTOR 成功です。

同じ流れで `User.find(id)` を作ります。ここで設計上の判断が1つ必要です。「見つからなかったとき、何を返すか」です。

- `nil` を返す：呼び出し側で `if user.nil?` と分岐する
- 例外を投げる：呼び出し側で `rescue` する。404 に変換しやすい

どちらが正解ということはありません。大事なのは、選んだ振る舞いを先にテストで固定することです。ちなみに Rails の ActiveRecord では、`find` は例外を投げ、`find_by` は `nil` を返します。両方の使い分けを用意しているわけです。

### DB接続をどこに持つか

`User.all` の中で、DB接続をどう手に入れるかも設計ポイントです。毎回 `SQLite3::Database.new` するのは無駄ですし、テストではメモリDBに差し替えたいはずです。

よくある形は、接続を持つ小さなモジュールを1つ作り、Model とテストの両方からそれを使う方法です。たとえば `Database.connection` という置き場所を作り、テストの `setup` でメモリDBを代入します。接続情報をどこに書くか（開発用DBのパスなど）も、この1か所に集めます。

`all` と `find` を作り終えたら、両者の中身を見比べてください。「SQLを実行する」「結果を受け取る」部分が似ているはずです。最小限の共通化として、SQL実行を private メソッドにまとめるくらいはしてよいです。ただし、まだ BaseModel（共通の親クラス）は作りません。重複が2つの Model にまたがって見えてからにします。

> さらに深く：[Phase 5.2: 1テーブル専用ModelをTDDで導入する](https://github.com/speee/training-web-app/blob/main/phase-5-2.md)

## ステップ3：Console で Model を対話的に試す

Model ができると、「`User.all` がちゃんと動くか、ちょっと試したい」と思う場面が増えます。そのたびにサーバを起動してブラウザで確かめるのは遠回りです。HTTP、Router、Controller、View を全部通るので、Model だけの挙動が見えにくくなります。

そこで、アプリのコードを読み込んだ状態で Ruby の対話環境を起動する「Console」を作ります。Rails の `rails console` の自作版です。

### IRB を使う

対話環境を自作する必要はありません。Ruby には IRB（アイアールビー）という標準の対話環境があります。第3章で使った `irb` コマンドの正体です。IRB はプログラムからも起動できます。

`bin/console` というファイルを作り、その中で次の3つを順番に行います。

1. アプリの初期化ファイル（DB接続の設定や Model の `require` をまとめたもの）を読み込む
2. 開発用DBに接続する
3. IRB を起動する

IRB をプログラムから起動する部分は、次の2行だけです。

```ruby
require "irb"
IRB.start
```

この2行の前に、自分のアプリを読み込む `require_relative` を書きます。何を読み込めば `User` が使えるようになるのかは、自分のフォルダ構成に合わせて考えてください。ファイルを作ったら `chmod +x bin/console` で実行権限を付け、`bin/console` で起動します。なお、Ruby 4.0 以降で Bundler 経由で起動する場合は、`Gemfile` に `gem "irb"` が必要です（Ruby 3.4 では警告が出ます）。

### 開発用DBとテスト用DBを分ける

Console はふだん開発用DB（`db/development.sqlite3`）につなぎます。テストはメモリDBを使います。この区別を、環境変数（例：`APP_ENV=test`）や引数で切り替えられるようにしておくと、後で混乱しません。テストを実行したら開発用のデータが消えた、という事故はよく起きます。

### Console はテストの代わりではない

Console で試せるからといって、テストを省いてはいけません。役割が違います。

- テスト：決めた仕様を自動で何度でも確かめる。仕様の「固定」
- Console：APIの使い心地や戻り値を、手で触って観察する。設計の「発見」

さっそく Console で `User.all` と `User.find(1)` を呼び、戻り値の `class` を表示してみてください。おそらく `Array` が返ってきます。1件分も、配列かハッシュです。「User と名乗っているのに、中身はただの配列」。この違和感が、次のステップの出発点です。

> さらに深く：[Phase 5.3: Consoleモードを実装する](https://github.com/speee/training-web-app/blob/main/phase-5-3.md)

## ステップ4：レコードをオブジェクトとして扱う

ここが ORM の核心です。DBから取ってきた1行を、`User` クラスのインスタンス（オブジェクト）に変換します。

### なぜオブジェクトにしたいのか

`user[1]` や `user["name"]` ではなく、`user.name` と書きたい。これは見た目の好みだけの話ではありません。

- `user.name` なら、View や Controller はDBの列の順番や、ハッシュのキーの形を知らなくてよい
- `User` クラスにメソッドを足せば、「表示用の名前」のようなふるまいを持たせられる（データ＋ふるまい）
- 存在しない属性を呼ぶと `NoMethodError` になるので、タイプミスにすぐ気づける

データは「値の集まり」です。オブジェクトは「値と、その値を扱うふるまいの集まり」です（第5章）。DBの行をオブジェクトに変換することで、アプリのコードからDBの都合が見えなくなります。ORM の本質は「SQLを書かずに済むこと」ではなく、この「永続化されたデータをオブジェクトとして扱えること」です。

### RED から始める

まず「`User.all` の中身が `User` のオブジェクトで、`name` メソッドが使える」というテストを書きます。

```ruby
require "minitest/autorun"

class UserTest < Minitest::Test
  def test_all_returns_user_objects
    users = User.all
    assert_equal "Alice", users.first.name
  end
end
```

この例ではわざと何も読み込んでいないので、`User` が見つからずに失敗します。

```text
Run options: --seed 6306

# Running:

E

Finished in 0.000479s, 2087.6827 runs/s, 0.0000 assertions/s.

  1) Error:
UserTest#test_all_returns_user_objects:
NameError: uninitialized constant UserTest::User
    red/user_test.rb:5:in `test_all_returns_user_objects'

1 runs, 0 assertions, 0 failures, 1 errors, 0 skips
```

自分の作業フォルダでは、`User` は存在するので、エラーの内容は違うはずです。たとえば「`Array` に `name` というメソッドはない」という `NoMethodError` になるでしょう。どんなエラーで失敗したかを読むことも、RED の大事な仕事です（第7章）。エラーの表示形式は Ruby のバージョンによって少し異なります。

### GREEN への道筋

1件分の行データを受け取り、インスタンス変数に入れる `initialize` を書きます。そのうえで、`all` の結果の各行を `User.new(...)` に変換します。配列の各要素を変換するには `map` が使えます（第4章）。`find` も同じ変換を通すようにします。

属性を読み書きするメソッドは、最初は `attr_accessor :id, :name, :email` のように手で書いてかまいません。カラムが増えたらどうするか、という問題には後で向き合います。

### 型の違いに注意する

DBとRubyでは、値の型の対応がずれることがあります。SQLite の `INTEGER` は Ruby の `Integer` に、`TEXT` は `String` に、`NULL` は `nil` になります。ところが SQLite には真偽値（true / false）の型がなく、`0` と `1` で表すのが普通です。`done` のようなカラムを作ると、`user.done` が `true` ではなく `1` を返します。日付も文字列で返ってきます。本物の ORM は、この型変換も引き受けています。自作版では、気づいた違いをメモしておくだけで十分です。

### DBの構造が漏れていないか確認する

ステップ4の最後に、Controller と View を見直してください。`row[1]` や `["name"]` が残っていたら、すべて `user.name` の形に直します。古いアクセス方法を残すと、2つの書き方が混ざり、どちらが正しいのか分からなくなります。

そして、Model の中を見てください。`initialize` に属性を1つずつ代入するコードや、`attr_accessor` の列挙が、カラムの数だけ並んでいるはずです。「自動化したい」と思ったら、それが次への伏線です。ただし、まだ自動化はしません。

> さらに深く：[Phase 5.4: レコードをオブジェクトとして扱う](https://github.com/speee/training-web-app/blob/main/phase-5-4.md)

## ステップ5：Finder を共通化し、where を安全に作る

### 2つ目の Model を作ってみる

`users` に加えて、`posts` テーブル（id, user_id, title など）を用意し、`Post.all` を使いたいというテストを書きます。まずは素直に `Post` クラスを作って GREEN にします。

書き終えたら、`User` と `Post` を並べて見比べてください。テーブル名と属性名以外は、ほとんど同じはずです。DB接続、SQL実行、行からオブジェクトへの変換、配列にまとめる処理。コピーして貼り付けたコードは、片方だけ直し忘れるバグの温床になります。

ここで初めて、共通の親クラスを作る理由が生まれました。第5章で学んだ継承を使い、共通処理を親クラスに移します。Model 固有に残るのは、主に次の2つです。

- どのテーブルを読むか（テーブル名）
- どの属性（カラム）を持つか

### テーブル名をどう決めるか

テーブル名は、各クラスに明示的に書かせる方法と、クラス名から推測する方法があります。Ruby では、クラスは自分の名前を文字列で返せます。

```ruby
class User; end
p User.name
p User.name.downcase + "s"
```

```text
"User"
"users"
```

ただし、英語の複数形は単純ではありません。`Person` は `persons` ではなく `people` です。`Category` は `categorys` ではなく `categories` です。Rails はこれを「活用形の辞書」で処理しています。自作版では、明示的に書く方式から始めるのが安全です。推測するにしても、上書きできる逃げ道を用意してください。

### where を作る

次に、条件付きで取得する `User.where(name: "Alice")` を作ります。引数はハッシュです。キーがカラム名、値が探したい値です。ここが、この章で最も安全上の注意が必要な場所です。

`where` の中では、キーからSQLの `WHERE name = ?` の部分を組み立てます。値はすべてプレースホルダでバインドします。そしてキー（カラム名）は、ステップ1で見たとおりバインドできません。だから Model 側に「使ってよいカラム名の一覧」を固定で持ち、その一覧にないキーが来たら、SQLを組み立てる前に例外を投げて拒否します。

先に失敗するテストを書きます。次の例は、メモリDBの準備と、3つの攻撃パターンのテストです。`Database.connection` はステップ2で決めた接続の置き場所の例です。自分の設計に合わせて読み替えてください。

```ruby
require "minitest/autorun"
require_relative "../app/models/user"

class UserSqlInjectionTest < Minitest::Test
  def setup
    Database.connection = SQLite3::Database.new(":memory:")
    Database.connection.execute(
      "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL)"
    )
    [["Alice", "alice@example.com"], ["Bob", "bob@example.com"]].each do |name, email|
      Database.connection.execute("INSERT INTO users (name, email) VALUES (?, ?)", [name, email])
    end
  end

  def test_where_treats_sql_fragment_as_value
    users = User.where(name: "' OR '1'='1")
    assert_equal [], users
  end

  def test_where_rejects_unknown_column
    assert_raises(ArgumentError) do
      User.where("name = name OR 1=1 --" => "x")
    end
  end

  def test_update_changes_only_target_row
    user = User.find(1)
    user.name = "' WHERE 1=1 --"
    user.save

    assert_equal "' WHERE 1=1 --", User.find(1).name
    assert_equal "Bob", User.find(2).name
  end
end
```

3つ目のテストは、ステップ6で `save` を作るまで RED のままで大丈夫です。投げる例外を `ArgumentError` にするか、自分で定義した例外クラスにするかは、自由に決めてください。大事なのは「DBに問い合わせる前に止まる」ことです。拒否したときにDBの中身が変わっていないことも、確かめておくとより安心です。

実装の考え方だけ示します。キーの一覧を `each` で回し、許可リスト（例：`["id", "name", "email"]`）に含まれるかを `include?` で調べます。含まれなければ `raise` します。全部が通過した後で初めて、`"name = ?"` のような部品を `join(" AND ")` でつなぎ、値の配列と一緒に `execute` に渡します。

「許可リスト」の反対は「拒否リスト」です。「`;` や `--` が入っていたら拒否する」という方式です。これは危険です。攻撃の書き方は無数にあり、すべてを列挙することはできません。許可リストは「知っているものだけ通す」ので、想定外の入力はすべて止まります。

### Query という考え方に気づく

`all` / `find` / `where` を並べると、流れは同じだと分かります。SQLを組み立て、実行し、行をオブジェクトに変換する。違うのは「条件」だけです。`all` は条件なし、`find` は id 条件で1件、`where` は任意の条件です。どれも「問い合わせ（Query）」のバリエーションです。本物の ORM では、この Query 自体がオブジェクトになっています。自作版では、そこまで作らなくてかまいません。気づくことが大切です。

> さらに深く：[Phase 5.5: Finder / Query をTDDで共通化する](https://github.com/speee/training-web-app/blob/main/phase-5-5.md)

## ステップ6：永続化（save / update / destroy）

ここまでは「読む」だけでした。次は「書く」です。新規作成・更新・削除を作ります。

### INSERT と UPDATE の違い

新しい行を作るのが INSERT、既にある行を書き換えるのが UPDATE です。ORM では、どちらも `save` という1つのメソッドで行うのが一般的です。

```ruby
user = User.new(name: "Alice", email: "alice@example.com")
user.save          # まだDBにない → INSERT

user.name = "Alice Updated"
user.save          # 既にDBにある → UPDATE
```

`save` を呼ぶ側は、INSERT か UPDATE かを気にしなくて済みます。その代わり、オブジェクト自身が「自分はDBに保存済みか」を知っている必要があります。これを「状態」と呼びます。

### 状態をどう判定するか

よく使われる目印は `id` です。`id` が `nil` なら、まだDBに保存されていない新しいオブジェクトです。`id` があれば、DBに対応する行があるはずです。Rails ではこれを `new_record?` / `persisted?` というメソッドで表しています。

INSERT した直後、DBは新しい行に id を割り振ります。この id をオブジェクトに戻さないと、同じオブジェクトで2回 `save` したときに、2行目が作られてしまいます。SQLite では、直前に挿入した行の id を `last_insert_row_id` というメソッドで取得できます。

### TDD の順番

元教材と同じく、次の順で小さく進めます。

1. 新規作成のテスト（`User.new(...).save` で件数が1増える）→ INSERT を実装
2. 保存後に id が入っているテスト → id を戻す
3. 既存レコードの更新テスト（`find` → 属性変更 → `save` → 再度 `find` すると変わっている）→ UPDATE を実装
4. 新規なら INSERT、既存なら UPDATE に分かれるテスト → 状態で分岐
5. 削除のテスト（`user.destroy` 後に `find` すると見つからない）→ DELETE を実装

UPDATE と DELETE では、`WHERE id = ?` の条件を絶対に忘れないでください。条件のない UPDATE や DELETE は、テーブルの全行を対象にします。ステップ5のテスト例の3つ目「対象の行だけが変わる」は、この事故を防ぐテストでもあります。DELETE についても「対象外の行の件数が減らない」テストを書きます。

### SQLインジェクション対策を引き継ぐ

INSERT と UPDATE では、属性の値をすべてバインドします。UPDATE と DELETE の id もバインドします。`UPDATE users SET name = ?, email = ? WHERE id = ?` の形です。

属性名（`name`, `email`）からSQLの `SET` 部分を組み立てる場合、それは「カラム名」です。だから where と同じく、許可リストに含まれる名前だけを使います。`User.new` に渡されたハッシュに、未知のキーが混ざっていたらどうするかも決めておきましょう。黙って無視するか、例外にするか。どちらにしても、その未知のキーをSQLに入れてはいけません。

### オブジェクトはDBの完全な写しではない

`destroy` した後のオブジェクトは、Ruby のメモリ上にはまだ残っています。`user.name` も読めます。でも、DBにはもう対応する行がありません。逆に、別の誰かがDBを更新しても、手元のオブジェクトは古い値のままです。

ORM の難しさは「取得」ではなく、「状態を持つオブジェクトを、DBと同期させ続けること」にあります。どの時点でDBと一致していて、どの時点でずれているのか。これを曖昧にすると、見つけにくいバグになります。

発展として、次の点も考えてみてください。`save` の戻り値は true / false にするか。`user.update(name: "...")` のような1行で更新するメソッドを作るか。何も変更していないのに `save` を呼んだら、UPDATE を発行すべきか。

> さらに深く：[Phase 5.6: 永続化（Create / Save / Update / Delete）をTDDで実装する](https://github.com/speee/training-web-app/blob/main/phase-5-6.md)

## ステップ7：バリデーションと errors

今の ORM は、名前が空でもメールアドレスが空でも、そのまま保存してしまいます。データの正しさを守る仕組みを足します。

### save は失敗を返せるようにする

まず「名前が空なら保存できない」というテストを書きます。このとき決めることが2つあります。

- 保存できなかったとき、`save` は何を返すか（`false` を返す？例外を投げる？）
- なぜ保存できなかったのかを、どこから知るか

Rails では、`save` は `false` を返し、理由は `errors` に入ります。例外を投げる `save!` も別に用意されています。自作版でも、まずは `false` と `errors` の組み合わせがおすすめです。フォームの入力ミスは「よくあること」で、例外（めったに起きない異常）として扱うのは大げさだからです。

```ruby
user = User.new(name: "", email: "")
user.save    # => false
user.errors  # => ["name can't be blank", "email can't be blank"]
```

### 小さく育てる

最初は `save` の中に `if name.empty?` と書いて false を返すだけで GREEN にします。次に「理由を知りたい」というテストを書き、`errors` 用の配列を持たせます。ここで「errors はいつ空に戻すべきか」を考えてください。1回目に失敗し、値を直して2回目の `save` をしたとき、古いエラーが残っていたら困ります。

`save` の中に if が増えてきたら、バリデーションを `validate` のような別メソッドに分けます。`save` の流れは、次の3段階に整理されます。

1. バリデーションを実行する
2. エラーがなければ保存する（INSERT か UPDATE）
3. エラーがあれば保存せずに false を返す

### バリデーションはどこに書くべきか

入力のチェックは、Controller にも書けます。それでも Model に書くのはなぜでしょうか。

同じ User を保存する経路は、1つとは限りません。登録フォーム、管理画面、Console、CSVの一括取り込み。Controller にチェックを書くと、経路ごとに同じチェックを書くことになり、どこかで書き忘れます。Model に書けば、どこから保存しても必ず同じルールを通ります。「データの正しさを守るのは Model の責任」です。

では、DBの `NOT NULL` 制約（第15章）は不要でしょうか。そうではありません。役割分担を表にします。

| 守る場所 | 得意なこと | 苦手なこと |
|---|---|---|
| Model のバリデーション | 利用者に分かりやすいエラーを返す。業務ルール（文字数・形式）を書ける | Model を通らない書き込み（直接SQL）は止められない |
| DB の制約 | どこから書き込んでも必ず止める。最後の砦 | エラーが分かりにくい。複雑なルールは書きにくい |

本番のアプリでは、両方を使います。そして前に述べたとおり、どちらも SQLインジェクション対策の代わりにはなりません。

発展として、空チェック（presence）以外のルール、たとえば文字数（length）や形式（format）を足してみてください。`validates :name, presence: true` のような書き方（DSL：その用途専用の小さな言語）にしたくなるかもしれません。それは次のステップの話題です。

> さらに深く：[Phase 5.7: バリデーションとモデル責務をTDDで設計する](https://github.com/speee/training-web-app/blob/main/phase-5-7.md)

## ステップ8：BaseModel にまとめる

ここまでで、`User` と `Post` の両方に、Finder・永続化・バリデーションの仕組みがあるはずです。ステップ5で親クラスを作っていれば、一部は既に共通化されています。このステップでは、残った重複を整理し、各 Model が「差分だけ」を持つ形にします。

```ruby
class User < BaseModel
  # テーブル名、カラム、ルールなど「User だけのこと」だけを書く
end

class Post < BaseModel
end
```

### 何を共通にし、何を残すか

まず、`all` / `find` / `where` / `save` / `destroy` / `validate` を1つずつ見て、共通か固有かを表に書き出してください。おおむね次のように分かれるはずです。

- BaseModel に置くもの：SQLの組み立てと実行、行からオブジェクトへの変換、save の分岐、エラー配列の管理、許可リストによる検査の仕組み
- 各 Model に残すもの：テーブル名、カラムの一覧、その Model 固有のバリデーションのルール、その Model 固有のふるまい

ここで注意したいのは、許可リストの「仕組み」は BaseModel に、許可リストの「中身」は各 Model に、という分け方です。共通化した後も、ステップ5のSQLインジェクションのテストが `User` と `Post` の両方で通ることを確かめてください。

### カラム定義の自動化とメタプログラミング

カラムごとに `attr_accessor` を並べるのが面倒になったら、Ruby の機能でメソッドを自動で作れます。プログラムがプログラムを作ることを「メタプログラミング」と呼びます。小さな例を示します（ORM とは無関係な例です）。

```ruby
class Point
  ATTRIBUTES = [:x, :y]

  ATTRIBUTES.each do |attr|
    define_method(attr) { @values[attr] }
  end

  def initialize(values)
    @values = values
  end
end

pt = Point.new(x: 3, y: 4)
p pt.x
p pt.y
p Point.instance_methods(false).sort
```

```text
3
4
[:x, :y]
```

`define_method` は、名前と中身を渡すとメソッドを定義します。配列を回せば、カラムの数だけメソッドができます。

ただし、元教材は「メタプログラミングに逃げない」ことを強く勧めています。メタプログラミングは強力ですが、コードを読んだだけではどんなメソッドがあるのか分からなくなります。エラーが起きたとき、どこで定義されたメソッドなのか追いにくくなります。まず普通のコードで書いて、重複がつらくなってから使ってください。使うなら、何が自動で作られるのかをコメントで残しましょう。

### 抽象化しすぎない

共通化を進めると、何でも BaseModel に入れたくなります。でも、BaseModel が巨大になると、それ自体が読めない塊になります。各 Model が何をしているのか、Model を見ても分からなくなります。

抽象化の判断基準として、次の問いを使ってください。

- その重複は、本当に「同じ理由で」変わるか（たまたま似ているだけではないか）
- 共通化した後のコードは、共通化する前より読みやすいか
- 3つ目の利用者が現れたときに、無理なく使えるか

よく言われる目安に「3回目で共通化する」があります。1回目は書く。2回目は「似ているな」と気づくだけ。3回目でまとめる、という考え方です。抽象化とは「共通化すること」ではなく、「変化に強い構造を作ること」です。

> さらに深く：[Phase 5.8: BaseModel / 抽象化をTDDで設計する](https://github.com/speee/training-web-app/blob/main/phase-5-8.md)

## ステップ9：ORMの限界と ActiveRecord

最後のステップでは、新しい機能はほとんど作りません。自分で作った ORM を、あえて壊すつもりで分析します。

### 簡単に見えて難しいこと

自作 ORM で次のことをやろうとしたら、どうなるか考えてみてください。

- 2つのテーブルを結合する（JOIN）
- 「名前が Alice、または作成日が今日」のような OR 条件
- 並び替え（ORDER BY）。並び順の向きは値ではないので、許可リストが必要
- ページ分け（LIMIT / OFFSET）
- 「User の Post 一覧」のような関連（`user.posts`）

どれも、`where` のハッシュだけでは表現できません。Query は思ったよりずっと複雑です。

### N+1問題

関連を素朴に実装すると、大きなパフォーマンス問題が起きます。次の例は、ユーザーごとに投稿を取りに行くコードです。`trace` という機能で、実際に発行されたSQLを表示しています。

```ruby
require "sqlite3"

db = SQLite3::Database.new(":memory:")
db.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
db.execute("CREATE TABLE posts (id INTEGER PRIMARY KEY, user_id INTEGER, title TEXT)")
%w[Alice Bob Carol].each { |n| db.execute("INSERT INTO users (name) VALUES (?)", [n]) }
[[1, "Hello"], [1, "Ruby"], [2, "SQL"], [3, "Rails"]].each do |uid, t|
  db.execute("INSERT INTO posts (user_id, title) VALUES (?, ?)", [uid, t])
end

count = 0
db.trace { |sql| count += 1; puts "SQL: #{sql}" }

users = db.execute("SELECT id, name FROM users")
users.each do |id, name|
  posts = db.execute("SELECT title FROM posts WHERE user_id = ?", [id])
  puts "#{name}: #{posts.flatten.join(', ')}"
end
puts "発行したSQL: #{count}回"
```

```text
SQL: SELECT id, name FROM users
SQL: SELECT title FROM posts WHERE user_id = 1
Alice: Hello, Ruby
SQL: SELECT title FROM posts WHERE user_id = 2
Bob: SQL
SQL: SELECT title FROM posts WHERE user_id = 3
Carol: Rails
発行したSQL: 4回
```

ユーザーの一覧で1回、ユーザーごとに1回ずつで、合計4回です。ユーザーが N 人いれば、1 + N 回になります。これを「N+1問題」と呼びます。100人なら101回、1万人なら10001回です。DBへの問い合わせは1回ごとに通信と処理の時間がかかるので、画面がどんどん遅くなります。

怖いのは、ORM を使うとこれがコードの見た目から分からないことです。`user.posts` と書くだけでSQLが発行されるからです。「ORM は簡単に見えて、パフォーマンスの問題を隠す」という性質があります。

解決策は、必要な投稿をまとめて1回で取ってくることです。ユーザーの id をすべて集め、`IN` を使って一度に問い合わせます。このとき、id の個数だけ `?` を並べてバインドします。

```ruby
ids = [1, 2, 3]
placeholders = Array.new(ids.size, "?").join(", ")
puts "SELECT user_id, title FROM posts WHERE user_id IN (#{placeholders})"
```

```text
SELECT user_id, title FROM posts WHERE user_id IN (?, ?, ?)
```

ここで埋め込んでいるのは `?` の並びだけで、id の値そのものではありません。値は `execute` の第2引数で、今までどおりバインドします。取ってきた投稿を `user_id` ごとに分ければ、合計2回のSQLで済みます。Rails ではこれを `includes` というメソッドで自動的に行います（第18章）。これを「イーガーローディング（先読み）」と呼びます。

### トランザクション

ユーザーを作り、続けてそのユーザーの最初の投稿を作る処理を考えます。ユーザーの作成には成功し、投稿の作成で失敗したらどうなるでしょう。投稿のないユーザーだけが残ります。「両方成功するか、両方なかったことにするか」のどちらかにしたい場面です。

これを実現するのがトランザクションです。複数の操作を1つのまとまりとして扱い、途中で失敗したら全部を取り消します（ロールバック）。銀行の振込で、出金だけ成功して入金が失敗したら困るのと同じです。

```ruby
require "sqlite3"

db = SQLite3::Database.new(":memory:")
db.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")
db.execute("CREATE TABLE posts (id INTEGER PRIMARY KEY, user_id INTEGER, title TEXT NOT NULL)")

begin
  db.transaction do
    db.execute("INSERT INTO users (name) VALUES (?)", ["Alice"])
    db.execute("INSERT INTO posts (user_id, title) VALUES (?, ?)", [1, nil]) # NOT NULL 違反
  end
rescue SQLite3::ConstraintException => e
  puts "失敗: #{e.message}"
end

p db.execute("SELECT COUNT(*) FROM users")
```

```text
失敗: NOT NULL constraint failed: posts.title
[[0]]
```

投稿の INSERT が失敗したので、ブロック全体が取り消されました。先に成功していたユーザーの INSERT もなかったことになり、件数は 0 です。自作 ORM に組み込むなら、「どの単位でトランザクションを張るか」「バリデーション失敗（例外ではない）のときはどうするか」を設計する必要があります。

### ActiveRecord と比べる

ここで初めて、本物の ActiveRecord を見てみます。第18章で詳しく扱うので、ここでは比較表を作るのが目的です。Rails のガイドや、ActiveRecord のソースコード（GitHub の rails/rails リポジトリの `activerecord` フォルダ）を眺めてください。全部を理解する必要はありません。「自分が作った `where` に当たるものはどこか」を探すつもりで読みます。

| 観点 | 自作ORM | ActiveRecord |
|---|---|---|
| 取得 | all / find / where（AND のみ） | where, or, not, joins, order, limit, group など多数 |
| 関連 | なし（または素朴な実装） | has_many, belongs_to など。N+1対策の includes |
| 永続化 | save / destroy | save, update, destroy に加え、変更検知、コールバック |
| バリデーション | 空チェック程度 | presence, length, format, uniqueness など多数 |
| 型変換 | ほぼなし | DBの型と Ruby の型を相互に変換 |
| トランザクション | なし（または素朴） | transaction ブロック、save 時に自動で張る |
| スキーマ管理 | 手書きの CREATE TABLE | マイグレーション |
| 読みやすさ | 全部自分で書いたので追える | 巨大。魔法に見える部分が多い |

ActiveRecord が複雑なのは、作者が複雑なものを好んだからではありません。この章で出会った「簡単に見えて難しいこと」を、ひとつひとつ解決してきた結果です。「ActiveRecord はすごい」で終わらせず、「何を、なぜ引き受けているのか」を言えるようになってください。

### ORM を使うべきでない場面

ORM は便利ですが、万能ではありません。便利さと透明性（何が起きているかの見えやすさ）は、トレードオフの関係にあります。複雑な集計や、大量データの一括更新では、生SQLのほうが速く、意図も明確なことがあります。現場では、ORM を基本にしつつ、必要な場所では生SQLを書くのが普通です。その場合でも、値のバインドと許可リストのルールは変わりません。

> さらに深く：[Phase 5.9: ORMの限界とActiveRecordの理解](https://github.com/speee/training-web-app/blob/main/phase-5-9.md)

## 実務ではこうなる

現場で自分の ORM を作ることは、まずありません。Rails なら ActiveRecord を使います。それでも、この章で作った経験は毎日のように役立ちます。

1つ目は、ログに出るSQLを読めるようになることです。Rails の開発ログには、発行されたSQLが全部出ます。一覧画面を開いただけで同じ形のSQLが何十行も並んでいたら、それは N+1 です。この章で `trace` の出力を見た人は、すぐに気づけます。

2つ目は、SQLインジェクションを見抜けることです。Rails でも、`where("name = '#{params[:name]}'")` と書けば普通に脆弱になります。ORM を使っていれば安全、ではありません。コードレビューでこの形を見つけたら、必ず指摘してください。並び替えのカラムを画面から選ばせる機能も、許可リストが必要な典型例です。

3つ目は、`save` が false を返す意味が分かることです。ジュニアがやりがちな失敗に、`save` の戻り値を確かめずに「保存しました」と表示してしまうものがあります。バリデーションで失敗していても、画面上は成功したように見えます。

他にも、よくある失敗を挙げます。

- テストが開発用DBを使っていて、テストを流すたびに手作業のデータが消える
- `UPDATE` や `DELETE` の `WHERE` を書き忘れ、全件を書き換える。本番DBで起きると大事故になる
- 一覧画面で、ループの中で関連データを取得し、件数が増えると極端に遅くなる
- バリデーションを Controller に書き、管理画面からの保存で素通りする

## 演習

すべての演習は `~/workspace/ch16-orm/` で行います。各演習の終わりにテストがすべて通る状態でコミットしてください。テストは `ruby -Itest test/xxx_test.rb` や `rake test` など、第14章までと同じ方法で実行します。データはすべてダミー（`alice@example.com` など）にします。

### 演習16-1：生SQLのつらさとSQLインジェクション対策

- やること
  1. `users` テーブルを作る SQL を `db/schema.sql` に書き、テストの `setup` でメモリDBに流し込めるようにする
  2. Controller に生SQLを書いて、一覧（`/users`）・詳細（`/users/:id`）・名前検索（`/users/search?name=...`）を TDD で作る
  3. 存在しない id のときの振る舞い（例：404）を先にテストで決めて実装する
  4. 検索で `' OR '1'='1` と `O'Reilly` を使うテストを書く
  5. 「なぜつらいのか」を `docs/orm-notes.md` に自分の言葉で書く（Controller に何が漏れていたか、どこに重複があったか、次に何が欲しいか）
- **完了条件**
  - 一覧・詳細・404・検索のテストがすべて通る
  - 検索に `' OR '1'='1` を渡すと0件になり、`O'Reilly` を渡すとその1件だけが返るテストが通る
  - `grep -rn '#{' app/` の結果に、SQL文の中で入力値を埋め込んでいる行が1つもない
  - `docs/orm-notes.md` に4つの問い（つらさ・重複・戻り値の不自然さ・欲しい抽象化）への回答がある
- ヒント
  - テストのたびに新しいメモリDBを作れば、データの初期化を考えなくて済みます
  - `grep` で見つかった `#{` が、SQL以外（HTMLの組み立てなど）であれば問題ありません。その判断も記録してください

### 演習16-2：Model・Console・オブジェクト化

- やること
  1. `User.all` / `User.find(id)` のテストを書き、Controller のSQLを `User` に移す
  2. `find` で見つからないときの振る舞い（nil か例外か）をテストで固定する
  3. `bin/console` を作り、開発用DBにつないだ状態で IRB を起動できるようにする
  4. `User.all` が `User` のオブジェクトの配列を返し、`user.name` で値を取れるようにする
  5. Controller と View から `row[1]` や `["name"]` の形のアクセスをなくす
- **完了条件**
  - `users.first.name` と `User.find(1).email` を使うテストが通る
  - `bin/console` を起動し、`User.find(1).class` と入力すると `User` と表示される（その画面の記録を `docs/orm-notes.md` に貼る）
  - `grep -rn 'execute' app/controllers/` の結果が0件である
  - 演習16-1のSQLインジェクションのテストが、すべて通ったままである
- ヒント
  - Console で開発用DBを使い、テストではメモリDBを使う切り替え方法を先に決めましょう
  - 「データ」と「オブジェクト」の違いを、`docs/orm-notes.md` に一言で書いておくと後で役立ちます

### 演習16-3：Finder 共通化・永続化・バリデーション

- やること
  1. `posts` テーブルと `Post` モデルを追加し、`Post.all` のテストを通す
  2. 重複を親クラスに移し、テーブル名を Model ごとに扱えるようにする
  3. `where(ハッシュ)` を作る。値はバインドし、カラム名は Model ごとの許可リストで検査する
  4. `save`（INSERT / UPDATE の分岐）と `destroy` を TDD で作る
  5. 名前とメールが空なら `save` が false を返し、`errors` に理由が入るようにする
- **完了条件**（すべてテストで確かめる）
  - `User.where(name: "' OR '1'='1")` が空配列を返す
  - 許可リストにないキー（例：`"name = name OR 1=1 --"`）を `where` に渡すと例外になり、SQLが実行されない
  - `find` → 名前を `"' WHERE 1=1 --"` に変更 → `save` したとき、その行だけが変わり、他の行の名前は変わらない
  - `destroy` した後、対象の行だけが消え、他の行の件数は変わらない
  - `User.new(name: "", email: "").save` が false を返し、`errors` が2件になる。DBの件数は増えない
  - 同じSQLインジェクションのテストが `Post` でも通る
- ヒント
  - 「SQLが実行されないこと」は、例外が出たうえでDBの件数や中身が変わっていないことで確かめられます
  - id を INSERT 後にオブジェクトへ戻すのを忘れると、2回目の `save` で行が増えます。それもテストにしましょう

### 演習16-4：BaseModel と「自作ORMの限界レポート」

- やること
  1. 共通処理を `BaseModel` にまとめ、`User` と `Post` は差分だけを持つ形にする
  2. 演習16-1〜16-3のテストがすべて通ったままであることを確認する
  3. `User` ごとに `Post` を取る処理を書き、発行されたSQLの回数を数えるテストを書く（N+1を観察する）
  4. `docs/orm-limits.md` に限界レポートを書く。項目は「自作ORMの機能一覧」「足りない機能」「難しかった部分」「ActiveRecord との差分（本文の表を自分の言葉で埋め直す）」「ORMのメリット・デメリット」「ORMを使うべき場面・使うべきでない場面」の6つ
  5. 発展（任意）：`IN` を使って投稿をまとめて取る方法を実装し、SQLの回数が2回になるテストを通す。または `user.posts` のような関連を設計する
- **完了条件**
  - `User` と `Post` のクラス定義の中に、SQL文字列が1つも残っていない（`grep -n 'SELECT\|INSERT\|UPDATE\|DELETE' app/models/user.rb app/models/post.rb` が0件）
  - すべてのテスト（SQLインジェクションのテストを含む）が通る
  - N+1を観察するテストで、ユーザー3人のときにSQLが4回発行されることが確かめられている
  - `docs/orm-limits.md` に6項目すべてがあり、それぞれに自分のコードの具体例が1つ以上書かれている
- ヒント
  - SQLの回数は、`db.trace` のブロックで数を数えるだけで測れます
  - レポートは「すごい」「難しい」で終わらせず、「なぜ」を書きます。たとえば「where で OR を表現できないのは、引数がハッシュだから」のように

## 確認クイズ

**Q1.** 検索欄の入力値から `"... WHERE name = '#{name}'"` とSQLを組み立てました。入力値から `'` を取り除く処理を足せば、安全と言えますか。

**Q2.** 並び替えのカラムを、利用者が画面で選べるようにしたいです。`ORDER BY ?` にカラム名をバインドすれば安全に実現できますか。

**Q3.** `save` が INSERT と UPDATE を正しく切り替えるために、オブジェクトは何を知っている必要がありますか。目印として何を使うのが一般的ですか。

**Q4.** 「入力チェックは Controller ですればよい。Model のバリデーションは不要だ」という意見に、どう反論しますか。

**Q5.** ユーザー一覧画面で、各ユーザーの投稿数を表示したら、ユーザーが増えるにつれて画面が極端に遅くなりました。考えられる原因と、その対策の方向性を答えてください。

### 解答と解説

**A1.** 言えません。取り除く方式は「拒否リスト」の発想で、攻撃の書き方をすべて想定することはできません。しかも `O'Reilly` さんのような正当な値が壊れます。正しい対策は、プレースホルダ `?` を使い、値を別の引数としてバインドすることです。

**A2.** できません。プレースホルダは「値」の穴なので、カラム名は文字列の定数として扱われ、並び替えが効きません。エラーにもならないので気づきにくいです。カラム名は、開発者が決めた許可リストにあるものだけを受け付け、それ以外はSQLを実行する前に拒否します。

**A3.** 自分がDBに保存済みかどうか（状態）を知っている必要があります。一般的には id の有無を使います。id が nil なら新規なので INSERT、id があれば既存なので UPDATE です。INSERT の後に、DBが割り振った id をオブジェクトに戻すことも必要です。

**A4.** 保存の経路は1つではないからです。フォーム、管理画面、Console、一括取り込みなど、Controller を通らない経路もあります。Model に書けば、どの経路でも同じルールが必ず適用されます。データの正しさを守るのは Model の責任です。さらに DB 制約を最後の砦として併用します。

**A5.** N+1問題が考えられます。一覧の取得で1回、ユーザーごとに投稿を数えるSQLが1回ずつ発行され、合計 1 + N 回になっています。対策は、必要なデータを `IN` やJOIN、集計（`GROUP BY`）でまとめて取ることです。Rails なら `includes` などを使います。まずログで、実際に発行されたSQLの回数を確かめます。

## AIの使い方（この章）

この章では、AIは「SQLやRubyの文法を教えてくれる先生」と「設計の壁打ち相手」として使います。ORMの実装そのものを書かせてはいけません。

推奨する聞き方の例です。

- 「SQLite の sqlite3 gem で、プレースホルダを使って複数の値をバインドする書き方を教えてください。私の ORM のコードは見せずに、一般的な例でお願いします」
- 「私の ORM の `where` は、カラム名を許可リストで検査しています。次のテストで攻撃パターンの抜けがないか、レビューしてください（テストコードを貼る）」
- 「`save` の戻り値を true / false にするか、例外にするかで迷っています。それぞれの利点と欠点を、Rails の save と save! を例に説明してください」

やってはいけない使い方です。

- 「BaseModel を実装して」「where を書いて」と完成コードを出力させ、そのまま貼り付けること。この章の学びは、自分でつらさを感じて設計することそのものです
- AIが出したSQLを、プレースホルダを使っているか確かめずに採用すること。AIも文字列埋め込みのSQLを平気で出力します。採用する前に、必ず自分で確かめてください

## まとめ

- ORM は「便利だから」ではなく、生SQLと生の行データを扱うつらさから「必要になって」生まれた
- ORM の本質は、SQLを隠すことではなく、永続化されたデータをオブジェクトとして扱えるようにすること
- 値は必ずプレースホルダでバインドし、カラム名など構造に関わる部分は許可リストで制限する。バリデーションはその代わりにならない
- 抽象化は重複が見えてから行い、BaseModel を巨大にしない。N+1 とトランザクションは、ORM の便利さの裏にある難しさの代表例

## 用語

| 用語 | 意味 |
|---|---|
| ORM | Object-Relational Mapping。DBの行とプログラムのオブジェクトを対応づける仕組み |
| Model | データとそのルールを担当する部品。DBアクセスと正しさの保証を受け持つ |
| SQLインジェクション | 入力がSQLの命令として解釈され、意図しない問い合わせが実行される脆弱性 |
| プレースホルダ / バインド | SQLに `?` の穴を空け、値を別の引数として渡すこと。値がSQLの構造を変えない |
| 許可リスト | 受け付けてよいものを開発者が固定で列挙し、それ以外をすべて拒否する方式 |
| Console | アプリのコードを読み込んだ状態で起動する対話環境。Rails では `rails console` |
| IRB | Ruby 標準の対話環境 |
| Finder | `all` / `find` / `where` のような、DBからデータを探して取り出すメソッド |
| 永続化 | プログラムの外（DB）にデータを保存し、プログラム終了後も残るようにすること |
| バリデーション | 保存前にデータが業務ルールに合っているか確かめる仕組み |
| BaseModel | 各 Model に共通の処理をまとめた親クラス |
| メタプログラミング | プログラムがメソッドなどのプログラムを生成すること。`define_method` など |
| N+1問題 | 一覧で1回、各要素ごとに1回ずつSQLが発行され、件数に比例して遅くなる問題 |
| トランザクション | 複数のDB操作を1つのまとまりとして扱い、途中で失敗したら全部を取り消す仕組み |
| ActiveRecord | Rails に含まれる ORM |

---
[← 前の章](./15-sql-and-database.md) | [目次](../README.md) | [次の章 →](./17-crud-app.md)
