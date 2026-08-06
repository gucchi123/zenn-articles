---
title: "PowerShellで `Test-Path $a -and $b` が謎のエラーになる罠"
emoji: "🪤"
type: "tech"
topics: ["powershell", "windows", "bug", "automation"]
published: true
---

個人の自動化バッチ（毎朝メールを送るスクリプト）が、ある日を境に**エラーも出さず、通知だけが届かなくなる**という不具合に遭遇しました。原因はPowerShellの構文解釈の癖に起因するもので、他の言語に慣れた人ほど踏みやすい罠だったので共有します。

![書いたつもりの解釈とPowerShellの実際の解釈の違い。-andは論理演算子ではなくTest-Pathの名前付きパラメータとして読まれる](/images/powershell-and-parser-trap.png)

## 問題のコード

```powershell
if (Test-Path $NotifyScript -and (Test-Path $StatusJson)) {
    # 通知処理
}
```

一見、「NotifyScriptが存在し、かつStatusJsonも存在すれば」という意図が明確なコードに見えます。しかし実行すると、こんなエラーが出ます。

```
Test-Path : A parameter cannot be found that matches parameter name 'and'.
```

## 原因

PowerShellでは、コマンドレット呼び出しの引数はスペース区切りで解釈されます。`Test-Path $NotifyScript -and (Test-Path $StatusJson)` は、bashやPythonの感覚だと「`Test-Path $NotifyScript` の結果」と「`-and`」と「`(Test-Path $StatusJson)` の結果」を論理演算していると読めますが、PowerShellのパーサーは違う解釈をします。

`Test-Path` はコマンドレットなので、**その後に続くトークンは全部 `Test-Path` への引数**として解釈されようとします。つまり `-and` は論理演算子ではなく、`Test-Path` の**名前付きパラメータ**として認識されてしまうのです。`Test-Path` に `-and` というパラメータは存在しないため、上記のエラーになります。

厄介なのは、このエラーが**スクリプト全体をクラッシュさせず、該当の `if` ブロックだけが例外扱いでスキップされる**ケースがあることです。`$ErrorActionPreference` の設定次第では、エラーメッセージがログの奥に埋もれたまま、**該当のif文の中身が何日も実行されない状態**が静かに継続します。今回のケースでは、これが原因で通知処理が4日間動いていませんでした。

## 修正方法

コマンドレット呼び出しを明示的に括弧で囲み、パーサーに「ここでコマンドレットの引数は終わり」と伝えます。

```powershell
if ((Test-Path $NotifyScript) -and (Test-Path $StatusJson)) {
    # 通知処理
}
```

これだけです。`Test-Path $NotifyScript` を丸ごと括弧で包むことで、PowerShellは先に `Test-Path $NotifyScript` を評価してBool値を確定させ、その後に `-and` を論理演算子として正しく解釈します。

## 教訓

- PowerShellでコマンドレットの戻り値を論理演算子でつなぐときは、**各コマンドレット呼び出しを必ず括弧で囲む**
- `-and` `-or` に限らず、コマンドレットの後に続く「見慣れた記号」はまず名前付きパラメータとして解釈されると疑う
- こうした「エラーにはなるが処理全体は止まらない」系のバグは、**失敗時に必ず通知が飛ぶような設計**にしておかないと発見が遅れる。今回も「通知が来ない」ことに気づいた人がいなければ、もっと長く放置されていたはずです

## 書く前に見つける

この罠は「エラーは出るが処理は止まらない」ため、実行して気づくのが遅れます。書いた時点で潰せるよう、リポジトリ全体を一度だけ検査しておくと安全です。

`Test-Path`（に限らず任意のコマンドレット）の引数の位置に `-and` / `-or` が裸で置かれている箇所を拾います。

```powershell
# 「閉じ括弧を挟まずに -and / -or が現れる」= パーサーが名前付きパラメータとして読む形
Get-ChildItem -Recurse -Filter *.ps1 |
  Select-String -Pattern '\b(Test-Path|Get-Item|Get-Command)\s+[^()\r\n]*\s-(and|or)\b' |
  ForEach-Object { "{0}:{1}: {2}" -f $_.Filename, $_.LineNumber, $_.Line.Trim() }
```

ヒットしたら、そのコマンドレット呼び出しを括弧で包めば解消します。正しく書かれたコード（`(Test-Path $a) -and (Test-Path $b)`）は `-and` の手前に `)` が来るのでヒットしません。

PSScriptAnalyzer にはこのパターン専用のルールが無いため、こうした grep 相当の検査を CI や pre-commit に1行足しておくのが現実的です。

## 関連

「エラーは出ないのに実は動いていない」タイプの静かな失敗は、個人開発の自動化を続ける中で何度も踏みました。この観点を含めた自動化基盤の全体像はこちらにまとめています: [AIエージェントで16媒体のコンテンツ運用を回す全体アーキテクチャ](https://zenn.dev/mameresearcher/articles/ai-agent-multi-channel-content-ops)

---

📩 LINE で深掘り配信中

AI / マーケ / 楽天モバの限定情報を 週1〜2回 お届け（無料）

興味のあるテーマだけ選んで受け取れます

[友だち追加する 👉](https://mame-follow.suikou0.workers.dev/follow?cha=zenn)

AIエージェント運用 / MMM / 楽天モバ紹介 の3テーマから選べます
