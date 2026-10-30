---
title: Phase 5 GitHub Releases の実装状況と制約
---

# Phase 5: 情報源拡張基盤・GitHub Releases 対応

2026-10-30。[実装指示](../ai-implementation/phase5-codex-implementation-instructions.md) に従い、調査計画の承認後に実装した。RSS / Atom を維持し、公開 GitHub Releases を同じ Worker の巡回・要約・通知・記事管理へ接続する。MCP / Tool Calling、一般 Web、Factorio、検索・推薦は追加していない。

**完了確認（2026-10-30）:** 利用者による機能確認と、未読一覧で詳細が消える最終不具合の修正確認を受け、Phase 5 は完了とする。[通知の閲覧終了と既読化](notification-reading-behavior.md) に最終の動作と検証範囲を記録した。記事を閲覧中は未読を維持し、手動未読を優先する。行は枠・背景だけで強調する。以下の各79件は GitHub 追加時点の記録で、最終修正後は全80件、Debug／Release ビルドと実 WinUI の回帰検証に成功した。実 API＋実 Ollama の通し検証等の未確認事項は本書末尾のとおり残す。

## 操作・更新手順

1. 更新前のアプリで旧 Worker を停止する。更新後の UI／Worker は実装版 **5.0.0**、プロトコル版は1。版不一致時は接続を拒否するため、旧アプリを利用できない場合はタスクマネージャーで自分の対象環境の Worker 実行ファイルを確認して終了する。Ollama サーバーは停止しない。
2. 「通知・情報収集」→「情報源設定」→「新しい情報源」。種別を「GitHub Releases」にする。
3. 表示名と `owner/repository`（例: `microsoft/vscode`）を入力する。周期は従来どおり初期60分・5〜10080分。自動取得は必要時に有効化する。
4. 必要なら「プレリリースを含める」を ON にして保存。既定 OFF、draft は常に除外。RSS では同項目は無効。
5. 情報源を選んで手動巡回するか、自動予定を待つ。AI 一時停止中も取得・記事登録は進み、モデル未ロード／自動ロード OFF の場合は要約待ちになる。
6. 要約完了後に通知・Markdown を確認する。元記事リンク、未読・既読、ブックマーク、手動タグ、単体／一括削除は従来どおり利用できる。

設定を編集中に定期更新や遅い保存が完了しても、新しいリポジトリ・種別・プレリリース入力を不必要に上書きしない。新しいメイン画面は追加していない。

## 採用仕様

| 項目 | 動作 |
| --- | --- |
| 対象・認証 | `github.com` の公開リポジトリ。認証なし。トークン保存・非公開・Enterprise は対象外 |
| 入力と取得元 | `owner/repository` を ASCII の許可文字で検証し小文字化。保存キーは `https://api.github.com/repos/{owner}/{repo}/releases`。表示名とは別 |
| HTTP | User-Agent、GitHub JSON Accept、API バージョン `2026-03-10`。30秒、展開後8 MiB上限、接続10秒。RSS の2 MiB制限は変更しない |
| ページ・件数 | 初回・以後とも `per_page=100&page=1` だけ取得。draft と任意の prerelease を除外し、公開日時降順で最大10件。100件を超える遡り・ページングはしない |
| 本文 | API 提供の本文のみ。最大12,000文字に制限し、切り詰めを入力範囲に明記する。空本文は空のまま、空タイトルはリリースタグへフォールバック |
| リンク・資産 | 記事リンクは GitHub の HTTPS `html_url`。HTML ページ・添付資産・リポジトリ本文を追加取得しない |
| 識別 | `github-releases:{正規化リポジトリ}` と `github-release:{Release ID}` を既存のハッシュ規則へ渡す。別情報源・同名タグ・RSS と衝突させない |
| 編集 | 同じキーの既存入力・要約・通知を上書きせず、再要約・再通知しない。記事の初回取得日時・既読・ブックマーク・タグも維持 |
| 日時 | API `published_at` は公開日時。欠落は不明のまま。アプリの UTC 初回取得日時を別保存し、従来どおりローカル日付でグループ化 |
| 取得タグ | Release ID・リポジトリ・`tag_name`・draft/prerelease をメタデータに保持。利用者の手動タグ定義へ自動登録しない |
| 取得成功0件 | 正常な空配列や条件に合う対象0件。HTTP失敗・不正JSONは「新着なし」にしない |
| ETag | 初期実装では利用しない。304と保存キャッシュ・削除後再取得の整合は後続検討 |

