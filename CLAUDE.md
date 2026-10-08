# 遊ぶところ撮影ツール（index.html）

YouTube企画の撮影用WebRTCビデオ通話アプリ。単一HTMLファイル、Firebase RTDBでシグナリング（メッシュ接続、最大6人）。
修正指示は `AI人狼_通話ツール_修正指示メモ.md` を参照。

## 画面の構成（2026-10-09 UC要望で組み直し）

機能を足すたびに縦に積んでいてわかりにくかったため、配信・収録ツール（StreamYard・OBS・Riverside）の形を参考に組み直した。UCとUI案（claude.aiのDesignキャンバス「撮影ツール UI案」）で形を決めてから実装。
- **上のバー（`#topBar`）**：今の状態だけ。ルームと人数（`#roomChip`）、録画中のチップ（`#recIndicator`）、全画面・別ウィンドウのボタン。その下にレイアウト同期の通知（`#layoutSyncPanel`。許可/拒否ボタンがあるので帯のまま）。
- **真ん中（`#stageArea`）**：配置エリア。16:9のまま画面の高さに収まる幅にする（`width: min(100%, calc((100vh - 上 - 下 - 余白) * 16/9))`）。`#callScreen`は`display:flex`の縦並び・高さ100vh（入室時に`callScreen.style.display = "flex"`）。
- **右のパネル（`#sidePanel`）**：タブ「参加者／早押し／サイコロ／配置／背景・枠／録画」（`.sideTab[data-tab]`→`.sidePane[data-pane]`、`showSideTab`、最後のタブはlocalStorage `sideTab`）。
  - 参加者：1人＝1枚のカード（`.pRow`、ドラッグで並び替えは従来どおり`swapOrder`）。マイク・カメラ・聞こえる音量だけ出し、枠ON/OFF・重なり順・ズーム・反転・共有の終了は「…」（`.pMore`、開いているものは`openMoreKeys`で再描画後も開いたまま）。
  - 早押し：ルール設定とイントロクイズは`<details class="fold">`で普段は閉じる（`#introRow`はdetails自体）。スコアボードのボタンもここ。
  - 配置：テンプレート・レイアウト同期・マス目・「配置エリアに出すもの」（名前・演出・バッジのON/OFF）。背景・枠：背景と共通の枠・自分の枠をプレビュー付きで（`updateBgPreview`/`updateFramePreviews`、`applyBgImage`/`applyFrameImageToAllTiles`から更新）。録画：録るもの・システム音声・保存したファイル・結果をコピー。
- **下のバー（`#controlBar`）**：自分の操作。マイク（音量メーター付き、`barMicMeter`）、ノイズ抑制（押すと「音声の設定」の小窓。中にノイズ抑制のON/OFF・強さ・メーター・EQ。表示は`updateAudioSettingsBtn`、無音警告が出たら赤くなる）、カメラ、画面共有（▴で画質・画面の音の小窓）、全体ミュート、早押しボタン（`#buzzBtn`をここに移した。早押しを使う間だけ表示）、サイコロ（`#barDiceBtn`、「振るダイス」の指定で振る）、録画開始/停止（`#recBtn`）。下のバーのマイク・カメラは`toggleMicFor(myId)`/`toggleCamFor(myId)`、表示は`updateControlBar`（`renderControls`の最後）。
- 小窓（`.popover`）は外側クリック・Escで閉じる。画面の音が取れなかった注意（`showScreenAudioNote`）は画面共有の小窓を開いて見せる。
- 要素のidは組み直し前と同じにしてあるので、各機能の処理はそのまま動く（`videoGridBar`・`displayOptionsBar`だけ廃止）。

## RTDB構造

```
rooms/{ROOM_ID}/members/{peerId}: { name, joinedAt, sharing, screens, cameraOn, buzzSpectator, buzzTeam }
rooms/{ROOM_ID}/signals/{toId}/{fromId}/{pushId}: { type: "offer"|"answer"|"candidate", payload }
rooms/{ROOM_ID}/status/{fromId}/{toId}: { state, updatedAt }
rooms/{ROOM_ID}/layoutSync/leaderId: peerId              // レイアウト同期の現在のリーダー（先着優先、runTransactionで排他制御）
rooms/{ROOM_ID}/layoutSync/followers/{peerId}: "accepted" | "rejected"
rooms/{ROOM_ID}/layoutBg/{leaderId}: { dataUrl|null, updatedAt }   // リーダーの背景画像（JPEG・最大1920×1080に縮小）
rooms/{ROOM_ID}/layouts/{leaderId}: { tiles: { [peerIdまたは`${peerId}-screen`]: {left,top,width,z}(0〜1の相対値) }, updatedAt }
rooms/{ROOM_ID}/memberFrames/{peerId}: { dataUrl, updatedAt }   // 各自の「自分の枠」（PNG・最大1920×1080）
rooms/{ROOM_ID}/scoreboard: { on }   // スコアボードタイルを全員に表示するか（誰でもON/OFF可）
rooms/{ROOM_ID}/dice/config: { enabled }   // サイコロ機能を使うか（誰でもON/OFF可）
rooms/{ROOM_ID}/dice/log/{pushId}: { name, n, m, values: [...], total, at }
rooms/{ROOM_ID}/buzzer/host: { name, at }
rooms/{ROOM_ID}/buzzer/music: { playing }   // イントロクイズの曲が流れているか（曲名は送らない）
rooms/{ROOM_ID}/buzzer/config: { enabled, ptsCorrect, ptsWrong, answerSec, winPts, maxWrong, teamMode }
rooms/{ROOM_ID}/buzzer/state: { question, phase, status: "open"|"answering"|"done", answererId, answerDeadline, lockedOut: {entityKey: true}, judge: {id, correct, name} }
rooms/{ROOM_ID}/buzzer/presses/{phase}/{peerId}: { at, recv, name, team }
rooms/{ROOM_ID}/buzzer/scores/{nameKey}: { name, team, points, correct, wrong }
rooms/{ROOM_ID}/buzzer/log/{pushId}: { question, name, correct, order: [{name, diffMs}] }
```

