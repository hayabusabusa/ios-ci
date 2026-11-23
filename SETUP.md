# ios-ci

Private の iOS アプリリポジトリで PR が作成・再開されたとき、または PR ブランチにコミットが push されたときに、`ios-ci` の GitHub Actions でビルドとテストを実行します。実行履歴とログは `hayabusabusa/ios-ci` の **Actions → iOS CI** に表示されます。

アプリ側の通知 workflow が `workflow_dispatch` でアプリのリポジトリ名、PR 番号、PR の head SHA を送ります。`ios-ci` が指定された SHA をチェックアウトするため、実行待ちの間に PR が更新されても対象コミットは変わりません。[GitHub API の workflow dispatch](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event)

## GitHub App

1. GitHub App を 1 つ作成し、Repository permissions に `Actions: Read and write`、`Contents: Read-only`、`Commit statuses: Read and write` を設定します。同じ個人アカウント内で使う場合は、App のインストールをそのアカウントに限定できます。Webhook は無効にして構いません。
2. App のインストール先で **Only select repositories** を選び、`hayabusabusa/ios-ci` と対象の Private アプリリポジトリを選択します。
3. `ios-ci` と通知を送る各アプリリポジトリの **Settings → Secrets and variables → Actions → Variables** に、同じ App の Client ID を `IOS_CI_APP_CLIENT_ID` として登録します。秘密鍵は次の手順で登録します。

