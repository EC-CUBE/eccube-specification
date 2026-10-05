# EX02_運用CLI_結合試験項目書

EX は画面を持たない機能（API・CLI）の結合試験項目です。

読み取り専用モード（EA13）で管理画面から保存できない操作の代わりに使う CLI（EX0201）、権限診断（EX0202）、権限分離構成でのキャッシュ（EX0203）、返品申請の一時ファイル掃除（EX0204）を扱う。

関連 Issue / PR: [EC-CUBE/ec-cube#7072](https://github.com/EC-CUBE/ec-cube/issues/7072)（親 Issue）、#7098 / #7100 / #7105 / #7114 / #7117 / #7119 / #7121 / #7124 / #6959。EX0204 は [EC-CUBE/ec-cube#6838](https://github.com/EC-CUBE/ec-cube/pull/6838)

### 事前準備

#### 権限分離構成（Web サーバーと CLI を別ユーザーにする）

本体リポジトリの `docker-compose.permission-lanes.yml` を重ねて起動する（`.github/workflows/permission-lanes-test.yml` と同じ構成）。

- `--build` は必須。公開イメージには権限分離のエントリポイントが含まれない
- DB は PostgreSQL または MySQL が必須。SQLite は Web と CLI の双方が同じファイルへ書くため起動を中止する
- 分離モードは `APP_ENV=prod` 固定。prod のセッション cookie は `SameSite=None` のため HTTP では管理画面にログインできない。HTTPS（自己署名、ポート 4430）でアクセスする
- `ECCUBE_RESTRICT_FILE_UPLOAD=1`、`ECCUBE_CLI_LOG_TO_FILE=0`、`ECCUBE_MAINTENANCE_FILE_PATH=/var/www/html/var/runtime/.maintenance`（絶対パス）は compose ファイル側で設定済み

```
docker compose -f docker-compose.yml -f docker-compose.dev.yml -f docker-compose.pgsql.yml -f docker-compose.permission-lanes.yml up -d --build
```

起動後、アクセスを受ける前にビルドディレクトリを生成し、続けてセッションを 1 件作る（`eccube:doctor:permissions` が Web サーバーの uid をセッションファイルの所有者から判定するため）。

```
docker compose exec -u <CLI_USER> ec-cube bin/console eccube:cache:build
curl -k -s -o /dev/null https://127.0.0.1:4430/
curl -k -s -o /dev/null https://127.0.0.1:4430/<ADMIN_ROUTE>/login
```

CLI ユーザー名（`<CLI_USER>`）は `composer.json` の所有者 uid を `getent passwd <uid>` で引く。既定では `eccube`（uid はホストユーザーに合わせる）。Web サーバーは `www-data`（uid 33）のまま。

所有者の期待値は次のとおり。

| レーン | ディレクトリ | 所有者 |
|---|---|---|
| S（CLI が書く） | `var/build` `var/cache` `app/template` `app/keystore` `html/user_data` `app/Plugin` `vendor` `.env` | CLI ユーザー |
| W（Web サーバーが書く） | `var/runtime` `var/sessions` `var/log` `html/upload` `html/upload/save_image` `html/upload/temp_image` `html/upload/refund_request` `html/upload/refund_request/save` `html/upload/refund_request/temp` | `www-data` |

`html/upload` 配下で `www-data` の所有になるのは上記のディレクトリ自体だけで、同梱のファイル（`html/upload/refund_request/.htaccess` や既定の画像）は CLI ユーザーの所有のまま残る。

以降の CLI 項目は、特記がなければ CLI ユーザーで実行する。docker 構成では `docker compose exec -T -u <CLI_USER> ec-cube` を前置する（標準入力を使うときは `-T` が必要）。レーン W を触る操作は `-u www-data` で実行する。

#### キャッシュの再生成

| 操作 | コマンド | 実行ユーザー |
|---|---|---|
| コンパイル済みコンテナ・テンプレート（レーン S） | `bin/console eccube:cache:build` | CLI ユーザー |
| 実行時 cache pool（レーン W） | `bin/console cache:pool:clear --all` | Web サーバー |
| 実行時 twig（`var/runtime/<env>/twig`） | 管理画面のキャッシュ管理、または Web サーバーのユーザーで `rm -rf` | Web サーバー |

#### CLI の終了コード

| コード | 意味 |
|---|---|
| 0 | 正常終了 |
| 1 | 対象が見つからない / 入力値が不正 / 書き込み失敗。plugin 系（`eccube:plugin:enable` 等）で `--code` を省略した場合も 1 |
| 2 | オプション・引数が不正（`Command::INVALID`。形式の誤り、未知の値、同時に指定できないオプションの併用等） |
| 3 | 本処理は完了したが手動操作が必要（キャッシュを削除できず `eccube:cache:build` 等の実行が要る）。`eccube:cache:build` 自体は、書き込み権限が無く何もせずに終了した場合に 3 を返す |

権限分離構成では、テンプレートを更新するコマンドは本処理を完了したうえで、Web サーバー所有（レーン W）の `var/runtime/<env>/pools`（フロントを表示した後は `var/runtime/<env>/twig` も）を削除できないため終了コード 3 を返す。これは想定どおりの成功系として扱う。`var/build/<env>/twig` は CLI ユーザーの所有のため削除され、警告は出ない。

#### 後片付け

```
docker compose -f docker-compose.yml -f docker-compose.dev.yml -f docker-compose.pgsql.yml -f docker-compose.permission-lanes.yml down -v
sudo chown -R <UID>:<GID> html/upload
```

#### 返品申請の一時ファイル掃除（EX0204）

- EF10 の事前準備（会員A・対象受注・商品P）を済ませておく
- `html/upload/refund_request/temp` はレーン W のため、権限分離構成では Web サーバーのユーザーで実行する

## EX0201-UC01-T01_eccube:page:apply（新規登録）

1. 任意の twig ファイル（例: `{% extends 'default_frame.twig' %}` から始まる本文）を用意する
1. 次を実行する

   ```
   cat <ファイルパス> | bin/console eccube:page:apply --route=<ルート名> --name=<ページ名> --body=-
   ```

1. `created: <ルート名>` と表示され、終了コードは 0（権限分離構成では `…/var/runtime/<env>/pools を削除できないため, 実行時キャッシュに古い内容が残ります.` の警告と終了コード 3）
1. `dtb_page` にレコードが追加され、`app/template/user_data/<ルート名>.twig` が作成される
1. 権限分離構成では続けて `bin/console eccube:cache:build` を実行する
1. フロントで `/user_data/<ルート名>` にアクセスすると 200 でページが表示される
1. 管理画面のページ管理一覧に登録したページが表示される

## EX0201-UC01-T02_eccube:page:apply（冪等・dry-run・JSON）

1. EX0201-UC01-T01 と同じ入力で再度 `eccube:page:apply` を実行する
1. `unchanged: <ルート名>` と表示され、終了コードは 0。ファイルの更新日時は変わらない
1. 本文を変更して `--dry-run` を付けて実行する

   ```
   bin/console eccube:page:apply --route=<ルート名> --body-file=<ファイルパス> --dry-run
   ```

1. 変更前後の差分が `- ` / `+ ` 行で表示され、「dry-run のため適用していません」と表示される。DB・ファイルは変更されない
1. `--format=json` を付けて実行すると `dry_run` / `status` を含む JSON が出力される

## EX0201-UC01-T03_eccube:page:apply（既定ページのメタ情報のみ更新）

1. 既定ページ（例: `--route=entry`）に対し本文を指定せずページ名だけを変更する

   ```
   bin/console eccube:page:apply --route=entry --name=<新しいページ名>
   ```

1. `updated: entry` と表示される
1. `app/template/default/` 配下に `Entry/index.twig` の写しが作成されない（`git status app/template` に差分が出ない）
1. ページ管理一覧でページ名が変わっている

## EX0201-UC01-T04_eccube:page:apply（異常系）

1. `--route` を省略して実行すると「--route を指定してください.」と表示され、終了コードは 2
1. `--format=yaml` を指定すると「--format は table / json のいずれかで指定してください.」と表示され、終了コードは 2
1. `--body` と `--body-file` を同時に指定すると終了コードは 2
1. 構文の誤った twig（例: `{% if %}` の閉じ忘れ）を本文に渡すと「ページを保存できません」とバリデーションエラーが表示され、終了コードは 1。DB・ファイルは変更されない
1. 既定ページ（例: `--route=entry`）に現在と異なる `--file-name` だけを指定すると、指定は無視されて `unchanged: entry` と表示され、終了コードは 0。ファイル名は変わらない（既定ページはファイル名を変更できない）

## EX0201-UC01-T05_eccube:page:show / list

1. `bin/console eccube:page:show --route=<ルート名>` を実行すると、テンプレートの本文がそのまま標準出力に出る（`> <ファイルパス>` でファイルへ書き出せる）
1. 存在しないルート名を指定すると「ページが見つかりません」と表示され、終了コードは 1
1. `bin/console eccube:page:list` を実行すると ID / ルーティング名 / ページ名 / ファイル名 / 編集可 の表が出力され、EX0201-UC01-T01 で登録したページが含まれる
1. `--format=json` で JSON の配列が出力される

## EX0201-UC01-T06_eccube:page:remove

1. 既定ページを指定して実行する

   ```
   bin/console eccube:page:remove --route=help_guide --force
   ```

1. 「既定ページのため削除できません」と表示され、終了コードは 1
1. ユーザー作成ページを `--force` なし・`--no-interaction` 付きで実行すると「中止しました.」と表示され、終了コードは 1。削除されない
1. `--force` を付けて実行すると `removed: ...` と表示され、`dtb_page` のレコードと `app/template/user_data/<ルート名>.twig` が削除される（終了コードは 0 または 3）
1. フロントで該当 URL にアクセスすると 404 になる

## EX0201-UC02-T01_eccube:block:apply / show / list

1. 任意の HTML を本文としてブロックを登録する

   ```
   printf '<div class="ec-cli-block"><p>CLI ブロック</p></div>\n' | bin/console eccube:block:apply --file-name=<ファイル名> --name=<ブロック名> --body=-
   ```

1. `created: <ファイル名>` と表示される。`dtb_block` にレコードが追加され `app/template/<テーマ>/Block/<ファイル名>.twig` が作成される
1. 同じ入力で再実行すると `unchanged` で終了コードは 0
1. `bin/console eccube:block:show --file-name=<ファイル名>` で本文が出力される
1. `bin/console eccube:block:list` に登録したブロックが含まれる
1. 管理画面のブロック管理一覧に表示され、レイアウト管理で配置するとフロントに表示される（レイアウト管理は読み取り専用モードの対象外で、権限分離構成でも管理画面から配置できる）

## EX0201-UC02-T02_eccube:block:remove

1. 削除不可のブロック（初期データのブロック）を指定して `--force` で実行すると「削除できないブロックです」と表示され、終了コードは 1
1. EX0201-UC02-T01 で登録したブロックを `--force` で削除すると、`dtb_block` のレコードと twig ファイルが削除される
1. 存在しないファイル名を指定すると「ブロックが見つかりません」で終了コードは 1
1. `--device-type` に存在しない ID を指定すると終了コードは 2

## EX0201-UC03-T01_eccube:mail-template:list / show / apply

1. `bin/console eccube:mail-template:list` で初期データのテンプレート（注文受付メール等）が一覧に出る
1. 件名を変更する

   ```
   bin/console eccube:mail-template:apply --id=1 --subject=<件名> --dry-run
   bin/console eccube:mail-template:apply --id=1 --subject=<件名>
   ```

1. dry-run では `mail_subject: <旧> -> <新>` が表示されるだけで DB は変わらない。本実行で `updated` となり、管理画面のメール設定で件名が変わっている
1. 新規テンプレートを登録する

   ```
   cat <ファイルパス> | bin/console eccube:mail-template:apply --file-name=<ファイル名> --name=<テンプレート名> --subject=<件名> --body=-
   ```

1. `created` と表示され、`dtb_mail_template` にレコードが追加され `app/template/<テーマ>/Mail/<ファイル名>.twig` が作成される
1. `bin/console eccube:mail-template:show --file-name=<ファイル名>` で本文が出力される。この時点では HTML パートが無いため、`--html` を付けると「HTML パートがありません: Mail/<ファイル名>.twig」と表示され、終了コードは 1

## EX0201-UC03-T02_eccube:mail-template:apply（HTML パート）

1. `--html-body-file=<ファイルパス>` を付けて apply すると `Mail/<ファイル名>.html.twig` が作成される。`eccube:mail-template:show --file-name=<ファイル名> --html` で HTML パートが出力される
1. `--html-body` を指定せずに本文だけを更新しても HTML パートは維持される
1. `--remove-html` を付けて apply すると HTML パートのファイルが削除される
1. `--remove-html` と `--html-body` を同時に指定すると終了コードは 2
1. `--body=-` と `--html-body=-` の双方に `-` を指定すると終了コードは 2

## EX0201-UC03-T03_eccube:mail-template:remove

1. 初期データのテンプレート（`--id=1` 等）を `--force` で削除しようとすると「削除できないメールテンプレートです」で終了コードは 1
1. EX0201-UC03-T01 で登録したテンプレートを `--force` で削除すると、レコードと twig（HTML パートがあればそれも）が削除される
1. 管理画面のメール設定の選択肢から消えている

## EX0201-UC04-T01_eccube:asset:show / apply（CSS / JS）

1. `bin/console eccube:asset:show --type=css > <ファイルパス>` で現在の `customize.css` が取り出せる
1. 内容を編集して適用する

   ```
   cat <ファイルパス> | bin/console eccube:asset:apply --type=css --body=- --dry-run
   cat <ファイルパス> | bin/console eccube:asset:apply --type=css --body=-
   ```

1. dry-run では差分のみ表示。本実行で `updated: css` と表示され、終了コードは 0（静的ファイルのためキャッシュ削除は不要で、権限分離構成でも 3 にならない）
1. 同じ入力で再実行すると `unchanged`
1. フロントで `customize.css` を取得すると適用した内容になっている。管理画面の CSS管理 にも同じ内容が表示される
1. `--type=js` でも同様に適用できる
1. `--type=scss` など未知の種別を指定すると終了コードは 2。`--body` / `--body-file` のいずれも指定しないと終了コードは 2

## EX0201-UC05-T01_eccube:user-data:list / put / show

1. `bin/console eccube:user-data:list` で `html/user_data` 直下の一覧が出る。`--recursive --format=json` で配下を含む JSON が出る
1. 中間ディレクトリを含むパスへファイルを配置する

   ```
   printf '<h1>CLI</h1>\n' | bin/console eccube:user-data:put --path=<ディレクトリ>/<ファイル名>.html --body=-
   ```

1. `created` と表示され、中間ディレクトリが自動作成されて `html/user_data/<ディレクトリ>/<ファイル名>.html` が置かれる
1. `bin/console eccube:user-data:show --path=<ディレクトリ>/<ファイル名>.html` で内容が出力される
1. フロントで `/html/user_data/<ディレクトリ>/<ファイル名>.html`（ドキュメントルートが `html/` の構成では `/user_data/...`）にアクセスすると配置した内容が表示される
1. 管理画面のファイル管理で該当ディレクトリとファイルが表示される
1. PNG 等のバイナリを `--body=-` で put し、`show` の出力を元ファイルと比較すると一致する。`show --format=json` ではバイナリは `body_base64` に入る

## EX0201-UC05-T02_eccube:user-data:put（拒否される入力）

いずれも終了コードは 1 でファイルは作成されない。

1. `--path=evil.php`（許可されていない拡張子）→「アップロードできないファイル拡張子です。」
1. `--path=.htaccess`（dotfile）→「.で始まるファイルはアップロードできません。」
1. `--path="'quote'.txt"`（使用できない文字）→「使用できない文字が含まれています。」
1. `--path=../../.env`（ディレクトリトラバーサル）→「html/user_data の配下ではありません」
1. `eccube:user-data:show --path=../../.env` も同様に拒否される

## EX0201-UC05-T03_eccube:user-data:remove

1. `--path=/ --force` を実行すると「html/user_data 自体は削除できません.」で終了コードは 1
1. 空でないディレクトリを `--recursive` なしで `--force` 削除しようとすると「--recursive を指定してください」で終了コードは 1
1. `--force` なし・`--no-interaction` 付きでファイルを削除しようとすると「中止しました.」で終了コードは 1
1. `--path=<ディレクトリ> --recursive --force` で配下ごと削除される。`--dry-run` を付けると削除対象の表示のみで削除されない
1. 管理画面のファイル管理から該当ファイルが消えている

## EX0201-UC06-T01_eccube:env:get

1. `bin/console eccube:env:get APP_ENV` で実行時の値が標準出力に出る
1. `bin/console eccube:env:get ECCUBE_TEMPLATE_CODE --format=json` で `key` / `file_value` / `effective_value` / `overridden` / `ineffective_reasons` を含む JSON が出る
1. `.env` にもプロセス環境にも無いキーを指定すると「設定されていません」で終了コードは 1
1. OS の環境変数で上書きされているキー（docker 構成では `APP_ENV` 等）を指定すると、標準エラー出力に「.env ではなく OS の環境変数 … の値が使われています」の警告が出て、標準出力には実行時の値が出る

## EX0201-UC06-T02_eccube:env:set（正常系）

1. 差分の確認

   ```
   bin/console eccube:env:set ECCUBE_FORCE_SSL=0 --dry-run
   ```

1. `ECCUBE_FORCE_SSL: <旧値> -> 0` と「dry-run のため適用していません.」が表示され、`.env` は変更されない
1. 本実行

   ```
   bin/console eccube:env:set ECCUBE_TEMPLATE_CODE=<コード>
   ```

1. `.env を更新しました: ECCUBE_TEMPLATE_CODE` と表示され、続けて `Run bin/console eccube:cache:build` が別プロセスで実行される。終了コードは 0
1. `.env` の該当行が書き換わり、所有者とパーミッションは変わらない
1. 同じ値で再実行すると「変更はありません.」で終了コードは 0
1. 複数キーを一度に渡すと（`KEY1=V1 KEY2=V2`）すべて書き込まれ、`eccube:cache:build` は 1 回だけ実行される
1. `--no-cache-clear` を付けると `eccube:cache:build` が実行されない
1. 管理画面のテンプレート一覧で選択中のテンプレートが変わっている（`ECCUBE_TEMPLATE_CODE` の場合）

## EX0201-UC06-T03_eccube:env:set（異常系・手動操作が必要）

1. `NOT_AN_ASSIGNMENT`（`=` なし）を渡すと「KEY=VALUE の形式で指定してください」で終了コードは 2
1. `1KEY=x` のように環境変数名として不正なキーを渡すと終了コードは 2
1. `.env` が無い状態で実行すると「.env を作成してから実行してください」で終了コードは 1
1. `.env` に書き込み権限が無いユーザーで実行すると「.env を所有するユーザー (レーン S) で実行してください」と `eccube:doctor:permissions` の案内が出て、終了コードは 1
1. `.env.local.php` が存在する状態で実行すると、`.env` は書き込まれたうえで「.env.local.php があるため, .env の変更は実行時に反映されません.」と `composer dump-env` の案内が出て、終了コードは 3
1. OS の環境変数で上書きされているキー（docker 構成では `APP_ENV` 等）を設定すると、書き込まれたうえで「OS の環境変数 (またはカスケードファイル) が優先されるため, 次のキーは .env を変更しても反映されません: <キー>」と出て、終了コードは 3

## EX0201-UC07-T01_eccube:plugin:install / enable（CLI からの導入）

権限分離構成で実施する。

1. プラグインのアーカイブ（tar.gz）を用意し、CLI ユーザーで実行する

   ```
   bin/console eccube:plugin:install --path=<アーカイブ>
   bin/console eccube:plugin:enable --code=<コード>
   ```

1. いずれも `Installed.` / `Plugin Enabled.` と表示される。終了コードは 0、またはキャッシュ削除に失敗した旨の警告付きで 3
1. この時点では `eccube:cache:build` を実行せずにフロントや管理画面へアクセスすると 500 になり得るため、続けて実行する

   ```
   bin/console eccube:cache:build
   ```

1. フロントと管理画面ログインが 200 で表示される
1. 管理画面のプラグイン一覧に導入したプラグインが「有効」で表示され、無効化ボタンは無効化されている

## EX0201-UC07-T02_eccube:plugin:disable / update / uninstall

1. `bin/console eccube:plugin:disable --code=<コード>` で無効化され、プラグイン一覧の表示が「無効」になる
1. `bin/console eccube:plugin:update <コード>` で `Updated.` と表示される
1. `bin/console eccube:plugin:uninstall --code=<コード> --uninstall-force` でアンインストールされ、一覧から消える
1. いずれも実行後に `eccube:cache:build` を実行してからサイトを確認する
1. 存在しないコードを指定すると、いずれも終了コードは 1。メッセージは disable が「Plugin `<コード>` is not found.」、update が「No such plugin `<コード>`.」、uninstall が「Plugin `<コード>` is not installed.」

## EX0201-UC08-T01_eccube:keystore:generate / list / show

権限分離構成で実施する。

1. 鍵が無い状態で `bin/console eccube:keystore:list` を実行すると `acp_webhook` / `ucp_signing` が「未生成」で表示される
1. フロントで `/.well-known/ucp` にアクセスすると 500 になる（Web サーバーが `app/keystore` に書き込めず実行時生成できない）
1. 生成する

   ```
   bin/console eccube:keystore:generate
   ```

1. 2 件とも `created`、Web サーバー列が「読み取り可」、「生成: 2 件 / 変更なし: 0 件」で終了コードは 0。`app/keystore` 配下の所有者は CLI ユーザー
1. 再度実行すると 2 件とも `unchanged`、「生成: 0 件 / 変更なし: 2 件」で終了コードは 0
1. `/.well-known/ucp` が 200 を返し、`signing_keys[]` に `kid` が載る。秘密鍵の `d` は含まれない
1. `bin/console eccube:keystore:show ucp_signing` に `algorithm: ES256` と `kid` が表示され、discovery の `kid` と一致する。鍵素材は表示されない
1. `bin/console eccube:keystore:show acp_webhook` に `algorithm: HMAC-SHA256` と `length` のみ表示され、シークレットの値は表示されない

## EX0201-UC08-T02_eccube:keystore:generate（異常系・差し替え）

1. 鍵がある状態で `bin/console eccube:keystore:generate ucp_signing --dry-run` を実行すると `unchanged` と表示される（`--force` を付けない限り既存の鍵は対象にならない）。`--force --dry-run` では `would_replace` を表示するだけで鍵を書かない（鍵が無い状態では `--dry-run` で `would_create`）
1. `bin/console eccube:keystore:generate unknown_purpose` は終了コードが 2
1. `bin/console eccube:keystore:show unknown_purpose` は終了コードが 2。鍵未生成の用途を `show` すると「生成されていません」で終了コードは 1
1. `bin/console eccube:keystore:generate ucp_signing --force` を実行すると `replaced` となり「既存の鍵を差し替えました. 広告済みの公開鍵 (kid) が変わる」旨の警告が出る。`/.well-known/ucp` の `kid` が変わる
1. `ECCUBE_KEYSTORE_STRICT_PERMISSIONS=1` を付けて `--force` 生成すると、鍵は 0600 で作られ「鍵ファイルの権限が 0600 のため Web サーバー … から読み取れません」と `chmod 0644` の案内が出て、終了コードは 1。この状態で `/.well-known/ucp` は 500 になる
1. 環境変数を外して `--force` 生成し直すと 200 に戻る

## EX0201-UC09-T01_eccube:contents:export

1. 既定の出力先へ書き出す

   ```
   bin/console eccube:contents:export
   ```

1. `app/contents/` に `manifest.yaml` / `layouts.yaml` / `blocks.yaml` / `pages.yaml` / `mail_templates.yaml` が作成され、セクション / 件数 / 出力先の表が表示される
1. 各 yaml にテンプレートの本文（twig の中身）は含まれない
1. `--to=<ディレクトリ>` で別ディレクトリへ書き出せる。同じ内容を 2 回書き出して `diff -r` すると差分が無い
1. `--dry-run` では「dry-run のため書き出していません.」でファイルは作られない
1. `--only=pages,blocks` で指定したセクションだけ、`--exclude=mail_templates` で除外したもの以外が出力される
1. 既定では `user_data` セクションは出力されず、`--include=user_data` を指定したときのみ `html/user_data` 配下が含まれる
1. `--only=unknown` を指定すると「未知のセクションです」で終了コードは 2
1. `dtb_layout` に同名のレイアウトが 2 件ある状態で export すると、名前が一意でない旨のエラーで終了コードは 1

## EX0201-UC09-T02_eccube:contents:import（冪等・dry-run）

1. EX0201-UC09-T01 で書き出した直後に取り込む

   ```
   bin/console eccube:contents:import --dry-run
   bin/console eccube:contents:import
   ```

1. 「対象 <N> 件 / 変更 0 件」と表示され、DB・`app/template` に変化が無い
1. `pages.yaml` の任意のページ名を書き換えて `--dry-run` を実行すると、そのページだけが `updated` として表に出て、DB は変わらない
1. 本実行で `updated` となり「取り込みました.」が表示される。終了コードは 0（権限分離構成では 3）。管理画面のページ管理でページ名が変わっている
1. `--format=json` で `results` / `warnings` を含む JSON が出力される

## EX0201-UC09-T03_eccube:contents:import（コミット済みテンプレートからページを作る）

1. `app/template/user_data/<ルート名>.twig` を配置し、`pages.yaml` にそのルート名のページ定義を 1 件追加する
1. `bin/console eccube:contents:import` を実行すると `created` となり、`dtb_page` にレコードが作られる。テンプレートの内容は変更されない
1. フロントで `/user_data/<ルート名>` が表示される
1. `dtb_page` に行がある既存ページのテンプレートファイルを削除し、そのページの行を `pages.yaml` に残したまま import すると「テンプレートが見つかりません: <パス>. リポジトリへ配置するか, アーカイブから該当の行を削除してください.」で終了コードは 1
1. `pages.yaml` に定義があるが `dtb_page` にもテンプレートファイルにも無い新規ルート名を追加して import すると「tpl_data: 入力されていません。」で終了コードは 1
1. `--continue-on-error` を付けると、エラーの行を飛ばして残りを取り込み、終了コードは 1

## EX0201-UC09-T04_eccube:contents:import（検証・--prune）

1. `manifest.yaml` の `schema` を存在しない版に書き換えて import すると「アーカイブの書式版 … はこのバージョンでは扱えません」で終了コードは 1
1. `manifest.yaml` の `template_code` を現在のテーマと異なる値にして import すると警告が表示される
1. `pages.yaml` の鍵（ルート名）に `../evil` のような不正な値を入れると取り込みを拒否して終了コードは 1
1. `pages.yaml` にレイアウト名として存在しない名前を書くと終了コードは 1
1. アーカイブから EX0201-UC01-T01 で作成したユーザーページの行を削除し、`--prune --dry-run` を実行すると、そのページだけが `removed` 対象として表示される。既定ページ・削除不可のブロック・メールテンプレートは対象に挙がらない
1. `--prune` を本実行すると該当ページの `dtb_page` レコードとテンプレートが削除される
1. `--include=user_data` で `.php` を含む `user_data` を取り込むと「配置できないため読み飛ばしました」の警告を出してスキップする

## EX0201-UC10-T01_実行ユーザーを誤った場合（Web サーバーのユーザーで実行）

権限分離構成で `-u www-data` を付けて実行する。

1. ページを登録する

   ```
   printf '<p>lane test</p>' | bin/console eccube:page:apply --route=<ルート名> --name=<ページ名> --body=-
   ```

1. 「ページを保存できません」「書き込み先 (レーン S) を所有するユーザーで実行してください. 期待値と実際の所有者は bin/console eccube:doctor:permissions で確認できます.」と表示され、終了コードは 1
1. `dtb_page` にレコードが作られておらず、`app/template` にファイルも残っていない
1. `eccube:asset:apply --type=css --body=-` も同様に終了コード 1 で `eccube:doctor:permissions` を案内する
1. `eccube:env:set ECCUBE_TEMPLATE_CODE=default` も「.env を所有するユーザー (レーン S) で実行してください」で終了コードは 1
1. `eccube:keystore:generate --force` も「<用途> の鍵を保管できませんでした.」「保管先 (レーン S) を所有するユーザーで実行してください」で終了コードは 1（`--force` を付けないと、鍵がある状態では書き込みを試みず `unchanged` で終了コード 0）
1. `eccube:cache:build` は「へ書き込めません」と `eccube:doctor:permissions` の案内で終了コードは 3

## EX0202-UC01-T01_eccube:doctor:permissions（正常な分離構成）

1. 事前準備どおりセッションを作成した状態で、CLI ユーザーで実行する

   ```
   bin/console eccube:doctor:permissions
   ```

1. 「Web サーバーの実行ユーザー: uid=33 gid=33 (var/sessions/prod/sess_… の所有者から判定)」と「診断の実行ユーザー: uid=<CLI uid>」が表示される
1. レーン / パス / 所有者 / 権限 / 判定 の表が表示され、`var/runtime/prod` `var/runtime`（メンテナンスファイルの生成先） `var/log` `var/sessions/prod` `html/upload/save_image` `html/upload/temp_image` `html/upload/refund_request/save` `html/upload/refund_request/temp` は `web` レーンで「Web サーバーから書き込めます」、`var/build/prod` `var/cache/prod` `app/template` `html/user_data` `app/Plugin` `vendor` `.env` 等は `ssh` レーンで「Web サーバーは読み取りのみです」
1. 末尾に「判定は所有者 uid / グループ gid / パーミッションビットからの推定です」の注意書きと `OK: <件数> / WARN: <件数> / NG: 0` が表示され、終了コードは 0（OK・WARN の件数は、任意のディレクトリの有無などの環境によって変わる）

## EX0202-UC01-T02_eccube:doctor:permissions（--format=json）

1. `bin/console eccube:doctor:permissions --format=json` を実行する
1. `web_server`（`uid` / `gid` / `source`）、`cli`（`uid` / `gid`）、`summary`（`ok` / `warn` / `ng`）、`findings`（`path` / `lane` / `severity` / `permissions` 等）を持つ JSON が出力される
1. `summary.ng` が 0、`web_server` が null でなく、`cli.uid` と `web_server.uid` が異なる
1. `--format=yaml` を指定すると終了コードは 2

## EX0202-UC01-T03_eccube:doctor:permissions（Web サーバーの uid を判定できない）

1. `var/sessions/prod` のセッションファイルと `html/upload/temp_image` 等の Web 生成ファイルが無い状態（起動直後にアクセスしていない状態）で実行する
1. 「Web サーバーの実行ユーザーを特定できませんでした.」「サイトへ一度アクセスしてから再実行するか …」の警告が表示される
1. 各行の判定が「Web サーバーの実行ユーザーを特定できないため判定できません」の WARN になり、終了コードは 0
1. サイトへアクセス後に再実行すると EX0202-UC01-T01 の結果になる

## EX0202-UC01-T04_eccube:doctor:permissions（NG の検出）

1. レーン S のディレクトリを Web サーバーから書き込めるようにする（例: root で `chmod o+w app/template`）
1. 実行すると `app/template` が `[NG]`「任意のローカルユーザーから書き込めます (想定: 読み取りのみ)」となり、`ECCUBE_UMASK` に関する対処が表示される。終了コードは 1
1. `chown www-data app/template` のように所有者を Web サーバーにした場合は「Web サーバーから書き込み可能です (想定: 読み取りのみ)」の NG になる
1. レーン W（例: `var/runtime`）の所有者を CLI ユーザーにすると「Web サーバーから書き込めません」の NG になる
1. 元に戻すと `NG: 0` で終了コードは 0

## EX0202-UC01-T05_eccube:doctor:permissions（権限を分離していない構成）

1. 既定の docker 構成（override なし、Web と CLI が同じ uid）で、`docker compose exec -u www-data ec-cube bin/console eccube:doctor:permissions` を実行する（root で実行すると Web サーバーと uid が異なるため、次の注記は出ない）
1. 「Web サーバーと診断の実行ユーザーが同じ uid です.」「共有ホスティング … 権限によるレーンの分離はできません.」の注記が表示される
1. レーン S の各行が「Web サーバーから書き込み可能です (想定: 読み取りのみ)」の NG になり、終了コードは 1

## EX0202-UC01-T06_eccube:doctor:permissions（事前コンパイル漏れの検出）

1. 権限分離構成（prod）で `var/build/prod/twig` を空にした状態で Web からページを表示し、`var/runtime/prod/twig` に `.php` が生成された状態にする
1. 実行すると `var/runtime/prod/twig` の行が `[WARN]`「事前コンパイルされていないテンプレートがリクエスト処理中にコンパイルされています」となり、`eccube:cache:build` の実行が案内される
1. `eccube:cache:build` を実行し、Web サーバーのユーザーで `var/runtime/prod/twig` を削除して再実行すると WARN が消える

## EX0203-UC01-T01_eccube:cache:build（正常系）

1. CLI ユーザーで実行する

   ```
   bin/console eccube:cache:build
   ```

1. 「ビルドディレクトリを再生成します (環境: prod, デバッグ: false)」「テンプレートを事前コンパイルしています...」「"prod" 環境のビルドディレクトリを生成しました.」が表示され、終了コードは 0
1. `var/build/prod` にコンパイル済みコンテナ（`Eccube_KernelProdContainer.php`）と `twig/` 配下の事前コンパイル結果が生成される。所有者は CLI ユーザー
1. `var/runtime/prod` 配下は変更されない（所有者は `www-data` のまま）
1. `bin/console about` の出力で Cache directory（`var/cache/prod`）、Build directory（`var/build/prod`）、Share directory（`var/runtime/prod`）が別パスになっている
1. `--no-twig` を付けると「テンプレートを事前コンパイルしています...」が表示されず `var/build/prod/twig` は生成されない

## EX0203-UC01-T02_実行時キャッシュの分離（アクセス前にビルド）

1. `var` ボリュームを作り直し（`down -v` → `up`）、Web へアクセスする前に CLI ユーザーで `eccube:cache:build` を実行する
1. フロントトップ、商品一覧、管理画面ログインへアクセスする
1. `var/build/prod/twig` 配下にテンプレートが事前コンパイルされており、`var/runtime/prod/twig` は存在しないか 0 件のまま
1. `var/runtime/prod/pools` に cache pool が生成され、所有者は `www-data`

## EX0203-UC01-T03_eccube:cache:build（権限不足）

1. Web サーバーのユーザー（`-u www-data`）で `eccube:cache:build` を実行する
1. 「<パス> へ書き込めません.」「ビルドディレクトリの生成には kernel.build_dir (とその親ディレクトリ) と kernel.cache_dir への書き込み権限が必要です.」と `bin/console eccube:doctor:permissions` の案内が表示され、終了コードは 3
1. `var/build/prod` は変更されない

## EX0203-UC01-T04_cache:clear（CLI ユーザー・実行時 pool の後始末）

1. CLI ユーザーで `bin/console cache:clear` を実行する
1. `[OK] Cache for the "prod" environment … was successfully cleared.` の後に「var/runtime/prod/pools を削除できないため, 実行時キャッシュに古い内容が残ります.」「Web サーバーのユーザーで bin/console cache:pool:clear --all … を実行するか, 管理画面のキャッシュ管理から削除してください.」の警告が表示される。終了コードは 0
1. `var/build/prod` が再生成されており、フロントは 200
1. Web サーバーのユーザーで `bin/console cache:pool:clear --all` を実行すると終了コードは 0 で `var/runtime/prod/pools` が空になる

## EX0203-UC01-T05_cache:clear --no-warmup（コンテナ消失の案内と復旧）

1. CLI ユーザーで `bin/console cache:clear --no-warmup` を実行する
1. 「var/build/prod にコンパイル済みコンテナがありません.」「ビルドディレクトリへ書き込めるユーザーで bin/console eccube:cache:build を実行してください. 実行するまで Web サーバーはアプリケーションを起動できず …」の警告が実行時 pool の警告より先に表示される。終了コードは 0
1. この状態でフロントへアクセスすると 500 になる
1. CLI ユーザーで `eccube:cache:build` を実行するとフロントが 200 に戻る

## EX0203-UC01-T06_cache:clear（Web サーバーのユーザー）

1. `-u www-data` で `bin/console cache:clear` を実行する
1. `Unable to write in the "…/var/cache/prod" directory.` で失敗し、終了コードは 1

## EX0203-UC01-T07_コンテンツ更新後のキャッシュ案内（終了コード 3）

1. 権限分離構成でフロントの該当ページを一度表示したうえで、`eccube:page:apply` によりページ本文を更新する
1. 本処理は完了し、次の 2 つの警告と終了コード 3 が返る。`var/build/prod/twig` に関する警告は出ない
   - 「…/var/runtime/prod/twig を削除できないため, 更新したテンプレートが反映されないことがあります.」と、管理画面のキャッシュ管理から削除するか Web サーバーのユーザーで `rm -rf …/var/runtime/prod/twig` を実行する旨の案内
   - 「…/var/runtime/prod/pools を削除できないため, 実行時キャッシュに古い内容が残ります.」「Web サーバーのユーザーで bin/console cache:pool:clear doctrine.app_cache_pool を実行するか, 管理画面のキャッシュ管理から削除してください.」
1. この時点でフロントの該当ページは更新前の内容のまま
1. `eccube:cache:build` を実行するとフロントに更新後の内容が反映される
1. `--no-cache-clear` を付けて apply すると警告は出ず終了コードは 0（キャッシュ削除は実行されない）

## EX0203-UC01-T08_管理画面のキャッシュ管理

1. 権限分離構成の管理画面で コンテンツ管理 → キャッシュ管理 → 「キャッシュ削除」を押下する
1. 「実行時キャッシュのみ削除しました。コンパイル済みコンテナとテンプレートを更新するには、CLI で bin/console eccube:cache:build を実行してください。」の警告が表示される
1. `var/runtime/prod/pools` と `var/runtime/prod/twig` が削除され、`var/build/prod` は残っている
1. 既定の（分離していない）構成で同じ操作をすると「削除しました」の成功メッセージが表示される

## EX0203-UC01-T09_ECCUBE_UMASK

1. CLI ユーザーで `var/build/prod` を削除し、`ECCUBE_UMASK` 未設定で `eccube:cache:build` を実行する
1. `var/build/prod` の `Eccube_KernelProdContainer.php` が 644、ディレクトリが 755 で作成される（OS の既定 umask 022 の場合）
1. 再度 `var/build/prod` を削除し、`ECCUBE_UMASK=0000` を環境変数に与えて `eccube:cache:build` を実行する
1. ファイルが 666、ディレクトリが 777 で作成される
1. `ECCUBE_UMASK=0000` の状態で `eccube:doctor:permissions` を実行すると、該当ディレクトリが「任意のローカルユーザーから書き込めます」の NG または「任意のローカルユーザーからも書き込めます」の WARN として検出される

## EX0204-UC01-T01_一時ファイルの掃除コマンド

1. TOPページ(会員Aでログイン状態)→対象受注の商品Pの[返品申請]で画像を 1 件添付し、[確認画面へ]ボタンを押下した後、申請せずにブラウザを閉じる
1. `html/upload/refund_request/temp/` 配下にハッシュ名のディレクトリとファイルが残っている
1. 以下のコマンドを実行する

    ```
    bin/console eccube:refund-request:cleanup-temp-files --hours=24
    ```

1. 「Deleted 0 expired refund request temp directories (older than 24 hours).」と表示され、ディレクトリは削除されない（24 時間を経過していないため）
1. `touch -t` 等で当該ディレクトリの更新日時を 25 時間以上前に変更し、同じコマンドを再実行する
1. 「Deleted 1 expired refund request temp directories (older than 24 hours).」と表示され、ディレクトリとファイルが削除される
1. `--hours` を省略して実行した場合も 24 時間が既定値として使われる

## EX0204-UC01-T02_一時ファイルの掃除コマンド（不正なオプション）

1. 以下のコマンドを実行する

    ```
    bin/console eccube:refund-request:cleanup-temp-files --hours=0
    ```

1. 「The "hours" option must be a positive integer.」のエラーが表示され、終了コードが 0 以外になる
