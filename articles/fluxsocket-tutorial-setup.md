---
title: "【FluxSocket 準備編①】アカウント作成から、最初の pub/sub まで"
emoji: "🔧"
type: "tech"
topics: ["FluxSocket", "WebSocket", "リアルタイム通信", "チュートリアル"]
published: false
---

:::message
**FluxSocket 準備編（全3回）**

1. **アカウント作成から、最初の pub/sub まで** 👈 この記事
2. チャンネル設計：Public / Private / Presence / Client Events（近日公開）
3. 認証と本番運用：認証エンドポイント・TLS・トラブルシュート（近日公開）

各回は独立して読めます。まず動かしたいなら本記事だけで「送って受け取る」ところまで到達できます。
:::

## このシリーズについて

**FluxSocket** で作る各種チュートリアル（[完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index) を参照）は、共通の「接続の基礎」を前提にしています。この準備編（全3回）で、その基礎を一度だけ固めておきます。

第1回のゴールはシンプルです — **アカウントを作り、サーバーから送ったイベントをブラウザで受け取る**。この 1 往復さえ通れば、あとはチャンネルの種類（第2回）や認証・本番運用（第3回）を足していくだけです。

**所要時間: 約 5 分**

---

## 1. FluxSocket とは

FluxSocket は、日本発の **Pusher 互換 WebSocket SaaS** です。Pusher のクライアントライブラリ（`pusher-js`）やサーバー SDK（`pusher-php-server` など）がそのまま使えるため、既存の知識やコードを活かしつつ、日本語ドキュメント・日本円決済という利点を得られます。

