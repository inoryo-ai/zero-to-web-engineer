# 第6章 テストとTDD入門（Minitest）

> **この章のゴール**：Minitest でテストを書き、RED → GREEN → REFACTOR の順に小さな機能を作れるようになる。Rakefile と SimpleCov を備えたテスト環境を、自分で用意できるようになる。
> **目安時間**：読む 3時間 ＋ 演習 6時間
> **前提**：第5章まで

## この章で学ぶこと

- テストを書く理由を、「手で確かめる」場合と比べて説明できる
- Gemfile と Bundler で、プロジェクトに必要な Gem をそろえられる
- Minitest でテストを書き、実行結果（成功・Failure・Error）を読める
- RED → GREEN → REFACTOR の手順で、テストから先に機能を作れる
- Rakefile と test_helper.rb を用意し、`bundle exec rake test` でテストを実行できる
- SimpleCov でカバレッジを確認し、その数字の意味と限界を説明できる

## テストとは何か、なぜ書くのか

第5章の演習では、確認用のコードを書いて、出力を目で見比べました。これも立派な確認です。しかし、この方法には弱点があります。

- 確かめるたびに、出力を目で見比べる必要がある
- 確認用のコードを消すと、次に変更したときに同じ確認ができない
- 確認する項目が増えると、見落としが出る

**テスト**とは、「このコードはこう動くはず」という期待を、プログラムとして書いたものです。正確には**自動テスト**と呼びます。テストはコマンド1つで何度でも実行でき、期待と違えば機械が教えてくれます。

料理にたとえます。手で確かめるのは、作るたびに味見をすることです。テストは、「塩分は何グラム以下」「中心温度は75度以上」のような検査項目を、計器で測れるように書いておくことです。誰が作っても、何回作っても、同じ基準で確かめられます。

テストを書く理由は、大きく3つあります。

1. **変更が怖くなくなる**：コードを直したとき、前に動いていた機能が壊れていないかを、すぐに確かめられます。前は動いていた機能が、変更で壊れることを**デグレード**（デグレ、リグレッション）と呼びます。テストはデグレを見つける網になります。
2. **仕様が残る**：テストを読めば、「このメソッドは空のカートなら0を返す」のような約束がわかります。説明書が、動く形で残るのです。
3. **設計がよくなる**：テストを書こうとすると、「このメソッドは使いやすいか」を使う側の目線で考えることになります。テストしにくいコードは、たいてい使いにくいコードです。

この教材の第11章以降は、すべてテストを先に書いて進めます。元にした研修（training-web-app）も同じ方針です。この章は、そのための土台作りです。

## 準備：Gemfile と Bundler

テストには **Minitest** という Gem を使います。Minitest は Ruby の標準的なテスト用ライブラリで、Rails でも最初から使われています。あわせて、テストをまとめて実行する **Rake**、テストが通ったコードの範囲を測る **SimpleCov** も使います。

まず作業フォルダを作ります。

```bash
mkdir -p ~/workspace/ch06-testing
cd ~/workspace/ch06-testing
mkdir lib test
```

次に、`Gemfile` という名前のファイルを作り、次の内容を書きます。

```ruby
source "https://rubygems.org"

gem "minitest"
gem "rake"
gem "simplecov", require: false
```

- `source` は、Gem をどこから取ってくるかの指定です。
- `gem "名前"` の行が、このプロジェクトで使う Gem の一覧です。
- `require: false` は、「Bundler に自動で読み込ませない」という指定です。SimpleCov は読み込む順番が大切なので、あとで自分で読み込みます。

Gemfile を書いたら、Bundler で Gem をそろえます。

```bash
bundle install
```

成功すると、`Gemfile.lock` というファイルができます。中身の一部はこのようになっています（版の数字は、実行した時期や環境で変わります）。

```text
GEM
  remote: https://rubygems.org/
  specs:
    docile (1.4.1)
    minitest (5.25.4)
    rake (13.4.2)
    simplecov (0.22.0)
      docile (~> 1.1)
      simplecov-html (~> 0.11)
      simplecov_json_formatter (~> 0.1)
```

**Gemfile.lock** は、「実際にどの版をそろえたか」の記録です。Gemfile が「材料の買い物リスト」なら、Gemfile.lock は「実際に買った商品のレシート」です。同じ Gemfile.lock を使えば、他の人のPCでも同じ版がそろいます。だから Gemfile.lock は Git にコミットします。

SimpleCov が依存している `docile` なども、自動でそろえられています。Gem が別の Gem を必要とすることを**依存関係**と呼びます。

最後に、`bundle exec` について説明します。以後、テストは `bundle exec rake test` のように、頭に `bundle exec` を付けて実行します。これは「Gemfile.lock に書かれた版の Gem を使って実行せよ」という意味です。PCに同じ Gem の別の版が入っていても、プロジェクトで決めた版が使われます。

> **Ruby 3.3 以降の注意**：minitest と rake は、Ruby 本体と一緒にインストールされる「バンドルされた Gem」です。それでも Bundler を使う場合は、Gemfile に書かないと `bundle exec` の中で使えません。必ず Gemfile に書いてください。

## 最初のテストを書く

`test/first_test.rb` を作ります。わざと1つだけ間違った期待を書いています。

