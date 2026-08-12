---
title: "無出力のプロセスを「固まった」と判断して殺した日 — 生存確認の設計"
emoji: "💀"
type: "tech"
topics: ["powershell", "automation", "cli", "devops", "monitoring"]
published: true
---

[エージェントに媒体運用を任せた3か月](https://zenn.dev/mameresearcher/articles/agent-media-ops-gates-and-verification) の続きです。前回は「関門」と「確認」で事故が起きるという話でした。今回はその確認のほうで、**正常に動いているプロセスを異常だと判断して殺した**話を書きます。

同じ日に2回やりました。原因は技術的な不具合ではなく、私の判断です。

![無出力のプロセスをどう見分けるか](/images/long-running-process-silence.png)

## 何が起きたか

日次で動画を作るパイプラインがあります。実体は PowerShell から CLI エージェントを叩くだけの構成です。

```powershell
$cmdLine = "$agent -p --trust --workspace $ws --output-format text < $promptFile"
& cmd /c $cmdLine 2>&1 | ForEach-Object {
    Log "  agent: $_"
}
```

ログを見ると、こう出て止まっていました。

```
[11:33:37] Launching cursor-agent in non-interactive mode...
[11:33:37] ----- Agent Output -----
[11:33:37]   prompt staged to: C:\...\prompt_3f95cdc7.txt
（以降なし）
```

7分待ってもログが1行も増えません。私は「固まっている」と判断してプロセスを殺し、手作業で作り直しました。

**それが間違いでした。**

## 実測してみると

原因究明のため、同じ呼び出しを手で再現しました。

```powershell
$sw = [Diagnostics.Stopwatch]::StartNew()
$out = & cmd /c $cmdLine 2>&1
$sw.Stop()
"rc=$LASTEXITCODE  所要=$([math]::Round($sw.Elapsed.TotalSeconds,1))秒  出力行=$(($out|Measure-Object).Count)"
```

結果はこうでした。

| プロンプト | 所要 | 出力行 |
|---|---:|---:|
| 最小（30字） | **84.6秒** | 1 |
| 本番（23,017字） | **197.6秒** | 64 |

**このCLIは完了まで1行も出力しません。** 内部でバッファリングして、終了時にまとめて吐き出します。つまり「ログが伸びない」は異常のサインではなく、**このプロセスの正常な挙動**でした。

7分＝420秒。本番の所要197秒の倍以上待っていたつもりでしたが、1回目の起動は別の理由で失敗しており、2回目は起動から7分後ではなく**起動直後を見ていた**のです。時系列を取り違えていました。

## もっと悪かったのは「プロセス数で生きていると判断した」こと

途中で私はこう考えました。「タスクマネージャに `cursor-agent` が5プロセスある。だから動いている」

これは根拠になりません。**その名前のプロセスは IDE 本体も使っています。** 私が起動したバッチ処理とは無関係のプロセスを数えて、生存の証拠にしていました。

日頃「帳簿ではなく成果物を見る」と自分に言い聞かせていたのに、**プロセス一覧という帳簿**を見ていたわけです。成果物（生成されたファイル）を見ていれば、11:01 で更新が止まっていること、しかもそのファイルが**2週間前の日付の素材**だということにすぐ気づけました。

## 対策：無音であることを、プロセス自身に喋らせる

無出力が正常なら、外から生死を判断する材料がありません。ならば**心拍を別に出す**しかありません。

```powershell
Log ("  ※ このCLIは完了まで出力しません。プロンプト {0:N0} 字で通常 3〜5 分かかります。" -f $Prompt.Length)
Log "  ※ 途中でログが伸びなくても停止と判断しないこと (60秒ごとに経過を出します)"

$sw = [Diagnostics.Stopwatch]::StartNew()
$job = Start-Job -ScriptBlock { param($c) & cmd /c $c 2>&1; $LASTEXITCODE } -ArgumentList $cmdLine

while ($job.State -eq 'Running') {
    Start-Sleep -Seconds 60
    if ($job.State -eq 'Running') {
        Log ("  [heartbeat] 実行中 {0:N0} 秒経過" -f $sw.Elapsed.TotalSeconds)
    }
    if ($sw.Elapsed.TotalMinutes -ge 20) {
        Log "[ERROR] 20分returnせず → 打ち切り"
        Stop-Job $job; break
    }
}
$sw.Stop()

$raw = @(Receive-Job $job)
Remove-Job $job -Force
# 最終要素が終了コード。それ以外が出力行
$rc = 99
if ($raw.Count -gt 0 -and $raw[-1] -is [int]) { $rc = $raw[-1]; $raw = $raw[0..($raw.Count-2)] }
foreach ($line in $raw) { Log "  agent: $line" }
Log ("  所要 {0:N1} 秒 / 出力 {1} 行" -f $sw.Elapsed.TotalSeconds, $raw.Count)
```

ポイントは3つです。

**1. 開始時に所要時間の目安をログへ書く。** 次に見る人（未来の自分を含む）が「3〜5分かかると書いてあるな」と読めれば、7分で殺しません。ドキュメントに書いても読まれませんが、ログには目が向きます。

**2. 60秒ごとに心拍を出す。** `Start-Job` でバックグラウンド化して、親側でポーリングします。パイプラインを直接回すと出力が来るまでブロックされるので、心拍を出す隙がありません。

**3. 上限を設けて自動で打ち切る。** 20分returnしなければ本当に異常です。人間の判断に頼らず機械的に切ります。

`Receive-Job` の戻り値の最後に終了コードが混ざるので、`$raw[-1] -is [int]` で分離しています。ここは素直に書くと出力行に終了コードが紛れ込みます。

## ついでに踏んだ、もう一つの計測ミス

原因調査中、「プロンプトが44,264文字で警告閾値の1.8倍だ」と結論しかけました。

```powershell
$raw = Get-Content $src -Raw
$raw.Length          # 44264
```

これは**バイト数**です。日本語UTF-8は1文字3バイトなので、実際は 23,017 文字。閾値24,000の**範囲内**でした。

```powershell
[System.IO.File]::ReadAllText($src, [System.Text.Encoding]::UTF8).Length   # 23454
```

`Get-Content -Raw` はファイルの中身をそのまま読むため、エンコーディングを解釈した文字数とは一致しません。存在しない問題を作り出して、不要なリファクタリングをするところでした。

## まとめ

- **無出力は異常とは限らない。** そのプロセスの正常な挙動かどうかを、殺す前に実測で確かめる
- **プロセス数は生存証明にならない。** 同名の無関係なプロセスが混ざる。成果物か、プロセス自身が出す心拍で判断する
- **無音のプロセスには心拍を持たせる。** 加えて開始時に所要目安をログへ書くと、次に見る人が誤判定しない
- **文字数とバイト数を混同しない。** 日本語は3倍になる

一番の教訓は、**「動いていないように見える」と「動いていない」は別物**だということです。前者は観測の話で、後者は事実の話です。観測手段を持たないまま前者を後者だと決めつけると、動いているものを壊して「壊れていた」と結論づけてしまいます。

今回はその結果、その日の自動生成が失われました。手作業で作り直せたので実害は時間だけでしたが、判断の順序としては逆でした。**まず観測手段を用意し、それから判断する**べきでした。
