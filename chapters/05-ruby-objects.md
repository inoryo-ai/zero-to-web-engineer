# 第5章 Ruby入門3：クラス・モジュール・例外・ファイル・require

> **この章のゴール**：データと処理をクラスにまとめ、失敗を例外で伝え、複数のファイルに分けた小さなプログラムを自分で書けるようになる。
> **目安時間**：読む 3時間 ＋ 演習 5時間
> **前提**：第4章まで

## この章で学ぶこと

- クラスを定義し、`new` でオブジェクトを作って使える
- インスタンス変数・`attr_reader`・`private` を使い分け、その理由を説明できる
- 継承とモジュール（ミックスイン）の違いを説明できる
- 例外を `raise` で発生させ、`rescue` / `ensure` で扱える
- ファイルを読み書きできる
- `require` と `require_relative` を使ってプログラムを複数ファイルに分けられる

## なぜクラスが必要なのか

第4章では、Hash を使ってデータをまとめました。たとえば本の情報は次のように書けます。

```ruby
book = { title: "Ruby入門", price: 2800 }
```

小さなプログラムなら、これで十分です。しかしプログラムが大きくなると、困ることが出てきます。

- キーを `:titel` と打ち間違えても、エラーにならずに `nil` が返る
- 「本の税込価格を計算する」処理が、あちこちのファイルにばらばらに書かれる
- どんなキーがそろっていれば「本」と呼べるのか、どこにも書かれていない

**クラス**は、この問題を解決する道具です。クラスとは「データ」と「そのデータを扱う処理」をひとまとめにした設計図です。

たい焼きでたとえます。クラスは、たい焼きの「型」です。型から焼いた1つ1つのたい焼きを**オブジェクト**（または**インスタンス**）と呼びます。同じ型から焼いても、中身があんこかクリームかは1つずつ違います。オブジェクトも同じで、同じクラスから作っても、持っているデータは1つずつ違います。

実は、これまで使ってきた値もすべてオブジェクトです。`"hello"` は String クラスのオブジェクト、`[1, 2]` は Array クラスのオブジェクトです。`"hello".upcase` と書けたのは、String クラスに `upcase` という処理が定義されているからです。この章では、その「クラス」を自分で作ります。

## クラスを定義する

犬を表すクラスを作ってみます。`dog.rb` という名前で保存してください。

```ruby
class Dog
  def initialize(name, age)
    @name = name
    @age = age
  end

  def bark
    "#{@name}「ワン！」"
  end

  def birthday
    @age += 1
  end

  def to_s
    "#{@name}（#{@age}歳）"
  end
end

pochi = Dog.new("ポチ", 3)
hachi = Dog.new("ハチ", 5)
puts pochi.bark
puts hachi.bark
pochi.birthday
puts pochi
puts hachi
p pochi.class
```

```console
$ ruby dog.rb
ポチ「ワン！」
ハチ「ワン！」
ポチ（4歳）
ハチ（5歳）
Dog
```

1つずつ見ていきます。

- `class Dog` 〜 `end` がクラスの定義です。クラス名は大文字で始めます。2語以上なら `TodoItem` のように単語の頭を大文字にします。
- `Dog.new("ポチ", 3)` で、型からたい焼きを1つ焼きます。このとき Ruby は自動で `initialize` メソッドを呼びます。`new` に渡した引数は、そのまま `initialize` に渡ります。
- `def bark` のように、クラスの中で定義したメソッドを**インスタンスメソッド**と呼びます。`pochi.bark` のように、オブジェクトに対して呼び出します。
- `to_s` は特別な名前です。`puts` はオブジェクトを表示するとき、`to_s` を呼んで文字列に変えます。だから `puts pochi` で「ポチ（4歳）」と表示されました。

`pochi.birthday` を呼んだあと、ポチは4歳になりました。一方、ハチは5歳のままです。同じクラスから作っても、データは別々に持っていることがわかります。

### インスタンス変数とローカル変数の違い

`@name` のように `@` で始まる変数を**インスタンス変数**と呼びます。オブジェクト1つ1つが持つ「持ち物」です。

第3章で学んだ、`@` の付かない変数は**ローカル変数**です。ローカル変数は、定義したメソッドの中でしか使えません。メソッドが終わると消えます。

インスタンス変数は、オブジェクトが生きている間ずっと残ります。そして同じオブジェクトのどのメソッドからでも使えます。`initialize` で入れた `@name` を `bark` で使えたのはこのためです。

ここでよくある間違いを1つ紹介します。`initialize` の中で `@` を付け忘れると、値はローカル変数に入って、メソッドの終わりで消えます。

```ruby
def initialize(name, age)
  name = name   # @ を付け忘れた。何も保存されない
end
```

この場合、エラーは出ません。あとで `@name` を使ったときに `nil` になっているだけです。エラーが出ないぶん、気づきにくい間違いです。

## 読み書きの窓口：attr_reader と attr_accessor

インスタンス変数は、オブジェクトの外から直接は見えません。`pochi.@name` のような書き方はできません。外から値を読みたいときは、値を返すメソッドを用意します。

```ruby
def name
  @name
end
```

このような「読むだけのメソッド」は頻繁に使うので、短く書く方法が用意されています。それが `attr_reader` です。

```ruby
class Book
  attr_reader :title, :price
  attr_accessor :stock

  def initialize(title, price)
    @title = title
    @price = price
    @stock = 0
  end
end

book = Book.new("Ruby入門", 2800)
puts book.title
book.stock = 5
puts book.stock
book.price = 3000
```

