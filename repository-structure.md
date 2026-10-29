---
title: リポジトリとソリューション構成
---

# リポジトリとソリューション構成

## 決定したリポジトリ

1. `ateliers-ai-assistant`: アプリ本体・テスト・ビルド。単一ソリューション内の複数 csproj を `ProjectReference` で結ぶ。
2. `ateliers-ai-assistant-documents`: Markdown/MDX ドキュメントの原本。

`ateliers-dev` は既存の Docusaurus サイトであり、`ateliers-ai-assistant-documents` を Git Submodule として参照する。`ateliers-ai-assistant` からも同じ文書リポジトリを Submodule 参照する。**2つは今回新たに管理するリポジトリであり、ateliers-dev は既存の第三の参照元**。

```text
ateliers-ai-assistant/
  Ateliers.Ai.Assistant.slnx
  src/
    Ateliers.Ai.Assistant.Core/
    Ateliers.Ai.Assistant.Application/
    Ateliers.Ai.Assistant.Infrastructure/
    Ateliers.Ai.Assistant.Desktop/       # WinUI 3
    Ateliers.Ai.Assistant.Worker/        # 後続
    Ateliers.Ai.Assistant.Api/           # 必要時
  tests/
    Ateliers.Ai.Assistant.Core.Tests/
    Ateliers.Ai.Assistant.Application.Tests/
  docs/                                 # submodule -> documents

ateliers-dev/
  docs/.../ateliers-ai-assistant/         # submodule -> documents（パスはサイト側で決定）
```

上記は構成案。初期段階は Core / Application / Infrastructure / Desktop とテストを中心にし、空の Feature プロジェクトや Host を先に量産しない。

Phase 2 は既存4プロジェクト内に会話保存・設定・パス・診断ログを配置する。EF Core Migration は `src/Ateliers.Ai.Assistant.Infrastructure/Conversations/Migrations/`、ローカルの EF ツール定義はルートの `dotnet-tools.json` に置く。保存先はソース・実行ファイルの配下ではなく [保存方針](data-storage-policy.md) のユーザーデータ領域とし、DB・ログ・設定の実データは Git に含めない。

[AI 管理コンソール構想](features/ai-management-console-concept.md) の追加は文書上の記録に留める。専用プロジェクト・MCP ホスト・オーケストレーター基盤は今は作成しない。着手時に既存の Desktop / Application / Infrastructure の責務へ配置できるかを検討し、具体的な必要性がある場合だけ分割する。

## 文書管理

Phase 3 は既存の Application / Infrastructure に `WebMonitoring/`、Desktop に Web 用 ViewModel・参照先起動・隔離診断を追加する。Migration は会話と同じ場所で管理し、新規プロジェクトは追加しない。生成 Markdown はソースやドキュメントリポジトリへ出力せず、Local データ配下に置く。

- 原本は `ateliers-ai-assistant-documents` のみ。コピーを二重管理しない。
- 通常の文書は `.md`、Docusaurus 固有の機能が必要な場合だけ `.mdx`。
- GitHub / VS Code でも読める相対リンクを基本とする。
- Docusaurus の sidebar や公開設定は `ateliers-dev` 側で管理する。
- Submodule はコミットを固定する。ドキュメント更新後は各親リポジトリで参照コミットを更新する。
- 公開サイトに含まれる文書には秘密情報、個人データ、ローカル巡回履歴を記載しない。

## NuGet 方針

自作ライブラリを現段階で NuGet パッケージ化しない。外部 NuGet 依存の利用は可。将来 OSS 公開したい部分が明確になった時点でリポジトリ・パッケージ分割を検討する。
