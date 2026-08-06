---
title: "Pillowで多言語の画像を作ると、タイ語・ミャンマー語だけ静かに壊れる"
emoji: "🔤"
type: "tech"
topics: ["python", "pillow", "i18n", "font", "playwright"]
published: true
---

個人の自動化プロジェクトで、同じレイアウトの図解画像を複数言語ぶん生成するスクリプトをPillowで書いていました。インドネシア語・ポルトガル語・モンゴル語・ベトナム語まではきれいに出ていたのに、**ミャンマー語を追加した瞬間に字が重なって読めなくなりました**。

しかも厄介なことに、事前に入れていた「フォントに全文字のグリフがあるか」の検査は**全部パスしていました**。原因はPillowのビルド構成に起因するもので、日本語・英語だけ扱っているうちは絶対に踏まないタイプの罠だったので共有します。

![検査は全部パスするのに、タイ語・ミャンマー語だけ描画が壊れる。真因はフォントではなくPillowのビルド構成](/images/pillow-raqm-why-checks-pass.png)

## 症状

- ミャンマー語: 積み字（子音の下に付く記号）が本体と重なり、明確に崩れる
- タイ語: 声調記号の位置がずれる。**パッと見は読めてしまうので見逃しやすい**
- ベトナム語・ラテン文字・キリル文字: 何の問題もない
- エラーも例外も出ない。生成は「成功」する

「フォントが足りないんだろう」と当たりをつけて、こういう検査を書いていました。

```python
from PIL import ImageFont

f = ImageFont.truetype(r"C:\Windows\Fonts\mmrtext.ttf", 48)  # Myanmar Text
text = "အနည်းဆုံး စာချုပ်ကာလ မရှိပါ"

missing = [c for c in text if c.strip() and f.getmask(c).getbbox() is None]
print("missing glyph coverage:", missing)
```

```
missing glyph coverage: []
```

**カバレッジは100%**。フォントには全部のグリフが入っている。それでも画像は壊れている。ここでしばらく詰まりました。

## 原因: complex text layout (libraqm) が無効だった

Pillowには文字の**レイアウトエンジン**が2種類あります。

| エンジン | 実体 | shaping |
| --- | --- | --- |
| `Layout.BASIC` | Pillow内蔵 | **しない**（コードポイント順にアドバンス幅を足すだけ） |
| `Layout.RAQM` | libraqm (= HarfBuzz + FriBidi) | する |

libraqmは**ビルド時にリンクされていないと使えない**オプション依存です。手元の環境を確認するとこうでした。

```python
from PIL import features
print(features.check("raqm"))     # False
print(features.version("raqm"))   # None
```

`False`。つまり全部BASICレイアウトで描かれていたわけです。

![BASICは順番どおりに並べるだけ、RAQMは文字体系のルールに従って整形する](/images/pillow-raqm-basic-vs-raqm.png)

### shapingとは何をする処理か

BASICレイアウトは「1コードポイント = 1グリフを、文字列の順番どおりに、アドバンス幅ぶん右にずらして並べる」だけの処理です。ラテン文字ならこれで正しく描けます。

しかしタイ語・ミャンマー語・クメール語・アラビア語などは、正しく表示するために**shaping（字形の整形）**が要ります。具体的には、

- **並べ替え**: 論理順（メモリ上の並び）と表示順が一致しない。ミャンマー語の `ြ` (U+103C) のような記号は、後続の子音の**左側**に回り込む
- **合字・字形選択**: 前後の文脈で使うグリフそのものが変わる
- **マーク位置決め (mark positioning)**: 声調記号や母音記号を、ベース文字のどの位置に打つか決める

BASICレイアウトはこのどれもしません。結果、**グリフは全部揃っているのに置き場所が全部間違っている**画像ができあがります。カバレッジ検査が通ってしまったのはこのためでした。検査していたのは「グリフがあるか」で、壊れていたのは「どこに置くか」です。

### 最悪なのは、明示指定しても黙って落ちること

「じゃあRAQMを明示的に指定すればいい」と思って書いたのがこれです。

```python
f = ImageFont.truetype(FONT, 48, layout_engine=ImageFont.Layout.RAQM)
print(f.layout_engine)   # 0  (= Layout.BASIC)
```

**例外は飛びません。** `Layout.RAQM` (=1) を要求したのに、返ってくるフォントの `layout_engine` は `0`、つまり `BASIC` です。Pillowは `UserWarning` を1回出すだけで、あとは何事もなかったかのようにBASICで描き続けます。

```
UserWarning: Raqm layout was requested, but Raqm is not available. Falling back to basic layout.
```

バッチ処理のログの奥に埋もれれば、まず気づきません。「RAQMを指定したから大丈夫」という思い込みだけが残ります。

### なぜベトナム語は無事だったのか

同じ「ラテン文字じゃない言語」なのにベトナム語が平気だった理由は、**正規化形の違い**です。

```python
import unicodedata
vi = unicodedata.normalize("NFC", "Không giới hạn dữ liệu")
print(len(vi))                                      # 22
print(len(unicodedata.normalize("NFD", vi)))        # 30
```

ベトナム語の声調付き文字は**NFCで合成済みの単一コードポイント**として存在します（`ữ` = U+1EEF が1文字）。1コードポイントが1グリフに対応するので、shapingが要りません。だからBASICレイアウトでも正しく出る。

逆に言うと、**入力がNFDで来ると同じ環境でベトナム語も崩れます**。「ベトナム語は大丈夫だった」は環境の性質ではなく、たまたま入力がNFCだっただけです。

## shapingが効いていないことを機械的に検出する

