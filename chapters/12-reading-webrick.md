# 第12章 他人のコードを読む：WEBrick読解（Phase 2）

> **この章のゴール**：大きなコードを全部読まずに構造をつかむ方法を身につける。WEBrickがリクエストを受けてからレスポンスを返すまでの流れを、ファイル名とメソッド名つきで説明し、第11章の自作サーバと比べられるようになる。
> **目安時間**：読む 3時間 ＋ 演習 6時間
> **前提**：第11章まで

## この章で学ぶこと

- 大きなコードを読むときの戦略（問いを決める・入口を探す・呼び出しを追う・メモを取る・動かして確かめる・AIに解説させる）を使える
- インストールされたgemのソースコードの場所を、`gem` コマンドや `source_location` で探せる
- WEBrickの処理の流れ（待ち受け → 受け付け → 解析 → ハンドラ呼び出し → レスポンス送信）を、ファイルとメソッドを挙げて説明できる
- `HTTPServer`・`HTTPRequest`・`HTTPResponse` の責務を、それぞれ一言で説明できる
- 自作サーバとWEBrickの違いを挙げ、「なぜWEBrickにはそれが必要なのか」を説明できる
- 読んで立てた仮説を、テストを書いて確かめられる

## なぜ「読む力」が必要なのか

エンジニアの仕事は、書くことより読むことのほうがずっと多いと言われます。チームに入ると、最初に渡されるのは何万行もある既存のコードです。新しい機能を1行足すために、関係する数百行を読みます。不具合を直すときは、その何倍も読みます。

フレームワークやライブラリも、すべて誰かが書いたRubyのコードです。「Railsのこのメソッドは何をしているのか」がわからないとき、ドキュメントに答えがなければ、ソースコードを読むのが一番確実です。この教材が大切にしている「フレームワークを読む側になる」とは、そういうことです。

さらに、AIがコードを書いてくれる時代になりました。AIが出したコードが正しいかどうかを判断するのは、人間の「読む力」です。読めない人は、AIの出力をそのまま信じるしかありません。

この章では、Ruby製の小さなWebサーバ **WEBrick** を題材にします。第9章と第10章で、ファイル配信のために `ruby -run -e httpd` を使いましたね。あの中で動いていたのがWEBrickです。第11章で自分のサーバを作ったばかりの今なら、「自分ならこう書いた」という比較の物差しを持って読めます。これが、この章をここに置いている理由です。

WEBrickは、ローカル開発・学習・テスト向けのサーバです。本番環境で使うものではありません。

## 読む前の心構え：全部は読まない

最初に、一番大切なことを書きます。**この章では、WEBrickのすべてを理解する必要はありません。** 目的は「構造をつかむ」ことです。

知らない町を歩く場面を想像してください。町じゅうの家を1軒ずつ訪ねる人はいません。まず駅を探し、目的地までの大通りをたどり、目印になる建物を覚えます。路地の1本1本は、用事ができたときに入ればよいのです。

コードも同じです。初めて読む人がやりがちな失敗は、ファイルの1行目から順番に、全部を理解しようとすることです。すぐに知らない書き方や知らないメソッドが出てきて、そこで止まり、疲れてやめてしまいます。

このやり方では、細部にこだわるほど全体が見えなくなります。わからない行があっても、「たぶんこういうことだろう」と仮に決めて先へ進みましょう。全体の流れがわかってから戻れば、細部は驚くほど読みやすくなっています。

## 大きなコードの読み方：6つの戦略

### 戦略1：問いを決めてから読む

読む前に、「何を知りたいのか」を1つの文にします。この章なら、たとえば次のような問いです。

- 「`server.start` を呼んでから、ブラウザに `Hello from WEBrick` が届くまでに、何が起きているのか」
- 「`mount_proc` に渡したブロックは、いつ、誰から呼ばれるのか」

問いがあると、読む範囲が決まります。問いに関係のないファイル（WEBrickなら認証やCGIの部分）は、今回は開かなくてよいとわかります。逆に、問いがないまま読み始めると、どこで終わればよいかもわかりません。

### 戦略2：入口を探す

コードを読むときの出発点を**入口**（エントリーポイント）といいます。入口の探し方はいくつかあります。

- **自分が呼んでいる場所から始める**：一番確実です。自分のコードで `WEBrick::HTTPServer.new` と `server.start` を呼んでいるなら、その2つのメソッドの定義が入口になります。
- **READMEを読む**：多くのライブラリは、READMEに使い方の例があります。例の中で呼ばれているメソッドが入口です。
- **`require` されるファイルを見る**：`require "webrick"` で最初に読み込まれる `webrick.rb` には、全体の部品の一覧が書かれていることが多いです。
- **Rubyに場所を聞く**：あるメソッドがどのファイルの何行目にあるかは、Rubyに聞けば教えてくれます。方法は次の「道具」の節で説明します。

### 戦略3：呼び出しを追う

入口のメソッドを開いたら、その中で呼ばれているメソッドを1つずつ追いかけます。このとき、2つのこつがあります。

- **今のケースに関係する分岐だけを追う**：`if` や `case` で処理が分かれていたら、自分の状況で通る側だけを読みます。たとえば「HTTPSではない」「GETリクエスト」という状況です。
- **名前で推測して、先に進む**：`setup_shutdown_pipe` のようなメソッドが出てきたとします。「終了の合図のための何かを準備するのだろう」と推測してメモし、中には入らずに先へ進みます。問いに関係しそうなものだけ、後で中に入ります。

### 戦略4：メモを取る

コードを読んでいると、3つ目のメソッドに入ったあたりで、最初のメソッドが何だったかを忘れます。だから、読みながら必ずメモを取ります。この教材では、次の形の**コードリーディングノート**をすすめます。

