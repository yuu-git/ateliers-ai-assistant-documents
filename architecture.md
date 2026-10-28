---
title: アーキテクチャ
---

# アーキテクチャ

## 採用方針

- **UI**: WinUI 3 + XAML + MVVM（決定）。Blazor は将来必要になれば別 UI として追加可能。
- **常駐**: .NET Worker Service を Windows Service として運用（情報収集の段階で導入）。
- **プロセス間通信**: localhost HTTP API、必要なら SignalR（候補）。
- **保存**: Phase 2 の保存済み会話は SQLite + EF Core。Markdown 原文・表示状態・下書きと画像 BLOB を保存する。将来の情報収集結果の Markdown ファイル出力は別責務とする。
- **AI**: Ollama と既存 C# 通信コード。Vision 対応モデルは画面監視段階で検証。
- **開発環境**: Aspire は必須としない。複数プロセスの開発時起動・観測が必要になれば検討。

## 実行時の境界

```text
[WinUI 3 Desktop / ユーザーセッション]
  - チャット・通知・設定 UI
  - 画面キャプチャ (将来)
  - スピーカー再生 (将来)
            |
      localhost API / events
            |
[Worker + API Host / Windows Service]
  - Web 巡回・スケジューリング
  - 差分検知・AI 解析・通知生成
  - データ永続化・Markdown 出力
            |                  |
         [SQLite]           [Ollama]
            |
     [Markdown files]
```

Windows Service は非対話セッションで動くため、デスクトップ画面の取得や対話的な音声再生を担当させない。UI を閉じても画面監視・音声再生を続ける必要が出た場合は、ログイン中ユーザーの **Desktop Agent** を独立プロセス化するか、ウィンドウを閉じても常駐する Desktop ホストを設ける。Windows Service だけではユーザーセッションの機能を代替できない。

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

```text
[RSS/API/Web] ----> [取得・差分検出] ----┐
                                      ├--> [AI 解析] --> [メッセージ] --> [通知 / Markdown / 音声]
[Screen Capture] -> [画像前処理] ------┘
```

共通化候補はモデル呼び出し、キャンセル・排他制御、メッセージ保存、通知、ログ、音声生成の契約。Web の差分検出と画面の変化検出は別機能として扱い、過度に汎用化しない。

## AI と決定的処理の分担

C# がスケジュール、HTTP 取得、記事 ID・ハッシュ比較、再試行、ファイル出力を担当する。LLM は意味理解、分類、要約、重要度の参考判定、コメント・通知文生成を担当する。LLM 出力は構造化・検証して保存し、原文と判断を混同しない。

## AI 管理コンソールとの境界（将来構想）

[AI 管理コンソール構想](features/ai-management-console-concept.md) は、モデル検証用チャットとは別の操作入口として検討する。通常 UI と管理コンソールは、同じ Application Service の業務操作・入力検証・整合性保証を利用する。ViewModel や LLM が DB を直接更新する構成にはしない。

LLM は操作と引数の候補を提示し、アプリケーション側が承認・検証・認可・実行制限を適用する。内部サービスの直接呼び出しと外部 MCP 経由の呼び出しの両方で、この実行制御を通す。成功・失敗はサービスの構造化された実行結果から判断する。

現在の `IChatService` / `OllamaChatService` は原文・画像の送信と回答本文の受信を扱い、ツール定義・ツール呼び出し・結果の再投入は実装していない。管理指示のオーケストレーションや必要な通信契約は着手時に検討し、現段階で変更しない。MCP は任意の接続アダプター候補とし、未使用の基盤を先に作らない。複数機能によるモデル共有が必要になった場合は、単一チャット用のライフサイクル管理も見直す。

## セキュリティ・運用

localhost のみにバインドし、不要な外部公開をしない。localhost でも呼び出し元制御や危険操作の許可設計を検討する。外部コンテンツは信頼できない入力として扱い、LLM の指示文として無条件に実行しない。取得先の利用規約、レート制限、著作権に配慮する。
