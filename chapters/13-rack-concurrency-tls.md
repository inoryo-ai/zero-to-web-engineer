# 第13章 Rack・並行処理・HTTPS

> **この章のゴール**：自作サーバとアプリを「Rack」という約束で切り離し、同時アクセスと暗号化（HTTPS）に対応させます。その過程で、抽象化・スレッドの危うさ・TLSの役割を自分の言葉で説明できるようになります。
> **目安時間**：読む 4時間 ＋ 演習 12時間
> **前提**：第12章まで

## この章で学ぶこと

- Rack の約束 `call(env) → [status, headers, body]` を説明し、最小の Rack アプリを書ける
- `rackup -o 127.0.0.1` でサーバを起動し、待受先が 127.0.0.1 だけであることを確かめられる
- ミドルウェアの仕組みを説明し、リクエストの前後に処理を挟むミドルウェアをTDDで書ける
- 第11章の自作サーバを Rack 対応にし、サーバとアプリを分離できる
- スレッドで並行処理を導入し、競合状態（race condition）を再現して Mutex で防げる
- 自己署名証明書でHTTPSサーバを動かし、`--cacert` による検証と `curl -k` の違いを説明できる

## この章の地図

この章は3つの話題でできています。どれも「サーバの中身を一段上の視点から見直す」話です。

1. **Rack**：サーバとアプリの間に「約束」を置き、両者を差し替え可能にする
2. **並行処理**：1人ずつしか相手にできないサーバを、同時に何人も相手にできるようにする
3. **HTTPS**：通信の途中で盗み見や改ざんをされないよう、HTTPを暗号の封筒に入れる

第11章では、Socket から HTTP サーバを自作しました。第12章では、本物のサーバ WEBrick のコードを読みました。この章ではその2つを材料にします。作業フォルダは `~/workspace/ch13-rack/` です。第11章のサーバをこのフォルダにコピーして使います。

ここは難しい章です。特に並行処理は、プロのエンジニアでも間違えます。全部を一度で理解しようとせず、「何が起きているか」を手で確かめながら進めてください。

## 準備：作業フォルダと Gemfile

まず作業フォルダを作ります。Gem（Rubyのライブラリ）は Bundler でまとめて管理します。Bundler は「このプロジェクトはこの Gem を使う」という一覧表（Gemfile）を読み、必要なものをそろえる道具です。

```bash
mkdir -p ~/workspace/ch13-rack
cd ~/workspace/ch13-rack
git init
```

`Gemfile` を作り、次の内容を書きます。

```ruby
source "https://rubygems.org"

gem "rack"
gem "rackup"
gem "webrick"
gem "rake"
gem "minitest"
```

Rack 3 からは、サーバを起動するコマンド `rackup` が別の Gem に分かれました。また Ruby 3.0 以降は WEBrick が標準添付ではなくなりました。そのため3つを明示します。

```bash
bundle install
```

テストを `bundle exec rake test` で動かせるように、`Rakefile` と `test/test_helper.rb` も用意します。中身は第6章で作ったものと同じ形で構いません。

```ruby
# Rakefile
require "rake/testtask"

Rake::TestTask.new do |t|
  t.libs << "test"
  t.libs << "lib"
  t.pattern = "test/**/*_test.rb"
end

task default: :test
```

## Rack とは「約束」である

### なぜ約束が必要なのか

第11章のサーバを思い出してください。`lib/` に部品を分けはしましたが、`MiniHTTP` や `Router` は自作サーバ専用の形でした。全体としては、次の2種類の仕事が1つのプログラムに同居していたはずです。

- **サーバの仕事**：ソケットで接続を受ける。リクエストの文字列を読み、解析する。レスポンスの文字列を書いて送る
- **アプリの仕事**：パスが `/` なら「Hello」、`/users` なら一覧、と中身を決める

この2つが混ざっていると困ります。アプリを変えたいだけなのに、サーバの部分も読む必要があります。サーバを速いものに替えたくても、アプリが絡みついていて外せません。

電気のコンセントを考えてください。差し込み口の形が決まっているので、どのメーカーのドライヤーでも、どの家のコンセントにも挿せます。家電メーカーは発電所の仕組みを知らなくてよいのです。

Rack は Ruby の Web の世界での「コンセントの形」です。サーバ側とアプリ側が同じ形を守れば、どの組み合わせでも動きます。Rails も Sinatra も Rack アプリです。Puma も WEBrick も Rack 対応サーバです。だから Rails アプリを Puma でも WEBrick でも動かせます。

### 約束の中身

Rack の約束は、たった1行にまとまります。

```
call(env) → [status, headers, body]
```

- **アプリは `call` というメソッドを持つ**。引数は1つ、`env` という Hash です
- **`env`**：リクエストの情報がすべて入った Hash です。メソッド、パス、ヘッダなどが入ります
- **戻り値は3つの要素を持つ配列**です
  - `status`：ステータスコードの整数（200、404 など）
  - `headers`：レスポンスヘッダの Hash
  - `body`：本文。`each` で文字列を順に取り出せるもの（文字列の配列が一番簡単）

最小の Rack アプリはこうなります。

```ruby
app = Proc.new do |env|
  [200, { "content-type" => "text/plain" }, ["Hello Rack\n"]]
end
```

`Proc` は「呼び出せる処理のかたまり」です。`Proc` は `call` メソッドを持っているので、これだけで約束を満たします。

**Rack 3 では、レスポンスヘッダ名は小文字で書きます。** `Content-Type` ではなく `content-type` です。HTTP のヘッダ名はもともと大文字小文字を区別しません。それでも書き方が混ざると、同じヘッダが2つできるなどの事故が起きます。そこで Rack 3 は「小文字に統一」と決めました。

