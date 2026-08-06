---
title: "「判定は正しいのに直らない」— 関門を1箇所に集約しても、全経路が通っていなければ意味がない"
emoji: "🚧"
type: "tech"
topics: ["python", "architecture", "testing", "refactoring"]
published: true
---

自動投稿・自動生成のパイプラインを個人プロジェクトで20本近く動かしていて、同じ形の事故を**3回**やりました。

1. ルールを各スクリプトに複製した → 片方の修正をもう片方が黙って上書きした
2. ルールを1箇所に集約した → **その関門を通らない経路が残っていた**
3. 関門は全経路に通した → **判定ロジック自体にバグがあり、正しいものを誤って弾いていた**

「共通化しましょう」で止まると 2 で死にます。この記事は 2 と 3 の話です。

![3回の事故の形: 複製・バイパス・判定バグ](/images/publish-gate-3-failures.png)
*3回の事故はどれも「ルールをどこに置き、どう通すか」の失敗だった*

## 1回目: ルールの複製で上書き事故

SNSへの自動投稿で、広告であることの開示表記(`【PR】`)を本文に入れる必要がありました。
最初の実装では、投稿スクリプトごとに開示処理を書いていました。

```python
# post_a.py
def ensure_disclosure(text):
    if "#PR" in text:
        return text
    return "#PR " + text

# post_b.py  (別の日に別の人格で書いた)
def ensure_disclosure(text):
    if "【PR】" in text:
        return text
    return "【PR】" + text
```

ある日、テンプレート側を `【PR】` 表記に統一しました。ところが `post_a.py` の判定は
`"#PR" in text` だったため、`【PR】` を開示と認識せず **`#PR` を再注入していました**。

テンプレートを直した数時間後には、実行時に元へ戻っていたわけです。
テンプレートを見ても、投稿結果を見ないと気づけません。

### 対処: SSOT モジュールに集約

判定と整形を1ファイルに寄せました。

```python
# text_rules.py  — ここがSSOT
DISCLOSURE = "【PR】"

def has_disclosure(text: str) -> bool:
    """【PR】でも #PR でも開示として認める(旧実装との後方互換)"""
    return bool(text) and (DISCLOSURE in text or "#PR" in text)

def normalize(text: str) -> tuple[str, dict]:
    """規約に本文を揃える。戻り値は (整えた本文, 何をしたかの記録)"""
    actions = {}
    out = text
    if needs_disclosure(out) and not has_disclosure(out):
        out = DISCLOSURE + out.lstrip()
        actions["disclosure_injected"] = True
    return out, actions
```

ポイントは**戻り値に「何をしたか」を含めること**です。
黙って書き換えると、次に同じ設計ミスを踏んだとき何が起きたか追えません。呼び出し側でログに出します。

## 2回目: 集約したのに、関門を通らない経路が残っていた

ここで満足したのが失敗でした。

`normalize()` を「投稿の直前」に置いたつもりでしたが、実際に投稿する経路は1つではありませんでした。
別のプロジェクトで同じことをやったとき、公開経路を数えたらこうなりました。

| 経路 | 関門を通るか |
|---|---|
| CLI ツール経由 | ✅ |
| 日次ニュース生成 | ❌ 直接 REST を叩いていた |
| 週次特集生成 | ❌ 同上 |
| 記事深化バッチ | ❌ 同上 |
| ...(他7本) | ❌ |

**11経路のうち、関門を通っていたのは1本だけでした。**
CLI にだけ入れて「対応済み」と思い込んでいたわけです。

### 経路の洗い出しは grep で機械的にやる

「たぶんこれで全部」は当てになりません。私の場合は REST の POST 先で洗いました。

```bash
# posts を新規作成している箇所 (posts/<id> は更新なので対象外)
grep -rn 'POST", "posts"\|/wp-json/wp/v2/posts"' --include=*.py .
```

これで最初に7本見つけ、後から4本追加になりました。
**1回の grep で見つかったものが全部だと思わない**のも教訓です。

### 配線されていることをテストで固定する

一番効いたのはこれです。判定ロジックのテストとは**別に**、
「全経路がその関門を呼んでいるか」をソースに対して検査します。

