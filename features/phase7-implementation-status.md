---
title: Phase 7 継続画面監視の実装・運用
---

# Phase 7 継続画面監視の実装・運用

**利用者確認・完了（2026-10-30）:** 機能の動作確認を受けて Phase 7 MVP を完了とした。その後の [複数画面監視基盤](multi-screen-monitoring.md) は MVP 完了後の拡張であり、現在の操作・ID API・版7.1.0はそちらを参照する。以下の単一監視・版7.0.0・93件の検証は Phase 7 導入時の記録として維持する。

2026-10-30。[実装指示](../ai-implementation/phase7-codex-implementation-instructions.md) と、利用者が承認した Worker 優先の構成に基づく。現在画像1枚による AI コメント MVP を実装・開発検証した。通常環境での利用者確認と、環境別の制約は以下の検証範囲と分ける。

## 採用構成と技術検証

既存 Worker が、独立した STA・Win32 メッセージループ・標準 `GraphicsCapturePicker` の所有者 HWND・WGC 対象・トレイを持つ。Desktop のプロセスやウィンドウをキャプチャの所有者にしない。標準 Picker の実選択、Worker 内のウィンドウ／ディスプレイ取得、実 `gemma3:4b` の Vision、Desktop 終了後の取得、再起動 Desktop の状態・コメント再表示が成立したため、本実装へ進んだ。専用 Capture Agent・Windows Service は追加していない。

Windows 固有の単発取得・画像正規化を `Ateliers.Ai.Assistant.Windows` に移し、Desktop の既存キャプチャアダプターと Worker から共有する。コードの名前空間は Windows SDK と衝突しない `Ateliers.Ai.Assistant.Platform.Windows`。Core / Application / Infrastructure は引き続き OS のキャプチャ API に依存しない。Worker の対象 TFM は `net8.0-windows10.0.19041.0` へ変更した。第三者 NuGet・DB Migration は追加していない。

## 操作と保持範囲

1. 旧 Worker を停止し、実装版 **7.0.0** の Desktop／Worker へ更新する。プロトコル版は1を維持し、実装版不一致は従来どおり接続を拒否する。
2. 設定画面で Worker を開始し、AI の一時停止を解除する。自動ロードを OFF にする場合は、利用するモデルを別途ロードしておく。
3. 「画面監視」の「対象を選択」で Worker が開く Windows 標準選択画面を操作する。対象はウィンドウかディスプレイ1件のみ。選択キャンセルは既存対象を維持する。
4. インストール済みモデル、プロンプト、周期を指定して保存し、「監視開始」を押す。開始時は直ちに一回試行する。初期周期60秒、選択肢10／30／60／120／300秒。モデルの能力は自動判定せず、非対応時は API エラーを表示する。
5. 取得時刻・モデル・生成状態を伴うコメントを既存 Markdown 表示で読む。Worker の状態取得は約5秒周期。表示層の安全化・ストリーミング描画集約・追従スクロールを再利用し、既存コメントのインスタンスと編集中入力を維持する。
6. Desktop を閉じても監視は続く。同じ Worker へ UI が再接続すれば、対象・状態・次回予定・コメントを再表示する。「監視停止」またはトレイ右クリックの「画面監視を停止」で停止できる。トレイは汎用アイコンと状態・停止だけで、対象名をツールチップへ出さない。表示不能時には監視を停止する。

既存 Roaming `worker-settings.json` の任意項目 `screenMonitor` にモデル・周期・プロンプトだけを保存する。旧 JSON の未指定時は初期値を使う。対象名・HWND・対象オブジェクト・稼働フラグ・画像・コメントは保存しない。起動時は常に停止・対象未選択で、コメントは空から始める。Debug／Release のデータ分離は維持する。

コメントは Worker メモリ内で最新200件、本文 UTF-16 の概算1 MiBまで、1件65536文字まで。古いコメントから除去する。対象再選択や監視再開で、それまでのコメントは上限内で残るが、Ollama に履歴として再送しない。監視画像は毎回 `CapturedImageStore` にセッション／対象 ID・実取得時刻で登録し、一枚のリースを取得して解析し、終了・中止・失敗で解放・削除する。PNG の長辺1536px以下等は Phase 6 と共通。Temp ディスクへの書込やプロセス間画像共有は行わない。

## 中止・負荷・モデル方針