### なぜ配列なのか

戻り値がクラスのオブジェクトではなく、ただの配列なのはなぜでしょう。理由は「誰でも作れて、誰でも読める」からです。特別なクラスに依存すると、サーバもアプリもそのクラスを読み込む必要があります。配列と Hash と文字列なら、Ruby が最初から持っています。約束は小さいほど守りやすいのです。

### rackup で起動する

`config.ru` というファイルを作ります。`.ru` は「rackup 用の設定ファイル」という意味の拡張子です。

```ruby
# config.ru
app = Proc.new do |env|
  [200, { "content-type" => "text/plain" }, ["Hello Rack\n"]]
end

run app
```

`run app` は「このアプリを動かして」という指示です。起動します。

```bash
bundle exec rackup -s webrick -o 127.0.0.1 -p 3000 config.ru
```

- `-s webrick`：サーバとして WEBrick を使う
- `-o 127.0.0.1`：**待受先を 127.0.0.1（自分のPCの中だけ）にする**
- `-p 3000`：ポート番号

`-o 127.0.0.1` は省略してはいけません。第10章で学んだとおり、全インターフェース（`0.0.0.0`）で待ち受けると、同じWi-Fiの他人が接続できてしまいます。

別のターミナルで確かめます。

```bash
curl -i http://127.0.0.1:3000/
```

実行結果の例です（日付やバージョンは環境で変わります）。

```
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 11
Server: WEBrick/1.4.4 (Ruby/2.6.10/2022-04-12)
Date: Thu, 01 Oct 2026 08:26:32 GMT
Connection: Keep-Alive

Hello Rack
```

アプリでは `content-type` と小文字で書きました。ところが届いたレスポンスは `Content-Type` です。WEBrick が送信するときに整えたからです。Rack の世界（アプリとサーバの間）では小文字、ネットワーク上の表記はサーバ次第、と覚えてください。

### 待受先を必ず確かめる

接続できたことは、ローカル限定である証拠になりません。OS に「今どこで待ち受けているか」を聞きます。

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

```
COMMAND  PID   USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ruby    5257 you       9u  IPv4 0xef3e32367ccded95      0t0  TCP 127.0.0.1:3000 (LISTEN)
```

最後の列が `127.0.0.1:3000` なら合格です。`*:3000` や `0.0.0.0:3000` と出たら、すぐ `Ctrl-C` で止めて設定を直します。Linux（WSL）では `ss -ltn` で確かめます。確認が終わったら、サーバは `Ctrl-C` で止めます。

### env の中身を見る

アプリの中で `env` を表示してみます。`puts` は出力が溜められて表示が遅れることがあるので、ここではすぐ出る `warn` を使います。

```ruby
# config.ru
app = Proc.new do |env|
  warn "REQUEST_METHOD=#{env["REQUEST_METHOD"]}"
  warn "PATH_INFO=#{env["PATH_INFO"]}"
  warn "QUERY_STRING=#{env["QUERY_STRING"]}"
  warn "HTTP_USER_AGENT=#{env["HTTP_USER_AGENT"]}"
  warn "HTTP_X_DEMO=#{env["HTTP_X_DEMO"]}"
  [200, { "content-type" => "text/plain" }, ["Hello Rack\n"]]
end

run app
```

自作のヘッダ `X-Demo` を付けてリクエストします。

```bash
curl -s 'http://127.0.0.1:3000/users?page=2' -H 'X-Demo: hello'
```

サーバ側のターミナルには、こう出ました。

```
REQUEST_METHOD=GET
PATH_INFO=/users
QUERY_STRING=page=2
HTTP_USER_AGENT=curl/8.7.1
HTTP_X_DEMO=hello
```

HTTP リクエストと `env` の対応が見えます。

| HTTP リクエストの部分 | env のキー |
|---|---|
| リクエスト行のメソッド `GET` | `REQUEST_METHOD` |
| パス `/users` | `PATH_INFO` |
| `?` の後ろ `page=2` | `QUERY_STRING` |
| ヘッダ `User-Agent` | `HTTP_USER_AGENT` |
| ヘッダ `X-Demo` | `HTTP_X_DEMO` |

ヘッダは「`HTTP_` を付け、大文字にし、`-` を `_` に変える」という規則で入ります。第11章で自分がやったリクエスト解析を、サーバが代わりにやって `env` に詰めてくれているのです。

### Rack::Lint で約束違反を見つける

約束を守れているかは、`Rack::Lint` という検査役が調べてくれます。わざとヘッダを大文字にしてみます。

```ruby
# lint_check.rb
require "rack"
require "rack/lint"
require "rack/mock"

bad_app = Proc.new do |env|
  [200, { "Content-Type" => "text/plain" }, ["Hello"]]
end

env = Rack::MockRequest.env_for("/")
begin
  Rack::Lint.new(bad_app).call(env)
rescue => e
  puts e.class
  puts e.message
end
```

```
Rack::Lint::LintError
uppercase character in header name: Content-Type
```

`Rack::MockRequest.env_for` は、テスト用に本物そっくりの `env` を作るメソッドです。これを使うと、サーバを起動せずにアプリを呼び出せます。この先のテストでもたくさん使います。

開発モードの `rackup` は、自動で `Rack::Lint` を挟みます。約束を破ると、起動中のサーバがエラーを報告します。

## ミドルウェア：前後に処理を挟む

### ミドルウェアとは

