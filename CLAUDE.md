# 遊ぶところ撮影ツール（index.html）

YouTube企画の撮影用WebRTCビデオ通話アプリ。単一HTMLファイル、Firebase RTDBでシグナリング（メッシュ接続、最大6人）。
修正指示は `AI人狼_通話ツール_修正指示メモ.md` を参照。

## RTDB構造

```
rooms/{ROOM_ID}/members/{peerId}: { name, joinedAt, sharing, cameraOn }
rooms/{ROOM_ID}/signals/{toId}/{fromId}/{pushId}: { type: "offer"|"answer"|"candidate", payload }
rooms/{ROOM_ID}/status/{fromId}/{toId}: { state, updatedAt }
rooms/{ROOM_ID}/layoutSync/leaderId: peerId              // レイアウト同期の現在のリーダー（先着優先、runTransactionで排他制御）
rooms/{ROOM_ID}/layoutSync/followers/{peerId}: "accepted" | "rejected"
rooms/{ROOM_ID}/layouts/{leaderId}: { tiles: { [peerIdまたは`${peerId}-screen`]: {left,top,width,z}(0〜1の相対値) }, updatedAt }
rooms/{ROOM_ID}/recording: { state: "recording"|"stopped", startedAt, syncMarkRequestedAt? }
rooms/{ROOM_ID}/recordingStatus/{peerId}: { state: "idle"|"ready"|"recording"|"saved"|"error", message?, updatedAt, startedAtMs?, startedAtServerMs? }
```

## 画面共有（カメラと別タイル表示）

- カメラ映像と画面共有映像は別々の `RTCRtpSender`（`addTrack`/`removeTrack`）で送信する。`replaceTrack`は使わない。
- 通話中のトラック追加・削除は再交渉(renegotiation)を伴うため、`onnegotiationneeded` + Perfect Negotiation パターンで衝突を解決する（`peers[id].polite`/`makingOffer`/`ignoreOffer`）。polite側は「後から接続してきた側（`isInitiator=false`）」に固定。
- ストリーム種別（camera/screen）の判別は、**RTDBの別ノードではなく、offer/answerのシグナルpayloadに`streamKinds: { [MediaStream.id]: "camera"|"screen" }`を同梱**して伝える方式にしている（mid はPCペアごとに採番されるため、共有ノードで持つと整合性が壊れるのを避けるため）。受信側は`entry.remoteStreamKinds`にマージして保持し、`pc.ontrack`の`e.streams[0].id`で参照する。
- 画面共有の開始・終了は自分の`members/{myId}/sharing`フラグに反映し、受信側はこのフラグを正として画面共有タイルの表示/削除を同期する（`removeTrack`後の相手側track状態イベントには依存しない）。
- タイルのDOM要素キーは、カメラ＝`peerId`、画面共有＝`` `${peerId}-screen` `` で区別する（`computeSlots()`が返すスロットに`kind: "camera"|"screen"`と`key`を持つ）。

## レイアウト同期（申請・許可制）

- リーダーは`runTransaction`で`layoutSync/leaderId`を排他的に確保する（先着優先。既に誰かがいる間はボタンを無効化）。切断時は`onDisconnect().remove()`で自動的にリーダー権を手放す。
- 各参加者は個別に`layoutSync/followers/{peerId}`へ"accepted"/"rejected"を書き込む。許可した人だけがリーダーの`layouts/{leaderId}`を購読して追従する。
- タイルの識別は表示スロットではなく`peerId`または`` `${peerId}-screen` ``（画面共有タイルも同期対象）。座標は0〜1の相対値。ドラッグ・リサイズ中はリーダー側で100ms間隔にスロットルして`layouts/{leaderId}`へ書き込む（`scheduleLayoutPublish`）。
- 追従中（`followingLayout`が非nullかつ自分がリーダーでない）はこの端末のドラッグ・リサイズ・非表示操作をロックする（`layoutInteractionLocked()`）。
- 追従解除（同期解除ボタン、リーダー消失、リーダーからの拒否状態変化）の際は、直前まで見えていたレイアウトをこの端末のローカル配置（スロット基準）に変換して引き継ぐ（`snapshotLayoutIntoLocal`）。手動解除は`myManualUnfollow`フラグで管理し、「再度追従する」で同じリーダーへ許可を取り直さず復帰できる。

## 一括録画（編集素材用・各自ローカル保存）

