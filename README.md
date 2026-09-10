# Fast QR Reader 1.0.0

ブラウザのカメラ映像からQRを読む、フレームワークに依存しないモジュールです。
`fast-qr-reader.js` 1ファイルにデコーダーを同梱。ビルド作業・実行時CDNアクセスは不要です。

## デモ

`index.html` と `fast-qr-reader.js` を同じフォルダに置き、**HTTPSで配信**して開きます。
「読み取り開始」を押し、カメラを許可してください。ファイルのプレビュー画面では実行できません。

PCだけの確認は、そのフォルダで `python3 -m http.server 8000` を実行し、
`http://localhost:8000/` を開いても構いません。スマートフォンからPCのLAN内HTTPアドレスを
開く方法ではカメラを使えません。スマートフォンにはHTTPSのURLが必要です。

iPhone/iPadではSafariで開き、同じページをホーム画面に追加して利用する想定です。
ホーム画面表示用metaタグを含みます。Service Worker・オフラインキャッシュ・Android用の
インストールmanifestは含めていません。読み込み済みページのQR解析は端末内で完結します。

## 組み込み

```html
<div style="position:relative;width:100%;height:60vh">
  <video id="video" muted playsinline
    style="width:100%;height:100%;object-fit:cover;object-position:50% 50%;border:0;padding:0"></video>
  <div id="target" style="position:absolute;left:50%;top:50%;width:55%;aspect-ratio:1;
    transform:translate(-50%,-50%);border:2px solid lime;pointer-events:none"></div>
</div>
<button id="start">開始</button>
<button id="stop">停止</button>
<output id="result"></output>
<script src="./fast-qr-reader.js"></script>
<script>
  const reader = new FastQRReader({
    video: document.getElementById('video'),
    targetElement: document.getElementById('target'), // 省略時は映像の中央
    onDetected(result) {
      document.getElementById('result').textContent = result.text;
    },
    onError(error) { console.error(error); }
  });
  document.getElementById('start').onclick = async () => {
    try { await reader.start(); } catch (error) {
      if (error.name !== 'AbortError') console.error(error);
    }
  };
  document.getElementById('stop').onclick = () => reader.stop();
</script>
```

| API | 動作 |
|---|---|
| `await reader.start()` | カメラとWorkerを並行起動。起動中の再呼び出しは同じPromise。起動失敗時はreject |
| `reader.stop()` | カメラ・Worker・予約処理を停止。起動途中でも停止可能。重複履歴は維持 |
| `reader.reset()` | 重複履歴を消去。次に読めたQRは再通知。進行中の古いフレーム結果は破棄 |
| `reader.allowDuplicate(true)` | 同じQRが映っていても再通知を許可。既定では250ms以上の間隔 |
| `reader.allowDuplicate(false)` | 同じQRの継続通知を抑制（初期値） |
| `reader.destroy()` | 停止してイベントリスナーも解除。以後は再利用不可 |
| `reader.running / state / decoder / lastResult` | 稼働状態・デコーダー名・最後に通知した結果 |

`onDetected(result)` の主なフィールド：

- `text`：読み取った文字列。HTMLとして挿入せず、表示には `textContent` を使用。
- `decoder`：`BarcodeDetector` または `ZXing WASM`。実際にその回で使用したデコーダー。
- `processingTimeMs`：フレームの切り出し開始からWorkerの結果受信まで。
- `decodeTimeMs`：Worker内の処理時間。フォールバック初回はWASM初期化時間も含む。
- `cornerPoints` / `boundingBox` / `center`：元の映像のピクセル座標。画面上のCSS座標ではありません。
- `region`：`fast` / `wide` / `full` / `detail`。`timestamp`：通知結果作成時のUnix時刻（ms）。

`onScan(stats)` は重複抑制中・未検出時にも呼ばれ、処理時間・decoder・region・候補数などを返します。
`onStateChange(state)` は `starting` / `running` / `paused` / `stopped` を返します。

## 速度と探索の設計

1. 中央枠の周辺を切り出し、長辺最大384pxで即座に探索。成功時に複数フレームの一致を待ちません。
2. 未検出なら次の新しいフレームで、中央の高精細720px → 拡大領域640px → 可視範囲全体960pxを探索。
   小さいQRを広域画像の縮小で見失わないよう、中央の精細探索を先に行います。
3. 中央で読めていても約350msごとに可視範囲全体を探索。周辺に残っている既読QRを確認します。
   全体探索の3回に1回は長辺最大1280pxで、小さい周辺QRも再確認します。
4. 各回で解読できた複数候補を枠中心との距離で並べ、最も近い1件だけを通知対象にします。
   中央の候補が既読でも、代わりに周囲の未読QRを通知することはありません。
5. QR解析はDedicated Worker。縮小・画素取得のみメインスレッドで実施し、画素バッファを転送。
   同時処理は常に1件。`requestVideoFrameCallback` を優先し、未対応時は新規映像フレームだけ処理します。

標準のスキャン開始間隔は65ms以上。重い端末では実際の処理時間に応じて間隔を延ばします。
カメラは背面・1280×720・30fpsを希望値として要求します。実際の解像度・レンズ・AFは端末が決めます。
画面を隠すとカメラを解放し、戻ると再開を試みます。再開できない場合は開始ボタンで再試行してください。

**処理時間の表示は「QRを向けてから反応するまでの時間」ではありません。**
実際の反応にはAF・露光・次フレーム待ちも影響します。初回はカメラ権限とデコーダー起動も必要です。