Rack アプリは `call(env)` を持つだけのオブジェクトでした。では「Rack アプリを包む Rack アプリ」も作れるはずです。これが**ミドルウェア**です。

たとえ話をします。郵便物（リクエスト）が受付（サーバ）から担当者（アプリ）に届くまでに、途中で「開封検査係」や「受付時刻のスタンプ係」を通るとします。各係は郵便物を見て、次の人に渡し、戻ってきた返事にも手を加えられます。この係がミドルウェアです。

### 形を見る

ミドルウェアは次の形をしています。

```ruby
class PoweredBy
  def initialize(app)
    @app = app          # 内側のアプリを覚えておく
  end

  def call(env)
    status, headers, body = @app.call(env)   # 内側に処理を任せる
    headers["x-powered-by"] = "my-middleware" # 返事に手を加える
    [status, headers, body]
  end
end
```

`config.ru` では `use` で差し込みます。

```ruby
use PoweredBy
run app
```

curl で確かめると、ヘッダが増えています。

```
HTTP/1.1 200 OK
Content-Type: text/plain
X-Powered-By: my-middleware
Content-Length: 11
...
```

ポイントは3つです。

- `initialize(app)` で**内側のアプリ**を受け取る
- `call(env)` の中で `@app.call(env)` を呼ぶ。**呼ぶ前**に書いた処理は前処理、**呼んだ後**は後処理になる
- 自分も `call(env)` を持つので、外から見れば普通の Rack アプリと区別できない

だから、ミドルウェアは何枚でも重ねられます。ログ出力、処理時間の計測、例外のキャッチ、圧縮など、どのアプリにも共通する処理はミドルウェアにすると便利です。Rails も、起動すると数十枚のミドルウェアを重ねています。

### アプリだけ差し替える

ここで Rack の価値をもう一度確かめます。`config.ru` のアプリを次に変えて再起動してください。

```ruby
app = Proc.new do |env|
  [200, { "content-type" => "text/html" }, ["<h1>New App</h1>"]]
end
```

サーバ（WEBrick）側のコードは1行も変えていません。それでも新しいアプリが動きます。これが「抽象化」です。抽象化とは、細かい違いを1つの約束の裏に隠し、取り替えられるようにすることです。

## 自作サーバを Rack 対応にする

### 何をすればよいか

第11章のサーバは、リクエストを読んで、その場でレスポンス文字列を組み立てていました。これを次の3段階に分けます。

1. リクエストを読んで **env を作る**（`build_env`）
2. **Rack アプリを呼ぶ**（`app.call(env)`）
3. 戻り値の配列を **HTTP レスポンス文字列に変換する**（`build_http_response`）

```ruby
# 考え方を示す疑似コード（このままでは動きません）
env = build_env(request)
status, headers, body = app.call(env)
response = build_http_response(status, headers, body)
```

こうすると、サーバは「どんなアプリが来るか」を知らなくてよくなります。`call` さえ持っていれば何でも動かせます。

### TDD で進める

第6章で学んだ RED → GREEN → REFACTOR で進めます。サーバ全体をいきなりテストするのは難しいので、**変換部分だけを小さなメソッドに切り出してテスト**します。ソケットを使わずに文字列だけで確かめられるので、速くて安定したテストになります。

たとえば `build_http_response` のテストは、次のような形が考えられます。

```ruby
def test_build_http_response_writes_status_line_and_headers
  response = MyServer.build_http_response(200, { "content-type" => "text/plain" }, ["Hi"])

  assert response.start_with?("HTTP/1.1 200 OK\r\n")
  assert_includes response, "content-type: text/plain\r\n"
  assert response.end_with?("\r\n\r\nHi")
end
```

`MyServer` という名前や、クラスメソッドにするかどうかは自分で決めて構いません。テストを書くと、その「決めごと」が自然と見えてきます。

`build_env` 側は、リクエスト文字列を渡したら `REQUEST_METHOD`、`PATH_INFO`、`QUERY_STRING` が正しく入っていることを確かめます。`/users?page=2` のように `?` を含むパスを分けられるかは、必ずテストにしてください。

`body` は「`each` で文字列を取り出せるもの」でした。配列以外が来ても動くように、`body.each` で連結するのが約束どおりの書き方です。また Rack の約束では、`body` が `close` を持っていれば、使い終わったあとに呼ぶことになっています。

## 並行処理：同時に何人も相手にする

### 逐次処理の限界

第11章のサーバは、`accept` → 処理 → `close` を `loop` で繰り返していました。1人の相手が終わるまで、次の人は待たされます。これを**逐次処理**といいます。窓口が1つしかない役所のようなものです。

体験してみます。アプリの処理にわざと1秒かかるようにし、2つのリクエストを**同時に**送ります。クライアント側はこう書けます。

```ruby
# concurrent_client.rb
require "net/http"

start = Time.now
threads = 2.times.map do
  Thread.new { Net::HTTP.get(URI("http://127.0.0.1:3000/")) }
end
threads.each(&:join)
puts "2リクエストの合計: #{(Time.now - start).round(2)}秒"
```

逐次処理のサーバ（1秒待ってから返す）に対して実行すると、こうなりました。

```
2リクエストの合計: 2.0秒
```

同じアプリを WEBrick（rackup）で動かすと、こうなりました。

```
2リクエストの合計: 1.0秒
```

WEBrick は接続ごとに別々の作業者を用意するので、2人を同時に相手にできます。この「作業者」が**スレッド**です。

### スレッドとは

スレッドは、1つのプログラムの中で同時に進む「作業の流れ」です。料理で言えば、1人のコックが「鍋を煮ている間に野菜を切る」ようなものです。