## 画面共有（カメラと別タイル表示）

- カメラ映像と画面共有映像は別々の `RTCRtpSender`（`addTrack`/`removeTrack`）で送信する。`replaceTrack`は使わない。
- 通話中のトラック追加・削除は再交渉(renegotiation)を伴うため、`onnegotiationneeded` + Perfect Negotiation パターンで衝突を解決する（`peers[id].polite`/`makingOffer`/`ignoreOffer`）。polite側は「後から接続してきた側（`isInitiator=false`）」に固定。
- **シグナルは相手ごとに届いた順で1つずつ処理する**（`signalChains[fromId]`のPromiseの鎖、2026-09-29修正）。以前は`handleSignal`を待たずに並行処理しており、共有中に途中入室した人に「answer」と「画面共有を足すoffer」が続けて（Firebaseの1回の更新で）届くと、answerの`setRemoteDescription`中にofferが衝突扱い（impolite側なので無視）で捨てられた。共有側は`have-local-offer`のまま止まり（接続状態はconnectedのまま）、その人にだけ共有画面が届かなかった（「しのだけ毎回見えない」の原因）。モックに受信のまとめ届け（`?lat=`）を入れて再現・修正を確認済み。
- 保険として、再交渉のofferに10秒返事が無いときは同じofferを送り直す（`updateStats`内、`entry.offerSentAt`）。相手が旧バージョンでofferを捨てた場合もこれで回復する（約10〜15秒後に共有画面が届く）。
- ストリーム種別（camera/screen）の判別は、**RTDBの別ノードではなく、offer/answerのシグナルpayloadに`streamKinds: { [MediaStream.id]: "camera"|"screen" }`を同梱**して伝える方式にしている（mid はPCペアごとに採番されるため、共有ノードで持つと整合性が壊れるのを避けるため）。受信側は`entry.remoteStreamKinds`にマージして保持し、`pc.ontrack`の`e.streams[0].id`で参照する。
- **画面共有の画質**：ヘッダーの選択（`SCREEN_QUALITY_PRESETS`、localStorage `screenQuality`）で、文字くっきり（contentHint=detail・15fps・2.5Mbps・maintain-resolution）／動き優先（motion・30fps・4Mbps・maintain-framerate）／高画質（detail・30fps・6Mbps）。共有中に変えても`applyScreenQualityToTrack`と`applyScreenSenderParams`で即反映。メッシュなので送信量は「ビットレート×相手の人数」。
- 送信側の`maxBitrate`等は接続確立前だと`encodings`が空で設定できないため、`updateStats`（2秒ごと）で毎回かけ直す（値が同じなら何もしない）。
- **1人で複数の画面を共有できる（最大`MAX_SCREENS_PER_PERSON`=4、2026-09-27〜）**。「🖥 画面共有」ボタンを押すたびに1画面追加（`addScreenShare`）。各共有に空いている最小の番号nを振り（`myScreens`: n→MediaStream）、相手ごとの送信は`entry.screenSenders[n]`。個別の終了は操作パネルの自分の画面共有行の「■ 終了」かブラウザの「共有を停止」（`stopScreenShare(n)`）。
- 画面番号は`members/{id}/screens`（`{ s1: true, s3: true }`。数字キーだとRTDBが配列にするので`s`付き）に書き、`sharing`は「1つ以上共有中か」として旧バージョン互換で残す（`publishMyScreens`）。`screens`が無い（旧バージョンの）相手は`sharing`だけで画面1つとみなす（`memberScreenNums`）。offer/answerに`streamKinds`と並べて`screenNums: { [MediaStream.id]: n }`も同梱し、受信側は`entry.remoteScreenNums`で番号を引く（無ければ1）。
- **画面の音も共有できる**（UC要望、2026-10-04）：ヘッダーの「画面の音も共有」（localStorage `screenShareAudio`）がONなら、`getDisplayMedia`に`audio`（エコー除去・ノイズ抑制・自動音量はOFF、`restrictOwnAudio: true`で画面全体の音からこのページ自身の音＝相手の声を除く（対応ブラウザのみ））と`systemAudio: "include"`を付ける。設定は共有を始めるときに読むので、共有中に変えても次の共有から。ウィンドウ共有などで音が取れなかったときはチェックボックスの横に注意を10秒出す（`showScreenAudioNote`）。
  - 音声トラックは画面の映像と**同じMediaStream**で`addTrack`し（`entry.screenAudioSenders[n]`、128kbps `SCREEN_AUDIO_BITRATE`）、相手はそのストリームを画面共有タイルの`<video>`でそのまま鳴らす。`ontrack`はトラックごとに来るので、audioのときに`connectAudioSource(screenKey)`（レイアウト録画の全員の音声へ）と操作パネルの再描画を行う。
  - 自分の画面共有タイルの`<video>`は常にミュート（自分のPCで既に鳴っているため）。別ウィンドウ表示の`syncPopAudio`・`popInGrid`も自分の画面は鳴らさない。自分の画面の音はレイアウト録画には入れる。
  - 操作パネルの画面共有行：相手の音ありの画面は「🔊 ミュート」（この端末だけ、`locallyMutedPeers`に画面のキー）と「聞こえる音量」（`peerVolumes`のキーは「名前（画面共有n）」、`volumeNameOf`）。自分の画面は「🔊 音あり」表示。全体ミュートは画面の音にも効く。
