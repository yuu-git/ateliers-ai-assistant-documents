---
title: 通知一覧のクイック操作・実装状況
---

# 通知一覧のクイック操作・実装状況

2026-10-29。[通知管理](notification-management-implementation-status.md) に続く操作性の追加調整。[実装指示](../ai-implementation/ui-notification-quick-actions-codex-instructions.md) に従い、既存のナビゲーション・日付グループ・記事詳細・タグ・Worker・収集・要約を維持する。

## 操作

### 2026-10-30 の配置・配色調整

横長画面の情報源ペインは、「表示」→すべて／未読のみ／ブックマークのみ→区切り線→「情報源」→すべての通知／登録情報源の順とする。表示条件を展開領域から移し、常時切り替えられるようにした。表示と情報源は独立した既存の `FilterIndex`／`SelectedFeedSource` を使い、相互の選択をリセットしない。

中幅・狭幅では、同じナビゲーション領域に「表示」と「情報源」の選択欄を横並びで置く。幅を変更しても条件と選択表示を同期する。タグ条件は従来の展開領域に残し、DB 条件・100件の範囲・既読化・保存仕様は変更しない。

ブックマーク中の ★ は、標準ダークテーマでは明るい黄色、ライトテーマではコントラストを確保した濃い黄色とする。未登録の ☆ は標準文字色。ハイコントラストではシステムの文字色を優先する。ツールチップ・状態別アクセシブル名と40×40の操作領域は維持する。実装は `WebMonitoringView.xaml/.cs` と実画面診断のみで、ViewModel・DB・Worker API への変更はない。

同日の確認: x64 Debug／Release ビルド成功（既存 NU1900 警告1件）、Debug の既存72シナリオ成功。実 WinUI で幅480～1600の配置、横長／狭幅の両操作欄の選択同期、情報源の維持、黄色い ★ を確認した。独立 Worker を使い、ナビゲーションからのブックマーク絞込・解除、星の表示状態・色・アクセシブル名、既読・タグ・再起動復元も成功。配色の目視確認は標準ダークテーマ・100%倍率。ライト／ハイコントラストの実画面確認は未実施。

- 各通知右側の **☆／★** でブックマークを追加・解除する。ツールチップとアクセシブル名は状態に応じ「ブックマークに追加／ブックマークを解除」。40×40 の標準 Button を使い、Tab で移動し Enter／Space で実行できる。
- 同じ操作列の下にあるゴミ箱で記事を単体削除する。ブックマーク・削除の操作だけでは記事を選択せず、既読にもしない。
- 既定は確認あり。記事タイトルと削除対象、再取得時は新規扱いとなる説明を表示し、キャンセルを既定ボタンとする。
- 「次回以降は確認しない」は **削除を確定し、削除サービスが成功を返した後**に保存する。キャンセル・閉じる・削除失敗・中止では変更しない。設定保存失敗では確認ありを維持し、「記事は削除したが設定を保存できなかった」と区別して通知する。
- 設定 → **データと保存先** の「記事の単体削除前に確認する」でいつでも再有効化できる。変更は即時保存。一括削除には適用せず、既存の複数選択・確認・件数警告を維持する。
- 一覧の独立した既読／未読ラベルを廃止し、未読記事だけ補助情報行の末尾へ赤系の **未読** を付ける。標準テーマの Critical ブラシと文字を用い、色だけに依存しない。同じ RichTextBlock 内で末尾を切り替え、状態ラベル用の独立行は確保しない。記事詳細の状態・手動切替は維持する。
- フィルターの表示名を「すべて／未読のみ／ブックマークのみ」に統一。現在の条件は折りたたみ見出しにも表示し、情報源・タグ・問い合わせ範囲の既存表示を維持する。

## 再利用した境界・設定

単体と一括は同じ `IWebCollectionOperations.DeleteAsync` と結果反映を使用する。通常起動では `WorkerApiClient` → 既存の `articles/delete` API → Worker → SQLite ストアとなり、Desktop から DB／Markdown を直接操作しない。

通知・入力・要約・整理情報・Markdown・重複判定キーを従来どおり削除し、巡回履歴と共有／孤立タグ定義は保持する。最新10件に残っていれば新 ID・未読・ブックマーク OFF・タグなしで再取得できる。

削除成功 ID だけを一覧から除き、選択中の記事なら詳細をクリアする。他記事の選択は維持し、一覧・未読総数・情報源別件数を既存の通知問い合わせで更新する。削除後の再取得に失敗した場合は削除済みの記事をキャッシュに戻さず、件数は最終集計としてエラーを表示する。確認から削除・設定保存まで多重実行を防ぎ、メタデータ操作と競合させない。

追加するユーザー設定は `UserSettings.ConfirmSingleArticleDeletion`（bool、既定 true）のみ。既存の Roaming `settings.json` と Debug／Release 分離を使い、旧 JSON に項目がなくても true になる。DB Migration、DB カラム、Worker API、Worker 実装版、新規 NuGet 依存の追加・変更はない。

設定保存は既存の保存キューと原子的 JSON 書込を利用する。この値は保存成功後にだけ画面とメモリへ確定し、失敗後のモデル保存で誤って確認省略を永続化しない。並行するモデル・Web 設定保存でも確認設定を巻き戻さない。読込失敗時は既存ファイルを保護し、確認を省略しない。

削除結果は `web.article-deletion`、設定変更は `settings.article-deletion` として固定文言・件数だけを記録する。記事タイトル・本文・URL を追加ログへ含めない。失敗は既存診断ログにも記録する。