```ruby
# thread_basic.rb
start = Time.now
threads = 3.times.map do |i|
  Thread.new do
    sleep 1
    puts "スレッド#{i} 終了"
  end
end
threads.each(&:join)
puts "かかった時間: #{(Time.now - start).round(1)}秒"
```

```
スレッド1 終了
スレッド0 終了
スレッド2 終了
かかった時間: 1.0秒
```

1秒待つ処理を3つ実行したのに、全体は1秒で終わりました。また、終了の順番が 0, 1, 2 になっていません。**スレッドの実行順は保証されない**のです。これが後で問題の種になります。

- `Thread.new { ... }`：ブロックの中身を新しいスレッドで動かし始める
- `join`：そのスレッドが終わるまで待つ

Ruby（標準の実装）には GVL という仕組みがあり、Ruby のコードを同時に動かせるスレッドは基本的に1つです。それでも、`sleep` やネットワークの待ち時間の間は別のスレッドが動けます。Web サーバの仕事はほとんどが「待ち」なので、スレッドで大きく速くなります。

### 先にテストで期待を固定する

スレッド化をいきなり書いてはいけません。この章で一番大事なルールです。

> 並行処理を入れる前に、「何ができるようになってほしいか」をテストで決める

テストの方向性は次のとおりです。

- 1秒かかるアプリを載せたサーバに、2つのリクエストを同時に送る
- 2つとも正しい応答が返る
- 合計時間が 2秒より十分短い（たとえば 1.5秒未満）

逐次処理のサーバでは、このテストは失敗します（RED）。次に `accept` した接続ごとに `Thread.new` で処理するよう変え、テストを通します（GREEN）。

テストの中でサーバを起動するときも、待受先は `127.0.0.1` です。ポートは他と重ならない番号にし、テストの最後に必ずサーバを止めます。サーバのスレッドを止め忘れると、次のテストで「ポートが使用中」になります。

### 競合状態を体験する

スレッド化すると、新しい種類のバグが生まれます。**競合状態（race condition）**です。複数のスレッドが同じデータを同時に書き換えると、結果がおかしくなる現象です。

カウンタで体験します。`increment` は「読む → 1足す → 書く」の3段階です。その間に `Thread.pass`（他のスレッドに順番を譲る）を入れて、割り込みを起きやすくします。

```ruby
# race.rb
class Counter
  attr_reader :count

  def initialize
    @count = 0
  end

  def increment
    current = @count     # 1. 読む
    Thread.pass          # 他のスレッドに順番を譲る = わざと隙間を作る
    @count = current + 1 # 2. 書く
  end
end

counter = Counter.new
threads = 10.times.map do
  Thread.new do
    100.times { counter.increment }
  end
end
threads.each(&:join)
puts "期待: 1000 / 実際: #{counter.count}"
```

5回実行した結果です。

```
期待: 1000 / 実際: 100
期待: 1000 / 実際: 103
期待: 1000 / 実際: 100
期待: 1000 / 実際: 101
期待: 1000 / 実際: 102
```

10スレッドが100回ずつ足したので、1000になるはずです。実際は100前後で、しかも毎回違います。

### なぜこうなるのか

2つのスレッド A と B が、ほぼ同時に `increment` したときを考えます。

| 時刻 | スレッドA | スレッドB | @count |
|---|---|---|---|
| 1 | `current = 5` を読む | | 5 |
| 2 | （順番を譲る） | `current = 5` を読む | 5 |
| 3 | `@count = 6` を書く | | 6 |
| 4 | | `@count = 6` を書く | 6 |

2回足したのに、6にしかなりません。B は「A が書く前の古い値」を読んでしまったからです。**読んでから書くまでの間に、他のスレッドが割り込む**。これが競合状態の正体です。

実は `Thread.pass` を外すと、手元では `1000` が出ることが多いです。GVL のおかげで、割り込みが起きにくいからです。しかしそれは「たまたま壊れにくい」だけです。本番で負荷が上がった日にだけ壊れる、というのが並行処理のバグの一番怖いところです。

### Mutex で守る

対策は、「読む → 書く」の間に他のスレッドを入れないことです。Ruby では **Mutex**（ミューテックス、相互排除）を使います。トイレの鍵のようなもので、1人が入っている間、他の人は外で待ちます。

```ruby
def initialize
  @count = 0
  @mutex = Mutex.new
end

def increment
  @mutex.synchronize do
    current = @count
    Thread.pass
    @count = current + 1
  end
end
```

```
期待: 1000 / 実際: 1000
期待: 1000 / 実際: 1000
期待: 1000 / 実際: 1000
```

`synchronize` のブロックの中は、同時に1つのスレッドしか実行できません。

ただし、鍵をかける範囲は**必要最小限**にします。サーバ全体を1つの鍵で囲むと、結局1人ずつしか処理できず、逐次処理に逆戻りします。「どの変数が共有されていて、どこで読み書きしているか」を説明できることが大切です。

### TDD の有効性と限界

競合状態のテストは「10スレッドで1000回足したら1000になる」と書けます。Mutex がなければ失敗し、あれば通ります。ここまではTDDが役に立ちます。

しかし注意が必要です。`Thread.pass` がなければ、Mutex なしでもテストが通ることがあります。**テストが通ったことは、競合がないことの証明にならない**のです。並行処理では、テストに加えて次が欠かせません。

- 共有している状態（グローバル変数、クラス変数、クラスのインスタンス変数、キャッシュ）を洗い出す
- そもそも共有状態を持たない設計にする（リクエストごとに新しいオブジェクトを作る、など）

