# realtime-comment-webapp 開発メモ

## 概要
半匿名でリアルタイムコメントを送れるWebアプリ

オンライン講義をコメントで盛り上げたい

text/markdown が送れるように -> サーバで入力内容の検証 -> フロントエンドでレンダリング

## 最低限の機能要件
1. セッション作成・閲覧
    - 利用者はセッション名を指定してセッションを作成できるようにします
    - サーバーはセッションごとにランダムな ID を発行します
    - 利用者はセッション URL から対象セッションを閲覧できるようにします
2. セッションの終了
    - セッション作成者はセッションを終了できます
    - 終了済みセッションは閲覧できるが、新規投稿はできなくなるようにします
3. 投稿本文の表示
    - 投稿本文はMarkdownとして表示します
    - 投稿本文の最大文字数はバックエンド設定ファイルで設定できるようにします
    - サーバーは文字数上限を超える投稿を拒否します
    - 投稿のレート制限は `user.id` 単位で適用し、しきい値はバックエンド設定ファイルで設定できるようにします
4. 最低限の入力検証
    - サーバーは必須項目、文字数上限、存在しないセッションへの投稿を検証します
    - エラー時は JSON 形式で理由を返すようにします
5. リアルタイム投稿
    - 利用者は開いているセッションへコメントを投稿できるようにします
    - 投稿は REST API で作成し、WebSocket で接続中の参加者へリアルタイム配信します
    - セッションを開いた利用者は、セッション開始後から現在までの投稿一覧を取得できるようにします
6. 半匿名の表示名
    - 利用者は表示名を指定できるようにます
    - 同じブラウザでは、指定した表示名を再利用できるようにします
    - 投稿には表示名と投稿日時を表示できるようにします

## アイデア
- 機能案
    - クライアント側でレンダリング
        - text/markdown
        - `||spoiler||` でスポイラー(クライアントがクリックすると表示)
        - `:short_code: ` で絵文字(クライアント側でレンダリング)
            - 絵文字データはAPIから取得
        - アイコン
            - 画像データはAPIから取得
            - `icon.id` で指定
    - バックエンド設定ファイル
        - yamlで記述
        - 文字数制限
        - カスタム絵文字登録
            - short_codeとimageへのリンク
    - 投稿はサーバー再起動後に消えるものとする
        - チャットの履歴をJSON形式でエクスポートできるようにしておく
    - delete機能
        - 削除は投稿の作成者・セッションの所有者が行える
    - edit機能
        - 編集は投稿の作成者のみが行える
    - ワードミュート機能
    - フロントエンドの設定をエクスポートする機能
        - JSONあたりで管理
        - ワードミュートやニックネームなど
    - ルーム or コメントセッション管理
        - ランダムなIDが割り当てられる

- APIエンドポイント `/api/v1`
    - `/sessions`
        - `GET`: セッション一覧を取得
        - `POST`: セッションを作成
    - `/sessions/{session_id}`
        - `GET`: セッション詳細を取得
        - `PATCH`: セッションを更新または終了
        - `DELETE`: セッションを削除
    - `/sessions/{session_id}/statuses`
        - `GET`: セッション内の投稿一覧を取得
        - `POST`: 投稿を作成
    - `/sessions/{session_id}/statuses/{status_id}`
        - `GET`: 投稿を取得
        - `PATCH`: 投稿本文を編集
        - `DELETE`: 投稿を削除
    - `/sessions/{session_id}/streaming`
        - `GET`: WebSocket 接続を開始
    - `/custom_emojis`
        - `GET`: カスタム絵文字を取得
        - カスタム絵文字の登録は設定ファイルにハードコードする
    - `/icons`
        - `GET`: アイコン一覧を取得
        - アイコンの登録は設定ファイルにハードコードする
    - `/health`
        - `GET`: ヘルスチェック用

