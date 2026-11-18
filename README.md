# CCM 公開 JSON 配信

Cinema Catalog Manager の承認済み JSON を GitHub Pages で配信するためのリポジトリです。
作品の探索・審査・メタデータ更新・定期処理は、管理用リポジトリと Cloudflare Worker / D1 が担当します。

## 公開 URL

- [library.json](https://qiqiqiqiqiq.github.io/cinema-catalog-public/library.json)
- [queue.json](https://qiqiqiqiqiq.github.io/cinema-catalog-public/queue.json)

初回デプロイが完了すると利用できます。VRChat 側にはこの 2 つの URL を設定します。

## 公開の仕組み

`Deploy CCM public JSON` ワークフローが、Worker から承認済みスナップショットを取得します。
`public-artifact/` 内の `library.json` と `queue.json` だけを許可し、内容が取得時のバイト列と一致することを検査して Pages Artifact にまとめます。
公開直前に D1 上の公開リビジョンを再検証し、公開後は両 URL の内容を照合して結果を D1 へ記録します。
同時公開は直列化し、中断した処理は GitHub の実行結果で終了を確認できる場合だけ回復します。

JSON は Git の履歴にコミットしません。Artifact の保存期間は 1 日です。
このリポジトリで公開されるコードは配信・回復用の最小スクリプトとそのテストだけです。
管理コード、未承認作品、YouTube API メタデータ、秘密情報は格納しません。

## 設定

GitHub の Settings → Pages → Build and deployment → Source を **GitHub Actions** にします。
Actions の Secrets and variables に次を設定します。

| 種類 | 名前 | 用途 |
| --- | --- | --- |
| Secret | `CCM_DEPLOY_TOKEN` | Worker の公開専用 API 認証 |
| Variable | `CCM_API_BASE_URL` | Worker の HTTPS URL |

YouTube API キー、Cloudflare API トークン、管理用トークンは不要です。
`GITHUB_TOKEN` は GitHub Actions がこのリポジトリに対して自動発行します。

## 起動と監視

`main` ブランチの `Deploy CCM public JSON` を `workflow_dispatch` で実行します。
管理側からの起動も同じ入口を使います。このリポジトリ自身では定期実行しません。

| Input | 内容 |
| --- | --- |
| `revision` | 期待する承認済みリビジョン。空欄なら取得時の最新値 |
| `request_id` | 管理側の実行 ID と試行番号。実行名は `CCM publish <request_id>`。手動実行では空欄可 |

入力値はシェルのコマンドへ埋め込みません。`revision` は環境変数から読み取り、Python 側で数値として検証します。
管理側は自分の `request_id` に一致するこのリポジトリの実行が成功したことと、D1 の検証済みリビジョンを確認します。
公開処理の途中失敗時は Actions の結果を確認し、停止確認できない旧処理のロックを手動で解除しないでください。

## テスト

Python 標準ライブラリだけで実行できます。テストは合成データとモックを使い、実 API へ接続しません。

```sh
python -m unittest discover -s tests -v
```

公開する 2 ファイルの制限、シンボリックリンクやハードリンクの拒否、古いリビジョン、公開後照合、D1 への成功・失敗記録、異常終了した実行の回復判定を検証します。
push / pull request 時の CI はテストだけを実行し、公開用 Secret は使いません。
