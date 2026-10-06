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

### [KonomiTV・DPlayer・aribb24.js] 字幕v2への移行とWorker描画

iPad AirのSafari環境において字幕の出入りで映像が引っ掛かる問題に対処するため、aribb24.js v2と対応環境での字幕Worker描画への移行を提案しています。映像再生経路や文字の独自変換処理に変更はありません。

依存関係の順序（aribb24.js → DPlayer → KonomiTV）に沿った各変更候補は以下のとおりです。

- **aribb24.js**: 同時刻字幕、seek時の再描画（再描画の責務はaribb24.jsのControllerが所有）、Native HLS、Worker障害、画像解放を修正したソース候補 [candidate/caption-v2](https://github.com/libratechw/aribb24.js/tree/candidate/caption-v2)（[3040a7f](https://github.com/libratechw/aribb24.js/commit/3040a7fedab608ffb6ce210a1fe94ab071ba054d)）です。Git依存用のビルド済みファイルは別のブランチ [distribution/caption-v2](https://github.com/libratechw/aribb24.js/tree/distribution/caption-v2)（[0dfd384](https://github.com/libratechw/aribb24.js/commit/0dfd384f6d18031627d467a578fb8b9e2885c00a)）で管理しています。同グループ管理再送後のシーク復元、過去へ遅延した字幕の復元、上書き字幕の履歴管理、古いバッファ履歴のメモリ解放を修正しています。

- **DPlayer**: v2 Controller/Feeder/Rendererと各再生バックエンドの接続、Worker障害時の通知付きmain復旧、snapshotに対応した [candidate/caption-v2](https://github.com/libratechw/DPlayer/tree/candidate/caption-v2)（[b9ad7fc](https://github.com/libratechw/DPlayer/commit/b9ad7fc1035510f91c9befc747664a984e957325)）です。旧flat設定や`getRawCanvas`等からの移行は [移行ガイド](https://github.com/libratechw/DPlayer/blob/b9ad7fc1035510f91c9befc747664a984e957325/docs/guide.md#arib-captions) を参照してください。

- **KonomiTV**: v2設定と字幕付きキャプチャに対応した [candidate/caption-v2](https://github.com/libratechw/KonomiTV/tree/candidate/caption-v2)（[7fe0279](https://github.com/libratechw/KonomiTV/commit/7fe02790d0361d43f23ff7f769e0aa29c054e0b0)）です。キャプチャ処理時、字幕または文字スーパーのsnapshot取得に失敗した場合はユーザーへ通知を行いつつ、正常に取得できた映像や字幕レイヤーのみを用いて画像保存を継続します（映像自体の取得に失敗した場合は従来通りキャプチャ失敗として扱います）。

Webフォントを使う設定はmain描画を維持します。字形はv2標準、DRCSは既存の置換表を維持し、矢印や絵文字の独自補正は行っていません。

- **検証結果**: aribb24.js単体・統合769件、Chromeの対象5件、DPlayer18件、KonomiTVのキャプチャ関連10件のテスト通過、型検査・ビルド、別ディレクトリでの配布物の再現性を確認済みです（SVG snapshotは同条件の基点でも1件不一致）。詳細は [検証結果](results/caption-v2-review-fixes-publication-20261006.json) を参照してください。

- **実機確認**: 先行候補にてiPad Air録画2素材各3分の字幕切り替え、Original/1080p/1080p60での字幕付きJPEGを確認し、本人視聴で引っ掛かり解消を確認しました。最新公式基点への更新版による全端末目視・音声/A-V・長時間は未確認です。

### [mpeg2toh264] otya128氏の変更を含む公式版への追従

2026年9月27日時点で、otya128氏の変更を取り込んだ [tsukumijima版mpeg2toh264](https://github.com/tsukumijima/mpeg2toh264/commit/23d270cd29beb8cd6ce8f00947fe84934219b8c9) を採用しました。新しい描画スケジューラーとGPU filmを含む版です。以前、個別の修正候補として掲載した「TS欠損直前の完成ピクチャ保持」と「録画末尾のHTTP 416処理」も公式版に含まれるため、独自パッチを重ねていません。

[KonomiTV](https://github.com/tsukumijima/KonomiTV/commit/423665b1f9b4bb24c1ce3486c34a4e9112c96874) と [DPlayer](https://github.com/tsukumijima/DPlayer/commit/e1132aa9bef23013dfd49ea228622daa6aebc0b0) の開発基準も、同日時点の公式最新版へ更新しました。24fps設定ではKonomiTVから新APIの `film` を指定します。film処理が失敗した場合、ライブラリは通常のYADIF表示を継続しますが、従来のCPU autoFilmへは戻りません。

現在のdogfoodは、これらの公式版を基点に、[KonomiTVのLive停止・復帰などの修正](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)と[DPlayerの同期・画質切替の修正](https://github.com/libratechw/DPlayer/tree/dogfood/integration)を維持した構成です。サーバーの起動、主要画面と録画Originalの応答、配信資産とビルドの一致を確認しました。この構成での実視聴による滑らかさ、音声同期、長時間再生、各端末の再評価はまだ行っていません。

以前の「完全取込2案」「完全取込＋追加3件」「CPU autoFilm最適化」とその性能表は、[当時の検証記録](https://github.com/libratechw/konomitv-experience/blob/e238e1f84ab4c5708024e865dfc05214b64c8454/README.md)として参照できます。数値は旧構成のものであり、今回の公式最新版の改善幅を示すものではありません。

### [mpeg2toh264] 色差AC量子化のラスタ順化

- **候補リンク**: [`perf/chroma-ac-raster`](https://github.com/libratechw/mpeg2toh264/tree/perf/chroma-ac-raster)（検証HEAD: [`64eaa6b`](https://github.com/libratechw/mpeg2toh264/commit/64eaa6bf67ce3484dd09d72635ca1373c619850d) / 基点: tsukumijima版 main [`69f2a47`](https://github.com/tsukumijima/mpeg2toh264/commit/69f2a47aa5bbeb8ffce57ecf5408eaa85a59d7d1)）
- **変更内容**: 色差ACの乗算と丸めを連続した係数位置順に行い、その後にframe/field scan順へ並べ替えます。係数値・丸め・DC・公開API・描画方式は不変で、他の最適化を含まない2commit（sourceとdist）構成です。
- **性能結果**（固定TS 1440x1080 1本・Worker内WASM変換、AB/BA交互8組）:

| 端末 | 平均変換時間短縮 | 短縮した比較組 |
| :--- | :--- | :--- |
| Linux Chrome 154 headless | 4.46% | 8/8組 |
| Windows ideapad Chrome 154 前面 | 3.69% | 8/8組 |
| MacBook Air M1 Safari 26.6.2 前面 | 6.23% | 8/8組 |
| POCO Android 13 Chrome 154 前面 | 2.65% | 7/8組 |

- **確認済み**:
  - 全72変換で出力全バイト・605映像サンプル・時刻情報が完全一致。
  - `cargo test --release`（278件成功）、固定fixtureハッシュ不変、型検査通過。
  - 候補はソースからの再ビルドでcommit済みdist（全41ファイル）および配布JS内WASMと完全一致。
  - 本比較は公式の配布バイナリーを直接使った比較ではなく、同一ビルド条件（ローカルRust 1.93.0）による公式ソースと候補ソースの比較です（基準の再ビルドWASMは測定WASMと一致しますが、公式のcommit済み配布WASMとはバイトが異なるため）。
- **未確認・留意点**:
  - native CLIはばらつき範囲で明確な改善は未確認です。
  - 外部CPU負荷・SoC温度は未定量であり、固定素材1本の結果を一般化するものではありません。
  - MSE、デコード、YADIF、実画面表示（コマ落ち・視聴fps・起動・シーク）は未測定です（詳細要約と各組生値: [`results/chroma-ac-raster-20261006.json`](results/chroma-ac-raster-20261006.json)）。

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

過去の修正候補・性能実験・診断版について、現役版にない独自変更を含め、distを除外し旧版から再適用検証済みの固定commitとソース差分を[旧候補の保存先と適用手順](results/retired-public-candidates-20261005.json)に保存しています。本記録は取り込み推奨ではなく、当時の実装を確認するための履歴参照用です。

- 旧まとめブランチ [`perf/symmetric-chroma-idct`](https://github.com/libratechw/mpeg2toh264/commit/93c3e295f768222713bab8b75e3b67e3794dc2a0) の6commit由来のソース差分は [`results/mpeg2toh264-symmetric-chroma-idct-retired.patch`](results/mpeg2toh264-symmetric-chroma-idct-retired.patch)（manifest: [`results/retired-public-candidates-20261005.json`](results/retired-public-candidates-20261005.json)）に保存し、基点への再適用で10ファイルが旧先端と一致することを確認済みです。まとめ全体の採用・破棄ではなく選別を行い、色差AC処理のみを独立候補 [`perf/chroma-ac-raster`](https://github.com/libratechw/mpeg2toh264/tree/perf/chroma-ac-raster) へ回収し、明確な改善が見られなかったCAVLC不要マスク除去は保留としています。

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

### [KonomiTV] Galaxy Tabの端末判定とタッチ操作UIの見直し

- **旧CSS案の検証と取り下げ**: Galaxy実機で中央ボタンの表示自体は確認できたものの（[実機比較データ](results/galaxy-touch-center-controls-live-ab.json)）、背景タップがUI表示ではなく再生・一時停止を切り替えるため旧案は取り下げ、[旧CSS案の差分](results/konomitv-touch-center-controls-retired.patch)として保存しています。
- **課題の再設定**: Galaxy Tabがデスクトップ扱いとなる要因やタッチUIとの不一致を調べるため、YouTube等の操作仕様も参考に判定を見直す方針として[Issue #4](https://github.com/libratechw/konomitv-experience/issues/4)を設定しました（現時点で新実装・新測定は未着手）。

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

- **問題と上流の状況**: クライアント切断後もFileResponseが送信を継続する問題について、[上流PR #3390](https://github.com/Kludex/starlette/pull/3390)へ[検証データの提供](https://github.com/Kludex/starlette/pull/3390#issuecomment-5548572632)を行い、影響を[KonomiTV Issue #279](https://github.com/tsukumijima/KonomiTV/issues/279)で報告しました。切断対策の[上流PR #3523](https://github.com/Kludex/starlette/pull/3523)はマージ済みです。
- **検証記録の保存**: 上流対応の完了に伴い独自forkの検証は終了し、過去の内容を[保存ソース差分](results/starlette-file-response-disconnect.patch)および[基点と適用手順](results/retired-public-branches-20261005.json)として保存しています（稼働中KonomiTVでの依存適用状況や、すべてのシーク問題の解消を示すものではありません）。

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
