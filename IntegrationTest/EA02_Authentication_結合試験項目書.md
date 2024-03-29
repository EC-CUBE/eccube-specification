# EA02_Authentication_結合試験項目書

## EA0201-UC01-T01_パスワード認証機能

1. ログインID/パスワード入力ページ＞管理画面TOPに遷移
1. 管理者登録したユーザのログインID/パスワードを入力して、ログインボタンを押下する
1. 管理画面TOPに遷移する
1. ログアウトをクリックすると管理画面のログイン画面に遷移する
1. 間違ったID/PASSを入力すると、「ログインできませんでした。入力内容に誤りがないかご確認ください。」と文言が表示される
1. ログイン後、画面右上の管理者名から最終ログイン時間が正常であることを確認

## EA0201-UC01-T02_パスワード認証機能(パスワード変更)

1. 設定→システム設定→メンバー管理
1. メンバー編集画面を表示する
1. パスワードを変更し、登録する
1. ログアウトする
1. 変更前のパスワードを入力し、「ログインできませんでした。 入力内容に誤りがないかご確認ください。」のエラーが表示される
1. 変更後のパスワードを入力し、ログインできることを確認する

## EA0201-UC01-T03_パスワード認証機能(パスワード変更：画面右上)

1. ユーザアイコン＞パスワード変更
1. パスワード変更画面を表示する
1. パスワードを変更し、登録する
1. ログアウトする
1. 変更前のパスワードを入力し、「ログインできませんでした。 入力内容に誤りがないかご確認ください。」のエラーが表示される
1. 変更後のパスワードを入力し、ログインできることを確認する

## EA0201-UC01-T04_パスワード認証機能(非稼働)

1. 設定→システム設定→メンバー管理
1. メンバー編集画面を表示する
1. 非稼働の状態に変更し、登録する
1. ログアウトする
1. 非稼働にしたメンバーでログインすると、「ログインできませんでした。 入力内容に誤りがないかご確認ください。」のエラーが表示される

## EA0201-UC01-T05_パスワード認証機能(メンバー削除)

1. 設定→システム設定→メンバー管理
1. メンバーを削除する
1. ログアウトする
1. 削除したメンバーでログインすると、「ログインできませんでした。 入力内容に誤りがないかご確認ください。」のエラーが表示される

## EA0201-UC02-T01_2段階認証(登録)

1. 設定→システム設定→メンバー管理
1. 「新規登録」ボタンを押下する
1. 2段階認証を有効化し、登録する
1. 一覧画面を確認し、2段階認証が有効かつ設定未完了の状態であることを確認する
1. 作成したメンバーでログインする
1. 2段階認証の登録画面が表示される
1. 適当な値を入力する
1. 「トークンに誤りがあります。」のエラーが表示される
1. Google Authenticatorアプリ等でQRコードをスキャンし、数値を入力する
1. ログインできることを確認する
1. 一覧画面を確認し、2段階認証が有効かつ設定完了の状態であることを確認する

## EA0201-UC02-T02_2段階認証(ログイン)

1. EA0201-UC02-T01に続き作業する
1. 管理画面からログアウトし、cookie「eccube2fa」を削除する
1. ログイン画面を表示する
1. id/passwordを入力し、ログインする
1. 2段階認証の入力画面が表示される
1. 適当な値を入力する
1. 「トークンに誤りがあります。再度入力してください。」のエラーが表示される
1. Google Authenticatorアプリ等でトークンを確認し、数値を入力する
1. ログインできることを確認する

## EA0201-UC02-T03_2段階認証(2段階認証の解除)

1. EA0201-UC02-T02に続き作業する
1. メンバー編集画面から、2段階認証を「無効」に変更し、登録する
1. 管理画面からログアウトする
1. ログイン画面を表示する
1. id/passwordを入力し、ログインする
1. ログインできることを確認する
