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

各候補は上流への個別提案を想定してブランチを分離しています。

### [DPlayer] ライブ同期計算の非有限・負値を`currentTime`へ渡さない
- **候補**: [`fix/guard-nonfinite-live-sync`](https://github.com/libratechw/DPlayer/tree/fix/guard-nonfinite-live-sync)（検証対象: [`a937e92`](https://github.com/libratechw/DPlayer/commit/a937e92)）
- **利用者に見える症状**: iPad SafariでTVライブストリーミングのOriginal画質を再生したとき、再生開始直後に映像が進まず、再生停止やプレイヤー再起動になることがありました。
- **原因と責任範囲**: Safariなどのメディア層は、開始直後にdurationやseekableが無効な値を返すことがあります。DPlayerの`sync()`はその値から同期先を計算し、「有限値かつ0以上」であることは検査せず`video.currentTime`へ渡していました。無効な値が入力される動作はSafari側のバグだったとしても解決困難なため、DPlayer側で入力値検証を行うのが現状最適だと判断しました。
- **Originalで表面化した理由と他画質との関係**: `sync()`はLive画質共通の入口なため、理論上はOriginal以外の画質でもこの現象は起こり得ます。Originalは`/original/mpegts`を`mpeg2toh264`で再生する直接TS経路、1080p等は`mpegts.js`/MSEと`canplay`・`MEDIA_INFO`・バッファ待ちを使う別の開始経路のため、今回の不安定な状態がOriginalで表面化しやすくなったのではと推測します。
- **修正**: 同期先が`Number.isFinite(time) && time >= 0`を満たす場合だけsetterを呼び、不正値（`NaN`、`Infinity`、負値）の場合はsetterを呼ばず既存の再生経路を継続します。
- **確認済み**: iPad AirのLive Originalで「修正前／修正後」をTV低遅延再生OFF→ON→OFFの3回で比較。修正前は3/3で不正値のsetter到達と進行停止を確認、修正後は3/3で不正値のスキップと進行を確認しました。  
[自動テスト用に、再生時間がまだ決まっていない状態や負の値になる状態を再現する13ケース](https://github.com/libratechw/DPlayer/blob/a937e92/tests/live-sync.html)を、HeadlessのChromeで実行しました。修正版は13ケースすべて成功し、修正前の版では今回のガードに関係する7ケースが失敗しました。  
Galaxyでもこの修正を組み込んだdogfoodブランチを日常利用していますが、不都合は起きていません。
- **残る未確認事項**: 幅広い端末での回帰テスト。

### [DPlayer] 画質切替後の旧videoイベント干渉防止
- **ブランチ / 対象commit**: [`candidate/ignore-stale-video-events`](https://github.com/libratechw/DPlayer/tree/candidate/ignore-stale-video-events)（検証対象: [`8e49bb7`](https://github.com/libratechw/DPlayer/commit/8e49bb7)）
- **利用者に見える症状**: 画質切替後に映像が意図せず一時停止（pause）したり、再生が不安定になる。
- **実装責任箇所と修正内容**: 切替前の旧video要素から遅延して発火するイベントや`play()`拒否プロミスが、新video要素へ中継されてpause状態を上書きする不整合を防ぎます。
- **確認済み（実機A/B）**: Galaxyでの比較試験において、旧videoのイベント中継（1回→0回）と拒否起因のpause干渉（1回→0回）の解消を確認。現行videoの制御、画質切替、全画面、キャプチャ、再生進行が維持されることを確認済み。  
Galaxyでもこの修正を組み込んだdogfoodブランチを日常利用していますが、不都合は起きていません。
- **残る未確認事項**: iOSにおける`InvalidStateError`やライブOriginal開始失敗への影響、同一video要素を再利用する`switchVideo()`への影響は未確認。幅広い端末での回帰テストも未実施。

### [KonomiTV] Native error handlerの重複登録防止
- **ブランチ / 対象commit**: [`candidate/register-native-error-once`](https://github.com/libratechw/KonomiTV/tree/candidate/register-native-error-once)（検証対象: [`03143a5`](https://github.com/libratechw/KonomiTV/commit/03143a5)）
- **利用者に見える症状**: 画質切替の繰り返しやシーク操作時に、多重にエラーが表示されて停止する。
- **実装責任箇所と修正内容**: DPlayerのNative error handlerが画質切替のたびに多重登録されていたのをDPlayer初期化時の1回のみに変更。エラー受付時およびライブ待機復帰時に、対象video要素と再生backendが現行世代であるかを照合するガードを追加。
- **確認済み（静的検証）**: 型検査（TypeScript）、ESLintを通過。  
Galaxyでもこの修正を組み込んだdogfoodブランチを日常利用していますが、不都合は起きていません。
- **残る未確認事項**: iOSでのHLS→Original反復切替、現行HLS videoエラー時の再起動連鎖防止、待機中の画質切替・再生成の実機検証は未完了。幅広い端末での回帰テストも未実施。

### [mpeg2toh264] otya128上流の完全取込（2案）

- **候補ブランチ**: [`candidate/full-upstream-take`](https://github.com/libratechw/mpeg2toh264/tree/candidate/full-upstream-take)
- **実装責任箇所と修正内容**: tsukumijima版 [`faf1464`](https://github.com/tsukumijima/mpeg2toh264/commit/faf1464) を基点に、otya128版コミット `7b008c6` までの21件の変更を統合した完全取込版です。上流のMBAFF対応・変換処理改善・GPU film・描画周期と表示予定時刻に基づく新スケジューラを取り込みつつ、tsukumijima版の公開API（`autoFilm`・キャプチャ・各種統計・WebWorker描画等）の互換性を維持しています。GPU filmおよびdebugは加算APIであり、既存の `autoFilm` の意味変更ではありません。

同一の統合ライブラリを基点とし、呼出側の変更範囲に応じて2つの選択肢を用意しています。共通ライブラリ自体は障害通知と明示的な再試行APIを提供するのみで、内部で設定を自動切替しません。②では呼出側（KonomiTV）が通知を受けてGPU要求を解除し、CPU autoFilm復帰を試みます。現在のdogfood環境は②を採用しています。

| 案 | 構成 | 24fps（film）動作 | 障害時の挙動 / UI表示 |
| :--- | :--- | :--- | :--- |
| **① API互換案** | ライブラリのみ更新<br>公式KonomiTV（`62b2fc5`相当）依存更新 / DPlayer変更なし | 従来設定はCPU autoFilmを維持（GPU film/debugは加算API） | ライブラリは通知のみ提供。呼出側・DPlayerのUI表示変更なし |
| **② 呼出側移行案** | 同一ライブラリ<br>＋ KonomiTV（[`candidate/gpu-film-migration`](https://github.com/libratechw/KonomiTV/tree/candidate/gpu-film-migration)）<br>＋ DPlayer（[`candidate/film-status`](https://github.com/libratechw/DPlayer/tree/candidate/film-status)） | 24fps設定からGPU filmを明示選択可能 | 障害通知を受けてGPU要求を解除しCPU autoFilmへの復帰を試みる。DPlayer上に状態表示 |

- **確認済み**:
  - **静的検証・ビルド**: Rustテスト274件（release 258件、doctest 16件）、TypeScript型検査、API・Worker時計・PiP・再試行・破棄処理の実ソース回帰を確認。クリーン環境での再ビルドによりライブラリ配布物（dist全41ファイル）のbyte完全一致を確認済み。両案でKonomiTVクライアントビルドおよび内包WASM/Workerの整合性を確認。
  - **動作確認**: POCO実機の独立テストフィクスチャ（消音）にて、再生進行、シーク、キャプチャ、破棄・クリーンアップの正常動作を確認。
  - **視聴確認（① API互換案）**: 公式KonomiTV環境へ本ライブラリのみを適用した実環境（ポート7075）において、筆者自身の視聴で「iPhone, iPad以外はかなり良好だね。」という感触を確認（※筆者による主観的な視聴確認であり、iPhone/iPadでの課題解決を示すものではありません。特定端末・ブラウザ・素材ごとの網羅的な評価データは未取得です）。
- **残る未確認事項**:
  - 最終版におけるSafariおよびFirefoxでの詳細な挙動検証、実ブラウザPiP動作、上流変更に伴うMBAFF行単位の映像比較。
  - ②呼出側移行案のDPlayer状態表示を含む全体構成での比較、および実機視聴の主観比較。下記の最小構成では短時間の機械比較を実施済みです。
  - 全所有端末での網羅テスト、長時間連続再生、字幕表示、可聴音声・A/V同期の詳細確認。

#### 公式版との最小構成比較（2026-09-21）

GPU filmの呼び出し変更まで含めると、描画スレッド時間の短縮と一部端末でのCanvas出力頻度向上を確認しました。一方、ライブラリ更新だけでは一律の改善はなく、判定安定性や出力低下の課題も残ります。全端末での体感上の滑らかさの向上を実証したものではありません。

- **比較対象**: 公式[KonomiTV `62b2fc5`](https://github.com/tsukumijima/KonomiTV/commit/62b2fc5)＋mpeg2toh264 `faf1464`（A）、Aの依存だけを完全取込候補[`c8fe232`](https://github.com/libratechw/mpeg2toh264/commit/c8fe232)へ更新してCPU autoFilmを維持（B）、B＋PlayerControllerのGPU film呼び出し・障害時CPU復帰処理（C）。DPlayerは全版で公式v1.33.1を維持し、状態表示・追加3改善・他のdogfood修正は含めていません。Cは②全体ではなく、呼び出し変更のみの比較です。
- **条件**: Linux実デスクトップChrome、Windows Chrome、POCO Chrome、Mac Safari。録画Original・字幕ON・コメント非表示で、同一素材・媒体区間、端末内の表示寸法を固定し、各条件60秒×2回を順序反転して測定（計56走行）。既存statsの読み取りのみで、追加rVFC/rAFループは使用していません。温度・動作周波数を固定したベンチマークではありません。

音楽素材・24fps設定ON時のCanvas出力fps（各試行の平均値、2回の範囲）:

| 端末・ブラウザ | A：公式CPU | C：完全取込＋GPU呼び出し | 留意点 |
| :--- | ---: | ---: | :--- |
| Linux Chrome | 52.1–52.2 | 52.1–52.8 | 平均fpsはほぼ同じ。missedは増加 |
| Windows Chrome | 54.3–54.5 | 59.7–59.9 | Canvas書き込み頻度は向上 |
| POCO Chrome | 25.7–28.0 | 44.9–54.3 | 元videoのdrop増加も併存。試行差が大きい |
| Mac Safari | 46.2–46.3 | 50.9–51.2 | Canvas書き込み頻度は向上 |

Canvas出力fpsは描画の書き込み頻度であり、実画面への提示や体感上の滑らかさを直接示すものではありません。この音楽ON条件ではBを測定していないため、依存更新単独とGPU呼び出し変更の寄与は分離できません。

- **依存更新のみの回帰候補**: 音楽素材・24fps設定OFFのA/B比較では、POCOが59.5–59.7 fpsから57.2–58.0 fpsへ2回とも低下しました。原因・体感影響は未確定ですが、採用前に調べるべき差です。他端末も含め、依存更新だけの一律の優位性は確認できていません。
- **映画素材・24fps設定ON**: A/B/C比較では、Cの入力1枚あたりの描画スレッド時間（`frameMs`）が全4端末で短縮しました。これはGPU実行時間・端末全体のCPU使用率・消費電力ではありません。Windows/Macは全採点statsでGPU film lockを報告し約24fpsを出力した一方、Linux/POCOはlockの出入りがあり、安定性に課題が残ります。lock報告率は判定の正解率ではありません。
- **未確認の範囲**: 今回の目視、可聴音声・A/V同期、長時間安定性、字幕出現時点との相関。iPad Airは自動化の初期化で停止したため比較対象外であり、iPhoneも未測定です。

### [mpeg2toh264] 完全取込＋追加3件の一括統合ブランチ

- **候補ブランチ**: [`candidate/combined-improvements`](https://github.com/libratechw/mpeg2toh264/tree/candidate/combined-improvements)
- **実装責任箇所と修正内容**: 上記の完全取込ブランチ（`candidate/full-upstream-take`）を基点に、後述の未収録改善3件（CPU IVTC comb-score行参照最適化、TSパケット欠損直前の完成ピクチャ保持、録画終端HTTP 416の正常EOF完了処理）を統合した現行推奨ブランチです（ソース3コミット＋dist再生成1コミット）。旧基点（`faf1464`）向けの旧統合ブランチ（`4ec1102`）は [`archive/20260921/combined-improvements-before-fulltake`](https://github.com/libratechw/mpeg2toh264/tree/archive/20260921/combined-improvements-before-fulltake) へ保存し、本ブランチを新完全取込ベースへ更新しました。
- **確認済み**:
  - **静的検証・ビルド**: 新基点上でRust全体テスト、IVTCテスト、HTTP 416関連テスト（正常EOFを含む実read-loop 7ケース）、TypeScript型検査を通過。先端distはRust/WASMから再生成し内包WASMの整合性を確認。
  - **動作確認（公開コミット・実機検証）**:
    - 最終配備版（KonomiTV `4477647` / DPlayer `9759653` / mpeg2toh264 `38d7e85`）の7016で、POCOは録画Original（ミュージックステーション15232）の一時停止5秒・60秒シーク・再開後45秒進行・設定復元と終了処理を確認。Linux Chrome実デスクトップは録画Originalの33秒進行・1920×1080キャプチャ・主要画面表示（未捕捉例外0件）を確認しました。
    - 同じmpeg2toh264・DPlayerを使う先行ビルド（`fc92834`）では、Windows Chrome（30秒）とMac Safari（約30秒）の録画pause/seek/play、およびPOCOのLive Original（低遅延ON/OFF各30秒）も確認しています。先行版には別依存のmpegts.jsのキャッシュ不一致があり、最終版での全端末再試験とは区別しています。
    - 再生制御・通信・終了処理の機械確認であり、視聴体感やA/V同期の評価ではありません。iPad Airは信頼後も自動化セッション初期化で停止し、製品再生は未確認です。
- **残る未確認事項**: 新基点における各修正単独の実機効果や性能改善率の測定。iPad Air・Firefoxでの再生検証と、全端末での回帰テスト。

依存をコミットハッシュ固定のGit URLで指定し、インストールされたライブラリdist（mpeg2toh264全41ファイル・DPlayer全50ファイル）を対象コミットと照合しました。最終統合版はクリーン再ビルド・コミット済みdist・7016配信物の全236ファイルがbyte一致。②単独のクライアントは他235ファイルが一致し、`sw.js`のみprecache配列の順序差があります（URL/revision全234件と残りのコードは同一）。

### [mpeg2toh264] 過去候補の整理とアーカイブ

旧候補のうち以下の2件は、新基点において前提条件の変化や処理の重複が生じたため、元ブランチを削除しアーカイブタグへ移行しました（過去の測定結果自体を否定するものではなく、新基点における採用根拠が不足しているための整理です。過去の経緯は [README履歴（`0ed1449`）](https://github.com/libratechw/konomitv-experience/blob/0ed1449475b009d13be9dca1e962b1572858dd25/README.md) を参照してください）。

| アーカイブタグ | 旧対象commit | 推奨一覧から外しアーカイブした理由 |
| :--- | :--- | :--- |
| [`archive/20260921/bit-exact-transcode-hot-paths`](https://github.com/libratechw/mpeg2toh264/tree/archive/20260921/bit-exact-transcode-hot-paths) | [`581f2b7`](https://github.com/libratechw/mpeg2toh264/commit/581f2b7) | 上流の変換最適化およびMBAFF変更と接触・重複し、旧パッチはそのまま適用不可（全内容が新基点に取り込まれたわけではありません）。過去の短縮率（23.2% / 7.7%）やbit-exact結果は新基点へ流用できないため、価値が残る部分は新基点を基準に再検討。 |
| [`archive/20260921/yadif-queue-fallback-removal`](https://github.com/libratechw/mpeg2toh264/tree/archive/20260921/yadif-queue-fallback-removal) | [`2bc48a0`](https://github.com/libratechw/mpeg2toh264/commit/2bc48a0) | 上流新スケジューラの導入に伴い前提構造が変化し、旧パッチはそのまま適用不可。新構造でも全破棄やslot再利用処理自体は残っているため上流で問題解消済みではなく、旧6,386状態の解析を根拠に新キューの処理を除去することは不可。 |

### [mpeg2toh264] 完全取込版に未収録の追加改善候補（個別参照用）

以下の3件は単体完全取込版（`candidate/full-upstream-take`）には含まれず、統合ブランチ（`candidate/combined-improvements`）に収録されています。統合版において新基点でのビルド・型検査・単体テスト通過は確認していますが、個別ブランチは従来基点の実装参照として維持しており、新基点における各修正単独の実機効果測定は未実施です。

#### [mpeg2toh264] autoFilmの同期解析負荷軽減
- **ブランチ / 対象commit**: [`candidate/autofilm-comb-score-indexing`](https://github.com/libratechw/mpeg2toh264/tree/candidate/autofilm-comb-score-indexing)（検証対象: [`dcfe571`](https://github.com/libratechw/mpeg2toh264/commit/dcfe571)）
- **利用者に見える症状**: 24fps化（autoFilm）有効時にコマ落ちや処理遅延が発生する。
- **実装責任箇所と修正内容**: comb score算出時の行ポインタ参照をピクセル走査ループ外へ移動し、判定結果の等価性を保ったままCPU側の同期解析時間を短縮（※CPU版 `ivtc.ts` の最適化であり、完全取込版のGPU filmを高速化するものではありません。①のCPU既定経路や②のCPU復帰先として適用対象は残りますが、新基点における有効性や性能向上は未検証です）。
- **確認済み（旧基点での測定）**: 4素材のオフライン解析で約6〜9%の処理時間短縮と判定完全一致を確認。旧版Galaxy実機診断でも同期解析時間短縮（17.7ms→16.8ms）を確認。旧版を組み込んだdogfoodブランチの日常利用では不都合は生じていませんでした。
- **残る未確認事項**: 新基点での各修正単独の実機効果測定。Windowsの同一runnerによる全編再生比較では短縮が確認できず。Galaxy以外の実表示品質、可聴A/V同期、コマ落ちへの直接寄与は未確認（[REPORT.md: autoFilmの表示負荷](REPORT.md#autofilmの表示負荷)）。幅広い端末での回帰テスト。

#### [mpeg2toh264] TS欠損直前の完成ピクチャ保持
- **ブランチ / 対象commit**: [`candidate/preserve-complete-pictures-before-loss`](https://github.com/libratechw/mpeg2toh264/tree/candidate/preserve-complete-pictures-before-loss)（検証対象: [`c3406ab`](https://github.com/libratechw/mpeg2toh264/commit/c3406ab)）
- **利用者に見える症状**: パケット欠落を含む放送を受信した際、映像が大きく乱れる・飛ぶ。
- **実装責任箇所と修正内容**: TSパケット欠落検知時、欠落直前までに組み立て（パケット結合）が完了していた圧縮pictureまで巻き込んで破棄しないよう保持処理を変更（単体完全取込版には未収録、統合版に収録）。
- **確認済み（旧基点での測定）**: 2種類の欠損パターンで映像sampleが10〜12枚多く残ることをオフライン確認。旧版Galaxy実機1時間比較で欠損1回あたりの`droppedVideoFrames`中央値が13枚から2枚へ減少。旧版を組み込んだdogfoodブランチの日常利用では不都合は起きておらず、パケット欠落箇所のカクつき緩和を確認していました。
- **残る未確認事項**: 新基点での単独実機効果測定。幅広い端末での回帰テスト。

#### [mpeg2toh264] 録画終端HTTP 416時の正常EOF完了処理
- **ブランチ / 対象commit**: [`candidate/complete-exhausted-http-range-v2`](https://github.com/libratechw/mpeg2toh264/tree/candidate/complete-exhausted-http-range-v2)（先端・dist: [`d011466`](https://github.com/libratechw/mpeg2toh264/commit/d011466) / source: [`9c0b1c7`](https://github.com/libratechw/mpeg2toh264/commit/9c0b1c7)、基点: [`faf1464`](https://github.com/libratechw/mpeg2toh264/commit/faf1464)）
- **利用者に見える症状**: 録画再生の末尾でエラーが表示される、または終了処理が完了しない。
- **実装責任箇所と修正内容**: 既知のファイルサイズ以降へのRangeリクエストがHTTP 416（Range Not Satisfiable）で返された場合に限り、変換済みバッファをフラッシュして正常完了（EOF）として処理。描画経路（CPU/GPU）とは独立したHTTP入力層の修正（単体完全取込版には未収録、統合版に収録）。
- **確認済み（旧基点での測定）**: 直接検証テスト（`test-range-eof`）、型検査、既存テスト、ビルドを通過。旧版を組み込んだdogfoodブランチの日常利用で不都合のないことを確認していました。
- **残る未確認事項**: iPad実機の録画Original再生における再現・効果確認。幅広い端末での回帰テスト。

---

> [!NOTE]
> **注意：ここから下はほぼ自分用メモです**

---

## 2. dogfood統合検証（[`dogfood/integration`](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)）

複数の修正を日常利用環境で横断評価するための統合ブランチです。

> **配備ビルドの境界と現行構成**:
> 以下のTVライブおよび録画再生の測定数値は、過去の固定スナップショット（KonomiTV [`e6d9cf7`](https://github.com/libratechw/KonomiTV/commit/e6d9cf7)、DPlayer [`2499f05`](https://github.com/libratechw/DPlayer/commit/2499f05e1850690b3764bf9d6d66961078df134b)）における記録です。現行の統合ブランチ（KonomiTV [`dogfood/integration`](https://github.com/libratechw/KonomiTV/tree/dogfood/integration)、DPlayer [`dogfood/integration`](https://github.com/libratechw/DPlayer/tree/dogfood/integration)）は②GPU film移行と新基点combined 3案を取り込んだ構成へ更新されており、過去の測定値を新配備の結果として流用するものではありません。

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