各ジョブは短命なインストールトークンを発行します。通知は `ios-ci` の `Actions: write`、checkout は対象アプリの `Contents: read`、実行状況の通知は対象アプリの `Commit statuses: write` に絞ります。既存の App に `Commit statuses` を追加した場合は、インストール先で追加権限を承認します。[GitHub App トークンの公式 Action](https://github.com/actions/create-github-app-token)

### 秘密鍵を生成して登録する

1. GitHub の個人設定で **Settings → Developer settings → GitHub Apps** を開き、対象 App の **Edit** を選びます。
2. **Credentials → Key pairs → New key** で秘密鍵を生成し、ダウンロードされた `.pem` ファイルを保存します。[GitHub App の秘密鍵の生成手順](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps#generating-private-keys)
3. `ios-ci` と通知を送る各アプリリポジトリで **Settings → Secrets and variables → Actions → Secrets → New repository secret** を開きます。名前に `IOS_CI_APP_PRIVATE_KEY` を指定し、値には同じ `.pem` の全文を開始行・終了行と改行を含めて貼り付けます。

GitHub CLI を使う場合は、ファイルを標準入力から登録できます。ファイルパスとアプリのリポジトリ名は実際の値に置き換えます。

```sh
gh secret set IOS_CI_APP_PRIVATE_KEY --repo hayabusabusa/ios-ci < /path/to/private-key.pem
gh secret set IOS_CI_APP_PRIVATE_KEY --repo hayabusabusa/example-ios < /path/to/private-key.pem
```

秘密鍵は App の全インストール先でトークンを発行できます。`.pem` はリポジトリの外に保管し、GitHub Actions には repository secret として登録します。Private のソースに由来するログが残るので、`ios-ci` の閲覧権限も対象アプリに合わせて設定してください。

以前の起動用・checkout 用の設定から移行する場合は、新しい変数とシークレットを両方のリポジトリに登録してから workflow を更新します。

## セットアップ

1. `ios-ci` のデフォルトブランチに [CI workflow](.github/workflows/ios-ci.yml) を push します。通知例はデフォルトブランチが `main` の構成です。
2. `ios-ci` の Actions variable `IOS_CI_PROJECTS` に、許可するリポジトリと Xcode の設定を JSON で登録します。キーにはアプリリポジトリの `owner/repo` を使用します。追加のアプリはこの JSON にエントリを追加します。

   ```json
   {
     "hayabusabusa/example-ios": {
       "project_path": "App.xcworkspace",
       "scheme": "App",
       "destination": "platform=iOS Simulator,name=iPhone 17,OS=27.2",
       "runner": "xcode-27",
       "xcode_version": "27.2",
       "test_targets": ["AppTests", "AppUITests"]
     }
   }
   ```

   `project_path` はアプリリポジトリのルートからの相対パスで、`.xcworkspace` と `.xcodeproj` に対応します。Scheme を Xcode で Shared に設定してコミットし、`destination` を選んだランナーと Xcode で利用できる Simulator に合わせます。

   Swift tools version 6.4 を必要とするアプリには、上の例のように `runner` に `xcode-27` を指定します。`xcode_version` はランナーにインストール済みのバージョンを文字列で指定します。`runner` を省略すると `macos-26`、`xcode_version` を省略すると選択したランナーのデフォルト Xcode が使われます。[Xcode 27 のランナーと Simulator 一覧](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md)

3. [通知例](.github/examples/notify-ios-ci.yml)を対象アプリリポジトリの `.github/workflows/notify-ios-ci.yml` としてデフォルトブランチに追加します。通知ジョブは `pull_request_target` を使って PR のメタデータを送信します。アプリのコードは `ios-ci` のジョブで取得・実行します。この通知例はアプリリポジトリ内のブランチから作成した PR が対象です。[pull_request_target の仕様](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target)
4. アプリの PR を作成するか、既存の PR にコミットを push します。対象コミットの確認が終わると、アプリ側の PR に `ios-ci / test` が保留中（`pending`）として表示され、macOS ランナーの待機中とテストの実行中はその表示が続きます。テスト終了後は同じステータスに結果が表示されます。ステータスの **Details** から `ios-ci` の実行ログを開けます。[GitHub のコミットステータス](https://docs.github.com/en/rest/commits/statuses)

デフォルトの `operation: test` は fastlane の `run_tests` でビルドとテストを実行します。ビルドだけ確認する場合は `ios-ci` の **Run workflow** で `operation: build` を選びます。fastlane の `build_app` が Simulator 向けにビルドし、結果は `ios-ci / build` に表示されます。アプリ側の **Notify iOS CI** は起動リクエストの受付結果です。

どちらのモードも `xcbeautify` で Xcode のログを整形し、警告やエラーを GitHub Actions のアノテーションとして表示します。`macos-26` と `xcode-27` には fastlane と `xcbeautify` が搭載されています。ほかのランナーを指定する場合は両方を使えるようにしてください。[fastlane のログ整形](https://docs.fastlane.tools/best-practices/xcodebuild-formatters/) · [xcbeautify の GitHub Actions 対応](https://github.com/cpisciotta/xcbeautify#github-actions)

### テスト対象を指定する

`ios-ci` の Actions variable `IOS_CI_PROJECTS` で、ビルドに使う Scheme を `scheme` に指定します。テストで別の Scheme を使う場合は `test_scheme` を追加します。`operation: test` では `scheme` でアプリをビルドしてから `test_scheme` でテストします。`test_scheme` を省略するか `scheme` と同じ名前を指定した場合は、その Scheme でビルドとテストをまとめて実行します。

`test_targets` には、テスト用 Scheme の Test アクションに含まれるテストターゲットを JSON の配列で指定します。`["AppTests", "AppUITests"]` のように複数指定でき、`"AppTests/PlaybackTests"` のようにスイート単位でも絞り込めます。

この設定は手動実行と PR からの自動実行の両方で、`operation: test` に適用されます。省略時や空配列の場合は既定のテスト設定を使います。[fastlane のテスト対象指定](https://docs.fastlane.tools/actions/run_tests/)

Swift Package の自動生成 Scheme でテストする場合は、`test_scheme` に Scheme 名、`test_package_path` にアプリルートから見た `Package.swift` のあるディレクトリを指定します。

```json
{
  "scheme": "App",
  "test_scheme": "PackageTests",
  "test_package_path": "Package"
}
```

この設定ではアプリのビルド後に、fastlane の Swift Package モードでテストします。`project_path` に `.xcworkspace` を指定している場合は、テストにも同じ workspace を渡します。[fastlane の Swift Package 設定](https://docs.fastlane.tools/actions/run_tests/)

### Swift マクロの承認

CI のビルドとテストでは、`xcodebuild` に常に `-skipMacroValidation` を渡し、その実行で使うすべての Swift マクロの承認チェックを省略します。対象アプリで使用する依存パッケージと `Package.resolved` の内容を確認してください。[Swift Forums の Xcode CLI オプション](https://forums.swift.org/t/is-there-a-way-to-programmatically-allow-trust-the-compiler-plugin-to-run-from-the-command-line/65690/2) · [Apple の CI 向け依存関係の設定](https://developer.apple.com/documentation/xcode/building-swift-packages-or-apps-that-use-them-in-continuous-integration-workflows)

## 配布

配布を追加する際は、署名とアップロードをリリース用 workflow に置き、リリースタグや手動実行から呼び出します。
