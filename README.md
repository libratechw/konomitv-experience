# KonomiTV 利用体験の改善 — 修正候補と検証結果

KonomiTVおよび関連ライブラリ（DPlayer、mpeg2toh264）について、PC・タブレット・スマートフォンをまたいでテレビライブや録画番組をストレスなく視聴できる体験（YouTube、Amazon Prime Video、Netflix等に匹敵する滑らかで途切れない再生）を目指した、不具合調査・修正候補・実機検証の記録です。

※本リポジトリはKonomiTV本体の配布元ではありません。上流（本体および各ライブラリ）への改善提案と検証データの共有を目的としています。

進捗管理や個別の議論は[Issue一覧](https://github.com/libratechw/konomitv-experience/issues)が、検証指標と合格判定の定義は[測定方法（METHODOLOGY.md）](METHODOLOGY.md)が、詳細なログと技術的背景は[調査報告（REPORT.md）](REPORT.md)が正本です。

---

## KonomiTV とこのリポジトリについて

KonomiTVは、PCやスマートフォンのWebブラウザから地上波・BS・CSのリアルタイム視聴（TVライブ）や録画番組の視聴を行えるWebアプリケーションです。録画サーバー（EDCB等）やチューナーサービス（Mirakurun等）と連携し、HLSや独自再生エンジン（mpeg2toh264 / WebAssembly）による画質選択（Original / 1080p / 720p等）を提供しています。

本プロジェクトでは、特に「再生が途中で止まる」「画質を切り替えるとエラーになる」「一時停止から復帰しない」といった視聴体験を損なう問題の解消に向けて、各コンポーネントの挙動を実機で計測・検証しています。

---

## 視聴環境と最初の再生（最短の手順と現実的な期待値）

### 想定・確認環境
- **PC**: Windows（Chrome）、Mac（Safari）
- **タブレット**: iPad Air（第5世代）、iPad mini（第6世代）
- **スマートフォン**: iPhone 15、Android（Galaxy、POCO X3 GT）

### 最初の再生と推奨設定
1. **確実な再生開始**:
   - 初期画質が「1080p」（HLS）の場合、多くの環境で最も安定して再生を開始できます。
   - 一部の環境（iOS/iPadOSなど）では、初期画質を「Original」（mpeg2toh264）に設定していると、チャンネル選択後に自動で再生が始まらず、画面中央やコントローラーの再生ボタンを押す必要があります。
2. **Original画質（無劣化ストリーム）の利用**:
   - 画質切替やシーク時に一時的な停止やエラー（`InvalidStateError`等）が発生する場合があります。途中で停止した場合は、プレイヤーの再起動や画質の再選択で復帰することが確認されています。
3. **一時停止と再開**:
   - TVライブのOriginal画質で長時間（120秒など）一時停止した場合、停止状態は維持されますが、再開時に再生位置がライブ先端へリセットされる、または再生開始までに追加の操作が必要になることがあります。

---

## 利用者視点での主な症状と現在の到達点

### 1. TVライブ Original再生の開始不能・同期ガード
- **利用者の目に見える症状**:
  TVライブをOriginal画質で開始しようとした際、または1080pからOriginalへ切り替えた際に、バッファを読み込んだまま再生時刻が進まず画面が停止する現象が発生していました。
- **原因と修正候補の範囲**:
  DPlayer内部のライブ同期計算が、再生準備中に非有限値（NaN等）や負の値を算出し、それをvideo要素の再生位置に代入してしまうことが停止の一因となっていました。これを代入しないよう除外するガード処理を評価しています。
- **現在の到達点と注意**:
  iPad Air 5での消音自動測定（スクリプトからのplay呼び出し）では再生進行を確認し、iPhone 15やiPad mini 6の実機でも利用時の正常再生が観察されています。ただし、これは**同期ガードという局所的な修正候補の評価**であり、通常UIのタップによる開始確認や、未修正masterとの厳密な直接A/B比較は未完了です。**iOS全般の開始問題を包括的に解決したわけではありません。** また、Android（POCO）では未修正上流版でもこの開始不能は再現していません。

### 2. TVライブの一時停止と再開
- **利用者の目に見える症状**:
  TVライブを一時停止したあと、再度再生ボタンを押しても映像が動かない、または再開までに長い待機時間が発生する。
- **現在の到達点**:
  停止状態の維持は確認できていますが、再生ボタン1回で確実に映像が復帰するかどうかは環境やタイミングに依存します。後述の未公開統合版において、切替中の論理状態を引き継ぐことで復帰率の改善を試みていますが、停止していた位置は失われ、最新時刻への再構築が発生します。

### 3. 画質切替・シーク時のエラー表示
- **利用者の目に見える症状**:
  録画やライブで画質を切り替えた際、あるいは連続してシークした際に、画面にエラーメッセージが表示されて再生が止まる。
- **現在の到達点**:
  DPlayer側で古いvideo要素のイベントを無視する修正（`candidate/ignore-stale-video-events`）や、KonomiTV側のエラーハンドラ多重登録を防ぐ修正（`candidate/register-native-error-once`）により、切替時の干渉を低減させています。ただし、Safariにおける`InvalidStateError`の根本解消には至っていません。

---

## 公開済みの修正候補（独立ブランチ一覧）

2026年9月12日時点。以下は上流への提案を検討している変更ですが、採用済みや提出準備完了を意味するものではありません。各候補のリンク先でコードを確認できます。検証内容（静的検証・オフライン検証・実機A/B）と、採用判断に残る未確認事項を分けて記載しています。

### 1. [DPlayer] 画質切替後に古い映像のイベントが干渉する問題
- **コード**: [`candidate/ignore-stale-video-events`](https://github.com/libratechw/DPlayer/tree/candidate/ignore-stale-video-events)（検証対象: `8e49bb7`）
- **内容**: 切替前のvideo要素から遅れて届くイベントや`play()`拒否が、切替後の新video要素を意図せずpauseさせる干渉を防ぎます。
- **確認済み（実機A/B）**: Galaxyの比較試験で、旧videoのイベント中継（1回→0回）と拒否によるpause干渉（1回→0回）が解消し、現行videoの制御・画質切替・全画面・キャプチャ・再生進行が維持されることを確認しました。
- **残る確認**: iOSでの`InvalidStateError`やライブOriginal開始失敗への効果、同じvideo要素を再利用する`switchVideo()`への影響は未確認です。

### 2. [KonomiTV] 画質切替ごとにエラー処理が重複登録される問題
- **コード**: [`candidate/register-native-error-once`](https://github.com/libratechw/KonomiTV/tree/candidate/register-native-error-once)（検証対象: `03143a5`）
- **内容**: DPlayerのNative error handlerを、画質切替ごとではなくDPlayer初期化時に1回だけ登録します。エラー受付時とライブ待機後に、対象videoと再生backendが現在のものかを照合します。
- **確認済み（静的検証）**: 型検査、ESLint、提出前レビューを通過しています。
- **残る確認**: iOSでのHLS→Original反復切替、現行HLS videoのエラーによる再起動、待機中の画質切替・再生成の実機確認が残っています。再起動連鎖の解消を実機で確認した段階ではありません。

### 3. [mpeg2toh264] autoFilmの解析負荷を減らす
- **コード**: [`candidate/autofilm-comb-score-indexing`](https://github.com/libratechw/mpeg2toh264/tree/candidate/autofilm-comb-score-indexing)（検証対象: `dcfe571`）
- **内容**: comb score算出時の行参照をpixel loop外へ移し、判定結果を変えずに同期解析処理を短縮します。
- **確認済み（オフライン検証・実機診断）**: 4素材のオフライン解析で約6〜9%の短縮と判定結果の一致を確認しました。Galaxyの実機診断でも同期解析時間（17.7ms→16.8ms）の短縮を確認しています。
- **残る確認**: 解析処理時間の改善であり、全端末でのコマ落ちや音ずれ軽減を示すものではありません。Windowsの同一runnerによる全編再生比較では候補の短縮を確認できず、Galaxy以外の実表示、画素品質、可聴A/V同期の確認が残っています（[REPORT.mdのautoFilmの表示負荷](REPORT.md#autofilmの表示負荷)）。

### 4. [mpeg2toh264] TSの欠損前に完成していた映像を残す
- **コード**: [`candidate/preserve-complete-pictures-before-loss`](https://github.com/libratechw/mpeg2toh264/tree/candidate/preserve-complete-pictures-before-loss)（検証対象: `c3406ab`）
- **内容**: TS packet欠落を検出した際、欠落直前までに完成していたpictureまで破棄しないようにします。
- **確認済み（オフライン検証・実機A/B）**: 2種類の欠損パターンで映像sampleが10〜12枚多く残ることをオフラインで確認しました。Galaxyの1時間比較では、欠損1回あたりのbrowser drop（`droppedVideoFrames`）中央値が13枚から2枚へ減少しました。
- **残る確認**: 正常TS、別の欠損パターン、画素品質、可聴A/V同期は未確認です。異常区間通過後にフレーム間隔が乱れる問題（cadence不良）は、この修正で解消したとは判断していません。

### 5. [mpeg2toh264] 描画待ちのフレームをまとめて捨てる処理を除く
- **コード**: [`candidate/yadif-queue-fallback-removal`](https://github.com/libratechw/mpeg2toh264/tree/candidate/yadif-queue-fallback-removal)（検証対象: `2bc48a0`）
- **内容**: YADIFのqueue全消去と、空きslot不足時にqueued slotを上書き再利用するfallbackを削除します。
- **確認済み（静的網羅検証・単体テスト）**: 全6,386状態の列挙で、容量整理後のslot割当失敗が0件であることを確認しました。正常60iの短時間試験でも既知の退行はありません。
- **残る確認**: 実機での改善効果、異常TSからの長時間復帰、Worker描画、可聴A/V同期は未確認です。状態列挙の成功だけで実際の表示品質が改善するとは判断しません。

### 6. [mpeg2toh264] 録画の終端でHTTP rangeが416になる場合の完了処理
- **コード**: [`candidate/complete-exhausted-http-range-v2`](https://github.com/libratechw/mpeg2toh264/tree/candidate/complete-exhausted-http-range-v2)（先端・dist `d011466` / source `9c0b1c7`、基点 `faf1464`）
- **内容**: 既知のファイル総量以降へのrange要求がHTTP 416で拒否された場合に限り、変換済み出力を処理して再生を完了（EOF）させます。それ以外の失敗はエラーとして停止します。
- **確認済み（自動テスト・静的検証）**: 実装を直接使う`test-range-eof`、型検査、既存テスト、ビルド、独立レビューを通過しています。
- **残る確認**: iPadの録画Originalでの再現・効果確認、正常TS、画素品質、可聴A/V同期は未確認です。Safariの録画停止全般を解消する修正とは判断していません。

---

## 設計を再検討している公開案

### タッチ端末の中央操作ボタン表示
- **コード**: [`candidate/touch-center-controls`](https://github.com/libratechw/KonomiTV/tree/candidate/touch-center-controls)（検証対象: `45d9a59`）
- **確認済み（実機A/B）**: Galaxyの横画面・録画再生・中央実タップにおいて、CSSセレクタ補正により中央操作ボタンが表示されることを確認しました（[実機比較](results/galaxy-touch-center-controls-live-ab.json)）。
- **課題と再検討**: 画面タップがUI表示ではなく再生・停止になる挙動があり、ボタン表示だけでなくデスクトップ／モバイルの操作判定を含めて見直しています。**この表示補正単独での取り込みは推奨していません。** POCOの実タップ、全画面、視認性、長時間操作は未確認です。Windowsは非タッチ表示のみ確認しています。

---

## dogfood統合環境と検証中の修正

複数の変更を統合して評価する公開branchは[`dogfood/integration`](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)です。

> **配備版に関する注意**: 実機測定を行った配備版（KonomiTV source `3b8aed1` / dist `e6d9cf7`、DPlayer `2499f05`）には**未公開のローカルコミット**が含まれており、公開branchの先端（`9adfec3`）と同一ではありません。以下の測定結果は記載した特定ビルド・条件に限定され、公開先端全体を保証するものではありません。また、dogfoodにのみ含まれる修正が将来独立候補として公開されるかは未定です。

### TVライブの一時停止と再開
Original設定で120秒一時停止し、再生ボタンを1回押す試験です。「停止維持」「再開後の時刻進行」「停止位置の保持」を別指標として評価しています。

- **旧測定版（KonomiTV source `3b8aed1` / dist `56f83a7`、DPlayer `2467f23`）**:
  - POCO / Android Chrome（低遅延OFF/ON各2回、計4回）:
    - 停止維持: 4/4（120秒待機中に勝手に再生されない）
    - 再開後の時刻進行: OFF 2/2、ON 1/2（15秒以内に進行）
    - 停止位置の保持: 0/4（全4件で再構築により停止位置喪失）
  - Mac / Safari（低遅延OFF/ON各1回）: 120秒待機後、単一操作で再生時刻・フレーム数が進行。厳密な映像要求経路は未捕捉。
  - 両環境とも物理表示、可聴音声、A/V同期は未確認です。

- **新測定版（未公開: KonomiTV dist `e6d9cf7` / DPlayer `2499f05`）**:
  - `switchQuality()`が開始時のnative `video.paused`を保持し続け、途中の操作意図が新videoへ引き継がれない仮説に対し、DPlayerが保持する論理状態（`this.paused`）を参照する修正をdogfood配備版に組み込みました。
  - POCO / Android Chrome（低遅延OFF/ON各3回、計6回）:
    - 停止維持: 6/6
    - 再開後の時刻進行: 6/6（15秒以内に進行。うち5回は操作後にもvideoが交換された後に進行）
    - 停止位置の保持: 0/6（全6件で停止位置喪失）
    - 開始時の追加操作: 視聴画面へ自動遷移した全6回で自動再生が始まらず、一時停止試験の前に開始ボタンを各1回押す必要がありました（停止後の復帰操作とは別）。
  - **残る確認**: 旧版にも成功例があり、この件数だけで再現頻度の改善や問題の全解消とは判断できません。物理表示、可聴音声、A/V同期、長時間安定性、media errorからの復旧テストは未確認です（[Issue #1 確定集計](https://github.com/libratechw/konomitv-experience/issues/1#issuecomment-5638943561)、[Issue #2 開始条件](https://github.com/libratechw/konomitv-experience/issues/2#issuecomment-5638943751)）。

### 統合版での録画Originalの短時間確認
- 未公開の配備版（dist `e6d9cf7` / DPlayer `2499f05`）を用い、POCOのChromeで同一録画をOriginal画質で3回再生しました。
- 対象録画へのHTTP 206応答、1秒以上の再生時刻進行、取得した状態（各試行5回の観測窓、計15回）にmedia errorがないことを確認し、3/3で通過しました（先行runner障害2件は分母から除外）。録画のため低遅延ON/OFFは条件外です。
- **残る確認**: 個々の再生時刻やHTTP 206件数は集計に保存されておらず、実際の画面表示、可聴音声、A/V同期、実指タップ、長時間安定性は未確認です。Safariの録画停止解消や個別修正の効果を示す比較ではありません。

---

## 継続して確認している問題

- **初期設定Originalで自動開始しない問題**: iPhone・iPadでは再生ボタンが必要でした。利用時観察のビルドは特定されておらず、通常UI操作による他端末比較と原因特定を進めています（[Issue #2](https://github.com/libratechw/konomitv-experience/issues/2)）。
- **Windowsネイティブ環境のAMD VCE**: IdeaPadのTVライブ1080pで再生時刻進行とVCEEncCプロセスの稼働（`--adapt-resolution 1920x1080`）を確認しました。別に行った画質ラベルOriginalでの各30分ブラウザ状態監視（再起動0）も確認していますが、30分間のVCE利用や映像の連続表示は未証明です。物理画面、可聴音声、A/V同期は未確認であり、VCEの長時間受入合格やLinuxのAMD runtime互換性を示すものではありません（[REPORT.mdのWindowsネイティブ環境のVCE再生](REPORT.md#windowsネイティブ環境のvce再生)）。
- **Safariの録画Original停止と、異常TS通過後の復帰**: ライブ開始や古いvideoのイベントを修正した結果だけで、これらが解消したとは判断していません。
- **端末ごとの描画差**: Androidの描画を一律メインスレッドへ移す案は、GalaxyとPOCOで結果が逆転したため撤回しました。

---

## 既存PRへの検証材料

- **Starletteの切断時ファイル送信継続問題**:
  - クライアント切断後も`FileResponse`が送信を続ける挙動に対し、既存の公式PR [encode/starlette#3390](https://github.com/Kludex/starlette/pull/3390) へ[実装と測定結果を共有](https://github.com/Kludex/starlette/pull/3390#issuecomment-5548572632)しました。KonomiTVでの影響は [Issue #279](https://github.com/tsukumijima/KonomiTV/issues/279) に報告しています。
  - Windowsの反復シークで復帰時間短縮を確認しましたが、効果は端末・素材・シーク位置に依存します（[REPORT.mdのHTTP Range切断](REPORT.md#http-range切断)）。
  - 比較用branch [`codex/fix-file-response-disconnect`](https://github.com/libratechw/starlette/tree/codex/fix-file-response-disconnect) は検証材料として保持しており、独立した提出候補には含めません。

---

## 再現報告・不具合報告について

動作の不具合や改善要望を発見された場合は、[Issue一覧](https://github.com/libratechw/konomitv-experience/issues)へご報告ください。

調査・再現の確度を高めるため、以下の情報を含めていただけますと大変助かります：
1. **ご利用環境**: 端末名（例: iPad Air 5, Windows PC）、OSバージョン、ブラウザ（Chrome, Safari等）
2. **再生条件**: TVライブか録画番組か、選択画質（Original, 1080p等）、低遅延モードのON/OFF
3. **発生契機**: 再生開始時、画質切替直後、シーク直後、長時間再生中、一時停止からの再開時など
4. **具体的な挙動**: エラーダイアログの文言（`InvalidStateError`など）、映像・音声の状態、操作を受け付けるか
5. **復旧に必要だった操作**: 再生ボタンの再押下、画質再選択、プレイヤー再起動、ブラウザリロードなど

---

## 詳細な証拠とコードの読み方

- [調査報告（REPORT.md）](REPORT.md): 問題別の原因、比較条件、採否判断、残る確認の正本。
- [測定方法（METHODOLOGY.md）](METHODOLOGY.md): 指標、復帰判定（`recoveryOutcome`）、合格条件の正本。
- [公開結果（results/）](results/): 測定集計と元記録のhash。試行数と1試行中の状態取得回数を区別して記録。

結果は測定したsource・dist・素材・runnerに対応付けます。診断コードや単体テストの成功を、実表示・音声・操作の合格へ読み替えません。ブランチ名は公開候補を`candidate/`、測定専用を`diagnostic/`、統合評価を`dogfood/integration`で区別しています。

<details>
<summary>測定専用branchと過去の基準版</summary>

- [`diagnostic/worker-presentation-observability`](https://github.com/libratechw/mpeg2toh264/tree/diagnostic/worker-presentation-observability): 描画backend、rAF、描画submit、frame取込、queueを記録する診断branch。
- [`diagnostic/autofilm-analysis-observability`](https://github.com/libratechw/mpeg2toh264/tree/diagnostic/autofilm-analysis-observability): `autoFilm`のGPU readback、field match、decimateとCPU内訳を記録する診断branch。
- mpeg2toh264・KonomiTVの[`diagnostic/mse-operation-context`](https://github.com/libratechw/mpeg2toh264/tree/diagnostic/mse-operation-context)（KonomiTV側 `4b307e9` / mpeg2toh264側 `a3c0cd3`）および統合診断版[`diagnostic/dogfood-mse-operation-context`](https://github.com/libratechw/KonomiTV/tree/diagnostic/dogfood-mse-operation-context)（`748d0b0`）: iOS実機で最初に失敗するMSE操作とplayer世代を特定する診断branch。

これらは計装・比較専用であり、修正候補としての取り込みは想定していません。過去の基準snapshot（mpeg2toh264 `faf1464`、KonomiTV `ea1962f`）の結果はその版の記録として保持し、現在の評価対象へ無条件に流用しません。

</details>

## 公開範囲

録画データ自体は配布せず、fixtureはSHA-256と欠陥構造で識別します。`results/`にはLAN情報、録画名、ローカルpathを除いた結果を配置しています。このリポジトリの文書とデータは[CC0 1.0](LICENSE)です。
