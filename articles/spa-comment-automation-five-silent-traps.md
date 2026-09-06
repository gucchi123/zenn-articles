---
title: "投稿1件を「3件」と数えた — SPAのコメント欄を自動化して踏んだ、全部エラーにならない5つの罠"
emoji: "🧵"
type: "tech"
topics: ["playwright", "javascript", "frontend", "automation", "個人開発"]
published: true
---

個人の自動化プロジェクトで、他媒体の記事に付けたコメントを追いかけ、著者から返信が来ていたらそこへ返す、という処理を回している。

対象はコメント欄がスレッド構造になっている Web サービス（Next.js の SPA、エディタは CodeMirror）。トップレベルにコメントを投稿するスクリプトは既にあったので、最初はそれを使い回そうとした。**が、それだと会話が2本に割れる。** 著者の返信への応答が、スレッドの中ではなく新しいトップレベルコメントとして生えてしまうからだ。

そこで「スレッド内に返信する」版を書いた。やることは、返信欄を開く・本文を入れる・ボタンを押す、それだけのはずだった。

実際には **5つの罠を踏んだ。そして5つとも、例外を投げなかった。** 終了コードは 0、ログにも異常なし、そして何も投稿されていない（あるいは二重投稿の一歩手前）。

![DOMのテキスト走査は、投稿1件を3件と数える](/images/spa-comment-three-counts.png)
*同じ本文がページ内に3箇所ある。素直に数えると「二重投稿」と誤警報する*

## 最初に書いた素直な実装

```js
// これが全部間違っている
const btn = [...document.querySelectorAll('button')]
  .find((x) => /投稿する/.test(x.innerText));   // ← 罠1
const ed = document.querySelector('[contenteditable="true"]');  // ← 罠2
if (ed.textContent.trim().length > 0) return '空じゃない';       // ← 罠3
paste(ed, BODY);                                                 // ← 罠4
btn.click();
console.log('送信した');                                         // ← 罠5
```

「動くはず」と思って書ける形をしている。ここから順に潰していく。

---

## 罠1: 同じラベルのボタンが、ページヘッダーにもある

返信の送信ボタンのラベルは **「返信する」** で、「投稿する」ではなかった。ここまでは単なる調査不足だ。問題は、`投稿する` で探したときに **何も見つからなかったのではなく、別のボタンが見つかった** ことだ。

このサービスのページヘッダーには「記事を投稿する」ボタンが常時ある。グローバルな `find` は DOM 順で最初に一致したものを返すので、**ヘッダーの `y=13` にあるボタン**を掴む。押すと記事投稿画面に行こうとするだけで、コメントは当然出ない。例外も出ない。

```js
// 悪い: ページ全体から探す。ヘッダーのボタンを掴む
const b = [...document.querySelectorAll('button')].find((x) => /投稿する/.test(x.innerText));

// 良い: 枠を特定してから、その枠の中だけで探す。しかも個数を検算する
const box = document.querySelector('[data-auto-thread="1"]');
const bs = [...box.querySelectorAll('button')]
  .filter((x) => (x.innerText || '').trim() === '返信する' && !x.disabled);
if (bs.length !== 1) return `枠内の「返信する」が ${bs.length} 個`;
bs[0].click();
```

部分一致 (`/返信する/`) ではなく完全一致 (`=== '返信する'`) にしているのも意図的で、「返信する」を含む長いラベル（「返信するにはログイン」など）を拾わないためだ。そして **1個であることを検算する**。0個なら押していないし、2個なら押した先が分からない。どちらも「押した」で先に進んではいけない。

## 罠2: 入力欄が2つあり、親を辿る絞り込みが効かない

スレッドの「返信を追加」を押すと返信欄が現れる。だが **トップレベルのコメント投稿欄も、ページ下部にそのまま在る。** `contenteditable` は常に2つだ。

普通なら「アンカーになる要素から親を辿って、返信欄を含む一番小さい枠を探す」で絞る。これが効かない。返信欄を含む枠を親方向に広げていくと、**広げきる前にトップレベルの投稿欄も入ってしまう**構造だった。しかも後述する下書き復元のせいで、両方に同じ本文が入っていて見分けが付かない。

解決は、探すのをやめて **印を付けて差分を取る**ことだった。

```js
// 開く前に、既に在る入力欄すべてに印を付ける
await page.evaluate(() => {
  document.querySelectorAll('[contenteditable="true"]')
    .forEach((e) => e.setAttribute('data-auto-old', '1'));
});

await openThread(ANCHOR);          // 「返信を追加」を押す
await page.waitForTimeout(3000);

// 後から増えた1つが返信欄
const marked = await page.evaluate(() => {
  const fresh = [...document.querySelectorAll('[contenteditable="true"]')]
    .filter((e) => !e.hasAttribute('data-auto-old'));
  if (fresh.length !== 1) return `新しく現れた入力欄が ${fresh.length} 個`;  // 0個でも2個でも止める
  fresh[0].setAttribute('data-auto-editor', '1');
  return true;
});
if (marked !== true) { console.log('🔴', marked); await bye(1); }
```