```markdown
## 問い
server.start から、ブロックが呼ばれるまで

## 追跡メモ
1. lib/webrick/server.rb:154 GenericServer#start
   - 何をする：ループで接続を待ち、来たら start_thread に渡す
   - 疑問：IO.select って何？ → 後で調べる
2. lib/webrick/server.rb:287 GenericServer#start_thread
   - 何をする：スレッドを作り、その中で run(sock) を呼ぶ（推測）
   ...

## 仮説
- 1つの接続ごとにスレッドを作っている？ → テストか実験で確かめる
```

ポイントは、**ファイル名と行番号を必ず書く**ことです。後で読み返したとき、すぐにその場所へ戻れます。また、「何をしているか」と「疑問」を分けて書くことです。わからないことを、わからないまま書いておいてかまいません。

処理の流れは、文章だけでなく図にするとさらに整理できます。箱と矢印の簡単な図で十分です。

### 戦略5：動かして確かめる

読んで立てた仮説は、実際に動かして確かめます。「たぶんこうだろう」と「確かにこうだった」の間には、大きな差があります。

- **irbや `ruby -e` で、部品を1つだけ動かす**：たとえばリクエストを解析するクラスに、文字列を直接渡してみます。
- **テストを書く**：「このクラスは、こう動くはずだ」という仮説をテストにします。既存のコードの今の動きを記録するテストを、**特性テスト**（キャラクタリゼーションテスト）と呼びます。実務で、テストのない古いコードを変更するときによく使う方法です。
- **コピーに `p` を入れる**：処理の途中の値を見たいときは、ソースコードの**コピー**に `p` を入れて実行します。インストールされたgemそのものは書き換えません。他のプログラムにも影響するからです。

### 戦略6：AIに解説させる

AIは、コードを読むときの心強い相棒です。ただし、使い方を間違えると、読む力がまったく育ちません。

よい使い方は、**自分で読んで、わからなかった部分を、範囲を絞って聞く**ことです。

- 数行から数十行を貼り、「この部分は何をしているか」「なぜこう書くのか」を聞く
- 自分の理解を書いて、「この理解は合っているか」と聞く
- 知らない書き方（たとえば `IO.select` や `Thread::SizedQueue`）の意味を聞く

そして、AIの説明は**必ずコードと突き合わせて確かめます**。AIは、存在しないメソッドを説明したり、別の版のコードについて話したりすることがあります。説明に出てきたメソッド名を、実際に `grep` で探してみてください。

逆に、よくない使い方は「WEBrickの処理の流れを要約して」と聞いて、その答えをメモに写して終わることです。それは読んだことになりません。

## 道具：コードの場所を探す

### WEBrickを用意する

Ruby 3.0以降では、WEBrickは標準ライブラリから外れ、gemとして別に入れる必要があります。

```bash
gem install webrick
```

### gemの場所を調べる

インストールしたgemのソースコードは、PCの中のどこかにあります。場所は `gem` コマンドで調べられます。

```bash
gem which webrick
```

このPC（Ruby 2.6、WEBrickが標準で入っている環境）では、次のように表示されました。

```
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick.rb
```

Ruby 3.3で `gem install` した場合は、`.../gems/webrick-1.9.2/lib/webrick.rb` のような場所になります。場所は、Rubyの入れ方によって人それぞれです。

gemに含まれるファイルの一覧は `gem contents` で見られます。

```bash
gem contents webrick
```

```
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick.rb
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick/accesslog.rb
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick/cgi.rb
...
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick/httprequest.rb
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick/httpresponse.rb
/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick/httpserver.rb
...
```

ファイル名を眺めるだけでも、多くのことがわかります。`httprequest.rb`（リクエスト）、`httpresponse.rb`（レスポンス）、`httpserver.rb`（サーバ）、`httpservlet/`（処理を担当する部品）。これが、町の地図でいう大通りの名前です。

### 自分用のコピーを作る

読むときは、自分の作業フォルダにコピーを作ると便利です。`gem unpack` を使うと、gemの中身をそのまま取り出せます。

```bash
cd ~/workspace/ch12-reading-webrick
gem unpack webrick
```

```
Unpacked gem: '/Users/you/workspace/ch12-reading-webrick/webrick-1.9.2'
```

このコピーなら、`p` を入れたり、メモをコメントで書き込んだりしても、他のプログラムに影響しません。このPCでは、標準で入っている1.4.4ではなく、配布されている1.9.2が取り出されました。版の違いは後で説明します。

### Rubyにメソッドの場所を聞く

あるメソッドがどこに定義されているかは、`source_location` で聞けます。

```bash
ruby -rwebrick -e 'p WEBrick::HTTPServer.instance_method(:start).source_location'
ruby -rwebrick -e 'p WEBrick::HTTPServer.instance_method(:run).source_location'
ruby -rwebrick -e 'p WEBrick::HTTPServer.ancestors.take(3)'
```

```
["/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick/server.rb", 151]
["/System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/webrick/httpserver.rb", 69]
[WEBrick::HTTPServer, WEBrick::GenericServer, Object]
```

ここで面白いことがわかります。`HTTPServer` の `start` メソッドは、`httpserver.rb` ではなく `server.rb` にあります。3行目の `ancestors` を見ると、`HTTPServer` は `GenericServer` を継承しています（第5章で学んだクラスの継承です）。`start` は親の `GenericServer` で定義されているのです。

ファイルを開いて探すだけでは、この関係に気づくまで時間がかかります。Rubyに直接聞くのは、とても強力な方法です。

### grepで探す

メソッドの定義を探すには、`grep` も使えます。`-n` は行番号を、`-r` はフォルダの中をすべて探すオプションです。

```bash
grep -rn "def start" webrick-1.9.2/lib
grep -rn "def mount_proc" webrick-1.9.2/lib
```

エディタの「定義へ移動」の機能も便利ですが、`grep` はどんな環境でも使えます。「この定数はどこで使われているか」のように、呼び出し側を探すときにも役立ちます。

## WEBrickを動かす