```python
PUBLISH_PATHS = [
    "wp_cli.py", "daily_news.py", "weekly_feature.py",
    "intent_daily.py", "research_article.py", # ...
]

for name in PUBLISH_PATHS:
    src = (HERE / name).read_text(encoding="utf-8")
    assert "assert_publishable" in src, f"{name} が関門を呼んでいない"

    # ルールの複製禁止: 各スクリプトが独自の閾値を持っていないこと
    own = re.search(r"^(?!from _thin_gate).*MIN_PUBLISH_CHARS\s*=\s*\d+", src, re.M)
    assert own is None, f"{name} が閾値を自前で持っている"
```

「ロジックが正しいこと」と「全経路がそれを通ること」は**別々に検証する**。
これを分けて書いてから、この種の事故は止まりました。

意図的に対象外にした経路も、理由をコードに書いておきます。書かないと後から「漏れ」として再調査されます。

```python
# 意図的に対象外:
#   _publish_oneoff.py : 実行済みの単発スクリプト(再実行しない)
#   builder.py         : 本文投入は CLI 経由なので二重
#   (プラグイン)        : アプリ内部で完結し、こちらのコードを通らないので捕捉不可
```

## 3回目: 判定ロジック自体が壊れていた

関門を全経路に通した後、実際に運用したら**正しい記事が「薄い」と判定されて止まりました**。

やりたかったのは「本文が短すぎるものを公開しない」でした。
ただし定型のCTAブロックを本文量に数えると、中身が空でもCTAだけで下限を超えてしまいます。
そこで「定型ブロックのマーカーを見つけたら、そこで本文を切る」実装にしていました。

```python
# ❌ 壊れた実装
def visible_body_len(html: str) -> int:
    cut = len(html)
    for marker in ("<!-- CTA", 'class="cta-block"'):
        i = html.find(marker)
        if i != -1:
            cut = min(cut, i)      # マーカー位置で切り捨てる
    body = html[:cut]
    body = re.sub(r"<[^>]+>", " ", body)
    return len(re.sub(r"\s+", "", body))
```

これには2つのバグがありました。

### バグ①: タグの途中で切れる

`class="cta-block"` は **div の属性の中**にあります。その位置で切ると

```
<div class="cta-b
```

が残ります。`<[^>]+>` は閉じ `>` を要求するので**この断片にマッチせず**、
属性文字列がそのまま本文としてカウントされます。

### バグ②: CTAの後ろの本文を丸ごと捨てる

こちらのほうが深刻でした。新規投稿はCTAが末尾に付くので問題が出ません。
しかし**既存記事はCTAが本文の途中に埋まっていることがあります**。

実測で、本文1,205字ある記事が「薄い」と判定されました。CTAが手前にあり、その後ろが全部捨てられていたからです。
関門は11経路に配線済みだったので、**正当な記事の公開を止めるところでした**。

### 対処: 切り捨てではなく、該当ブロックだけを除去する

```python
# 対のコメントで囲まれたブロックは開き→閉じを丸ごと除去
_PAIRED_BLOCKS = ("CTA", "FAQ-SCHEMA", "STOCK-LINKS")
# 閉じコメントが無く div 1個で表現されるものは対応を数えて除去
_DIV_BLOCK_MARKERS = ('class="cta-block"', "<!-- BACKLINK -->")

def strip_boilerplate(html: str) -> str:
    body = html or ""
    for name in _PAIRED_BLOCKS:
        body = re.sub(rf"<!--\s*{name}[^>]*-->.*?<!--\s*/{name}\s*-->", " ",
                      body, flags=re.S | re.I)
        # 閉じマーカーが失われた壊れたデータ用のフォールバック
        body = re.sub(rf"<!--\s*{name}[^>]*-->.*\Z", " ", body, flags=re.S | re.I)
    for marker in _DIV_BLOCK_MARKERS:
        body = _strip_div_block(body, marker)
    return body
```

div は入れ子になるので、非貪欲マッチでは閉じ位置を誤ります。開閉を数えて対応を取ります。