## 実装しない/目指さないところ
- データの永続化
    - サーバ永続化データを管理したくない = DBを持って運用保守したくない
    - 再起動したらすべて初期化されるように
    - 厳密なユーザ・ロールの作成
        - 簡易的なユーザ管理にとどめておきたい
- 固定投稿
    - セッションに説明書いておけるようにしておけばいい
- 返信スレッド
    - リアルタイムチャットなのでエアリプしてもらえばいい
- ふぁぼ・絵文字リアクション
    - SNSなんだよね2
    - あくまでもリアルタイムチャットなので
- ユーザのブロック機能・通報機能
    - ここまでくるとSNSなんだよね
- 投稿検索機能
    - ブラウザの検索or`grep`で十分


## 技術スタック
- Backend: Go + Echo + WebSocket
- Frontend: TypeScript + React

## アセット
アイコン・カスタム絵文字をアセットとしてBackend側で配信します。
`PersistentVolume`に入れるかbackendのコンテナイメージに入れるかで迷いました。
差し替えの頻度を考えるとbackend側でコンテナイメージでもって置いて更新時にビルドし直せばいいと考えました。

## API

### 共通仕様

- API のベース URL は `/api/v1`。リクエストとレスポンスの本文は JSON を使います。
- サーバーが生成する `session_id`、`status_id`、`event_id` には UUIDv7 を使います。クライアントはこれらの ID を指定して作成しません。
- UUIDv7 は小文字の canonical 形式で扱います。UUIDv7 の時刻情報を利用して、投稿・イベントは概ね時系列に並べます。ただし、同一ミリ秒に発生した操作について、厳密な因果順序は保証しません。
- 日時は UTC の RFC 3339 形式とします。並び順は特記がない限り `created_at` の昇順とします。
- 読み取り系のレスポンスには、権限トークンを含めません。
- 更新 API には `PATCH` を使います。リクエストに含まれた可変フィールドだけを変更し、含まれないフィールドは保持します。完全なリソース表現で置き換える API が必要になった場合だけ `PUT` を使います。

#### 半匿名ユーザーと所有権

`user.id` はフロントエンドが初回利用時に生成してローカルに保持する UUID で、同じブラウザの投稿を見分けるための公開 ID とします。これは認証情報ではないため、編集・削除の権限判定には使いません。

投稿作成時は `user.id` をレート制限の集計キーとして使い、セッションをまたいで集計します。この ID はクライアントが再生成できる公開値なので、レート制限は通常利用時の公平性を保つためのものであり、意図的な濫用を防ぐ認証・認可の境界にはしません。

セッションまたは投稿を作成したときだけ、サーバーは対応する `owner_token` を返します。以降の変更系 API では `Authorization: Bearer <owner_token>` を送ります。トークンは以下のルールにします。

- サーバーでは平文で保存せず、ハッシュのみ保存します。
- 一覧取得・個別取得・WebSocket イベントには含めません。
- セッション所有者のトークンはセッションの更新・削除、およびセッション内の投稿削除に使えます。
- 投稿所有者のトークンは、その投稿の編集・削除にだけ使えます
- フロントエンド設定の JSON エクスポートには、`owner_token` を含めません。

公開するユーザー情報は次の形にします。`acct_color` は`#RRGGBB[AA]`値だけをサーバーが受け付けます。`icon.id` は `GET /api/v1/icons` で返す ID のいずれかを指定し、未登録の値は `404` で拒否します。

```json
{
    "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
    "username": "hik",
    "acct_color": "#e974a3",
    "icon": {
        "type": "preset",
        "id": 1
    }
}
```

#### エラー形式

エラーはすべて次の形で返します。

```json
{
    "error": {
        "code": "status_too_long",
        "message": "status exceeds the configured character limit"
    }
}
```

- 主な HTTP ステータス
    - `400` (リクエストボディの形式不正)
    - `401` (トークン不足または無効)
    - `403` (権限不足, 終了済みセッションへの投稿)
    - `404` (存在しない ID)
    - `422` (入力ポリシー違反, 本文サイズ超過など)
    - `429` (`user.id` ごとの投稿レート制限超過)

