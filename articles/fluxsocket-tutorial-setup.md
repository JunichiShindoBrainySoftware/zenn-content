---
title: "【FluxSocket入門】チュートリアルを始める前の準備 — アカウント作成からAPIキー取得まで"
emoji: "🔧"
type: "tech"
topics: ["FluxSocket", "WebSocket", "リアルタイム通信", "チュートリアル"]
published: false
---

## このシリーズについて

本シリーズでは、**FluxSocket** を使って 8 つのリアルタイムアプリを構築していきます。チャット、通知、オークション、ダッシュボードなど、実務でよく求められるリアルタイム機能を手を動かしながら学べる構成です。

この **第 0 回（準備編）** では、すべてのチュートリアルに共通する FluxSocket のセットアップ手順を解説します。アカウント作成、アプリの作成、API キーの取得、そしてクライアント・サーバー双方の基本的な接続方法までをカバーします。

**所要時間: 約 5 分**

---

## 1. FluxSocket とは

FluxSocket は、日本発の **Pusher 互換 WebSocket SaaS** です。Pusher のクライアントライブラリやサーバー SDK がそのまま使えるため、既存の知識やコードを活かしつつ、日本語ドキュメント・日本円決済という利点を得られます。

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

:::message
登録後、確認メールが届きます。メール内のリンクをクリックしてアカウントを有効化してください。
:::

![登録画面](/images/tutorial/tutorial-register.png)

---

## 3. アプリを作成する

メール確認が完了したら、ダッシュボードにログインします。

**「新規アプリ作成」** ボタンをクリックし、以下を設定します。

| 項目 | 必須 | 説明 |
|------|------|------|
| アプリ名 | はい | 識別しやすい名前を付けます（例: `my-chat-app`） |
| 環境 | はい | 「本番」または「開発テスト」を選択します。チュートリアルでは「開発テスト」を推奨します |
| クライアントイベント | いいえ | クライアントから直接イベントを送信する場合に有効化します（後から変更可能） |

![アプリ作成](/images/tutorial/tutorial-app-create.png)

---

## 4. API キーを確認する

アプリを作成すると、アプリ詳細画面に以下の 3 つの認証情報が表示されます。

| キー | 用途 | 公開可否 |
|------|------|---------|
| **App ID** | アプリの識別子 | サーバー側で使用 |
| **App Key** | クライアント側の接続に使用 | 公開情報（HTML/JS に埋め込み可） |
| **App Secret** | サーバー側の認証に使用 | **絶対に公開しないでください** |

ダッシュボードには `.env` 形式でコピーできるスニペットが用意されています。ボタン一つでクリップボードにコピーできるので活用してください。

![アプリ詳細](/images/tutorial/tutorial-app-detail.png)

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
FLUX_HOST=localhost
FLUX_PORT=8080
```

各変数の説明:

| 変数名 | 説明 |
|--------|------|
| `FLUX_APP_ID` | ダッシュボードで確認した App ID |
| `FLUX_APP_KEY` | ダッシュボードで確認した App Key |
| `FLUX_APP_SECRET` | ダッシュボードで確認した App Secret |
| `FLUX_HOST` | FluxSocket サーバーのホスト名。本番環境では FluxSocket から提供されるホスト名を指定します |
| `FLUX_PORT` | 接続ポート番号。TLS 使用時は `443` を指定します |

:::message
各チュートリアルで `.env.example` を用意しています。`cp .env.example .env` でコピーしてから値を書き換えてください。
:::

---

## 6. クライアント側の接続（Pusher.js）

FluxSocket は Pusher 互換なので、公式の **Pusher.js** クライアントライブラリがそのまま使えます。

### CDN から読み込む

```html
<script src="https://js.pusher.com/8.2.0/pusher.min.js"></script>
```

npm を使う場合は `npm install pusher-js` でもインストールできます。

### 初期化と接続

```javascript
const pusher = new Pusher('YOUR_APP_KEY', {
    wsHost: 'your-fluxsocket-host',
    wsPort: 443,
    forceTLS: true,
    enabledTransports: ['ws'],
    cluster: 'mt1',
    disableStats: true
});

