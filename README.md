# html-in-canvas-playground

Chrome 157 で使えるようになった HTML-in-Canvas API（DOM を canvas のテクスチャに描ける）を、
Babylon.js の `HtmlTexture` で試すデモです。DOM で作ったパネルを 3D の平面に貼り、パネル上の
ボタンを 3D の平面の上から押せます。

## 開き方

ローカルサーバーで開きます。

```
python3 -m http.server 8080
```

<http://localhost:8080/>

動く経路は 3 つあり、開いたときの「経路」表示でどれが使われたかが分かります。

1. **native（本体 API）** … Chrome 157。DOM の操作（hover、テキスト選択など）が本物どおりに 3D で動く
2. **polyfill（three-html-render）** … 本体 API の無い環境。CDN からポリフィルを読む
3. **SVG フォールバック** … Babylon.js 内蔵。静的なスナップショットで、DOM の更新は
   `update()` を通じて像に反映される

Babylon.js は v9.33.2 を固定で CDN から読みます。`HtmlTexture` とインタラクション管理の
クラス（`HtmlRaycastInteractionManager`）がこの版に含まれているためです。

## 操作

| 操作 | 動き |
| --- | --- |
| 3D 平面上の「+1」 | DOM のカウンタが増え、像に反映される |
| 3D 平面上の「色を変える」 | DOM パネルの背景色が変わり、像に反映される |
| 右ドラッグ | パン（カメラ移動） |
| ホイール | ズーム |

## 確認のしかた

`?selftest` を付けると、テクスチャに載った像を console で確認できます。

```
[hic] route=native（本体 API） nativeBefore=true        ← どの経路で動いたか
[hic] 初回 画素[0] RGBA=17,26,51,255                     ← DOM が像に貼れている（背景色と一致）
[hic] update 後 画素[0] RGBA=35,43,71,255                ← DOM の更新が像に反映されている
[hic] selftest DONE count=1 tint=1
```

初回の画素がパネルの背景色 `#111a33`（17, 26, 51）と一致することが「DOM が像に載っている」の
確認、update 後に画素が変わることが「DOM の更新が像に写る」の確認になります。

GPU の無い環境（Linux など）での headless 実行の手順は、AGENTS.md の「起動と確認」にあります。