```python
def _strip_div_block(body: str, marker: str) -> str:
    """marker を含む div を、対応の取れる </div> まで除去する"""
    while True:
        i = body.find(marker)
        if i == -1:
            return body
        start = body.rfind("<div", 0, i + len(marker))
        if start == -1:
            nxt = re.search(r"<div\b", body[i:], re.I)
            if not nxt:
                return body[:i] + body[i + len(marker):]
            start = i + nxt.start()
        depth, pos, end = 0, start, None
        while pos < len(body):
            mo = re.compile(r"<div\b", re.I).search(body, pos)
            mc = re.compile(r"</div\s*>", re.I).search(body, pos)
            if mc is None:
                break
            if mo is not None and mo.start() < mc.start():
                depth += 1
                pos = mo.end()
                continue
            depth -= 1
            pos = mc.end()
            if depth == 0:
                end = pos
                break
        if end is None:
            return body[:start]   # 閉じ切れていない = 以降は定型として捨てる
        body = body[:start] + " " + body[end:]
```

## テストは「誤って止めないか」も書く

バグ②を見つけたのは回帰テストでした。ただし最初に書いたテストは
「薄い本文が止まること」しか見ておらず、通り抜けていました。

```python
# これだけでは足りない
assert blocked("<p>短い。</p>")
assert not blocked("<p>" + "あ" * 1500 + "</p>")

# 🔴 これを足して初めてバグ②が見つかった
CTA = open("cta.html").read()
mid = "<p>" + "あ"*700 + "</p>" + CTA + "<p>" + "い"*700 + "</p>"
assert G.visible_body_len(mid) == 1400, "CTAの後ろの本文も数えること"
assert not blocked(mid), "誤って publish を止めないこと"
```

関門系のロジックは、**通すべきものを通すかのテストのほうが漏れやすい**です。
止める側は作った本人がすぐ思いつくのに対し、通す側は「当然通るだろう」で書かれないからです。

## 閾値は実データの分布を測ってから決める

最後にもう1つ。下限値を決めるとき、勘で決めると誤検知でパイプラインが全停止します。
先に既存データの分布を測る小さなツールを用意しました。

```
=== siteA 直近30本の本文量 (定型CTA除外) ===
  最短 1497字 / 中央値 2337字 / 最長 11563字
  下限 1200字 で判定
       〜500字    0
   1000-1200字    0
   1200-2000字   12
   ...
  ゲートに掛かる記事: 0本 (0%)

=== siteB 直近30本 ===
  最短 147字 / 中央値 2161字
  ゲートに掛かる記事: 4本 (13%)
     147字  xrp-sec-new-rule
     160字  sp500-record-high
```

閾値を上げ下げするときは必ずこれを先に流す、とテストにも書いておきました。

```python
ck("下限は実測最短(1,866字)より低い", G.MIN_PUBLISH_CHARS < 1866)
```

## まとめ

![あるべき配線: 全経路を1つの関門に通し、関門に回帰テストを付ける](/images/publish-gate-correct-wiring.png)
*最終形: 経路を増やすたびに「関門への配線」を最初にやる*

- ルールは1箇所に集約する。ただし**それは出発点**であって完成ではない
- **全経路がその関門を通っているか**を、ロジックのテストとは別に検証する。
  経路の洗い出しは grep で機械的にやり、意図的な対象外は理由をコードに残す
- 関門のロジックは「止めるべきものを止めるか」だけでなく
  **「通すべきものを通すか」**をテストする。後者のほうが漏れやすい
- 閾値は勘で決めず、実データの分布を測ってから置く

3回とも、原因は「共通化が足りない」ではなく「共通化した**つもり**の範囲が実際より狭かった」ことでした。
範囲を人間の記憶ではなくテストに書かせるのが、いまのところ一番効いています。

---

📩 LINE で深掘り配信中

AI / マーケ / 楽天モバの限定情報を 週1〜2回 お届け（無料）

興味のあるテーマだけ選んで受け取れます

[友だち追加する 👉](https://mame-follow.suikou0.workers.dev/follow?cha=zenn)

AIエージェント運用 / MMM / 楽天モバ紹介 の3テーマから選べます
