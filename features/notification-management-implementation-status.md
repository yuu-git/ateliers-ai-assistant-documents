---
title: 通知管理・グローバル操作の実装状況
---

# 通知管理・グローバル操作の実装状況

2026-10-29。Phase 4 後の追加改修。WinUI 3 / MVVM / Worker / SQLite を維持し、既読・ブックマーク・手動タグとグローバル操作を追加した。チャット、モデル管理、ログ、RSS の最新10件取得、要約・Markdown 確定、保存先分離は維持する。

## 操作と表示範囲

続く [通知一覧のクイック操作](notification-quick-actions-implementation-status.md) で、☆／★ のアイコン化、単体削除と確認設定、補助情報行末尾の未読表示へ調整した。DB・Worker API・以下の検索範囲は維持する。検証は同改修で各72シナリオへ追加した。

| 操作 | 場所・動作 |
| --- | --- |
| Worker 開始・停止 | ナビゲーション下部を開く。設定画面と同じコマンド。開始受付と疎通確認、停止受付とプロセス終了確認を区別する |
| AI の即時一時停止・再開 | 同領域または設定画面。他の編集中設定は保存しない。RSS は継続し、進行中の要約はキャンセルして再開後に再試行する |
| 通知の既読化 | 一覧の記事ボタンを明示的に開いた場合だけ行う。初期選択・自動更新・戻る操作だけでは既読化しない |
| 未読へ戻す | 記事詳細の「記事の操作」内。操作結果を API で確認して反映する |
| ブックマーク | 一覧の独立した ☆/★ ボタン、または記事詳細。別の記事を選択せず、既読化もしない |
| 表示切替 | 一覧上部のフィルターを展開し「すべて／未読／ブックマーク」を選ぶ。情報源・タグと併用できる |
| タグを付ける | 記事詳細の「記事の操作」→タグ編集。候補を選ぶか名前を入力して追加。複数付与・関連の解除が可能 |
| タグで絞る | 記事のタグボタン、またはフィルターでタグ名を指定。「表示条件を解除」は情報源を維持する |
| 一括削除 | 管理メニューから現在の表示対象（最大100件）を複数選択。選択数に加えブックマーク付き・タグ付き記事の件数を確認する |

「すべて・タグ指定なし」は従来どおり **全体の最新100件を先に選び、その中で情報源を絞る**。「未読」「ブックマーク」またはタグ指定時は **DB 全表示対象通知へ情報源・状態・タグ条件を適用し、その結果の最新100件**を取得する。ページングはない。画面の範囲説明でこの違いと該当件数を表示する。

全体・ナビゲーション・情報源の未読件数は、100件上限とは独立して DB 全表示対象通知から集計する。削除済み情報源の通知も全体に含め、情報源選択肢は現在登録中のものだけとする。初回取得前は件数を未確認として扱い、接続失敗時は最後の集計を保持する。

## 日時・通知の成立・状態保持

- 初回取得日時は既存の `WebArticles.FetchedTicks`（UTC ticks）。公開日時、`WebNotifications.CreatedTicks`（通知確定時刻）と区別し、推測で上書きしない。DTO の `FetchedAt` をローカル日付へ変換し、「今日／昨日／yyyy/MM/dd」でグループ化する。同日内も初回取得の降順。
- 要約成功、Markdown ファイルの確定、DB の `Notified` と通知行の確定が完了した記事だけを表示・未読集計する。要約待ちは件数に含めない。
- `SourceUrl` は元記事、`FeedSourceUrl` は DB の取得元フィード URL。元記事のホストから情報源を推測しない。
- 未読・ブックマーク・タグは独立。更新は API 成功後に反映し、失敗時はエラーを表示する。記事を再度開いたときも既存の既読日時を更新しない。
- 同じ記事の行・グループを再利用し、メタデータだけの更新では Markdown を再描画しない。選択・情報源・フィルター・詳細の読取位置を保持する。古い問い合わせ結果は、新しい表示条件へ適用しない。
- 開いた記事が既読化やブックマーク解除で現在の一覧条件から外れても、その詳細は読める状態に保持する。別の記事・明示的な条件変更・削除で更新する。

## スキーマと移行

Migration: `20261029100645_NotificationReadingAndTags`。既存 DB を削除・再作成しない。