## 重複判定

同じ文字列ごとに、通知済み状態を保持します。中央ROIの未検出だけでは解除しません。
**全体探索で不在を確認し始めてから700ms以上、かつ2回以上の全体未検出**で解除します。
途中でどの領域でも読み取れたら、不在カウントを取り消します。通常、画面から外して約1秒後に
戻すと同じQRを再通知できます。候補数が上限に達した回は不在の証拠として使いません。

「画面外」は画像からの推定です。長くぼける・隠れる・小さすぎて解読不能になる場合も
不在として扱われ得ます。瞬間的に外して戻す動作は重複抑制が続く場合があります。
同じ内容の別の印刷物は区別しません。必要時は `reset()` または重複許可を使ってください。

主な調整項目（コンストラクタに渡します）：

| オプション | 初期値 | 用途 |
|---|---:|---|
| `scanIntervalMs` | 65 | 最小スキャン間隔。小さくすると負荷増加 |
| `fullScanIntervalMs` | 350 | 全体探索の目安間隔 |
| `fastSize / wideSize / fullSize / detailSize` | 384 / 640 / 960 / 720 | 各解析画像の長辺上限。2048以下 |
| `fullDetailSize` | 1280 | 全体探索3回に1回の精細確認用。2048以下 |
| `leaveDelayMs` | 700 | 不在確認を継続する最短時間 |
| `leaveConfirmations` | 2 | 必要な全体未検出回数 |
| `allowDuplicates` | false | 初期の重複許可 |
| `duplicateIntervalMs` | 250 | 重複許可時の同一内容の通知間隔。0なら各成功回 |
| `decoder` | `'auto'` | `'zxing'` でWASMのみを使用して比較可能 |
| `videoConstraints` | 背面・720p相当・30fps | 指定時はカメラのvideo制約全体を置換。audioは常にfalse |

## デコーダーの比較と採用理由

| 候補 | この用途での特徴 | 判断 |
|---|---|---|
| BarcodeDetector | ブラウザ組み込み・複数候補。ただし利用できるブラウザに制限 | Worker内でQR対応と実際のdetectを確認して優先。エラー時はWASMへ |
| jsQR | JSだけでQRを解読。標準APIは1候補。複数候補の中央順位付けには追加探索が必要 | 今回は同梱せず |
| ZXing-C++ / zxing-wasm | WASM・複数候補・回転等の探索。初期化とバイナリサイズのコストあり | 読み取り専用3.1.3をWorkerに同梱。Safari用の標準フォールバック |

ネイティブ版が正常でも、精細探索で未検出ならWASMを試します。すべての端末で
BarcodeDetectorの方が速いと保証するものではありません。
Safariの対応有無はUAで決めつけず機能検出します。jsQRを含む3方式の実機速度比較は未実施です。

## 運用上の条件

- iOS Safari・ホーム画面Webアプリを優先した実装。iPhone/iPad/Androidの実カメラでの速度・復帰動作は未確認です。
- 極端な斜め、反射、手ぶれ、欠けた余白、解像度不足は読めない場合があります。中央優先は**解読できた候補内**での順位付けです。
- `object-fit: cover / contain / fill` と中央表示を想定し、画面に見えている映像範囲へ座標変換します。
  video自身のCSS回転・反転・border・padding、4値のobject-positionには対応しません。
- WorkerとWebAssemblyが必須。厳しいCSPのページでは `worker-src 'self' blob:` と
  WASM実行許可（対応ブラウザでは `script-src` の `'wasm-unsafe-eval'`）を設定してください。
  Workerが使えない場合はエラーにし、メインスレッドで重いデコードを続ける代替動作はしません。
- iframeに組み込む場合は親側のカメラ権限設定も必要です。複数readerによる同じvideoの共有は不可。
- デモはQRを表示するだけです。撮影保存・録画・画像アップロード・外部アプリ連携はありません。

## 検証済みの範囲

LinuxのChromium 92で、生成画像をCanvasの映像ストリームとして入力し、実際のvideo・
切り出し・Worker・同梱WASMを通して検証しました。大小・回転・遠近変形・軽いぼけ・白黒反転・
周辺配置・複数配置・高密度・日本語・空画像を確認しています。
連続通知抑制、画面から外した後の再通知、重複許可切替、枠位置変更、縦画面、停止／再開、
非表示／復帰、映像フレームコールバック未対応時の処理も確認しました。
BarcodeDetectorの選択・故障時フォールバックはモックによる分岐テストです。
**実カメラのAF・暗所性能、およびiOS Safari／Android実機の速度は未検証です。**

## 一次資料・ライセンス

確認日：2026-09-10。依存コードとライセンス全文は `fast-qr-reader.js` 内に同梱しています。
自作部分は自由に改変・再利用できます。再配布時は同梱第三者コードのライセンス表示を保持してください。

- [BarcodeDetector（MDN）](https://developer.mozilla.org/en-US/docs/Web/API/BarcodeDetector)
- [jsQR公式README](https://github.com/cozmo/jsQR)
- [zxing-wasm公式README・API](https://github.com/Sec-ant/zxing-wasm) — MIT
- [ZXing-C++](https://github.com/zxing-cpp/zxing-cpp) — Apache-2.0
- [getUserMedia（MDN）](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)
- [requestVideoFrameCallback（MDN）](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback)