「どれが目的の要素か」をセレクタで言い当てるより、**操作の前後で増えたものを取る**ほうが確実だった。セレクタは DOM 構造の変更で壊れるが、「押したら1個増える」は仕様そのものなので壊れにくい。

同じ要領で、返信欄と「返信する」ボタンが 1対1 で対応する最小の枠にも印を付けておく。これが罠1で使った `data-auto-thread` になる。

```js
let up = fresh[0];
for (let i = 0; i < 10 && up; i++, up = up.parentElement) {
  const eds = up.querySelectorAll('[contenteditable="true"]').length;
  const bs = [...up.querySelectorAll('button')]
    .filter((x) => (x.innerText || '').trim() === '返信する');
  if (eds === 1 && bs.length === 1) { up.setAttribute('data-auto-thread', '1'); return true; }
}
```

条件が `eds === 1 && bs.length === 1`（入力欄1つ・ボタン1つ）なのが肝で、広げすぎて2つ目の入力欄が入った瞬間に条件が外れる。**「見つかった」ではなく「1対1になった」で止める。**

## 罠3: 空判定が、プレースホルダの文字数を数えていた

貼る前に入力欄が空であることを確かめたい。素直に書くとこうなる。

```js
if (ed.textContent.trim().length > 0) return '空じゃない';
```

これが **永久に通らない**。エディタが CodeMirror で、**空のときもプレースホルダが実要素として DOM に描画される**からだ。`textContent` には「返信内容を入力」の7文字が常に入っている。`<textarea>` の `placeholder` 属性は `value` に影響しないので同じ感覚で書くと必ず外す。

```js
const left = await ed.evaluate((e) => {
  const c = e.cloneNode(true);                      // 表示中のDOMは壊さない
  c.querySelectorAll('.cm-placeholder, [class*="placeholder"]').forEach((x) => x.remove());
  return { real: (c.textContent || '').trim(), 見えている文字: (e.textContent || '').trim() };
});
console.log('貼る前の残り:', JSON.stringify(left));
if (left.real.length > 0) { console.log('🔴 空にできなかった。送らない'); await bye(1); }
```

実測ログはこうなる。**`real` と `見えている文字` の両方を出しているのが重要**で、片方だけだと「空にできていない」のか「プレースホルダを数えている」のかが切り分けられない。

```
貼る前の残り: {"real":"","見えている文字":"返信内容を入力"}
```

## 罠4: 下書きが自動保存され、貼ると追記されて二重になる

このサービスはコメントの下書きをローカルに保存する。ページを開き直すと、**未送信の本文が入力欄に復元される。** そこへ貼ると置き換えではなく追記になり、本文が2回入る。以前これで 416 文字の投稿を作りかけた。

だから貼る前に必ず空にする。ただし `document.execCommand('selectAll')` は CodeMirror / ProseMirror 系のエディタでは効かない（`false` を返すか、何も起きずに `true` を返す）。**Playwright の実キー入力**で消す必要がある。

```js
const ed = page.locator('[data-auto-editor="1"]');
await ed.click();
await page.keyboard.press('Control+A');
await page.keyboard.press('Delete');
await page.waitForTimeout(800);
// ここで罠3の検算を通す。「消したつもり」で先に進まない
```

