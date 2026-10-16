---
title: "subprocess.run(timeout=600) が5時間戻らなかった — Windows の .cmd ラッパーと孫プロセスのパイプ"
emoji: "⏳"
type: "tech"
topics: ["python", "windows", "subprocess", "automation", "個人開発"]
published: true
---

個人の自動化プロジェクトで、記事やSNS投稿の下書きを LLM の CLI に書かせている。Python から CLI を呼ぶ箇所は30か所ほどあり、すべてに `timeout=` を付けていた。CLI が固まっても、10分で諦めて次に進むつもりだった。

ある日、そのうちの1本が **5時間** 止まっていた。Task Scheduler の実行時間の上限に当たって強制終了されるまで、一度も戻ってこなかった(結果コード `0x41306` = 267014、タスクの強制終了)。

`timeout=600` を付けた `subprocess.run` が、600秒で戻らなかったということになる。

![subprocess.run(timeout=) が戻らない: 止まるのは cmd.exe だけ](/images/subprocess-timeout-grandchild-pipe.png)

## 症状: タイムアウトを付けたのに戻らない

問題のコードはこう書いていた。

```python
import shutil, subprocess

claude = shutil.which("claude")   # Windows では ...\npm\claude.cmd が返る
r = subprocess.run(
    [claude, "-p", "--output-format", "text"],
    input=prompt,
    capture_output=True, text=True, encoding="utf-8",
    timeout=600,
)
```

ログには「CLI を呼び出した」行までしか残っていなかった。`TimeoutExpired` の行も、例外のトレースバックも無い。プロセスは生きたまま、何も出力せずに止まっていた。

その日は CLI 側が利用上限に当たっていて、応答を返さずに固まる状態だった。固まること自体は想定していて、そのために `timeout=600` を付けていた。

## 原因: kill されるのは cmd.exe だけ、パイプを握っているのは孫

npm で入れた CLI は、Windows では `claude.cmd` というバッチファイルとして置かれる。これを `subprocess.run` で起動すると、プロセスの親子関係はこうなる。

```
python.exe
 └─ cmd.exe /c claude.cmd    ← subprocess が知っているのはこれだけ
     └─ node.exe ...          ← 本体。stdout/stderr のパイプを継承している
```

`capture_output=True` で作ったパイプは、cmd.exe を経由して孫の node.exe にも継承される。

ここで CPython の `subprocess.run` の実装を見る(3.12 の `Lib/subprocess.py`)。

```python
        except TimeoutExpired as exc:
            process.kill()
            if _mswindows:
                # Windows accumulates the output in a single blocking
                # read() call run on child threads, with the timeout
                # being done in a join() on those threads.  communicate()
                # _after_ kill() is required to collect that and add it
                # to the exception.
                exc.stdout, exc.stderr = process.communicate()
            else:
                process.wait()
            raise
```

タイムアウトすると、次の順に処理が進む。

1. `process.kill()` が終了させるのは **直接の子の cmd.exe だけ**。Windows の `TerminateProcess` は子孫を巻き込まない
2. Windows では、出力を回収するために kill のあと `communicate()` を **タイムアウト無しで** 呼ぶ
3. `communicate()` はパイプの EOF を待つ。しかしパイプの書き込み側は、生き残った node.exe がまだ握っている
4. node.exe が固まったままなら EOF は永久に来ない。`TimeoutExpired` は投げられない

つまり `timeout=600` は、600秒後に cmd.exe を kill するところまでは効いていた。止まっていたのは、その後の出力回収だった。

### 再現

本物の CLI を使わなくても再現できる。孫プロセスが15秒眠るだけのバッチファイルを用意する。

```bat
@echo off
python -c "import time; time.sleep(15); print('done')"
```

```python
import subprocess, time

t = time.time()
try:
    subprocess.run([r"C:\path\to\slow.cmd"], capture_output=True, text=True, timeout=2)
except subprocess.TimeoutExpired:
    print("TimeoutExpired after %.1fs" % (time.time() - t))
```

```
TimeoutExpired after 16.2s
```

`timeout=2` なのに、戻ったのは16秒後。孫が自然に終わってパイプが閉じるまで待っていた。孫が15秒で終わらず固まっていれば、この呼び出しは戻らない。

## 対処: Popen で起動し、タイムアウトしたら木ごと止める

`subprocess.run` と同じ引数・戻り値・例外で使えるラッパーを1つ作り、LLM の CLI を呼ぶ箇所をすべて置き換えた。

