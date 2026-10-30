---
title: アーキテクチャ
---

# アーキテクチャ

## 採用方針

- **UI**: WinUI 3 + XAML + MVVM（決定）。Blazor は将来必要になれば別 UI として追加可能。
- **常駐**: Phase 4 は現在ユーザーの独立 Worker プロセス。Windows Service は採用しない。UI 終了後も巡回し、サインイン時起動は明示 opt-in。
- **プロセス間通信**: 認証付き loopback HTTP API。UI は約5秒のポーリングで状態を更新する。
- **保存**: Phase 2 の保存済み会話は SQLite + EF Core。Markdown 原文・表示状態・下書きと画像 BLOB を保存する。将来の情報収集結果の Markdown ファイル出力は別責務とする。
- **AI**: Ollama と既存 C# 通信コード。Vision 対応モデルは画面監視段階で検証。
- **開発環境**: Aspire は必須としない。複数プロセスの開発時起動・観測が必要になれば検討。

## 実行時の境界

MVP 完了後の複数監視は、既存 Worker の `MultiScreenMonitorService` が不変 MonitorId ごとの対象・状態・取消・コメントと全体1件の実行枠を所有する。既存 STA／トレイは共有し、`CaptureWindowHost` の取得スコープを監視ごとに分離する。モデル保護は全監視集合、画像ストアと本文容量は Worker 全体で共有する。Desktop は一覧と選択監視の詳細を取得し、画像は IPC で渡さない。専用 Agent・追加 DB は導入しない。移行・ID API・公平性・実処理終了待ちは [複数画面監視基盤](features/multi-screen-monitoring.md) を現在仕様とする。以下の Phase 7 単一監視は導入時の採用経緯を含む。

Phase 6 完了後の横断改善として、Application の `Storage` に時系列画像・使用権と Temp 契約、Infrastructure の `Storage/Temp` にファイル管理を追加した。UI／Worker の Host が起動・終了清掃へ接続する。所有者はプロセス起動ごとに分離し、プロセスをまたぐデータ共有は実装しない。FS の安全性は `ITempFileSystem` に分離し、既存のパス・診断ログを利用する。詳細は [Temp 管理](features/temp-management.md) を参照する。

```text
[WinUI 3 Desktop / ユーザーセッション]
  - チャット・通知・設定 UI
  - 手動画面キャプチャ・Vision 解析 UI (Phase 6)
  - スピーカー再生 (将来)
            |
      認証付き localhost API
            |
[Worker + API Host / ユーザーセッション]
  - Web 巡回・スケジューリング
  - 差分検知・AI 解析・通知生成
  - データ永続化・Markdown 出力
  - STA・標準 Picker・WGC による継続画面監視、Vision コメント、トレイ (Phase 7)
            |                  |
         [SQLite]           [Ollama]
            |
     [Markdown files]
```

Phase 4 の Worker はユーザーセッション内で動き、サインアウト中の動作は保証しない。Phase 6 の手動キャプチャ・画像解析は Desktop 側へ実装し、既存 Ollama 接続を直接使う。図の音声再生は未実装の候補配置。詳細は [Worker の実装・運用](features/background-worker.md) を参照する。

## Phase 6 の実装と Phase 7 の境界

Phase 6 は利用者が対象を選択する手動キャプチャ・プレビュー・Vision 解析・結果表示に限定する。Desktop の `WindowsScreenCaptureService` が WGC／標準 Picker／D3D を扱い、Application の `IScreenCaptureService` で UI 非依存の画像データを渡す。`ScreenAnalysisService` は既存 `IChatService` と `IModelWorkCoordinator`、`ChatModelSession` の排他・ライフサイクルを再利用する。`ScreenAnalysisViewModel` が操作状態を保持し、`ScreenAnalysisView` が同じ PNG をプレビューする。新規プロジェクト・DB 変更はない。

解析は画像・プロンプト一回分だけを送り、会話履歴や DB へ追加しない。結果表示は既存 `ChatTranscriptView` を再利用する。AI 一時停止中・Worker 状態不明時は解析しない方針を採用し、排他取得後にも API で確認する。UI の既存状態更新で停止・未接続を検知した場合も中止を通知する。詳細な制約と未確認事項は [画面監視](features/screen-monitoring.md) を参照する。

Phase 7 は既存 Worker が独立 STA・所有者 HWND・標準 Picker・WGC 対象・トレイを持ち、取得と現在画像1枚の Vision を実行する。Desktop 終了後も継続し、認証 API の約5秒ポーリングで状態とメモリ内コメントを再表示する。専用 Agent は追加しない。`ScreenMonitorService` は Application のユースケース、Windows 固有処理は新しい `Ateliers.Ai.Assistant.Windows` プロジェクトへ分離し、Phase 6 の Desktop アダプターも同じ取得・正規化を使う。Phase 6 導入時の配置を移動するもので、手動操作の仕様は維持する。

