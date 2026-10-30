---
title: データとメッセージの設計
---

# データとメッセージの設計

## 保存方針

通知の単体削除確認は既存の Roaming `settings.json` の `ConfirmSingleArticleDeletion`（既定 true）へ統合する。削除成功後に確認省略を保存し、設定保存失敗では前の確認方針を維持する。DB・Migration・Worker API への追加はない。[クイック操作](features/notification-quick-actions-implementation-status.md) を参照する。

Phase 2 の採用方針は [アプリケーションデータ保存方針](data-storage-policy.md) を正とする。小容量のユーザー設定は Roaming の JSON、明示保存した会話は Local の SQLite / EF Core、診断ログは Local のローテーションファイルに分ける。設定を SQLite に格納する方針ではない。

Phase 3 の実行履歴・記事入力と処理段階・通知は SQLite に保存する。生成 Markdown は通常ファイルとし、DB に Local 基点の相対パスを記録する。音声・動画は将来機能とする。

## Phase 2 の会話スキーマ

Phase 4 は記事に `Processing` / `RetryAfterTicks`、DB に `WorkerSourceStates`（情報源 ID と状態 JSON）を追加する。永続待機は既存記事の未完了段階から判定し、最新10件の取得範囲とは分離する。情報源削除でも記事と URL 単位の重複判定を残す。Roaming の版付き `worker-settings.json` に設定を保存し、Web 更新は Worker が担当する。DB と Migration の排他はプロセス間で共通化する。詳細は [Worker のデータ・復旧](features/background-worker.md#データ復旧) を参照。

| テーブル | 主な情報 | 関連・制約 |
| --- | --- | --- |
| Conversations | Id、Title、作成・更新 UTC ticks、選択モデル、下書き本文、Version | 明示保存時に作成。Version で上書き競合を検出 |
| Messages | Id、ThreadId、Position、Role、Content、Model、StatusText、作成 UTC ticks、IncludedInHistory | 会話内の順序は一意。原文と失敗・中止状態を保持 |
| Images | Id、ThreadId、任意の MessageId、PngBytes、Width、Height | PNG は BLOB。1メッセージ1枚、MessageId が null の下書きも会話に1枚 |

会話・メッセージの削除は外部キーで画像までカスケードする。保存時は同じトランザクションで本文・画像・下書きを置き換え、失敗時は以前の内容を維持する。新規の空会話は作らず、明示保存後に下書きだけを空にした既存スレッドは同じスレッドとして保持する。HTML・DOM・元の未縮小画像は保存しない。

`IncludedInHistory` は完了した送受信だけ true にする。表示状態の文字列から履歴を推測せず、復元後も中止・失敗ターンを送信対象から除外する。スキーマ変更は EF Core Migration で管理し、通常起動で DB を削除・再作成しない。

## 概念モデル（案）

### Phase 3 の追加スキーマ

Migration `20261028204923_WebMonitoring` は会話テーブルを変更せず、以下を追加する。

| テーブル | 主な情報 | 制約・目的 |
| --- | --- | --- |
| WebArticles | Id、Key、Source、Payload、FetchedTicks、Summary、Model、MarkdownPath、Notified、Error | Key が一意。入力 JSON と成功済み要約を保持し、段階別に再試行 |
| WebCollectionRuns | Id、StartedTicks、Payload | 情報源・モデル・開始終了・状態・件数・エラーを持つ実行 DTO の JSON。StartedTicks でソート |
| WebNotifications | ArticleId、CreatedTicks | ArticleId が主キーかつ WebArticles への外部キー。記事ごと最大1件 |

Markdown 出力後に保存パス・Notified・通知行を同じ DB トランザクションで確定する。ファイルと DB をまたぐ失敗は、安定した内部 ID のファイル名と通知一意制約によって冪等に再試行する。詳細は [実装状況](features/web-monitoring-implementation-status.md) を参照。

### Phase 4 後の通知管理スキーマ

Migration `20261029100645_NotificationReadingAndTags` で `WebArticles.ReadAtTicks`（nullable、UTC ticks）、`IsBookmarked`（初期 false）、`WebTags`（Id/Name/NormalizedName 一意）、`WebArticleTags`（ArticleId/TagId 複合主キー・外部キー）を追加した。既存の `FetchedTicks` を初回取得日時として表示し、公開日時・通知確定日時とは区別する。

既存通知は承認済み方針に従い未読・ブックマーク OFF・タグなしへ移行する。要約・記事入力・初回取得日時は変更しない。通知確定済み記事だけを全件集計し、未読／ブックマーク／タグ検索は DB 条件適用後の最新100件とする。「すべて・タグなし」は全体最新100件を先に適用する。

記事削除で通知・タグ関連をカスケード削除し、共有／孤立タグ定義は再利用用に残す。タグは Trim＋NFC、最大64 UTF-16 コード単位、内部空白保持、英字大小同一・全半角区別。Web 更新の所有者は Worker のままで、既存のプロセス間 DB ゲートを利用する。移行と Down 時のメタデータ消失、索引・API・確認結果は [通知管理の実装状況](features/notification-management-implementation-status.md) を参照する。

以降は将来の構想を含む概念案である。

| 概念 | 主な属性 |
| --- | --- |
| ContentSource | Id, Name, Kind, Endpoint, Interval, Enabled |
| CollectionRun | Id, SourceId, StartedAt, FinishedAt, Status, Error |
| CollectedItem | Id, SourceId, ExternalId, Url, Title, PublishedAt, ContentHash |
| ItemRevision | Id, ItemId, FetchedAt, ContentRef, ContentHash |
| AnalysisResult | Id, ItemRevisionId, ModelId, Summary, Category, Importance, Evidence |
| Notification | Id, Kind, Title, Body, CreatedAt, ReadAt, RelatedResourceId |
| Conversation | Id, Title, CreatedAt, UpdatedAt |
| ChatMessage | Id, ConversationId, Role, Content, CreatedAt, ModelId |
| GeneratedArtifact | Id, Kind, Path, SourceRef, CreatedAt |

上記は概念レベルであり、DB テーブル・キー・カラムの確定仕様ではない。

## チャットと通知

Phase 5 は DB スキーマと既存記事・通知 ID を変更しない。`SourceConfiguration.Kind` と `IncludePrereleases` を既存 Worker 設定 JSON へ、`SourceState.GitHubRetryAt` を `WorkerSourceStates` の JSON へ、任意の `GitHubReleaseMetadata` を `WebArticles.Payload` へ保持する。旧設定は RSS / Atom、旧記事はリリースメタデータなしとして読む。取得タグは手動タグテーブルへ自動登録しない。新規 Migration はない。[採用契約と互換性](features/github-releases-implementation-status.md#契約保存既存データ) を参照する。

共通の時系列表示コンポーネントは検討できるが、チャットと通知のドメインモデルを最初から同一化しない。チャットは会話文脈と Role、通知は発生源・重要度・既読・参照先を持つ。表示用 DTO で統合できる。

## AI の構造化結果例

```json
{
  "category": "Factorio",
  "importance": "High",
  "title": "MOD の更新を検出",
  "summary": "互換性に影響する変更の可能性があります",
  "breakingChange": "Possible",
  "sourceUrls": ["https://example.com/release"]
}
```

この JSON は設計例。実装時にスキーマ・列挙値・必須項目を定義し、LLM 出力を検証する。`breakingChange` は推定と事実を区別し、根拠を参照可能にする。

## 同時実行

将来 Desktop と Worker が別プロセスになるため、SQLite への書き込みは Worker/API 側へ集約することを基本案とする。Phase 2 では Desktop ホストから Infrastructure のストアを使うが、Application 契約で差し替え可能にする。トランザクション、Migration、会話保存の上書き競合検出は Phase 2 で導入する。サービスのデータ所有範囲・バックアップ方式は引き続き後続で検討する。