目視は当てになりません。読めない言語なら特にそうです。そこで、CIに置けるアサーションとしてこれを使っています。

```python
from PIL import Image, ImageDraw, ImageFont

f = ImageFont.truetype(FONT, 48)
d = ImageDraw.Draw(Image.new("RGB", (10, 10)))
text = "အနည်းဆုံး စာချုပ်ကာလ မရှိပါ"

whole    = d.textlength(text, font=f)
per_char = sum(d.textlength(c, font=f) for c in text)
print(whole, per_char, abs(whole - per_char) < 0.01)
```

```
571.0 571.0 True
```

**文字列全体の幅と、1文字ずつ測った幅の単純合計が、小数点以下まで完全に一致する。** これは「文字列を一体として見る処理が何一つ行われていない」ことの証拠です。合字も、カーニングも、マーク位置決めも走っていない。

実務上は、これを直接判定するより最初にこう倒しておくのが確実です。

```python
from PIL import features

if not features.check("raqm"):
    raise RuntimeError(
        "libraqm 無効。complex script (th/my/km/ar/hi...) の描画は Pillow では不可"
    )
```

`ImageFont.Layout.RAQM` を渡した「つもり」を信じず、`features.check("raqm")` という**環境そのものの事実**を見るのがポイントです。

## 対処: ブラウザに描かせる

libraqm有効のPillowをビルドし直す道もありますが、Windows + 複数マシンで同じスクリプトを回している都合上、**環境依存を持ち込まない**方を選びました。

HTMLを組んで、ヘッドレスChrome (Playwright) でスクリーンショットを撮ります。ブラウザは当然HarfBuzzでshapingするので、全文字体系で正しく出ます。

```javascript
// render_figure_html.mjs
import { chromium } from 'playwright';
import { readFileSync } from 'fs';

const spec = JSON.parse(readFileSync(process.argv[3], 'utf-8'));

// 言語別フォントスタック。OS同梱フォントを名前で指定してChromeに解決させる
const FONTS = {
  th: '"Leelawadee UI", "Noto Sans Thai", sans-serif',
  my: '"Myanmar Text", "Noto Sans Myanmar", sans-serif',
  km: '"Khmer UI", sans-serif',
};
const fontStack = (FONTS[spec.lang] || '"Segoe UI", Arial, sans-serif')
  + ', "Yu Gothic UI", sans-serif';   // 和文フォールバック

const html = `<!doctype html><meta charset="utf-8"><style>
  body { width: 1240px; height: 800px; font-family: ${fontStack}; }
  .head { font-size: 30px; font-weight: 700; line-height: 1.25; }
</style>
<div class="head">${spec.title}</div>`;

const browser = await chromium.launch({ headless: true, channel: 'chrome' });
const page = await browser.newPage({
  viewport: { width: 1240, height: 800 },
  deviceScaleFactor: 2,            // 2倍で撮ってRetina相当の解像度を稼ぐ
});
await page.setContent(html, { waitUntil: 'load' });
await page.waitForTimeout(600);    // Webフォント/システムフォント適用待ち
await page.screenshot({ path: process.argv[5] });
await browser.close();
```

Python側からは薄く叩くだけです。

```python
subprocess.run(["node", str(RENDERER), "--spec", str(spec_path), "--out", str(out)],
               capture_output=True, text=True, encoding="utf-8")
```

`line-height` や `font-family` のフォールバックをCSSでそのまま書けるので、Pillowで自前実装していた折り返し処理（`textlength` を舐めながら手で改行位置を探すあれ）が丸ごと消えたのは、副次的な収穫でした。complex scriptでは「どこで改行してよいか」自体が自明ではないので、これを自前で持たない判断は結果的に正解でした。

なお既存のラテン文字・ベトナム語の資産まで作り直す必要はないので、Pillow版のスクリプトはそのまま残し、**新規言語は必ずHTML版を通す**という運用にしています。

## 教訓

- **「フォントにグリフがあるか」と「正しく組めるか」は別問題**。カバレッジ検査は前者しか見ない。complex scriptでは後者が壊れる
- Pillowで非ラテン文字を扱うなら、まず `PIL.features.check("raqm")` を確認する。`layout_engine=Layout.RAQM` の指定は**要求であって保証ではない**。使えなければ警告1行でBASICに落ちる
- ベトナム語が通ってタイ語が落ちたように、**「非ラテン文字」はひとくくりにできない**。判断軸はNFCで合成済みか（shaping不要）／結合記号の整形が要るか（shaping必須）
- 読めない言語の出力を目視で検品するのは不可能に近い。`textlength(全体) == Σtextlength(1文字)` のような**機械的なアサーション**か、そもそも環境チェックで早期に落とす仕組みを持つ
- ライブラリのオプション依存が無効なとき、**例外ではなく警告で劣化動作に落ちる**のは珍しくない。「動いている」は「意図どおり動いている」の証明にならない

## 関連

「エラーは出ないが実は壊れている」系の静かな失敗は、自動化を続けているとかたちを変えて何度も出てきます。同種の罠をこちらにも書きました: [PowerShellで `Test-Path $a -and $b` が謎のエラーになる罠](https://zenn.dev/mameresearcher/articles/powershell-test-path-and-operator-trap)

---

📩 LINE で深掘り配信中

AI / マーケ / 楽天モバの限定情報を 週1〜2回 お届け（無料）

興味のあるテーマだけ選んで受け取れます

[友だち追加する 👉](https://mame-follow.suikou0.workers.dev/follow?cha=zenn)

AIエージェント運用 / MMM / 楽天モバ紹介 の3テーマから選べます