自作サーバの中でも、アクセス数のカウンタやログ出力の部分は共有状態になりやすい場所です。Rack アプリ側に共有状態があれば、同じ問題が起きます。

## HTTPS：通信を暗号化する

### 平文の HTTP は丸見え

第8章と第11章で見たとおり、HTTP はただの文字列です。経路の途中にいる人（同じWi-Fiの他人、ルーターの管理者など）は、その文字列を読めます。書き換えることもできます。パスワードやクッキーがそのまま流れるのは危険です。

**HTTPS** は、HTTP を **TLS** という仕組みで包んだものです。TLS は3つのことを守ります。

- **機密性**：途中の人が中身を読めない（暗号化）
- **完全性**：途中で書き換えられたら気づける
- **認証**：接続先が本物のサーバであると確かめられる（証明書）

封筒にたとえると、TLS は「中が透けない、開封したら分かる封筒」で、証明書は「差出人の身分証」です。

大事なのは、**HTTP の中身自体は変わらない**ことです。TLS の封筒を開けた後は、いつもの `GET / HTTP/1.1` が出てきます。だから、サーバのコードはほとんど変えずに HTTPS 化できます。

### 証明書と秘密鍵

TLS には2つのファイルが必要です。

- **証明書（cert.pem）**：「このサーバは localhost です」という身分証。公開してよい情報が入っている
- **秘密鍵（key.pem）**：身分証が本物であることを示すための鍵。**絶対に他人に渡してはいけない**

本物のサイトの証明書は、認証局（CA）という信頼された機関が発行します。演習では自分で自分に発行する**自己署名証明書**を使います。「自分で書いた身分証」なので、ブラウザや curl は最初は信頼しません。

### 生成する前に .gitignore を書く

**鍵を作る前に**、Git に鍵を無視させる設定をします。順番が逆だと、うっかり `git add .` した瞬間に鍵がコミットされます。

`.gitignore` に次を追加します。

```gitignore
# ローカルTLS演習の生成物
*.pem
*.key
```

次の3つのコマンドで確認します。

```bash
git check-ignore -v key.pem cert.pem
git ls-files -- key.pem cert.pem
git diff --cached --name-only
```

実行例です。

```
.gitignore:2:*.pem	key.pem
.gitignore:2:*.pem	cert.pem
```

1つ目で ignore の規則が表示されれば合格です。2つ目と3つ目は何も表示されなければ合格です（追跡中でもコミット予定でもない）。

### 証明書を生成する

```bash
umask 077
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost"
chmod 600 key.pem
```

- `umask 077`：これ以降に作るファイルを「自分だけが読み書きできる」権限で作る設定
- `-x509`：自己署名証明書を作る
- `-nodes`：秘密鍵をパスフレーズで暗号化しない（演習用の簡略化）
- `-subj "/CN=localhost"` と `subjectAltName=DNS:localhost`：この証明書は「localhost」という名前のサーバ用、と書き込む
- `chmod 600 key.pem`：念のため、所有者だけが読み書きできる権限にする

```bash
ls -l *.pem
```

```
-rw-------  1 you     wheel  1017 10  1 17:27 cert.pem
-rw-------  1 you     wheel  1704 10  1 17:27 key.pem
```

`-rw-------` は「所有者だけ読み書き可」の意味です。`git status` で `key.pem` が出てこないことも確かめてください。

秘密鍵をコミットや共有してしまったら、「漏れた」と考えます。削除コミットを足しても Git の履歴には残ります。その鍵は使うのをやめ、作り直してください。

### TLS サーバを書く

Ruby の `openssl` ライブラリで、TCP ソケットを TLS で包みます。

```ruby
require "socket"
require "openssl"

tcp_server = TCPServer.new("127.0.0.1", 3001)

ssl_context = OpenSSL::SSL::SSLContext.new
ssl_context.cert = OpenSSL::X509::Certificate.new(File.read("cert.pem"))
ssl_context.key = OpenSSL::PKey::RSA.new(File.read("key.pem"))

ssl_server = OpenSSL::SSL::SSLServer.new(tcp_server, ssl_context)
ssl_server.start_immediately = false # ハンドシェイクはループ内で行う
```

`TCPServer` を `SSLServer` で包んでいるのが分かります。この先のループは第11章とほぼ同じです。違いは、`accept` で受け取った接続に対して `ssl_socket.accept` を呼び、**TLS ハンドシェイク**（暗号の合言葉を決めるやり取り）を済ませることです。そのあとは `gets` と `write` で、いつもの HTTP を読み書きします。

ハンドシェイクは失敗することがあります。証明書を信頼しないクライアントが途中で切断するからです。そのときサーバまで落ちないよう、`OpenSSL::SSL::SSLError` を `rescue` し、`ensure` で接続を閉じます。

HTTPS の既定ポートは 443 です。443 番は管理者権限が必要なことが多いので、演習では 3001 番を使います。URL は `https://localhost:3001/` になります。

### curl で検証する

サーバを起動したら、まず待受先を確かめます。

```bash
lsof -nP -iTCP:3001 -sTCP:LISTEN
```

```
ruby    5949 you       9u  IPv4 0xdb545f52aee1c173      0t0  TCP 127.0.0.1:3001 (LISTEN)
```

次に、普通に curl でアクセスします。

```bash
curl --http1.1 https://localhost:3001/
```

```
curl: (60) SSL certificate problem: self signed certificate
```

失敗しました。これが正しい動きです。curl は「この身分証は知らない人が書いたものだ」と判断し、接続を止めました。

