---
title: "ast.parse は構文検査ではない — 「全ファイルOK」の裏で import できないコードが混ざっていた"
emoji: "🩻"
type: "tech"
topics: ["python", "ast", "静的解析", "自動化"]
published: true
---

個人でいくつもの自動化スクリプトを回している。同期フォルダ上で運用しているせいで、たまに同じファイルの競合コピーができる。片方だけに入っている改良もあるので、機械的にどちらかを捨てるわけにいかず、両方を読んで統合する作業が発生する。

その統合をやった直後に、自前の見張りスクリプトで全 Python ファイルの構文検査を回して、こう報告した。

```
構文検査: 486ファイル OK
```

嘘だった。

統合したファイルのうち1本は **import すらできない状態**で、そこに繋いでいた日次の処理が前日から止まっていた。もう1本は構文としては完全に正しいのに、**承認済みだった設定値が静かに元へ戻っていた**。

検査そのものが甘かった。原因は2つあって、どちらも「知らないと一生気づかない」型だったので書き残しておく。

![ast.parse は構文木が作れるかしか見ていない。compile() なら弾ける](/images/ast-parse-not-a-syntax-check.png)

## 症状

統合したファイルの1本は、こういう形になっていた。

```python
r = requests.get(f"{base}/wp-json/wp/v2/posts",
                 params={"slug": slug, "context": "edit", "_fields": fields},
                 params={"slug": slug, "context": "edit",
                         "_fields": "id,slug,link,title,content"},
                 headers=h, timeout=45)
```

`params=` が2回ある。片方は新しい版、片方は古い版で、「両方の行を残す」形でマージした結果こうなった。

これは Python では `SyntaxError` だ。当然モジュールは import できない。日次のジョブは前日から静かにこけていた。

なのに私の検査は「OK」と言った。検査はこう書いてあった。

```python
import ast

with open(path, encoding="utf-8") as f:
    ast.parse(f.read(), filename=path)   # ← ここ
```

## 原因1: `ast.parse` は「構文木が作れるか」しか見ていない

手元で確かめた結果がこれ。Python 3.11.15 と 3.12.10 で同じだった。

| ソース | `ast.parse` | `compile(..., "exec")` |
|---|---|---|
| `f(a=1, a=2)` | **通る** 🔴 | `keyword argument repeated: a` |
| `def g(x, x): pass` | **通る** 🔴 | `duplicate argument 'x' in function definition` |
| `return 5`（関数の外） | **通る** 🔴 | `'return' outside function` |
| `await foo()`（async の外） | **通る** 🔴 | `'await' outside function` |
| `break`（ループの外） | **通る** 🔴 | `'break' outside loop` |
| `1 = 2` | SyntaxError | SyntaxError |

再現コードはこれだけ。

```python
import ast

ast.parse("f(a=1, a=2)")                  # 何も起きない
compile("f(a=1, a=2)", "<t>", "exec")     # SyntaxError: keyword argument repeated: a
```

CPython は、ソースをパースして AST を作ったあと、**シンボルテーブルの構築とバイトコード生成の段階でさらに検査をしている**。重複キーワード引数、重複する仮引数名、`return` / `await` / `break` の置き場所は、どれも後者の段階のチェックだ。

`ast.parse` は `PyCF_ONLY_AST` 付きの `compile` で、AST を作ったところで止まる。つまり**後段の検査を丸ごとスキップする**。

ここが直感に反するところで、これらは実行時エラーではなく `SyntaxError` として報告される。だから「`SyntaxError` を捕まえたいなら `ast.parse` で十分だろう」と思ってしまう。実際には**`SyntaxError` の一部は `ast.parse` を通り抜ける**。

「関数の外の `return` くらい書かないよ」と思うかもしれないが、`f(a=1, a=2)` はマージやコピペで普通に発生する。私はそれで刺さった。

## 対処1: 実際に読み込む側と同じ厳しさで検査する

`ast.parse` を `compile` に変えるだけ。

```python
- ast.parse(f.read(), filename=path)
+ compile(f.read(), path, "exec")
```

コード自体は実行されない（バイトコードを作るだけ）ので、コストの性質は変わらない。

自分で書かずに済ませたいなら標準ライブラリで足りる。どちらも上の壊れたファイルを正しく落とした。

```console
$ python -m py_compile bad_mod.py
  File "bad_mod.py", line 2
    x.f(a=1, a=2)
             ^^^
SyntaxError: keyword argument repeated: a
$ echo $?
1

$ python -m compileall -q .    # ディレクトリ丸ごと
```

CI に1行入れるなら `python -m compileall -q .` が手軽だと思う。

教訓の形にするとこうなる。**検査は、実際に読み込む側と同じ厳しさで行う。** import されるコードを守りたいなら、import と同じ経路（= `compile`）まで通す。一段手前で止まる道具は「通った」と言うが、それは「読み込める」という意味ではない。

## 原因2: 構文が正しいまま、決定が消える

もう1本のファイルは、`compile` に変えても捕まらなかった。こうなっていたからだ。

