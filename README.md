# 開発者向け検証インデックス・修正候補

KonomiTV 本体および関連ライブラリの作者・メンテナ向けに、利用者に見える不具合、実装責任箇所、検証結果、未確認範囲を整理します。技術背景は [REPORT.md](REPORT.md)、測定方法は [METHODOLOGY.md](METHODOLOGY.md)、公開結果は [results/](results/)、個別の議論は [Issue 一覧](https://github.com/libratechw/konomitv-experience/issues)を参照してください。

## PLAYBACK-LIVE-002 / LIVE-RESUME

### 利用者への影響

テレビの Live Original で一時停止して待っている間にプレイヤーが再構築されると、意図しない自動再生が始まる、または再生ボタンを押しても再開しないように見えることがあります。利用者が追加のタップや再読み込みを必要とする問題です。

### 実装責任と変更

KonomiTV の `PlayerController` / `PlayerStore` が、利用者による停止と内部処理の停止を区別して保持し、ライブの player/video 再構築や画質変更後も停止意図を引き継ぐようにしました。停止中は自動再生を抑制し、明示的な再生操作または手動復旧でだけ停止意図を解除します。再構築時にライブエッジへ追従する `currentTime` の変更自体は維持します。

- 変更コミット: `fb4d74b30b19f02d8ad6f93bca1d0e7928717cf5`
- オフライン: Node 20 の `live-pause-restart-contract.mjs` が pass
- 共通入口の有界実機観測: Windows Chrome、Mac Safari、Linux Chrome、Android POCO、Galaxy `SM-X930` で pause → hold → 1 回の入力による play → 進行を確認
- Galaxy の最新 run `20260915t103050253959z-galaxy-live-pause-hold10-resume`: 低遅延 ON、10.049 秒 hold、同一 route/timeOrigin、進行 10 サンプル、cleanup verified
- Linux Chrome の最新 run `20260915t105317881597z-linux-live-pause-hold10-resume`: 低遅延 ON、10.004 秒 hold、66 サンプル、MPEGTS/events/PSI HTTP 200、owned Chrome cleanup verified

iPad mini/Air の最新 Appium/WDA 試行は RemoteXPC 8111 の接続拒否、automation-mode timeout、`xcodebuild` code 65 によりタブ作成前に停止しました。これは測定環境のセットアップ阻害であり、製品再生失敗とは分類しません。

物理表示、音声、A/V 同期、微細なカクつき、広範な長時間品質は未確認です。低遅延 ON/OFF は Live 専用の条件です。

## DPlayer Live Original finite/non-negative sync guard

DPlayer の `sync()` が初期タイムラインから非有限値（`NaN` / `Infinity`）または負値を算出した場合に、`video.currentTime` setter へ渡さないガード候補です。iPad Air の同一条件 A/B で upstream の開始停止と candidate の進行を比較した証跡があります。

- 候補コミット: DPlayer `a937e92`
- 物理表示・音声・A/V 同期・広範な長期品質は別途未確認

## 継続中の確認

候補の採否は、対象コミットと配信 bundle の対応を確認したうえで、原因に対応する実機・回帰証拠と未確認範囲を分けて判断します。Dogfood の配備状態は各配備マニフェスト、詳細な raw evidence は非公開の運用結果を正本とします。
