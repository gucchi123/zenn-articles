---
title: "スマホ4社の月額を「ギガ数」で比べるツールをCloudflare Pagesに無料で置いた（povoはトッピングの組み合わせを全探索）"
emoji: "📱"
type: "tech"
topics: ["cloudflare", "cloudflarepages", "javascript", "個人開発", "seo"]
published: true
---

スマホの料金を比べる記事は多いのですが、ほとんどが「3GBなら」「20GBなら」と数点だけを並べています。実際には、毎月何GB使うかで一番安い会社が何度も入れ替わります。

そこで、ギガ数をスライダーで動かすと楽天モバイル・ahamo・povo・LINEMO の月額が並ぶページを作り、Cloudflare Pages に置きました。費用は0円です。

https://rakumoba-lab.pages.dev/

この記事では、作り方と、作ってみて気づいたことを書きます。

:::message
筆者は楽天グループの従業員です。このツールのサイトには、楽天モバイルの従業員紹介プログラムの案内ページ（#PR と明記）があります。比較の結果は、楽天モバイルが高くなる条件もそのまま表示しています。
:::

## 構成：フレームワークなしの静的サイト

- HTML 4枚（比較・乗り換え前チェック・紹介の案内・運営者）
- CSS 1枚、JavaScript 1枚（計算部分は100行ほど）
- ビルドなし。`wrangler pages deploy` でフォルダをそのまま上げる

```bash
npx wrangler pages project create rakumoba-lab --production-branch main
npx wrangler pages deploy ./site --project-name rakumoba-lab --branch main
```

スマホで見られることが前提なので、`prefers-color-scheme` のダークモードと、幅 720px までの1カラムだけを用意しました。

## 料金は「確認日つき」で持つ

料金は変わります。2026年は、ahamo が12月から3段階になり、LINEMO も9月にプランが変わりました。

そこで、金額は1か所（社内用の事実の元帳）にまとめ、各社の公式ページで確かめた日付と一緒に管理しています。ページにも「確認日」を出します。

ページを書き換える前には、事実と食い違う言い切り（たとえば「〇〇は終了」）や「最安」という言葉を機械的に弾く関門を通しています。比較記事は「一番安い」と言いたくなりますが、条件（通話の有無など）で入れ替わるので、表示は「この条件で一番低い」にしました。

## povo は「組み合わせ」を全探索する

povo 2.0 は基本料0円で、30日間のデータ（3GB 990円・5GB 1,380円・30GB 2,780円）を足す方式です。

最初は「そのギガ数に届く一番小さいトッピング1つ」で計算していましたが、10GB を入れたところで間違いに気づきました。

- 30GB を1つ：2,780円
- 5GB を2つ：2,760円 ← こちらが20円安い

ギガ数は最大120なので、組み合わせを総当たりしても一瞬です。

```js
// 30日トッピング（3GB 990・5GB 1,380・30GB 2,780）を組み合わせて g 以上にする一番安い組み合わせ
function povo(g) {
  var best = null;
  for (var c = 0; c <= Math.ceil(g / 30); c++) {          // 30GB の個数
    for (var b = 0; b <= Math.ceil(g / 5); b++) {          // 5GB の個数
      var rest = g - c * 30 - b * 5;
      var a = rest > 0 ? Math.ceil(rest / 3) : 0;          // 残りを 3GB で埋める
      var y = a * 990 + b * 1380 + c * 2780;
      if (!best || y < best.y) best = { y: y, a: a, b: b, c: c };
    }
  }
  return best;
}
```

結果の一部です。

| GB | 組み合わせ | 金額 |
|---:|---|---:|
| 6 | 3GB×2 | 1,980円 |
| 8 | 5GB＋3GB | 2,370円 |
| 10 | 5GB×2 | 2,760円 |
| 35 | 30GB＋5GB | 4,160円 |

「一番小さいトッピング1つ」で書いていた版では、6〜10GB の povo を最大800円ほど高く見せていました。比較ツールで1社だけ不利に見せるのは、意図していなくても信用を落とします。

## 1GB刻みで並べると、安い会社が5回入れ替わる

電話をほとんどかけない前提で、1GB から 40GB まで並べると、一番低い会社はこう変わりました（2026年10月の料金）。

| 毎月の量 | 一番低い |
|---|---|
| 1〜3GB | LINEMO・povo（同額） |
| 4〜6GB | povo |
| 7〜10GB | LINEMO |
| 11〜20GB | 楽天モバイル |
| 21〜30GB | povo |
| 31GB〜 | 楽天モバイル |

![毎月の量で一番低い会社が5回入れ替わる（通話ほぼ無し・2026年10月）](/images/phone-price-tool-switch-bands.png)

さらに通話条件（5分以内無料・アプリ経由で無料・30秒22円）が入ると、順番がまた変わります。数点だけの比較表では、この入れ替わりが見えません。

## Search Console への登録も API で済ませる

`pages.dev` のサブドメインは DNS を触れないので、所有権の確認は HTML ファイル方式にしました。Site Verification API でトークンをもらい、そのファイルを置いてデプロイし、確認 → プロパティ追加 → サイトマップ送信までを1本のスクリプトで流しています。

```python
sv = build("siteVerification", "v1", credentials=creds)
tok = sv.webResource().getToken(body={"verificationMethod": "FILE",
      "site": {"type": "SITE", "identifier": "https://rakumoba-lab.pages.dev/"}}).execute()
# → tok["token"] = "google××××.html" を site/ に置いて deploy
sv.webResource().insert(verificationMethod="FILE",
      body={"site": {"type": "SITE", "identifier": "https://rakumoba-lab.pages.dev/"}}).execute()
sc.sites().add(siteUrl="https://rakumoba-lab.pages.dev/").execute()
sc.sitemaps().submit(siteUrl="https://rakumoba-lab.pages.dev/",
                     feedpath="https://rakumoba-lab.pages.dev/sitemap.xml").execute()
```

## 効果は毎週同じ形で測る

新しいドメインは、検索に出るまで数週間かかります。最初の4週は「索引に載ったか → 検索で表示されたか → クリック」の順に、週1回同じ表を作って見ています。URL 検査 API で各ページの登録状態も取れるので、「表示0」が「まだ索引に無い」のか「索引にはあるが順位が低い」のかを分けて見られます。

## まとめ

- 静的な比較ツールなら、Cloudflare Pages で費用0円・ビルドなしで置ける
- 料金は確認日つきで1か所に持ち、言い切りを機械的に弾く
- 組み合わせで買える料金は全探索する。1つで計算すると、特定の会社を不利に見せてしまう
- 1GB刻みで並べると、数点の比較表では見えない入れ替わりが見える

---

📩 LINE で深掘り配信中

AI / マーケ / 楽天モバの限定情報を 週1〜2回 お届け（無料）

興味のあるテーマだけ選んで受け取れます

[友だち追加する 👉](https://mame-follow.suikou0.workers.dev/follow?cha=zenn)

AIエージェント運用 / MMM / 楽天モバ紹介 の3テーマから選べます