次に、自分で作った証明書を「信頼してよい身分証」として curl に渡します。

```bash
curl --http1.1 --cacert cert.pem https://localhost:3001/
```

```
<h1>Hello HTTPS</h1>
```

今度は成功です。`--cacert` は「この証明書を信頼の基準にして、きちんと検証して」という指定です。検証は有効のままです。

名前が違うとどうなるかも試します。証明書には「localhost」と書きました。IP アドレスで接続してみます。

```bash
curl --http1.1 --cacert cert.pem https://127.0.0.1:3001/
```

```
curl: (60) SSL: no alternative certificate subject name matches target ipv4 address '127.0.0.1'
```

同じサーバなのに失敗しました。curl は「身分証の名前（localhost）と、訪ねた相手の名前（127.0.0.1）が違う」と判断したのです。証明書の検証は、**名前まで照合している**ことが分かります。

### curl -k は比較のためだけに使う

最後に、比較のためだけに検証を無効にします。

```bash
curl --http1.1 -k https://localhost:3001/
```

```
<h1>Hello HTTPS</h1>
```

`-k`（`--insecure`）は、証明書の検証をまるごと止めます。通信は暗号化されますが、**相手が本物かどうかは確かめません**。偽物のサーバが間に入っても（中間者攻撃）気づけません。

`-k` を使うのは、この章の比較演習だけです。「エラーが出たから `-k` を付ける」のは、問題を隠すだけです。実務では、証明書のエラーは原因（名前の不一致、期限切れ、信頼設定）を調べて直します。

ブラウザで `https://localhost:3001/` を開くと、警告画面が出ます。自分で作った localhost 用の証明書なので、今回は理由が分かっています。しかし、外部のサイトで警告を無条件に回避する習慣は付けないでください。

## 実務ではこうなる

**Rack は毎日使っています。** Rails アプリのルートには `config.ru` があります。`bin/rails middleware` を実行すると、Rails が重ねているミドルウェアの一覧が見えます。ログ、セッション、例外表示などは、この章で書いたミドルウェアと同じ形です。不具合調査で「どのミドルウェアが何をしているか」を読む場面は、実務で何度も来ます。

**本番のサーバは Puma が主流です。** Puma はスレッドを複数使います。だから Rails アプリのコードも、スレッドから同時に呼ばれる前提で書く必要があります。クラス変数やクラスのインスタンス変数に、リクエストごとの情報を入れてはいけません。

**TLS はアプリの手前で終わることが多いです。** 本番では、ロードバランサや Nginx が HTTPS を受けて復号し（TLS 終端）、アプリには HTTP で渡す構成がよくあります。アプリのコードが HTTP とほぼ同じで済むのは、この章で見たとおり、TLS が下の層にあるからです。ただし「内部だから平文でよい」とは限りません。誰がその経路を見られるかで判断します。

**ジュニアがやりがちな失敗：**

- `rackup` や `rails server` を、ホスト指定なしや `-b 0.0.0.0` で起動する。カフェで他人から接続される
- クラス変数でキャッシュやカウンタを作り、本番の負荷でだけ数字が狂う
- 証明書エラーに `curl -k` や `verify_mode = VERIFY_NONE` で対処し、そのままコミットする
- `key.pem` を作ってから `.gitignore` を書き、その間に `git add .` してしまう

## 演習

### 演習13-1：Rack アプリとミドルウェア

- `~/workspace/ch13-rack/` に Gemfile・Rakefile・`test/test_helper.rb` を用意し、`bundle install` する
- 本文の `config.ru` を作り、`bundle exec rackup -s webrick -o 127.0.0.1 -p 3000 config.ru` で起動する
- 別ターミナルで `lsof -nP -iTCP:3000 -sTCP:LISTEN` を実行し、待受先を確認する
- `env` のキーを10個以上表示させ、HTTP リクエストのどこに対応するかを `notes/rack.md` に表で書く
- 処理時間を計測し、レスポンスヘッダ `x-runtime` に秒数を入れるミドルウェア `Runtime` を `lib/runtime.rb` に作る。**テストから書く**
  - RED：`Runtime.new(inner_app).call(env)` の戻り値のヘッダに `x-runtime` があるテストを書き、失敗を確認する
  - GREEN：最小の実装で通す
  - REFACTOR：名前や重複を整える
- `Rack::Lint` で包んでも通ることをテストに加える
- 確認が終わったらサーバを `Ctrl-C` で止める
- **完了条件**：
  - `lsof` の出力に `127.0.0.1:3000` と表示され、`*:3000` や `0.0.0.0:3000` ではない
  - `bundle exec rake test` が失敗0・エラー0で通る
  - `curl -i http://127.0.0.1:3000/` の結果に `X-Runtime` ヘッダが含まれる
- ヒント：テストでは `Rack::MockRequest.env_for("/")` で env を作れます。内側のアプリはテストの中で `Proc.new { |env| [200, {}, ["ok"]] }` のように用意すると、ミドルウェアだけを確かめられます。時間の計測には `Process.clock_gettime(Process::CLOCK_MONOTONIC)` が使えます。

### 演習13-2：自作サーバを Rack 対応にする