```console
$ ruby attr.rb
Ruby入門
5
attr.rb:16:in `<main>': undefined method `price=' for #<Book:0x00007fce92025608> (NoMethodError)
Did you mean?  price
```

（`0x00007f...` の部分は実行のたびに変わります。オブジェクトのメモリ上の場所を表す番号です。）

- `attr_reader :title` は、`@title` を返す `title` メソッドを作ります。
- `attr_accessor :stock` は、読む `stock` と書く `stock=` の両方を作ります。
- `attr_writer` という「書くだけ」もありますが、使う機会は少なめです。

`price` は `attr_reader` にしたので、`book.price = 3000` はエラーになりました。これは失敗ではなく、狙いどおりの動きです。

なぜ全部を `attr_accessor` にしないのでしょうか。理由は、**変えてはいけない値を守るため**です。本の価格が外から自由に書き換えられると、どこで誰が変えたのか追えなくなります。銀行の窓口にたとえると、残高を「見る」窓口はあっても、残高を「好きな数字に書き換える」窓口はありません。お金を動かすときは、「入金」「出金」という決まった手続きを通します。オブジェクトも同じで、外に見せる窓口は必要なものだけにします。この考え方を**カプセル化**と呼びます。

なお、エラーの最後に `Did you mean?  price` と出ています。これは Ruby が「もしかして `price` のことですか」と候補を出してくれる機能です。いつも正しい候補とは限りませんが、打ち間違いを見つける手がかりになります。

> **Ruby のバージョンによる表示の違い**：本書のエラー表示の例は、古い Ruby で実行したものも含みます。Ruby 3.4 以降では、メソッド名を囲む記号が `` `price=' `` から `'price='` に変わるなど、表示が少し違います。読み方は同じです。

## self・クラスメソッド・private

### self は「今のオブジェクト自身」

メソッドの中で `self` と書くと、そのメソッドを呼ばれたオブジェクト自身を指します。日本語の「自分」にあたる言葉です。`pochi.bark` の実行中なら、`self` は `pochi` です。

### クラスメソッド

`def self.メソッド名` と書くと、**クラスメソッド**になります。クラスメソッドは、オブジェクトではなくクラスそのものに対して呼びます。`Dog.new` の `new` も、実はクラスメソッドの一種です。

### private

`private` と書いた行より下のメソッドは、**プライベートメソッド**になります。クラスの中からは呼べますが、外からは呼べません。

3つをまとめて見てみます。

```ruby
class Account
  attr_reader :balance

  def initialize(balance)
    @balance = balance
  end

  def withdraw(amount)
    check_amount!(amount)
    @balance -= amount
  end

  def self.empty
    new(0)
  end

  private

  def check_amount!(amount)
    raise ArgumentError, "金額は1以上にしてください" if amount < 1
  end
end

account = Account.new(1000)
account.withdraw(300)
p account.balance
p Account.empty.balance
account.check_amount!(5)
```

```console
$ ruby account.rb
700
0
account.rb:28:in `<main>': private method `check_amount!' called for #<Account:0x00007f8f321213d0 @balance=700> (NoMethodError)
```

- `Account.empty` はクラスメソッドです。「残高0の口座を作る」という作り方に名前を付けています。中の `new(0)` は `self.new(0)` の省略形です。
- `check_amount!` はプライベートなので、外から呼ぶとエラーになります。
- `raise` はこの章の後半で説明します。ここでは「エラーを発生させる命令」と思ってください。

`private` を使う理由は、「外の人が使ってよいメソッド」と「中の都合で作った部品」を分けるためです。外に出したメソッドは、他の人のコードから呼ばれるかもしれません。一度呼ばれると、名前や引数を気軽に変えられなくなります。プライベートにしておけば、クラスの中だけを直せば済みます。

メソッド名の最後の `!` や `?` は、Ruby の慣習です。`?` は true / false を返すメソッドに付けます（例：`empty?`）。`!` は「注意して使うべきもの」に付けます（例：破壊的に変更する `upcase!`、失敗すると例外を出すもの）。

## 継承と super

**継承**は、あるクラスの性質を引き継いで、新しいクラスを作る仕組みです。引き継がれる側を**親クラス**（スーパークラス）、引き継ぐ側を**子クラス**（サブクラス）と呼びます。

```ruby
class Animal
  attr_reader :name

  def initialize(name)
    @name = name
  end

  def introduce
    "私は#{name}です"
  end
end

class Cat < Animal
  def introduce
    super + "。ニャー"
  end
end

class Bird < Animal
  def initialize(name, can_fly)
    super(name)
    @can_fly = can_fly
  end

  def fly?
    @can_fly
  end
end