```ruby
require "minitest/autorun"

class FirstTest < Minitest::Test
  def test_addition
    assert_equal 4, 2 + 2
  end

  def test_upcase
    assert_equal "HELLO", "hello".upcase
  end

  def test_wrong_expectation
    assert_equal 5, 2 + 2
  end
end
```

```console
$ bundle exec ruby test/first_test.rb
Run options: --seed 17730

# Running:

.F.

Finished in 0.002375s, 1263.1579 runs/s, 1263.1579 assertions/s.

  1) Failure:
FirstTest#test_wrong_expectation [test/first_test.rb:13]:
Expected: 5
  Actual: 4

3 runs, 3 assertions, 1 failures, 0 errors, 0 skips
```

コードの読み方から説明します。

- `require "minitest/autorun"` で Minitest を読み込みます。`autorun` は「ファイルの最後まで読んだら、テストを自動で実行する」という意味です。
- テストは `Minitest::Test` を継承したクラスに書きます。第5章で学んだ継承です。テストを書くための便利なメソッドを、親クラスから引き継いでいます。
- `test_` で始まるメソッドが、1つ1つのテストです。Minitest はこの名前を手がかりにテストを探します。`test_` を付け忘れると、そのメソッドは実行されません。
- `assert_equal 期待する値, 実際の値` は、「2つが等しいはず」と主張するメソッドです。このような確認用のメソッドを**アサーション**と呼びます。

次に、出力の読み方です。

- `Run options: --seed 17730` は、テストを実行した順番の「種」です。Minitest は、毎回テストの順番をランダムに入れ替えます。理由は後で説明します。数字は実行のたびに変わります。
- `.F.` は、1つ1つのテストの結果です。`.` は成功、`F` は Failure（期待と違った）、`E` は Error（途中で例外が起きた）、`S` はスキップです。
- `1) Failure:` の下に、失敗したテストの名前と場所（13行目）があります。
- `Expected: 5` と `Actual: 4` は、「5を期待したが、実際は4だった」という意味です。
- 最後の行が集計です。3つ実行し、アサーションは3回、失敗が1つでした。

最後の行を確認したら、`test_wrong_expectation` を消すか、`5` を `4` に直してください。もう一度実行すると `...` と表示され、`0 failures` になります。

`assert_equal` は、**期待する値を先に**書きます。逆に書いてもテストの成否は同じですが、失敗したときの `Expected` と `Actual` が入れ替わって表示されます。読む人が混乱するので、必ず「期待、実際」の順にしてください。

### テストの基本形：準備・実行・確認

1つのテストは、たいてい次の3段階でできています。

1. **準備**：テストに必要なオブジェクトやデータを用意する
2. **実行**：テストしたいメソッドを呼ぶ
3. **確認**：結果が期待どおりかをアサーションで確かめる

英語では Arrange・Act・Assert と呼ばれます。テストを書くときに迷ったら、この3つの順に並べてみてください。

### よく使うアサーション

| アサーション | 確かめること | 例 |
|---|---|---|
| `assert_equal 期待, 実際` | 2つが等しい | `assert_equal 3, list.size` |
| `assert 式` | 式が真（nil と false 以外） | `assert item.done?` |
| `refute 式` | 式が偽（nil か false） | `refute item.done?` |
| `assert_nil 式` | 式が nil | `assert_nil hash[:missing]` |
| `assert_includes 集まり, 要素` | 配列などに要素が含まれる | `assert_includes names, "花子"` |
| `assert_match 正規表現, 文字列` | 文字列がパターンに合う | `assert_match(/円\z/, text)` |
| `assert_raises(例外クラス) { ... }` | ブロックの中で、その例外が起きる | 後の節で説明 |

`nil` を確かめるときは、`assert_equal nil, x` ではなく `assert_nil x` を使ってください。新しい Minitest では、`assert_equal nil, ...` は警告やエラーになります。

### Failure と Error の違い

テストの失敗には2種類あります。

- **Failure（F）**：アサーションまでたどり着いたが、結果が期待と違った。
- **Error（E）**：アサーションにたどり着く前に、例外が起きて止まった。

Failure は「答えが違う」、Error は「答えを出す前に倒れた」です。Error のときは、メッセージの下にスタックトレース（どこで例外が起きたかの記録）が表示されます。読み方は第7章で詳しく学びます。

## TDD：RED → GREEN → REFACTOR

**TDD**（テスト駆動開発、Test-Driven Development）は、テストを先に書いてから実装する開発の進め方です。次の3つの手順を、小さく何度も繰り返します。

1. **RED（レッド）**：これから作る機能のテストを書く。まだ実装していないので、テストは失敗する。失敗することを実際に確かめる。
2. **GREEN（グリーン）**：テストが通る最小限のコードを書く。きれいさより、まず通すことを優先する。
3. **REFACTOR（リファクタ）**：テストが通る状態を保ったまま、コードを読みやすく整える。

名前の由来は、多くのテストツールが失敗を赤、成功を緑で表示することです。

### なぜテストを先に書くのか

「実装してからテストを書けばよいのでは」と思うかもしれません。テストを先に書く理由は3つあります。

