# 遊ぶところ撮影ツール（index.html）

YouTube企画の撮影用WebRTCビデオ通話アプリ。単一HTMLファイル、Firebase RTDBでシグナリング（メッシュ接続、最大6人）。
修正指示は `AI人狼_通話ツール_修正指示メモ.md` を参照。

## RTDB構造

```
rooms/{ROOM_ID}/members/{peerId}: { name, joinedAt, sharing }
rooms/{ROOM_ID}/signals/{toId}/{fromId}/{pushId}: { type: "offer"|"answer"|"candidate", payload }
rooms/{ROOM_ID}/status/{fromId}/{toId}: { state, updatedAt }
rooms/{ROOM_ID}/layoutSync/leaderId: peerId              // レイアウト同期の現在のリーダー（先着優先、runTransactionで排他制御）
rooms/{ROOM_ID}/layoutSync/followers/{peerId}: "accepted" | "rejected"
rooms/{ROOM_ID}/layouts/{leaderId}: { tiles: { [peerIdまたは`${peerId}-screen`]: {left,top,width,z,hidden}(0〜1の相対値) }, updatedAt }
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

## 既知の制約

- 実カメラ・実マイク・画面共有ピッカーはブラウザのネイティブ許可ダイアログを伴うため、ブラウザ自動操作だけでは動作確認が完結しない。コード変更後は実機（複数タブ/複数人）での確認が必要。