公式サイト: [https://fluxsocket.com](https://fluxsocket.com)

---

## 2. アカウントを作成する

[FluxSocket 登録ページ](https://fluxsocket.com/register) にアクセスし、以下の情報を入力します。

| 項目 | 説明 |
|------|------|
| お名前 / チーム名 | 個人名または組織名 |
| メールアドレス | ログインに使用します |
| パスワード | 8 文字以上を推奨 |

登録すると **Hobby プラン（永久無料）** が自動的に適用されます。チュートリアルを進めるにはこのプランで十分です。

> 登録後、確認メールが届きます。メール内のリンクをクリックしてアカウントを有効化してください。

---

## 3. アプリを作成する

メール確認が完了したら、ダッシュボードにログインします。

**「新規アプリ作成」** ボタンをクリックし、以下を設定します。

| 項目 | 必須 | 説明 |
|------|------|------|
| アプリ名 | はい | 識別しやすい名前を付けます（例: `my-first-app`） |
| 環境 | はい | 「本番環境」または「開発・テスト」を選択します。チュートリアルでは、デバッグ機能が有効になる「開発・テスト」を推奨します |
| クライアントイベント | いいえ | クライアントから直接イベントを送信する場合に有効化します（後から変更可能。第2回で扱います） |

---

## 4. API キーを確認する

アプリを作成すると、アプリ詳細画面に以下の 3 つの認証情報が表示されます。

| キー | 用途 | 公開可否 |
|------|------|---------|
| **App ID** | アプリの識別子 | サーバー側で使用 |
| **App Key** | クライアント側の接続に使用 | 公開情報（HTML/JS に埋め込み可） |
| **App Secret** | サーバー側の認証に使用 | **絶対に公開しないでください** |

:::message alert
**App Secret** は秘密鍵です。Git リポジトリにコミットしたり、クライアント側のコードに含めたりしないでください。必ず環境変数で管理しましょう。
:::

---

## 5. 環境変数を設定する

プロジェクトのルートに `.env` ファイルを作成し、以下の変数を設定します。

```bash
FLUX_APP_ID=your-app-id
FLUX_APP_KEY=your-app-key
FLUX_APP_SECRET=your-app-secret
FLUX_HOST=ws.fluxsocket.com
FLUX_PORT=443
FLUX_USE_TLS=true
```

| 変数名 | 説明 |
|--------|------|
| `FLUX_APP_ID` | ダッシュボードで確認した App ID |
| `FLUX_APP_KEY` | ダッシュボードで確認した App Key |
| `FLUX_APP_SECRET` | ダッシュボードで確認した App Secret |
| `FLUX_HOST` | FluxSocket の WebSocket ホスト名。`ws.fluxsocket.com` です |
| `FLUX_PORT` | 接続ポート番号。`443` です |
| `FLUX_USE_TLS` | TLS（wss://）で接続するか。`true` です |

:::message
**接続先は、手順 3 で選んだ環境（本番 / 開発・テスト）にかかわらず同じ** `ws.fluxsocket.com:443`（TLS）です。環境の選択はデバッグ機能の有無を切り替えるもので、接続先は変わりません。ダッシュボードのアプリ詳細画面にも同じ値が表示されます。
:::

:::message
ダッシュボードのアプリ詳細画面には、`FLUXSOCKET_APP_ID` のように **`FLUXSOCKET_` 始まり**のスニペットが表示されます。本シリーズでは短い `FLUX_` を使っていますが、指すものは同じです。画面からコピーした場合は変数名を読み替えてください（`FLUXSOCKET_SCHEME=https` が `FLUX_USE_TLS=true` に対応します）。
:::

---

## 6. クライアント側で受信する（Pusher.js）

FluxSocket は Pusher 互換なので、公式の **Pusher.js** クライアントライブラリがそのまま使えます。

### CDN から読み込む

```html
<script src="https://js.pusher.com/8.2.0/pusher.min.js"></script>
```

npm を使う場合は `npm install pusher-js` でもインストールできます。

### 初期化して購読する

```javascript
const pusher = new Pusher('YOUR_APP_KEY', {
    wsHost: 'ws.fluxsocket.com',
    wsPort: 443,
    wssPort: 443,
    forceTLS: true,
    enabledTransports: ['ws'],
    cluster: 'mt1',         // FluxSocket では任意。'mt1' を指定
    disableStats: true
});

// チャンネルを購読して、イベントを待ち受ける
const channel = pusher.subscribe('hello');
channel.bind('greeting', (data) => {
    console.log('受信:', data.message);
});
```

| パラメータ | 説明 |
|-----------|------|
| `wsHost` | FluxSocket の WebSocket ホスト名（`ws.fluxsocket.com`） |
| `wsPort` | 非 TLS 時の接続ポート |
| `wssPort` | **TLS 時の接続ポート。`forceTLS: true` ではこちらが使われます**（指定漏れに注意） |
| `forceTLS` | `true` で TLS（wss://）接続。FluxSocket は TLS のみです |
| `enabledTransports` | `['ws']` を指定して WebSocket のみを使用（HTTP フォールバック不要） |
| `cluster` | Pusher 互換のため必要ですが、FluxSocket では任意の値で構いません（`'mt1'`） |
| `disableStats` | `true` にして Pusher の統計送信を無効化します |

これで、`hello` チャンネルの `greeting` イベントを待ち受ける状態になりました。あとはサーバーから送るだけです。

---

## 7. サーバー側からイベントを送る（trigger）

FluxSocket は Pusher 互換の HTTP API を持っているので、Pusher のサーバー SDK がそのまま使えます。

### PHP（pusher-php-server をそのまま使う）

```bash
composer require pusher/pusher-php-server
```

```php
$pusher = new Pusher\Pusher(
    getenv('FLUX_APP_KEY'),
    getenv('FLUX_APP_SECRET'),
    getenv('FLUX_APP_ID'),
    [
        'host'   => getenv('FLUX_HOST'),
        'port'   => (int) getenv('FLUX_PORT'),
        'useTLS' => getenv('FLUX_USE_TLS') === 'true',
    ]
);

$pusher->trigger('hello', 'greeting', [
    'message' => 'Hello FluxSocket!',
]);
```

### Node.js（pusher パッケージ）

```bash
npm install pusher
```

```javascript
const Pusher = require('pusher');

const pusher = new Pusher({
    appId: process.env.FLUX_APP_ID,
    key: process.env.FLUX_APP_KEY,
    secret: process.env.FLUX_APP_SECRET,
    host: process.env.FLUX_HOST,
    port: parseInt(process.env.FLUX_PORT, 10),
    useTLS: process.env.FLUX_USE_TLS === 'true',
});

pusher.trigger('hello', 'greeting', {
    message: 'Hello FluxSocket!',
});
```

Pusher 互換なので、接続先のホスト・ポートを FluxSocket に向けるだけで動きます。

:::message
FluxSocket には純正のネイティブSDK（PHP は `flux-socket/php-sdk`、JS は `flux-socket-js` など）もあります。本シリーズでは、既存の Pusher 資産や知識をそのまま活かせる**互換ライブラリ（pusher-js / pusher-php-server）**を使う方針で進めます。ゼロから作る場合は純正SDKも選べます。
:::

---

## 8. 1 往復を通す（動作確認）

1. 手順 6 の HTML/JS を開いたブラウザのコンソールを表示しておく
2. 手順 7 のサーバーコードを実行する
3. ブラウザのコンソールに `受信: Hello FluxSocket!` が出れば成功

```
サーバー（trigger）              FluxSocket                ブラウザ（subscribe）
    |                              |                              |
    |-- HTTP: greeting ----------->|                              |
    |    channel: "hello"          |-- WebSocket: greeting ------>|
    |                              |                              |  console.log('受信: ...')
```

サーバーは「イベントを送る」だけ、ブラウザは「待ち受ける」だけ。WebSocket の接続管理は FluxSocket が担います。**この 1 往復が、すべてのリアルタイム機能の最小単位**です。

---

## 次のステップ

準備は整いました。ここから先は 2 つの方向に広がります。

- **第2回：チャンネル設計** — いま使った `hello` は誰でも購読できる **Public Channel** でした。ユーザー個別に届ける **Private**、在室者が分かる **Presence**、クライアント同士で直接やり取りする **Client Events** を、実例つきで扱います。
- **第3回：認証と本番運用** — Private / Presence に必要な**認証エンドポイント**の実装、TLS まわりの詳細、つまずきやすいトラブルの対処。

各チュートリアル（[完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)）は、**この準備編が完了していること**を前提としています。

---

ここまで通せた方へ ── FluxSocket は現在 **ベータユーザーを募集中** です。実際に動かしてみて詰まった点や、欲しい機能があれば、X（[@brainysoftware](https://x.com/brainysoftware)）で教えてください。開発に直接反映します。

Hobby プランは永久無料なので、このまま試し続けていただけます。

## 関連記事

- [FluxSocketで作る8つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [WebSocketとは？HTTP通信との違いと使いどころをわかりやすく解説](https://zenn.dev/brainy_software/articles/websocket-vs-http-realtime)