- **画面共有タイルは共有元の縦横比で表示する**（UC要望、2026-09-28。16:9以外のウィンドウや縦長の画面も映すため）：`.tile.screenTile`の`<video>`に、`loadedmetadata`/`resize`イベントで`videoWidth/videoHeight`の`aspect-ratio`をインラインで設定する（`applyScreenAspect`、比率は`tile.dataset.aspect`。1%未満の揺れは無視）。配置データは従来どおりleft/top/widthだけで、高さは各端末が映像の比率から決める（レイアウト同期・テンプレートの形式は変更なし）。配置エリアの下にはみ出すときは上へずらし、それでも入らない縦長は幅を縮める（`fitTileInGrid`、追従中は何もしない）。リサイズハンドルも下にはみ出さない幅までに制限。合成録画はタイルの矩形に映像を描くのでそのまま同じ比率になる。カメラ・スコアボードは16:9のまま。
- 画面共有の開始・終了は自分の`members/{myId}/sharing`/`screens`に反映し、受信側はこのフラグを正として画面共有タイルの表示/削除を同期する（`removeTrack`後の相手側track状態イベントには依存しない）。
- タイルのDOM要素キーは、カメラ＝`peerId`、画面共有＝1つ目`` `${peerId}-screen` ``・2つ目以降`` `${peerId}-screen${n}` ``（`screenKey`。1つ目は旧キーのままなので保存済みの配置が効く）で区別する（`computeSlots()`が返すスロットに`kind: "camera"|"screen"`と`key`を持つ）。

## レイアウト同期（申請・許可制）

- リーダーは`runTransaction`で`layoutSync/leaderId`を排他的に確保する（先着優先。既に誰かがいる間はボタンを無効化）。切断時は`onDisconnect().remove()`で自動的にリーダー権を手放す。
- 各参加者は個別に`layoutSync/followers/{peerId}`へ"accepted"/"rejected"を書き込む。許可した人だけがリーダーの`layouts/{leaderId}`を購読して追従する。
- タイルの識別は表示スロットではなく`peerId`または`screenKey(peerId, n)`（画面共有タイルも同期対象）。座標は0〜1の相対値。ドラッグ・リサイズ中はリーダー側で100ms間隔にスロットルして`layouts/{leaderId}`へ書き込む（`scheduleLayoutPublish`）。
- 追従中（`followingLayout`が非nullかつ自分がリーダーでない）はこの端末のドラッグ・リサイズ・非表示操作をロックする（`layoutInteractionLocked()`）。
- **背景画像も同期**：リーダーは同期開始時と背景の変更・削除時に、自分の背景を最大1920×1080のJPEGに縮めて`layoutBg/{leaderId}`へ書く（`publishLeaderBg`。位置の同期ノードとは分けて、大きな画像を毎回送らない）。追従中の端末は`followingBg`としてそれを表示し（リーダーが背景なしなら背景なし）、追従をやめると自分の背景（localStorage）に戻す（`applyBgImage`）。合成録画にも背景画像を描く（cover相当）。
- 追従解除（同期解除ボタン、リーダー消失、リーダーからの拒否状態変化）の際は、直前まで見えていたレイアウトをこの端末のローカル配置（スロット基準）に変換して引き継ぐ（`snapshotLayoutIntoLocal`）。手動解除は`myManualUnfollow`フラグで管理し、「再度追従する」で同じリーダーへ許可を取り直さず復帰できる。

## イントロクイズ（早押しの追加機能）

- ホストの操作欄の「🎵 イントロクイズ」で、この端末の音声ファイルを選んで再生する。曲は**通話の送信音声に混ぜて**全員に流す（`introBus`）。混ぜる位置はノイズ抑制・ゲート・EQの**後段**（`setupNoiseGate`内の`connectIntroBusToSend`）。RNNoise/ゲートを通すと音楽が削られるため。`introBus`はスピーカー（本人のモニター）と`audioDest`（合成録画）にも繋ぐ。ノイズ抑制の初期化に失敗して生マイクにフォールバックした場合は曲が送られない。
- 再生は`<audio>`ではなく、通話で既に動いているAudioContextの`AudioBufferSourceNode`で行う（`decodeAudioData`）。`<audio>.play()`はブラウザの自動再生制限で、クリック直後以外（誤答後の自動再開など）の再生が拒否されることがあるため。一時停止位置は`introOffset`で自前管理。
- **早押しで自動停止**：その問題（phase）で最初の押下が届いた瞬間（`announceBuzzPress`→`introAutoStopOnPress`）に、曲を流している端末で一時停止する。ホスト判定に関係なく「流している端末」で止まるので、途中でホストが交代しても止まる。
- **不正解なら続きから再生**：早押しで自動停止した場合だけ（`introPausedByBuzz`）、×の演出後に再開する。手動の一時停止では再開しない。
- 正解（または「次の問題へ」）で次の曲を頭出し（再生はホストが▶）。「押し直し」では曲はそのまま。
- 曲名は流している端末にだけ表示する。他の人には`buzzer/music/playing`で「♪ 曲が流れています」とだけ出す。

## 配置エリアの別ウィンドウ表示（配信用）

- 「🗗 配置エリアを別ウィンドウで表示」で`#videoGrid`の要素そのものを`window.open`したウィンドウへ移す（配置・枠・背景・名前表示・ドラッグ操作がそのまま使える）。メイン画面には`#gridPlaceholder`（元に戻すボタン）を出す。ウィンドウを閉じる（pagehide）か「元に戻す」で戻す。
- 別ウィンドウに移すとタイル関連要素はそちらの文書に入るため、タイル・映像要素の取得はすべて`gridEl(id)`（両方の文書を探す）を使うこと。`document.getElementById`でタイルを取らない。
- 音声：別ウィンドウでは音声付き再生が自動再生制限で止められることがあるため、移している間はタイルの`<video>`をミュートにし、同じストリームをメイン画面の`#popAudioContainer`内の`<audio>`（`popAudio-<videoのid>`）で鳴らす（`syncPopAudio`）。ミュート・音量は`peerAudioEl(id)`経由で、表示中の方の要素に効かせる（`applyPeerMute`/`applyPeerVolume`）。
- 要素を別文書へ移すと`<video>`は一時停止するので`resumeGridVideos()`で再生し直す。合成録画の描画ループ（rAF）は配置エリアを表示しているウィンドウのものを使う（`recordLoopWin`）。スペースキーの早押しは別ウィンドウにも登録（`onBuzzHotkey`）。
- ボタンのクリックから開くのでポップアップはブロックされない想定。ブロックされた場合はテンプレート欄にメッセージを出す。

