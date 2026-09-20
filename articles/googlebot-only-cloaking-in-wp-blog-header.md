---
title: "「順位が低い」ではなかった — Googlebot にだけ別サイトを返す改ざんの見つけ方"
emoji: "🔍"
type: "tech"
topics: ["wordpress", "seo", "security", "運用", "個人開発"]
published: true
---

個人でメディアサイトを何本か自動運用している。そのうちの1本は記事数が 43,488 本あって、毎日の自動投稿も止まっていない。それなのに検索からの流入がほぼゼロだった。

ずっと「テーマの選び方が悪い」「薄い記事ばかりでインデックスされていない」と解釈していた。実際には、**Google にだけまったく別のサイトが返っていた**。

![人と Googlebot に別のページが返っていた](/images/googlebot-cloaking-two-faces.png)

## 症状: 説明のつかない3つが並んでいた

困っていたことは3つあった。それぞれ別の問題として、別々に対処しようとしていた。

1. 4万本の記事があるのに、検索経由の流入がほぼゼロ
2. Search Console の所有権確認が、何をやっても通らない（`siteUnverifiedUser` のまま）
3. アナリティクス経由での所有権確認も失敗する

1 は SEO の問題、2 と 3 は設定ミスだと思っていた。meta タグを置き直したり、計測タグの所有者を確認したりした。どれも直らない。

そして**ブラウザで開くと、サイトは完全に正常だった**。日本語の記事がちゃんと出る。だから「壊れている」という発想にならなかった。

## 気づいた瞬間: UA を変えて取ってみた

流入の実数を測る作業のついでに、Googlebot の UA で取得してみた。深い意図はなく、「bot から見た HTML はどうなっているか」を確認したかっただけだった。

```bash
UA_BOT='Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)'
UA_HUMAN='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36'

for ua in "$UA_BOT" "$UA_HUMAN"; do
  curl -s -A "$ua" -H 'Accept-Language: ja,en;q=0.9' "https://example.com/" > /tmp/body.html
  printf '%8s bytes  %s\n' "$(wc -c < /tmp/body.html)" "$(grep -o -m1 '<html[^>]*lang="[^"]*"' /tmp/body.html)"
done
```

出力はこうなった。

```
  470131 bytes  <html lang="es-ES"
  125004 bytes  <html lang="ja"
```

同じ URL である。Googlebot にはスペイン語の通販ページが、人間には本来の日本語記事が返っていた。しかも Googlebot 側は**取得のたびに違う商品**が出る。静的な差し替えではなく、動的に生成されていた。

わかったことを並べるとこうなる。

| 取得者 | 返るもの |
| --- | --- |
| Googlebot / Googlebot-Image | `lang="es-ES"` のスペイン語通販ページ・約470KB |
| bingbot | 正常な日本語サイト（125KB） |
| 実ユーザー | 正常な日本語サイト（125KB） |

**bingbot には正常なページを返す。Google だけを狙っている。** 「検索エンジン全般に対する挙動」ではなく「Google に対する挙動」だったので、他の検索エンジンからの流入だけを見ていたら永遠に気づけなかった。

### 全パスが乗っ取られていた

存在しないはずの URL を叩くと、差がもっとはっきり出た。

```bash
path="/this-page-does-not-exist-$RANDOM/"
curl -s -o /dev/null -w 'bot   %{http_code} %{size_download}\n' -A "$UA_BOT"   "https://example.com$path"
curl -s -o /dev/null -w 'human %{http_code} %{size_download}\n' -A "$UA_HUMAN" "https://example.com$path"
```

```
bot   200 463193
human 404 0
```

人間には正しく 404 を返し、Googlebot にはどんなパスでも 200 を返す。つまり**サイトのすべての URL が、Google から見ると無限のスパムページ**になっていた。

`robots.txt` にも差分があった。Googlebot が取る版にだけ、見覚えのない `Sitemap:` 行が1つ入っている。そのサイトマップを辿ると、日次サイトマップが21本・1本あたり約805 URL で、合計 **17,000 本ほどのスパム URL が毎日 Google に送られ続けていた**。当日分まで生成されていたので、現在進行形だった。