- 第11章のサーバを `~/workspace/ch13-rack/lib/` にコピーする
- `build_env`（リクエスト文字列 → env）と `build_http_response`（Rack の3要素 → レスポンス文字列）のテストを先に書く
- テストを通したら、サーバ本体が `app.call(env)` を呼ぶように書き換える
- 起動スクリプト `bin/my_server` を作り、`config.ru` と同じアプリを自作サーバで動かす。待受先は必ず `127.0.0.1`
- アプリだけを `<h1>New App</h1>` を返すものに差し替え、サーバのコードを変えずに動くことを確かめる
- **完了条件**：
  - `build_env` のテストに、`GET /users?page=2` から `PATH_INFO` が `/users`、`QUERY_STRING` が `page=2` になるケースが含まれ、通る
  - `build_http_response` のテストに、ヘッダと本文が CRLF で区切られるケースが含まれ、通る
  - 自作サーバ起動中に `curl http://127.0.0.1:3000/` が `Hello Rack` を返し、`lsof` で待受先が `127.0.0.1:3000` である
- ヒント：env に最低限必要なキーは何かを考えてください。Rack の仕様書（SPEC）に必須キーの一覧があります。`rack.input` には `StringIO.new("")` のような空の入力を入れておけます。

### 演習13-3：スレッドで並行処理し、競合状態を防ぐ

- RED：1秒かかるアプリを載せた自作サーバに、2つのリクエストを同時に送るテストを書く。「合計 1.5秒未満」を確かめ、逐次処理のままでは失敗することを確認する
- GREEN：接続ごとに `Thread.new` で処理するよう変え、テストを通す
- RED：共有カウンタ `RequestCounter` のテストを書く。10スレッド×100回 `increment` したら1000になること。本文と同じく `Thread.pass` で隙間を作り、失敗を観察する
- GREEN：Mutex で必要最小限の範囲を守り、テストを通す
- テストを10回続けて実行し、毎回通ることを確かめる
- `notes/concurrency.md` に次を書く：なぜ値が1000にならなかったか（時系列の表で）／どこに鍵をかけたか、なぜそこだけか／テストが通ってもバグがないとは言えない理由
- **完了条件**：
  - 並行処理のテストとカウンタのテストを含め、`bundle exec rake test` が10回連続で失敗0・エラー0
  - Mutex を外す（コメントアウトする）と、カウンタのテストが失敗することを確認し、その出力をメモに貼っている
- ヒント：テストの中でサーバを動かすときは、`Thread.new { server.start }` のように別スレッドで起動し、テストの最後に止めます。ポートが空くまで待つ処理や、止め忘れに注意してください。`for i in $(seq 10); do bundle exec rake test || break; done` で連続実行できます。

### 演習13-4：HTTPS で応答する

- `.gitignore` に `*.pem` と `*.key` を追加し、`git check-ignore -v key.pem cert.pem` などの3コマンドで確認する
- `umask 077` のあと、本文の `openssl req` で `key.pem` と `cert.pem` を作り、`chmod 600 key.pem` する
- 本文の TLS サーバの断片を土台に、`tls_server.rb` を完成させる。ループ部分は第11章のサーバを参考に自分で書く
- 次の4つを試し、結果と理由を `notes/tls.md` にまとめる
  - `curl --http1.1 https://localhost:3001/`
  - `curl --http1.1 --cacert cert.pem https://localhost:3001/`
  - `curl --http1.1 --cacert cert.pem https://127.0.0.1:3001/`
  - `curl --http1.1 -k https://localhost:3001/`（比較のためだけ）
- 確認が終わったらサーバを止める
- **完了条件**：
  - `git check-ignore -v key.pem cert.pem` が規則を表示し、`git ls-files -- '*.pem' '*.key'` が何も表示しない
  - `ls -l key.pem` の権限が `-rw-------`
  - `lsof -nP -iTCP:3001 -sTCP:LISTEN` が `127.0.0.1:3001` を示す
  - `--cacert` を付けた localhost への curl が `<h1>Hello HTTPS</h1>` を返す
  - メモに「`--cacert` と `-k` の違い」「127.0.0.1 で失敗した理由」が自分の言葉で書かれている
- ヒント：TLS で包んでも、HTTP のヘッダの終わり（空行）と `content-length` は必要です。改行は `\r\n` で書きます。

### 演習13-5（発展）：TLS をどこで終わらせるか

- 本番でよくある次の3つの構成を図に描き、「どこで暗号化が解かれるか」を書き込む
  - A：クライアント → HTTPS → アプリ
  - B：クライアント → HTTPS → ロードバランサ → HTTP → アプリ
  - C：クライアント → HTTPS → ロードバランサ → HTTPS → アプリ
- 各構成について、秘密鍵を置く場所、証明書を更新する手間、内部で平文になる区間のリスクを比べる
- 時間があれば、演習13-2の Rack 対応サーバを TLS 化する
- **完了条件**：`notes/tls-termination.md` に3構成の図と比較表があり、「自分ならどれを選ぶか、その条件は何か」が書かれている
- ヒント：TLS 終端の後ろのアプリは、元の通信が HTTPS だったことをどう知るでしょうか。`X-Forwarded-Proto` ヘッダを調べてください。

## 確認クイズ

**Q1.** Rack アプリが満たすべき約束を、引数と戻り値に触れて説明してください。

**Q2.** 次のレスポンスは Rack 3 の約束違反です。どこが問題ですか。
`[200, { "Content-Type" => "text/html" }, ["<h1>Hi</h1>"]]`

**Q3.** ミドルウェアの `call(env)` の中で、`@app.call(env)` の前に書いた処理と後に書いた処理は、それぞれ何に使えますか。

**Q4.** 10スレッドで100回ずつカウンタを増やすテストが、Mutex なしでも通りました。「競合は起きない」と結論してよいですか。

