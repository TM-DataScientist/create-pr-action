# create-pr-action

`create-pr-action` は、GitHub Actions のワークフロー内で変更をコミットし、そのまま pull request を作成するための Composite Action です。

生成ファイルの更新、定期的なメンテナンス、ドキュメント自動更新のように、「ワークフローで変更を作るところ」から「PR を作ってレビューに載せるところ」までを一気に自動化したい場面を想定しています。

このリポジトリは GitHub Marketplace 公開を前提にした最小構成です。Marketplace の要件に合わせるため、`.github/workflows` のような workflow files は含めていません。

## できること

- ワークフロー実行ごとに専用ブランチを作成します
- ワークスペース内の変更をまとめてコミットします
- リモートへ push します
- 指定したメッセージで pull request を作成します
- 作成したブランチ名を output として返します

この Action が実際に行う処理は、概ね次の 4 ステップです。

1. `auto/<run_id>/<run_attempt>` 形式のブランチを作成する
2. `git add .` で変更を追加する
3. `git commit -m "<message>"` でコミットする
4. `gh pr create` で pull request を作成する

## 想定ユースケース

- ビルド成果物や生成ファイルを定期更新したい
- Lint や Formatter の結果を自動でコミットしたい
- 依存関係更新後の変更を PR として積みたい
- 毎日・毎週のレポートや集計ファイルを自動更新したい

## 前提条件

この Action を使う前に、次の条件を満たしていることを確認してください。

### 1. ワークフロー権限

呼び出し元ワークフローに、少なくとも次の権限が必要です。

```yaml
permissions:
  contents: write
  pull-requests: write
```

### 2. リポジトリ設定

GitHub Actions から pull request を作成するには、リポジトリ設定で GitHub Actions に PR 作成を許可する必要があります。

GitHub のリポジトリ設定で次を有効にしてください。

- `Settings`
- `Actions`
- `General`
- `Workflow permissions`
- `Allow GitHub Actions to create and approve pull requests`

この設定が無効だと、`gh pr create` 実行時に次のようなエラーで失敗します。

```text
pull request create failed: GraphQL: GitHub Actions is not permitted to create or approve pull requests
```

### 3. 実行環境

この Action は内部で次を使います。

- `bash`
- `git`
- `gh` (GitHub CLI)

`ubuntu-latest` の GitHub-hosted runner で使うことを前提にしています。self-hosted runner で使う場合は、これらのツールが利用可能であることを確認してください。

### 4. 参考ドキュメント

関連する公式ドキュメントです。

- GitHub Docs: https://docs.github.com/github/administering-a-repository/managing-repository-settings/disabling-or-limiting-github-actions-for-a-repository
- GitHub CLI manual (`gh pr create`): https://cli.github.com/manual/gh_pr_create

## 使い方

最小構成の例です。

```yaml
name: Update generated files

on:
  workflow_dispatch:

jobs:
  update:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - name: Generate file
        run: date > report.txt

      - name: Create pull request
        id: create-pr
        uses: TM-DataScientist/create-pr-action@v1
        with:
          message: Update generated report

      - name: Show branch name
        run: echo "${{ steps.create-pr.outputs.branch }}"
```

まだタグを切っていない開発中の検証では、同一リポジトリ内から `uses: ./` でも呼び出せます。

```yaml
- name: Exercise local action
  uses: ./
  with:
    message: Test
```

## Inputs

| Name | Required | Description |
| --- | --- | --- |
| `message` | Yes | コミットメッセージです。pull request の title と body にも同じ値が使われます。 |

## Outputs

| Name | Description |
| --- | --- |
| `branch` | 作成されたブランチ名です。形式は `auto/<run_id>/<run_attempt>` です。 |

## Pull Request の作られ方

この Action は次のような方針で pull request を作成します。

- ブランチ名は `auto/${{ github.run_id }}/${{ github.run_attempt }}` です
- コミット作者は `github-actions[bot]` です
- PR の title と body はどちらも `message` の値です
- PR の base branch は `gh pr create` の既定動作に従います

通常はデフォルトブランチに対して PR が作成されますが、`gh` 側の設定によっては挙動が変わる可能性があります。

## 注意点

### 変更がないと失敗します

この Action は `git commit` をそのまま実行するため、コミット対象の変更が 1 つもない場合は失敗します。

つまり、次のような流れは失敗します。

- `actions/checkout` だけ実行する
- 何もファイルを変更しない
- この Action を呼ぶ

Action を呼ぶ前に、必ずコミット対象の差分を作ってください。

### `git add .` ですべて追加します

この Action は `git add .` を使います。ワークスペース内の変更を個別に選別せず、まとめて追加します。

そのため、次のようなケースでは注意が必要です。

- 生成物以外の変更も同じジョブ内に混ざっている
- 不要な一時ファイルが残っている
- 意図しないファイル更新が発生している

必要であれば、この Action を呼ぶ前にワークスペースを整理してください。

### PR 本文はカスタマイズできません

現在の実装では、PR 本文は `message` と同じ内容になります。タイトルと本文を分けたい場合や、テンプレート化したい場合は Action の拡張が必要です。

## サンプル: 定期実行でドキュメントを更新する

```yaml
name: Refresh docs

on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:

jobs:
  refresh:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - name: Update docs
        run: |
          date > docs/generated.md

      - name: Create PR
        uses: TM-DataScientist/create-pr-action@v1
        with:
          message: Refresh generated docs
```

## テスト方法

GitHub Marketplace に公開する action repository には workflow files を含められません。そのため、この公開用リポジトリには CI 用 workflow を置いていません。

検証したい場合は、次のいずれかの方法を取るのが安全です。

1. 別の開発用リポジトリを作り、そこからこの Action を呼び出してテストする
2. 一時的なローカル検証用 workflow を別リポジトリに作り、`uses: owner/repo@ref` で試す
3. Marketplace 公開前の作業ブランチでだけ workflow を使い、公開用タグには含めない

最小の検証例は次のようになります。

```yaml
name: Test create-pr-action

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - name: Create test file
        run: date > foo.md

      - name: Run action
        id: exercise
        uses: TM-DataScientist/create-pr-action@v1
        with:
          message: Test
```

## リリース方法

この公開用リポジトリでは、Marketplace 要件に合わせて release 用 workflow も含めていません。リリースは GitHub の Release 機能から手動で行います。

基本手順は次のとおりです。

1. `main` に公開したい状態を反映する
2. `action.yml` の公開メタデータを確認する
3. `vX.Y.Z` 形式のタグを作成する
4. GitHub Releases で新しい release を作成する
5. `Publish this Action to the GitHub Marketplace` を有効にして公開する

利用者には、固定のフルバージョンよりも `@v1` のようなメジャータグを案内するのが一般的です。

もし CI やリリース自動化を維持したい場合は、公開用 repo と開発用 repo を分ける運用が実務上は扱いやすいです。

## リポジトリ構成

```text
.
├─ action.yml
└─ README.md
```

## 補足

この Action はシンプルさを優先した実装です。用途によっては、今後次のような拡張が考えられます。

- PR 本文の個別指定
- base branch の明示指定
- 変更がない場合のスキップ
- 特定パスのみを add するオプション

必要であれば、これらを追加しやすいように Composite Action のまま保っています。