puts Animal.new("ふつうの動物").introduce
puts Cat.new("タマ").introduce
piyo = Bird.new("ピヨ", true)
puts piyo.introduce
p piyo.fly?
p Cat.ancestors.take(3)
p piyo.is_a?(Animal)
```

```console
$ ruby inherit.rb
私はふつうの動物です
私はタマです。ニャー
私はピヨです
true
[Cat, Animal, Object]
true
```

- `class Cat < Animal` で、Cat は Animal を継承します。Cat の中に `initialize` や `name` を書いていないのに使えるのは、Animal から引き継いだからです。
- Cat は `introduce` を自分で定義し直しています。これを**オーバーライド**（上書き）と呼びます。
- `super` は「親クラスの同じ名前のメソッドを呼ぶ」という意味です。Cat の `introduce` は、親の結果に「。ニャー」を足しています。
- Bird の `initialize` では、`super(name)` で親の初期化を呼んでから、自分だけの `@can_fly` を設定しています。
- `ancestors` は、メソッドを探す順番を返します。Cat のメソッドを呼ぶと、まず Cat、なければ Animal、なければ Object の順に探します。

継承は便利ですが、使いすぎに注意が必要です。継承は「子は親の一種である」という関係のときに使います。「猫は動物の一種」は自然です。しかし「コードを共有したいから」という理由だけで継承すると、関係のないクラス同士が強く結びつきます。親を1か所変えただけで、思わぬ子クラスが壊れることがあります。共有したいのが「振る舞いの一部」だけなら、次に説明するモジュールが向いています。

この教材の後半では、継承を何度も使います。第14章で作る「コントローラの親クラス」や、第16章で作る「モデルの親クラス」がその例です。Rails でも、自分のクラスを `ApplicationController` や `ApplicationRecord` から継承させます。ここで「親から引き継ぎ、必要なら上書きし、`super` で親の処理も呼べる」という感覚をつかんでおいてください。

## モジュール：名前空間とミックスイン

**モジュール**は、メソッドや定数をまとめる入れ物です。クラスと似ていますが、`new` でオブジェクトを作れません。モジュールには、大きく2つの使い方があります。

### 使い方1：ミックスイン

1つ目は、複数のクラスに同じ振る舞いを「混ぜ込む」使い方です。これを**ミックスイン**と呼びます。

```ruby
module Greeting
  def hello
    "こんにちは、#{name}です"
  end
end

class Person
  include Greeting
  attr_reader :name

  def initialize(name)
    @name = name
  end
end

class Robot
  include Greeting

  def name
    "ロボ1号"
  end
end

puts Person.new("花子").hello
puts Robot.new.hello
p Person.ancestors.take(3)
```

```console
$ ruby modules.rb
こんにちは、花子です
こんにちは、ロボ1号です
[Person, Greeting, Object]
```

Person と Robot は、親子関係がまったくありません。それでも `include Greeting` と書くだけで、両方に `hello` が使えるようになりました。`ancestors` を見ると、Greeting がメソッドを探す道筋に入っていることがわかります。

継承とモジュールの違いを、たとえで整理します。継承は「血筋」です。子は親を1人しか持てません（Ruby の継承は1つの親からだけです）。モジュールは「習い事」です。ピアノも水泳も習えるように、1つのクラスに複数のモジュールを `include` できます。

### Ruby 標準のモジュールを借りる：Comparable

Ruby には便利なモジュールが最初から用意されています。代表例が `Comparable` です。`<=>` という比較用のメソッドを1つ定義するだけで、`<`、`>`、`==`、`between?` などがまとめて使えるようになります。

```ruby
class Score
  include Comparable
  attr_reader :point

  def initialize(point)
    @point = point
  end

  def <=>(other)
    point <=> other.point
  end
end

a = Score.new(70)
b = Score.new(85)
p a < b
p a == Score.new(70)
p [b, a].min.point
p a.between?(Score.new(60), b)
```

```console
$ ruby comparable.rb
true
true
70
true
```

`<=>` は「宇宙船演算子」と呼ばれます。左が小さければ -1、等しければ 0、大きければ 1 を返すという約束です。数値どうしの `<=>` はすでに Ruby が持っているので、点数で比べるように書いただけです。

これは「約束を1つ守れば、たくさんの機能がもらえる」という仕組みです。同じ考え方の `Enumerable` は、`each` を1つ定義すれば `map`・`select`・`count` などが使えるようになります。Array や Hash で `map` が使えたのも、`Enumerable` を include しているからです。演習5-2で実際に使います。

この「約束（インターフェース）を守れば、部品として差し込める」という考え方は、第13章の Rack でも中心になります。今のうちに覚えておいてください。

### 使い方2：名前空間

2つ目は、名前をグループ分けする使い方です。これを**名前空間**と呼びます。

```ruby
module Shop
  class Item
    def initialize(name)
      @name = name
    end
  end

  TAX_RATE = 10

  def self.price_with_tax(price)
    price * (100 + TAX_RATE) / 100
  end
end

p Shop::Item.new("りんご")
p Shop::TAX_RATE
p Shop.price_with_tax(200)
```

```console
$ ruby namespace.rb
#<Shop::Item:0x00007fc681015780 @name="りんご">
10
220
```

`Shop::Item` のように `::` でつなぐと、「Shop の中の Item」という意味になります。大きなプログラムでは、`Item` という名前のクラスが別の場所にもあるかもしれません。名前空間に入れておけば、名前がぶつかりません。住所の「東京都の中央区」と「大阪府の中央区」を区別するのと同じです。

`TAX_RATE` のように、大文字だけで書いた名前は**定数**です。「途中で変えない値」という意味を込めて使います。

## 例外：失敗を伝える仕組み

### 例外が起きるとどうなるか

プログラムが続けられない問題が起きると、Ruby は**例外**を発生させます。例外とは「ここで問題が起きたので、通常の処理を中断します」という知らせです。

```ruby
puts "計算を始めます"
puts 10 / 0
puts "計算が終わりました"
```

```console
$ ruby exc1.rb
計算を始めます
exc1.rb:2:in `/': divided by 0 (ZeroDivisionError)
	from exc1.rb:2:in `<main>'
```

0で割ることはできないので、2行目で `ZeroDivisionError` という例外が起きました。3行目は実行されていません。誰も例外を受け止めなかったので、プログラム全体が止まりました。

表示されたメッセージの読み方は、第7章で詳しく学びます。ここでは「何行目で、どんな種類の例外が起きたか」が書いてあることだけ確認してください。

### rescue で受け止める

例外は `rescue` で受け止められます。受け止めれば、プログラムは止まらずに続きます。

```ruby
def safe_divide(a, b)
  a / b