### セッション

セッションは `open` または `closed` の状態を持ちます。`closed` のセッションは閲覧できますが、新規投稿は受け付けません。

#### セッションを作成

`POST` `/api/v1/sessions`

- `description`は省略可能で、省略した場合デフォルトで空文字が入ります。
- `state` は省略可能で、省略した場合デフォルトで`open`が入ります。

```json
{
    "name": "sample_room",
    "description": "テスト用ルーム",
    "state": "open",
    "owner": {
        "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
        "username": "hik",
        "acct_color": "#e974a3",
        "icon": {
            "type": "preset",
            "id": 1
        }
    }
}
```

`201 Created` を返します。`owner_token` はこのレスポンスにだけ含めます。

```json
{
    "id": "019f747c-719a-7000-8000-000000000001",
    "name": "sample_room",
    "description": "テスト用ルーム",
    "state": "open",
    "created_at": "2019-12-08T03:48:33.901Z",
    "updated_at": "2019-12-08T03:48:33.901Z",
    "owner": {
        "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
        "username": "hik",
        "acct_color": "#e974a3",
        "icon": {
            "type": "preset",
            "id": 1
        }
    },
    "owner_token": "session-owner-token"
}
```

#### セッション一覧を取得

`GET` `/api/v1/sessions?state=open&limit=20&cursor={cursor}`

- `state` は省略可能で、`open` または `closed` を指定します。
- `limit` は 1 から 100、未指定時は 20 とします。
- `cursor` と `next_cursor` によりページングします。
    - クライアントは値を不透明な文字列として扱います。

```json
{
    "items": [
        {
            "id": "019f747c-719a-7000-8000-000000000001",
            "name": "sample_room",
            "description": "テスト用ルーム",
            "state": "open",
            "created_at": "2019-12-08T03:48:33.901Z",
            "updated_at": "2019-12-08T03:48:33.901Z",
            "owner": {
                "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
                "username": "hik",
                "acct_color": "#e974a3",
                "icon": {
                    "type": "preset",
                    "id": 1
                }
            }
        }
    ],
    "next_cursor": null
}
```

#### セッション詳細を取得

`GET` `/api/v1/sessions/{session_id}`

作成時のレスポンスと同じセッション情報を返します。`owner_token` は含めません。投稿一覧は含めず、次の投稿一覧 API で取得します。

#### セッションを更新・終了・削除

`PATCH` `/api/v1/sessions/{session_id}`

```http
Authorization: Bearer <session_owner_token>
```

```json
{
    "name": "sample_room",
    "description": "午前のテスト用ルーム",
    "state": "closed"
}
```

指定したフィールドだけを更新し、`200 OK` でセッション情報を返します。`state` を `closed` にする操作を、イベント終了時の標準的な終了操作とします。

`DELETE` `/api/v1/sessions/{session_id}`

同じ認証ヘッダーを必要とし、成功時は `200 OK` と削除したセッションの ID を返します。
実装では論理削除にして監査・障害復旧に備えます。(本当に消してもいいかもしれないです)

```json
{
    "id": "019f747c-719a-7000-8000-000000000001",
    "deleted_at": "2019-12-08T03:48:33.901Z"
}
```

### 投稿

投稿の取得・作成・更新・削除は、必ず対象セッションの配下に置きます。
これにより、異なるイベントの投稿を誤って操作できないようにします。

#### 投稿を作成

`POST` `/api/v1/sessions/{session_id}/statuses`

```json
{
    "status": "hi",
    "user": {
        "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
        "username": "hik",
        "acct_color": "#e974a3",
        "icon": {
            "type": "preset",
            "id": 1
        }
    }
}
```

`201 Created` を返します。作成者だけが受け取る `owner_token` を除き、以下が投稿の公開表現となります