読む前に、まず動かします。`~/workspace/ch12-reading-webrick/webrick_server.rb` に次の内容を保存してください。これは元教材が示している観察用のコードです。

```ruby
require 'webrick'

server = WEBrick::HTTPServer.new(Port: 3000, BindAddress: "127.0.0.1")

server.mount_proc '/' do |req, res|
  res.body = "Hello from WEBrick"
end

trap 'INT' do server.shutdown end

server.start
```

`BindAddress: "127.0.0.1"` を必ず書いてください。第10章で見たように、省略すると全入口で待ち受ける可能性があります。`trap 'INT'` は「`Ctrl-C` が押されたら、`server.shutdown` を呼んで止まる」という指定です。

起動して、待受先を確認します。

```bash
ruby webrick_server.rb
```

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

```
COMMAND  PID   USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ruby    5094 you       9u  IPv4 0x3605a6b25444db45      0t0  TCP 127.0.0.1:3000 (LISTEN)
```

`curl -i` で、ヘッダも見てみます。

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
Server: WEBrick/1.4.4 (Ruby/2.6.10/2022-04-12)
Date: Thu, 01 Oct 2026 08:26:10 GMT
Content-Length: 18
Connection: Keep-Alive

Hello from WEBrick
```

自分のサーバと比べてみましょう。`Content-Length` は自動で計算されています。`Connection` は `close` ではなく `Keep-Alive` です。WEBrickは、1つの接続で複数のリクエストを処理できるのです。この違いが、コードのどこから来ているのかを、これから探します。

サーバのターミナルには、次のようなログが出ます。

```
[2026-10-01 17:26:08] INFO  WEBrick 1.4.4
[2026-10-01 17:26:08] INFO  ruby 2.6.10 (2022-04-12) [universal.x86_64-darwin23]
[2026-10-01 17:26:08] INFO  WEBrick::HTTPServer#start: pid=5094 port=3000
127.0.0.1 - - [01/Oct/2026:17:26:10 JST] "GET / HTTP/1.1" 200 18
```

ログの文字列も、読解の手がかりになります。たとえば `HTTPServer#start: pid=` という文字列を `grep` すれば、そのログを出している場所がわかります。

確認が終わったら `Ctrl-C` で止め、`lsof` で何も表示されないことを確かめます。

## WEBrickの地図

読み始める前に、主なファイルの役割を表にしておきます。これは、町の地図の「大通り」です。

| ファイル | 中心となるクラス | 役割（ひとことで） |
| --- | --- | --- |
| `webrick.rb` | — | 全部品を `require` する入口 |
| `webrick/config.rb` | `WEBrick::Config` | 初期設定の値（ポート、タイムアウト、上限など） |
| `webrick/server.rb` | `GenericServer` | 待ち受け・接続の受け付け・スレッドの管理 |
| `webrick/httpserver.rb` | `HTTPServer` | 1つの接続でのHTTPのやり取りと、処理の振り分け |
| `webrick/httprequest.rb` | `HTTPRequest` | 届いたバイト列を解析して、リクエストのオブジェクトにする |
| `webrick/httpresponse.rb` | `HTTPResponse` | レスポンスの内容を持ち、バイト列にして送る |
| `webrick/httpservlet/abstract.rb` | `AbstractServlet` | 処理を担当する部品の親クラス |
| `webrick/httpservlet/prochandler.rb` | `ProcHandler` | `mount_proc` で渡したブロックを包む部品 |
| `webrick/httpservlet/filehandler.rb` | `FileHandler` | ファイル配信（`ruby -run -e httpd` が使う） |
| `webrick/httpstatus.rb` | `HTTPStatus` | ステータスコードを表す例外クラスたち |

この表は、初めから知っていたわけではありません。ファイル名とクラス名、各ファイルの先頭のコメントを読んで作ったものです。演習12-1で、自分でも同じような表を作ります。

## 処理の流れを追う

ここからは、問い「`server.start` から、ブラウザに返事が届くまでに何が起きているか」を、実際に追っていきます。行番号は、`gem unpack` で取り出した **WEBrick 1.9.2** のものです。版が違うと行番号は変わるので、メソッド名で探してください。

### 1. `HTTPServer.new`：窓口を開ける

`HTTPServer.new` を呼ぶと、親クラス `GenericServer` の `initialize`（`server.rb`）が動きます。中心となる部分を抜き出します。

```ruby
@tokens = Thread::SizedQueue.new(@config[:MaxClients])
@config[:MaxClients].times{ @tokens.push(nil) }
# ...（中略）...
unless @config[:DoNotListen]
  # ...（中略）...
  listen(@config[:BindAddress], @config[:Port])
```

`listen` は、`BindAddress` と `Port` を使って待ち受けを始めます。第11章の `TCPServer.new("127.0.0.1", 3000)` に当たる部分です。つまり、**`new` した時点で、もう待ち受けは始まっています。**

1つ目の `@tokens` は、今は「推測してメモ」の段階で十分です。`MaxClients`（同時に扱う接続の最大数）の数だけ何かを入れています。後の `start` で、これが「同時接続数の上限」に使われていることがわかります。

`config.rb` も開いてみましょう。

```ruby
:BindAddress    => nil,   # "0.0.0.0" or "::" or nil
:Port           => nil,   # users MUST specify this!!
:MaxClients     => 100,   # maximum number of the concurrent connections
```

`BindAddress` の初期値は `nil` で、コメントには `"0.0.0.0" or "::" or nil` とあります。第10章で「省略しない」と決めた理由が、ソースコードで確かめられました。読解は、ルールの根拠を自分で確かめる手段にもなります。

### 2. `start`：接続を待ち、スレッドに渡す

`GenericServer#start`（`server.rb:154`）が、サーバの心臓部です。問いに関係する部分だけを抜き出します。