rescue ZeroDivisionError => e
  puts "エラーを捕まえました: #{e.message}（#{e.class}）"
  nil
end

p safe_divide(10, 2)
p safe_divide(10, 0)
puts "プログラムは続きます"
```

```console
$ ruby exc2.rb
5
エラーを捕まえました: divided by 0（ZeroDivisionError）
nil
プログラムは続きます
```

- `rescue ZeroDivisionError => e` は、「ZeroDivisionError が起きたら、その例外オブジェクトを `e` に入れて、ここに飛ぶ」という意味です。
- 例外もオブジェクトです。`e.message` で説明文、`e.class` で種類がわかります。
- メソッドの中なら、`def` 〜 `end` の中に直接 `rescue` を書けます。

### raise で自分から失敗を伝える／ensure で後片付け

`raise` を使うと、自分で例外を発生させられます。「この入力ではこれ以上進めない」と呼び出し元に伝えたいときに使います。

`begin` 〜 `end` で囲めば、メソッドの途中でも `rescue` を書けます。`ensure` に書いた処理は、例外が起きても起きなくても必ず実行されます。

```ruby
def parse_age(text)
  age = Integer(text)
  raise ArgumentError, "年齢は0以上です: #{age}" if age < 0
  age
end

["20", "-3", "abc"].each do |input|
  begin
    puts "入力 #{input.inspect} → #{parse_age(input)}"
  rescue ArgumentError => e
    puts "入力 #{input.inspect} → エラー: #{e.message}"
  ensure
    puts "  （#{input.inspect} の処理を終了）"
  end
end
```

```console
$ ruby exc3.rb
入力 "20" → 20
  （"20" の処理を終了）
入力 "-3" → エラー: 年齢は0以上です: -3
  （"-3" の処理を終了）
入力 "abc" → エラー: invalid value for Integer(): "abc"
  （"abc" の処理を終了）
```

- `Integer("abc")` は、数字に変換できないので ArgumentError を発生させます。`"abc".to_i` は 0 を返してしまいますが、`Integer()` は「変換できない」とはっきり教えてくれます。
- `-3` は数字には変換できますが、年齢としてはおかしい値です。そこで自分で `raise` しています。
- どちらの例外も同じ `rescue ArgumentError` で受け止められています。
- `ensure` の行は、3回とも実行されています。

`ensure` は、ファイルやネットワーク接続を必ず閉じたいときに使います。第11章で作る HTTP サーバでも、「エラーが起きても接続を閉じる」ために登場します。

### 自分の例外クラスを作る

例外の種類は、クラスで表されています。`StandardError` を継承すれば、自分の例外クラスを作れます。

```ruby
class OutOfStockError < StandardError
  def initialize(item_name)
    super("#{item_name} は在庫切れです")
  end
end

class Shelf
  def initialize
    @stock = { "りんご" => 1 }
  end

  def take(name)
    raise OutOfStockError.new(name) if @stock.fetch(name, 0) < 1
    @stock[name] -= 1
    name
  end
end

shelf = Shelf.new
puts shelf.take("りんご")
begin
  shelf.take("りんご")
rescue OutOfStockError => e
  puts "失敗: #{e.message}"
end
p OutOfStockError.ancestors.take(4)
```

```console
$ ruby exc4.rb
りんご
失敗: りんご は在庫切れです
[OutOfStockError, StandardError, Exception, Object]
```

自分の例外クラスを作ると、呼び出し側が「在庫切れのときだけ」を狙って `rescue` できます。ArgumentError のような汎用の例外だと、在庫切れ以外の失敗まで一緒に受け止めてしまうかもしれません。

ここでも継承と `super` が使われています。`super("...")` で、親の StandardError にメッセージを渡しています。

### 例外の使いどころ：握りつぶさない

例外で一番やってはいけないのは、**何でも受け止めて、何もしないこと**です。これを「例外を握りつぶす」と言います。

```ruby
begin
  do_something
rescue => e
  # 何もしない
end
```

`rescue` の後に種類を書かないと、StandardError とその子クラスをすべて受け止めます。打ち間違いによる NoMethodError まで消えてしまいます。プログラムは何事もなかったように進みますが、データは壊れているかもしれません。火災報知器が鳴ったのに、うるさいからと電池を抜くようなものです。

守ってほしいルールは3つです。

1. `rescue` には、受け止めたい例外の種類を書く。
2. 受け止めたら、利用者へのメッセージ表示や記録など、意味のある対応をする。
3. 対応できないなら、受け止めずにそのまま上に伝える。

## ファイルを読み書きする

プログラムを終了すると、変数の中身は消えます。データを残したいときは、ファイルに書き込みます。

```ruby
path = File.join(__dir__, "memo.txt")

File.write(path, "1行目\n2行目\n")
File.open(path, "a") do |f|
  f.puts "3行目"
end

puts File.read(path)
p File.readlines(path, chomp: true)
p File.exist?(path)
p File.basename(path)
p File.extname(path)

File.foreach(path).with_index(1) do |line, number|
  puts "#{number}: #{line}"