```json
{
    "id": "019f747d-719a-74e0-888c-9f117d391252",
    "session_id": "019f747c-719a-7000-8000-000000000001",
    "created_at": "2019-12-08T03:48:33.901Z",
    "updated_at": null,
    "status": "hi",
    "user": {
        "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
        "username": "hik",
        "acct_color": "#e974a3",
        "icon": {
            "type": "preset",
            "id": 1
        }
    },
    "owner_token": "status-owner-token"
}
```

#### 投稿レート制限

- 投稿の作成にだけ適用し、編集・削除・取得・WebSocket 接続には適用しません。
- `user.id` ごとに、全セッションを通じて直近の `window` 内に作成できる投稿数を `max_posts_per_user` 以下に制限します。時間窓はスライディングウィンドウとします。
- 初期設定の `window: 1h` と `max_posts_per_user: 10000` は、任意の連続した 1 時間に 1 ユーザーが最大 10,000 件投稿できることを表します。
- 上限を超えた場合は `429 Too Many Requests` を返し、次に投稿できるまでの秒数を `Retry-After` ヘッダーに設定します。エラー本文の `error.code` は `status_rate_limited` とします。
- カウンタはプロセス内で保持し、サーバー再起動時にリセットします。

#### 投稿履歴・個別投稿を取得

`GET` `/api/v1/sessions/{session_id}/statuses?limit=50&before={status_id}`

- 投稿履歴は REST API で取得します。WebSocket は投稿履歴を再送せず、接続後に発生した差分イベントだけを配信します。
- `before` を省略した場合は最新の投稿から取得します。レスポンスの `items` 自体は画面に表示しやすいよう `status_id` の昇順にします。
- `before` を指定すると、その `status_id` より古い投稿を取得します。UUIDv7 の文字列順は概ね作成時刻順として扱います。
- `limit` は 1 から 100、未指定時は 50 とします。
- `next_before` は次に古い投稿を取得するためのカーソルです。これ以上古い投稿がなければ `null` を返します。

```json
{
    "items": [
        {
            "id": "019f747d-719a-74e0-888c-9f117d391252",
            "session_id": "019f747c-719a-7000-8000-000000000001",
            "created_at": "2019-12-08T03:48:33.901Z",
            "updated_at": null,
            "status": "hi",
            "user": {
                "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
                "username": "hik",
                "acct_color": "#e974a3",
                "icon": {
                    "type": "preset",
                    "id": 1
                }
            }
        }
    ],
    "next_before": null
}
```

`GET` `/api/v1/sessions/{session_id}/statuses/{status_id}`

投稿の公開表現を 1 件返します。`owner_token` は含めません。

#### 投稿を編集・削除

`PATCH` `/api/v1/sessions/{session_id}/statuses/{status_id}`

```http
Authorization: Bearer <status_owner_token>
```

```json
{
    "status": "Hello, World!"
}
```

成功時は `200 OK` で更新後の投稿を返します。投稿者情報は編集できず、`updated_at` を更新します。

`DELETE` `/api/v1/sessions/{session_id}/statuses/{status_id}`

投稿所有者またはセッション所有者のトークンを認証ヘッダーに指定します。成功時は `200 OK` と削除した投稿の ID を返します。
`session_id` はクライアントが複数セッションを扱うときに対象を一意に特定するために含めます。
削除イベントを配信できるよう、投稿 ID は保持します。

```json
{
    "id": "019f747d-719a-74e0-888c-9f117d391252",
    "session_id": "019f747c-719a-7000-8000-000000000001",
    "deleted_at": "2019-12-08T03:48:33.901Z"
}
```

### リアルタイム配信

`GET` `/api/v1/sessions/{session_id}/streaming`

WebSocketを用いて投稿内容のストリーミングを行います

WebSocket はサーバーからのイベント配信専用にします。
クライアントからの投稿操作は REST API で行います。