先頭100件の範囲で選別するため、正式版が少なければ10件未満になる。API 全履歴に対する公開日時上位10件を保証するものではない。リポジトリ名変更・移管や別名間の履歴自動統合は行わず、取得元を変更しても過去記事を残す。同一 API ホスト内の HTTP リダイレクトは最大5回、HTTPS・標準ポート・資格情報なしに限定し、常に先頭ページの件数条件を維持する。

## 失敗・レート制限

- 404は「存在しない／公開されていない」、通常403はアクセス拒否、429・制限情報付き403はレート制限として表示する。5xx、ネットワーク断、タイムアウト、不正 JSON、応答上限超過も取得失敗として巡回履歴へ残す。
- 二次制限を示す403の JSON `message` は最大4 KiBだけを内部判定に使い、外部本文をログ・画面へ転送しない。
- `Retry-After` と `X-RateLimit-Reset` の遅い期限を尊重する。期限なしは5分、過去の期限等でも少なくとも1分待つ。
- 制限期限は既存 `SourceState` JSON に保持する。登録・削除済み情報源の状態から GitHub 全体の待機を復元し、UI／Worker の再起動や制限された情報源の削除でも迂回しない。
- 待機中は GitHub の自動取得を見送り、手動要求も理由・解除予定付きで拒否する。RSS の取得と保存済み記事の AI 要約は継続する。自動の次回予定は通常周期と制限解除の遅い方を表示する。
- 既存 localhost API の400応答は、安全な業務エラーを最大2 KiBで Desktop に表示する。一般障害の応答は従来どおり共通文言とし、内部例外を公開しない。