## スコアボードタイル

- ヘッダーの「📊 スコアボード」で`rooms/{ROOM_ID}/scoreboard/on`を切り替え、全員の配置エリアにスコアボードをタイルとして出す（画面共有と同じ扱い）。
- 各端末がRTDBの早押しスコアから1280×720のcanvasに描き（`drawScoreboard`、`renderBuzzer`のたびに再描画）、`canvas.captureStream()`をタイルの`<video>`に流す。既存の映像タイルと同じ仕組みなので、移動・サイズ変更・重なり順・レイアウト同期・合成録画にそのまま乗る。タイルのキーは`scoreboard`（`SCOREBOARD_KEY`）、スロットは`kind: "scoreboard"`。
- カメラ枠（MAX_PEERS個・2段）より後ろのスロット（画面共有・スコアボード）の既定位置は、3段目だとエリア外に出てしまうため、エリア中央付近に少しずつずらして重ねる（`defaultPos`）。

## 録画（同時録画）

- 録画対象はチェックボックスで複数選べる（`recTargetGrid`/`recTargetSelf`/`recTargetScreen`、localStorageに保存）。「録画開始」でチェックしたものを**同時に**開始し（`activeRecorders`）、停止するとそれぞれ別ファイルになる。ファイル名はUC指定のルール（2026-09-27）で`YYYYMMDD_all_HH-MM.mp4`（レイアウト全体）／`YYYYMMDD_{名前}_HH-MM.mp4`（自分のカメラ）／`YYYYMMDD_{名前}(PC画面)_HH-MM.mp4`（自分のPC画面）。時刻は録画終了時刻（ローカル、`stopRecording`で`recordingEndedAt`に記録し全ファイル共通）。UCの指定は「00:00」だがWindowsは「:」不可なので「-」。名前のファイル名禁止文字は「_」に置換（`recordingFileName`）。
  - レイアウト全体：配置エリアの合成キャンバス＋全員の音声（`audioDest`）
  - 自分のカメラ：自分のカメラ映像＋**自分の声だけ**
  - 自分のPC画面：録画開始時に画面を選択（getDisplayMedia）。映像＋**自分の声だけ**（画面の音は取らない）
  - 個人録画（カメラ・PC画面）の「自分の声」は`ownVoiceTrack`＝ノイズ抑制・ゲート・EQ後、イントロクイズの曲を混ぜる前（`setupNoiseGate`で送信用とは別のdestinationに分岐）。通話音声・システム音声・曲は入れない（UCの要望、2026-09-27）。ミュート中は無音。
- 「システム音声を使う」とPC画面の録画は、どちらも画面共有ダイアログから取るので**1回の選択で両方に使う**。システム音声はレイアウト全体の録画にだけ使い、「システム音声＋自分のマイク」にする。
- 選んだ画面の共有を停止すると録画全体を止める。全部のonstopが終わったら描画ループと画面共有を片付ける（`finishRecordingSession`）。
- 画質：カメラは1920×1080・30fps（ideal）で取り込み、通話へは`applyCameraSenderParams`で`scaleResolutionDownBy`（取り込み高さ/360）・`maxFramerate`24・400kbpsに縮めて送る（通話の送信量は以前の640×360取り込みと同じ。自分のカメラの録画だけ高画質になる）。レイアウト全体は表示サイズに関係なく常に1920×1080のキャンバスに描く。ビットレートは明示（レイアウト8Mbps・カメラ4Mbps・PC画面8Mbps、音声128kbps。ブラウザ既定だと低い）。
- 形式はMP4（H.264 High@L4.0 `avc1.640028`＋AAC `mp4a.40.2`、`pickMimeType`）。編集ソフトで読めるようにするためのUC要望（2026-09-27）。ChromeのMediaRecorderが書くMP4は**フラグメント形式**（moof/mdatの繰り返し）でDaVinci Resolve（無料版）が読み込めないため、停止時に`remuxFragmentedMp4`で通常のMP4（ftyp→moov→mdat、moovにstts/ctts/stss/stsc/stsz/co64、1チャンク＝1サンプル、トラックの開始差はelstの空編集）へ並べ替えてから保存する。再エンコードなし・元Blobを`slice()`で参照するのでメモリはほぼ増えない。変換に失敗したら元のBlobを保存。PyAV（FFmpeg）で元ファイルと同じフレーム数・PTSでデコードできることを確認済み。MP4非対応ブラウザではWebMにフォールバックし、拡張子も`recorder.mimeType`に合わせる。
- 停止すると各ファイルを自動でダウンロードする（`onstop`でリンクを`click()`）。リンクは取り直し用に一覧へ残す。同時録画で複数ファイルになるとき、Chromeは初回だけ「複数ファイルのダウンロード」の許可を求める。
- 録画データはメモリに溜める方式のまま（同時録画はメモリ使用量が増える）。

## 一括録画（廃止）

- 2026-09-23にUCの判断で一括録画（各自ローカル録画・同期合図）は削除した。ホスト判定は`isRoomHost()`（最初に入室した人）として早押しで引き続き使用。