入力自体も `type` ではなく `ClipboardEvent('paste')` を使っている。`type` は環境によって半角英数字を落とすことがあり、それも例外にならないためだ（[別記事](https://zenn.dev/mameresearcher/articles/browser-type-silently-drops-ascii)に書いた）。そして貼った直後に完全一致を検算し、違えば送らない。

```js
const chk = await page.evaluate((text) => {
  const e = document.querySelector('[data-auto-editor="1"]');
  e.focus();
  const dt = new DataTransfer();
  dt.setData('text/plain', text);
  e.dispatchEvent(new ClipboardEvent('paste', { clipboardData: dt, bubbles: true, cancelable: true }));
  const got = ([...e.children].map((c) => c.textContent).join('\n') || e.innerText).trim();
  return { got, ok: got.replace(/\s/g, '') === text.replace(/\s/g, ''), lenGot: got.length };
}, BODY);
console.log(`検算: 文字 ${BODY.length} -> ${chk.lenGot} / 一致 ${chk.ok}`);
if (!chk.ok) { console.log('🔴 一致しない。送らない'); await bye(1); }
```

改行の扱いだけは CodeMirror が行を `.cm-line` に分けるため、`children` を `\n` で連結して復元している。比較は空白を除いた上で行う。

## 罠5: 投稿1件を「3件」と数える

一番厄介だったのがこれだ。二重投稿を防ぐため、送信の前後で「同じ書き出しがページに何件あるか」を数えていた。前が0件なら送る、後が1件なら成功、という設計だ。

`document.body` のテキストを素直に走査すると、**投稿1件に対して 3 が返ってきた。**

| 数えていたもの | 正体 |
| --- | --- |
| 1件目 | 実際に投稿されたコメント（これだけが本物） |
| 2件目 | 入力欄に復元された**自分の下書き**（罠4の副作用） |
| 3件目 | `__NEXT_DATA__` の中の JSON |

3件目が盲点だった。Next.js は SSR したページの props を `<script id="__NEXT_DATA__" type="application/json">` としてページ内に埋め込む。**コメント一覧がそこに JSON でもう1回入っている。** `innerText` では拾われないが、テキストノードを再帰的に走査する実装だと確実に入る。

この誤カウントは **両方向に嘘をつく**のが厄介だ。

- **送信前**のチェックで数えると、自分の下書きを見て「既に投稿済み」と判断し、**送信を止める**（偽陽性）
- **送信後**のチェックで数えると、1件しか投稿していないのに「2件ある＝二重投稿だ」と**誤警報する**

同じ関数が、呼ぶ場所によって逆向きの嘘をつく。修正は、走査から「自分が書き込んだ場所」と「機械可読の埋め込みデータ」を外すことだった。

```js
const countHead = () => page.evaluate((h) => {
  let n = 0;
  const walk = (el) => {
    if (el.nodeType === 1) {
      // 自分の下書き（入力欄）を外す
      if (el.getAttribute('contenteditable') === 'true') return;
      if (/\bcm-editor\b/.test((el.className || '').toString())) return;
      // 機械可読の埋め込みを外す。__NEXT_DATA__ はここで落ちる
      if (el.tagName === 'SCRIPT' || el.tagName === 'STYLE' || el.tagName === 'TEMPLATE') return;
    }
    for (const c of el.childNodes) {
      if (c.nodeType === 3) n += c.textContent.split(h).length - 1;  // 正規表現を使わない
      else if (c.nodeType === 1) walk(c);
    }
  };
  walk(document.body);
  return n;
}, HEAD);
```

数え方に正規表現を使わず `split().length - 1` にしているのは、本文に `(` や `?` などの記号が入ると `new RegExp(本文)` が壊れる（あるいは意図しない一致をする）からだ。エスケープを書くより、リテラル分割のほうが事故がない。

## 最終形

送信までの流れはこうなった。**5箇所すべてに「違ったら送らない」を置いている。**

```js
await load();

// 1. 送信前: 同じ書き出しが既に無いか（下書きと __NEXT_DATA__ を除いて数える）
const dup = await countHead();
if (dup > 0) { console.log('🔴 既に同じ内容がある。送らない'); await bye(1); }

// 2. 開く前に印 → 開く → 増えた1つを返信欄として同定（1個でなければ止める）
// 3. 実キー入力で空にする
// 4. プレースホルダを除いて「本当に空」を検算（空でなければ送らない）
// 5. paste 後に完全一致を検算（一致しなければ送らない）
if (DRY) { console.log('[dry-run] 送信しない'); await bye(0); }

// 6. 枠内の「返信する」がちょうど1個であることを確かめて押す
// 7. 🔴 「押した」で終わらせない。読み直して1件だけ在ることを見る
await load();
const after = await countHead();
console.log('成立確認: 本文の出現', after, '件', after === 1 ? '✅' : '🔴');
await bye(after === 1 ? 0 : 1);
```

最後の `load()` （ページの再読み込み）は省きたくなるところだが、省くと**送信直後の楽観的更新（クライアント側だけの描画）を「成立」と読んでしまう**。サーバに入ったことを確かめたいなら、読み直すしかない。

## 教訓

1. **「押した」を成功と数えない。** クリックが例外を投げないのは、正しいボタンを押した証拠にならない。読み直して、成果物を数える。
2. **数えるときは、自分が書き込んだ場所を数から外す。** 入力欄・下書き・キャッシュは「自分が入れたもの」であって「相手に届いたもの」ではない。検証が自分の入力を数えると、成功も失敗も同じ数字になる。
3. **グローバルな検索は必ず枠に閉じる。そして個数を検算する。** ラベルの重複は珍しくない。`find` は「見つからない」より **「別のものが見つかる」ほうが怖い**。0個でも2個でも止める。
4. **要素の同定は「探す」より「印を付けて差分を取る」。** 「押したら1個増える」は仕様だが、セレクタは実装の詳細だ。壊れにくいほうに賭ける。
5. **SPA の DOM は、見えているものと入っているものが違う。** プレースホルダは実要素として在り、SSR の props は script に JSON で在り、下書きは復元されて在る。`textContent` はそのすべてを平等に返す。

そして 5 つに共通するのは、**どれも例外を投げなかった**ことだ。この種の失敗は、**検算を書いた分しか見つからない。** 上のスクリプトが 173 行あるうち、実際の操作は 20 行ほどで、残りはほぼ全部「違ったら送らない」の検査だ。比率としては正しいと思っている。
