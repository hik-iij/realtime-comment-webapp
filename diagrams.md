# mermaid diagrams

## システムフロー

投稿の取得と作成・更新・削除には REST API を使います。
WebSocket は、接続後に発生した状態変更イベントを配信するためだけに使います。

```mermaid
flowchart LR
    User[User] --> Frontend["Frontend<br/>TypeScript + React"]

    Frontend -->|"GET / POST / PATCH / DELETE"| RestApi["REST API<br/>Go + Echo"]
    RestApi -->|"JSON response"| Frontend

    Frontend -->|"WebSocket connect"| Streaming["Streaming endpoint<br/>/sessions/{session_id}/streaming"]
    Streaming -->|"status.* and session.* events"| Frontend

    RestApi --> Service["Session and status service"]
    Service --> Store[("In-memory session and status store")]
    Store --> Service

    Service -->|"publish state changes"| Hub["WebSocket hub"]
    Streaming -->|"register and unregister client"| Hub
    Hub -->|"broadcast events"| Streaming

    RestApi -->|"read manifests and serve files"| Assets[("Icons and custom emoji assets")]
    Assets --> RestApi
```

- サーバーの再起動時にインメモリのセッション・投稿データは初期化されます
- アイコンとカスタム絵文字の一覧・画像はバックエンド側で管理し、フロントエンドは API の応答を使って表示します
- 投稿・セッション・イベントのサーバー生成 ID には UUIDv7 を使います

## コンポーネント図

フロントエンドは REST API と WebSocket のクライアントを分離し、バックエンドは HTTP の入出力、業務処理、状態保持、イベント配信、アセット配信を別コンポーネントとして扱います。

```mermaid
flowchart TB
    subgraph Browser[Browser]
        App["React application"]
        ApiClient["REST API client"]
        WsClient["WebSocket client"]
        LocalState["Client state<br/>profiles, statuses, buffered events"]
        Renderer["Markdown, spoiler, and emoji renderer"]

        App --> ApiClient
        App --> WsClient
        App --> LocalState
        App --> Renderer
        ApiClient --> LocalState
        WsClient --> LocalState
    end

    subgraph Backend[Go backend]
        Router["Echo router and middleware"]
        SessionHandler["Session handler"]
        StatusHandler["Status handler"]
        StreamingHandler["Streaming handler"]
        AssetHandler["Asset handler"]
        SessionService["Session service"]
        StatusService["Status service"]
        Validator["Input validator"]
        Store[("In-memory store")]
        Hub["WebSocket hub"]
        AssetRegistry["Asset registry"]

        Router --> SessionHandler
        Router --> StatusHandler
        Router --> StreamingHandler
        Router --> AssetHandler

        SessionHandler --> SessionService
        StatusHandler --> StatusService
        SessionHandler --> Validator
        StatusHandler --> Validator
        SessionService --> Store
        StatusService --> Store
        SessionService --> Hub
        StatusService --> Hub
        StreamingHandler --> Hub
        AssetHandler --> AssetRegistry
    end

    subgraph Assets[Backend managed assets]
        Manifest["icons.json and custom_emojis.json"]
        Files["Icon and emoji files"]
        Manifest --> AssetRegistry
        Files --> AssetRegistry
    end

    ApiClient -->|"REST JSON"| Router
    WsClient -->|"WebSocket"| StreamingHandler
    AssetRegistry -->|"asset metadata and files"| Renderer
```

- `StatusService` と `SessionService` は、状態を変更した後に WebSocket Hub へイベントを発行します。
- `Validator` は `config.yaml` の文字数制限と、許可されたアイコン ID を検証します。
- `AssetRegistry` は JSON マニフェストを読み込み、アイコン・カスタム絵文字の一覧 API と静的アセット配信に利用します。
- `Client state` は REST の投稿履歴と WebSocket の差分イベントを統合し、再接続時には履歴 API の結果を正とします。

## 初回同期と投稿配信

クライアントは WebSocket を先に接続してイベントをバッファし、その後に REST API で投稿履歴を取得します。
これにより、履歴取得中に届いたイベントを取りこぼさないようにします。

```mermaid
sequenceDiagram
    autonumber
    participant Client as client
    participant Streaming as Streaming endpoint
    participant Hub as WebSocket hub
    participant API as REST API
    participant Store as Session and status store
    participant Other as Another client

    Client->>Streaming: GET /sessions/{session_id}/streaming
    Streaming->>Hub: Register client
    Hub-->>Streaming: Registration complete
    Streaming-->>Client: WebSocket connected
    Note over Client: Buffer received events

    Client->>API: GET /sessions/{session_id}/statuses?limit=50
    API->>Store: Read current status history
    Store-->>API: items and next_before
    API-->>Client: History response
    Client->>Client: Render history items
    Client->>Client: Sort buffered events by UUIDv7 event_id and apply them

    Other->>API: POST /sessions/{session_id}/statuses
    API->>Store: Validate input and create a UUIDv7 status_id
    Store-->>API: Created status
    API->>Hub: Publish status.created
    Hub-->>Streaming: Broadcast event
    Streaming-->>Client: status.created
    Client->>Client: De-duplicate and insert the status

    Note over Client,Hub: On reconnect, connect, buffer events, and fetch history again
```

- `GET /sessions/{session_id}/statuses` は投稿履歴を返します。`before` と `next_before` を使い、古い履歴を追加で取得します
- WebSocket は履歴を再送せず、`status.created`、`status.updated`、`status.deleted` などの差分イベントだけを配信します
- UUIDv7 の順序は概ね時系列ではありますが、同一ミリ秒に発生した操作の厳密な因果順序は保証しないものとします
- 削除は冪等に扱います
- 同じ投稿 ID の更新は `updated_at` が新しい状態を優先します
- 同期結果に矛盾を検出した場合、クライアントは投稿履歴 API を再取得して最新状態へ戻します