## 表示オプション（この端末のみ・localStorage保存。今は右パネルの「配置」「背景・枠」タブにある）

- **名前表示ON/OFF**：`#videoGrid.hideNames`クラスで`.nameLabel`（各タイル右下）をCSSごと隠す。テキストは常にDOMへ設定しておき、表示/非表示だけを切り替える。
- **早押しの演出・サイコロの出目の表示ON/OFF**（UC要望、2026-10-04）：「早押しの演出を表示」「サイコロの出目を表示」（localStorage `showBuzzFx`/`showDiceFx`、既定ON）。OFFだと`#videoGrid.hideBuzzFx`/`.hideDiceFx`で、配置エリアのカットイン・○×・回答者の金枠（`#videoGrid:not(.hideBuzzFx) .tile.buzzAnswer`なので金枠とz-index: 50の最前面化の両方が止まる）／サイコロの出目を隠し、合成録画にも描かない。この端末だけの設定で、早押し・サイコロ自体（パネル・履歴・効果音）はそのまま使える。
- **グリッドスナップ（正方形のマス目）**：横を`gridSnapCols`等分（「マス目 横○分割」4〜128、既定32、localStorage `aiJinroGridCols`）し、縦も同じピクセル幅で刻む（`snapX`/`snapY`。縦の刻み%＝横の刻み%×配置エリアの幅/高さ）。配置エリアが16:9なので、%で同じ刻みにすると長方形になってしまうため。「線を表示」で`#gridLinesOverlay`（CSSのlinear-gradient、pointer-events:none）にマス目を描く。録画（合成キャンバスは`.tile`だけ描く）には入らない。配置エリアのサイズが変わると`ResizeObserver`で線を引き直す。
- **背景画像**：選択した画像をlocalStorageにdata URLで保存し、`#videoGrid`の`background-image`に設定（`applyBgImage`）。
- **枠画像（PNG）**：各タイル内の`.frameOverlay`（z-index最上位・`pointer-events:none`）にdata URLを設定して重ねる（`applyFrameImageToTile`/`applyFrameImageToAllTiles`）。表示ON/OFFは`#videoGrid.showFrame`で制御。
- **タイルごとの枠ON/OFF**（UC要望、2026-09-28）：操作パネルの「枠」列のON/OFFで、タイルごと（カメラ・画面共有）に枠を消せる。`.tile.frameOff`をCSSで隠す（`applyFrameState`、`renderGrid`の最後で毎回適用）。再入室でpeerIdが変わっても残るよう「名前＋画面共有の番号」（`frameTileId`＝名前＋`key`のpeerId以降）をlocalStorage `aiJinroFrameOffTiles`に保存。
- **枠もレイアウト同期する**：以前は枠がこの端末だけの設定で同期されず、追従側で枠が消えていた。リーダーの枠画像は`layoutFrame/{leaderId}`（背景と同じく別ノード、透過を残すためPNGで最大1920×1080、`publishLeaderFrame`）、表示ON/OFFとタイルごとのOFFは`layouts/{leaderId}/frame`（`{show, off:[frameTileId...]}`）で送る。追従中は`followingFrameImage`が非undefinedになり、`effectiveFrameSettings()`がリーダーの設定を返す（枠ボタンは押せない）。追従をやめると自分の設定に戻る。RTDBは配列をオブジェクトで返すことがあるので`off`は`Object.values`で読む。
- **枠は録画にも入る**（2026-09-28）：合成録画（`drawRecordFrame`）で、各タイルの映像を描いた直後に、画面と同じ条件（`#videoGrid.showFrame`・`.frameOverlay.hasImage`・`.frameOff`でない）で`frameImageEl`をオーバーレイの範囲に重ねる。`frameImageEl`は`syncFrameImageEl`で表示中の枠（追従中はリーダーの枠）に合わせる。
- **録画は画面の重なり順どおり**（2026-09-28）：`drawRecordFrame`はタイルを計算後のz-index（`getComputedStyle`。早押しの回答者強調`.buzzAnswer`のz-index: 50 !importantも反映）の小さい順（同じ値はDOM順）に描く。以前はDOM順で、「最前面へ」などの重なり順が録画に反映されていなかった。
- **自分の枠（人ごとに別の枠）**（UC要望、2026-09-28）：「自分の枠(PNG)を設定」で各自が自分のカメラタイル用の枠を選ぶ。PNGのまま最大1920×1080に縮めてlocalStorage `aiJinroMyFrameImage`に保存し、入室時・変更時に`rooms/{ROOM_ID}/memberFrames/{peerId}: { dataUrl, updatedAt }`へ書く（`publishMyFrame`、`onDisconnect`で削除。membersとは別ノードにして大きな画像を毎回配らない）。全員が`listenMemberFrames`で購読する。タイルの枠は`frameUrlForKey`で決め、自分の枠を設定している人のカメラタイルはその枠（レイアウト追従中でも）、それ以外のタイル・画面共有・スコアボードは従来の枠（「共通の枠」＝`effectiveFrameSettings().url`。追従中はリーダーの共通の枠）。枠を表示ON/OFFとタイルごとのOFFは従来どおり見る側（追従中はリーダー）の設定に従う。合成録画は`tile.frameImage`（枠URLごとに共有する`Image`、`frameImageFor`）を描く。
- **無音警告のレイアウト固定**：`#gateWarning`は常にDOMに存在させ、`display`ではなく`visibility`（`.show`クラス）で切り替える。これにより表示/非表示で他要素の行がずれない。
- **聞こえる音量（相手ごと）**：操作テーブルの「聞こえる音量」スライダー（0〜100%）で、この端末で聞こえる相手の音量を変える（`<video>.volume`）。ミュートとは独立。再入室でpeerIdが変わっても残るよう**名前をキー**にlocalStorage（`peerVolumes`）へ保存。100%超の増幅は、Web Audio経由で再生する必要があり、Chromeではその経路だとエコーキャンセルが効かなくなることがあるため行っていない。スライダー操作中は行のドラッグ並び替えを始めない（`volSliderActive`）。
- **タイルの重なり順（手動）**：操作テーブルの各行に「⬆最前面へ/⬇最背面へ」ボタンを持つ（画面共有タイルの行も含む）。`bringTileToFront`は現在の全タイルの最大z-indexを都度計算して+1する（固定カウンタ方式だと、スロット番号由来の既定z＝`idx+1`を追い抜けないバグがあったため修正済み）。`sendTileKeyToBack`は他タイルの最小z-index-1を設定する。