```ruby
while @status == :Running
  begin
    sp = shutdown_pipe[0]
    if svrs = IO.select([sp, *@listeners])
      # ...（中略：終了の合図が来ていたら break）...
      svrs[0].each{|svr|
        @tokens.pop          # blocks while no token is there.
        if sock = accept_client(svr)
          # ...（中略）...
          th = start_thread(sock, &block)
          th[:WEBrickThread] = true
          thgroup.add(th)
        else
          @tokens.push(nil)
        end
      }
    end
```

読み取れることを、メモの形で書き出します。

- `while @status == :Running`：止める指示が来るまで、ずっと繰り返す。第11章の自作サーバの「次の接続を待つ」ループと同じ役割。
- `IO.select([sp, *@listeners])`：複数の窓口のどれかに動きがあるまで待つ。`sp` は終了の合図用の通り道（`shutdown_pipe`）らしい。だから `Ctrl-C` で止められる。
- `@tokens.pop`：コメントに「トークンがなければここで止まる」とある。`@tokens` には `MaxClients` 個しか入っていないので、**同時接続数の上限**になっている。
- `accept_client(svr)`：接続を受け付ける。第11章の `server.accept` に当たる。
- `start_thread(sock, &block)`：接続ごとに**スレッドを作って**処理を任せる。

スレッドは、1つのプログラムの中で複数の処理を同時に進める仕組みです。自作サーバは、1件の処理が終わるまで次の接続を受け付けませんでした。WEBrickは、受け付けた接続をスレッドに任せて、すぐに次の接続を待ちに戻ります。銀行の窓口が1つから、最大100個に増えたようなものです。並行処理は第13章で詳しく学びます。

### 3. `start_thread` と `run`：1つの接続を担当する

`start_thread`（`server.rb:287`）は、`Thread.start` でスレッドを作り、その中で `run(sock)` を呼びます。そして、`ensure` の中で必ず `sock.close` します。`ensure` は、途中で例外が起きても必ず実行される部分です（第5章）。接続が漏れないように、確実に閉じているのです。

`run` は `HTTPServer` 側（`httpserver.rb:69`）にあります。

```ruby
def run(sock)
  while true
    req = create_request(@config)
    res = create_response(@config)
    server = self
    begin
      timeout = @config[:RequestTimeout]
      while timeout > 0
        break if sock.to_io.wait_readable(0.5)
        break if @status != :Running
        timeout -= 0.5
      end
      raise HTTPStatus::EOFError if timeout <= 0 || @status != :Running
      raise HTTPStatus::EOFError if sock.eof?
      req.parse(sock)
      # ...（中略）...
      server.service(req, res)
```

ここも、メモの形で読み解きます。

- `while true`：1つの接続の中で、リクエストを**繰り返し**処理する。これが `Connection: Keep-Alive` の正体。
- `create_request` / `create_response`：リクエストとレスポンスの「空のオブジェクト」を先に作っている。
- `wait_readable(0.5)` のループ：0.5秒ずつ、データが届くのを待つ。`RequestTimeout`（初期値30秒）を過ぎたら諦める。**タイムアウト**の仕組みです。自作サーバには、これがありませんでした。
- `req.parse(sock)`：ソケットから読んで、リクエストを解析する。
- `server.service(req, res)`：解析したリクエストを、担当の部品に振り分ける。

`run` の後半（ここでは省略）には `rescue` が並び、`ensure` の中で `res.send_response(sock)` を呼んでいます。最後に、keep-aliveを続けない条件なら `break` してループを抜けます。

### 4. `HTTPRequest#parse`：バイト列をオブジェクトにする

`parse`（`httprequest.rb:193`）の中では、`read_request_line` と `read_header` が呼ばれます。`read_request_line`（`httprequest.rb:451`）を見てみましょう。

```ruby
def read_request_line(socket)
  @request_line = read_line(socket, MAX_URI_LENGTH) if socket
  raise HTTPStatus::EOFError unless @request_line

  @request_bytes = @request_line.bytesize
  if @request_bytes >= MAX_URI_LENGTH and @request_line[-1, 1] != LF
    raise HTTPStatus::RequestURITooLarge
  end
  # ...（中略）...
  if /^(\S+) (\S++)(?: HTTP\/(\d+\.\d+))?\r\n/mo =~ @request_line
    @request_method = $1
    @unparsed_uri   = $2
    @http_version   = HTTPVersion.new($3 ? $3 : "0.9")
  else
    rl = @request_line.sub(/\x0d?\x0a\z/o, '')
    raise HTTPStatus::BadRequest, "bad Request-Line '#{rl}'."
  end
end
```

第11章で自分が書いた `parse_request_line` と比べてみてください。

- **長さの上限がある**：`MAX_URI_LENGTH`（2083バイト）を超えるリクエスト行は、`RequestURITooLarge`（414）として断ります。少し上には、ヘッダ全体の上限 `MAX_HEADER_LENGTH`（112KB）もあります。巨大なリクエストでメモリを使い果たされないための防御です。
- **形を正規表現で確かめる**：メソッド・空白1つ・URI・空白1つ・HTTPの版・`\r\n`、という形に合わなければ `BadRequest`（400）です。
- **エラーを例外で表す**：問題があれば `raise` します。この例外が `run` の `rescue` で受け止められ、エラーのレスポンスになります。

ここで1つ、版による違いを紹介します。このPCに標準で入っている1.4.4では、同じ部分の正規表現が `/^(\S+)\s+(\S++)(?:\s+HTTP\/(\d+\.\d+))?\r?\n/mo` でした。空白は「1つ以上」、行の終わりは `\n` だけでもよい、という緩い形です。1.9.2では「空白はちょうど1つ」「行の終わりは `\r\n`」と厳しくなっています。

なぜ厳しくしたのでしょうか。第10章の「防御の不足」の表にあった「HTTPの厳密な検証」を思い出してください。サーバとその前にある別の仕組みとで、リクエストの区切り方の解釈がずれると、攻撃に使われることがあります。だから、仕様どおりの形だけを受け付けるように変わってきたと考えられます。**同じライブラリでも、版によってコードは変わります。** 読むときは、自分が読んでいる版を必ず確かめてください。

