---
title: "【FluxSocket 準備編③】認証エンドポイントと本番運用 — Private/Presence の署名・TLS・トラブル対処"
emoji: "🔑"
type: "tech"
topics: ["FluxSocket", "WebSocket", "リアルタイム通信", "認証"]
published: false
---

:::message
**FluxSocket 準備編（全3回・完）**

1. [アカウント作成から、最初の pub/sub まで](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-setup)
2. [チャンネル設計：Public / Private / Presence / Client Events](https://zenn.dev/brainy_software/articles/fluxsocket-channel-types)
3. **認証エンドポイントと本番運用** 👈 この記事
:::

[第2回](https://zenn.dev/brainy_software/articles/fluxsocket-channel-types)で、Private と Presence は「購読に認証が要る」と繰り返し出てきました。この最終回で、その**認証エンドポイント**を実装し、**本番接続**（`ws.fluxsocket.com`）と**つまずきどころの対処**までをまとめます。ここまで通せば準備は完全に整います。

---

## 1. なぜ認証が要るのか

Public Channel は誰でも購読できます。一方 `private-` / `presence-` チャンネルは、購読しようとした瞬間に **pusher-js が `authEndpoint` へ HTTP リクエスト**を送ります。サーバーは「このユーザーはこのチャンネルを購読してよいか」を判定し、**App Secret で署名**した文字列を返します。署名が正しければ購読が成立します。

```
ブラウザ                          あなたのサーバー              FluxSocket
  |                                  |                            |
  |-- subscribe('private-user-123')  |                            |
  |-- POST /auth/channel ----------->|                            |
  |     socket_id, channel_name      |  App Secret で署名          |
  |<-- { auth: "key:signature" } ----|                            |
  |-- 署名を提示して購読 ------------------------------------------>|  検証OK → 購読成立
```

ポイントは、**署名はサーバー側でしか作れない**こと（App Secret を使うため）。だからクライアントに Secret を置かず、サーバーの認証エンドポイントを経由します。

---

## 2. 認証エンドポイントを実装する

Pusher 互換なので、**Pusher 公式 SDK の認証ヘルパーがそのまま使えます**。手書きの HMAC は不要です。

### PHP（pusher-php-server）

```php
// POST /auth/channel
$channelName = $_POST['channel_name'];
$socketId    = $_POST['socket_id'];

if (str_starts_with($channelName, 'presence-')) {
    // Presence: ユーザー情報を添えて署名
    $userId   = $_POST['user_id'] ?? uniqid();
    $userInfo = ['name' => $_POST['user_name'] ?? 'Guest'];
    echo $pusher->authorizePresenceChannel($channelName, $socketId, $userId, $userInfo);
} else {
    // Private: チャンネル名 + socket_id を署名
    echo $pusher->authorizeChannel($channelName, $socketId);
}
```

（`$pusher` の生成は[第1回・手順7](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-setup)と同じです。）

### Node.js（pusher パッケージ）

```javascript
app.post('/auth/channel', (req, res) => {
    const { socket_id, channel_name } = req.body;

    if (channel_name.startsWith('presence-')) {
        const presenceData = {
            user_id: req.body.user_id || crypto.randomUUID(),
            user_info: { name: req.body.user_name || 'Guest' },
        };
        return res.send(pusher.authorizeChannel(socket_id, channel_name, presenceData));
    }
    res.send(pusher.authorizeChannel(socket_id, channel_name));
});
```

### クライアント側で認証情報を渡す

`authEndpoint` を指定し、`auth.params` でユーザー情報を送ります。

```javascript
const pusher = new Pusher('YOUR_APP_KEY', {
    wsHost: 'YOUR_FLUX_HOST',
    wsPort: 8080,
    forceTLS: false,
    enabledTransports: ['ws'],
    cluster: 'mt1',
    disableStats: true,
    authEndpoint: '/auth/channel',
    auth: { params: { user_id: 'user-123', user_name: '田中' } },
});
```

:::message
署名の中身（`socket_id:channel_name` の HMAC-SHA256）を自分で組み立てる例は、[オークション編](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-3)に載せています。仕組みを知りたいときはそちらを。通常は上記の SDK ヘルパーで十分です。
:::

---

## 3. 本番に接続する

ここまでは開発用に `localhost:8080`（TLS なし）で動かしてきました。本番は **`ws.fluxsocket.com` に 443 / TLS** で接続します。

### 環境変数（本番）

```bash
FLUX_APP_ID=your-app-id
FLUX_APP_KEY=your-app-key
FLUX_APP_SECRET=your-app-secret
FLUX_HOST=ws.fluxsocket.com
FLUX_PORT=443
FLUX_USE_TLS=true
```

### クライアント（本番）

```javascript
const pusher = new Pusher('YOUR_APP_KEY', {
    wsHost: 'ws.fluxsocket.com',
    wsPort: 443,
    wssPort: 443,
    forceTLS: true,          // ← 本番は必ず true
    enabledTransports: ['ws'],
    cluster: 'mt1',
    disableStats: true,
    authEndpoint: '/auth/channel',
});
```

開発と本番の差は **ホスト・ポート・TLS の 3 つだけ**です。ここを環境変数で切り替えられるようにしておけば、コードは共通で済みます。

:::message
接続先のホスト名はダッシュボードでも確認できます。本番で `forceTLS: true` にしたら、`wsPort` / `wssPort` は必ず `443` に。TLS 有効なのにポートが `8080` のまま、が最頻出のハマりどころです。
:::

---

## 4. つまずきどころと対処

| 症状 | よくある原因 | 対処 |
|------|------------|------|
| ずっと `connecting` のまま | `wsHost` 未指定で Pusher 本家に接続しにいっている / ポートと TLS の不一致 | `wsHost` を FluxSocket に向ける。本番は `443` + `forceTLS: true` |
| `cluster` 関連のエラー | Pusher 互換のための項目。FluxSocket では中身は任意 | `cluster: 'mt1'`（または空）を指定。`wsHost` を必ず併記 |
| 本番だけ繋がらない | ビルド後の JS が開発用ポート（`8080`）を見ている | 本番ビルドの `wsPort` / `wssPort` を `443` に |
| Private が 403 / 購読できない | `authEndpoint` 未指定 / Secret 誤りで署名不一致 / `socket_id`・`channel_name` を受け取れていない | 認証エンドポイントのリクエストボディと Secret を確認 |
| Presence でメンバー情報が出ない | 認証時に `channel_data`（`user_id` / `user_info`）を含めていない | `authorizePresenceChannel` を使う（手順2） |
| Client Events が飛ばない | ダッシュボードで未有効 / Public チャンネルで使用 | クライアントイベントを有効化。Private / Presence 上でのみ使用 |

デバッグの起点は 2 つです。**(1)** ブラウザの Network タブで `/auth/channel` のレスポンス（`auth` が返っているか）、**(2)** FluxSocket ダッシュボードの**デバッグコンソール**で接続とイベントの流れ。この 2 つを見れば、クライアント側かサーバー側かはすぐ切り分けられます。

---

## まとめ（準備編・完）

準備編の 3 回で、次のことを固めました。

- 第1回：アカウント・APIキー・環境変数・**最小の pub/sub 往復**
- 第2回：**Public / Private / Presence / Client Events** の選び方
- 第3回：**認証エンドポイント**・本番接続（`ws.fluxsocket.com:443`/TLS）・トラブル対処

ここから先は、[完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)の各チュートリアル（チャット・オークション・通知…）へ。どれも、この準備編を前提に「作りたい機能」に集中できます。

---

:::message
FluxSocket は現在ベータユーザーを募集しています。無料の Hobby プランで気軽にお試しいただけます。
[FluxSocket 公式サイト](https://fluxsocket.com)
:::

## 関連記事

- [FluxSocketで作る8つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [FluxSocket 準備編② チャンネル設計](https://zenn.dev/brainy_software/articles/fluxsocket-channel-types)
- [FluxSocketでリアルタイム化する — 入札が全員に即座に届く（認証エンドポイントの実装例）](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-3)
