---
title: アイコン・製品情報と MVP 仕上げ
---

# アイコン・製品情報と MVP 仕上げ

現在の仕上げ版は **0.7.0**。第二段階の一般アイコン素材を採用し、Desktop / Worker の EXE、Desktop のウィンドウ、監視対象選択ウィンドウ、監視中のトレイ、既存パッケージのロゴを統一する。第三段階のキャラクター制作と MSIX の新規対応は含めない。

## 素材と固定パス

アプリリポジトリの `assets/branding/` を素材の原本とする。`app.ico` が Windows 用のマルチサイズアイコン、`icon-1024.png` が既存パッケージ用画像の生成元、`app-icon-master.svg` が編集用マスター。README のヘッダーは `readme-header.png`。第三段階のボードやキャラクター素材は取り込まない。

`build/BrandAssets.props` を Desktop / Worker が共用し、同じ ICO を EXE に埋め込む。実行時は両ホストの出力に `Assets/Branding/app.ico` としてコピーする。Desktop の発行先では Worker 用のコピーを `Worker/Assets/Branding/app.ico` に含める。ウィンドウとトレイはカレントディレクトリに依存せず、実行ファイルの基点から読み込む。

Desktop は `AppWindow.SetIcon` を使用する。Worker のネイティブウィンドウは大小のクラスアイコンと小さいトレイアイコンを読み込み、トレイ・ウィンドウ・クラスの終了後に所有ハンドルを解放する。読み込み失敗時は標準アイコンへ退避し、Trace 警告を出す。監視中だけトレイを表示する既存仕様、右クリックの全停止、Explorer 再起動時の再登録は維持する。

## 将来の画像差し替え

1. 新素材を同じファイル名で `assets/branding/` に置く。ICO には 16 / 24 / 32 / 48 / 64 / 128 / 256px を含める。
2. ICO と PNG は自動的に相互変換されない。新しい SVG から必要な ICO・PNG を用意し、`app.ico` と `icon-1024.png` の両方を更新する。README 用画像は必要に応じて差し替える。
3. `pwsh -File tools/Update-BrandingAssets.ps1` を実行し、既存のパッケージ用 PNG を更新する。命名・寸法・縦横比を維持し、横長画像では中央配置する。
4. Debug / Release を再ビルドし、`tests/Verify-Branding.ps1` と実表示を確認する。発行物全体を更新する。EXE 埋め込み画像は実行中の外部 ICO 差し替えだけでは更新されない。

画像更新だけの場合、C#・csproj・manifest の参照先を編集する必要はない。Windows が古いアイコンをキャッシュする場合は、アプリを再起動し、必要に応じて利用者がショートカットやピン留めを作り直す。

## 版番号と識別情報

`Directory.Build.props` が製品版、AssemblyVersion、FileVersion、製品名、Company の共通設定。Worker の実装版は Application の AssemblyVersion から3桁で取得し、Desktop との既存の同版確認に使用する。API プロトコル版は1を維持する。

Win32 マニフェストと既存 MSIX の版は `0.7.0.0` に合わせる。将来の版更新では共通設定と Win32 / Debug・Release MSIX の3つの manifest を更新し、検証スクリプトで不一致を検出する。署名用 Publisher、パッケージ ID、StartupTask、保存先は変更しない。Company は既存 Publisher の `yuuko` に合わせ、権利者の確認なしに新しい Copyright を設定しない。Debug はウィンドウ名と既存パッケージ表示名に `(Debug)` を付ける。

## 検証

アプリ側の `CHANGELOG.md` に今回の結果・未検証事項を記録する。既存のテストランナーは `dotnet test` ではなく `dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests` で実行する。

`tests/Verify-Branding.ps1` は、ICO の各サイズと範囲、EXE 内画像と原本のピクセル一致、Desktop / Worker の実行時 ICO 同梱、EXE / DLL の製品・版情報、既存パッケージ画像の寸法、manifest の版番号と Publisher を検証する。発行物は `-DesktopDirectory` で指定できる。利用者の DB・設定・起動登録には触れない。

MSIX の署名・インストール・Store 配布、x86 / ARM64、各 DPI・テーマ・ピン留め状態の網羅的確認は別の環境確認とする。配布方式の追加や変更は行わない。

今回の配布版番号は利用者の指定により **0.7.0**。過去文書の7.xはフェーズ由来の導入時記録として保持する。タグは形式を含めて保留し、今回は作成しない。