### 5. `service`：担当の部品に振り分ける

`HTTPServer#service`（`httpserver.rb:125`）は、リクエストのパスから担当の部品を探して、処理を任せます。

```ruby
servlet, options, script_name, path_info = search_servlet(req.path)
raise HTTPStatus::NotFound, "`#{req.path}' not found." unless servlet
# ...（中略）...
si = servlet.get_instance(self, *options)
# ...（中略）...
si.service(req, res)
```

WEBrickでは、処理を担当する部品を**サーブレット**と呼びます。`search_servlet` は、`mount` や `mount_proc` で登録したパスの中から、リクエストのパスに合うものを探します。見つからなければ `NotFound`（404）を `raise` します。自作サーバのルーティングと同じ役割です。

では、`mount_proc` で渡したブロックはどうなっているのでしょうか。`mount_proc`（`httpserver.rb:164`）を見ると、ブロックを `HTTPServlet::ProcHandler.new(proc)` で包んで `mount` していました。

サーブレットの `service` は、親クラス `AbstractServlet`（`httpservlet/abstract.rb:102`）にあります。

```ruby
def service(req, res)
  method_name = "do_" + req.request_method.gsub(/-/, "_")
  if respond_to?(method_name)
    __send__(method_name, req, res)
  else
    raise HTTPStatus::MethodNotAllowed,
          "unsupported method `#{req.request_method}'."
  end
end
```

これは面白い書き方です。リクエストのメソッドが `GET` なら、`"do_GET"` という**メソッド名の文字列を作り**、`__send__` でそのメソッドを呼びます。`do_GET` が定義されていなければ `MethodNotAllowed`（405）です。

`ProcHandler`（`httpservlet/prochandler.rb`）を見ると、`do_GET` の中身は `@proc.call(request, response)` でした。そして `alias do_POST do_GET` で、POSTも同じ処理にしています。

ここで、最初の問いの1つ「`mount_proc` に渡したブロックは、いつ、誰から呼ばれるのか」に答えられます。**`AbstractServlet#service` が `do_GET` を呼び、`ProcHandler#do_GET` が、あなたのブロックを `call` している**のです。ブロックの引数 `req` と `res` は、`run` の中で作られたあのオブジェクトです。

### 6. `send_response`：レスポンスを送る

ブロックの中で `res.body = "Hello from WEBrick"` と書きました。この時点では、まだ何も送られていません。`res` というオブジェクトに、値を入れただけです。

実際に送るのは、`run` の `ensure` の中で呼ばれる `HTTPResponse#send_response`（`httpresponse.rb:238`）です。

```ruby
def send_response(socket) # :nodoc:
  begin
    setup_header()
    send_header(socket)
    send_body(socket)
  rescue Errno::EPIPE, Errno::ECONNRESET, Errno::ENOTCONN => ex
    @logger.debug(ex)
    @keep_alive = false
  rescue Exception => ex
    @logger.error(ex)
    @keep_alive = false
  end
end
```

3つの手順に分かれています。

- `setup_header`：送る直前に、ヘッダを整えます。`Content-Length` を本文から計算したり、`Connection` を決めたりします。自分で `Content-Length` を書かなくてよかったのは、ここで計算してくれていたからです。中を見ると `@body.bytesize` を使っています。第11章で学んだ「文字数ではなくバイト数」が、ここでも守られています。
- `send_header`：ステータス行とヘッダを送ります。
- `send_body`：本文を送ります。

送っている途中で相手が接続を切った場合（`EPIPE` や `ECONNRESET`）も、例外を受け止めて、keep-aliveをやめるだけにしています。サーバ全体は止まりません。

### 流れの全体像

ここまでの流れを図にまとめます。演習12-3では、これを自分の言葉と自分の調べた行番号で書き直します。

```
[あなたのコード] HTTPServer.new(Port:, BindAddress:)
   └─ GenericServer#initialize  … 設定を読み、listen で待ち受け開始
[あなたのコード] server.start
   └─ GenericServer#start       … IO.select で待つ → accept_client
        └─ start_thread         … 接続ごとにスレッド。最後に必ず sock.close
             └─ HTTPServer#run  … keep-alive のループ。タイムアウトつきで待つ
                  ├─ HTTPRequest#parse      … リクエスト行・ヘッダを解析（上限・形の検査）
                  ├─ HTTPServer#service     … search_servlet でパスから担当を探す
                  │    └─ AbstractServlet#service … "do_" + メソッド名 を呼ぶ
                  │         └─ ProcHandler#do_GET … あなたのブロックを call
                  └─ HTTPResponse#send_response … setup_header → send_header → send_body
```

## 3つのクラスの責務

ここまで読むと、3つの中心的なクラスの**責務**（そのクラスが引き受けている仕事）が見えてきます。

| クラス | 責務（ひとことで） |
| --- | --- |
| `HTTPServer` | 接続ごとのHTTPのやり取りを回し、リクエストを担当の部品に振り分ける |
| `HTTPRequest` | 届いたバイト列を検査・解析し、メソッドやパスを取り出せる形にする |
| `HTTPResponse` | 返す内容を受け取って保持し、送る直前にHTTPの形に整えて送る |

演習では、自分の言葉で書き直してください。「ひとことで言えない」と感じたら、そのクラスが多くの仕事を抱えすぎている、というサインかもしれません。

## 自作サーバとの比較

第11章の自作サーバと、WEBrickを並べてみます。

