---
title: MVP 完了後の複数画面監視基盤
---

# MVP 完了後の複数画面監視基盤

2026-10-30。[Phase 7 MVP](phase7-implementation-status.md) は利用者の動作確認を受けて完了した。今回の変更は [複数監視の実装指示](../ai-implementation/multi-screen-monitor-codex-instructions.md) と、その後の承認条件に基づく基盤拡張である。新しい番号付き Phase、画像比較、複数画像同時送信、認識精度改善、音声は追加しない。

## 現在仕様と操作

- 実装版 **7.1.0**。旧 Worker を停止してから、同じ版の Desktop／Worker を起動する。プロトコル版1と認証方式は維持し、既存の実装版完全一致チェックで旧 UI を拒否する。
- 「画面監視」の一覧で追加・選択・削除し、選択中の監視の名前、対象、モデル、周期、プロンプトを設定する。名前は前後の空白を除く1〜64文字、制御文字は禁止し、同名は許可する。識別には名前や HWND ではなく `MonitorId` を使う。
- **登録上限4件**。停止中も数える。`ScreenMonitorLimits.MaximumMonitors` の内部定数として管理し、設定 UI は追加しない。上限時は追加ボタンを無効化し、API も拒否する。
- 周期は既存の10／30／60／120／300秒、初期60秒。設定・対象変更は当該監視の停止と処理終了後に行い、別の監視の実行を止めない。
- 一覧には対象・状態・次回予定・最終解析・エラーを表示する。選択中の設定とコメントを右側、狭い幅では一覧の下に表示する。狭幅時は設定を初期折り畳みにしてコメント領域を確保する。
- 各監視は独立した WGC 対象・取消・コメントを持ち、同一ウィンドウ／ディスプレイの重複選択も拒否・統合しない。フレーム共有は行わない。
- Desktop 終了後も監視は続く。再接続時に一覧と各監視のコメントを復元表示する。トレイは稼働件数と「全監視を停止」を提供し、利用不能時は全監視を安全に停止する。
- AI 一時停止中はすべての監視の取得と解析を中止する。RSS／GitHub 等の取得は継続する。再開時は現在画像から処理し、過去画像をまとめて解析しない。
- 画面ロック・Worker 終了は全監視を停止する。対象消失・取得失敗・モデルエラーは該当監視だけを停止する。同じ対象自体が閉じた場合は、それを使う複数の監視へそれぞれ対象消失が通知される。

## 所有権・状態・破棄

既存 Worker が唯一の管理主体であり、専用 Agent・Windows Service・別 DB は追加しない。`MultiScreenMonitorService` が全監視と実処理1件を所有する。旧単一 `ScreenMonitorService` は置き換えた。

`CaptureWindowHost` は STA・所有者 HWND・トレイ・ロック通知・Picker 排他を共有し、取得対象自体は持たない。`CreateScope` が監視ごとに `WindowsScreenCaptureService` を持つ取得ポートを返す。停止では再選択せず再開できるよう対象を維持し、削除では対象イベントの購読と WGC 対象参照を解放する。

| 識別・状態 | 範囲 |
| --- | --- |
| `MonitorId` | 定義の不変 GUID。再起動でも維持 |
| CaptureSessionId | 監視開始時に更新する実行世代 |
| TargetId | 対象を選び直した時に更新。名前や HWND を代用しない |
| Worker InstanceId | 再起動・遅い応答・旧コメントを識別 |
| Active / Picking / Processing / Deleting | 監視ごとに分離。取消要求だけで Processing を偽にしない |

停止は取消を即時通知し、実際の処理が残っていれば「停止処理中」と返す。稼働中の削除は停止・削除予約として受け付け、実処理と画像リースの終了後に定義を保存削除し、対象とコメントを破棄する。保存に失敗した場合は定義を残して停止状態・エラーとし、再試行できるようにする。