// チャンネルを購読してイベントを受信する
const channel = pusher.subscribe('my-channel');
channel.bind('my-event', (data) => {
    console.log('受信:', data);
});
```

### 設定項目の説明

| パラメータ | 説明 |
|-----------|------|
| `wsHost` | FluxSocket サーバーのホスト名を指定します |
| `wsPort` | WebSocket の接続ポート。TLS 使用時は `443` |
| `forceTLS` | `true` にすると TLS（wss://）で接続します。本番環境では必ず `true` にしてください |
| `enabledTransports` | `['ws']` を指定して WebSocket のみを使用します（HTTP フォールバックは不要） |
| `cluster` | Pusher 互換のため必要ですが、FluxSocket では任意の値で構いません。`'mt1'` を指定してください |
| `disableStats` | `true` にして Pusher の統計送信を無効化します |

---

## 7. サーバー側からのイベント送信

サーバーからチャンネルにイベントを送信（トリガー）するには、サーバー SDK を使用します。

### PHP（FluxSocket PHP SDK）

```bash
composer require fluxsocket/fluxsocket-php
```

```php
$client = new \FluxSocket\Client([
    'app_id' => getenv('FLUX_APP_ID'),
    'key'    => getenv('FLUX_APP_KEY'),
    'secret' => getenv('FLUX_APP_SECRET'),
    'host'   => getenv('FLUX_HOST'),
    'port'   => (int) getenv('FLUX_PORT'),
    'use_tls' => false,
]);

$client->trigger('my-channel', 'my-event', [
    'message' => 'Hello World',
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
    port: process.env.FLUX_PORT,
    useTLS: false,
});

pusher.trigger('my-channel', 'my-event', {
    message: 'Hello World',
});
```

Pusher 互換なので、**既存の Pusher サーバー SDK がそのまま動作します**。接続先のホスト・ポートを FluxSocket に向けるだけで切り替えが完了します。

---

## 8. チャンネルの種類

FluxSocket（Pusher 互換）では、用途に応じて 3 種類のチャンネルとクライアントイベントが用意されています。

### Public Channel

プレフィックスなしのチャンネルです。認証不要で、誰でも購読できます。

```javascript
const channel = pusher.subscribe('news');
```

チャットルームの公開メッセージや、全体向けの通知などに適しています。

### Private Channel

`private-` プレフィックスを付けたチャンネルです。購読時にサーバー側で認証が行われ、許可されたユーザーのみが購読できます。

```javascript
const channel = pusher.subscribe('private-user-123');
```

ユーザー個別の通知や、アクセス制限のあるデータ配信に使います。

### Presence Channel

`presence-` プレフィックスを付けたチャンネルです。Private Channel の機能に加えて、**誰がオンラインか** をリアルタイムに共有できます。

```javascript
const channel = pusher.subscribe('presence-room-1');
channel.bind('pusher:member_added', (member) => {
    console.log(`${member.info.name} が参加しました`);
});
```

チャットルームの在室表示や、共同編集のカーソル表示などに最適です。

### Client Events

`client-` プレフィックスを付けたイベントは、サーバーを経由せずにクライアント間で直接送受信できます。Private Channel または Presence Channel 上でのみ使用可能です。

```javascript
const channel = pusher.subscribe('private-room-1');
channel.trigger('client-typing', { user: 'Taro' });
```

タイピングインジケーターやカーソル位置の共有など、低遅延が求められる用途に向いています。

:::message
Client Events を使用するには、ダッシュボードのアプリ設定で「クライアントイベント」を有効にする必要があります。
:::

---

## 9. 準備完了！次のステップ

ここまでで、FluxSocket を使ったリアルタイムアプリ開発の準備が整いました。

実際に動くデモを確認したい場合は、デモサイトをご覧ください。

**デモサイト:** [https://demo.fluxsocket.com](https://demo.fluxsocket.com)

### チュートリアル一覧

以下のチュートリアルを順次公開予定です。興味のあるものから取り組んでいただけます。

| # | タイトル | 技術スタック |
|---|---------|-------------|
| 1 | Laravel でリアルタイムチャットを作る | Laravel + Blade |
| 2 | React でリアルタイムオークションを作る | React + Vite |
| 3 | Node.js でユーザー別通知を実装する | Express + Vanilla JS |
| 4 | Vue.js でライブダッシュボードを作る | Vue 3 + Chart.js |
| 5 | Next.js で共同編集ドキュメントを作る | Next.js + Tiptap |
| 6 | Vanilla JS でタイピングインジケーターを作る | HTML + JS |
| 7 | Laravel + React でプレゼンスチャットを作る | Laravel + React |
| 8 | Node.js でリアルタイム位置情報共有を作る | Express + Leaflet |

各チュートリアルは **この準備編が完了していること** を前提としています。

---

FluxSocket は現在 **ベータユーザーを募集中** です。Hobby プランは永久無料で、個人開発や学習用途に最適です。ぜひ [fluxsocket.com](https://fluxsocket.com) でアカウントを作成して、リアルタイム通信の世界を体験してみてください。
