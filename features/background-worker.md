---
title: Phase 4 常駐 Worker の実装・運用
---

# Phase 4 常駐 Worker の実装・運用

**MVP 完了後の追加:** 現在の実装版は **0.7.0**。0.7.0 は [アイコン・製品情報の仕上げ](branding-and-release.md) で、以下の複数監視の導入版は7.1.0。監視は最大4件の定義を既存 Worker が所有し、取得・Vision を全体1件へ直列化する。監視ID API、全監視のモデル保護と自動解放、トレイ全停止、旧設定移行を追加した。一般の設定 API で監視一覧を直接上書きせず、監視別Revisionで変更を調整する。既存の Web 取得・要約・認証・保存仕様は維持する。[複数監視の実装と検証](multi-screen-monitoring.md) を現在仕様とし、以下の版番号は各導入時の記録として読む。

**Phase 7 の追加:** 現在の UI／Worker 実装版は **7.0.0**。既存 Worker が独立 STA・標準 Picker・WGC・トレイを持ち、継続画面監視と Vision コメントを担当する。Web 取得・要約ループ、起動登録、データ保存仕様は維持する。監視では AI 一時停止中にキャプチャも止めるが、RSS 等の情報取得は続ける。対象・コメント・画像は永続化せず、Worker 再起動時は監視停止。操作とモデル共有・環境別制約は [Phase 7 実装記録](phase7-implementation-status.md) を参照する。以下の版番号は過去フェーズ導入時の記録。

**Phase 5 の追加:** [GitHub Releases](github-releases-implementation-status.md) を既存の取得・AIループへ接続した。導入時の UI／Worker 実装版は **5.0.0**。既存版は停止してから更新後に開始する。ユーザーセッション、起動登録、モデル方針、保存先分離は維持し、GitHub のレート制限時は API 全体を待機させて RSS と保存済み記事の要約は継続する。以下の Phase 4 導入記録の版番号は当時のもの。

Phase 4 では Web 情報収集を現在ユーザーの独立 Worker に移した。Windows Service は導入していない。サインアウト中・PC の電源 OFF 中は動作せず、スリープ中の巡回も保証しない。チャットは引き続き Desktop が担当する。

Phase 4 後の [UI 整理](../ui-design/implementation-status.md) により、操作入口は「通知・情報収集」と設定の3カテゴリへ再配置した。通知 DTO の `SourceUrl` は元記事 URL、`FeedSourceUrl` は既存 DB の取得元フィード URL。続く [通知管理の追加改修](notification-management-implementation-status.md) で既読・ブックマーク・タグ用 API、日付表示、固定フッターを追加した。実装版 **4.1.0** への更新時は旧 Worker を明示停止して開始し直す。

「すべて・タグなし」の情報源別表示は全体最新100件内を対象とし、未読／ブックマーク／タグ指定は DB 全通知に条件を適用した最新100件とする。未読集計は全表示対象通知から行う。固定フッターと設定画面は同じ開始・停止・AI即時操作コマンドを使い、既存約5秒の状態同期を共有する。停止受付だけで終了完了と表示しない。AI即時操作は未保存のモデル・情報源設定を保存せず、一時停止だけを Worker へ反映する。

## 利用方法

1. アプリと Ollama を起動し、設定画面の「Worker を開始」で Worker を起動する。UI の終了では Worker を停止しない。明示停止は設定画面から行う。
2. Web 情報収集画面で情報源を追加し、名称、公開 RSS/Atom URL、巡回周期、自動巡回を保存する。周期は **5〜10080分、初期60分**。保存すると次回予定を再計算し、再起動なしで反映する。最終取得・次回予定・状態・エラーを一覧表示する。
3. 自動巡回 OFF でも手動取得できる。取得は最新10件まで。情報源単位の重複実行は抑止し、複数情報源の取得も逐次実行する。
4. 設定画面で要約モデルと AI 方針を保存する。AI 一時停止は進行中の要約も中止するが、RSS 取得は継続する。未処理件数には失敗・処理中の記事も含む。再開時は未完了段階から処理する。
5. サインイン時起動は利用者が明示的に有効化する。初期状態で登録しない。Windows のスタートアップ設定で無効化・制限された場合は、その状態を画面で確認する。