end
```

```console
$ ruby files.rb
1行目
2行目
3行目
["1行目", "2行目", "3行目"]
true
"memo.txt"
".txt"
1: 1行目
2: 2行目
3: 3行目
```

| 書き方 | 意味 |
|---|---|
| `File.write(path, 文字列)` | ファイルを丸ごと書く。前の中身は消える |
| `File.open(path, "a") { \|f\| ... }` | 追記モード（`"a"`）で開き、ブロックが終わると自動で閉じる |
| `File.read(path)` | ファイル全体を1つの文字列として読む |
| `File.readlines(path, chomp: true)` | 1行ずつの配列で読む。`chomp: true` で行末の改行を除く |
| `File.foreach(path)` | 1行ずつ順番に処理する。大きなファイル向き |
| `File.exist?(path)` | ファイルがあるか調べる |

`File.open` にブロックを渡す書き方は大切です。ファイルを開いたら、使い終わったときに閉じる必要があります。ブロックを使えば、途中で例外が起きても Ruby が閉じてくれます。中で `ensure` と同じ仕組みが働いているからです。

存在しないファイルを読むと、例外が発生します。

```console
$ ruby -e 'File.read("nothing.txt")'
-e:1:in `read': No such file or directory @ rb_sysopen - nothing.txt (Errno::ENOENT)
	from -e:1:in `<main>'
```

`Errno::ENOENT` は「そのファイルやフォルダは存在しない」という意味の例外です。`Errno::` の後ろは OS が返すエラーの名前です。ENOENT（エラー・ノー・エントリ）は頻繁に見かけるので、名前ごと覚えてしまいましょう。

### `__dir__` と File.join を使う理由

上の例では、`"memo.txt"` と直接書かずに `File.join(__dir__, "memo.txt")` と書きました。

`"memo.txt"` とだけ書くと、Ruby は「今いるフォルダ（カレントディレクトリ）」を基準に探します。第1章で学んだ `cd` の場所です。同じスクリプトでも、どこから実行したかで、読み書きする場所が変わってしまいます。

`__dir__` は「このスクリプトファイルが置かれているフォルダ」を返します。これを基準にすれば、どこから実行しても同じ場所を指せます。`File.join` は、フォルダ名とファイル名を `/` でつなぎます。

### ファイル操作の安全上の注意

ファイルを扱うときは、次の2点を守ってください。

- 演習で書き込むのは、自分の演習フォルダ（`~/workspace/` の下）だけにする。
- 利用者が入力した文字列を、そのままファイルのパスに使わない。

2つ目は、第10章と第11章で詳しく扱う**パストラバーサル**という攻撃につながります。`../` を含む入力で、公開するつもりのないファイルを読まれる攻撃です。今は「入力をそのままパスにするのは危ない」とだけ覚えておいてください。

## require と require_relative

プログラムが長くなったら、ファイルを分けます。分けたファイルを読み込むのが `require_relative` と `require` です。

次のようなフォルダ構成を作ります。

```text
ch05-sample/
├── main.rb
└── lib/
    └── calculator.rb
```

```ruby
# lib/calculator.rb
class Calculator
  def add(a, b)
    a + b
  end
end
```

```ruby
# main.rb
require_relative "lib/calculator"
require "json"

calc = Calculator.new
puts calc.add(2, 3)
puts JSON.generate({ "result" => calc.add(2, 3) })
```

```console
$ ruby main.rb
5
{"result":5}
```

- `require_relative "lib/calculator"` は、「このファイル（main.rb）から見た `lib/calculator.rb`」を読み込みます。`.rb` は省略できます。
- `require "json"` は、Ruby に付属しているライブラリ（JSON）を読み込みます。

2つの違いは「どこを探すか」です。

| 書き方 | 探す場所 | 主な用途 |
|---|---|---|
| `require_relative` | 書いたファイルの場所から見た相対パス | 自分のプロジェクトの別ファイル |
| `require` | **ロードパス**（`$LOAD_PATH`）に登録されたフォルダ | Ruby の標準ライブラリや Gem |

**ロードパス**とは、`require` がファイルを探しに行くフォルダの一覧です。本棚の一覧のようなものです。`require` は、その一覧の本棚を順番に探します。

自分のファイルを `require "calculator"` と書くと、どうなるでしょうか。次の `main2.rb` で試します。

```ruby
# main2.rb
require "calculator"
puts Calculator.new.add(1, 1)
```

```console
$ ruby main2.rb
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/rubygems/core_ext/kernel_require.rb:54:in `require': cannot load such file -- calculator (LoadError)
	from /System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/rubygems/core_ext/kernel_require.rb:54:in `require'
	from main2.rb:1:in `<main>'
```

`lib` フォルダはロードパスに入っていないので、見つかりません（1行目のパスは環境によって違います）。`ruby -I lib main2.rb` のように `-I` オプションで本棚を追加すると、見つかるようになります。

```console
$ ruby -I lib main2.rb
2
```

次の第6章では、テスト用の設定ファイルで `lib` をロードパスに追加します。そうすると、テストから `require "cart"` のように短く書けます。今は「`require` はロードパスを探す」「`require_relative` は自分の場所から探す」の2点を押さえてください。

なお `require` は、同じファイルを2回目以降は読み込みません。何度書いても、読み込みは1回だけです。

### Gem と Bundler（次章の予告）

**Gem** は、他の人が作って公開している Ruby のライブラリです。たとえばテストのための minitest、カバレッジ測定の simplecov などがあります。Gem は rubygems.org という場所で配布されています。

プロジェクトで使う Gem とその版は、**Gemfile** というファイルに書きます。その一覧に従って Gem をそろえる道具が **Bundler**（`bundle` コマンド）です。料理のレシピに書かれた材料表が Gemfile、材料を買いそろえる係が Bundler、と考えてください。実際の使い方は第6章で手を動かしながら学びます。

