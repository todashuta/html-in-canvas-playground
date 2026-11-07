# AGENTS.md

Chrome 157から入ったHTML-in-Canvas APIは、DOMをcanvasのWebGL/WebGPUテクスチャに描ける。ここは、Babylon.jsの`HtmlTexture`で試す専用リポジトリ。

## 構成

| ファイル | 役割 |
| --- | --- |
| `index.html` | デモ本体。スタイルとスクリプトを全部内蔵した単一ファイル。外部スクリプトはCDNから読むBabylon.jsとpolyfillの2つ。Babylon.jsはv9.33.2を固定して読む。polyfillは本体APIの無い環境で動的に読む |
| `README.md` | 使い方と動作確認のしかた |

## 起動と確認

```
python3 -m http.server 8080
# http://localhost:8080/
```

`?selftest` を付けると、consoleに像の確認の行が出る。結果の判定はその行で行える。

GPUの無いLinuxでは、snapのChromiumをラッパーではなく実体で呼ぶ。ラッパーはsystemdのtransient scopeを作れずに落ちる。`HOME` / `XDG_*` / `TMPDIR` / `--user-data-dir` は `/tmp` 以下のような短いパスに置く。chromiumは起動時に `--user-data-dir` の下へUNIXソケットを作る。このソケットのパスが長いと、パス長制限で落ちる。

```
mkdir -p /tmp/cr-home /tmp/cr-run
DBUS_SESSION_BUS_ADDRESS=disabled: \
  HOME=/tmp/cr-home XDG_CONFIG_HOME=/tmp/cr-home/.config XDG_CACHE_HOME=/tmp/cr-home/.cache \
  XDG_RUNTIME_DIR=/tmp/cr-run TMPDIR=/tmp \
  timeout 45 /snap/chromium/current/usr/lib/chromium-browser/chrome \
  --headless=new --no-sandbox --use-angle=swiftshader --enable-unsafe-swiftshader \
  --enable-logging=stderr \
  --user-data-dir="/tmp/cr-prof-$(date +%s)" \
  'http://127.0.0.1:8080/?selftest' 2>&1 | grep -a '\[hic\]'
```

```
[hic] route=polyfill（three-html-render） nativeBefore=false
[hic] 初回 画素[0] RGBA=17,26,51,255      ← パネルの背景色 #111a33 と一致で OK
[hic] update 後 画素[0] RGBA=35,43,71,255 ← 「色を変える」で換えた色に差し替わっていれば OK
[hic] selftest DONE count=1 tint=1
```

`--virtual-time-budget` は付けない。polyfill経路での実測では、仮想時間でpaintが発火せず、onLoadまで来ない。selftestはrouteの表示まで進んだのち、「onLoad が 9 秒で来ませんでした」のエラーで止まる。像の画素は読まれない。paintやonLoadの待ちには実時間の描画が要るから、実時間で流す。

初回の画素が一致すれば「DOMが像に載っている」、update後に画素が変われば「DOMの更新が像に写っている」を機械的に確かめられる。テクスチャの左上の画素の色が、確かめたい対象の背景色と一致するかを見る方法は、DOMが貼れているかの判定に使える基本的な手。

テストはない。動作の確認は `?selftest` と画面の診断表示で行う。

## 動く経路（3段）

デモは次の順で経路を選び、開いたときの「経路」表示でどれが使われたかを出す。

1. native（本体 API）…Chrome 157+。判定は `"captureElementImage" in HTMLCanvasElement.prototype`。hover・テキスト選択などDOMの操作が本物どおりに動く
2. polyfill（three-html-render）…本体APIの無い環境でCDNから読む。読めなくても、fallbackで続行する
3. SVG フォールバック…Babylonの`HtmlTexture`内蔵。どんなmodernブラウザでも動く。静的なスナップショットのため、`update()`を呼ぶまで像は更新されない

## 像が出ない三大の原因と、切り分けの方法

デモの初版では像を出せなかった。原因は3つ重なっていた。どれも「像が載っているか」と「見えているか」を混同すると気付けない。テクスチャの画素が正しいのに画面に出ないときは、平面側を疑う。画素が正しくないときは、texture側を疑う。平面側の原因は材質と面の向き、texture側の原因は経路とpaint待ち。この切り分けで、次の節の診断表示を分けてある。

1. planeへのmaterialの割り当てが抜けている。`HtmlTexture` は `planeMat` に像を載せるが、`plane.material = planeMat` を書かないと平面は既定の材質のまま描かれる。像は「載っているのに見えない」状態になる。診断が像の画素の色を読むところまで進んで、初めて分かった
2. emissiveTextureがemissiveColorを使えなくする。像が無い段階から `emissiveTexture` を付けると、`emissiveColor` が使われなくなり平面が真っ黒になる。先に `emissiveColor` で平面の居場所を示しておく。`onLoadObservable` の発火で `emissiveTexture` を紐付けて、`emissiveColor` を黒へ戻す
3. CreatePlaneのfront faceは-Z方向を向く。`lookAt(camera.position)` は+Zをカメラに向けるので、補正なしではカメラから見て背面になり、`backFaceCulling` の既定で平面ごと消える。`lookAt` の第2引数 `Math.PI` でfront faceをカメラへ向ける。`backFaceCulling = false` は像の向きの保険

## 診断はこう作った

「どので止まってるか」を1画面で切り分けられるものとして、左上のパネルに出す。

- 経路表示 … 本体APIの有無とpolyfillの読み込みの可否から、どの前提が揃ってるかが分かる
- paint発火回数 … ネイティブ経路では最初のuploadがpaintイベントを待つ。paintの発火が0回のままなら、paintの待ちで止まってる。発火があっても像が載らないなら、upload以降に原因がある
- 像の画素 … `readPixels` で像の左上の画素を読み、像が載ったか、何色が載ってるかを確かめる

Babylon 9.33.2のネイティブ経路は、uploadが失敗したときにconsoleへ1度だけ警告を出す。そのあとpaintの再試行を待つ。paint待ちのときはpaintの発火が0回のまま。警告も出ない。uploadが弾かれたときは警告が出る。この2つで見分けがつく。

## 実装上のメモ（Babylon 9.33.2 / HTML-in-Canvasでのハマりどころ）

- `content="drawable"`は、本体APIでcanvasに必須の目印。Babylon 9.33.2はこれを付けてくれない。HTML側でcanvasに直接書く。古いブラウザでは未知の属性として無視される
- Babylon 9.33.2のUMDには `HtmlTexture` / `HtmlRaycastInteractionManager` / `InstallHtmlInCanvasPolyfill` が全部入ってる。CDNの配布物で確認済み。デモのCDNはこの版を固定して読む。版を上げるときは、このクラス群の在りかをもう一度確かめる
- `Observable` を直接awaitしない。thenableとは限らないため、Promiseで包んで待つ。selftestのonLoad待ちはこの形。タイムアウトを付けてあるので、永遠に返らないケースも自前に切れる
- DOMパネルのスタイルは全部inlineにする。SVGフォールバックは外部スタイルシートを読まないから、inlineだけがスナップショットに反映される
- DOMの更新を像に反映するのが `update()`。SVGフォールバックでは自動更新されないので、クリックの後に明示的に呼ぶ。native / polyfillでは `autoUpdate` が描画の更新を追うが、呼んでも余計なものは増えない
- ヘッドレスのスクショなどの一時的な検証物は、`.tmp/` に置く。`.gitignore` で管理外にする