| 観点 | 自作サーバ（第11章） | WEBrick | WEBrickに必要な理由 |
| --- | --- | --- | --- |
| 同時接続 | 1件ずつ順番に処理 | 接続ごとにスレッド、上限100 | 遅い1人のせいで全員が待たされないように |
| 接続の再利用 | 1リクエストで閉じる | keep-aliveのループ | 接続を作り直す時間を減らすため |
| タイムアウト | なし | `RequestTimeout`（30秒） | 何も送ってこない接続に居座られないように |
| 入力の上限 | なし | リクエスト行2083バイト、ヘッダ112KB | 巨大な入力でメモリを使い果たされないように |
| 形の検査 | `split` で分けるだけ | 正規表現で厳密に確認し、400 | 解釈のずれを攻撃に使われないように |
| エラー処理 | 例外で止まることがある | `HTTPStatus` の例外を `rescue` してエラー応答 | 1件のエラーでサーバ全体が止まらないように |
| 資源の回収 | 自分で `close` | `ensure` で必ず `close` | 例外が起きても接続を漏らさないように |
| ルーティング | `if` やHash | `mount` とサーブレット | 処理の部品を差し替え・追加しやすく |
| リクエスト・レスポンス | 文字列とHash | 専用のオブジェクト | 解析済みの値や、送る前の加工をまとめて扱う |
| ログ | `p` | ロガーとアクセスログ | 運用中に何が起きたかを後から追えるように |

この表の右側の多くは、第10章の「防御の不足」の5つの領域（資源の上限・タイムアウト・資源の回収・HTTPの厳密な検証・運用上の防御）に対応しています。自作サーバに足りなかったものは、実際のサーバではちゃんとコードとして存在している。それが確かめられたことが、この章の大きな収穫です。

ただし、WEBrickでさえ本番向けではありません。本番で使われるサーバ（Pumaなど）は、さらに多くの工夫をしています。

## なぜこの設計なのか

最後に、設計の意図を考えます。正解が1つではない問いなので、自分の考えを持つことが大切です。

### なぜリクエストとレスポンスはオブジェクトなのか

自作サーバでは、リクエストを文字列やHashで扱いました。小さなサーバならそれで十分です。WEBrickでは、`HTTPRequest` が「解析」「検査」「値の取り出し（`req.path` や `req.query`）」をまとめて引き受けています。使う側は、バイト列の細かい形を気にせず、`req.path` と書くだけで済みます。

`HTTPResponse` も同じです。使う側は `res.body = ...` と値を入れるだけで、`Content-Length` の計算や、ヘッダの形式は `HTTPResponse` が引き受けます。**難しいことを1か所に閉じ込めて、使う側を楽にする**。これがオブジェクトにする一番の理由です。

### なぜ処理をブロックで渡すのか

`mount_proc` は、「何をするか」をブロックで受け取ります。サーバ側は「いつ呼ぶか」を決め、利用者は「呼ばれたら何をするか」だけを書きます。役割がきれいに分かれています。第4章で学んだブロックの考え方が、ここで生きています。第13章で学ぶRackも、同じ考え方の延長にあります。

### なぜエラーを例外で表すのか

WEBrickでは、404や400を `raise HTTPStatus::NotFound` のように例外で表しています。処理の深いところで問題が見つかっても、`raise` すれば一気に `run` の `rescue` まで戻れます。途中のメソッドが、いちいち「エラーかどうか」を戻り値で伝え合う必要がありません。一方で、例外を使いすぎると流れが追いにくくなる、という欠点もあります。

## 実務ではこうなる

実務では、この章の読み方を毎日のように使います。

- **gemのソースを読む**：Bundlerを使うプロジェクトでは、`bundle info --path gem名` でそのプロジェクトが使っているgemの場所がわかります。READMEに書いていない動きは、ソースで確かめるのが一番確実です。
- **Railsの中を読む**：「`find` が見つからないときに、なぜ404になるのか」のような疑問は、Railsのソースを読めば答えがあります。第18章で実際に試します。
- **既存のアプリのコードを読む**：配属されて最初の仕事は、たいてい既存のコードを読んで、小さな修正をすることです。入口（ルーティングやコントローラ）から追う方法は、そのまま使えます。
- **テストのないコードを変更する前に、特性テストを書く**：今の動きを記録するテストを先に書けば、変更で何かを壊したときに気づけます。

ジュニアがやりがちな失敗です。

- ファイルの1行目から全部読もうとして、数時間かけても全体像がつかめない。
- インストールされたgemを直接書き換えて調査し、元に戻し忘れて、他のプロジェクトが壊れる。
- 読んでいるソースの版と、プロジェクトで使っている版が違うことに気づかず、存在しない動きを前提に話してしまう。
- AIの要約を読んで「理解した」つもりになり、レビューで質問されて答えられない。

## 演習

演習は `~/workspace/ch12-reading-webrick/` で行います。

```bash
mkdir -p ~/workspace/ch12-reading-webrick
cd ~/workspace/ch12-reading-webrick
git init
```

`gem unpack` で取り出したフォルダは、自分のコードではないのでコミットしません。`.gitignore` に `webrick-*/` を追加しておきます。第11章のコードと比べるので、`~/workspace/ch11-http-server/` もすぐ開けるようにしておきます。

### 演習12-1：WEBrickを動かし、ソースの場所を突き止める

- 本文の `webrick_server.rb` を保存して起動し、`lsof` で待受先を確かめます。`curl -i http://127.0.0.1:3000/` の結果を `notes.md` に貼ります。止めた後に、もう一度 `lsof` を実行します。
- `gem which webrick`・`gem contents webrick` の結果と、`gem unpack webrick` で取り出したフォルダの版を `notes.md` に記録します。
- `source_location` を使い、`start`・`run`・`mount_proc`・`service` の4つのメソッドの定義場所を調べます。
- 本文の「WEBrickの地図」を見ずに、`lib/webrick/` の主なファイル5つ以上について「役割をひとことで」書いた表を作ります。各ファイルの先頭のコメントとクラス名を手がかりにします。終わったら本文の表と比べます。
- **完了条件**：
  - `notes.md` に、起動中の `lsof` の `NAME` が `127.0.0.1:3000` である出力、`curl -i` の出力、停止後に何も表示されなかった記録がある
  - 4つのメソッドについて、ファイル名と行番号が記録されている
  - 5つ以上のファイルの役割表がある
