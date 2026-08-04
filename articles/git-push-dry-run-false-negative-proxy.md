---
title: "`git push --dry-run` は「pushできるか」の確認にならない"
emoji: "🚧"
type: "tech"
topics: ["git", "github", "proxy", "automation", "troubleshooting"]
published: true
---

個人の自動化プロジェクトで、記事の自動公開が滞留していました。切り分けのつもりで打った `git push --dry-run` が**成功したので「通信経路は問題ない」と結論した**のですが、これが完全な誤診でした。実際には塞がれていて、別系統では**7日間まるごと動画の自動投稿が止まっていた**のに誰も気付いていませんでした。

`--dry-run` が何を「やらない」のかを知らないと踏む罠なので共有します。

![dry-runはGETしか出さないので検査対象にならず成功する。本番pushはpackfileをボディに載せたPOSTを出すため403で遮断される](/images/git-push-dryrun-vs-real.png)

## 症状

自動化スクリプトからの `git push` がこう落ちます。

```
error: RPC failed; HTTP 403 curl 22 The requested URL returned error: 403
send-pack: unexpected disconnect while reading sideband packet
fatal: the remote end hung up unexpectedly
```

一方で、手元での操作は普通に通ります。`git clone` も `git fetch` も成功する。認証も切れていない。

## やってしまった切り分け

「403 なら認証かリポジトリ権限だろう」と当たりをつけつつ、まず経路を確認しました。

| 確認 | 結果 |
|---|---|
| `git ls-remote origin` | ✅ 成功 |
| `git push --dry-run origin main` | ✅ **成功** |
| `git push origin main` | ❌ HTTP 403 |

`--dry-run` が通ったので、**「ネットワークとしては push できている。403 はサーバー側の都合だ」**と結論しました。ここから権限設定やトークンのスコープを何時間も疑うことになります。全部ハズレでした。

## 原因: `--dry-run` は pack 本体を送らない

`git push --dry-run` の man の記述はこうです。

> `-n, --dry-run`
> Do everything except actually send the updates.

「更新を実際に送る以外の全部をやる」。この "actually send" が効いています。HTTP smart protocol での push は 2 段階です。

1. `GET /info/refs?service=git-receive-pack` — 相手の ref を取得してネゴシエーション
2. `POST /git-receive-pack` — **packfile をリクエストボディに載せて送る**

`--dry-run` は 1 で止まります。実際に `GIT_TRACE_CURL=1` でトレースを取ると一目瞭然でした（リモートに存在しない新規 ref を指定して、ネゴシエーションが省略されないようにしています）。

```console
$ GIT_TRACE_CURL=1 git push --dry-run origin HEAD:refs/heads/__probe 2>&1 \
    | grep -E "=> Send header: (GET|POST)"

=> Send header: GET /<owner>/<repo>.git/info/refs?service=git-receive-pack HTTP/2
=> Send header: GET /<owner>/<repo>.git/info/refs?service=git-receive-pack HTTP/2
```

**`POST` が 1 本も出ていません。** GET しか飛ばない。同じコマンドから `--dry-run` を外すと、ここに `POST /<owner>/<repo>.git/git-receive-pack` が加わります。

そして今回ブロックしていたのは、端末に常駐するタイプのセキュリティプロキシ（DLP エージェント）でした。この手の製品は**リクエストボディを検査して落とす**ので、ボディを持つリクエストを一度も発行しない `--dry-run` は検査対象にすらならず、素通りして成功します。

つまり `git push --dry-run` は、

- ✅ 「経路が通っているか」「認証が生きているか」の確認にはなる
- ❌ 「push が通るか」の確認には**ならない**

## 見分け方: 403 の**レスポンス本文の形**を見る

同じ 403 でも、誰が返した 403 かは本文で判別できます。REST API で叩くと分かりやすいです。

```console
# 読み取り → 通る
$ curl -s -o /dev/null -w '%{http_code}\n' \
    -H "Authorization: Bearer $TOKEN" \
    https://api.github.com/repos/<owner>/<repo>
200

# 書き込み → 403
$ curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
    https://api.github.com/repos/<owner>/<repo>/contents/test.txt \
    -d '{"message":"probe","content":"aGVsbG8="}' | head -c 200
<!DOCTYPE html><html><head><title>Access Denied</title>...
```