```json
{
    "event_id": "019f747d-719b-7000-8000-000000000001",
    "type": "status.created",
    "session_id": "019f747c-719a-7000-8000-000000000001",
    "occurred_at": "2019-12-08T03:48:33.901Z",
    "data": {
        "id": "019f747d-719a-74e0-888c-9f117d391252",
        "session_id": "019f747c-719a-7000-8000-000000000001",
        "created_at": "2019-12-08T03:48:33.901Z",
        "updated_at": null,
        "status": "hi",
        "user": {
            "id": "acf12938-d532-4e11-8fdb-fdf04e23d587",
            "username": "hik",
            "acct_color": "#e974a3",
            "icon": {
                "type": "preset",
                "id": 1
            }
        }
    }
}
```

イベント種別は `status.created`、`status.updated`、`status.deleted`、`session.updated`、`session.closed`、`session.deleted` とします。
`status.deleted` の `data` は少なくとも `id`、`session_id`、`deleted_at` を含めます。
`session.deleted` の `data` は少なくとも `id` と `deleted_at` を含めます。
いずれのイベントにも `owner_token` を含めません。

WebSocket は切断中のイベント配信を保証しません。
接続または再接続したクライアントは、次の手順で投稿履歴と差分イベントを同期します。

1. WebSocket に接続し、受信したイベントを一時的にバッファします。
2. 投稿履歴 API を呼び出し、レスポンスの `items` を初期表示します。
3. バッファしたイベントを `event_id` の UUIDv7 順で反映し、その後は受信時に反映します。
4. `event_id` と投稿 ID を用いて重複イベントを除去します。

同一ミリ秒に発生したイベントの厳密な因果順序は保証しないため、クライアントは削除を冪等に扱い、同じ投稿 ID の更新は `updated_at` が新しい状態を優先します。同期結果に矛盾を検出した場合は、投稿履歴 API を再取得して最新状態を正とします。

### アイコン

`GET` `/api/v1/icons`

利用可能なアイコン一覧を返します。アイコンは設定ファイルに静的に登録し、サーバー起動中は `id` を変更しません。`id` は公開ユーザー情報の `icon.id` と対応します。
アイコン数は少数の固定セットを想定するため、ページングは行いません。

`asset_path` はバックエンドのパスです。フロントエンドからは`baseurl`を別途指定して`baseurl`と組み合わせて使います。 `alt` は`name`がそれに該当します。

```json
{
    "items": [
        {
            "id": 1,
            "name": "default",
            "asset_path": "assets/icons/default.svg"
        }
    ]
}
```

### カスタム絵文字

`GET` `/api/v1/custom_emojis`

利用可能なカスタム絵文字一覧を返します。カスタム絵文字は設定ファイルに静的に登録します。
カスタム絵文字は少数の固定セットを想定するため、ページングは行いません。

`asset_path` はバックエンドのパスです。フロントエンドからはバックエンドの`baseurl`を別途指定して`baseurl`と組み合わせて使います。 `alt` は`short_code`がそれに該当します。

```json
{
    "items": [
        {
            "shortcode": "blank",
            "asset_path": "assets/emojis/blank.png"
        }
    ]
}
```

### ヘルスチェック

`GET` `/api/v1/health`

KubernetesのLivenessProve/ReadinessProveに利用します。

```json
{
    "status": "ok"
}
```

### 入力検証とレンダリング
- サーバーは本文の必須チェック、設定ファイルの文字数制限、リクエストサイズ制限、および `validation.rate_limits.status` による投稿レート制限を適用します。
- サーバーは Markdown の元テキストだけを保存し、HTML を保存・生成しません。
- フロントエンドは Markdown の生 HTML を無効化し、レンダリング結果をサニタイズします。リンク URL と画像 URL も許可するスキーム・ホストに限定します。
- `||spoiler||` と `:shortcode:` の変換はフロントエンドで行います。未登録の shortcode は通常のテキストとして表示します。