| 対象 | 追加内容 |
| --- | --- |
| `WebArticles` | `ReadAtTicks` nullable INTEGER（UTC）、`IsBookmarked` NOT NULL INTEGER（初期 false） |
| `WebTags` | `Id` 主キー、`Name`、`NormalizedName` 一意 |
| `WebArticleTags` | `ArticleId` + `TagId` 複合主キー、両外部キー、タグから記事を引く索引 |
| 記事の索引 | `FetchedTicks`、`Source/FetchedTicks`、`ReadAtTicks/FetchedTicks`、`IsBookmarked/FetchedTicks` |

承認済み方針に従い既存通知もすべて未読、ブックマーク OFF・タグなしで開始する。記事入力・要約・モデル・初回取得日時・通知確定日時・会話は変更しない。初回は未読数が100件を超える場合がある。

旧 Migration への Down は今回追加した状態・関連・索引を削除するため、既読・ブックマーク・タグは失われる。記事や会話は維持する。再適用後は未読・OFF・タグなしになる。実 SQLite の旧スキーマ→適用→Down→再適用で記事 ID・本文・初回取得日時の維持を確認した。通常起動では Down を実行しない。

Debug / Release の既存の Roaming・Local 分離は維持し、相互のデータを移行・コピーしない。移行テストは隔離した一時 DB だけを使用し、利用者 DB は変更していない。通常利用時の Migration は既存のプロセス間 DB ゲートで直列化する。

### タグと削除

タグ名は前後を Trim し NFC 正規化、内部空白を維持する。空白のみ・制御文字・64 UTF-16 コード単位超を拒否する。比較キーは大文字化し英字の大文字小文字を同一扱い、全角半角は区別する。表示名は最初の登録を再利用する。同じ記事への同じタグの重複は複合主キーで防ぐ。作成・関連更新はトランザクション内で行う。

記事削除では記事の既読・ブックマーク、通知、タグ関連を削除し、共有／未使用になったタグ定義は再利用用に残す。巡回履歴は残し、Markdown・重複判定キーの従来の削除処理を維持する。最新10件に含まれる同記事を再取得すると新 ID・未読・ブックマーク OFF・タグなしになる。

## API と責務

Desktop は `INotificationManagement` を `WorkerApiClient` 経由で呼び、Web DB の更新は Worker の `SqliteWebMonitoringStore` が担当する。既存の Bearer 認証、ループバック制限、タイムアウト、応答上限を維持する。

| API（`POST /api/v1/`） | 要求・応答 |
| --- | --- |
| `notifications/query` | `NotificationQuery`（情報源・All/Unread/Bookmarked・任意タグ）→ `NotificationPage`（最大100件・全未読数・情報源別未読数・条件一致数・タグ候補） |
| `notifications/read` | `SetArticleReadRequest` → 更新済み `ArticleMetadata` |
| `notifications/bookmark` | `SetArticleBookmarkRequest` → 更新済み `ArticleMetadata` |
| `notifications/tags/add` | `AddArticleTagRequest` → 更新済み `ArticleMetadata` |
| `notifications/tags/remove` | `RemoveArticleTagRequest` → 更新済み `ArticleMetadata` |
| `ai/pause` | `AiPauseRequest`（Paused）→ 保存済み `WorkerConfiguration`。他設定を変更しない |

一覧は件数集計、対象100件、対象通知・タグをまとめて取得する。通知全本文の全件読み込みや記事ごとのタグ問い合わせは行わない。タグ候補は最大500定義を表示するが、候補外も名前を入力して検索・再利用できる。旧 `GET notifications` は既存診断用に保持するが、新 UI は `notifications/query` を利用する。

Worker 実装版は **4.1.0**、プロトコル版は1。版一致確認を維持する。更新前のアプリで旧 Worker を停止してから更新後に開始する。Ollama サーバー停止は不要。旧アプリが使えない場合は、タスクマネージャーで対象環境の自分の Worker 実行ファイルを確認して終了する。

Worker の固定領域と設定画面は同じ ViewModel・コマンドを共有する。約5秒の既存同期を使い、専用ポーリングは増やさない。停止受付だけでは「停止中」にせず、単一起動ハンドルの開放まで「停止処理中」とする。20秒後も応答する場合は停止未完了を表示し再試行できる。接続不能とプロセス停止を区別し、状態不明時に AI の稼働を断定しない。

## 変更ファイル