Picker は Worker 全体で一つだけ開く。対象選択中の同一監視の削除・設定変更・開始を拒否し、別の Picker も開かない。**結果の取消と OS 操作の完了を区別**し、取消通知だけで Picker の所有権を解放しない。OS 画面が残った場合は利用者がキャンセルして閉じる。選択キャンセルでは既存対象を維持し、遅れて返った取消済み結果を採用しない。他の監視の取得は継続する。

## 周期・公平性・取消

Worker の約1秒ループから全監視を調べる。各監視は次の周期予定と**実行資格を最大一つ**持つ。周期到来時に実行枠が埋まっていれば、見送りとして回数・理由を記録して資格を一つに集約する。見送った回の画像・時刻付きジョブを保持したり、回数分を追いつき実行したりしない。

枠が空いたら、実行資格がある監視を最終開始時刻の古い順、同時ならスキップ状況等で選ぶ。取得時には現在画像を撮る。短周期の監視が連続して枠を独占しないようにし、将来の周期を待つためだけに枠を空ける予約制御は導入しない。周期の厳密な実行保証ではなく、状態・見送り理由を確認できる方式とする。

モデルのロード・一枚取得・Vision 全体で **最大10分**。単発取得の10秒制限は維持する。停止・削除・AI 一時停止を速やかに伝播するが、取消やタイムアウトを通知した時点では枠を解放せず、実際の呼出しタスクが終了するまで次の取得・推論を開始しない。中止を無視する偽推論でも同時件数1を確認する。Worker 終了は全監視を中止し、既存の期限付き終了処理で待つ。強制終了時の完全な後片付けは保証対象外。

## モデル・Temp・コメントの共有

既存のモデルファイル排他をチャット・手動解析・要約・監視で共有する。モデルの保護は稼働中／実処理中の監視集合で判定する。同一モデル名の既定タグ表記は既存 `SameModel` で照合し、Aを停止・削除してもBの予約・使用が残るモデルを自動解放しない。

自動ロード OFF では未ロードモデルの監視は取得前に待機する。監視が実際に使用したモデルと要約から引き継いだロード対象を追跡し、最後の使用・予約がなくなった時だけ既存 `AllowUnload` 方針に戻す。解放前に `/api/ps` で状態を確認し、未ロードモデルへ不要な操作を送らない。取得した全モデルを管理対象にはしない。

監視は `keep_alive=-1` を使い、同じ Worker の要約も監視集合が使うモデルの保持期限を短縮しない。Desktop の自動解放判定も一覧全体の稼働・実処理を参照する。モデル解放が他の処理等でできない場合は保留・エラー表示とし、短時間に失敗ログを連打しない。他アプリや Ollama 内部の利用元を完全に識別できない制約、利用者による明示モデル停止の意味は維持する。

画像ストアは **Worker に一つ**だけ注入する。既存のセッション別64枚／5分・プロセス全体128 MiB・一括リース保護を維持し、実行する一枚を登録・リース・解析後解放・削除する。他監視の使用中画像を削除しない。ディスク Temp の環境全体64件／256 MiB等も変更せず、画像を自動的にファイルへ退避・保存しない。

コメントは次の順で制限する。

1. 1件65536文字まで。監視ごとに最大200件を適用する。
2. 全監視の本文 UTF-16 概算を合計し、Worker 全体 **1 MiB**以内にする。
3. 超過時は古い完了・中止・失敗コメントから除去し、受信中本文を保護する。

指示書の「本文8 MiB」は導入時調査で訂正した。8 MiBは既存 API 応答読込上限であり、本文保持枠ではない。UI はコメント ID の差分で削除・更新し、選択切替・定義削除・Worker 再起動後の古い非同期応答を採用しない。編集中入力は監視ごとに維持する。履歴は DB 保存しない。

## 保存と旧設定移行

既存 Roaming `worker-settings.json` に任意項目 `screenMonitors` を追加する。各定義は MonitorId、名前、モデル・周期・プロンプト、更新競合用 Revision を持つ。対象・HWND・稼働フラグ・画像・コメント・次回予定は保存しない。Debug／Release の分離を維持する。

