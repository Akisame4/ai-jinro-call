# 遊ぶところ撮影ツール（index.html）

YouTube企画の撮影用WebRTCビデオ通話アプリ。単一HTMLファイル、Firebase RTDBでシグナリング（メッシュ接続、最大6人）。
修正指示は `AI人狼_通話ツール_修正指示メモ.md` を参照。

## RTDB構造

```
rooms/{ROOM_ID}/members/{peerId}: { name, joinedAt, sharing, cameraOn, buzzSpectator, buzzTeam }
rooms/{ROOM_ID}/signals/{toId}/{fromId}/{pushId}: { type: "offer"|"answer"|"candidate", payload }
rooms/{ROOM_ID}/status/{fromId}/{toId}: { state, updatedAt }
rooms/{ROOM_ID}/layoutSync/leaderId: peerId              // レイアウト同期の現在のリーダー（先着優先、runTransactionで排他制御）
rooms/{ROOM_ID}/layoutSync/followers/{peerId}: "accepted" | "rejected"
rooms/{ROOM_ID}/layouts/{leaderId}: { tiles: { [peerIdまたは`${peerId}-screen`]: {left,top,width,z}(0〜1の相対値) }, updatedAt }
rooms/{ROOM_ID}/buzzer/config: { enabled, ptsCorrect, ptsWrong, answerSec, winPts, maxWrong, teamMode }
rooms/{ROOM_ID}/buzzer/state: { question, phase, status: "open"|"answering"|"done", answererId, answerDeadline, lockedOut: {entityKey: true}, judge: {id, correct, name} }
rooms/{ROOM_ID}/buzzer/presses/{phase}/{peerId}: { at, recv, name, team }
rooms/{ROOM_ID}/buzzer/scores/{nameKey}: { name, team, points, correct, wrong }
rooms/{ROOM_ID}/buzzer/log/{pushId}: { question, name, correct, order: [{name, diffMs}] }
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

## 一括録画（廃止）

- 2026-09-23にUCの判断で一括録画（各自ローカル録画・同期合図）は削除した。ホスト判定は`isRoomHost()`（最初に入室した人）として早押しで引き続き使用。

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

## 早押し（オプション機能・第一弾）

- ホスト（`isRoomHost()`＝最初に入室した人）が「早押しを使う」をONにすると全員に早押しパネルが出る（`buzzer/config/enabled`）。判定・状態遷移・設定の書き込みはホスト端末だけが行う。
- **公平性**：押下時刻は各自の`Date.now() + serverTimeOffset`（押した瞬間のサーバー時刻換算）で記録し、届いた順ではなくこの時刻順で順位を決める。最初の押下が届いてから`BUZZ_WINDOW_MS`（500ms）は他の人の押下も受け付け、その後ボタンをロックしてホストが回答者を確定する（`hostAssignAnswerer`）。同時刻は`recv`（サーバー受信時刻）→peerIdで決める。
- **phase**：押し直しの単位。誤答で押していた人が残っていないときは`phase+1`で「誤答した人以外」で押し直し。`question`は問題番号で、「次の問題へ」で`phase`とともに進み、`presses`と`lockedOut`を消す。「押し直し」は問題番号を変えずに同じ処理（押下記録・回答権・誤答ロックを消す。得点は変えない）を行う（`hostResetBuzzQuestion(false)`）。受付開始・お手つき判定は仕様で不要とされたため無い（リセット直後から押せる）。
- **誤答**：回答者（チーム戦ならチーム）を`lockedOut`に入れ、着順で次の人へ回答権を移す。
- **チーム戦**：`members/{id}/buzzTeam`（各自が入力、localStorageにも保存）。チーム戦中は同じチームは1枠として扱い、誤答・勝ち抜け・失格もチーム単位。チーム未入力の人は個人扱い。
- **スコア**：再入室でpeerIdが変わっても残るよう**名前キー**（`fbKey(name)`）で保持。チーム合計は各人のteamから集計。ホストは+1/−1で手動修正でき、「スコアを全消去」は2回押しで実行（confirmダイアログは使わない）。
- 回答者が押した後に退室しても、押下記録の名前で判定・得点できる（`buzzNameOf`）。
- 制限時間は表示と時間切れ音のみで、自動で不正解にはしない（判定はホスト）。
- **演出**：回答権が決まると名前のカットイン＋タイルを金色に光らせる＋ピンポーン。正解/不正解は画面中央に○/×＋効果音。効果音は`audioCtx.destination`と`audioDest`の両方へ出し、合成録画キャンバスにも枠・カットイン・○×を描く（`drawBuzzerOverlayOnCanvas`）。入室時点の状態では演出しない。
- 「結果をコピー」に早押しの履歴（問題ごとの判定と着順・時間差）とスコアを出力する。

## 既知の制約

- 実カメラ・実マイク・画面共有ピッカーはブラウザのネイティブ許可ダイアログを伴うため、ブラウザ自動操作だけでは動作確認が完結しない。コード変更後は実機（複数タブ/複数人）での確認が必要。
