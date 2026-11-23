# AGENTS.md

## リポジトリの役割

別リポジトリにある iOS アプリを GitHub Actions でビルド・テストするためのリポジトリです。実行履歴とログは `ios-ci` にあり、実行状況はアプリ側の PR にコミットステータスとして表示されます。

- [中央 workflow](.github/workflows/ios-ci.yml) は `workflow_dispatch` を受け付け、デフォルトブランチで検証、ビルド・テスト、結果通知を実行します。
- [アプリ側の通知例](.github/examples/notify-ios-ci.yml) は対象アプリの `.github/workflows/` に配置するファイルです。アプリと同じリポジトリ内のブランチから作成した PR の `opened`、`synchronize`、`reopened` で、リポジトリ名、PR 番号、head SHA を送ります。
- GitHub App、Secrets、Actions variables、アプリの追加手順は [SETUP.md](SETUP.md) にまとめます。

## Workflow を変更するとき

- `prepare` は `owner/repo` の `IOS_CI_PROJECTS` への登録とアプリ設定を確認し、PR 番号、コミット SHA、実行モードを検証します。検証後は対象 SHA に `pending` を送ります。通知側で受け取る PR の値は環境変数を経由してシェルに渡してください。
- `xcode` は指定された SHA を `app/` にチェックアウトします。`project_path` はアプリルートからの相対パスで、`.xcworkspace` と `.xcodeproj` に対応します。共有 Scheme と Simulator の destination は `IOS_CI_PROJECTS` で指定します。任意の `runner` と `xcode_version` を省略した場合は、`macos-26` と、そのランナーで選択されている Xcode を使います。
- `operation: test` は既定値で、fastlane の `run_tests` を使います。`build` は `build_app` の `skip_archive:true` で Simulator 向けにビルドします。ランナーの `fastlane` と `xcbeautify` を使用し、GitHub Actions 向けにログを整形します。両モードで `-skipMacroValidation` を Xcode に渡します。
- `report` はビルド・テストが失敗した場合も結果を公開します。開始時と終了時で対象 SHA、コンテキスト `ios-ci / <operation>`、実行ログへの URL を揃えてください。同じリポジトリ・SHA・実行モードの実行は同じ concurrency group に入ります。
- 1 つの GitHub App を共有し、トークンは起動、checkout、ステータス更新の各用途に必要なリポジトリと権限に絞ります。通知例は `pull_request_target` で PR の情報だけを送信します。通知ジョブやステータス更新ジョブで PR のコードを実行しないでください。checkout では `persist-credentials: false` を維持し、秘密鍵やトークンをコミットやログに含めないでください。
- workflow の入力、設定名、権限を変更した場合は、影響する通知例と [SETUP.md](SETUP.md) も確認してください。配布を実装する場合は、署名とアップロードを配布用 workflow に置きます。

## 変更の確認

- `git diff --check` で差分を確認します。workflow を変更した場合は、YAML の構造と変更したシェルの構文も確認してください。
- 起動、ビルド、ステータス通知を変更した場合は、実際の PR か手動実行で、対象 SHA、Xcode と fastlane の実行、アプリ側の `pending` から終了状態への更新を確認します。ローカルでの構文や引数の検証と、GitHub Actions 上での実行結果は区別して報告してください。
