# KonomiTV 利用体験の改善 — 修正候補と検証結果

KonomiTV本体および関連ライブラリ（DPlayer、mpeg2toh264）の作者・メンテナ向けに、視聴時の不具合調査、各実装箇所の修正候補、および実機・オフライン検証データを共有・整理した記録です。

上流への個別取り込み可否を判断しやすいよう、**「利用者に見える症状」「実装責任箇所」「確認済み事実（静的／オフライン／実機）」「残る未確認事項・推測との境界」**を明確に分離して記載しています。

- **技術背景・比較条件の正本**: [調査報告（REPORT.md）](REPORT.md)
- **指標定義・判定基準の正本**: [測定方法（METHODOLOGY.md）](METHODOLOGY.md)
- **生データ・集計値**: [公開結果一覧（results/）](results/)
- **個別の課題・議論**: [Issue一覧](https://github.com/libratechw/konomitv-experience/issues)

---

## 1. 独立修正候補の一覧と採否判断材料

各候補は上流への個別提案を想定してブランチを分離しています（※採用済みやPR作成完了を意味するものではありません）。

### [DPlayer] ライブ同期計算の非有限・負値を`currentTime`へ渡さない
- **候補**: [`fix/guard-nonfinite-live-sync`](REPORT.md#tvライブoriginalの開始不能)（提出前のpresentation commit: [`a937e92`](REPORT.md#tvライブoriginalの開始不能)、保全branchのHEAD: [`66e5a69`](results/ipad-live-original-negative-sync-guard.json)）。この候補はDPlayerの5ファイルに限定され、KonomiTV本体や配備版を変更しません。
- **利用者に見える症状**: iPad SafariでテレビのLive Originalを再生したとき、再生開始直後に映像が進まず、停止や再起動通知に至ることがあります。
- **原因と責任範囲**: Safariなどのメディア層は、開始直後にdurationやseekableがまだ確定していない過渡的な値を返すことがあります。DPlayerの`sync()`はその値から同期先を計算し、有限値かつ0以上かを検査せず`video.currentTime`へ渡していました。Safari単独の仕様違反と断定する根拠はなく、未確定値を安全に扱う直接の修正対象はDPlayerです。
- **Originalで表面化した理由と他画質との関係**: `sync()`はLive画質共通の入口です。Originalは`/original/mpegts`を`mpeg2toh264`で再生する直接TS経路、1080p等は`mpegts.js`/MSEと`canplay`・`MEDIA_INFO`・バッファ待ちを使う別の開始経路であるため、今回の過渡状態がOriginalで表面化しやすくなりました。他画質に同じガードが既に入っていた、または同じ問題が存在しない、とは判断していません。
- **修正**: 同期先が`Number.isFinite(time) && time >= 0`を満たす場合だけsetterを呼び、不正値（`NaN`、`Infinity`、負値）はsetterを呼ばず既存の再生経路を継続します。
- **確認済み**: 同一iPad AirのLive Originalでupstream/candidateを低遅延OFF→ON→OFFの3組比較。upstreamは3/3で不正値のsetter到達と進行停止、candidateは3/3で不正値のスキップと進行を確認しました。candidateのfixtureはChrome headlessで13/13件pass、baseではガード依存7件が失敗しました。
- **残る未確認事項**: 物理画面、音声、A/V同期、体感上のコマ落ち、全端末・全画質への一般化、candidateの長時間品質は未確認です。実機診断用の観測コードは提出候補に含めていません。

### [DPlayer] 画質切替後の旧videoイベント干渉防止
- **ブランチ / 対象commit**: [`candidate/ignore-stale-video-events`](https://github.com/libratechw/DPlayer/tree/candidate/ignore-stale-video-events)（検証対象: [`8e49bb7`](https://github.com/libratechw/DPlayer/commit/8e49bb7)）
- **利用者に見える症状**: 画質切替後に映像が意図せず一時停止（pause）したり、再生が不安定になる。
- **実装責任箇所と修正内容**: 切替前の旧video要素から遅延して発火するイベントや`play()`拒否プロミスが、新video要素へ中継されてpause状態を上書きする不整合を防ぎます。
- **確認済み（実機A/B）**: Galaxyでの比較試験において、旧videoのイベント中継（1回→0回）と拒否起因のpause干渉（1回→0回）の解消を確認。現行videoの制御、画質切替、全画面、キャプチャ、再生進行が維持されることを確認済み。
- **残る未確認事項**: iOSにおける`InvalidStateError`やライブOriginal開始失敗への影響、同一video要素を再利用する`switchVideo()`への影響は未確認。

### [KonomiTV] Native error handlerの重複登録防止
- **ブランチ / 対象commit**: [`candidate/register-native-error-once`](https://github.com/libratechw/KonomiTV/tree/candidate/register-native-error-once)（検証対象: [`03143a5`](https://github.com/libratechw/KonomiTV/commit/03143a5)）
- **利用者に見える症状**: 画質切替の繰り返しやシーク操作時に、多重にエラーが表示されて停止する。
- **実装責任箇所と修正内容**: DPlayerのNative error handlerが画質切替のたびに多重登録されていたのをDPlayer初期化時の1回のみに変更。エラー受付時およびライブ待機復帰時に、対象video要素と再生backendが現行世代であるかを照合するガードを追加。
- **確認済み（静的検証）**: 型検査（TypeScript）、ESLint、提出前コードレビューを通過。
- **残る未確認事項**: iOSでのHLS→Original反復切替、現行HLS videoエラー時の再起動連鎖防止、待機中の画質切替・再生成の実機検証は未完了。

### [mpeg2toh264] トランスコード主要処理の計算負荷軽減（出力一致）

- **ブランチ / 対象commit**: [`perf/bit-exact-transcode-hot-paths`](https://github.com/libratechw/mpeg2toh264/tree/perf/bit-exact-transcode-hot-paths)（検証対象: [`581f2b7`](https://github.com/libratechw/mpeg2toh264/commit/581f2b7)、base: [`faf1464`](https://github.com/libratechw/mpeg2toh264/commit/faf1464)）
- **利用者への狙い**: Original再生時の変換負荷を、変換結果を変えずに軽減する。
- **実装責任箇所と修正内容**: MPEG-2係数のVLCデコードinline化、H.264量子化処理のラスタ順化による参照負荷軽減、native量子化丸めの整理、CAVLCの不要ビットマスク除去およびluma入力の並び替え。変更前と同じ変換出力を保つ方針で、計算処理を最適化。
- **確認済み**（オフライン）: 同一の約60秒素材による8組の交互比較で、全組の高速化を確認。同環境内の変更前後比において、Linux x86_64ネイティブ版で平均処理時間約23.2%短縮、Node.js上のWASM版で平均変換処理時間約7.7%短縮。WASM版は比較に用いた1素材、ネイティブ版は同素材を含む3素材で、変更前後の出力SHA-256一致を確認。
- **残る未確認事項**: 本最適化単独での実機端末ブラウザにおけるWASM処理性能、および再生開始時間・シーク追随性・コマ落ち解消への直接的な改善効果は未確認。

### [mpeg2toh264] autoFilmの同期解析負荷軽減
- **ブランチ / 対象commit**: [`candidate/autofilm-comb-score-indexing`](https://github.com/libratechw/mpeg2toh264/tree/candidate/autofilm-comb-score-indexing)（検証対象: [`dcfe571`](https://github.com/libratechw/mpeg2toh264/commit/dcfe571)）
- **利用者に見える症状**: 24fps化（autoFilm）有効時にコマ落ちや処理遅延が発生する。
- **実装責任箇所と修正内容**: comb score算出時の行ポインタ参照をピクセル走査ループ外へ移動し、判定結果の等価性を保ったまま同期解析の計算時間を短縮。
- **確認済み（オフライン・実機診断）**: 4素材のオフライン解析で約6〜9%の処理時間短縮と判定完全一致を確認。Galaxy実機診断でも同期解析時間短縮（17.7ms→16.8ms）を確認。
- **残る未確認事項**: Windowsの同一runnerによる全編再生比較では短縮が確認できず。Galaxy以外の実表示品質、可聴A/V同期、コマ落ちへの直接寄与は未確認（[REPORT.md: autoFilmの表示負荷](REPORT.md#autofilmの表示負荷)）。

### [mpeg2toh264] TS欠損直前の完成ピクチャ保持
- **ブランチ / 対象commit**: [`candidate/preserve-complete-pictures-before-loss`](https://github.com/libratechw/mpeg2toh264/tree/candidate/preserve-complete-pictures-before-loss)（検証対象: [`c3406ab`](https://github.com/libratechw/mpeg2toh264/commit/c3406ab)）
- **利用者に見える症状**: パケット欠落を含む放送を受信した際、映像が大きく乱れる・飛ぶ。
- **実装責任箇所と修正内容**: TSパケット欠落検知時、欠落直前までにデコードが完了していたpictureまで巻き込んで破棄しないよう保持処理を変更。
- **確認済み（オフライン・実機A/B）**: 2種類の欠損パターンで映像sampleが10〜12枚多く残ることをオフライン確認。Galaxy実機1時間比較で欠損1回あたりの`droppedVideoFrames`中央値が13枚から2枚へ減少。
- **残る未確認事項**: 正常TSへの影響、他欠損パターン、可聴A/V同期は未確認。欠損区間通過後のフレーム周期乱れ（cadence不良）の解消は本修正の対象外。

### [mpeg2toh264] YADIF描画待ちキューの全破棄・上書き処理削除
- **ブランチ / 対象commit**: [`candidate/yadif-queue-fallback-removal`](https://github.com/libratechw/mpeg2toh264/tree/candidate/yadif-queue-fallback-removal)（検証対象: [`2bc48a0`](https://github.com/libratechw/mpeg2toh264/commit/2bc48a0)）
- **利用者に見える症状**: インターレース解除（YADIF）処理中に急激なフレームドロップや破綻が起きる。
- **実装責任箇所と修正内容**: YADIFのqueue全消去、および空きslot枯渇時にqueued slotを上書き再利用するfallback分岐を削除。
- **確認済み（静的網羅・単体テスト）**: 全6,386状態の列挙により、容量整理後のslot割当失敗が0件であることを確認。正常60i短時間試験で既知のデグレなし。
- **残る未確認事項**: 状態列挙による安全性の確認にとどまり、実機での表示品質改善、異常TSからの長時間復帰、Worker描画、可聴A/V同期への寄与は未確認。

### [mpeg2toh264] 録画終端HTTP 416時の正常EOF完了処理
- **ブランチ / 対象commit**: [`candidate/complete-exhausted-http-range-v2`](https://github.com/libratechw/mpeg2toh264/tree/candidate/complete-exhausted-http-range-v2)（先端・dist: [`d011466`](https://github.com/libratechw/mpeg2toh264/commit/d011466) / source: [`9c0b1c7`](https://github.com/libratechw/mpeg2toh264/commit/9c0b1c7)、基点: [`faf1464`](https://github.com/libratechw/mpeg2toh264/commit/faf1464)）
- **利用者に見える症状**: 録画再生の末尾でエラーが表示される、または終了処理が完了しない。
- **実装責任箇所と修正内容**: 既知のファイルサイズ以降へのRangeリクエストがHTTP 416（Range Not Satisfiable）で返された場合に限り、変換済みバッファをフラッシュして正常完了（EOF）として処理。それ以外のHTTPエラーは従来通り停止。
- **確認済み（自動テスト・静的検証）**: 直接検証テスト（`test-range-eof`）、型検査、既存テスト、ビルド、独立レビューを通過。
- **残る未確認事項**: iPad実機の録画Original再生における再現・効果確認、正常TS・画素品質・可聴A/V同期は未確認。Safari特有の録画停止問題全般を解決するものではない。

---

## 2. dogfood統合検証（[`dogfood/integration`](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)）

複数の修正を日常利用環境で横断評価するための統合ブランチです。

> **配備ビルドの境界**:
> 実機測定に用いた配備環境（KonomiTV source [`e6d9cf7`](https://github.com/libratechw/KonomiTV/commit/e6d9cf7)、DPlayer [`2499f05`](REPORT.md#tvライブの一時停止と再開)）は公開forkの[`dogfood/integration`](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)に含まれます。ただし、配備イメージはビルド時点の固定スナップショットであり、ブランチ上のREADME更新等が配備済みコンテナへ遡って反映されるものではありません。以下の数値は特定環境下での測定結果です。

### TVライブ Original再生の同期ガード & 一時停止／再開（Live pause / resume）
- **対象commit / 修正内容**:
  - DPlayer [`2499f05`](REPORT.md#tvライブの一時停止と再開): ライブ同期計算が非有限値（NaN等）や負値を算出した場合に`currentTime`への代入を除外するガードを追加。
  - DPlayer [`2499f05`](REPORT.md#tvライブの一時停止と再開): 画質切替時にnative `video.paused`ではなくDPlayerの論理状態（`this.paused`）を参照し、再生意図を新videoへ引き継ぐ処理を統合。
- **測定方法**: TVライブ Original設定で120秒一時停止後、再生ボタンを1回押下。「停止維持」「15秒以内の時刻進行」「停止位置の保持」を分離して計測。
- **測定結果の推移**:
  - **旧版（KonomiTV [`3b8aed1`](https://github.com/libratechw/KonomiTV/commit/3b8aed1) / dist [`56f83a7`](https://github.com/libratechw/KonomiTV/commit/56f83a7)、DPlayer [`2467f23`](REPORT.md#tvライブの一時停止と再開)）**:
    - POCO / Android Chrome: 停止維持 4/4、再開後進行（OFF 2/2, ON 1/2）、停止位置保持 0/4（全件で最新時刻への再構築が発生）。
    - Mac / Safari: 低遅延OFF/ON各1回で単一操作による時刻進行を確認。
  - **新版（KonomiTV [`e6d9cf7`](https://github.com/libratechw/KonomiTV/commit/e6d9cf7)、DPlayer [`2499f05`](REPORT.md#tvライブの一時停止と再開)）**:
    - POCO / Android Chrome: 停止維持 6/6（低遅延OFF 3/3, ON 3/3）、再開後進行 6/6（低遅延OFF 3/3, ON 3/3、うち5回は操作後にvideo要素が交換されて進行）、停止位置保持 0/6。
    - ※開始時の追加操作: 視聴画面自動遷移後の全6回で自動再生が始まらず、事前開始操作（各1回）を要した。
- **残る未確認事項**: 旧版でも成功例があり、試行数から再現頻度の完全解消とは断定不可。実機の物理画面表示、可聴音声、A/V同期、長時間安定性、他端末への一般化は未確認（[Issue #1](https://github.com/libratechw/konomitv-experience/issues/1#issuecomment-5638943561)、[Issue #2](https://github.com/libratechw/konomitv-experience/issues/2#issuecomment-5638943751)）。

### 統合版での録画Original短時間確認
- **配備版（KonomiTV [`e6d9cf7`](https://github.com/libratechw/KonomiTV/commit/e6d9cf7) / DPlayer [`2499f05`](REPORT.md#tvライブの一時停止と再開)）**:
  - POCO / Android Chromeにて同一録画のOriginal再生を3回実施。
  - 対象録画へのHTTP 206応答、1秒以上の再生時刻進行、観測窓（各試行5回、計15回）でのmedia error不在を確認（3/3通過。先行runner障害2件は分母から除外）。
- **残る未確認事項**: 実表示品質、可聴音声、A/V同期、実指タップ操作、長時間安定性は未確認。Safariでの録画停止問題への有効性は未確認。

---

## 3. 設計再検討中の案

### [KonomiTV] モバイル・タッチ端末の中央操作UI表示
- **ブランチ / 対象commit**: [`candidate/touch-center-controls`](https://github.com/libratechw/KonomiTV/tree/candidate/touch-center-controls)（検証対象: [`45d9a59`](https://github.com/libratechw/KonomiTV/commit/45d9a59)）
- **確認事実**: Galaxy実機（横画面・録画・中央実タップ）にて、CSSセレクタ補正により中央操作ボタンが表示されることを確認（[実機比較データ](results/galaxy-touch-center-controls-live-ab.json)）。
- **再検討理由**: 画面タップ時にUIトグルではなく直接「再生／停止」がトリガーされる挙動が確認され、CSS補正単独では操作性が損なわれるため**単独での取り込みは非推奨**。デスクトップ／モバイルの操作イベント判定全体の再設計が必要。

---

## 4. 継続調査・未解決の課題

1. **初期画質Original時の自動再生開始失敗**:
   - iPhone・iPadにおいて、チャンネル遷移後に自動再生されず手動タップが必要となる問題。同期ガード適用後も通常UI遷移時の挙動差異が残っており継続調査中（[Issue #2](https://github.com/libratechw/konomitv-experience/issues/2)）。
2. **Windowsネイティブ環境でのAMD VCE**:
   - IdeaPad実機にてTVライブ1080p再生進行と`VCEEncC`プロセス（`--adapt-resolution 1920x1080`）の稼働を確認。Originalラベルでの30分間監視（再起動0）も確認したが、30分間の連続VCE稼働・実映像表示は未証明。実表示・音声・A/V同期は未確認であり、Linux等の他環境への互換性は未保証（[REPORT.md: Windowsネイティブ環境のVCE再生](REPORT.md#windowsネイティブ環境のvce再生)）。
3. **Safariにおける録画Original停止および異常TS通過後の復帰**:
   - ライブ開始の同期ガードや旧videoイベント無視修正のみでは解決せず、根本原因の追跡を継続中。
4. **描画スレッド（Main vs Worker）の端末間差異**:
   - Android端末においてメインスレッド描画へ一律移行する案は、GalaxyとPOCOで性能指標が逆転したため撤回済み。

---

## 5. 外部ライブラリ（Starlette）切断問題の検証材料

- **問題の所在**: クライアント切断後も`FileResponse`がバックグラウンドで不要な送信を継続し、シーク復帰性能を圧迫する問題。
- **状況**: 上流PR [encode/starlette#3390](https://github.com/Kludex/starlette/pull/3390) へ[検証データを提供](https://github.com/Kludex/starlette/pull/3390#issuecomment-5548572632)。KonomiTV側の影響は [KonomiTV Issue #279](https://github.com/tsukumijima/KonomiTV/issues/279) にて報告。
- **検証ブランチ**: [`codex/fix-file-response-disconnect`](https://github.com/libratechw/starlette/tree/codex/fix-file-response-disconnect) は検証用材料として保持し、独自PRとしては提出しません。

---

## 6. 検証アーティファクトと診断コードの参照

### 診断・計測専用ブランチ（取り込み対象外）
以下のブランチは問題切り分けと観測ログ取得のための計装コードであり、上流へのマージは想定していません。
- [`diagnostic/worker-presentation-observability`](https://github.com/libratechw/mpeg2toh264/tree/diagnostic/worker-presentation-observability): 描画backend、rAF、submit、フレーム取込キューの記録。
- [`diagnostic/autofilm-analysis-observability`](https://github.com/libratechw/mpeg2toh264/tree/diagnostic/autofilm-analysis-observability): `autoFilm`のGPU readback、field match、decimate処理時間内訳の計測。
- [`diagnostic/mse-operation-context`](https://github.com/libratechw/mpeg2toh264/tree/diagnostic/mse-operation-context)（KonomiTV: [`4b307e9`](https://github.com/libratechw/KonomiTV/commit/4b307e9) / mpeg2toh264: [`a3c0cd3`](https://github.com/libratechw/mpeg2toh264/commit/a3c0cd3)）および統合診断版[`diagnostic/dogfood-mse-operation-context`](https://github.com/libratechw/KonomiTV/tree/diagnostic/dogfood-mse-operation-context)（[`748d0b0`](https://github.com/libratechw/KonomiTV/commit/748d0b0)）: iOS実機におけるMSE操作失敗箇所の特定。

### データの扱いについて
- 過去の基準版（mpeg2toh264 [`faf1464`](https://github.com/libratechw/mpeg2toh264/commit/faf1464)、KonomiTV [`ea1962f`](https://github.com/libratechw/KonomiTV/commit/ea1962f)）の測定データは特定条件の記録として保持し、現行コードへの無条件な当てはめは行いません。
- 録画データ自体は再配布せず、SHA-256ハッシュおよびパケット欠落構造で識別しています。
- 公開ファイル群（`results/`等）にはLAN情報、実録画タイトル、ローカルパスは含まれません。
- 本リポジトリのドキュメントおよび検証データは [CC0 1.0](LICENSE) で公開されています。
