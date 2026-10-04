# KonomiTV 利用体験の改善 — 修正候補と検証結果

※大半をLLMで書かせています、読みづらいところがあったら申し訳ありません（2026-09-15 人間が修正）

KonomiTV本体および関連ライブラリ（DPlayer、mpeg2toh264）の作者・メンテナ向けに、視聴時の不具合調査、各実装箇所の修正候補、および実機・オフライン検証データを共有・整理した記録です。

- **技術背景・比較条件**: [調査報告（REPORT.md）](REPORT.md)
- **指標定義・判定基準**: [測定方法（METHODOLOGY.md）](METHODOLOGY.md)
- **生データ・集計値**: [公開結果一覧（results/）](results/)
- **個別の課題・議論**: [Issue一覧](https://github.com/libratechw/konomitv-experience/issues)

## 所有中の検証端末

- **Linux (Ubuntu 26.04)**: GMKtec EVO-X2 (FHD / AMD Ryzen AI Max+ 395 / 128GB RAM)
- **Windows 11**: BTO (FHD / Core i7-12700 / RTX3060 12GB / 64GB RAM)
- **Windows 11**: Lenovo IdeaPad Flex 550 (FHD / AMD Ryzen 7 4700U / 16GB RAM)
- **Mac**: MacBook Air M1 2020 (M1 / RAM 16GB)
- **Android**: Galaxy Tab S11 Ultra (14.6inch 2960x1848 120fps対応 / MediaTek Dimensity 9400+ / 12GB RAM)
- **Android**: POCO X3 GT (2400x1080 120fps対応 / MediaTek Dimensity 1100 / 8GB RAM)
- **iPad**: iPad Air Gen 5
- **iPad**: iPad mini Gen 6
- **iPhone**: iPhone 15

---

## 1. 独立修正候補の一覧と採否判断材料

独立修正候補は上流への個別提案を想定してブランチを分離しています。公式版への追従状況も併記します。

### [mpeg2toh264] otya128氏の変更を含む公式版への追従

2026年9月27日時点で、otya128氏の変更を取り込んだ [tsukumijima版mpeg2toh264](https://github.com/tsukumijima/mpeg2toh264/commit/23d270cd29beb8cd6ce8f00947fe84934219b8c9) を採用しました。新しい描画スケジューラーとGPU filmを含む版です。以前、個別の修正候補として掲載した「TS欠損直前の完成ピクチャ保持」と「録画末尾のHTTP 416処理」も公式版に含まれるため、独自パッチを重ねていません。

[KonomiTV](https://github.com/tsukumijima/KonomiTV/commit/423665b1f9b4bb24c1ce3486c34a4e9112c96874) と [DPlayer](https://github.com/tsukumijima/DPlayer/commit/e1132aa9bef23013dfd49ea228622daa6aebc0b0) の開発基準も、同日時点の公式最新版へ更新しました。24fps設定ではKonomiTVから新APIの `film` を指定します。film処理が失敗した場合、ライブラリは通常のYADIF表示を継続しますが、従来のCPU autoFilmへは戻りません。

現在のdogfoodは、これらの公式版を基点に、[KonomiTVのLive停止・復帰などの修正](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)と[DPlayerの同期・画質切替の修正](https://github.com/libratechw/DPlayer/tree/dogfood/integration)を維持した構成です。サーバーの起動、主要画面と録画Originalの応答、配信資産とビルドの一致を確認しました。この構成での実視聴による滑らかさ、音声同期、長時間再生、各端末の再評価はまだ行っていません。

以前の「完全取込2案」「完全取込＋追加3件」「CPU autoFilm最適化」とその性能表は、[当時の検証記録](https://github.com/libratechw/konomitv-experience/blob/e238e1f84ab4c5708024e865dfc05214b64c8454/README.md)として参照できます。数値は旧構成のものであり、今回の公式最新版の改善幅を示すものではありません。

### [DPlayer] ライブ同期計算の非有限・負値を`currentTime`へ渡さない
- **候補**: [`fix/guard-nonfinite-live-sync`](https://github.com/libratechw/DPlayer/tree/fix/guard-nonfinite-live-sync)（検証対象: [`a937e92`](https://github.com/libratechw/DPlayer/commit/a937e92)）
- **利用者に見える症状**: iPad SafariでTVライブストリーミングのOriginal画質を再生したとき、再生開始直後に映像が進まず、再生停止やプレイヤー再起動になることがありました。
- **原因と責任範囲**: Safariなどのメディア層は、開始直後にdurationやseekableが無効な値を返すことがあります。DPlayerの`sync()`はその値から同期先を計算し、「有限値かつ0以上」であることは検査せず`video.currentTime`へ渡していました。無効な値が入力される動作はSafari側のバグだったとしても解決困難なため、DPlayer側で入力値検証を行うのが現状最適だと判断しました。
- **Originalで表面化した理由と他画質との関係**: `sync()`はLive画質共通の入口なため、理論上はOriginal以外の画質でもこの現象は起こり得ます。Originalは`/original/mpegts`を`mpeg2toh264`で再生する直接TS経路、1080p等は`mpegts.js`/MSEと`canplay`・`MEDIA_INFO`・バッファ待ちを使う別の開始経路のため、今回の不安定な状態がOriginalで表面化しやすくなったのではと推測します。
- **修正**: 同期先が`Number.isFinite(time) && time >= 0`を満たす場合だけsetterを呼び、不正値（`NaN`、`Infinity`、負値）の場合はsetterを呼ばず既存の再生経路を継続します。
- **確認済み**: iPad AirのLive Originalで「修正前／修正後」をTV低遅延再生OFF→ON→OFFの3回で比較。修正前は3/3で不正値のsetter到達と進行停止を確認、修正後は3/3で不正値のスキップと進行を確認しました。  
[自動テスト用に、再生時間がまだ決まっていない状態や負の値になる状態を再現する13ケース](https://github.com/libratechw/DPlayer/blob/a937e92/tests/live-sync.html)を、HeadlessのChromeで実行しました。修正版は13ケースすべて成功し、修正前の版では今回のガードに関係する7ケースが失敗しました。  
以前のdogfood構成でこの修正を組み込み、Galaxyで日常利用した範囲では不都合は起きませんでした。
- **残る未確認事項**: 幅広い端末での回帰テスト。

### [DPlayer] 画質切替後の旧videoイベント干渉防止
- **ブランチ / 対象commit**: [`candidate/ignore-stale-video-events`](https://github.com/libratechw/DPlayer/tree/candidate/ignore-stale-video-events)（検証対象: [`8e49bb7`](https://github.com/libratechw/DPlayer/commit/8e49bb7)）
- **利用者に見える症状**: 画質切替後に映像が意図せず一時停止（pause）したり、再生が不安定になる。
- **実装責任箇所と修正内容**: 切替前の旧video要素から遅延して発火するイベントや`play()`拒否プロミスが、新video要素へ中継されてpause状態を上書きする不整合を防ぎます。
- **確認済み（実機A/B）**: Galaxyでの比較試験において、旧videoのイベント中継（1回→0回）と拒否起因のpause干渉（1回→0回）の解消を確認。現行videoの制御、画質切替、全画面、キャプチャ、再生進行が維持されることを確認済み。  
以前のdogfood構成でこの修正を組み込み、Galaxyで日常利用した範囲では不都合は起きませんでした。
- **残る未確認事項**: iOSにおける`InvalidStateError`やライブOriginal開始失敗への影響、同一video要素を再利用する`switchVideo()`への影響は未確認。幅広い端末での回帰テストも未実施。

### [KonomiTV] Native error handlerの重複登録防止
- **ブランチ / 対象commit**: [`candidate/register-native-error-once`](https://github.com/libratechw/KonomiTV/tree/candidate/register-native-error-once)（検証対象: [`03143a5`](https://github.com/libratechw/KonomiTV/commit/03143a5)）
- **利用者に見える症状**: 画質切替の繰り返しやシーク操作時に、多重にエラーが表示されて停止する。
- **実装責任箇所と修正内容**: DPlayerのNative error handlerが画質切替のたびに多重登録されていたのをDPlayer初期化時の1回のみに変更。エラー受付時およびライブ待機復帰時に、対象video要素と再生backendが現行世代であるかを照合するガードを追加。
- **確認済み（静的検証）**: 型検査（TypeScript）、ESLintを通過。  
以前のdogfood構成でこの修正を組み込み、Galaxyで日常利用した範囲では不都合は起きませんでした。
- **残る未確認事項**: iOSでのHLS→Original反復切替、現行HLS videoエラー時の再起動連鎖防止、待機中の画質切替・再生成の実機検証は未完了。幅広い端末での回帰テストも未実施。


---

> [!NOTE]
> **注意：ここから下はほぼ自分用メモです**

旧mpeg2toh264候補の実装内容・A/B/C比較表・アーカイブ理由は、[更新前のREADME](https://github.com/libratechw/konomitv-experience/blob/e238e1f84ab4c5708024e865dfc05214b64c8454/README.md)に保存しています。現在の独立修正候補には含めていません。

---

## 2. dogfood統合検証（[`dogfood/integration`](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)）

複数の修正を日常利用環境で横断評価するための統合ブランチです。

> **配備ビルドの境界**:
> 以下のTVライブおよび録画再生の測定数値は、過去の固定スナップショット（KonomiTV [`e6d9cf7`](https://github.com/libratechw/KonomiTV/commit/e6d9cf7)、DPlayer [`2499f05`](https://github.com/libratechw/DPlayer/commit/2499f05e1850690b3764bf9d6d66961078df134b)）における記録です。2026年9月27日時点の統合ブランチ（KonomiTV [`565fdab`](https://github.com/libratechw/KonomiTV/commit/565fdabde5f593a98d41f79d0d271229cd2391db)、DPlayer [`e0c3389`](https://github.com/libratechw/DPlayer/commit/e0c3389b3fdc126ba4c04ac60247ca2816bd09e0)、mpeg2toh264 [`23d270c`](https://github.com/tsukumijima/mpeg2toh264/commit/23d270cd29beb8cd6ce8f00947fe84934219b8c9)）へ、過去の測定値を流用するものではありません。

### TVライブ Original再生の同期ガード & 一時停止／再開（Live pause / resume）
- **対象commit / 修正内容**:
  - DPlayer [`2499f05`](https://github.com/libratechw/DPlayer/commit/2499f05e1850690b3764bf9d6d66961078df134b): ライブ同期計算が非有限値（NaN等）や負値を算出した場合に`currentTime`への代入を除外するガードを追加。
  - DPlayer [`2499f05`](https://github.com/libratechw/DPlayer/commit/2499f05e1850690b3764bf9d6d66961078df134b): 画質切替時にnative `video.paused`ではなくDPlayerの論理状態（`this.paused`）を参照し、再生意図を新videoへ引き継ぐ処理を統合。
- **測定方法**: TVライブ Original設定で120秒一時停止後、再生ボタンを1回押下。「停止維持」「15秒以内の時刻進行」「停止位置の保持」を分離して計測。
- **測定結果の推移**:
  - **旧版（KonomiTV [`3b8aed1`](https://github.com/libratechw/KonomiTV/commit/3b8aed1) / dist [`56f83a7`](https://github.com/libratechw/KonomiTV/commit/56f83a7)、DPlayer [`2467f23`](REPORT.md#tvライブの一時停止と再開)）**:
    - POCO / Android Chrome: 停止維持 4/4、再開後進行（OFF 2/2, ON 1/2）、停止位置保持 0/4（全件で最新時刻への再構築が発生）。
    - Mac / Safari: 低遅延OFF/ON各1回で単一操作による時刻進行を確認。
  - **新版（KonomiTV [`e6d9cf7`](https://github.com/libratechw/KonomiTV/commit/e6d9cf7)、DPlayer [`2499f05`](https://github.com/libratechw/DPlayer/commit/2499f05e1850690b3764bf9d6d66961078df134b)）**:
    - POCO / Android Chrome: 停止維持 6/6（低遅延OFF 3/3, ON 3/3）、再開後進行 6/6（低遅延OFF 3/3, ON 3/3、うち5回は操作後にvideo要素が交換されて進行）、停止位置保持 0/6。
    - ※開始時の追加操作: 視聴画面自動遷移後の全6回で自動再生が始まらず、事前開始操作（各1回）を要した。
- **残る未確認事項**: 旧版でも成功例があり、試行数から再現頻度の完全解消とは断定不可。実機の物理画面表示、可聴音声、A/V同期、長時間安定性、他端末への一般化は未確認（[Issue #1](https://github.com/libratechw/konomitv-experience/issues/1#issuecomment-5638943561)、[Issue #2](https://github.com/libratechw/konomitv-experience/issues/2#issuecomment-5638943751)）。

### 統合版での録画Original短時間確認
- **配備版（KonomiTV [`e6d9cf7`](https://github.com/libratechw/KonomiTV/commit/e6d9cf7) / DPlayer [`2499f05`](https://github.com/libratechw/DPlayer/commit/2499f05e1850690b3764bf9d6d66961078df134b)）**:
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
- **状況**: 上流PR [encode/starlette#3390](https://github.com/Kludex/starlette/pull/3390) へ[検証データを提供](https://github.com/Kludex/starlette/pull/3390#issuecomment-5548572632)。KonomiTV側の影響は [KonomiTV Issue #279](https://github.com/tsukumijima/KonomiTV/issues/279) にて報告。なお、切断対策の上流PR [encode/starlette#3523](https://github.com/Kludex/starlette/pull/3523) はマージ済み。
- **検証コード**: [`codex/fix-file-response-disconnect`](https://github.com/libratechw/starlette/commit/17e3955f997c2f271a08057fe649abadcc482f77)（固定commit: [`17e3955`](https://github.com/libratechw/starlette/commit/17e3955f997c2f271a08057fe649abadcc482f77)、差分: [results/starlette-file-response-disconnect.patch](results/starlette-file-response-disconnect.patch)）は検証用材料として保持し、独自PRとしては提出しません。

---

## 6. 検証アーティファクトと診断コードの参照

### 診断・計測用差分（取り込み対象外）

問題切り分けと観測ログ取得のための計装コードであり、上流へのマージは想定していません。固定コミットおよびソース差分（生成済みdistは含みません）として保存されています。基点・適用手順・SHA256一覧は [results/retired-public-branches-20261005.json](results/retired-public-branches-20261005.json) を参照してください。

- [`diagnostic/worker-presentation-observability`](https://github.com/libratechw/mpeg2toh264/commit/485838acb022da16aa74b2b4dc53bbd5ed3f5a1a)（固定commit: [`485838a`](https://github.com/libratechw/mpeg2toh264/commit/485838acb022da16aa74b2b4dc53bbd5ed3f5a1a)、差分: [results/mpeg2toh264-worker-presentation-observability.patch](results/mpeg2toh264-worker-presentation-observability.patch)）: 描画backend、rAF、submit、フレーム取込キューの記録。
- [`diagnostic/autofilm-analysis-observability`](https://github.com/libratechw/mpeg2toh264/commit/89e2e04941212f76349f0a32c78de54314abc908)（固定commit: [`89e2e04`](https://github.com/libratechw/mpeg2toh264/commit/89e2e04941212f76349f0a32c78de54314abc908)、差分: [results/mpeg2toh264-autofilm-analysis-observability.patch](results/mpeg2toh264-autofilm-analysis-observability.patch)）: `autoFilm`のGPU readback、field match、decimate処理時間内訳の計測。
- [`diagnostic/mse-operation-context`](https://github.com/libratechw/mpeg2toh264/commit/a3c0cd3dc9cae8c6c1493427f6a762dd4de8b25c)（KonomiTV: [`4b307e9`](https://github.com/libratechw/KonomiTV/commit/4b307e9) / mpeg2toh264: [`a3c0cd3`](https://github.com/libratechw/mpeg2toh264/commit/a3c0cd3dc9cae8c6c1493427f6a762dd4de8b25c)、差分: [results/mpeg2toh264-mse-operation-context.patch](results/mpeg2toh264-mse-operation-context.patch)）および統合診断版[`diagnostic/dogfood-mse-operation-context`](https://github.com/libratechw/KonomiTV/commit/748d0b0c969a654a015f075f05f67042fc972db9)（[`748d0b0`](https://github.com/libratechw/KonomiTV/commit/748d0b0c969a654a015f075f05f67042fc972db9)、差分: [results/konomitv-dogfood-mse-operation-context.patch](results/konomitv-dogfood-mse-operation-context.patch)）: iOS実機におけるMSE操作失敗箇所の特定。

### データの扱いについて
- 過去の基準版（mpeg2toh264 [`faf1464`](https://github.com/libratechw/mpeg2toh264/commit/faf1464)、KonomiTV [`ea1962f`](https://github.com/libratechw/KonomiTV/commit/ea1962f)）の測定データは特定条件の記録として保持し、現行コードへの無条件な当てはめは行いません。
- 録画データ自体は再配布せず、SHA-256ハッシュおよびパケット欠落構造で識別しています。
- 公開ファイル群（`results/`等）にはLAN情報、実録画タイトル、ローカルパスは含まれません。
- 本リポジトリのドキュメントおよび検証データは [CC0 1.0](LICENSE) で公開されています。