- **テストが本当に失敗を見つけられるかを確かめられる**：実装の後にテストを書くと、最初から緑になります。そのテストが、本当にバグを見つけられるのかはわかりません。たとえば、アサーションを書き忘れたテストも緑になります。先に赤を見ておけば、「このテストは壊れたものを見分けられる」と確かめられます。
- **作るものがはっきりする**：テストを書くには、「何を渡すと、何が返るのか」を決める必要があります。作り始める前に、ゴールが決まります。
- **一歩が小さくなる**：1回に考えるのは、テスト1つ分だけです。大きな機能を一気に書いて、どこが悪いかわからなくなる事態を防げます。

正直に言うと、TDD は最初は遠回りに感じます。特に GREEN で「わざと単純なコード」を書く場面は、もどかしく感じるはずです。それでも、この章では手順を省かずに体験してください。第13章で並行処理を扱うとき、この「小さく確かめながら進む」習慣が効いてきます。

### TDD の実例：買い物カート

例として、買い物カートの合計金額を計算する `Cart` クラスを TDD で作ります。第5章で学んだクラスと例外を使います。

**1周目の RED：空のカートは合計0円**

まずテストだけを書きます。`test/cart_test.rb` を作ってください。

```ruby
require "minitest/autorun"
require_relative "../lib/cart"

class CartTest < Minitest::Test
  def test_total_of_empty_cart_is_zero
    cart = Cart.new
    assert_equal 0, cart.total
  end
end
```

`lib/cart.rb` はまだありません。実行してみます。

```console
$ bundle exec ruby test/cart_test.rb
test/cart_test.rb:2:in `require_relative': cannot load such file -- /Users/you/workspace/ch06-testing/lib/cart (LoadError)
	from test/cart_test.rb:2:in `<main>'
```

（パスの部分は環境によって違います。）

ファイルがないので LoadError です。これも RED の一種です。空の `lib/cart.rb` を作って、もう一度実行します。

```console
$ touch lib/cart.rb
$ bundle exec ruby test/cart_test.rb
Run options: --seed 33765

# Running:

E

Finished in 0.000579s, 1727.1157 runs/s, 0.0000 assertions/s.

  1) Error:
CartTest#test_total_of_empty_cart_is_zero:
NameError: uninitialized constant CartTest::Cart
    test/cart_test.rb:6:in `test_total_of_empty_cart_is_zero'

1 runs, 0 assertions, 0 failures, 1 errors, 0 skips
```

`Cart` というクラスがない、という Error です。`lib/cart.rb` に空のクラスを書きます。

```ruby
class Cart
end
```

```console
$ bundle exec ruby test/cart_test.rb
（前略）
  1) Error:
CartTest#test_total_of_empty_cart_is_zero:
NoMethodError: undefined method `total' for #<Cart:0x00007fdd1013f1b0>
    test/cart_test.rb:7:in `test_total_of_empty_cart_is_zero'

1 runs, 0 assertions, 0 failures, 1 errors, 0 skips
```

エラーの内容が「クラスがない」から「`total` メソッドがない」に変わりました。エラーメッセージが、次に何を書けばよいかを教えてくれています。この「エラーに導かれて一歩ずつ進む」感覚が TDD の特徴です。

**1周目の GREEN：仮実装**

テストを通す最小限のコードを書きます。

```ruby
class Cart
  def total
    0
  end
end
```

```console
$ bundle exec ruby test/cart_test.rb
Run options: --seed 16245

# Running:

.

Finished in 0.000406s, 2463.0542 runs/s, 2463.0542 assertions/s.

1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

緑になりました。「`0` を返すだけなんて、ずるい」と感じたかもしれません。これは**仮実装**と呼ばれる、TDD のれっきとした手法です。今あるテストは「空なら0」しか要求していないので、`0` を返せば十分です。本物の計算が必要になるのは、それを要求するテストを書いたときです。

**2周目の RED：商品を入れたら合計が変わる**

次のテストを追加します。

```ruby
  def test_total_is_sum_of_price_times_quantity
    cart = Cart.new
    cart.add("りんご", 120, 2)
    cart.add("みかん", 80, 3)
    assert_equal 480, cart.total
  end
```

実行すると、`add` がないという Error になります。そこで、何もしない `add` を足してみます。

```ruby
class Cart
  def add(name, price, quantity)
  end

  def total
    0
  end
end
```

```console
$ bundle exec ruby test/cart_test.rb
Run options: --seed 61405

# Running:

F.

Finished in 0.000661s, 3025.7186 runs/s, 3025.7186 assertions/s.

  1) Failure:
CartTest#test_total_is_sum_of_price_times_quantity [test/cart_test.rb:14]:
Expected: 480
  Actual: 0

2 runs, 2 assertions, 1 failures, 0 errors, 0 skips
```

今度は Error ではなく Failure です。メソッドは呼べるようになり、「答えが違う」段階まで来ました。2つ目のテストが加わったことで、`0` を返すだけの仮実装では通らなくなりました。このように、テストを増やして実装を本物に近づけていく方法を**三角測量**と呼びます。

**2周目の GREEN：本当に計算する**

```ruby
class Cart
  def initialize
    @items = []
  end

  def add(name, price, quantity)
    @items << { name: name, price: price, quantity: quantity }
  end

  def total
    sum = 0
    @items.each do |item|
      sum += item[:price] * item[:quantity]
    end
    sum
  end
end
```