モデル排他は従来どおり共有する。自動ロード・自動アンロードは既存 Worker 設定に従い、監視中モデルの保持期限を要約で短縮しない。Desktop の自動解放前には稼働中 Worker の監視モデルを確認する。監視は解析中の周期をスキップし、AI 一時停止では取得も中止する。RSS 等の取得継続とは区別する。ロック・対象消失・Worker 終了で安全に止め、Worker 再起動は停止状態とする。画像は登録・一枚リース・解放後削除とし、画像・コメントの永続化や IPC 画像転送は行わない。設定・検証と制約は [Phase 7 実装記録](features/phase7-implementation-status.md) を正とする。

音声出力は両 Phase の対象外で、Web 情報収集と画面監視の双方から利用できる時期未定の基盤候補とする。MCP 動画生成連携も旧 Phase 7 から時期未定へ移す。初期 MVP の到達点は [ロードマップ](roadmap.md#マイルストーン-ateliers-ai-assistant-初期-mvp) を参照する。

## 論理的な責務

- **Domain/Core**: 情報源、記事、会話、通知などのモデルとドメイン規則。UI・DB・Ollama への依存なし。
- **Application**: チャット、巡回、解析、通知などのユースケースと必要なポート（インターフェース）。
- **Infrastructure**: Ollama、HTTP、RSS、SQLite、Markdown、外部 API 等のアダプター。
- **Hosts**: Desktop、Worker、API の起動・DI・設定・ライフサイクル。
- **Features**: WebMonitoring、ScreenMonitoring、Speech 等の機能群。論理的区分であり必ずしも個別 csproj ではない。

依存方向は Domain ← Application ← ホスト／アダプターを基本とする。Infrastructure は Application が定義する契約を実装し、Host で DI 接続する。Desktop と Worker は互いの ViewModel や実装クラスを直接参照しない。

Phase 1 のモデル管理では、Application の `ChatModelSession` がチャットのポートを包み、管理対象の追跡・排他・期限付き終了を担当する。Infrastructure は `IModelControlService` を実装する。これは単一チャット用であり、将来の複数機能による共有モデルの参照数管理は導入していない。詳細は [モデル管理](features/model-management.md) を参照。

Phase 2 の永続化は Application の `IConversationStore`、`IUserSettingsStore`、`IApplicationPaths`、`IAppDiagnostics` を境界とし、Infrastructure が SQLite / JSON / 固定パス / ローテーションログを実装する。Desktop の ViewModel は DB・ファイル・Explorer を直接扱わない。SQLite I/O は背景処理で直列化し、会話全体をトランザクションで置き換える。保存先とユーザー単位の所有範囲は [保存方針](data-storage-policy.md) に従う。

`AppDataPaths` は通常、Infrastructure のビルド構成から `AppDataEnvironment` を決定する。ホストと依存プロジェクトは同じ Debug / Release 構成でビルドする。Roaming / Local ともに Debug は `Ai-Assistant.Debug`、Release は既存の `Ai-Assistant` を使う。環境の表示情報も同じ `IApplicationPaths` から取得し、UI と保存処理の判断を重複させない。

終了時は、生成の中止待ちと未保存内容の確認・保存を先に完了する。キャンセル時はウィンドウとメモリ内容を維持し、その後にだけ既存の期限付きモデル解放と終了を行う。EF Core のリフレクション・Migration を保持するため、Phase 2 の Release は trimming を無効にする。

## 共通処理と固有処理

Phase 4 後の UI 整理では、通知閲覧と設定表示を Desktop のコントロールへ分離する。情報源別の通知表示は既存 DB に保存された取得元を `ArticleNotification.FeedSourceUrl` として応答へ含め、記事 URL の `SourceUrl` と区別する。新しい Migration や収集・要約処理の変更は行わない。詳細は [UI 整理の実装状況](ui-design/implementation-status.md) を参照する。

続く通知管理の追加改修では `INotificationManagement` と Worker API を追加し、既読・ブックマーク・タグ用の Migration を適用した。Web の状態変更は引き続き Worker 側の SQLite ストアへ集約し、Desktop は API 結果を表示する。日付グループ・選択・Markdown は Desktop の表示責務とする。ナビゲーションの固定領域は設定画面と同じ Worker ViewModel・コマンド・約5秒のポーリングを共有し、追加の監視ループを作らない。[通知管理の境界・API・移行](features/notification-management-implementation-status.md) を参照する。

Phase 3 の `WebCollectionService` は手動収集の実装として残す。通常起動の Phase 4 は Application の `WorkerCoordinator` が取得と AI を別ループで実行する。Infrastructure の `RssFeedSource` / `SqliteWebMonitoringStore` / `ArticleMarkdownWriter` は再利用し、要約処理は `ArticleSummary` に共通化する。Desktop の Web 用 ViewModel は `IWorkerClient` 経由で操作し、Web の DB 更新・Markdown 書込は Worker に集約する。会話保存と Web 保存は DB 単位のプロセス間ファイルゲートを共有し、Migration の競合も防ぐ。

Phase 4 は `IModelUseGate` を UI と Worker のモデル利用に導入し、同一ユーザーの Ollama 操作をプロセス間で調整する。Worker 自動ロード・解放の方針と Desktop のチャット管理は別責務とする。利用元を識別できない外部アプリとの共有判定や参照数管理は未導入。

```text
[RSS/API/Web] ----> [取得・差分検出] ----┐
                                      ├--> [AI 解析] --> [メッセージ] --> [通知 / Markdown / 音声]
[Screen Capture] -> [画像前処理] ------┘
```

共通化候補はモデル呼び出し、キャンセル・排他制御、メッセージ保存、通知、ログ、音声生成の契約。Web の差分検出と画面の変化検出は別機能として扱い、過度に汎用化しない。

## AI と決定的処理の分担

C# がスケジュール、HTTP 取得、記事 ID・ハッシュ比較、再試行、ファイル出力を担当する。LLM は意味理解、分類、要約、重要度の参考判定、コメント・通知文生成を担当する。LLM 出力は構造化・検証して保存し、原文と判断を混同しない。

## Phase 5 の情報源拡張

RSS / Atom と GitHub Releases で異なる取得・重複／差分判定を扱い、既存 Worker の定期巡回・手動実行から共通の要約・通知・記事管理へ接続する。情報源固有の取得と識別の処理を分け、UI へ外部 API や DB 更新の詳細を持ち込まない。既存の未読・既読、ブックマーク、タグ、削除、保存先分離を維持する。

Application の `IContentSource` を取得境界にし、既存 `IFeedSource` は RSS の互換ポートとして継承する。Infrastructure の `ContentSourceRouter` が `RssFeedSource` と `GitHubReleasesSource` を振り分け、Worker が設定に基づき呼び出す。HTTP/JSON は Infrastructure に閉じ、既存の後段とユーザーセッション常駐を維持する。GitHub は30秒・8 MiBの上限と安全なエラー分類を持つ。

種別・プレリリース設定は Worker 設定 JSON、API待機期限は既存状態 JSON、リリースメタデータは既存記事入力 JSON に追加する。DB Migration・通知 ID の変更はない。型の任意フィールドで旧 RSS 設定・記事を読み、UI／Worker の同版確認は維持する。将来の汎用プラグイン基盤や MCP / Tool Calling は追加しない。[Phase 5 の実装・制限](features/github-releases-implementation-status.md) と [確認済み事項](open-questions.md#phase-5-で確認済み) を参照する。

汎用 Web 監視は指定ページの変更検出、AI 情報収集・推薦は関心テーマからの情報発見・選別、AI 管理コンソールは既存業務操作の自然言語による呼び出しとして、異なる責務の後続構想に分ける。推薦では一次選別とローカル LLM による二次評価の分担を検討するが、プロセス配置・連携・実装順序は未定。Factorio MOD Portal も後続の専用 API 型情報源候補とする。

## AI 管理コンソールとの境界（将来構想）

[AI 管理コンソール構想](features/ai-management-console-concept.md) は、モデル検証用チャットとは別の操作入口として検討する。通常 UI と管理コンソールは、同じ Application Service の業務操作・入力検証・整合性保証を利用する。ViewModel や LLM が DB を直接更新する構成にはしない。

LLM は操作と引数の候補を提示し、アプリケーション側が承認・検証・認可・実行制限を適用する。内部サービスの直接呼び出しと外部 MCP 経由の呼び出しの両方で、この実行制御を通す。成功・失敗はサービスの構造化された実行結果から判断する。

現在の `IChatService` / `OllamaChatService` は原文・画像の送信と回答本文の受信を扱い、ツール定義・ツール呼び出し・結果の再投入は実装していない。管理指示のオーケストレーションや必要な通信契約は着手時に検討し、現段階で変更しない。MCP は任意の接続アダプター候補とし、未使用の基盤を先に作らない。複数機能によるモデル共有が必要になった場合は、単一チャット用のライフサイクル管理も見直す。

## セキュリティ・運用

localhost のみにバインドし、不要な外部公開をしない。localhost でも呼び出し元制御や危険操作の許可設計を検討する。外部コンテンツは信頼できない入力として扱い、LLM の指示文として無条件に実行しない。取得先の利用規約、レート制限、著作権に配慮する。