- 通話には表示していない画面（プレイ視点等）を、各自のブラウザ内だけで`MediaRecorder`に録画し、`showSaveFilePicker`で選んだファイルへ1秒間隔（timeslice）で逐次書き込む（メモリに溜め込まない）。映像は通話（RTCPeerConnection）には一切送信しない。
- **ホスト＝最も早く入室した参加者**（`isBulkHost()`。新たな役職選定UIを増やさず、既存の`joinedAt`順で決定的に決める）。ホストのみ「一斉録画開始/停止」ボタンが表示される。
- 事前準備は2クリック必須：①`getDisplayMedia`で画面選択→②`showSaveFilePicker`で保存先選択。**この2つを1つの非同期関数内で連続awaitすると、2つ目の呼び出しがユーザー操作起点と認識されず失敗するブラウザがあるため、必ず別々のクリックハンドラに分離している**（`selectBulkRecordScreen` → `chooseBulkRecordSaveDestination`）。
- MP4（`avc1,mp4a.40.2`）を優先し、`MediaRecorder.isTypeSupported()`で非対応の場合のみWebM(vp8,opus)にフォールバック（`pickBulkRecordMimeType`）。
- 空き容量は`navigator.storage.estimate()`で概算表示する（オリジンのストレージクォータであり、`showSaveFilePicker`で選んだ実際の保存先ドライブの空き容量とは正確には一致しない前提の目安表示）。
- 各自の録画開始時刻（ミリ秒, `startedAtMs`＝ローカル時計、`startedAtServerMs`＝`.info/serverTimeOffset`でサーバー時刻に換算した値）は`recordingStatus`に記録し、「結果をコピー」の出力にも含める（編集時のファイル間同期用）。
- **同期合図（フラッシュ＋ビープ）**：録画開始（`recording/startedAt`）の3秒後に、全員の端末で同時に画面全体を200ms白く光らせ（`#syncFlashOverlay`）、1kHzのビープを鳴らす（`fireSyncMark`）。時刻はサーバー時刻基準で各端末が`setTimeout`で予約する（`scheduleSyncMark`）。`serverTimestamp()`は書き込んだ本人の端末で「推定値→確定値」の2回通知されるため、合図は種類（auto/manual）ごとに未発火のものを置き換えて二重発火を防いでいる。ビープは`audioCtx.destination`（スピーカー＝システム音声として一括録画に入る）と`audioDest`（合成録画の音声）の両方に出し、合成録画キャンバスにも白フレームを描く。録画中はホストの「同期合図を出す」ボタンで`syncMarkRequestedAt`を書き込み、その1.5秒後に追加の合図を出せる。1秒以上過ぎた合図は発火しない（途中参加・再読み込み時の誤発火防止）。「結果をコピー」には各自の録画ファイル内での合図の位置（秒）を出力する。
- 準備完了時に空き容量の目安を常に表示し、9GB未満またはWebMフォールバック時は黄色の警告色にする。

## 表示オプション（この端末のみ・localStorage保存）

- **名前表示ON/OFF**：`#videoGrid.hideNames`クラスで`.nameLabel`（各タイル右下）をCSSごと隠す。テキストは常にDOMへ設定しておき、表示/非表示だけを切り替える。
- **グリッドスナップON/OFF**：`gridSnapEnabled`が真の間、ドラッグ・リサイズ中の値を`snapPct()`で`GRID_SNAP_STEP_PCT`（5%）刻みに丸める。
- **背景画像**：選択した画像をlocalStorageにdata URLで保存し、`#videoGrid`の`background-image`に設定（`applyBgImage`）。
- **枠画像（PNG）**：各タイル内の`.frameOverlay`（z-index最上位・`pointer-events:none`）にdata URLを設定して重ねる（`applyFrameImageToTile`/`applyFrameImageToAllTiles`）。表示ON/OFFは`#videoGrid.showFrame`で制御。
- **無音警告のレイアウト固定**：`#gateWarning`は常にDOMに存在させ、`display`ではなく`visibility`（`.show`クラス）で切り替える。これにより表示/非表示で他要素の行がずれない。
- **タイルの重なり順（手動）**：操作テーブルの各行に「⬆最前面へ/⬇最背面へ」ボタンを持つ（画面共有タイルの行も含む）。`bringTileToFront`は現在の全タイルの最大z-indexを都度計算して+1する（固定カウンタ方式だと、スロット番号由来の既定z＝`idx+1`を追い抜けないバグがあったため修正済み）。`sendTileKeyToBack`は他タイルの最小z-index-1を設定する。

## カメラオフ＝タイル非表示

- 「表示/非表示」ボタンは廃止。カメラがオフの参加者はタイルごと非表示にし、真っ黒な枠を出さない（`applyCameraVisibility`、`renderGrid`の最後で毎回適用）。音声は再生し続ける。
- 自分のカメラON/OFFは`members/{myId}/cameraOn`に書き込み、全員の画面で同じタイルが消える。相手の行の「📷 オフ」はこの端末だけでそのタイルを隠す（`locallyHiddenVideoPeers`）。
- レイアウト同期のデータから`hidden`は削除した（表示/非表示は各端末がカメラ状態から決める）。

## 既知の制約

- 実カメラ・実マイク・画面共有ピッカーはブラウザのネイティブ許可ダイアログを伴うため、ブラウザ自動操作だけでは動作確認が完結しない。コード変更後は実機（複数タブ/複数人）での確認が必要。
- `showSaveFilePicker`（File System Access API）はChrome/Edge系のみ対応。Firefox/Safariでは一括録画機能が使えない（非対応時はメッセージを表示するのみ）。
- MP4出力ファイルをDaVinci Resolve等で読み込んだ際のシーク・音ズレ、システム音声キャプチャでゲーム音を含められるかは未検証（メモ記載の「要検証」項目のまま）。