| 層 | 主な追加・変更 |
| --- | --- |
| Application | `NotificationManagement.cs`（契約）、`TagNameRules.cs`、`WebContracts.cs`、`WorkerOperationContracts.cs`、`WorkerCoordinator.cs` |
| Infrastructure | `WebRows.cs`、`SqliteWebMonitoringStore.Notifications.cs` と本体、Migration 2ファイルと snapshot、`WorkerApiClient.Notifications.cs` と本体、`WorkerJsonContext.cs`、`WorkerConnectionFiles.cs` |
| Worker | `WorkerHost.cs`（認証済み API） |
| Desktop | `WebMonitoringView.xaml/.cs`、`WebMonitoringViewModel.cs` / `.Notifications.cs`、`WorkerViewModel.cs` / `.Status.cs`、`WorkerStatusView.xaml/.cs`、`WorkerOptionsView.xaml`、`MainWindow.xaml/.cs` |
| 検証 | `NotificationManagementTests.cs`、`NotificationViewModelTests.cs`、`WorkerTests.cs`、`PersistenceTests.cs`、テスト登録、`NotificationFeatureVerification.cs`、既存 DEBUG 診断・Worker/UI 検証スクリプト |

新規 NuGet 依存はない。通知フィルターと記事操作は展開領域へまとめ、狭幅の本文領域を確保した。参考画像にはない未読バッジ・日付見出し・独立したブックマーク・タグ・グローバルフッターを追加し、既存配色・レスポンシブ切替を維持する。削除ダイアログの行は操作ボタンを持たない専用テンプレートとし、選択中に記事を開く誤操作を防ぐ。

## 確認と制約

```powershell
dotnet build src/Ateliers.Ai.Assistant.Desktop -p:Platform=x64 -p:WindowsPackageType=None -p:WindowsAppSDKSelfContained=false
dotnet build src/Ateliers.Ai.Assistant.Desktop -c Release -p:Platform=x64 -p:WindowsPackageType=None -p:WindowsAppSDKSelfContained=false
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests -c Release
pwsh -File tests/Verify-WorkerDisplay.ps1 -Notifications
pwsh -File tests/Verify-UiRefactoring.ps1
```

| 検証 | 結果 |
| --- | --- |
| x64 Debug / Release・Unpackaged ビルド | 両方成功、エラー0。Worker の既存 NU1900 警告1件（NuGet 脆弱性情報の取得先へ接続できない）。監査を無効化して回避していない |
| 自動テスト | Debug / Release **各70シナリオ成功**。既存チャット・画像・Markdown・モデル管理・会話保存・Web・Worker の回帰を含む |
| 130通知＋2待機記事の実 SQLite | 全未読・情報源別集計、削除済み情報源、100件超の未読／ブックマーク／タグ、条件適用後100件、100件より古い記事の組合せ検索が成功 |
| 既存データ・移行 | 旧スキーマからの移行、Down と再適用、本文・ID・初回取得時刻保持、Debug/Release 分離が成功 |
| 独立 Worker＋実 XAML | 実記事ボタン・独立ブックマーク・タグ入力、未読バッジ・情報源集計・日付グループ、選択だけでは既読化しないこと、AI 即時操作と編集中設定の保持、停止・再起動後の ID と状態の復元が成功 |
| UI 閉鎖後・停止中 | 実 Desktop 終了後に Worker が生存し再取得すること、Worker 停止後の保存済みログ・独立チャットが利用できることを確認 |
| レイアウト・詳細・一括削除 | 幅480～1600相当の実画面、一覧／詳細の戻る、長文 Markdown の実スクロール保持、通知／履歴切替、削除対象範囲・選択とブックマーク1件／タグ1件の警告を確認 |

Worker 再起動を伴う検証中に、DEBUG 専用の60倍加速時計がプロセス開始ごとに巻き戻る問題を修正した。隔離テストルートの作成時刻を共通基点にする。再検証は UI 閉鎖後の巡回まで成功。通常利用の時計・巡回周期は変更していない。

画面画像とレイアウト検証結果はアプリ側の `artifacts/ui-refactoring/`（Git 対象外）へ出力する。確認環境の描画倍率は100%。実キーボードだけでの全経路操作、複数モニター・各 DPI 倍率、MSIX 配布後の確認、実 RSS＋実 Ollama の今回の追加機能 E2E は未実施。ネイティブ Button の Automation Invoke と実入力バインド、隔離された偽 RSS/Ollama を使った独立 Worker API で検証した。

表示は1問い合わせ100件、タグ候補500件。接続断中はキャッシュを読めるが更新・再集計には Worker が必要で、キャッシュ内の未読件数は最終集計となる。他クライアントが削除した開き済み記事の詳細がキャッシュに残る場合は、次の操作で削除済みを通知する。全件既読、ページング、タグ定義の改名・削除や孤立タグの自動削除は追加していない。
