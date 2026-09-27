---
title: "SNSアプリ内ブラウザから来た11人が、1人も先へ進まなかった — Cookie非共有と x-safari-https"
emoji: "🚪"
type: "tech"
topics: ["webview", "ios", "android", "cloudflareworkers", "個人開発"]
published: true
---

個人の自動化プロジェクトで、SNS に貼るリンクの「受け皿」ページを運用している。Cloudflare Workers で短縮 URL を返し、いったん自前の案内ページを見せてから、外部サービスの申し込みページへ送る構成だ。

```
SNS投稿のリンク → /g/<ch>（受け皿・案内ページ） → /<ch>（計測して302） → 外部サービス（要ログイン）
```

受け皿の表示数と、その先への遷移数は Workers の D1 に記録している。2週間分を経路別に並べたところ、1行だけ異様な数字があった。

| 流入元 | 受け皿の表示 | 先へ進んだ数 |
|---|---|---|
| Threads（iPhoneアプリ内） | 11 | **0** |

![アプリ内ブラウザのまま進むと止まる経路と、外部ブラウザへ出す経路](/images/in-app-browser-cookie-wall-open-external.png)

11人来て0人。他の経路では少ないながら進んでいる人がいるので、ページが壊れているわけではない。

## 症状: 「ボタンが押されていない」のではなく「押しても意味がない」

最初は文言やボタン位置の問題だと考えていた。実際、iPhone の 390x844 で最初の CTA が上から 662px にあって画面外だったので、そこは直した。それでも SNS アプリ経由の遷移はほとんど増えなかった。

実機で辿ってみて原因が分かった。遷移先は5回のリダイレクトの末に、外部サービスのログイン画面（`prompt=login`）に着く。**普段 Safari でそのサービスにログインしている人でも、ここで ID とパスワードを聞かれる。**

## 原因: アプリ内ブラウザは Safari と Cookie を共有しない

Threads・Instagram・Facebook・LINE・X などの SNS アプリは、リンクを自前のアプリ内ブラウザで開く。iOS ではこれは WKWebView で、Cookie ストアはアプリごとに分かれる。Safari のログイン状態は引き継がれない。Android の WebView も同じだ。

そのため、ログインが必要なページへ送る導線はこうなる。

1. SNS でリンクをタップする
2. アプリ内ブラウザで受け皿が開く
3. ボタンを押す
4. **未ログイン扱いになり、ID とパスワードを聞かれる**
5. その場でパスワードを覚えていない人はここで閉じる

ページ側の不具合は1つも無い。ログは「受け皿は表示された、その先は0」としか言わないので、コピーやデザインの問題に見えてしまう。

## 対処1: アプリ内ブラウザを検出する

まずアプリ内ブラウザかどうかを UA で判定する。誤検出で通常のブラウザにまで案内を出すと邪魔なので、条件は狭く取った。

```js
var ua = navigator.userAgent || '';
var has = function (s) { return ua.indexOf(s) >= 0; };
var iOS = has('iPhone') || has('iPad') || has('iPod');
// iOS: Safari 本体は "Safari/" と "Version/" を両方持つ。WKWebView は持たない
var iosInApp = iOS && !(has('Safari/') && has('Version/'));
// Android: WebView は "; wv)" が入る
var andInApp = has('Android') && has('; wv)');
// 既知のアプリ内ブラウザ名
var names = ['FBAN', 'FBAV', 'Instagram', 'Line/', 'Twitter', 'Threads', 'MicroMessenger'];
var named = names.some(has);
```

iOS の Chrome（CriOS）も `Version/` を持たないので `iosInApp` に入る。Chrome も Safari と Cookie を共有しないので、案内が出ても害はない。

## 対処2: 「…からブラウザで開く」を読ませず、1タップで外に出す

最初の版では、検出したら「右上の … から『ブラウザで開く』を選んでください」と文章で案内していた。それでも 11→0 だった。メニューの位置も文言もアプリごとに違うので、手順を読ませる案内は重い。