## 実務ではこうなる

実務の Ruby / Rails のコードは、ほとんどがクラスとモジュールでできています。Rails アプリを開くと、`app/models/user.rb` には `class User < ApplicationRecord` と書かれています。この1行だけで、継承・名前空間・ロードの仕組みがすべて関わっています。この章の内容は、それを読むための土台です。

現場でよく見る場面と、ジュニアがやりがちな失敗を挙げます。

- **何でも `attr_accessor`**：とりあえず全部を読み書き可能にすると、どこで値が変わったのか追えなくなります。レビューで「この値は外から書き換える必要がありますか」と聞かれます。まず `attr_reader` で始め、必要になってから書き込みを足します。
- **例外の握りつぶし**：`rescue => e` の後に何もしないコードは、レビューでほぼ確実に指摘されます。本番で起きた不具合の原因が、ログに何も残らないからです。
- **継承の乱用**：「似ているから」と継承を重ねると、親クラスの変更が広い範囲に影響します。「子は親の一種か」を自問する習慣を付けてください。
- **カレントディレクトリ依存**：手元では動くのに、別の場所から実行すると「ファイルが見つからない」と言われる。原因の多くは、相対パスの基準がずれていることです。
- **独自例外で意図を伝える**：実務のコードでは `class PaymentFailedError < StandardError` のような独自例外をよく見ます。呼び出し側が「どの失敗にどう対応するか」を選べるようにするためです。

## 演習

演習のコードは `~/workspace/ch05-ruby-objects/` に作ります。最初に次を実行してください。

```bash
mkdir -p ~/workspace/ch05-ruby-objects
cd ~/workspace/ch05-ruby-objects
```

この章では、小さな ToDo 管理プログラムを少しずつ育てます。演習5-1から5-4は順番に積み上がるので、飛ばさずに進めてください。ここで作るコードには、第6章でテストを付けます。

### 演習5-1：TodoItem クラスを作る

- `todo_item.rb` に `TodoItem` クラスを作ってください。仕様は次のとおりです。
  - `TodoItem.new("牛乳を買う")` で作れる。作った直後は未完了。
  - `title` でタイトルを読める。タイトルを外から書き換えることはできない。
  - `done?` で完了かどうかを true / false で返す。
  - `complete!` で完了にする。
  - `to_s` は、未完了なら `[ ] 牛乳を買う`、完了なら `[x] 牛乳を買う` を返す。
- ファイルの末尾に、次の確認用コードをそのまま書いてください。

```ruby
item = TodoItem.new("牛乳を買う")
puts item
p item.done?
item.complete!
puts item
p item.done?
p item.respond_to?(:title=)
```

- **完了条件**：`ruby todo_item.rb` の出力が次の5行と一致する。

```text
[ ] 牛乳を買う
false
[x] 牛乳を買う
true
false
```

- ヒント：完了状態は、`initialize` でインスタンス変数に `false` を入れておきます。`respond_to?(:title=)` は「`title=` メソッドを持っているか」を調べます。最後が `false` になるのは、どの `attr_` を選んだときでしょうか。

### 演習5-2：TodoList に Enumerable を混ぜる

- 新しく `todo_list.rb` を作り、先頭で `require_relative "todo_item"` と書いてください。演習5-1の確認用コードは、`todo_item.rb` から消しておきます。
- `TodoList` クラスを作ってください。
  - `add(title)` で TodoItem を作って追加する。
  - `complete(number)` で、1から数えて `number` 番目の項目を完了にする。
  - `each` を定義し、各 TodoItem をブロックに渡す。
  - `include Enumerable` する。
  - `to_s` は、各項目を `1. [ ] 牛乳を買う` の形で、改行区切りで並べた文字列を返す。
- `todo_list.rb` の末尾に、次の確認用コードを書いてください。

```ruby
list = TodoList.new
list.add("牛乳を買う")
list.add("本を返す")
list.add("Rubyの演習")
list.complete(2)
puts list
p list.count(&:done?)
p list.map(&:title)
p list.reject(&:done?).size
```

- **完了条件**：`ruby todo_list.rb` の出力が次と一致する。`count`・`map`・`reject` は自分で定義していないこと。

```text
1. [ ] 牛乳を買う
2. [x] 本を返す
3. [ ] Rubyの演習
1
["牛乳を買う", "本を返す", "Rubyの演習"]
2
```