- ヒント：`HTTPServer` の `start` が `httpserver.rb` にない理由は、`ancestors` の結果からわかります。

### 演習12-2：入口から start を読む

- `reading-notes.md` に、本文の「戦略4」の形でコードリーディングノートを作ります。問いは「`HTTPServer.new` と `start` の中で、何が起きているか」です。
- 次の問いに、ファイル名と行番号をつけて答えます。
  1. `HTTPServer.new` を呼んだとき、待ち受けはどこで始まるか
  2. `BindAddress` を省略すると、何が渡されるか（`config.rb` を根拠に）
  3. `start` のループは、どんな条件で終わるか
  4. 同時に処理できる接続の数は、どこでどう制限されているか
  5. `Ctrl-C` を押してから、ループが終わるまでに何が起きるか（推測でもよい。推測なら「推測」と書く）
- **完了条件**：5つの問いすべてに答えがあり、各答えにファイル名と行番号が1つ以上ついている。推測で書いた部分が「推測」と区別されている
- ヒント：5番は、`trap 'INT'` → `shutdown` → `shutdown_pipe` の順に追うと手がかりがつかめます。わからない部分は、わからないと書いてかまいません。

### 演習12-3：リクエスト処理の流れを図にする

- `flow.md` に、次の6つの段階を順に並べた図か文章を書きます。各段階に、担当するファイル・クラス・メソッド名を書きます。
  1. TCP接続の受け付け
  2. リクエストの読み込み
  3. リクエストオブジェクトの生成と解析
  4. ハンドラ（`mount_proc` のブロック）の呼び出し
  5. レスポンスの生成
  6. クライアントへの返却と、接続を閉じる処理
- さらに、「keep-aliveのとき、2つ目のリクエストはどこから処理が始まるか」を1文で書きます。
- **完了条件**：6段階すべてにファイル名とメソッド名がついている。本文の図をそのまま写したのではなく、自分で `grep` や `source_location` で確かめた行番号が含まれている
- ヒント：自分の環境のWEBrickの版を使ってください。本文の行番号と違っていて当然です。違いがあれば、その違いもメモしておきましょう。

### 演習12-4：読んだ仮説をテストで確かめる

- `test/webrick_request_test.rb` を作り、`WEBrick::HTTPRequest` の動きを確かめる特性テストを書きます。ソケットの代わりに `StringIO`（文字列をファイルのように読めるクラス）を使うと、サーバを起動せずにテストできます。書き方の例を1つ示します。

```ruby
require "minitest/autorun"
require "webrick"
require "stringio"

class WEBrickRequestTest < Minitest::Test
  def parse(raw)
    req = WEBrick::HTTPRequest.new(WEBrick::Config::HTTP)
    req.parse(StringIO.new(raw))
    req
  end

  def test_splits_path_and_query
    req = parse("GET /hello?name=Ota HTTP/1.1\r\nHost: localhost:3000\r\n\r\n")
    assert_equal "/hello", req.path
    assert_equal({ "name" => "Ota" }, req.query)
  end
end
```

- この例に加えて、本文で読んだ内容から仮説を3つ以上立て、テストにします。少なくとも1つは「長すぎるリクエスト行が `WEBrick::HTTPStatus::RequestURITooLarge` になる」ことを確かめるテストにします。
- 同じ「3000文字のパス」を、WEBrickと第11章の自作サーバの両方に `curl` で送り、結果を比べて `notes.md` に書きます。
- **完了条件**：
  - `ruby test/webrick_request_test.rb` が `0 failures, 0 errors` で終わり、例を含めて4つ以上のテストがある
  - WEBrickでは `414` が返ることを `curl -s -o /dev/null -w '%{http_code}\n' ...` で確かめた結果と、自作サーバでの結果が `notes.md` に記録されている
- ヒント：例外を確かめるには `assert_raises(例外クラス) { ... }` を使います。長いパスは `"a" * 3000` で作れます。`curl` に渡すときは `"http://127.0.0.1:3000/$(ruby -e 'print "a"*3000')"` のように書けます。

### 演習12-5：比較レポートを書く

- `report.md` に、次の4つを書きます。
  1. `HTTPServer`・`HTTPRequest`・`HTTPResponse` の責務を、それぞれ一言で
  2. 自作サーバになかったものを5つ以上。それぞれ「WEBrickではどのファイル・メソッドにあるか」と「なぜ必要か」をつける
  3. 次の3つの問いへの自分の答え：「なぜリクエストはオブジェクトなのか」「なぜレスポンスもオブジェクトなのか」「なぜ処理をブロックで渡すのか」
  4. 気づき：「自分の実装は、なぜ単純すぎたのか」を3〜5文で
- 第10章の `security-record.md` を開き、WEBrickを読んで知ったことをもとに、自作サーバの各領域の状態を見直します。
- **完了条件**：`report.md` の4項目がすべて埋まり、2番の5つ以上の項目すべてにファイル名とメソッド名がついている。`security-record.md` の更新がコミットされている
- ヒント：本文の比較表を写すのではなく、自分で確かめた箇所を書きます。本文の表にない違いを1つでも見つけられたら、立派な成果です。

### 演習12-6（発展）：ログ・例外・ファイル配信を追う

- 次の3つから1つ以上を選び、コードリーディングノートを作ります。
  1. **ログ**：アクセスログの1行（`"GET / HTTP/1.1" 200 18`）は、どこで、どの情報から作られているか
  2. **例外**：`mount_proc` のブロックの中で `raise "oops"` したら、ブラウザには何が返り、ログには何が出るか。実際に試し、コードのどこを通ったかを説明する
  3. **ファイル配信**：`FileHandler#prevent_directory_traversal`（`httpservlet/filehandler.rb`）を読み、第11章の自分の `StaticFiles.resolve` と比べる。WEBrickが何をしていて、何をしていないかを書く