```python
import os
import subprocess


def _kill_tree(p: subprocess.Popen) -> None:
    if os.name == "nt":
        # /T で子孫まで、/F で強制終了
        subprocess.run(["taskkill", "/T", "/F", "/PID", str(p.pid)],
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    else:
        p.kill()


def run(*popenargs, input=None, capture_output=False, timeout=None, check=False, **kwargs):
    if input is not None:
        kwargs["stdin"] = subprocess.PIPE
    if capture_output:
        kwargs["stdout"] = subprocess.PIPE
        kwargs["stderr"] = subprocess.PIPE
    with subprocess.Popen(*popenargs, **kwargs) as p:
        try:
            out, err = p.communicate(input, timeout=timeout)
        except subprocess.TimeoutExpired as e:
            _kill_tree(p)
            try:
                # 回収にも上限を付ける。パイプがまだ握られていても待ち続けない
                out, err = p.communicate(timeout=10)
            except Exception:
                out, err = None, None
            raise subprocess.TimeoutExpired(p.args, timeout, output=out, stderr=err) from e
        except BaseException:
            _kill_tree(p)
            raise
        rc = p.poll()
    if check and rc:
        raise subprocess.CalledProcessError(rc, p.args, output=out, stderr=err)
    return subprocess.CompletedProcess(p.args, rc, out, err)
```

ポイントは2つ。

- **kill の範囲を木全体にする。** `taskkill /T /PID <cmd.exe の PID>` で cmd.exe の子孫(node.exe)まで止める。孫が消えればパイプの書き込み側が閉じ、EOF が届く
- **kill 後の回収にも上限を付ける。** `communicate(timeout=10)` にしておけば、万一どこかにパイプが残っていても10秒で諦める

同じ再現スクリプトをこのラッパーで動かすと、こうなった。

```
TimeoutExpired after 3.9s
```

呼び出し側は `subprocess.run` を `run` に差し替えるだけでよい。Python の子プロセスなど、孫を作らないことが分かっている呼び出しは触っていない。

### PowerShell 側も同じ穴があった

PowerShell から `& cmd /c "claude.cmd -p ... < prompt.txt"` と呼んでいるスクリプトも2本あった。こちらはそもそもタイムアウトが無く、CLI が固まればタスクの上限まで待つ。しかもタスク上限で止まるのは powershell.exe で、cmd.exe と node.exe は残ることがある。

こちらも `Start-Process -PassThru` で起動し、`WaitForExit(60000)` を回して上限を超えたら `taskkill /PID <pid> /T /F` する関数にまとめた。タイムアウト時の終了コードは `timeout` コマンドにならって 124 にしている。

```powershell
$proc = Start-Process -FilePath "$env:SystemRoot\System32\cmd.exe" `
    -ArgumentList ('/d /s /c "' + $full + '"') -WindowStyle Hidden -PassThru
$null = $proc.Handle   # PS 5.1: 先に Handle を取らないと終了後に ExitCode が $null になる
$sw = [Diagnostics.Stopwatch]::StartNew()
while (-not $proc.WaitForExit(60000)) {
    if ($sw.Elapsed.TotalMinutes -ge $LimitMinutes) {
        Start-Process taskkill.exe -ArgumentList "/PID $($proc.Id) /T /F" -WindowStyle Hidden -Wait
        $rc = 124; break
    }
}
```

## どこで踏みやすいか

`shutil.which()` で見つけた実行ファイルが `.cmd` / `.bat` なら、間に cmd.exe が挟まる。npm で入れた CLI(Node 製の多くのツール)は、Windows ではほぼこの形になる。

- `.exe` を直接起動していれば、kill の対象は本体そのものなのでこの問題は起きにくい
- `.cmd` 経由でも、本体が正常に終われば何も起きない。**本体が固まった時にだけ** 表に出る
- `capture_output=True`(あるいは `stdout=PIPE`)を指定しているときに起きる。出力をファイルに向けていれば `communicate()` が EOF を待つことはない

「固まった時にだけ」起きるので、普段のテストでは見つからない。タイムアウトが本当に必要になった日に初めて、タイムアウトが効かないことが分かる。

POSIX でも、`subprocess.run` の kill が止めるのは直接の子だけという点は同じだ。POSIX では kill 後に `wait()` するだけなので呼び出し自体は戻るが、孫は孤児として残り続ける。こちらで木ごと止めたいなら、`start_new_session=True` で起動して `os.killpg()` でプロセスグループごと止めるのが定石になる。

## 教訓

- **`timeout=` は「この時刻に kill する」という指定であって、「この時間で必ず戻る」という保証ではない。** kill が届く範囲の外にパイプを握ったプロセスがいれば、待ちは終わらない
- **Windows で `.cmd` / `.bat` を起動すると、kill されるのは cmd.exe だけになる。** タイムアウトを付けるなら `taskkill /T` でプロセスの木ごと止める
- **タイムアウトの後始末にもタイムアウトを付ける。** kill 後の `communicate()` を無期限にしないこと
- **タイムアウトの実装は、本当に固まる子プロセスで試す。** 正常に終わる子プロセスでは、この穴は見えない

---

📩 LINE で深掘り配信中

AI / マーケ / 楽天モバの限定情報を 週1〜2回 お届け（無料）

興味のあるテーマだけ選んで受け取れます

[友だち追加する 👉](https://mame-follow.suikou0.workers.dev/follow?cha=zenn)

AIエージェント運用 / MMM / 楽天モバ紹介 の3テーマから選べます