## カメラオフ＝タイル非表示

- 入室画面の「カメラOFFで入室」「マイクをミュートして入室」（localStorage `joinCamOff`/`joinMicMuted`に保存）。デバイスは取得したまま、members登録前にトラックを`enabled=false`にする（入室後の📷/🎤ボタンと同じ仕組みなので、後からONにしても再交渉不要）。

- 「表示/非表示」ボタンは廃止。カメラがオフの参加者はタイルごと非表示にし、真っ黒な枠を出さない（`applyCameraVisibility`、`renderGrid`の最後で毎回適用）。音声は再生し続ける。
- 自分のカメラON/OFFは`members/{myId}/cameraOn`に書き込み、全員の画面で同じタイルが消える。相手の行の「📷 オフ」はこの端末だけでそのタイルを隠す（`locallyHiddenVideoPeers`）。
- レイアウト同期のデータから`hidden`は削除した（表示/非表示は各端末がカメラ状態から決める）。

## サイコロ（オプション機能）

- 「🎲 サイコロ」パネルの「サイコロを使う」で`dice/config/enabled`を切り替える（誰でもON/OFF可。早押しと違いホスト制ではない）。ONの間だけ振る操作欄と出目の表示が全員に出る。
- 「個数d面数」（NdM）で指定（`parseDiceNotation`。`d20`＝1個、全角の「２Ｄ６」も可。個数1〜100・面数2〜10000）。よく使うものはボタン（1d6/2d6/3d6/1d8/1d10/1d20/1d100）。最後に使った指定はlocalStorage `diceNotation`。
- 出目は**振った人の端末**で`crypto.getRandomValues`（棄却法で偏りなし、`rollDie`）で決め、`dice/log`へpushする。全員が同じ値を表示する。
- 表示は配置エリア内の`#diceOverlay`（上部中央。別ウィンドウ表示にも一緒に移る）。届いた瞬間に約1.2秒ランダムな目で転がる演出→確定（効果音は`playBuzzTones`なので合成録画の音声にも入る）→「表示 ○秒」（既定8秒、localStorage `diceShowSec`）で消える。21個以上は出目を並べず合計だけ。入室時点で既にあったログは演出しない（`diceSeenIds`）。
- 合成録画には`drawDiceOverlayOnCanvas`で画面と同じ見た目を描く（`drawRecordFrame`の最後）。
- **立体ダイスの演出**（UC要望、2026-10-08。自作ゲーム「サイトリー」（`D:\遊ぶところ\ゲーム制作\サイトリー\saitory.html`のDGL）の演出を移植）：three.js r128（cdnjsのclassic script）で、配置エリア全体に重ねたWebGLキャンバス`#diceStage`（`Dice3D`）にダイスが下から転がってきて跳ね、出目の面を上にして止まる。形は面数で決める（4=四面体／6=立方体／8=八面体／10=十面体（五角の凧形、`d10Geometry`で自作）／12=十二面体／20=二十面体／それ以外（d2・d3・d100など）=立方体で上の面に出目、他の面は適当な数）。着地まで`DICE_ROLL_ANIM_MS`（1.9秒）、着地のたびに「カッ」（`clack`、`audioCtx.destination`と`audioDest`へ）。31個以上（`DICE_3D_MAX_COUNT`）やthree.jsが読めないときは従来の平面の演出。立体のときは上の箱に出目を並べず「◯◯が NdM を振った」と結果（合計）だけ。止まった絵は表示秒数のあいだ残し、`hideDiceOverlay`で`Dice3D.clear()`（フェードアウト）。
  - キャンバスは`#videoGrid`の中なので別ウィンドウ表示にも一緒に移る。描画はrAF（`gridWin()`）とメイン画面の`setInterval`の両方から進める（別ウィンドウのrAFが止まっても最後は出目の面で止まるように。ヘッドレスの別ウィンドウでrAFが止まって途中で固まったのを確認して追加）。
  - 合成録画は`Dice3D.drawTo`で`#diceStage`をそのまま描く（`preserveDrawingBuffer: true`。配置エリアが小さくても録画がぼやけないよう、ピクセル比を1920/幅まで上げる）。表示オプション「サイコロの出目を表示」OFFでは`#diceStage`も隠れ、録画にも描かない。
- 履歴はパネルに直近30件（`limitToLast`）、「結果をコピー」にも出力する。
- **各自のカメラに結果を表示**（UC要望、2026-10-07。○×のバッジと同じ仕組み）：サイコロを使っている間、カメラタイルの右上（既定。UC指示で2026-10-08に左上から変更）に、その人が最後に振った**結果だけ**（1個なら出目、2個以上なら合計の数字のみ。NdMや個々の出目は出さない、UC指示）を出し続ける（`.diceBadge`、`updateDiceBadges`。`renderGrid`とサイコロのログ・設定の購読から更新）。人の対応は名前（`diceLatestByName`、`fbKey(名前)`）。新しく振られた結果は、中央の転がる演出が確定するのと同時（`DICE_ROLL_ANIM_MS`後、`revealDiceBadge`）に差し替える（それまでは前回の結果のまま）。入室時点のログからも最新の結果を出す。○×制のバッジ（同じく右上）と重なるときは、出目を○×のすぐ下にずらす（`stackDiceBadgeUnderMb`。ドラッグで位置を決めた出目はそのまま。`updateMbBadges`の最後でも置き直す）。ドラッグで移動・ダブルクリックで元の位置、位置はlocalStorage `diceBadgePos`・レイアウト同期は`layouts/{id}/diceBadge`、録画にも描く。○×と共通の部品：`makeTileBadgeDraggable(badge, tile, kind)`（kind＝`{posMap, syncField, save}`）・`applyBadgeKindPos`・`drawTileBadgeOnCanvas`（背景色のあるspanは箱も描く）。表示オプションの「出目をカメラに表示」（localStorage `showDiceBadge`、既定ON、この端末だけ）で消せる。