```python
WEEKLY_CAP = {"site-a": 5, "site-b": 1, "site-c": 1}   # 承認済みの新しい値
# ...（間に長いコメントが挟まっている）
WEEKLY_CAP = {"site-a": 5, "site-b": 0, "site-c": 0}   # 統合で復活した古い行
```

構文は完全に正しい。import もできる。テストも通る。ただ Python はモジュール直下の代入を上から実行するので、**後の代入だけが効く**。数日前に決めて反映したはずの設定が、統合の副作用で静かに元へ戻っていた。エラーは何も出ない。

そして厄介なのは、これが**マージの構造上ほぼ必ず起きる**ことだ。競合の解消で「どちらの行も消したくない」と考えて両方残すと、代入文はこの形になる。差分を目で見ているときは、2つの `WEEKLY_CAP` が数十行離れていると気づけない。

dict のキーの重複（`{"a": 1, "a": 2}`）ならリンタが警告してくれるが、**別々の文に分かれた同名への代入**は、モジュールのトップレベルでは合法な再代入でしかないので誰も文句を言わない。

## 対処2: モジュール直下の定数の二重代入を見る

AST で拾える。トップレベルの `Assign` のうち、大文字の名前だけを数える。

```python
import ast, collections

def shadowed_constants(src: str) -> list[str]:
    tree = ast.parse(src)
    assigns = collections.defaultdict(list)
    for node in tree.body:                       # トップレベルだけ
        if not isinstance(node, ast.Assign):
            continue
        for t in node.targets:
            if isinstance(t, ast.Name) and t.id.isupper() and len(t.id) > 2:
                assigns[t.id].append(ast.dump(node.value))
    return sorted(k for k, v in assigns.items() if len(v) > 1)
```

これを全ファイルに掛けたら、**健全なファイル5本で誤検知した**。引っかかったのは2つのイディオムだった。

```python
ROOT = os.path.abspath(ROOT)     # ① 自己参照：前の値を使っている
BASE = BASE.rstrip("/")          #    → 決定は消えていない

CHANNEL_ID = "UC..."             # ② 完全に同じ値の再代入
CHANNEL_ID = "UC..."             #    → 冗長なだけで無害
```

どちらもマージ事故ではない。誤警報を放置すると、翌日から警報そのものが読まれなくなる。狼少年になるくらいなら検査を入れないほうがましなので、**マージ事故の形だけ**を指すように絞った。条件は「自己参照しておらず、かつ値が違う代入が2つ以上」。

```python
def shadowed_constants(src: str) -> list[str]:
    tree = ast.parse(src)
    assigns = collections.defaultdict(list)
    for node in tree.body:
        if not isinstance(node, ast.Assign):
            continue
        for t in node.targets:
            if isinstance(t, ast.Name) and t.id.isupper() and len(t.id) > 2:
                # X = f(X) の形か？ 前の値を使っているなら決定は消えていない
                refs_self = any(isinstance(n, ast.Name) and n.id == t.id
                                for n in ast.walk(node.value))
                assigns[t.id].append((refs_self, ast.dump(node.value)))

    out = []
    for name, rows in assigns.items():
        independent = [dump for refs_self, dump in rows if not refs_self]
        if len(independent) > 1 and len(set(independent)) > 1:
            out.append(name)
    return sorted(out)
```

`ast.dump(node.value)` を値の同一性の代わりに使っているのがポイントで、リテラルでも式でも構造が一致すれば同じ文字列になる。実際の挙動はこうなった。

```
merge accident         CAP = {"a": 1} / CAP = {"a": 0}        -> ['CAP']
self-referential       ROOT = "/x/y/" / ROOT = ROOT.rstrip()  -> []
identical dup          CHANNEL_ID = "UC123" ×2                -> []
three, one self-ref    BASE = "a" / BASE = BASE+"b" / BASE="c"-> ['BASE']
```

壊れた検体を置いて「鳴ること」、片付けて「鳴らないこと」を両方確かめてから入れた。検知器を書いたら、この2方向のテストは必ずやったほうがいい。片方だけだと「常に鳴らない検知器」を平気で本番に置ける。

## 教訓

ひとつめ。**「静的解析が通った」は「実行できる」の証明にならない。** どの段階まで通したかを意識する。パース・コンパイル・import・実行は別の関門で、それぞれ違うものを弾く。`ast.parse` が守ってくれる範囲は思っているより狭い。

ふたつめ。**構文検査で捕まる壊れ方は、むしろ幸運なほう**だった。import できないファイルは、次に実行したとき必ず落ちる。本当に怖かったのは二重代入のほうで、こちらは何も落ちず、何も出力せず、ただ設定が数日前の値で動き続けていた。気づいたのは「なぜかこのサイトだけ記事が生成されない」と別件を追ったときだ。

みっつめ。今回いちばん効いたのは、検査を直したことより**「全ファイルOK」と報告した自分を疑ったこと**だった。報告と実態がずれていたのは、検査が嘘をついたからではなく、私が検査の範囲を確かめずに「OK」を額面どおり読んだからだ。自動化を増やすほど、この種の「もっともらしい緑色」が増えていく。