そこで、押すと外部ブラウザに切り替わるボタンにした。

```js
var a = document.getElementById('openext');
var target = location.host + '/' + ch + '?sf=1';
if (iOS) {
  a.href = 'x-safari-https://' + target;               // iOS 17+ で Safari が開く
} else if (has('Android')) {
  a.href = 'intent://' + target + '#Intent;scheme=https;end';  // 既定ブラウザで開く
}
```

- **iOS**: `x-safari-https://`（http なら `x-safari-http://`）スキームは iOS 17 以降で使え、Safari で開き直す。
- **Android**: `intent://…#Intent;scheme=https;end` で、WebView ではなく既定のブラウザに渡せる。

どちらもアプリ側がスキームを握りつぶす可能性はあるので、ボタンの下に「切り替わらないときは … から『ブラウザで開く』」という文言は残した。それでも無理ならログインを求められる通常のボタンで進める。案内がどれも効かないときに先へ進めなくなるのは避けたい。

## 対処3: 外に出た人を別に数える

外部ブラウザで開き直すと `Referer` が空になる。何もしないと「どこから来たか分からないクリック」として記録され、ボタンが効いたかどうかを判定できない。

そこでボタンの URL に `?sf=1` を付け、Worker 側で印を付けて記録するようにした。

```ts
if (recordable && KNOWN.has(ch)) {
  // 外部ブラウザ切替ボタン(?sf=1)経由は referer が空になるので印を付ける
  ctx.waitUntil(record(env.DB!, "go_" + ch, request, DEDUP_WINDOW_SEC,
    url.searchParams.has("sf") ? "ext-browser" : undefined));
}
```

これで「アプリ内のまま進んだ数」と「外に出てから進んだ数」を分けて見られる。本稿の時点ではまだ効果は測れていない。ボタンを出したのは今日なので、数字が出たら追記する。

## ついでに踏んだ罠2つ

### テンプレートリテラルの中の正規表現で `\/` が潰れた

受け皿の HTML は、Worker 内の TypeScript で `` `<!doctype html>...` `` というテンプレートリテラルから組み立てている。最初は UA 判定を正規表現で書いていた。

```ts
return `...
<script>
  var iosInApp = /iPhone/.test(ua) && !/Version\/.*Safari\//.test(ua);
</script>`;
```

テンプレートリテラルは中のバックスラッシュをエスケープとして解釈する。`\/` は `/` になって出力され、ブラウザに届く JS は `/Version/.*Safari//` になる。正規表現が途中で終わって**構文エラーになり、スクリプト全体が動かなくなった。** アプリ内ブラウザの案内が出ないだけで、ページ自体は表示されるので気づきにくい。

`\\/` と書けば回避できるが、次に触る人（自分）がまた間違える。スラッシュを含む判定は `indexOf` で書くことにした（上のコードがそうなっている理由）。

### HTML コメントに書いた運用メモが、閲覧者に配信されていた

「CTA が 662px で画面外だった」「ここで落ちていた」といった経緯を、テンプレート内の `<!-- -->` に残していた。HTML コメントは画面には出ないが、**レスポンスには含まれる**。ソースを表示すれば誰でも読める。

経緯のメモは TypeScript 側の `//` コメントに移した。こちらはビルド時に消えるので配信されない。

## 教訓

- **ログイン必須のページへ SNS から送るなら、アプリ内ブラウザでは「ログイン済みの人」がいない前提で設計する。** 計測上は「受け皿は見られたが先へ進まない」としか見えず、コピーの問題と取り違えやすい。
- 「… からブラウザで開く」を読ませる案内より、`x-safari-https://` / `intent://` で1タップで外に出すほうが手数が少ない。効かない環境のために文言の案内は残しておく。
- 外に出すと `Referer` が消える。効果を測るなら、切替用の URL に印を付けておく。
- サーバー側で文字列から HTML/JS を組み立てるときは、テンプレートリテラルのエスケープと HTML コメントの配信という、「画面では分からない」2つの経路に注意する。