| 契機 | 動作 |
| --- | --- |
| 解析が次回周期に間に合わない | 次回分をスキップ。処理を並列実行・蓄積しない |
| 他の AI 処理がモデル排他を保持 | 約1秒でその回を見送り、次の周期で試す。取得済み画像を待機列に置かない |
| AI 一時停止 | 取得と解析を中止し、周期取得も止める。Web 情報の取得は継続する |
| AI 再開 | 監視セッションが稼働中なら新しい一枚から再試行。過去キャプチャの一括処理はしない |
| 監視停止・対象消失・セッションロック | 取得と解析を中止。ロック解除だけで自動再開しない |
| API／キャプチャ失敗 | 部分コメントを状態付きで保持し、監視を停止。エラー確認後に明示的に再開する |
| Worker 終了・再起動 | 終了時に中止と期限付きモデル解放。再起動後は停止状態 |

従来のファイル排他をチャット・手動解析・要約・監視で共有する。毎回 `/api/ps` を確認し、モデル未ロードかつ `AllowLoad=false` ならキャプチャ前に待機状態とする。設定を勝手に変更しない。監視で使用したモデルだけを追跡し、停止時に `AllowUnload=true` なら最大4秒で自動アンロードを試みる。失敗は画面と内容を含まない診断ログへ通知する。`AllowUnload=false` の停止ではモデルを残し、後から設定変更しただけでその停止済みセッションのモデルを遡及して停止しない。

監視の生成は `keep_alive=-1`。同じ Worker による同じモデルの Web 要約も、監視が使用する間は保持を300秒へ短縮しない。非監視モデルの要約は従来の300秒・ロード方針・解放方針を維持する。Desktop のモデル変更・終了時の**自動**アンロードは、モデル排他内で同じユーザーの Debug／Release Worker の監視状態を確認し、使用中または状態不明なら保護する。利用者によるモデル管理画面の明示停止は従来どおり。明示停止後に監視が継続していれば、自動ロード設定に応じて次回ロードまたは待機する。

Ollama はモデルの利用元アプリを識別できないため、他アプリとの完全な共有判定はできない。`/api/ps` の全モデルを管理対象に取り込まず、一括停止しない。AI 一時停止は監視セッション自体の終了ではないため、使用モデルが保持される場合がある。

## 主な変更ファイル

| 配置 | 変更 |
| --- | --- |
| Application `ScreenAnalysis/ScreenMonitorContracts.cs`, `ScreenMonitorService.cs` | 設定・コメント・状態の契約、周期制御、取得・一枚解析・停止・モデル使用追跡 |
| Application `Worker/WorkerContracts.cs`, `WorkerCoordinator.cs` | 任意の監視設定、保存済み AI 方針と即時変更通知、版7.0.0 |
| Application `Models/ChatModelSession.cs` | 自動解放前に継続監視の使用モデルを確認 |
| Windows `Capture/`, `Images/` と新 csproj | Phase 6 の WGC・縮小処理を共有、Worker の STA／Picker／トレイ／ロック通知 |
| Infrastructure `Worker/WorkerApiClient.Screen.cs`, `WorkerJsonContext.cs` | 既存認証 API の監視クライアントと DTO JSON、保守的な自動解放確認 |
| Infrastructure `Chat/OllamaChatService.cs` | 共有中モデルの保持秒数をホストから指定可能にする |
| Worker `WorkerHost.cs`, csproj | Windows TFM、ホスト構成、監視 API、終了処理。独立した取得・要約ループは維持 |
| Desktop `Controls/ScreenMonitorView`, `ViewModels/ScreenMonitorViewModel.cs` | 開始・停止・入力・状態・コメント表示。WebView2 は表示層に限定 |
| Desktop `App.xaml.cs`, `MainWindow.xaml(.cs)`, `WorkerViewModel.cs` | 構成、ナビゲーション、既存5秒タイマー、監視設定だけの変更時も他画面の入力を保持 |
| Desktop `Capture/WindowsScreenCaptureService.cs`, `Images/ClipboardImageService.cs` | Windows 共通処理へ委譲し、既存プレビュー・貼り付け仕様を維持 |
| `ScreenMonitorTests.cs`, `UiRefactoringTests.cs`, `Verify-ScreenMonitor.ps1`, Debug 診断 | 方針・競合・再接続・実機検証。Release に診断 API は含めない |

