# 233-gyoretsu-sabaki-shotengai ｜ 学び

## 2026-09-12 BGM実装（音楽を本番に繋いだ）

### 1. `.m4a` という拡張子は、中身がAACである保証にならない

フォルダに置いてあった `商店街の午後.m4a`（3.2MB）を ffprobe にかけたら **codec は Opus**、
コンテナは ISO/MP4 だった。つまり「Opus in MP4」。iOS Safari が対応しているのは MP4 中の AAC で、
この組み合わせは実機で無音になる危険がある（拡張子だけ見て「m4aだからOK」と判断すると事故る）。

- `ffmpeg -c:a aac_at -b:a 128k -ar 44100 -ac 2 -movflags +faststart` で AAC-LC へ変換
- 3,209,506B（Opus 134kbps）→ 3,054,550B（AAC-LC 128kbps）。長さ 188.7秒は保持
- 変換後 `file` の出力が `Apple iTunes ALAC/AAC-LC (.M4A) Audio` に変わることを確認した
- **教訓**: 音を繋ぐ前に必ず `ffprobe` で codec を見る。231-sky-reign では MP3→AAC だったので気づけなかった穴

### 2. 日本語ファイル名は公開前にASCIIへ直す

`商店街の午後.m4a` のままでは git が `"\345\225\206..."` とエスケープ表示し、
Pages/iOS で URL エンコード事故の余地が残る。`git mv` で `bgm-shotengai.m4a` に改名した。

### 3. `preload='auto'` は「初期ロード3.13MB」という形で跳ね返る

BGM を `preload='auto'` にしていたため、**タイトル画面を見ているだけで 3.2MB 落ちていた**。
harness の「リクエスト失敗1件」も、この先読みがページ終了時に中断された結果だった（404ではない）。

- `preload='none'` に変更 → **初期通信量 3.13MB → 0.07MB**
- 再生はタップ由来の `play()` が引き金になるので、鳴り始めは変わらない（WebKitで実測・後述）
- **教訓**: モバイル回線が主戦場なら、音楽は「起動時に落とす」ではなく「遊び始めたら落とす」

### 4. harness の「タップFAIL」は幕付きタイトル設計に対する誤判定だった

harness は `button, [role=button], canvas` の**DOM順1番目**を叩く。このゲームは `<canvas id="gl">` が
1番目だが、`.sheet{position:absolute;inset:0}` のタイトル幕が全面を覆っている。
`document.elementFromPoint(canvas中心)` で確認したら **タイトル文字の `SPAN`** が返った＝インターセプトで
`locator.tap` がタイムアウトしていた。ゲームの不具合ではない。

- 幕が消えた後に `#gl` を叩けば `OK`（実測済み）
- 検証器は書き換えない（鉄則）。`docs/audit-calibration.md` に誤判定として記録した

### 5. iOS必須要件に穴が2つあった（BGMのついでに塞いだ）

- viewport に `user-scalable=no,maximum-scale=1` が無く、ダブルタップ拡大が生きていた
- `<title>` が `<body>` の中にあった（head へ移設）
- `body` に `touch-action:manipulation` / `-webkit-touch-callout:none` / `user-select:none` を追加

### 実装した音まわりの仕様

- BGM ON/OFF を `localStorage['gyoretsu-bgm']` に保存（再訪時に設定が残る）
- `visibilitychange`: 非表示で一時停止、復帰時に `currentTime>0` なら鳴らし直す（iOSで曲が死ぬのを防ぐ）
- `bgm.setAttribute('playsinline','')` で全画面プレイヤーへ奪われないようにした

### 証跡

- harness: `docs/harness-reports/233-gyoretsu-sabaki-shotengai-2026-09-12T09-21-50-518Z.md`
  → 8項目中7 PASS（コンソールエラー0 / リクエスト失敗0 / 通信量0.07MB / 描画53 RAF秒）。
  残る「タップ」は上記4の誤判定
- WebKit（iOS Safari近似・Playwright）実測: 「▶ はじめる」タップ後に
  `paused=false` / `currentTime 2.01→3.51`（＝実際に再生位置が進んでいる）/ `loop=true` / `volume=0.32` /
  `duration=189秒` / コンソールエラー0 / 失敗リクエスト0 / 幕解除後の canvas タップ OK
- **未検証**: 実機 iPhone での発音そのもの（AAC-LC は iOS ネイティブ対応形式だが、実機確認はしていない）