```console
$ bundle exec ruby test/cart_test.rb
（前略）
2 runs, 2 assertions, 0 failures, 0 errors, 0 skips
```

**2周目の REFACTOR：テストに守られて整える**

`total` は、第4章で学んだ `sum` とブロックで短く書けます。

```ruby
  def total
    @items.sum { |item| item[:price] * item[:quantity] }
  end
```

```console
$ bundle exec ruby test/cart_test.rb
（前略）
2 runs, 2 assertions, 0 failures, 0 errors, 0 skips
```

書き換えた後も、テストは緑のままです。動きを変えずにコードの形を整えることを**リファクタリング**と呼びます。テストがあるから、「書き換えても壊れていない」と自信を持てます。テストがなければ、毎回手で確かめ直すことになります。

リファクタリングでは、**テストが緑のときだけ**コードを変えます。赤のまま形を整えようとすると、「直したから赤なのか、もともと赤なのか」がわからなくなります。

**3周目：例外が起きることをテストする**

最後に、「数量が0以下なら ArgumentError」という仕様を足します。例外が起きることは `assert_raises` で確かめます。同時に、テストの重複も整理します。

```ruby
require "minitest/autorun"
require_relative "../lib/cart"

class CartTest < Minitest::Test
  def setup
    @cart = Cart.new
  end

  def test_total_of_empty_cart_is_zero
    assert_equal 0, @cart.total
  end

  def test_total_is_sum_of_price_times_quantity
    @cart.add("りんご", 120, 2)
    @cart.add("みかん", 80, 3)
    assert_equal 480, @cart.total
  end

  def test_add_rejects_zero_quantity
    error = assert_raises(ArgumentError) do
      @cart.add("りんご", 120, 0)
    end
    assert_equal "数量は1以上にしてください", error.message
  end
end
```

- `setup` は特別な名前のメソッドです。**各テストの前に毎回**実行されます。どのテストも新しい `@cart` から始まるので、テスト同士が影響し合いません。
- `assert_raises(ArgumentError) { ... }` は、ブロックの中で ArgumentError が起きれば成功です。起きた例外を返すので、メッセージも確かめられます。

```console
$ bundle exec ruby test/cart_test.rb
Run options: --seed 59020

# Running:

..F

Finished in 0.000593s, 5059.0219 runs/s, 5059.0219 assertions/s.

  1) Failure:
CartTest#test_add_rejects_zero_quantity [test/cart_test.rb:20]:
ArgumentError expected but nothing was raised.

3 runs, 3 assertions, 1 failures, 0 errors, 0 skips
```

「ArgumentError が起きるはずなのに、何も起きなかった」という RED です。`add` の先頭にチェックを1行足すと、GREEN になります。

```ruby
  def add(name, price, quantity)
    raise ArgumentError, "数量は1以上にしてください" if quantity < 1

    @items << { name: name, price: price, quantity: quantity }
  end
```

```console
$ bundle exec ruby test/cart_test.rb
（前略）
3 runs, 4 assertions, 0 failures, 0 errors, 0 skips
```

ここまでが TDD の1サイクルの体験です。流れを整理します。

| 段階 | やったこと | 確かめたこと |
|---|---|---|
| RED | テストを書いて実行した | テストが、期待どおりの理由で失敗する |
| GREEN | 通る最小限のコードを書いた | テストが通る |
| REFACTOR | 形を整えた | テストが通ったまま |

RED で大切なのは「**期待どおりの理由で**失敗する」ことです。打ち間違いによる NoMethodError で赤くなっていても、それは確かめたい失敗ではありません。失敗のメッセージを毎回読み、「今はこの理由で赤いはず」と確認してください。

## Rakefile と test_helper.rb でテスト環境を整える

テストファイルが増えると、1つずつ `ruby test/xxx_test.rb` と打つのは手間です。また、どのテストファイルにも同じ準備（`require "minitest/autorun"` など）を書くことになります。そこで、次の3つを用意して、テスト環境を整えます。

```text
ch06-testing/
├── Gemfile
├── Gemfile.lock
├── Rakefile
├── lib/
│   └── cart.rb
└── test/
    ├── test_helper.rb
    └── cart_test.rb
```

### Rakefile：テストの実行方法を1つに決める

**Rake** は、Ruby で書く「作業の手順書」の道具です。手順書のファイルを **Rakefile** と呼びます。プロジェクトのフォルダの一番上に、次の `Rakefile` を作ります。

```ruby
require "rake/testtask"

Rake::TestTask.new do |t|
  t.libs << "test"
  t.pattern = "test/**/*_test.rb"
end

task default: :test
```

- `Rake::TestTask.new` で、`test` という名前の作業（タスク）を作ります。
- `t.libs << "test"` は、`test` フォルダをロードパスに追加します。第5章で学んだ、`require` が探す本棚の一覧です。
- `t.pattern` は、実行するテストファイルの探し方です。`test/**/*_test.rb` は「test フォルダの下の、どの深さにあってもよい、`_test.rb` で終わるファイル」という意味です。
- `task default: :test` は、タスク名を省略して `rake` とだけ打ったときに、`test` を実行するという意味です。

