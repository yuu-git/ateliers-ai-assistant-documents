---
title: Codex 向け UI 改修指示
---

# Codex 向け UI 改修指示（Phase 4 後の UI 整理）

## 依頼内容

Ateliers AI Assistant の WinUI 3 デスクトップ UI を、既存の動作を保ったまま情報設計から整理してください。現在は機能追加が先行し、設定や履歴、通知が一画面に混在しています。配色変更や機能追加ではなく、**ナビゲーション・Web 情報収集・設定の3箇所**を優先して改善します。

## 最初に読む資料

- `ateliers-ai-assistant-documents/ui-design/README.md`
- `ateliers-ai-assistant-documents/ui-design/ui-design-principles.md`
- `ateliers-ai-assistant-documents/ui-design/navigation-and-screens.md`
- `ateliers-ai-assistant-documents/ui-design/screen-specifications.md`
- `ateliers-ai-assistant-documents/ui-design/references/reference-notification-original.png`
- `ateliers-ai-assistant-documents/ui-design/mockups/notification-layout.png`
- `ateliers-ai-assistant-documents/ui-design/mockups/settings-layout.png`
- 既存の `features/web-monitoring*.md`、`features/background-worker.md`、`features/model-management.md`、`architecture.md`、`development-guidelines.md` 等（存在するもの）

## 事前調査（コードを変更する前）

1. 現在の NavigationView/ページ構成、Web 情報収集ページ、設定ページ、ViewModel、サービス/API 呼び出しを調査。
2. 各画面の現行操作と状態を列挙し、変更後にどこへ移るかの**機能対応表**を提示。
3. 既存の XAML/WinUI コンポーネントを活用した変更案と、変更ファイル一覧を提示。
4. 既存 API・永続化・Worker の契約変更が必要に見える場合は、独断で変更せず相談。
5. **まず計画と機能対応表を提示し、ユーザーの確認を待つ。**

## 実装順序

### Step 1: ナビゲーション

「機能」（通知・情報収集、AI チャット）と「システム・管理」（モデル管理、ログ、設定）に分類。既存ルートや画面を維持。

### Step 2: Web 情報収集

初期表示を通知閲覧中心に再構成。情報源選択ペイン、通知一覧、記事詳細を分離。巡回履歴はタブ/切替、情報源設定は管理領域/ダイアログへ。手動巡回、Worker 状態、記事一括削除の既存操作を残す。最新10件・最大100件一括削除など現行の仕様を変更しない。実際の情報源数が1件の場合も使いやすくする。

### Step 3: 設定

Worker／バックグラウンド AI／データと保存先の3カテゴリへ整理。モデル自動ロード/アンロード、AI 一時停止、サインイン時自動起動、保存/再読込、各保存先のエクスプローラー起動などを残す。設定の保存単位と注意文を明確にする。

### Step 4: 軽微な整合調整

ページ見出し、余白、操作文言を統一。AI チャット、モデル管理、ログは原則変更しない。

## 実装制約

- **WinUI 3 + XAML + MVVM 継続。Blazor 移行しない。** チャット内の既存 WebView2 はそのまま。
- 機能削除、API 契約変更、DB Migration、保存先変更、Worker の起動方式変更、Ollama の制御方針変更は対象外。
- 画面デザインのための大量の外部ライブラリ導入を避ける。
- スクリーンショットのデモ文言・例示データを固定値で実装しない。
- UI の破壊的操作には従来の確認を残す。
- MVVM の責務を守り、View コードビハインドにビジネスロジックを移さない。
- ユーザーの指示がない限り既存のドキュメントやデータを削除しない。

## 検証

- Debug / Release ビルド、既存テストを実施。
- 手動確認: ナビゲーション全項目、Web 情報源選択、通知詳細、巡回履歴、手動巡回、記事一括削除、設定保存/再読込、Worker 停止時、AI 一時停止時、保存先を開く。
- ウィンドウ幅を変更し、狭幅時のはみ出し、スクロール、操作不能を確認。
- 実画面スクリーンショットを撮り、`ui-design/references/` の元画像と `ui-design/mockups/` を比較。差分があれば理由を説明。
- 変更内容・移動した操作・未対応点を報告し、UI 設計資料を必要に応じて更新。

## 完了条件

「通知を確認したい」「巡回履歴を見たい」「情報源を設定したい」「AI を一時停止したい」「ログを見たい」という目的に対し、ユーザーが迷わず該当箇所へ到達でき、既存機能がすべて動作すること。