**Q5.** `rackup` を `-o 127.0.0.1` 付きで起動し、`curl http://localhost:3000/` が成功しました。これで待受がローカル限定だと確認できたと言えますか。

**Q6.** `curl --cacert cert.pem` と `curl -k` は、どちらも自己署名証明書のサーバに接続できます。何が違いますか。

### 解答と解説

**A1.** `call` メソッドを持ち、引数としてリクエスト情報の Hash `env` を1つ受け取ります。戻り値は `[ステータスコード, ヘッダの Hash, body]` の3要素の配列です。body は `each` で文字列を順に返すものです。この小さな約束のおかげで、サーバとアプリを自由に組み合わせられます。

**A2.** ヘッダ名に大文字が含まれています。Rack 3 ではレスポンスヘッダ名を小文字で書きます。`"content-type"` が正しい書き方です。`Rack::Lint` は `uppercase character in header name` というエラーで知らせてくれます。

**A3.** 前の処理はリクエストに対する前処理です。ログの記録、計測の開始、認証のチェックなどに使えます。後の処理はレスポンスに対する後処理です。ヘッダの追加、計測の終了、本文の加工などに使えます。前処理で内側を呼ばずに自分でレスポンスを返せば、処理を途中で止めることもできます。

**A4.** 結論してはいけません。Ruby の GVL のため、手元では割り込みが起きにくく、たまたま通ることがあります。テストが通ったことは、競合がないことの証明になりません。共有状態を洗い出し、読みと書きの間に割り込みが入りうる場所を Mutex で守るか、共有しない設計にします。

**A5.** 言えません。接続できたことは、外から接続できないことの証明になりません。`lsof -nP -iTCP:3000 -sTCP:LISTEN`（Linux なら `ss -ltn`）で、待受先が `127.0.0.1:3000` であることを確かめます。

**A6.** `--cacert` は「この証明書を信頼の基準にする」という指定で、検証は有効のままです。名前の照合も行われ、偽のサーバなら失敗します。`-k` は検証そのものを止めます。暗号化はされますが、相手が本物か確かめないので、なりすましに気づけません。`-k` は比較演習以外では使いません。

## AIの使い方（この章）

推奨する聞き方の例です。

- 「Rack の `env` に入る `SCRIPT_NAME` と `PATH_INFO` の違いを、具体的な URL の例で説明してください。コードは書かないでください」
- 「次の表は、私が考えた競合状態の時系列です。どこが間違っているか指摘してください」（自分で書いた表を貼る）
- 「`curl: (60) SSL certificate problem: self signed certificate` というエラーの意味と、`-k` を使わずに解決する方法の考え方を教えてください」

この章でやってはいけない使い方です。

- 「スレッド対応の HTTP サーバを書いて」「Rack 対応にして」と実装を丸ごと生成させること。この章の学びは、自分で分けて、自分で壊して、自分で直すことにあります
- 秘密鍵 `key.pem` の中身や、それを含むログをそのまま AI に貼り付けること。演習用でも、秘密鍵を外に出す習慣を付けないでください

## まとめ

Rack は `call(env) → [status, headers, body]` という小さな約束で、サーバとアプリを分離します。ミドルウェアは同じ約束を使って、前後に処理を重ねる仕組みです。並行処理はサーバを速くしますが、共有状態に競合状態を生みます。テストで期待を先に決め、Mutex で必要な範囲だけを守り、それでもテストは万能でないと知っておきます。HTTPS は HTTP を TLS で包むだけなので、アプリはほぼ変わりません。証明書の検証は、暗号化と同じくらい大切です。

さらに深く：[Phase 3.1](https://github.com/speee/training-web-app/blob/main/phase-3-1.md)、[Phase 3.2](https://github.com/speee/training-web-app/blob/main/phase-3-2.md)、[Phase 3.3](https://github.com/speee/training-web-app/blob/main/phase-3-3.md)、[Rack 3.2 SPEC](https://github.com/rack/rack/blob/v3.2.7/SPEC.rdoc)

## 用語

| 用語 | 意味 |
|---|---|
| Rack | Ruby の Web サーバとアプリの間の約束（インターフェース）。`call(env)` で3要素の配列を返す |
| env | リクエストの情報を詰めた Hash。メソッド、パス、ヘッダなどが入る |
| config.ru | rackup が読む設定ファイル。`run` で動かすアプリを指定する |
| rackup | Rack アプリをサーバで起動するコマンド。`-o` で待受先を指定する |
| ミドルウェア | Rack アプリを包む Rack アプリ。前後に共通処理を挟む |
| Rack::Lint | Rack の約束を守っているか検査するミドルウェア |
| 抽象化 | 細かな違いを約束の裏に隠し、部品を取り替え可能にすること |
| 逐次処理 | 1つずつ順番に処理すること |
| スレッド | 1つのプログラムの中で並行して進む作業の流れ |
| 競合状態（race condition） | 複数のスレッドが同じデータを読み書きし、結果が実行のタイミング次第で変わる不具合 |
| Mutex | 同時に1つのスレッドだけが特定の範囲を実行できるようにする鍵 |
| TLS | 通信の暗号化・改ざん検知・接続先の認証を行う仕組み。HTTPS の土台 |
| 証明書 | サーバの名前と公開鍵を結び付けた身分証。CA が署名する |
| 秘密鍵 | 証明書の持ち主だけが持つ鍵。漏れたら作り直す |
| 自己署名証明書 | 自分で自分に発行した証明書。初期状態では誰からも信頼されない |

---
[← 前の章](./12-reading-webrick.md) | [目次](../README.md) | [次の章 →](./14-build-a-framework.md)