`bundle exec rake -T` で、使えるタスクの一覧を確かめられます。

```console
$ bundle exec rake -T
rake test  # Run tests
```

### test_helper.rb：テストの共通準備をまとめる

`test/test_helper.rb` を作ります。

```ruby
require "simplecov"

SimpleCov.start do
  add_filter "/test/"
end

require "minitest/autorun"

$LOAD_PATH.unshift(File.expand_path("../lib", __dir__))
```

- 最初の4行は SimpleCov の設定です。次の節で説明します。
- `require "minitest/autorun"` は、各テストファイルから、ここに引っ越しました。
- 最後の行は、`lib` フォルダをロードパスの先頭に追加します。`File.expand_path("../lib", __dir__)` は、「このファイルのある `test` フォルダから1つ上がった `lib`」の絶対パスです。これで、テストから `require "cart"` と短く書けます。

テストファイルの先頭は、次のように変わります。

```ruby
require_relative "test_helper"
require "cart"

class CartTest < Minitest::Test
  # 以下は同じ
```

すべてのテストファイルの先頭で `require_relative "test_helper"` を書く、というのが約束です。Rails のプロジェクトでも、`test/test_helper.rb` が同じ役割を持っています。

### bundle exec rake test で実行する

```console
$ bundle exec rake test
Run options: --seed 39529

# Running:

...

Finished in 0.000718s, 4178.2730 runs/s, 5571.0306 assertions/s.

3 runs, 4 assertions, 0 failures, 0 errors, 0 skips
Coverage report generated for Unit Tests to /Users/you/workspace/ch06-testing/coverage.
Line Coverage: 90.0% (9 / 10)
```

（パスは環境によって違います。この例の `lib/cart.rb` には、テストしていない `item_names` メソッドを足してあります。そのため 90% になっています。）

`bundle exec rake` とだけ打っても、同じ結果になります。テストが1つでも失敗すると、`rake` は「失敗した」という合図（0以外の終了コード）を返します。第22章で学ぶ CI（自動でテストを実行する仕組み）は、この合図を見て成功・失敗を判断します。

なぜ毎回 `bundle exec rake test` という同じコマンドを使うのでしょうか。実行方法がチームで統一されていれば、新しく入った人も迷いません。CI でも同じコマンドを使えます。「自分のPCではこう実行していた」という食い違いがなくなります。

### 一部のテストだけを実行する

テストが増えてくると、今触っているテストだけを実行したくなります。

```console
$ bundle exec rake test TEST=test/cart_test.rb
（前略）
3 runs, 4 assertions, 0 failures, 0 errors, 0 skips

$ bundle exec ruby -Itest test/cart_test.rb -n test_add_rejects_zero_quantity
（前略）
1 runs, 2 assertions, 0 failures, 0 errors, 0 skips
```

- `TEST=ファイル名` で、1つのファイルだけを実行します。
- `ruby -Itest ファイル -n テスト名` で、1つのテストだけを実行します。`-Itest` は、`test` フォルダをロードパスに加える指定です。`require_relative "test_helper"` の中で `require` を使うときに必要になります。

作業中は一部だけを実行して素早く確かめ、区切りでは必ず全体（`bundle exec rake test`）を実行してください。一部だけ緑でも、他のテストを壊しているかもしれません。

## SimpleCov でカバレッジを見る

**カバレッジ**とは、テストを実行したときに、コードのどの行が実行されたかの割合です。「テストがコードのどこを通ったか」の地図と考えてください。

test_helper.rb の `SimpleCov.start` が、これを測っています。`add_filter "/test/"` は、テストファイル自身を測定対象から外す設定です。測りたいのはテストではなく、`lib` の中の本体だからです。

SimpleCov は、**テスト対象のコードより先に**読み込む必要があります。読み込む前に実行されたコードは、測定できないからです。test_helper.rb の先頭に書いたのはこのためです。テストファイル側でも、`require "cart"` より前に `require_relative "test_helper"` を書きます。

テストを実行すると、`coverage/index.html` ができます。ブラウザで開いてみましょう。

```bash
open coverage/index.html     # macOS
explorer.exe coverage/index.html   # WSL の場合の一例
```

ファイルごとの割合と、行ごとの色分けが表示されます。緑の行は実行された行、赤の行は一度も実行されなかった行です。先ほどの例では、`item_names` メソッドの中の行が赤くなります。

```ruby
  def item_names
    @items.map { |item| item[:name] }   # ← 赤：どのテストも通っていない
  end
```

赤い行は、「ここが壊れても、テストは気づかない」という場所です。テストを足すか、そもそも不要なコードではないかを考えるきっかけになります。

### カバレッジ100%は「正しい」の証明ではない

カバレッジには大きな限界があります。**ある行を実行したことと、その行の結果を正しく確かめたことは別**です。

たとえば、次のテストは `total` の行を実行するので、カバレッジは上がります。

```ruby
  def test_total_runs
    @cart.add("りんご", 120, 2)
    @cart.total
  end
```

しかし、アサーションがないので、`total` が 999 を返しても緑のままです。カバレッジは「テストが通っていない場所」を見つける道具であり、「テストが十分である」ことの証明にはなりません。この研修でも、100%を目標にはしません。数字を上げることより、「自分のテストがどこを通っているか」を見ることが目的です。