認証なしの一次制限は IP 単位で毎時60リクエスト。他のアプリや情報源数・手動実行も消費に影響するため、初期60分から必要に応じて長い周期にする。[公式の制限](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)、[再試行指針](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api#handle-rate-limit-errors-appropriately) に従う。

## 契約・保存・既存データ

| 対象 | 追加・互換性 |
| --- | --- |
| `SourceConfiguration` | `Kind`（省略0＝RSS / Atom）と `IncludePrereleases`（省略 false）。既存ソース ID・URL・周期を維持 |
| `SourceState` | 任意の `GitHubRetryAt`。既存 `WorkerSourceStates` の JSON に保存 |
| `FeedArticle` | 任意の `GitHubReleaseMetadata`。既存 `WebArticles.Payload` JSON に保存 |
| `ArticleNotification` | 同じ任意メタデータを追加し、詳細・保存 Markdown に表示 |

**DB Migration・新規カラム・新規テーブルは不要。** 既存記事・通知 ID、RSS の重複キー、会話、画像を再作成・移行しない。設定は既存の Roaming `worker-settings.json`、記事・通知・予定は Local SQLite のまま。Debug / Release 分離・原子的 JSON 保存・DB 排他を維持する。

`SourceUrl` は元記事、`FeedSourceUrl` は取得元の正規化 API URL／フィード URL。既存の名称を維持し、元記事 URL から情報源を推測しない。記事削除は同じ Worker API とストアを通し、Markdown・通知・入力・重複キーを削除、巡回履歴・共有／孤立タグ定義を保持する。次回対象に残るリリースは新 ID・未読・ブックマーク OFF・タグなしで取得できる。

## 実装の境界と変更ファイル

`IContentSource` が情報源設定を受け取る取得ポート。既存 `IFeedSource` は RSS 取得と既存テストの互換アダプターとして継承する。`ContentSourceRouter` は2種の振り分けだけを担当し、プラグイン登録基盤を持たない。HTTP/JSON は Infrastructure、巡回・待機・要約は Application の既存 Worker、入力と表示は Desktop に分離する。

| 主なファイル（`src/Ateliers.Ai.Assistant.*` 以下） | 役割 |
| --- | --- |
| Application `WebMonitoring/ContentSources.cs` | 種別・取得ポート・GitHub入力正規化・メタデータ・安全な失敗 |
| Application `WebMonitoring/WebContracts.cs` / `ArticleSummary.cs` | 任意メタデータと要約入力。ツール実行は追加しない |
| Application `Worker/WorkerContracts.cs` / `WorkerCoordinator.cs` | 互換DTO、共通取得、GitHub全体待機と復旧、実装版更新 |
| Infrastructure `WebMonitoring/GitHubReleasesSource.cs` | API取得・上限・DTO・リダイレクト・対象選別・エラー分類・振り分け |
| Infrastructure `WebMonitoring/SqliteWebMonitoringStore.cs` / `.Notifications.cs` / `ArticleMarkdownWriter.cs` | JSONからのメタデータ復元、詳細・Markdownへの投影 |
| Infrastructure `Worker/WorkerApiClient.cs` | 上限付き400理由の表示 |
| Worker `WorkerHost.cs` / `VerificationAdapters.cs` | 実サービスの接続、隔離された偽HTTPの実プロセス診断 |
| Desktop `ViewModels/WorkerViewModel.cs` / `.Sources.cs` / `.Status.cs` | 種別・リポジトリ・プレリリース編集と新しい入力の保持、共通の取得状態表示 |
| Desktop `ViewModels/WebMonitoringViewModel.cs` | 取得種別・Release ID・リリースタグの詳細表示 |
| Desktop `Controls/WorkerConsoleView.xaml/.cs` / `WorkerOptionsView.xaml` / `WorkerStatusView.xaml` | 既存ダイアログの入力追加、RSS専用だった文言を情報取得へ統一 |
| Desktop `App.xaml.cs` / `MainWindow.xaml.cs` / `Controls/WebMonitoringView.xaml.cs` / `Diagnostics/WorkerDisplayVerification.cs` | DEBUG実画面検証のGitHubモード |
| テスト | `GitHubReleaseTests.cs`、`Program.cs`、`UiRefactoringTests.cs`、`Verify-WorkerDisplay.ps1` |

新規プロジェクト・NuGet 依存はない。`WorkerStateStore`・JSON source generation・既存スキーマは新しい任意フィールドをそのまま扱い、保存方式は変えていない。

## 確認方法

```powershell
dotnet build src/Ateliers.Ai.Assistant.Desktop -p:Platform=x64 -p:WindowsPackageType=None -p:WindowsAppSDKSelfContained=false
dotnet build src/Ateliers.Ai.Assistant.Desktop -c Release -p:Platform=x64 -p:WindowsPackageType=None -p:WindowsAppSDKSelfContained=false
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests -c Release
pwsh -File tests/Verify-WorkerDisplay.ps1 -GitHub
pwsh -File tests/Verify-WorkerDisplay.ps1 -Notifications
pwsh -File tests/Verify-UiRefactoring.ps1
```

明示的な実 API の取得確認（1ページ取得・永続化／Ollama操作なし）:

```powershell
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests -- --github-repository microsoft/vscode
```

## 確認結果と残る制約

| 検証 | 結果 |
| --- | --- |
| x64 Debug／Release Unpackaged | 両ビルド成功・エラー0。既存の NU1900 警告1件（NuGet脆弱性情報取得先への接続失敗）は継続。監査を無効化して回避していない |
| 自動テスト | 両構成 **各79シナリオ成功**。既存72件に GitHub の7件を追加。HTTP契約・対象選別・入力・識別、各HTTP障害・JSON・上限・中止、リダイレクト、Workerから通知まで、共有制限・復元、設定互換、編集保持・安全なAPI理由表示を扱う |
| 実 SQLite／JSON | 旧 RSS 設定・記事 ID を保持。同ID編集で再要約・通知せず、整理情報を維持。削除後は新ID・初期状態へ戻る。制限された情報源の削除・Worker再起動でも共有待機を復元し、RSSは継続 |
| 実 WinUI＋独立 Worker | `-GitHub` と `-Notifications` の両方成功。GitHub の種別・owner/repository・プレリリース入力、通常のHTTP/JSON解析を通した要約・通知・タグ・ブックマーク・既読、実装版更新後の復元を確認 |
| UI・既存機能 | 複数幅、情報源／記事選択、Markdownとスクロール、設定編集、一括・単体削除と確認設定の回帰に成功。UI終了後の実 Worker 継続巡回、停止中のログ・独立チャットも確認 |
| 実 GitHub API | `microsoft/vscode` を認証なしで1回取得し正式版10件の選別に成功。DB・設定・実 Ollama は触れていない |

設定更新と取得完了の状態保存は直列化し、古い保存で GitHub 全体の待機期限を巻き戻さない。待機期限は GitHub の設定だけに依存させず、既存の情報源状態 JSON に共有状態として保存・復元する。

実 API＋実 Ollama を組み合わせた通常アプリの E2E、実サインイン／スリープ復帰、MSIX配布、各DPI・テーマでの全経路は未実施。実画面検証は100%倍率と隔離された偽HTTP／Ollamaを使用する。実運用では旧 Worker の停止後に新版を開始し、要約モデルの設定・ロード方針、GitHub情報源の取得、2回目の重複抑止、削除後の再取得を確認する。

認証なしのIP制限、先頭100件からの選別、12,000文字の入力制限、編集無視、改名・移管の自動統合なし、ETag／追加ページなしは採用したMVPの制約。取得範囲外の取り逃しを無制限に補完する機能や、非公開情報源を扱う機能は追加していない。
