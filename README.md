# Contribution Graph Studio

GitHub GraphQL APIからコントリビューションを取得し、色・マスのサイズ・間隔を調整できるAstro製のアプリです。

```sh
npm install
npm run dev
```

画面にGitHubユーザー名とパーソナルアクセストークンを入力し、期間を選んで取得します。初期値は現在から過去1年で、年を指定することもできます。公開情報の取得には、公開リソースへの読み取りアクセスがあるトークンを利用できます。トークンはGitHub APIへのリクエストに直接使用し、アプリには保存しません。プライベートな活動は、トークンの権限とGitHubの公開設定に応じて表示されます。

```sh
npm run build
npm run preview
```

## 実装の構成

画面と処理は `src/pages/index.astro` にまとめています。Astroの静的ビルドでHTMLを生成し、グラフの取得や設定の変更はブラウザー側のスクリプトで行います。独自のバックエンドはありません。

### GitHubからデータを取得する

フォーム送信時にユーザー名、トークン、選択した期間を読み取り、GitHub GraphQL API (`https://api.github.com/graphql`) に `fetch` でPOSTします。トークンは `Authorization: Bearer ...` ヘッダーに設定します。

GraphQLの `user(login:)` から `contributionsCollection` を取得し、初期設定では送信時点から過去1年を、年を選んだ場合はその年の1月1日から12月31日までをUTCで指定します。レスポンスの `contributionCalendar` には、期間の合計件数と、週ごとにまとめられた日付・件数・活動レベルが含まれます。

取得中はボタンを無効にし、完了後に合計件数とユーザー名を表示します。通信エラー、APIエラー、ユーザーが見つからない場合は画面内にメッセージを表示します。トークンはフォームとリクエスト内で扱い、ローカルストレージなどには保存しません。

### グラフを描画する

`render()` はAPIから受け取った週ごとのデータを使って、月ラベル、曜日ラベル、日ごとのマスを作成します。週を縦の列として並べ、日曜日から土曜日までを上から配置しています。最初の週が日曜日から始まらない場合は、空のマスを挿入して曜日の位置を揃えます。

各マスには日付と件数をツールチップとアクセシブルなラベルとして設定します。GitHubが返す `contributionLevel` を0〜4の段階に変換し、色を決めています。初期表示では過去1年分の空のグラフを描画し、取得後に実際のデータで置き換えます。

### 色やサイズを変更する

色、マスの大きさ、間隔の入力が変わるたびに `applyStyle()` がCSSカスタムプロパティを更新します。テーマカラーは `--accent`、マスの大きさは `--cell`、間隔は `--gap` です。活動レベルごとの色は `color-mix()` でテーマカラーと白を混ぜて作ります。リセットボタンは各入力を初期値に戻します。

グラフが画面に収まらない場合は、グラフ部分を横スクロールできます。

## GraphQLクエリの解説

フォームの送信時に、次のクエリをGitHub GraphQL APIに送ります。

```graphql
query($login: String!, $from: DateTime!, $to: DateTime!) {
  user(login: $login) {
    login
    contributionsCollection(from: $from, to: $to) {
      contributionCalendar {
        totalContributions
        weeks {
          contributionDays {
            date
            contributionCount
            contributionLevel
          }
        }
      }
    }
  }
}
```

- `$login` は入力したユーザー名、`$from` と `$to` は選択した期間の開始・終了日時です。`!` は必須の変数を表します。値はクエリ文字列に埋め込まず、リクエストの `variables` として渡します。
- `user(login: $login)` で対象ユーザーを指定し、返された `login` を画面に表示します。ユーザーが存在しない場合、`user` は `null` になります。
- `contributionsCollection` は指定期間の活動を表し、その中の `contributionCalendar` から表示に必要な項目を取得します。
- `totalContributions` は期間の合計件数です。`weeks` 内の `contributionDays` は日ごとの `date`、`contributionCount`（件数）、`contributionLevel`（色分け用の段階）を含みます。

GraphQLでは必要なフィールドを指定できるため、このクエリはリポジトリ名や個々のコミット内容を取得しません。取得結果は画面の合計とマスの描画に使います。フィールドの詳細は [GitHubのGraphQLリファレンス](https://docs.github.com/ja/graphql/reference/users) を参照してください。

## セキュリティとトークンの取り扱い

### Fine-grained personal access tokens とは

Fine-grained personal access tokenは、個人がGitHub APIやGit over HTTPSを利用するときの認証情報です。アクセス対象の所有者やリポジトリを限定し、リポジトリ・アカウントなどに対する権限を細かく設定できます。作成時は必要な権限だけを選び、有効期限を設定してください。

このアプリでは、作成したトークンをアクセストークン欄に入力し、GitHub GraphQL APIの認証に使います。公開情報の取得には公開リソースへの読み取りアクセスを使えます。非公開の活動が含まれるかは、トークンの権限とGitHubの設定に依存します。組織のリソースにアクセスする場合、組織の承認が必要になることもあります。

[GitHubのトークン作成・管理ガイド](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)に手順と制限が記載されています。

- トークンはパスワードと同様に扱い、他人に渡したり、README・ソースコード・スクリーンショットに掲載したりしないでください。必要最小限の権限と適切な有効期限を設定してください。
- 入力されたトークンはブラウザーからHTTPSでGitHub APIに直接送信されます。このアプリは独自のサーバーに送信せず、ローカルストレージなどにも保存しません。ただし、入力欄には残り、ページ上のスクリプトやブラウザー拡張機能から読み取られる可能性があります。信頼できる環境で利用し、使用後は入力欄を消すかページを閉じてください。
- 公開情報の取得に必要な権限のみを使ってください。非公開の活動の表示はトークンの種類・権限とGitHub側の設定に依存します。GitHubのリファレンスでは、非公開・内部リポジトリの活動を含めるには `read:user` スコープが必要と説明されています。
- トークンの漏えいが疑われる場合は、GitHubの設定から直ちに無効化し、必要なら新しいトークンを発行してください。詳しくは [トークンの管理](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) と [トークンの失効](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/token-expiration-and-revocation) を参照してください。