## 早押し（オプション機能・第一弾）

- **ホスト（司会）の決め方**：各自が早押しパネルの「ホスト（司会）になる」にチェックすると`buzzer/host/name`に自分の名前を書き込み、ホストになる（後からチェックした人に交代。観戦者チェックと同じ操作感）。名前で持つので再読み込みしてもホストのまま。チェックを外すと指定解除。指定が無い／指定された名前の人がルームにいないときは、最も早く入室した人が代行する（`isRoomHost()`）。
- ホストが「早押しを使う」をONにすると全員に早押しパネルが出る（`buzzer/config/enabled`）。判定・状態遷移・設定の書き込みはホスト端末だけが行う。
- **公平性**：押下時刻は各自の`Date.now() + serverTimeOffset`（押した瞬間のサーバー時刻換算）で記録し、届いた順ではなくこの時刻順で順位を決める。最初の押下が届いてから`BUZZ_WINDOW_MS`（500ms）は他の人の押下も受け付け、その後ボタンをロックしてホストが回答者を確定する（`hostAssignAnswerer`）。同時刻は`recv`（サーバー受信時刻）→peerIdで決める。
- **phase**：押し直しの単位。誤答で押していた人が残っていないときは`phase+1`で「誤答した人以外」で押し直し。`question`は問題番号で、「次の問題へ」で`phase`とともに進み、`presses`と`lockedOut`を消す。「押し直し」は問題番号を変えずに同じ処理（押下記録・回答権・誤答ロックを消す。得点は変えない）を行う（`hostResetBuzzQuestion(false)`）。受付開始・お手つき判定は仕様で不要とされたため無い（リセット直後から押せる）。
- **判定後はすぐ押せる状態に戻す**（2026-09-26 UC指示）：正解 → 得点を加えてそのまま次の問題へ（`hostResetBuzzQuestion(true, {judge})`、イントロクイズも次の曲へ）。不正解 → 回答者（チーム戦ならチーム）を`lockedOut`に入れ、同じ問題のまま`phase+1`で押し直し。2着以降へ回答権を回す処理は廃止した。
- **チーム戦**：`members/{id}/buzzTeam`（各自が入力、localStorageにも保存）。チーム戦中は同じチームは1枠として扱い、誤答・勝ち抜け・失格もチーム単位。チーム未入力の人は個人扱い。
- **スコア**：再入室でpeerIdが変わっても残るよう**名前キー**（`fbKey(name)`）で保持。チーム合計は各人のteamから集計。ホストは+1/−1で手動修正でき、「スコアを全消去」は2回押しで実行（confirmダイアログは使わない）。
- 回答者が押した後に退室しても、押下記録の名前で判定・得点できる（`buzzNameOf`）。
- 制限時間は表示と時間切れ音のみで、自動で不正解にはしない（判定はホスト）。
- **押下のP2P即時通知**：各ピア接続に`negotiated: true, id: 0`のデータチャネル`buzz`（順序保証なし・再送2回）を接続作成時に両側で作り、押した瞬間に`{t:"press", phase, at, name, team}`を直接送る。受信側は`buzzFastPresses`に入れて`buzzCurrentPresses()`でFirebaseの記録とマージする（Firebaseの記録が正式・後から上書き）。FirebaseのRTDBはシンガポールにあり往復が遅いため。同一PCの計測でFirebase経由89ms→P2P 9ms。旧バージョンのクライアントとも接続できることを確認済み（旧側はチャネルを作らないので、その相手にはFirebase経由のみで届く）。
- **押した瞬間の即時演出**：誰かの押下がRTDBで届いた瞬間に、全員の端末でピンポーン＋名前のカットイン＋タイルの金色強調を出す（ホストの回答者確定＝受付窓の500ms＋往復を待たない。押した本人の端末は送信前に即時）。2着以降の押下は短い「ピッ」。受付窓の間に、演出済みの人より早い時刻の押下が届いたときは音を鳴らさず表示だけ差し替え、回答者確定時に演出済みの人と同じなら鳴らし直さない（`handleNewBuzzPresses`/`announceBuzzPress`/`buzzAnnounced`）。
- **演出**：回答権が決まる（誤答で次の人に移る等）と名前のカットイン＋タイルを金色に光らせる＋ピンポーン。正解/不正解は画面中央に○/×＋効果音。効果音は`audioCtx.destination`と`audioDest`の両方へ出し、合成録画キャンバスにも枠・カットイン・○×を描く（`drawBuzzerOverlayOnCanvas`）。入室時点の状態では演出しない。
- 「結果をコピー」に早押しの履歴（問題ごとの判定と着順・時間差）とスコアを出力する。
- **方式の切り替え（得点制／○×制）**（UC要望、2026-10-05）：ホストの設定欄「方式」で`buzzer/config/scoreMode`（`"points"`＝従来の得点制・既定／`"marubatsu"`＝○×制）を切り替える。○×制では正解数＝○・誤答数＝×として表示し、勝ち抜け・失格は`maruWin`（○N個で勝ち抜け、既定7）・`batsuOut`（×N個で失格、既定3）で決める（0=なし）。得点制の`winPts`/`maxWrong`とは別の設定（`buzzJudgeStats`が方式ごとに見分ける）。記録はどちらの方式でも同じ`scores/{nameKey}`の`points`/`correct`/`wrong`に加算するので、途中で切り替えても失われない。
- ○×制の表示：パネルのスコア表は○/×を目標数ぶんの枠で並べ（`buzzMarksHtml`）、勝ち抜け→残り→失格、○が多い順・×が少ない順に並べる（`buzzMbCompare`）。ホストの修正ボタンは○と×をそれぞれ±1（`hostAdjustBuzzScore(key, delta, field)`）。スコアボードのタイルも○×の列で描き、失格は薄く表示（`drawScoreboardMbRow`）。
- **各自のカメラに○×を表示**（UC要望、2026-10-07）：○×制の間、参加者（観戦者以外）のカメラタイルの右上に、その人（チーム戦ならチーム）の○の列・×の列（目標数ぶん、まだの分は薄く）と勝ち抜け／失格の札を重ねる（`.mbBadge`、`updateMbBadges`。`renderBuzzer`と`renderGrid`から更新）。大きさはタイル幅に追従（`.tile`に`container-type: inline-size`、`cqw`）。ドラッグでタイル内を移動（タイルの移動にはしない）、ダブルクリックで右上に戻す。位置は人ごと（`fbKey(名前)`）に`{ax:"l"|"r", x, y}`（近い方の左右の辺・上からの距離、タイルに対する割合）でlocalStorage `mbBadgePos`に保存し、レイアウト同期のリーダーは`layouts/{id}/mbBadge`で配る（追従中はリーダーの位置を使い、ドラッグ不可）。合成録画にも描く（`drawTileBadgeOnCanvas`：画面の各spanの位置・色を写す。サイコロの出目のバッジと共通）。表示オプションの「○×をカメラに表示」（localStorage `showMbBadge`、既定ON、この端末だけ）で消せる。
- **勝ち抜け・失格の演出**：判定でその人（チーム）が勝ち抜け・失格になったら、ホストが`judge.result`（`"win"`/`"out"`）と`judge.label`を書き込み、全員の端末で○×の後に「◯◯ 勝ち抜け！」「◯◯ 失格…」のカットインと効果音を出す（得点制の点数先取・誤答失格でも同じ）。