### coverage フォルダはコミットしない

`coverage/` は、テストを実行するたびに作り直される生成物です。Git にコミットする必要はありません。第2章で学んだ `.gitignore` に追加します。

```gitignore
coverage/
```

`Gemfile.lock` はコミットし、`coverage/` はコミットしない。この区別を覚えておいてください。

## よいテストを書くためのコツ

TDD の手順に慣れてきたら、テストそのものの質にも目を向けます。

- **1つのテストでは、1つの振る舞いを確かめる**：テストが失敗したとき、名前を見ただけで何が壊れたかわかるようにします。
- **テスト名で仕様を語る**：`test_1` ではなく `test_add_rejects_zero_quantity` のように、何を確かめるかを名前にします。英語が苦手なら、短い英単語を並べるだけで十分です。
- **テスト同士を独立させる**：あるテストの結果に、別のテストが頼ってはいけません。Minitest が毎回順番をランダムにするのは、隠れた依存関係をあぶり出すためです。順番によって成功したり失敗したりする場合は、`--seed` の数字を使って同じ順番を再現できます（`bundle exec rake test TESTOPTS="--seed=59020"`）。
- **結果が毎回同じになるようにする**：現在時刻や乱数に頼ると、実行するたびに結果が変わることがあります。時々だけ失敗するテストは、信用されなくなります。
- **公開されたメソッドを通して確かめる**：private メソッドを直接テストするより、外から呼べるメソッドの結果で確かめます。中身を書き換えても、テストを直さずに済みます。
- **ダミーデータを使う**：テストに実在の人の名前やメールアドレスを書かないでください。`テスト太郎`、`user1@example.com` のように、明らかに架空のものを使います。テストコードも Git に残り、他の人に共有されるからです。

## 実務ではこうなる

現場では、テストはコードの一部として扱われます。プルリクエスト（変更の提案。第24章で扱います）にテストがないと、レビューで「テストを追加してください」と言われるのが普通です。

- **CI で必ず実行される**：プルリクエストを出すと、CI がテストを自動で実行します。1つでも赤ければ、多くのチームでは取り込めません（第22章）。
- **Rails では test フォルダにテストがある**：Rails アプリには最初から `test/` と `test/test_helper.rb` があります。RSpec という別のテストツールを使う会社も多いですが、「準備・実行・確認」の考え方は同じです。
- **バグ修正はテストから**：不具合の報告を受けたら、まずその不具合を再現するテストを書きます（RED）。直すと緑になり（GREEN）、同じ不具合が二度と戻ってこないことを保証できます。

ジュニアがやりがちな失敗も挙げておきます。

- **テストを後回しにして、最後にまとめて書く**：書いた時点で全部緑になるので、本当に壊れたものを見つけられるか確かめていません。
- **赤いテストを消す・スキップして先に進む**：テストが赤いのは、何かを教えてくれているサインです。理由を理解しないまま消してはいけません。
- **カバレッジの数字を上げるためだけのテスト**：アサーションのないテストは、数字だけを上げて、安心感を偽ります。
- **一部のテストしか実行せずにプッシュする**：手元の1ファイルだけ緑でも、他を壊していることがあります。

## 演習

演習のコードは `~/workspace/ch06-testing/` に作ります。本文の Cart の例を手元で試した場合は、そのまま同じフォルダを使って構いません。

### 演習6-1：テスト環境を自分で作る

- `~/workspace/ch06-testing/` に、本文の構成どおり `Gemfile`・`Rakefile`・`test/test_helper.rb` を作ってください。
- 動作確認用に、`test/sample_test.rb` を作ります。中身は「`require_relative "test_helper"` を読み込み、`assert true` を1つだけ持つテスト」です。
- `.gitignore` に `coverage/` を追加します。`~/workspace/` 全体で1つのリポジトリにしている場合は、`~/workspace/.gitignore` に書いても構いません。
- **完了条件**：
  - `bundle exec rake test` の最後の集計行が `1 runs, 1 assertions, 0 failures, 0 errors, 0 skips` になる（本文の Cart も置いた場合は、その分 runs が増える）。
  - `coverage/index.html` が作られ、ブラウザで開ける。
  - `git status` を実行したとき、`coverage/` が表示されない。`Gemfile.lock` は表示される（またはコミット済みである）。
- ヒント：`bundle install` がうまくいかないときは、エラーメッセージの最後の数行を読み、第7章の「調べ方」の節も参考にしてください。

### 演習6-2：TDD で金額の表示を作る

- `lib/yen.rb` に、`Yen.format(整数)` というクラスメソッドを TDD で作ってください。仕様は次のとおりです。

| 入力 | 戻り値 |
|---|---|
| `0` | `"0円"` |
| `980` | `"980円"` |
| `1000` | `"1,000円"` |
| `1234567` | `"1,234,567円"` |
| `-1500` | `"-1,500円"` |
| `"1000"`（文字列） | `ArgumentError` を発生させる |

- 進め方のルール：
  1. 表の上から1行ずつ、テストを1つ書く → `bundle exec rake test` で RED を確認する → 実装して GREEN → 必要ならリファクタ、を繰り返す。
  2. RED を確認したら、テストだけをコミットする（例：`git commit -m "RED: 1000円にカンマが付くテスト"`）。GREEN になったら実装をコミットする。
  3. 最初の `0円` は、仮実装（定数を返すだけ）から始める。
