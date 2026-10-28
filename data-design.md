---
title: データとメッセージの設計
---

# データとメッセージの設計

## 保存方針

Phase 2 の採用方針は [アプリケーションデータ保存方針](data-storage-policy.md) を正とする。小容量のユーザー設定は Roaming の JSON、明示保存した会話は Local の SQLite / EF Core、診断ログは Local のローテーションファイルに分ける。設定を SQLite に格納する方針ではない。

将来の実行履歴・記事メタデータ・通知は SQLite の候補とする。生成 Markdown と音声・動画は通常のファイルとして保存し、DB に相対パスや参照情報を記録する。これらの将来データは Phase 2 では実装しない。

## Phase 2 の会話スキーマ

| テーブル | 主な情報 | 関連・制約 |
| --- | --- | --- |
| Conversations | Id、Title、作成・更新 UTC ticks、選択モデル、下書き本文、Version | 明示保存時に作成。Version で上書き競合を検出 |
| Messages | Id、ThreadId、Position、Role、Content、Model、StatusText、作成 UTC ticks、IncludedInHistory | 会話内の順序は一意。原文と失敗・中止状態を保持 |
| Images | Id、ThreadId、任意の MessageId、PngBytes、Width、Height | PNG は BLOB。1メッセージ1枚、MessageId が null の下書きも会話に1枚 |

会話・メッセージの削除は外部キーで画像までカスケードする。保存時は同じトランザクションで本文・画像・下書きを置き換え、失敗時は以前の内容を維持する。新規の空会話は作らず、明示保存後に下書きだけを空にした既存スレッドは同じスレッドとして保持する。HTML・DOM・元の未縮小画像は保存しない。

`IncludedInHistory` は完了した送受信だけ true にする。表示状態の文字列から履歴を推測せず、復元後も中止・失敗ターンを送信対象から除外する。スキーマ変更は EF Core Migration で管理し、通常起動で DB を削除・再作成しない。

## 概念モデル（案）

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
