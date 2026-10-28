---
title: データとメッセージの設計
---

# データとメッセージの設計

## 保存方針

SQLite を設定・実行履歴・記事メタデータ・通知・会話の候補 DB とする。生成 Markdown と音声・動画は通常のファイルとして保存し、DB に相対パスや参照情報を記録する。取得した原文・正規化済み本文は、差分判定や再解析に必要な範囲で保持する。

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

将来 Desktop と Worker が別プロセスになるため、SQLite への書き込みは Worker/API 側へ集約することを基本案とする。初期チャット段階では直接アクセスも許容するが、Repository/サービス契約で差し替え可能にする。トランザクション、重複防止、マイグレーション、バックアップを後続で検討する。
