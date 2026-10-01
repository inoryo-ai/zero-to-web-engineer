# 未経験からWebエンジニアへ（完全版）

プログラミング未経験の人が、最初の章から順に読み、すべての演習をやり切ることで、Ruby / Rails のWeb開発チームにジュニアエンジニアとして参加できる状態を目指す教材です。

元にしたのは [speee/training-web-app](https://github.com/speee/training-web-app) の考え方です。

- **抽象を降りて、また登る**：Socket → HTTP → Webサーバ → Rack → フレームワーク → アプリの順に、自分の手で作ってから Rails に戻ります。
- **フレームワークを読む側になる**：便利な道具を「なぜ動くのか説明できる」状態で使います。
- **AIは実装者ではなく、解説者・壁打ち相手**：コードを書く練習そのものが学習の中心です。

## この教材を終えるとできること

- Rails で、認証・認可・CRUD・バリデーション・テスト付きのWebアプリを、要件から設計して作れる
- フレームワークの裏側（HTTP、Rack、ルーティング、ORM、SQL）を、自作した経験をもとに説明できる
- XSS・CSRF・SQLインジェクションなどの代表的な攻撃を理解し、対策とテストを書ける
- Git / GitHub で Issue・ブランチ・PR・コードレビューを使い、チームで開発できる
- エラーを読み、調べ、最小の再現を作って質問できる
- AIの出力を鵜呑みにせず、読んで評価できる

ただし、教材を終えた時点はゴールではなくスタートラインです。現場では、この教材にない技術や慣習にも出会います。学び続ける前提で読んでください（[第27章](./chapters/27-job-ready-checklist.md)）。

## 必要な時間

各章の目安を合計すると、**読む約85時間＋演習約270〜290時間、合わせて約360〜380時間**です。
これは「つまずきが少ない場合」の目安です。完全未経験だと、1.5〜2倍（500〜750時間）かかることも珍しくありません。

| 学習ペース | 期間の目安 |
| --- | --- |
| 週10時間 | 9〜18か月 |
| 週20時間 | 4.5〜9か月 |
| 週40時間（研修として専念） | 2〜4.5か月 |

まず第1部を終えたら、かかった時間を記録して計画を見直してください。

1日8時間の研修として進める場合の章ごと・日ごとの予定表は [SCHEDULE.md](./SCHEDULE.md) にあります。

## 目次

### 第1部 準備
| 章 | 内容 | 目安 |
| --- | --- | --- |
| [第1章](./chapters/01-computer-and-terminal.md) | PCとターミナル（Ruby・Git・エディタの導入） | 5時間 |
| [第2章](./chapters/02-git-and-github.md) | Git と GitHub | 5時間 |

### 第2部 プログラミングの基礎（Ruby）
| 章 | 内容 | 目安 |
| --- | --- | --- |
| [第3章](./chapters/03-ruby-basics.md) | Ruby入門1：値・変数・条件分岐・繰り返し | 6.5時間 |
| [第4章](./chapters/04-ruby-collections-methods.md) | Ruby入門2：配列・Hash・メソッド・ブロック | 8時間 |
| [第5章](./chapters/05-ruby-objects.md) | Ruby入門3：クラス・モジュール・例外・ファイル・require | 8時間 |
| [第6章](./chapters/06-testing-and-tdd.md) | テストとTDD入門（Minitest） | 9時間 |
| [第7章](./chapters/07-debugging-and-research.md) | エラーの読み方・デバッグ・調べ方・質問の仕方 | 6.5時間 |

### 第3部 Webの基礎
| 章 | 内容 | 目安 |
| --- | --- | --- |
| [第8章](./chapters/08-network-and-http.md) | ネットワークとHTTP | 6時間 |
| [第9章](./chapters/09-html-css-js.md) | HTML・CSS・JavaScriptの最低限（フォーム中心） | 7時間 |
| [第10章](./chapters/10-security-basics.md) | 安全上の注意とWebセキュリティの基礎 | 4.5時間 |

### 第4部 低レイヤーから作る
| 章 | 内容 | 目安 |
| --- | --- | --- |
| [第11章](./chapters/11-http-server-from-socket.md) | SocketでHTTPサーバを作る | 13時間 |
| [第12章](./chapters/12-reading-webrick.md) | 他人のコードを読む：WEBrick読解 | 9時間 |
| [第13章](./chapters/13-rack-concurrency-tls.md) | Rack・並行処理・HTTPS | 16時間 |
| [第14章](./chapters/14-build-a-framework.md) | フレームワークを自作する | 36時間 |
| [第15章](./chapters/15-sql-and-database.md) | SQLとデータベース設計 | 14時間 |
| [第16章](./chapters/16-build-an-orm.md) | ORMを自作する | 30時間 |
| [第17章](./chapters/17-crud-app.md) | 自作部品でCRUDアプリを作る | 23時間 |

### 第5部 実務のWeb開発
| 章 | 内容 | 目安 |
| --- | --- | --- |
| [第18章](./chapters/18-rails.md) | Railsで作り直す：自作との対応で理解する | 25時間 |
| [第19章](./chapters/19-auth-session.md) | Cookie・セッション・認証・認可 | 20時間 |
| [第20章](./chapters/20-security-in-practice.md) | セキュリティ実践（OWASP Top 10・秘密情報・依存関係） | 9時間 |
| [第21章](./chapters/21-api-and-async.md) | API・JSON・外部連携・非同期処理 | 11時間 |
| [第22章](./chapters/22-testing-strategy-ci.md) | テスト戦略とCI | 9.5時間 |
| [第23章](./chapters/23-deploy-and-operations.md) | デプロイと運用（環境・ログ・監視・障害対応・Docker入門） | 11.5時間 |

### 第6部 チームで働く
| 章 | 内容 | 目安 |
| --- | --- | --- |
| [第24章](./chapters/24-team-development.md) | チーム開発（Issue・ブランチ・PR・コードレビュー・見積もり・報連相） | 8時間 |
| [第25章](./chapters/25-ai-era-engineering.md) | AI時代のエンジニアリング | 6.5時間 |

### 第7部 総仕上げ
| 章 | 内容 | 目安 |
| --- | --- | --- |
| [第26章](./chapters/26-final-project.md) | 最終課題：要件からアプリを作り、発表する | 42〜62時間 |
| [第27章](./chapters/27-job-ready-checklist.md) | 業務レディ・チェックリストと次の学習 | 5.5時間 |

## 進め方

1. 章を順番に読みます。各章の冒頭に「ゴール」「目安時間」「前提」があります。
2. 本文のコマンドやコードは、読むだけでなく自分の手で打って動かします。
3. 演習は、**完了条件**を満たすまでやり切ってから次へ進みます。第11章以降の実装は、TDD（RED → GREEN → REFACTOR）で進めます。
4. 章末の確認クイズを、解答を見る前に自分の言葉で答えます。
5. 演習のコードは、第2章で作る自分のリポジトリ `~/workspace/` に章ごとのフォルダを作って保存し、こまめにコミットします。

完成コードは載せていません。自分で書くことが、この教材の学習の中心だからです。行き詰まったら[第7章](./chapters/07-debugging-and-research.md)の調べ方と質問の仕方を使ってください。

## 守ってほしい2つのルール

**安全のルール**（詳しくは[第10章](./chapters/10-security-basics.md)）

- 自作のサーバは必ず `127.0.0.1` で待ち受け、`lsof` などで確かめます。`0.0.0.0`、公開トンネル、ポート転送は使いません。
- 演習ではダミーデータだけを使います。秘密鍵・パスワード・トークンはコミットしません。
- インターネットへの公開は、[第23章](./chapters/23-deploy-and-operations.md)で示す条件を満たした場合だけにします。

**AIのルール**（詳しくは[第25章](./chapters/25-ai-era-engineering.md)）

- 使ってよい：概念の説明、コードリーディングの補助、設計の比較、自分の理解の確認
- 使わない：演習の実装を丸ごと書かせる、課題の答えを出させる
- 迷ったら、その作業は自分でやるべき領域です。

## 動作環境と、確認の範囲

- 基準：macOS（Windows は WSL）、Ruby 3.3 以降（元教材は Ruby 4.0系・Rack 3系が基準）、SQLite、Rails 8系
- 本文の Ruby・SQL・curl の出力例は、作成時に実際に実行して取ったものです。ただし作成環境の都合で、Ruby の出力の多くは **Ruby 2.6 のもの**です。エラー表示や Hash の表示は、Ruby 3.3 / 3.4 と細部が違います（各章に注記あり）。
- Rails・Docker・GitHub Actions の部分は、ソースコードや公式ドキュメントと照合して書いていますが、実機での実行確認はしていません。バージョンで違う箇所は本文に明記しています。
- 誤りに気づいたら、Issue で知らせてください。

## 出典とライセンス

本教材は次の著作物を参考・改変して作成した、**非公式**の教材です。株式会社Speeeが本教材を作成・支持・保証しているものではありません。

```text
「AI時代における低レイヤーから理解するWebアプリケーション研修ガイダンス」
© 2026 株式会社Speee (Speee, Inc.)
元の教材: https://github.com/speee/training-web-app
ライセンス: CC BY 4.0 https://creativecommons.org/licenses/by/4.0/
変更内容: 2026-10-01、元教材の Phase 1〜6・最終課題・安全上の注意の構成と考え方をもとに、
          未経験者向けの全27章として再構成・大幅に加筆（プログラミング基礎、Web基礎、SQL、
          Rails、認証、セキュリティ実践、API、テスト戦略、運用、チーム開発、AI、最終課題など）。
```

関連する副教材：[安全上の注意 研修教材](https://github.com/inoryo-ai/training-web-app-safety)

本教材も [クリエイティブ・コモンズ 表示 4.0 国際ライセンス（CC BY 4.0）](https://creativecommons.org/licenses/by/4.0/deed.ja) で提供します。