### 一覧の検索範囲（変更なし）

| 条件 | 対象 |
| --- | --- |
| すべて・タグなし | 全体の最新100件を先に取り、情報源を絞る |
| 未読のみ／ブックマークのみ／タグ指定あり | DB 全表示対象通知へ情報源・状態・タグ条件を適用し、初回取得日時の降順で最新100件 |
| 未読件数 | 全表示対象通知。100件上限に依存せず、要約待ちは含めない |

既読化やブックマーク解除で一覧から外れた開き済み記事の詳細保持、定期更新での条件・行・スクロール保持、該当なしの空状態も維持する。

## 変更ファイル

| ファイル（アプリリポジトリ内） | 変更 |
| --- | --- |
| `Application/Settings/IUserSettingsStore.cs` | `UserSettings` の任意設定追加・旧 JSON 互換 |
| `Desktop/ViewModels/SettingsViewModel.cs` | 成功時の確認設定確定、保存キューとの整合、設定エラー・ログ |
| `Desktop/ViewModels/WebMonitoringViewModel.QuickActions.cs` | 確認結果、単体削除、共通結果反映と削除ログ |
| `Desktop/ViewModels/WebMonitoringViewModel.cs` / `.Notifications.cs` | 一括処理の共有、実行可否、アイコン・アクセシブル名・未読末尾、削除中に明示的に開いた記事の既読要求を後へ送る |
| `Desktop/Controls/WebMonitoringView.xaml` / `.cs` | 独立したアイコン列・削除イベント・未読メタデータ・条件名 |
| `Desktop/Controls/DataStorageView.xaml` | 確認設定と保存タイミングの説明 |
| `Desktop/MainWindow.xaml.cs` | タイトル・確認省略チェックを含む単体確認ダイアログ |
| `Desktop/Diagnostics/UiLayoutVerification.cs` | 実画面でのキャンセル・削除・省略・再有効化 |
| `Desktop/Diagnostics/NotificationFeatureVerification.cs` とビューの DEBUG ヘルパー | 固定待機時間で次の操作を始めず、メタデータ・コマンドの完了を確認して回帰検証 |
| `tests/NotificationQuickActionTests.cs`・`Program.cs` | 実 SQLite／JSON、削除・設定失敗・順序・連打の追加検証と登録 |
| `tests/Verify-UiRefactoring.ps1` | 実ダイアログの CheckBox／Button と設定 CheckBox を Automation で操作 |

上表の層名は `src/Ateliers.Ai.Assistant.*`、テスト C# は `tests/Ateliers.Ai.Assistant.Desktop.Tests/` を省略している。関連ドキュメントとアプリ README も更新した。

## 確認

```powershell
dotnet build src/Ateliers.Ai.Assistant.Desktop -p:Platform=x64 -p:WindowsPackageType=None -p:WindowsAppSDKSelfContained=false
dotnet build src/Ateliers.Ai.Assistant.Desktop -c Release -p:Platform=x64 -p:WindowsPackageType=None -p:WindowsAppSDKSelfContained=false
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests -c Release
pwsh -File tests/Verify-UiRefactoring.ps1
pwsh -File tests/Verify-WorkerDisplay.ps1 -Notifications
```

| 確認 | 結果 |
| --- | --- |
| x64 Debug／Release Unpackaged ビルド | 成功、エラー0。既存の NU1900 警告1件（NuGet の脆弱性情報取得先への接続失敗）は継続 |
| 自動テスト | 両構成 **各72シナリオ成功**。既存70件に、単体削除・確認・連打・一括確認の独立と、旧 JSON・設定保存順序・失敗時維持の2件を追加 |
| 単体削除と設定 | 実 SQLite／JSON と既存削除サービスを使用。キャンセル・削除失敗・設定失敗、成功後の確認省略、選択記事削除時の詳細クリア、別記事選択の保持、Markdown 削除・件数更新・巡回履歴保持を確認 |
| 実 WinUI | ゴミ箱ボタンからキャンセル／確認チェック付き削除／確認省略での次の削除、設定 CheckBox からの再有効化と JSON 保存を確認 |
| 一覧・狭幅 | 幅480～1600相当の実画面でアイコン列・補助行末尾の赤系未読を確認。日付グループ、一覧／詳細の戻る、一括削除・件数警告、長文 Markdown のスクロール保持も成功 |
| 実 Worker API 回帰 | 新しいアイコンと状態別アクセシブル名、ブックマーク単独操作、既読・未読・タグ・件数、Worker 停止／再起動後の復元、UI 終了後の継続巡回、停止中のログ・独立チャットが成功 |

画面検証は隔離ルートのデータと偽 RSS／Ollama を使い、利用者の DB・設定・実 Ollama は変更していない。画像はアプリ側 `artifacts/ui-refactoring/`（Git 対象外）へ出力する。描画倍率100%・標準ダークテーマで確認した。実キーボードだけでの全経路、ライト／ハイコントラスト各テーマ・各 DPI・複数モニター、MSIX 配布後、実 RSS＋実 Ollama での再確認は未実施。操作の自動確認は標準 Button／CheckBox の Automation パターンを利用した。

既存の1問い合わせ100件、タグ候補500件、Worker 接続が必要な更新・再集計という制約は維持する。削除後の最新10件からの新規再取得・重複判定・巡回履歴保持は既存のサービス／DB テストでも回帰確認した。大量履歴のページングや新たな削除方式は追加していない。