ここで冒頭の3つの症状が、全部ひとつの原因で説明できた。

- 流入ゼロ: Google がインデックスしているのは日本語記事ではない
- 所有権確認が通らない: `<meta google-site-verification>` を置いても、**Google が取りに来る HTML はスパム側**なのでトークンが載っていない
- アナリティクス経由の確認が失敗するのも、まったく同じ理由

## どの層で差し替えられているか、はヘッダでわかる

次に知りたいのは「WordPress の中なのか、外なのか」だった。プラグインを止めれば直るのか、それともサーバー側なのか。これはレスポンスヘッダを比べるだけで切り分けられた。

```bash
for ua in "$UA_BOT" "$UA_HUMAN"; do
  echo "--- $ua"
  curl -sI -A "$ua" -H 'Accept-Language: ja' "https://example.com/" \
    | grep -iE '^(link|vary|content-type):'
done
```

| ヘッダ | Googlebot | 実ユーザー |
| --- | --- | --- |
| `Link: <…/wp-json/>; rel="https://api.w.org/"` | **無し** | あり |
| `Vary` | `Accept-Encoding` のみ | `accept,content-type` も |
| charset | `utf-8` | `UTF-8` |

決め手は1行目だ。**WordPress が必ず出す REST API の `Link` ヘッダが、Googlebot 向けの応答にだけ無い。** ということは、WordPress が応答を組み立てるより前に横取りされて `exit` されている。

この時点で「プラグインを全部止めても直らない」と予測が立った。charset の大文字小文字が違うのも、同じ結論を補強する（別のコードが書いた `header()` だということ）。

:::message
応答の差が「どの層で生まれたか」は、本文よりヘッダのほうが雄弁なことが多い。アプリケーションが必ず付けるヘッダが欠けていたら、その応答はアプリケーションを通っていない。
:::

## 注入点: WordPress コアのファイルそのもの

サーバー側でファイルを確認したら、犯人はすぐ見つかった。

```
wp-blog-header.php   11,755 バイト / 170 行   ← 正規版は約 350 バイト / 20 行
  base64_decode(  39 行目
  eval(           73 行目
  7,136 文字の 1 行（base64 の塊）
```

`wp-blog-header.php` は WordPress の読み込み順で `index.php` の直後に来る。テーマもプラグインも、`wp-load.php` すら読まれる前だ。ここで UA を見て `echo` して `exit` すれば、WordPress は一切動かない。観測していた「REST の `Link` ヘッダが無い」と完全に一致する。

質が悪いのは、**正規のコードが消されずに残されていたこと**だった。

```php
<?php
/**
 * Loads the WordPress environment and template.
 *
 * @package WordPress
 */

if ( ! isset( $wp_did_header ) ) {
    $wp_did_header = true;
    require_once __DIR__ . '/wp-load.php';
    wp();
    require_once ABSPATH . WPINC . '/template-loader.php';
}
```

この 20 行は丸ごと生きている。だから通常の閲覧も、管理画面も、自動投稿も、何ひとつ壊れない。**壊れないので、1か月気づかない。**

タイムスタンプを並べると全体像が出た。

```bash
find /path/to/public_html -name '*.php' -newermt '2026-08-19' -printf '%T+ %10s %p\n' | sort | head -50
```

- 侵入は約1か月前の21:15。同じ分に `wp-config.php`・`wp-settings.php`・`wp-includes/` も書き換えられていた
- `wp-config.php` のパーミッションが **755**（通常 644）。実行可能になっていた
- WordPress に存在しないディレクトリが同日に作られていた（中身は空）
- スパムサイトマップの生成開始は**侵入の12日後**。すぐには動かず、様子を見てから稼働している

### 1コマンドで出たはずだった

あとから思えば、コアファイルの改変は WP-CLI の標準機能で検出できる。

