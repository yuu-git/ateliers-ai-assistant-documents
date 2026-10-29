---
title: UI 設計資料
---

# UI 設計資料

このディレクトリは **Ateliers AI Assistant の WinUI 3 UI に関する基準資料**です。既存機能・データ・通信契約を維持したまま、画面の情報設計と視認性を改善するために使用します。

## 読む順番

改修後の配置、元画像との相違、確認結果は [UI 整理の実装状況](implementation-status.md) に記録する。

初回の UI 整理に続き、[通知管理の追加改修](../features/notification-management-implementation-status.md) を実装した。既読・ブックマーク・タグと固定 Worker フッターは追加指示で承認された範囲であり、当初の「新機能追加・Migration は対象外」という制限をこの範囲だけ更新する。

1. [UI 設計原則](ui-design-principles.md)
2. [画面構成と遷移](navigation-and-screens.md)
3. [画面別仕様](screen-specifications.md)
4. [画像の見方](references/README.md)
5. [Codex 向け UI 改修指示](../ai-implementation/ui-refactoring-codex-instructions.md)

## 画像

- `references/reference-notification-original.png` — **通知画面の最重要デザイン参考**。過去に作成したイメージをそのまま収録。画像内の「Blazor」への言及は採用技術を意味しない。
- `references/current-*.png` — 2026-10-29 時点の現状確認用。変更前の状態を示す。
- `mockups/*.png` — 改善後の**構成・配置**を示す参考画像。色・フォント・寸法を厳密に固定するものではない。
- `mockups/*.svg` — 同じ構成の編集可能な図。VS Code で確認可能。

## 優先順位

1. ナビゲーションを「機能」「システム・管理」に分離
2. Web 情報収集を「通知閲覧中心」に再設計
3. 設定をカテゴリ別に整理
4. モデル管理・ログ・チャットは現状維持を原則とし、必要最低限の整合調整のみ

**今回の UI 改修は Phase 4 の既存機能を再実装する作業ではない。** 新機能追加、Blazor 移行、データベース再設計はスコープ外。