**本文が HTML なら、間に何かが挟まっています。** オリジン（GitHub 自身）が返す 403 は必ず JSON で、`{"message": "...", "documentation_url": "..."}` の形をしています。HTML のブロックページが返ってきた時点で、それは GitHub の応答ではありません。

ここが今回の本質で、遮断の形は「GitHub 全体が使えない」ではなく **「読みは許可・書きだけ遮断」の DLP 型**でした。この形だと、**GET ベースの疎通確認をいくら並べても永遠に検出できません。**

なお「回線を変えれば直るのでは」も外れです。端末常駐エージェント型は OS レベルで全通信をトンネルするので、テザリングや別 Wi-Fi に切り替えても同じように遮断されます。疑うべきは回線ではなく常駐プロセスの有無です。

## 対処: git transport を捨てる

プロキシ側の設定を変えられない前提だったので、**そもそも `git push` を使わない**方向に倒しました。GitHub Contents API なら、ファイル内容は Base64 の JSON ボディとして送られます。

```python
import base64, requests

def put_file(owner, repo, path, content: bytes, message, token, branch="main"):
    url = f"https://api.github.com/repos/{owner}/{repo}/contents/{path}"
    headers = {"Authorization": f"Bearer {token}",
               "Accept": "application/vnd.github+json"}

    # 既存ファイルの更新には sha が要る。無ければ新規作成
    r = requests.get(url, headers=headers, params={"ref": branch})
    sha = r.json().get("sha") if r.status_code == 200 else None

    body = {"message": message,
            "content": base64.b64encode(content).decode(),
            "branch": branch}
    if sha:
        body["sha"] = sha

    r = requests.put(url, headers=headers, json=body)

    # 🔴 ここが重要: 403 の中身を見て切り分ける
    if r.status_code == 403 and "text/html" in r.headers.get("content-type", ""):
        raise RuntimeError("proxy blocked (HTML body) — not a GitHub permission error")
    r.raise_for_status()
    return r.json()
```

環境によっては、この直接 PUT も同じく HTML 403 で落ちます。その場合は中継を 1 枚挟む（特定リポジトリの特定パス配下だけを許可する薄いプロキシを自前で立て、そこ経由で PUT する）という回避になりました。

### おまけの罠: `_` 始まりのファイルは GitHub Pages が配信しない

復旧確認のとき、デバッグ用に `_test.mp4` という名前でアップして 404 が返り、**「まだ遮断されている」と誤診しました。** これは無関係で、GitHub Pages（Jekyll）は `_` で始まるファイル・ディレクトリを既定で出力から除外します。リポジトリ直下に空の `.nojekyll` を置けば Jekyll 処理自体が無効になり、配信されるようになります。

障害調査中に**別の原因で同じ症状（404 / 403）が出る**と、直った変更を「効かなかった」と判定して捨ててしまうので、検証用のファイル名は普通のものにしておくべきでした。

## 教訓

- **dry-run / preflight を信用する前に「本番と同じコードパスを通るか」を確認する。** 通らない部分の検証にはならない。`--dry-run` は定義上ボディを送らないので、ボディに起因する失敗は原理的に検出できない
- **疎通確認を GET だけで組むと「読めるが書けない」障害を検出できない。** 書き込み系には書き込みのヘルスチェックを別に持つ
- **403 は本文の Content-Type で切り分ける。** JSON ならオリジン、HTML なら経路上の何かがブロックしている。ステータスコードだけ見て権限を疑い始めると時間が溶ける
- **失敗が終了コードにしか出ない自動化は、静かに何日も死ぬ。** 今回、動画の自動投稿が 7 日間まるごと停止していたのに気付けませんでした。「エラーが出ていない」ではなく**「成果物が出ているか」**で監視する必要があります

## 関連

「チェックは通っているのに実は壊れている」系の静かな失敗は、自動化を長く回しているとかなりの頻度で踏みます。同じ観点の別ケースはこちら: [Pillowで多言語テキストが静かに崩れる](https://zenn.dev/mameresearcher/articles/pillow-raqm-complex-text-layout)

自動化基盤の全体像はこちらにまとめています: [AIエージェントで16媒体のコンテンツ運用を回す全体アーキテクチャ](https://zenn.dev/mameresearcher/articles/ai-agent-multi-channel-content-ops)
