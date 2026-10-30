---
title: Ateliers AI Assistant ドキュメント
---

# Ateliers AI Assistant

Windows 上で動作する、自分専用のローカル AI アシスタントを長期的に育てるプロジェクト。

初期の目的は **WinUI 3 から Ollama と会話できるチャットアプリ**を完成させること。将来的には Web 情報収集、画面監視、音声通知、Markdown 生成、既存 MCP ツールによる動画生成まで拡張する。

**現在の計画（2026-10-30）:** Phase 6 の手動キャプチャ・Vision 解析まで利用者確認を受けて完了。Phase 6〜7 を支える横断改善として、アプリ専用 Temp 管理と複数時点画像のメモリ保持基盤を追加した。Phase 7 の継続画面監視・AI コメントは暫定計画とする。Phase 7 完了をアプリケーションの初期 MVP 到達点とし、情報収集と画面アシスタントの二本柱を満たす。音声・MCP 動画生成・AI 管理コンソール等は時期未定。範囲と未確認事項は [ロードマップ](roadmap.md) を参照する。

## ドキュメント一覧

- [プロジェクト概要・要件](requirements.md)
- [アーキテクチャ](architecture.md)
- [ソリューション・リポジトリ構成](repository-structure.md)
- [初期 AI チャット機能](features/ai-chat.md)
- [AI チャットの実装状況と制約](features/ai-chat-implementation-status.md)
- [Ollama モデル管理](features/model-management.md)
- [Markdown 回答表示](features/markdown-display.md)
- [チャットへの画像貼り付け](features/image-paste.md)
- [Web 情報収集](features/web-monitoring.md)
- [Web 情報収集の実装状況と制約](features/web-monitoring-implementation-status.md)
- [Phase 5 GitHub Releases の実装状況と制約](features/github-releases-implementation-status.md)
- [通知管理・既読・ブックマーク・タグ・グローバル操作](features/notification-management-implementation-status.md)
- [通知一覧のクイック操作・単体削除と確認設定](features/notification-quick-actions-implementation-status.md)
- [通知の閲覧終了と既読化・手動未読指定の優先](features/notification-reading-behavior.md)
- [画面監視・ゲームアシスタント](features/screen-monitoring.md)
- [アプリ専用 Temp 管理・複数時点画像の使用権](features/temp-management.md)
- [音声・動画生成連携](features/media-integration.md)
- [AI 管理コンソール構想（未実装・時期未定）](features/ai-management-console-concept.md)
- [データとメッセージの設計](data-design.md)
- [アプリケーションデータ保存方針](data-storage-policy.md)
- [Phase 2 実装指示と確認済み補足](ai-implementation/phase2-codex-implementation-instructions.md)
- [Phase 3 実装指示](ai-implementation/phase3-codex-implementation-instructions.md)
- [開発ロードマップと完了条件](roadmap.md)
- [開発・テスト方針](development-guidelines.md)
- [未決定事項と技術検証](open-questions.md)

## 文書の読み方

Phase 4 の現在の動作、起動登録、API・ログ・モデル利用の方針と確認手順は [常駐 Worker の実装・運用](features/background-worker.md) を参照する。[Phase 4 実装指示](ai-implementation/phase4-codex-implementation-instructions.md) からの確定事項も記録している。

「決定」は現在の採用方針、「候補」は実装前に比較する選択肢、「将来構想」は実装を約束しない拡張案を示す。サンプルのクラス名・ディレクトリ名は設計案であり、固定 API ではない。

## 原則

- 個人利用が最優先。公開・OSS 化は将来必要になった場合に判断する。
- AI に任せるのは主に意味理解・要約・文章生成。取得、差分、スケジュール、永続化は決定的な C# 処理を優先する。
- UI、ホスト、アプリケーションロジック、外部連携を分離する。
- 最初から全機能を作らず、実際に動くマイルストーンを積み上げる。
- 実装・検証で判明した事実を文書に反映する。