Worker が停止している場合もチャット、保存済み会話、ローカルのログ閲覧は利用できる。通知と巡回履歴の再取得・変更には Worker 接続が必要。接続状態、最終疎通、実装版、AI 状態、直近エラーを UI に表示し、接続状態を約5秒ごとに更新する。

### 起動・配布

- Desktop のビルド・発行時に `Worker/` を同梱する。Desktop と同じ Debug / Release 構成でビルドする。Worker は現在 .NET 8 / ASP.NET Core 8 のフレームワーク依存実行ファイルであり、配布先にも対応する **ASP.NET Core Runtime 8（x64）** が必要。Desktop の Windows App Runtime と WebView2 の前提条件も従来どおり。
- 非パッケージ版: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` に固定の Worker 実行ファイルを引用符付きで登録する。値名は Release `AteliersAiWorker`、Debug `AteliersAiWorker.Debug`。解除操作は該当する値だけを削除する。配置先を移動した場合は新しい場所から再登録する。
- MSIX 版: 同梱マニフェストの `StartupTask`（`AteliersAiWorker`、初期無効）を使う。Windows が利用者・ポリシーによる無効状態を返す場合、アプリから強制的には解除しない。
- Debug のパッケージ ID は `Ateliers.Ai.Assistant.Debug`、Release は従来の ID を維持する。設定・DB・エンドポイント・起動登録を環境別に分離する。
- 同じ環境の Worker はロックファイルで1プロセスに限定し、重複起動は終了する。異常終了で OS がハンドルを解放すれば再起動可能。UI からの手動開始は可能だが、クラッシュ後の自動プロセス再起動や更新インストーラーは今回実装していない。

新しい第三者 NuGet パッケージは追加せず、Worker は ASP.NET Core の共有フレームワークを利用する。Release の trimming 無効は維持する。開発環境では Debug / Release ビルドと Release の非パッケージ発行、Debug MSIX の生成・Worker 同梱を確認した。NuGet の脆弱性情報へ接続できない `NU1900`、MSIX のシンボル生成用 `mspdbcmf.exe` 不在の警告は環境由来として残り、チェックを無効化していない。

## データ・復旧

主な実装配置:

| 層・クラス | 責務 |
| --- | --- |
| Application `WorkerCoordinator` / `WorkerContracts` | 巡回予定、取得・AI の分離、設定変更、状態・API 契約、削除調停 |
| Application `ArticleSummary` / `Logging` | 共通要約、構造化ログの契約 |
| Infrastructure `WorkerStateStore` / `SqliteWebMonitoringStore.Worker` | 版付き設定、永続待機と巡回状態 |
| Infrastructure `WorkerApiClient` / `WorkerConnectionFiles` / `FileOperationGate` | 認証通信、秘密情報・単一起動、プロセス間排他 |
| Infrastructure `FileDiagnosticLog` / `FileAppLogReader` | 構造化書込、ローテーション、オフライン読取 |
| Worker `WorkerHost` | loopback API、独立実行ループ、終了調停 |
| Desktop `WorkerViewModel` / `WorkerConsoleView` / `WorkerOptionsView` / `WorkerLauncher` | 情報源・AI 設定、次回予定、手動開始・停止、起動登録 |
| Desktop `LogsViewModel` / `MainWindow` | ログ画面、明示コピー、画面統合 |

既存の `ChatModelSession` には共有モデルゲート、会話・Web ストアには DB のプロセス間ゲートを追加する。`WebMonitoringViewModel` は API アダプターを利用し、定期更新時も変更のない通知行と選択を維持して Markdown の読取位置をリセットしない。

| データ | 保存先・所有者 |
| --- | --- |
| チャット設定 | Roaming `settings.json`、Desktop |
| 情報源・周期・モデル・AI 方針 | Roaming `worker-settings.json`、Worker が版付きで更新 |
| 記事・通知・巡回・処理中フラグ・再試行時刻・情報源状態 | Local `data/assistant.db`、Web 更新は Worker |
| 要約 Markdown | Local `web-monitoring/articles/{記事ID}.md`、Worker |
| API 接続情報 | Local `worker/endpoint.json`、起動ごとに更新する秘密トークンを含む |
| 診断ログ | Local `logs/ui.log` / `worker.log` と各 `.1` / `.2` |

Roaming / Local の基点は Release `Ateliers-Dev/Ai-Assistant`、Debug `Ateliers-Dev/Ai-Assistant.Debug`。既存 Phase 3 の URL・要約モデルは Worker 設定がまだない初回に1情報源として取り込み、自動巡回 OFF で開始する。壊れた設定を初期値で上書きしない。

Migration `WorkerQueueAndSchedule` は既存の記事・通知・会話を残し、記事の処理状態と情報源状態テーブルを追加する。Desktop / Worker の DB 操作・Migration は同じファイルゲートで調整する。チャットの保存処理は Desktop に残す。

取得と AI を別ループにし、取得時点で記事を永続化する。AI 待機記事は最新10件の範囲から外れても処理対象に残る。要約成功後は要約保存、Markdown の一時ファイルからの確定配置、通知登録の順に進み、段階ごとの再実行で修復する。通知は記事 ID で一意。再起動時に処理中フラグを回復し、受理済みの未完了取得要求も再実行する。

起動・スリープ復帰時に期限超過した情報源は1回だけ追いつき実行し、取り逃した周期を連続再生しない。RSS 失敗は次の巡回で再試行する。AI 失敗は記事単位の待機時間を設け、短時間の無限ループを避ける。

情報源削除は今後の取得・待機 AI 処理を止める操作であり、過去の記事・通知・Markdown・巡回履歴を削除しない。同じ URL を再登録すると既存の重複判定を引き継ぐ。通知側の記事削除は従来どおり複数選択可能。記事を削除し、その後の最新10件に含まれれば新しい記事 ID で再取得される。非表示リストは持たない。

## モデル利用

- AI 一時停止 OFF、自動ロード OFF、自動アンロード OFF が初期値。
- 自動ロード OFF は `/api/ps` でモデルがロード済みの場合だけ要約する。未ロードの場合にチャット API を送って暗黙ロードさせない。
- 自動ロード ON の Worker はロード要求と生成の双方で `keep_alive: 300` を使う。自動アンロード OFF は明示停止しないという意味であり、Ollama の保持期限による解放を禁止する意味ではない。UI チャットの無期限保持は従来どおり。
- 自動アンロード ON でも、今回の処理で未ロードからロードし、実際のロードを確認したモデルだけを処理後に停止する。先に他の用途でロードされていたモデルは停止しない。
- UI チャット・モデル操作と Worker の AI 処理は同一ユーザーのファイルゲートで調整する。Debug / Release が同じ Ollama を使うため、モデルゲートだけは両環境で共通にする。DB・設定・トークンは共通にしない。
- Worker 使用中の UI モデル操作は短い待機の後に案内を表示する。チャットを優先したい場合は Worker の AI を一時停止する。UI 終了時も Worker が使用中ならそのモデルを停止しない。
- Ollama 自体は起動・停止しない。Ollama API は利用元アプリを識別できず、外部アプリとの競合を完全には防げない。参照数を持つ全アプリ共通のモデル所有権管理は対象外。

## localhost API

Kestrel は `127.0.0.1` の動的ポートだけにバインドする。API の基点は `/api/v1/`。UI は各要求で現在の接続情報を読み、Worker 再起動後のポート・トークン変更に追従する。公開 API としての互換性は今回保証しない。

| メソッド・パス | 用途 |
| --- | --- |
| `GET status` | 設定・情報源状態・待機数・AI 状態・版 |
| `PUT configuration` | 楽観的な版チェック付き設定更新 |
| `POST fetch` | 情報源 ID・要求 ID による取得受付 |
| `GET runs` / `GET notifications` | 保存済み結果の参照 |
| `POST articles/delete` | 1〜100記事の削除 |
| `POST shutdown` | Worker の明示停止 |

手動取得は要求 ID を永続化して再送を判定する。書込が通信切断で成功不明になっても、クライアントは無条件の自動再送をしない。設定競合時は明示再読込してから保存する。

安全対策:

- ユーザー専用フォルダーに接続情報を置き、ACL を現在ユーザーと SYSTEM に限定する。各要求でランダム Bearer トークンを定時間比較する。
- 実際の loopback の Host とポートを検証する。Origin または Sec-Fetch ヘッダーのある要求を拒否し、変更要求は JSON に限定する。CORS の許可は返さない。
- 要求本文256KiB、応答受信8MiB、通常8秒・記事削除30秒のクライアント期限、API 契約版のチェックを設ける。
- 情報源は最大50件、名称120文字、URL4096文字、周期5〜10080分。既存の RSS 取得側で公開 HTTP(S) のみ許可し、DNS 解決・リダイレクト先にもプライベート IP 等の拒否とサイズ・時間制限を適用する。
- Web 原文はツール実行権限を持たない要約用の外部データとして渡す。チャット履歴とは混ぜない。通知の Markdown は既存の安全化・ローカル WebView2 表示を維持する。
- 同じユーザー権限で動く悪意あるプロセスまで隔離するセキュリティ境界ではない。トークンのコピー・公開・ログ出力は禁止。

設計上の根拠: [Kestrel のエンドポイント](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/endpoints)、[StartupTask](https://learn.microsoft.com/uwp/api/windows.applicationmodel.startuptask)。

## ログ

「アプリケーション」「システム」は本アプリ内の分類であり、Windows イベントログの読み取りではない。Info / Warning / Error、期間、キーワードで絞り込み、最新最大500件を表示する。手動更新、複数選択コピー、表示一覧コピー、詳細コピー、固定ログフォルダーを開く操作を提供する。Worker が停止しても UI がファイルを読み取れる。

UI / Worker 別の JSON Lines に日時・分類・レベル・発生元・操作・相関 ID・制限した説明を記録する。1ファイル2MiB、現行と過去2世代。書込をプロセス間で調整し、末尾の不完全行・壊れた行・ローテーションを読み取り側で許容する。旧 `assistant.log` も互換読み取りする。新規例外ログは型と HResult に限定し、本文・画像・認証情報・外部エラー本文を保存しない。

## 確認と残る制約

開発用テストは一意の一時領域と偽 RSS / Ollama を使い、通常のユーザーデータや実モデルを変更しない。Debug のみ隔離検証用引数を持ち、Release では受け付けない。

```powershell
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests -c Release
powershell -File tests/Verify-WorkerProcess.ps1
powershell -File tests/Verify-WorkerDisplay.ps1
```

自動検証には RSS 継続・AI 一時停止、最新10件外の待機処理、周期変更・追いつき1回・要求再送、処理中クラッシュからの復旧、モデル方針、ファイル失敗後の修復、情報源削除・再登録、記事再取得、ログ分類・コピー・秘密除外を含む。実プロセス検証では API の認証・Origin・Host・単一起動・再起動時のトークン更新を確認し、実 WinUI 検証では画面から設定した周期・次回予定、UI 終了後の Worker の取得継続、Worker 停止後のログとチャットを確認する。

Debug / Release とも全62シナリオが成功。追加で実ログの書込・ローテーション中の読取、最大500件・3世代の上限、不完全行の無視、Phase 3 の実データを保持した Migration、DB ロック後に受理済み取得要求が失われないこと、Worker のロードと生成で保持時間が一致することも確認した。既存の実 WinUI Web 表示検証では通知の Markdown・C# ハイライト・安全化・再起動復元を確認した。

サインアウト／サインイン、再起動、実スリープ復帰による OS の自動起動は利用者環境での最終確認が必要。自動テストで現在の利用者をサインアウトさせたり、通常アプリの起動登録を勝手に有効化したりしない。設定画面で有効化→Windows のスタートアップ設定を確認→サインインし直す→UI を開く前から Worker が動作していることを確認→不要なら登録解除、の順で確認する。スリープ確認では次回予定を過ぎて復帰し、履歴が追いつき1回分だけ増えることを確認する。

大量履歴のページング・検索、自動プロセス再起動、OS トースト、サービス化、トークン認証の他ユーザーへの委譲は対象外。最新一覧は通知100件・ログ500件とし、蓄積 DB の全体保持期間や自動削除は導入していない。