- ヒント：`each` の中では、内部の配列の `each` にブロックを渡せば済みます。ブロックを受け取って別のメソッドに渡す書き方（`&block`）は、[公式ドキュメントの「メソッド定義」](https://docs.ruby-lang.org/ja/latest/doc/spec=2fdef.html)で調べてください。`list.count(&:done?)` は `list.count { |item| item.done? }` の省略形です。

### 演習5-3：例外で失敗を伝える

- `todo_list.rb` を次のように変更してください。
  - `todo_list.rb` の中に、`StandardError` を継承した `TodoNotFoundError` を定義する。
  - `complete(number)` で、存在しない番号（0、負の数、項目数より大きい数）を指定されたら `TodoNotFoundError` を発生させる。メッセージは `5番のToDoはありません` の形にする。
  - `add(title)` で、タイトルが空文字や空白だけなら `ArgumentError` を発生させる。メッセージは `タイトルを入力してください` にする。
- 確認用コードを次に差し替えてください。

```ruby
list = TodoList.new
list.add("牛乳を買う")
[1, 5, 0].each do |number|
  begin
    list.complete(number)
    puts "#{number}番を完了しました"
  rescue TodoNotFoundError => e
    puts "エラー: #{e.message}"
  end
end
begin
  list.add("   ")
rescue ArgumentError => e
  puts "エラー: #{e.message}"
end
puts list
```

- **完了条件**：`ruby todo_list.rb` の出力が次と一致する。

```text
1番を完了しました
エラー: 5番のToDoはありません
エラー: 0番のToDoはありません
エラー: タイトルを入力してください
1. [x] 牛乳を買う
```

- ヒント：0番を「存在しない」と判定できるか注意してください。Ruby の配列で `items[-1]` は「最後の要素」を返すので、`items[number - 1]` だけでは 0 番を見逃します。空白だけの判定には `strip` が使えます。

### 演習5-4：ファイルに保存し、コマンドとして使う

- フォルダ構成を次のように変えてください。演習5-2・5-3の確認用コードは消します。

```text
ch05-ruby-objects/
├── todo.rb            ← 実行の入口（新規）
├── lib/
│   ├── todo_item.rb   ← 移動
│   └── todo_list.rb   ← 移動
└── todos.txt          ← プログラムが作る
```

- `todo.rb` を、次の3つのコマンドで使えるようにしてください。`ARGV` は「コマンドに渡された引数の配列」です。`ruby todo.rb add 牛乳を買う` なら `ARGV` は `["add", "牛乳を買う"]` になります。
  - `ruby todo.rb add タイトル`：追加して保存する
  - `ruby todo.rb list`：一覧を表示する
  - `ruby todo.rb done 番号`：完了にして保存する
- 保存形式は、1行に1件、`完了なら1・未完了なら0` とタイトルをタブで区切ります（例：`0\t牛乳を買う`）。
- `todos.txt` は `todo.rb` と同じフォルダに置きます。どこから実行しても同じファイルを使うようにしてください。
- `done` で存在しない番号を指定されたら、エラーメッセージを表示します。プログラムが例外の表示で止まってはいけません。
- **完了条件**：`todos.txt` を消した状態から、別のフォルダで次を実行し、出力が一致する。

```console
$ rm -f ~/workspace/ch05-ruby-objects/todos.txt
$ cd ~
$ ruby ~/workspace/ch05-ruby-objects/todo.rb list
$ ruby ~/workspace/ch05-ruby-objects/todo.rb add 牛乳を買う
$ ruby ~/workspace/ch05-ruby-objects/todo.rb add 本を返す
$ ruby ~/workspace/ch05-ruby-objects/todo.rb done 2
$ ruby ~/workspace/ch05-ruby-objects/todo.rb done 9
エラー: 9番のToDoはありません
$ ruby ~/workspace/ch05-ruby-objects/todo.rb list
1. [ ] 牛乳を買う
2. [x] 本を返す
```

- ヒント：
  - 1回目の `list` は、ファイルがまだないので何も表示されません（空行が1行出るのは構いません）。ファイルがないときに例外で止まらないよう、`File.exist?` で確かめてから読みましょう。
  - 保存と読み込みは TodoList に `save(path)` と `self.load(path)` のようなメソッドとして持たせると、`todo.rb` が短くなります。
  - 読み込みのときは、完了済みの項目を完了状態で復元する必要があります。`TodoItem.new` に「完了済みか」を渡せるようにする方法と、作った後に `complete!` を呼ぶ方法があります。どちらがよいか考えてみてください。
  - タイトルにタブが含まれると保存形式が壊れます。今回は対象外にしてよいですが、なぜ壊れるかは説明できるようにしてください。
- 終わったら Git でコミットしましょう（第2章の手順）。`todos.txt` はコミットしてもしなくても構いません。

### 演習5-5（発展）：継承で期限付き ToDo を作る

- `lib/deadline_todo_item.rb` に、`TodoItem` を継承した `DeadlineTodoItem` を作ってください。
  - `DeadlineTodoItem.new("レポート提出", "2026-10-10")` で作れる。
  - `to_s` は `[ ] レポート提出（期限: 2026-10-10）` を返す。親の `to_s` の結果を `super` で使い、同じ文字列を2か所に書かない。
  - `include Comparable` し、期限の早い順に `sort` できるようにする。
- **完了条件**：期限が `"2026-12-01"`、`"2026-10-10"`、`"2026-11-05"` の3つを作り、配列にして `sort` して `puts` すると、10月・11月・12月の順に表示される。`DeadlineTodoItem.ancestors` の結果に `TodoItem` と `Comparable` の両方が含まれる。
- ヒント：`"2026-10-10" <=> "2026-11-05"` は文字列の比較ですが、この形式なら日付の順と一致します。なぜ一致するのか、`"2026-9-1"` のような形式だと何が起きるかも考えてください。日付を正しく扱うには `require "date"` の Date クラスを使う方法があります。[公式ドキュメント](https://docs.ruby-lang.org/ja/latest/doc/index.html)で調べてみてください。

## 確認クイズ

**Q1.** `initialize` の中で `@name = name` と書くべきところを `name = name` と書きました。何が起き、なぜ気づきにくいのでしょうか。

**Q2.** あるクラスの価格を `attr_accessor :price` にしようとしたら、レビューで `attr_reader` にするよう言われました。理由を説明してください。

**Q3.** 継承とモジュールの `include` は、どう使い分けますか。

**Q4.** 次のコードの問題点を2つ挙げてください。

```ruby
begin
  save_order(order)
rescue => e
end
```

**Q5.** `File.read("data.txt")` が、あるフォルダから実行すると動き、別のフォルダから実行すると `Errno::ENOENT` になります。原因と直し方を説明してください。

**Q6.** `require "calculator"` が `LoadError` になり、`require_relative "lib/calculator"` なら動きました。2つは何が違いますか。

### 解答と解説

**A1.** 値はローカル変数 `name` に入るだけで、`initialize` が終わると消えます。インスタンス変数 `@name` には何も入らず、後で読むと `nil` になります。文法上は正しいのでエラーが出ません。問題が表に出るのは、`nil` を使って何かしようとしたときです。そのため、原因の行と、エラーが出る行が離れてしまいます。

**A2.** `attr_accessor` は書き込み用の `price=` も作るので、外のコードがどこからでも価格を書き換えられるようになります。変える必要がない値は `attr_reader` にして、外からの変更を防ぎます（カプセル化）。値を変える必要がある場合も、意味のある名前のメソッドを用意し、その中で値をチェックする方が安全です。

**A3.** 継承は「子は親の一種である」関係に使います。親は1つだけです。モジュールは、関係のない複数のクラスに共通の振る舞いを足したいときに使います。1つのクラスに複数 include できます。「コードを共有したいだけ」なら、継承よりモジュールを先に検討します。

**A4.** 1つ目は、例外の種類を指定していないことです。打ち間違いによる NoMethodError なども含め、ほぼすべての失敗を受け止めてしまいます。2つ目は、受け止めた後に何もしていないことです。保存に失敗したのに、利用者にも開発者にも伝わりません。受け止める例外を絞り、メッセージの表示や記録などの対応を書くべきです。

**A5.** `"data.txt"` という相対パスは、実行したときのカレントディレクトリを基準に探されます。だから実行場所によって結果が変わります。スクリプトの場所を基準にするには、`File.join(__dir__, "data.txt")` のように `__dir__` を使います。

**A6.** `require` はロードパス（`$LOAD_PATH`）に登録されたフォルダを探します。`lib` が登録されていなければ見つかりません。`require_relative` は、書いたファイルの場所からの相対パスで探します。`require` で読みたい場合は、`ruby -I lib` やテストの設定でロードパスに `lib` を追加します。

## AIの使い方（この章）

この章は、Ruby の「部品の作り方」を体に覚えさせる章です。AIには解説役と確認役をお願いします。

**推奨する聞き方の例**

- 「Ruby の `attr_reader` と `attr_accessor` の違いを、銀行口座のたとえで説明してください。そのうえで、私の理解『reader は読むだけ、accessor は読み書き両方で、変えてはいけない値は reader にする』が合っているか確認してください。」
- 「次のエラーメッセージの各部分が何を表しているか、1行ずつ説明してください。コードの修正案は出さないでください。（エラーメッセージを貼る）」
- 「継承とモジュールの使い分けについて、私は『〇〇』と考えました。この考えの弱い点や、例外になるケースを指摘してください。」

**この章でやってはいけない使い方**

- 演習の仕様を丸ごと貼って「TodoList クラスを書いて」と頼むこと。クラス設計を自分で考える経験が、この章の学習の中心です。
- AIが出したコードを、意味を理解しないまま貼り付けること。特に `rescue => e` で例外を握りつぶすコードを、AIが出すことがあります。

## まとめ

クラスは、データと処理をまとめた設計図です。`attr_reader` や `private` で、外に見せる窓口を必要なものだけにします。継承は「〜の一種」の関係に、モジュールは振る舞いの共有と名前空間に使います。失敗は例外で伝え、受け止めるときは種類を絞って意味のある対応をします。ファイルのパスは `__dir__` を基準にし、ファイルの分割には `require_relative` と `require` を使い分けます。

## 用語

| 用語 | 意味 |
|---|---|
| クラス | データと処理をまとめた設計図。`class 名前 ... end` で定義する |
| オブジェクト（インスタンス） | クラスから `new` で作った1つ1つの実体 |
| インスタンス変数 | `@` で始まる変数。オブジェクトごとの持ち物で、同じオブジェクトのどのメソッドからも使える |
| initialize | `new` したときに自動で呼ばれる初期化用のメソッド |
| attr_reader / attr_accessor | インスタンス変数を読む（読み書きする）メソッドを自動で作る命令 |
| カプセル化 | オブジェクトの中身を隠し、外に見せる窓口を必要なものだけにする考え方 |
| self | 今のメソッドを呼ばれているオブジェクト自身 |
| クラスメソッド | `def self.名前` で定義し、クラスに対して呼ぶメソッド |
| private | クラスの外から呼べないメソッドにする指定 |
| 継承 | 親クラスの性質を引き継いで子クラスを作る仕組み |
| super | 親クラスの同じ名前のメソッドを呼ぶキーワード |
| モジュール | メソッドや定数をまとめる入れ物。ミックスインと名前空間に使う |
| ミックスイン | `include` でモジュールの振る舞いをクラスに混ぜ込むこと |
| 名前空間 | `Shop::Item` のように、名前をグループ分けして衝突を防ぐ仕組み |
| 例外 | 処理を続けられない問題が起きたことを知らせる仕組み。`raise` で発生、`rescue` で受け止める |
| ensure | 例外が起きても起きなくても必ず実行される部分 |
| ロードパス | `require` がファイルを探すフォルダの一覧（`$LOAD_PATH`） |
| Gem / Bundler | 公開されている Ruby のライブラリ ／ Gemfile に従って Gem をそろえる道具 |

---
[← 前の章](./04-ruby-collections-methods.md) | [目次](../README.md) | [次の章 →](./06-testing-and-tdd.md)