- 新一覧が未指定で旧 `screenMonitor` があれば、「画面監視1」へ設定を移し、生成 ID と新一覧を原子的に保存する。旧項目は新形式の保存時に null とする。
- 新一覧が未指定で旧設定もなければ空一覧で開始する。
- 新一覧が空を含めて存在すれば移行済みとし、旧項目から再作成しない。
- 保存済み値を先に検証し、一時ファイルから置換する。256 KiBの上限も保存前に確認する。移行・保存失敗時に元設定を初期化・上書きしない。
- 起動時はすべて停止・対象再選択・コメントなし。定義 ID と設定は維持する。更新のため Worker を再起動した場合も、コメントは従来どおり失われる。
- 個別更新は当該 Revision を検査し、Worker の最新設定にその定義だけを合成する。他監視・情報源・AI 設定を古いスナップショットで上書きしない。一般の設定 API による監視一覧の直接変更は拒否する。

## Worker API と変更箇所

既存の認証・Origin／Host拒否・JSON・応答容量制限を使う。`/api/v1/monitors` が一覧取得・作成、`/monitors/{id}` が個別状態・コメント・設定更新、`/{id}/pick|start|stop|delete` が明示操作、`/monitors/stop-all` が全停止となる。旧 `/screen/status|settings|pick|start|stop` の単一 API は削除し、先頭監視へ割り当てる互換処理は作らない。実装版不一致時に接続拒否されるため、UI と Worker は同時に更新する。

主な変更ファイル:

| 配置 | 責務 |
| --- | --- |
| Application `ScreenAnalysis/ScreenMonitorContracts.cs` | 定義・ID操作・一覧・詳細・内部上限 |
| Application `ScreenAnalysis/MultiScreenMonitorService.cs` | セッション所有・周期調停・モデル追跡・履歴容量・停止削除 |
| Application `Worker/WorkerContracts.cs`, `WorkerCoordinator.cs` | 一覧設定、ID単位の原子的合成、要約のモデル保護・引継ぎ、版7.1.0 |
| Infrastructure `Worker/WorkerStateStore.cs` | 旧形式移行・保存前容量検証 |
| Infrastructure `Worker/WorkerApiClient.Screen.cs`, `WorkerJsonContext.cs` | ID API・JSON・Desktopの自動解放判定 |
| Windows `Capture/CaptureWindowHost.cs`, `WindowsScreenCaptureService.cs` | STA共有、対象の独立スコープ、Pickerの実完了までの排他 |
| Worker `WorkerHost.cs` | 構成・ID API・全体トレイ・終了制御 |
| Desktop `ViewModels/ScreenMonitorViewModel.cs`, `WorkerViewModel.cs` | 監視ごとの一覧・下書き・コメント・版追従 |
| Desktop `Controls/ScreenMonitorView.xaml(.cs)` | 一覧と詳細・狭幅・削除確認・上限表示 |
| `MultiScreenMonitorTests.cs`, `ScreenMonitorTests.cs`, `Program.cs` | 単一監視の既存検証を新制御へ移し、複数監視の検証を追加 |
| `Verify-MultiScreenMonitor.ps1`, `Verify-MonitorApplicationTargets.ps1`, `ScreenWorkerVerification.cs` | 実 Worker・実 WGC・実 MainWindow と実モデルの検証 |

## 検証と結果

```powershell
dotnet build src/Ateliers.Ai.Assistant.Desktop/Ateliers.Ai.Assistant.Desktop.csproj -p:Platform=x64 -p:WindowsPackageType=None
dotnet build src/Ateliers.Ai.Assistant.Desktop/Ateliers.Ai.Assistant.Desktop.csproj -c Release -p:Platform=x64 -p:WindowsPackageType=None
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests
dotnet run --project tests/Ateliers.Ai.Assistant.Desktop.Tests -c Release
pwsh -File tests/Verify-MultiScreenMonitor.ps1 -Model gemma3:4b -SecondModel qwen3.5:4b
pwsh -File tests/Verify-ScreenMonitor.ps1 -Model gemma3:4b
pwsh -File tests/Verify-MonitorApplicationTargets.ps1
```