## 既知の制約

- 実カメラ・実マイク・画面共有ピッカーはブラウザのネイティブ許可ダイアログを伴うため、ブラウザ自動操作だけでは動作確認が完結しない。コード変更後は実機（複数タブ/複数人）での確認が必要。

## ノイズ抑制の設定の保存

- ノイズ抑制のON/OFF・強さ（%）・EQのON/OFFとバンドごとのゲインは、端末ごとにlocalStorage `noiseSettings`（JSON）へ保存し、次回の読み込み時に復元する（`saveNoiseSettings`）。変更のたびに保存。以前は毎回既定値（ON・20%・EQ OFF）に戻っていた（UCの要望、2026-09-27）。

## 自分のカメラのズーム（配信画面用）

- 自分のカメラのタイル上でホイール＝ズーム（カーソル位置を中心、100〜400%）、Shift＋ドラッグ＝映す位置の調整（タイル移動より優先。`attachCamZoomHandlers`を`makeDraggable`より先に登録し`stopImmediatePropagation`）。操作パネルの自分の行にも－/＋/↺。設定はlocalStorage `camZoom`（`{z, cx, cy}`、cx/cyは切り抜き中心の0〜1）。UC要望（2026-09-27）。
- 実装は**送信するカメラ映像そのものを切り抜く**（`setupCameraZoom`：MediaStreamTrackProcessor→`new VideoFrame(frame, {visibleRect})`→MediaStreamTrackGenerator。座標・サイズは偶数に丸める）。この`camSendTrack`を送信と自分のタイルに使うので、相手の画面・レイアウト録画にもそのまま反映される。1080p取り込みから切り抜いてから`applyCameraSenderParams`で360pに縮める（縮小率は切り抜き後の高さ`camCropHeight`基準）ので、ズームしても画質が落ちにくい。
- **ズーム固定**（UC要望、2026-10-02。うっかりホイールでズームしてしまうため）：ズーム操作の横の「🔓 ズーム固定／🔒 ズーム固定中」。ONの間はタイル上のホイール・Shift＋ドラッグ・－/＋/↺が効かない（ホイールはpreventDefaultしないのでページが普通にスクロールする。Shift＋ドラッグは通常のタイル移動になる）。設定はlocalStorage `camZoomLock`（"1"/"0"、既定OFF）。
- 「自分のカメラ」の個人録画は`localStream`の元トラックなので**ズームも左右反転もしない**。非対応ブラウザ（Processor/Generatorが無い）ではズーム・反転UIを出さず元トラックを送る。
- **左右反転ON/OFF**（UC要望、2026-09-28）：操作パネルの自分の行の「↔ 反転ON/OFF」。**送るカメラ映像そのもの**を反転する（`setupCameraZoom`内で、切り抜いた範囲をOffscreenCanvasに`setTransform(-1,0,0,1,w,0)`で描き直して`new VideoFrame(canvas, {timestamp})`。VideoFrameだけでは反転できないため）。相手の画面・配信画面・レイアウト録画すべて同じ向きになる。設定はlocalStorage `camMirror`（"1"/"0"、既定OFF）。
- 以前は自分のタイルだけCSS（`.tile.mirror`）で常に左右反転表示していたが廃止し、自分のタイルも送る映像と同じ向きで表示する。ホイール・Shift＋ドラッグの座標は`camMirror`がONのとき反転を戻して計算する。