- **完了条件**：
  - `test/yen_test.rb` に6つ以上のテストがあり、`bundle exec rake test` がすべて成功する。
  - `git log --oneline` で、`RED:` で始まるコミットと、その実装のコミットが交互に並んでいる。
  - SimpleCov のレポートで、`lib/yen.rb` が 100% になっている。
- ヒント：数字を3桁ごとに区切る方法はいくつかあります。たとえば、文字列を逆順にして3文字ずつ区切る方法があります。どの方法でも、テストが通れば正解です。REFACTOR の段階で、自分のコードを読み直して、もっと読みやすい書き方がないか考えてみてください。負の数は、符号を先に取り分けて考えると整理しやすくなります。

### 演習6-3：第5章の ToDo にテストを後付けする

- 第5章の演習5-4で作った `lib/todo_item.rb` と `lib/todo_list.rb` を、`~/workspace/ch06-testing/lib/` にコピーしてください。
- `test/todo_item_test.rb` と `test/todo_list_test.rb` を作り、次を確かめるテストを書きます。
  - TodoItem：作った直後は未完了、`complete!` で完了、`to_s` の両方の形
  - TodoList：`add` と `to_s`、`complete` の正常系、存在しない番号（0・負の数・大きすぎる数）で `TodoNotFoundError`、空白だけのタイトルで `ArgumentError`
  - 保存と読み込み：`save` してから `load` すると、タイトルと完了状態が元に戻る
- 保存と読み込みのテストでは、本物の `todos.txt` を上書きしてはいけません。テスト用の一時ファイルを使ってください。
- **完了条件**：
  - `bundle exec rake test` がすべて成功する。
  - SimpleCov のレポートで、`todo_item.rb` と `todo_list.rb` がどちらも 100% になっている。
  - **テストが壊れたものを見つけられるかの確認**：`todo_list.rb` の範囲チェックの条件を1か所わざと壊す（例：`<` を `<=` にする）と、少なくとも1つのテストが失敗する。確認できたら元に戻す。
- ヒント：一時ファイルには、Ruby 付属の `tmpdir` ライブラリが使えます（`require "tmpdir"` して `Dir.mktmpdir` を使う）。使い方は公式ドキュメントで調べてください。後付けのテストは最初から緑になりがちです。だからこそ、最後の「わざと壊す」確認が大切です。

### 演習6-4：TDD で ToDo に機能を足す

- 演習6-3のコードに、次の2つの機能を TDD で追加してください。どちらも、テストを書いて RED を確認してから実装します。
  - `TodoList#remove(number)`：1から数えて `number` 番目を削除する。存在しない番号なら `TodoNotFoundError`。
  - `TodoList#pending`：未完了の TodoItem だけの配列を返す。
- **完了条件**：
  - 2つの機能それぞれについて、正常系と異常系のテストがある（`pending` は「全部完了済みなら空配列」を異常系の代わりにしてよい）。
  - `bundle exec rake test` がすべて成功し、カバレッジは両ファイルとも 100% のまま。
  - `git log` で、テストのコミットが実装のコミットより先にある。
- ヒント：`remove` と `complete` には、「番号が範囲内か調べる」という共通の処理が出てくるはずです。2回同じことを書いたら、REFACTOR の段階で private メソッドにまとめましょう。まとめた後もテストが緑なら、リファクタリング成功です。

### 演習6-5（発展）：テストの順番に依存するテストを観察する

- 新しく `test/order_dependency_test.rb` を作り、**わざと**順番に依存するテストを書いてください。たとえば、1つ目のテストがグローバル変数（`$count` など）に値を入れ、2つ目のテストがその値を前提にする、という形です。
- `bundle exec rake test` を何度か実行し、成功したり失敗したりすることを観察します。失敗したときの `--seed` の値を控え、`bundle exec rake test TESTOPTS="--seed=控えた値"` で同じ失敗を再現してください。
- **完了条件**：
  - 失敗を `--seed` 付きで再現できた。そのコマンドと出力を、学習メモに貼った。
  - 「なぜ Minitest はテストの順番をランダムにするのか」を、3文以内で学習メモに書いた。
  - 観察が終わったら、このテストファイルを削除し、`bundle exec rake test` が安定して成功する。
- ヒント：第13章では、複数の処理が同時に動くときの「成功したり失敗したりする」問題を扱います。原因は違いますが、「たまたま通った」を成功と思わないという姿勢は同じです。

## 確認クイズ

**Q1.** TDD の RED の段階で、テストが失敗することを実際に確かめるのはなぜですか。

**Q2.** 次の出力の `E` と `F` は、それぞれどんな状態を表していますか。

```text
.E.F
```

**Q3.** `assert_equal user.name, "テスト太郎"` というテストには、どんな問題がありますか。

**Q4.** SimpleCov でカバレッジが 100% になりました。このコードにバグがないと言えますか。

**Q5.** test_helper.rb で、`require "simplecov"` と `SimpleCov.start` をファイルの先頭に書くのはなぜですか。