## 確認方法と結果

Debug／Release の x64 Unpackaged ビルドは警告0・エラー0。両構成の全93件（従来89件＋監視4件）の自動テストが成功した。監視専用では周期スキップ、取得中・解析中の中止、自動ロード OFF、モデル使用保護、失敗した部分応答、保持期限、再接続、編集・古い更新の保護を確認する。既存設定画面では、監視設定だけの保存後も編集中入力を維持して保存できることを追加確認した。

`Verify-WorkerProcess.ps1`、`Verify-ScreenAnalysis.ps1`、`Verify-WorkerDisplay.ps1 -GitHub`、`Verify-UiRefactoring.ps1` が成功した。実 Worker の認証・RSS・永続待機・起動時 Temp 回収、既存手動画面解析・ウィンドウ／ディスプレイ取得・18回反復、実 WinUI の GitHub 取得・通知・未読・タグ・削除・ログ・レイアウト・詳細スクロール保持・オフラインのチャット独立性を回帰確認した。

```powershell
dotnet build src/Ateliers.Ai.Assistant.Desktop/Ateliers.Ai.Assistant.Desktop.csproj -p:Platform=x64 -p:WindowsPackageType=None
dotnet build src/Ateliers.Ai.Assistant.Desktop/Ateliers.Ai.Assistant.Desktop.csproj -c Release -p:Platform=x64 -p:WindowsPackageType=None
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests -c Release
pwsh -File tests/Verify-ScreenMonitor.ps1 -Model gemma3:4b
```

`Verify-ScreenMonitor` は専用の一時 AppData・実 Worker・実 Desktop を使う。標準 Picker の UI Automation は試験で新しく開いた画面と既知の検証ウィンドウだけを操作する。実モデルには検証用ウィンドウの画像だけを送り、ディスプレイ取得は画像を保存・送信せず画素数のみ確認する。既存 DB・巡回情報・起動登録は変更しない。通常ユーザーの画面やゲームを実 Vision へ送った試験とは区別する。

実機試験で、標準 Picker の対象選択、60秒周期の複数コメント、Desktop 終了後の次回取得、UI 再起動後の同一セッションとコメント復元、自動アンロード、AI 一時停止中の周期取得停止、新しい画像での再開、トレイメニューからの停止、対象消失、Worker 再起動後の停止・設定復元が成功した。再接続した実 MainWindow で本文と時刻の WebView2 DOM 反映を確認し、Desktop の管理対象に同じモデルを登録してから通常の非同期終了処理を通し、監視中モデルが保持されることも確認した。セッションロックは WTS メッセージの模擬通知による試験であり、利用者の Windows を実際にロックした試験ではない。

計測例は最初の Vision 7.4〜7.9秒、Worker working set 約129〜130 MiB、ディスプレイ取得後約222〜224 MiB・911〜923ハンドル、Ollama `/api/ps` のモデル VRAM 約2749 MiB。最小化した検証用ウィンドウの取得は約10.1秒でタイムアウトし、復元後は586×393pxで再取得できた。初期化とデコードの一時割り当てを含む短時間の値であり、増加分をすべてリークと判定せず、長時間運用や GPU 使用率を保証する測定とも扱わない。

## 残る制約と最終確認

- 一般ウィンドウ・ディスプレイの取得は開発環境で確認した。Factorio／FF14、排他フルスクリーン、HDR、複数 GPU・DPI、MSIX 配布・実サインイン、スリープ復帰、実 Windows ロックは環境別の最終確認を残す。
- 最小化・保護コンテンツでは取得不能、停止画面、黒い画像になる場合がある。取得タイムアウト等の失敗では停止するが、正常な画像と黒画面の意味的な判別は今回実装しない。
- 標準 Picker は OS の別ウィンドウで動く。中止・Worker 終了で結果は破棄するが、OS の選択画面が残った場合は「キャンセル」で閉じる。選択が完了しただけで監視を自動開始することはない。
- UI の状態表示は約5秒周期で、短い応答は完了内容としてまとめて見える場合がある。Worker 停止中は直前の表示を残して状態不明と表示する。
- コメントの永続化、画像差分、複数画像同時解析、音声生成、MCP 連携、長時間運用の保証は今回の対象外。初期 MVP の技術的な成立と、利用者による最終完了確認を区別する。