```bash
wp core verify-checksums
# Warning: File doesn't verify against checksum: wp-blog-header.php
```

`wp-blog-header.php` は WordPress 本体のファイルなので、公式配布物のチェックサムと突き合わせれば一発でわかる。プラグインやテーマばかり疑って、**「コアのファイルが書き換わっている」という可能性を検査していなかった**。

## 自分で入れたアクセス制限が、自分の監視を殺していた

もうひとつ、発見が遅れた直接の理由がある。

このサイトには以前、海外からの機械的アクセスを減らす目的で「`Accept-Language` に日本語が含まれないリクエストは 403」というゲートを自分で入れていた。`curl` も `requests` も既定では `Accept-Language` を送らないので、**外形監視のスクリプトからはサイト全体が 403 に見える**。

なので「このサイトは自動チェックできない」ということにして、チェック自体を書いていなかった。

そして、そのゲートは **Googlebot を免除していた**。つまり、

- 自分の監視 → 403 で何も見えない
- Googlebot → 200 でスパムを受け取る

という状態を、自分で作っていたことになる。ヘッダを1つ足すだけで通る話だった。

```python
headers = {"User-Agent": UA, "Accept-Language": "ja,en;q=0.9"}  # これで 200
```

:::message alert
アクセス制限を入れるときは、**自分の監視をその例外に入れたか**を同時に確認する。監視が弾かれると、「監視できない」が「監視しない」に静かに変わる。
:::

## 対処と、日次で回す1行

駆除の手順は普通の内容になる（証拠を残したバックアップ → コアファイルを正規版に戻す → パーミッションを戻す → 同時刻に触られたファイルを全部確認 → **管理者・FTP・DB・ホスティングのパスワードを全部変更**）。侵入経路が特定できていない以上、最後のパスワード変更を省くと再感染する。

再発を検知する側は、これだけで足りる。日次タスクに入れた。

```bash
#!/usr/bin/env bash
UA_BOT='Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)'

for url in "${SITES[@]}"; do
  lang=$(curl -s --max-time 20 -A "$UA_BOT" -H 'Accept-Language: ja,en;q=0.9' "$url" \
         | grep -o -m1 'lang="[^"]*"')
  if [ "$lang" != 'lang="ja"' ]; then
    echo "CLOAKING SUSPECTED: $url -> ${lang:-(取得失敗)}"
  fi
done
```

サイズの比較でも十分に検知できる。125KB と 470KB のように桁が違えば、中身を読むまでもない。

## 教訓

**1. クライアントによって応答が変わるものは、クライアントを変えて比べるまで何も判定できない。** 自分のブラウザで正常に見えることは、何の証拠にもならなかった。UA・`Accept-Language`・Cookie の有無で分岐するコードはそこら中にあるので、これは改ざんに限った話ではない。

**2. 症状が3つ並んだら、まず1つの原因で説明できないか疑う。** 「流入ゼロ」「所有権確認が通らない」「アナリティクス認証が失敗」を、SEO の問題と設定の問題に分けて別々に潰そうとしていた。3つとも同じ1行で説明がついた。**別々の対処を始める前に、共通の根を探す**ほうが早い。

**3. 層の切り分けは本文ではなくヘッダでやる。** 「アプリケーションが必ず付けるヘッダが欠けている」は、アプリケーションより手前で応答が作られている決定的な証拠になる。これがわかっていたので、プラグインを1つずつ止める作業をせずに済んだ。

**4. 「壊れない改ざん」が一番見つからない。** 正規のコードを残して分岐を足すタイプは、動作確認では絶対に見つからない。だから**動作ではなくファイルの同一性を検査する**手段（コアのチェックサム検証、更新日時の一覧）を、定期的に回す側に置いておく。

**5. 自分で入れた制限は、自分の観測も止める。** 監視が 403 で弾かれていることに気づいたとき、「監視を直す」ではなく「このサイトは監視できない」と結論づけたのが、1か月の空白の正体だった。