**Q6.** `Gemfile.lock` と `coverage/` は、それぞれ Git にコミットすべきですか。

### 解答と解説

**A1.** テストが本当に「壊れたもの」を見分けられるか確かめるためです。最初から緑のテストは、アサーションの書き忘れなどで、何も確かめていない可能性があります。また、失敗のメッセージが期待どおりの理由かを確認することで、テストの書き間違いにも気づけます。

**A2.** `E` は Error で、アサーションにたどり着く前に例外が起きたことを表します。`F` は Failure で、アサーションまで進んだが結果が期待と違ったことを表します。`.` は成功です。

**A3.** 引数の順番が逆です。`assert_equal` は「期待する値、実際の値」の順に書きます。成否は変わりませんが、失敗したときに `Expected` と `Actual` が入れ替わって表示され、読み間違いの原因になります。正しくは `assert_equal "テスト太郎", user.name` です。

**A4.** 言えません。カバレッジは「その行が実行されたか」しか測りません。結果を正しく確かめたかどうかは測りません。アサーションのないテストでも、カバレッジは上がります。

**A5.** SimpleCov は、読み込まれた後に実行されたコードしか測定できないからです。テスト対象のコードを先に読み込むと、その部分が測定から漏れます。

**A6.** `Gemfile.lock` はコミットします。誰のPCでも同じ版の Gem をそろえるための記録だからです。`coverage/` はコミットしません。テストのたびに作り直される生成物なので、`.gitignore` に入れます。

## AIの使い方（この章）

この章の学習の中心は、テストを先に書き、赤を自分の目で確かめる体験です。AIには、テストの「レビュー役」をお願いするのが効果的です。

**推奨する聞き方の例**

- 「次の Minitest の出力を読んで、どのテストが、どんな理由で失敗しているか説明してください。修正コードは出さないでください。（出力を貼る）」
- 「`Yen.format` について、私は次の6つのテストケースを考えました。見落としている境界の値や、確かめるべきケースがあれば、ケースの名前だけを挙げてください。」
- 「私の理解では『カバレッジは実行された行の割合で、正しさの証明にはならない』です。この理解に不足があれば指摘してください。」

**この章でやってはいけない使い方**

- 実装を書かせてから、そのコードに合わせたテストもAIに書かせること。テストと実装が同じ思い込みを共有するので、間違いを見つけられません。TDD の意味がなくなります。
- テストが赤いときに、エラーを貼って「通るように直して」と頼むこと。まず自分でメッセージを読み、なぜ赤いのかを言葉にしてください。それでもわからなければ、「この失敗はどういう意味か」を聞きます。

## まとめ

テストは「こう動くはず」をプログラムとして書いたもので、変更を安全にし、仕様を残し、設計を良くします。TDD は、RED（失敗を確かめる）→ GREEN（最小限で通す）→ REFACTOR（緑のまま整える）の小さな繰り返しです。Gemfile・Rakefile・test_helper.rb をそろえれば、`bundle exec rake test` という1つのコマンドで、誰でも同じようにテストを実行できます。SimpleCov は「テストが通っていない場所」を見つける道具であり、数字そのものを目標にはしません。

さらに深く：元の研修では、Phase 3.2 でこの章と同じテスト環境を用意し、並行処理を TDD で安全に導入します（[Phase 3.2](https://github.com/speee/training-web-app/blob/main/phase-3-2.md)）。Minitest のアサーションの一覧は [Minitest のドキュメント](https://docs.seattlerb.org/minitest/)、SimpleCov の設定は [SimpleCov の README](https://github.com/simplecov-ruby/simplecov) で確認できます。

## 用語

| 用語 | 意味 |
|---|---|
| テスト（自動テスト） | 「こう動くはず」をプログラムとして書き、機械に確かめさせるもの |
| デグレード（リグレッション） | 変更によって、前は動いていた機能が壊れること |
| Minitest | Ruby の標準的なテスト用ライブラリ。Rails でも使われる |
| アサーション | 期待どおりかを確かめるメソッド。`assert_equal` など |
| Failure / Error | 結果が期待と違った失敗 ／ 確認の前に例外で止まった失敗 |
| TDD（テスト駆動開発） | テストを先に書き、RED → GREEN → REFACTOR を繰り返す進め方 |
| 仮実装 | テストを通すために、とりあえず決まった値を返すだけの実装 |
| 三角測量 | テストを増やすことで、仮実装を本物の実装へ導く方法 |
| リファクタリング | 振る舞いを変えずに、コードの形を整えること |
| setup | 各テストの前に毎回実行される準備用メソッド |
| Gemfile / Gemfile.lock | 使う Gem の一覧 ／ 実際にそろえた版の記録 |
| bundle exec | Gemfile.lock の版の Gem を使ってコマンドを実行する指定 |
| Rake / Rakefile | 作業手順を Ruby で書く道具 ／ その手順書のファイル |
| test_helper.rb | テストの共通準備をまとめたファイル |
| カバレッジ | テストの実行中に、コードのどの行が実行されたかの割合 |
| シード（seed） | テストの実行順を決める乱数の種。同じ値で同じ順番を再現できる |

---
[← 前の章](./05-ruby-objects.md) | [目次](../README.md) | [次の章 →](./07-debugging-and-research.md)
