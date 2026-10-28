---
title: Markdown 回答表示（Phase 1 追加）
---

# Markdown 回答表示

Markdown 原文は `ChatMessage.Content` に保持し、Ollama に送る履歴も原文のまま扱う。Markdown・HTML・JavaScript・WebView2 の処理は Desktop の表示層に配置し、Core / Application / ViewModel の通信・モデル管理に依存を追加しない。

## 表示構成

- Markdig 1.5.0 で Markdown を HTML に変換し、HtmlSanitizer 9.2.1039 で安全化する。基本書式、パイプ形式の表、fenced code block に対応する。生 HTML と任意属性等の拡張は有効にしない。
- 汎用の `MarkdownMessageRenderer` と、チャット固有の `ChatTranscriptView` / 表示スナップショット・差分処理を分離する。
- WebView2 はチャット表示領域全体で 1 インスタンス。入力・モデル選択・起動停止・ナビゲーションは WinUI のまま維持する。
- HTML、CSS、JavaScript、highlight.js 11.11.1 はアプリへ同梱する。実行時に CDN や外部ページを読み込まない。highlight.js のライセンスを同梱する。
- 言語指定付きコードブロックは、完了・中止・失敗時に対応言語のハイライトを行う。受信中と未知の言語は等幅のプレーンコード表示とする。

## 更新と原文の保持

- `Content` の受信追記は従来どおり行い、表示更新だけを約 100ms 間隔で集約する。状態変更と会話リセットは優先して反映する。
- 解析・安全化は背景処理で行う。1つの処理だけを実行し、完了済み本文の HTML はキャッシュする。メタデータだけの変更では本文を再変換しない。
- WebView のページ全体を再読み込みせず、変更メッセージだけを JSON で送る。既存の DOM 要素は再利用する。
- 世代番号・更新番号で古い結果を拒否し、会話クリア後に旧会話を復活させない。WebView 初期化前の更新は、ページ準備完了後に最新状態から反映する。
- 利用者が末尾を表示している場合だけ自動追従する。過去の会話を読んでいる場合は表示位置を維持する。
- WebView2 の初期化・表示に失敗した場合はエラーを示し、原文表示でチャット操作を継続できるようにする。自動再初期化は行わない。

## 安全性

- HTML の処理には既存ライブラリを使用し、独自パーサーや正規表現による HTML サニタイザーを作らない。
- Markdig の生 HTML を無効化したうえで、HtmlSanitizer の許可タグ・属性・URL スキームを限定する。安全化済み HTML だけを本文の `innerHTML` に渡し、名前・状態などは `textContent` に設定する。
- Markdown 由来の画像、外部画像、SVG、iframe、フォーム、イベント属性、任意スタイルは許可しない。画像貼り付けの追加要件では、別フィールドで受け取ったアプリ生成 PNG の Data URL だけを画像要素へ設定する。リンクは絶対 HTTP / HTTPS URL のみとし、ローカル表示ホスト・ユーザー情報付き URL・その他のスキームは拒否する。
- 専用の仮想ホストで表示用資産フォルダーだけを公開する。リソース要求を既知の同梱ファイルに限定し、CSP でもスクリプト・スタイルを同一オリジンに制限、外部通信・埋め込みを禁止する。
- WebView 内の外部遷移、子フレーム遷移、新規ウィンドウ、ダウンロード、権限要求を禁止する。
- ページからのメッセージは送信元・種類・サイズ・URL を検証する。利用者のリンククリックだけを既定ブラウザーへ渡し、汎用のホストオブジェクトやファイル操作等は公開しない。
- C# からページへは JSON ライブラリでシリアライズしたデータを送信し、モデル出力を JavaScript コードへ連結しない。

## MVP の制約

- DOM の仮想化・古い履歴の段階読み込みは未実装。更新対象の限定と HTML キャッシュを行うが、非常に長い会話ではメモリ・描画負荷が増える。
- Markdown の画像構文、数式、Mermaid、生 HTML、独自拡張は対象外。手動の画像添付は [画像貼り付け](image-paste.md) に従う。コードの自動言語判定は行わず、同梱した highlight.js の対応言語だけを使う。
- WebView2 Runtime が別途必要。Windows App Runtime の導入だけで WebView2 Runtime の存在は保証しない。
- ビルド・検証手順、実施結果はアプリ側 README を参照する。

参照: [Markdig](https://github.com/xoofx/markdig)、[HtmlSanitizer](https://github.com/mganss/HtmlSanitizer)、[WebView2 のセキュリティ](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/security)、[highlight.js](https://github.com/highlightjs/highlight.js)。
