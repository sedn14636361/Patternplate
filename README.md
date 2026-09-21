# PATTERN PLATE

ベクターパターンとアブストラクトなグラデーションをつくって書き出すツールです。
**ビルド工程はありません。** `patternplate.html` をブラウザで直接開けばそのまま動きます。
npm も bundler も使いません。

👉 **[sedn14636361.github.io/Patternplate](https://sedn14636361.github.io/Patternplate/)**

## できること

- **図案モード** — 継ぎ目のない繰り返しパターン
- **グラデーションモード** — 全11種
  - 図形を並べる8種: ブロブ / 煙 / 波 / メッシュ / オーロラ / 線形 / 放射 / 円錐
  - 画素で解く3種（場）: 流体 / 空 / 薄明
- 8つのムードからの自動配色、刷り色ごとの色名とコントラスト比の表示
- 網点・質感フィルタ・グリッチの重ねがけ
- アニメーション（平行移動 / 立ちのぼる / 流れる / 波 / 回転 / 浮遊）
- 書き出し: SVG / PNG / APNG / CSS

## ファイル

| ファイル | 役割 |
|---|---|
| `patternplate.html` | **本体。これが正。** 3202行の単一HTML |
| `pp_artifact.html` | `<!DOCTYPE>`〜`<body>` を剥がした公開用の断片。**生成物なので手で編集しない** |
| `tools/frag.py` | 断片を生成する |
| `tools/syntax_check.sh` | 本体の構文チェック |

`index.html` はリポジトリに置いていません。GitHub Pages へ上げるときに workflow が
`patternplate.html` を複製して作ります。本体のファイル名を唯一の正に保つためです。

## 編集の流れ

巨大な1ファイルなので、アンカー文字列で置換する形で編集し、**置換前に出現回数を必ず確かめて**ください。
似た行を巻き添えで書き換える事故が防げます。

```bash
tools/syntax_check.sh     # 1. 構文チェック（編集後に必ず）
python3 tools/frag.py     # 2. 断片を作り直す
```

この2つは `main` への push 時に CI でも回ります。断片が本体と食い違っていると失敗します。

## 破ってはいけない約束

**すべて実測で痛い目を見た結果です。理由なく変えないでください。**

1. **`gradientArt()` は同期関数のまま**にする。書き出しが「SVG文字列 → Image → canvas」の同期的な組み立てに依存しています。場（field）も canvas の同期APIだけで済ませてあります。

2. **変形アニメに `additive="sum"` を使わない。入れ子の `<g>` で重ねる。** 拡大縮小のように中心を保つ変形が他と噛み合いません。`animStages()` / `wrapTransforms()` がその実装で、ライブ再生と焼き込みのCTMが6通りの組み合わせで一致することを確認済みです。

3. **ライブ再生と焼き込みは同じ式を使う。** 共有点は `triWave()` と `sineVals()`。SMILの `values="0;peak;0"` と `triWave` は同じ形になります。片方だけ直すとプレビューと書き出しがずれます。

4. **ループは必ず閉じる。** 平行移動・立ちのぼる・流れるは `driftLayers()` が半周期ずらした2枚をレイズドコサインで交差させます（不透明度の和が常に1）。波は波長を描画幅のちょうど整数分の1にします。場はノイズを読む位置を閉じた楕円で一周させます。検証は `t=0` と `t=1` の画像が完全一致すること。

5. **UPNG は `blend=0`（SOURCE）を必ず書く**ようにパッチ済み（`UPNG.encode` 内、`blend = 0;` の直前に経緯のコメントあり）。`blend_op=OVER` のAPNGは読み手によって結果が割れます（Pillow がα43を43²/255と解釈しました）。ファイルも小さくなります。

6. **減色は最後の手段。** 5MBに収めるときは、まず可逆のまま縮小し（下限 `LOSSLESS_FLOOR=0.6`）、そこまでで収まらないときだけ256色に落とします。落とす直前に 4×4 Bayer ディザ（`DITHER_AMP=5`）を必ず掛けます。掛けないと帯が出ます（ブロブ 5%→47.7%、円錐 11.4%→70.5%。ディザ後は 2.3% / 7.5%）。

7. **縦横比は相乗平均で扱う。** `GM=Math.sqrt(W*H)`, `ax=W/GM`, `ay=H/GM`。短い辺を基準にしていたころは横長にすると地色が見えていました（21:9のブロブで22.6%）。回転する種類は逆回転の外接矩形 `RW/RH/RX/RY` を使います。現在は全種類・全比率で 0〜0.3%。

8. **網点は変形の外側に置く。** `body = wrapTransforms(...) + ht`。アニメーションは下地にだけ掛かり、点の格子は動きません（網戸ごしに景色が動く見え方）。点の上限は `MAXD=26000`。

9. **`vnoise` のハッシュは `Math.imul` で書く。** 素直に掛けると32ビットが溢れて下位が落ち、値が0.5より上に出ません（平均0.255・上限0.49になっていました）。現在は平均0.500。

10. **はみ出しの割合 `0.4` は2か所にある。** `gradientArt()` の `const O=.4`（L311付近）と `fieldLiveStart()` が `fieldURI()` へ渡す `.4`（L1074付近）です。**片方だけ変えるとライブ再生で絵が飛びます。** 直すときは必ず両方。

11. **UIを切ってあるときは `disabled` を立てる。** `pointer-events:none` だけだとキーボードで到達して操作できてしまいます。

## 描画経路は2本ある

```
(A) 図形を並べる種類  blob smoke wave mesh aurora linear radial conic
    → gradientArt() が <ellipse>/<path>/<rect> を文字列で組み立てる

(B) 画素で解く種類    flow sky veil          ← FIELD_TYPES / isField()
    → fieldURI() が canvas に1画素ずつ描き、PNG の data URI にして
       <image data-field="1"> として SVG に貼る
```

**(B) でも、そのあとの流れ（変形の `<g>`・ぼかしのフィルタ・網点・書き出し・5MB調整）は
(A) と完全に共通です。ここを分岐させないことが設計の要です。**

場（field）まわりの要点:

| 記号 | 意味 |
|---|---|
| `FIELD_PX_BASE = 440` | 画面用ラスタの長辺。書き出し時だけ `FIELD_PX` を差し替える |
| `fieldPxFor(long, cap)` | `long*1.8` を cap で頭打ち。1.8倍は上下左右40%のはみ出しを含むため |
| 静止画書き出し | cap 1600（`exportSvg()` が一括して面倒を見る） |
| APNG書き出し | cap 560（実測1コマ418ms。760にすると732msになり120コマで1分超） |
| `fieldCache` | 場の**値だけ**を持つ Map（最大8件のLRU）。色や色の位置だけの変更なら再計算しない（154ms → 49ms） |
| `fieldLiveStart()` | 流れる/立ちのぼるのライブ再生。SMILでは中身を動かせないので240pxで12コマ作って `href` を貼り替える |

## 検証のやり方

**推測でなく数値で確かめてください。** JSの例外は画面上は無言で消えるので、`pageerror` を必ず拾うこと。

```python
from playwright.sync_api import sync_playwright
with sync_playwright() as pw:
    b = pw.chromium.launch()
    pg = b.new_page(viewport={'width':1500,'height':1000})
    errs = []
    pg.on('pageerror', lambda e: errs.append(str(e)))
    pg.goto('file:///.../patternplate.html'); pg.wait_for_timeout(700)
    pg.evaluate("() => { setMode('gradient'); S.gtype='flow'; render(); }")
    pg.locator('#plate').screenshot(path='out.png')
    print(errs)
    b.close()
```

よく測っている指標: 帯の面積% / 地色の露出%（許容±6）/ ループ継ぎ目の段差倍率 /
場の標準偏差と5分位ヒストグラム / ライブ再生と焼き込みの `getScreenCTM()` の一致。

**退行チェック**（既存種類のSVG出力がバイト単位で変わっていないこと）は毎回やる価値があります。

```python
JS = """() => {
  const out={}; setMode('gradient');
  for (const t of ['blob','smoke','wave','mesh','aurora','linear','radial','conic']) {
    S.gtype=t; S.ratio=[16,9]; const d=dims(1000);
    out[t]=svgMarkup(S,d.w,d.h,{standalone:true,textures:true}).replace(/\s+/g,'');
  }
  return out;
}"""
```

## 過去に踏んだ罠

| 症状 | 原因 | 直し方 |
|---|---|---|
| グラデーションが等高線状の帯になる | 256色量子化 | 量子化直前に Bayer ディザ |
| APNGの半透明が他ツールで濃く出る | `blend_op=OVER` | UPNGを `blend=0` に |
| 平行移動に切れ目（段差倍率26.49） | 端で折り返していた | 半周期ずらした2枚のクロスフェード（0.67まで改善） |
| 煙がただの淡い染みになる | ノイズの後にぼかしていた | 煙だけ「先にぼかす→ノイズで裂く」 |
| うねり網点が真っ黒に潰れる | コントラスト1.9 | 1.35に |
| 横長で地色が見える | 短辺基準のサイズ決め | `GM/ax/ay` ＋ 伸びた分だけ塊を追加 |
| `O` is not defined | `RW/RH` の計算を `const O=.4` より前に挿入した | 順序を戻す |
| `stops` が上書きされる | `gradientArt` 内のローカル関数 `stops` と名前衝突 | 変数名を `cstops` に |
| コントラスト等級が全部AAA / 全部「—」 | 白地だけ / 白黒の良い方で判定していた | いま選んでいる地色との比で出す |
| 場（flow）がほぼ単色（sd 0.04） | fbmを重ねると平均に寄る | `(vv-.5)*5.2+.30` で伸ばし、ridge項 `.28` で筋を足す（sd 0.242） |

## 既知の制約

- 場の3種（流体・空・薄明）の **SVG書き出しは中身が画像1枚**（約2.7MB）。拡大しても線はなめらかになりません。書き出し時にトーストで伝えています。
- 場の「流れる／立ちのぼる」のプレビューは240px・12コマの簡易再生。書き出しは出力解像度で計算し直します。
- APNGの書き出しは120コマで1分前後かかります。

## 保留・未着手（勝手に復活させないこと）

一度検討して「やらない」「後回し」と決めたものです。やるなら改めて確認を取ってください。

- 色だまりをドラッグで動かす（現在はSEEDからの自動配置のみ）
- 色の位置のドラッグ移動（スライダーまでで止めている）
- 円錐グラデーションの分割の継ぎ目を増やす
- 網点の濃度を「下地をラスタライズして読む」方式にする（いまは式で直接求めている＝同期で済む）
- Silk / Mesh / Still / Retro / iOS を別々の描画にする（いまは flow/sky/veil の3本のみ）
- Text / Image タブ
- SVG（真のベクター）/ MP4 書き出し
- Figma 向けの名前つきSVGレイヤー

## 同梱しているライブラリ

どちらも本体HTMLの中に貼り込んであります（`<script>` の1本目と2本目）。**別ファイルとしては読み込みません。**

| ライブラリ | 版 | ライセンス | 上流 |
|---|---|---|---|
| pako | 3.0.1 | MIT / Zlib | https://github.com/nodeca/pako |
| UPNG.js | — | MIT | https://github.com/photopea/UPNG.js |

UPNG.js には上記「約束5」の `blend=0` パッチを当ててあります。**上流版とは異なります。**
MITライセンス全文はまだ同梱していません（ヘッダコメントに表示とURLのみ）。
MITは全文の同梱を求めるので、`THIRD-PARTY-NOTICES.md` を置くのが本来です。