- **完了条件**：選んだテーマについて、ファイル名と行番号つきのノートがあり、実際に動かして確かめた結果（ログやテスト）が1つ以上含まれている
- ヒント：2を試すときも、`BindAddress: "127.0.0.1"` と `lsof` の確認を忘れずに。3は、第11章で学んだ落とし穴（エンコード、シンボリックリンク、境界の確認）の観点で読むと、違いが見えてきます。

## 確認クイズ

**Q1.** 初めてのライブラリのコードを読むとき、最初にすべきことは何か。「1行目から順に読む」以外で答えよ。

**Q2.** `WEBrick::HTTPServer` の `start` メソッドの定義を `httpserver.rb` で探したが、見つからなかった。なぜか。どうすれば場所がわかるか。

**Q3.** WEBrickのレスポンスは `Connection: Keep-Alive` だった。1つの接続で複数のリクエストを処理しているのは、どのメソッドのどんな仕組みか。

**Q4.** `mount_proc` に渡したブロックは、どのような順で呼ばれるか。クラスとメソッドの名前を3つ以上挙げて説明せよ。

**Q5.** 自作サーバにはなく、WEBrickにはある「入力の上限」の例を1つ挙げ、それがない場合に何が起きうるかを説明せよ。

### 解答と解説

1. **問いを決めて、入口を探す。** 「何を知りたいか」を1文にし、自分が呼んでいるメソッド（`new` や `start`）の定義を入口にします。問いに関係ないファイルは読みません。
2. **`start` は親クラスの `GenericServer` に定義されているから。** `HTTPServer` は `GenericServer` を継承しています。`WEBrick::HTTPServer.instance_method(:start).source_location` で、ファイルと行番号がわかります。`ancestors` で継承関係も確かめられます。
3. **`HTTPServer#run` の `while true` のループ。** 1つの接続の中で、リクエストの解析・処理・送信を繰り返します。keep-aliveを続けない条件（HTTP/1.0、`Connection: close` など）になると `break` で抜けます。
4. **例：`HTTPServer#service` が `search_servlet` で担当のサーブレット（`ProcHandler`）を探す → `AbstractServlet#service` が `"do_" + メソッド名` から `do_GET` を呼ぶ → `ProcHandler#do_GET` がブロックを `call` する。** `run` の中で作られた `req` と `res` が、ブロックの引数になります。
5. **例：リクエスト行の長さの上限 `MAX_URI_LENGTH`（2083バイト）。** 上限がないと、とても長いリクエスト行を送られたときに、サーバがそれをすべてメモリに読み込もうとします。大量に送られると、メモリを使い果たして止まるおそれがあります。ヘッダ全体の上限 `MAX_HEADER_LENGTH` を挙げても正解です。

## AIの使い方（この章）

推奨する聞き方の例です。

- 「WEBrick 1.9.2 の `GenericServer#start` の次の部分を読みました（コードを貼る）。`@tokens.pop` が同時接続数の上限になっていると理解しましたが、合っていますか。違っていたら、どの行を読み直せばよいか教えてください。」
- 「`IO.select` と `Thread::SizedQueue` が何をするものか、WEBrickの文脈で説明してください。説明に出てくるメソッドは、私が実際にソースで確かめます。」
- 「私の書いた処理フロー図です（貼る）。抜けている段階や、順番がおかしいところがあれば指摘してください。」

この章でやってはいけない使い方です。

- 「WEBrickの処理の流れを要約して」と聞き、その答えをそのまま `flow.md` やレポートに写すこと。この章の目的は、自分でコードを追う経験そのものです。AIの要約は、読んだ後に自分の理解と比べるために使います。
- AIが示したファイル名・行番号・メソッド名を、自分のソースで確かめずに使うこと。版が違えば行番号は変わりますし、AIは存在しないメソッドを挙げることもあります。

## まとめ

大きなコードは、全部読まずに「問い」と「入口」から始め、呼び出しを追い、メモを取り、動かして確かめます。WEBrickは、`GenericServer` が接続を受けてスレッドに渡し、`HTTPServer#run` がkeep-aliveのループでリクエストを解析・振り分け・送信しています。自作サーバに足りなかった上限・タイムアウト・資源の回収・厳密な検証は、WEBrickではコードとして存在していました。読む力は、フレームワークを理解し、AIの出力を判断するための土台です。

さらに深く：元教材の [Phase 2](https://github.com/speee/training-web-app/blob/main/phase-2.md) と、[WEBrick 1.9.2 のソースコード](https://github.com/ruby/webrick/tree/v1.9.2) をGitHub上で読むこともできます。

## 用語

| 用語 | 意味 |
| --- | --- |
| WEBrick | Ruby製の小さなWebサーバ。学習・開発・テスト向け |
| 入口（エントリーポイント） | コードを読み始める出発点。自分が呼んでいるメソッドなど |
| コードリーディングノート | ファイル名・行番号・何をしているか・疑問を書き留めるメモ |
| `gem contents` | gemに含まれるファイルの一覧を表示するコマンド |
| `gem unpack` | gemの中身を作業フォルダに取り出すコマンド |
| `source_location` | メソッドが定義されているファイルと行番号を返すメソッド |
| 継承 | 親クラスのメソッドを子クラスが引き継ぐこと |
| スレッド | 1つのプログラムの中で、複数の処理を同時に進める仕組み |
| keep-alive | 1つの接続で複数のリクエストをやり取りすること |
| タイムアウト | 決めた時間内に終わらなければ打ち切ること |
| サーブレット | WEBrickで、リクエストを処理する担当の部品 |
| 責務 | そのクラスやメソッドが引き受けている仕事 |
| 特性テスト | 既存のコードの今の動きを記録するためのテスト |
| StringIO | 文字列を、ファイルやソケットのように読み書きできるクラス |

---
[← 前の章](./11-http-server-from-socket.md) | [目次](../README.md) | [次の章 →](./13-rack-concurrency-tls.md)