自動テストは全99件。監視専用で独立設定、公平実行、取消を無視する実処理の枠保持、同一モデル保護、急な一時停止・再開、全体10分期限、他セッションの画像リース保護、移行再実行・保存失敗、上限超過、コメント削除、設定競合を確認する。UI では監視別の下書き保持、削除されたコメントの反映、古い詳細応答・失敗による表示上書きの防止も検証する。旧単一制御の別実装は残さず、従来の監視回帰ケースも新制御で検証する。

Debug／Release の Desktop x64 ビルドは、ともに警告0・エラー0。自動テストも両構成で99件すべて通過した。既存 Worker の認証・巡回・保存・強制終了からの回復、情報源／通知／ログの実 UI、Phase 6 の単発解析・18回の反復キャプチャも回帰確認した。既存 UI の広幅・中幅・狭幅、通知の選択・スクロール保持、削除確認、設定編集、ナビゲーションも実 UI で通過した。

実機で以下が成功した。

- 同一ウィンドウを2件の独立 WGC 対象で保持し、片方の破棄後も他方を取得できる。
- 標準 Picker で同じ検証用ウィンドウを2回選択し、別周期・別プロンプトで両方の Vision コメントを受信できる。
- 実 MainWindow の一覧・監視別コメント・時刻付き Markdown、通常終了時のモデル保護、Desktop 終了後の両監視継続、再起動 UI の復元が成立する。
- Picker 結果の取消後も OS 選択完了まで所有状態を維持し、キャンセルで旧対象を残す。他監視は継続する。
- 別ウィンドウと別モデル `gemma3:4b`／`qwen3.5:4b` によるコメントを分離でき、稼働中Aの削除後もBが進む。
- AI停止・再開、トレイ全停止、最後の使用モデルの自動解放、Worker 再起動後の停止・ID／設定復元・コメント消去が成立する。
- 従来の単一監視の検証も同じ複数監視制御で通過した。UI を介さない標準 Picker 自動試験では、所有ウィンドウを表示して前面化してから選択操作を行う。
- 専用プロファイルのローカル Chrome ページと既存タスクマネージャーの実 WGC 取得を反復し、Chrome監視削除後もタスクマネージャー監視が継続した。この試験の推論はプロセス内の偽物であり、実画面画像を Ollama に送信・保存していない。検証用ウィンドウ画像による実 Vision と区別する。

`ministral-3:3b` は当該環境の Ollama で CLIP 読込に失敗した。該当監視だけがエラー停止し、他方の `gemma3:4b` が継続することを確認した。モデルの修復・再取得は行わず、導入済み別モデルで正常系を確認した。

## 残る制約

登録4件は GPU メモリを保証する値ではない。複数の大きなモデルの常駐可否・退避は Ollama と端末に依存し、推論を直列化しても全モデルが同時に常駐する保証はない。モデルごとの認識精度、意味的な画像差分、録画、通知、音声、履歴の永続化は対象外。

取消後の実行枠の解放は、アプリケーション側の取得・通信・推論タスクの実完了を基準とする。HTTP 接続が終了した後の Ollama 内部の GPU 処理終了までは、クライアント側から確認・保証できない。

最小化・保護コンテンツ・ゲームの排他フルスクリーン・HDR・複数 GPU／DPI・MSIX・スリープ復帰などは、[Phase 7 の環境別制約](phase7-implementation-status.md#残る制約と最終確認)を引き継ぐ。実 Windows のロックを伴う全環境の操作試験は別途必要。今回のモデル修復や Ollama 本体更新は行っていない。
