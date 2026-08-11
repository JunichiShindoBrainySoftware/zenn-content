---
title: "PusherからFluxSocketへの移行ガイド"
emoji: "🔄"
type: "tech"
topics: ["FluxSocket", "pusher", "Laravel", "移行"]
published: false
---

FluxSocketはPusher Protocol v7に完全互換です。そのため、Pusherからの移行は**基本的に設定ファイルの変更だけ**で完了します。

この記事では、Laravel + Pusherの環境からFluxSocketへ移行する具体的な手順を解説します。

## 移行の前提

- Laravel 10以上でPusherを使用中
- `pusher/pusher-php-server` パッケージを使用中
- フロントエンドで `pusher-js` または `laravel-echo` を使用中

**これらのパッケージはそのまま使い続けます。** FluxSocket専用のSDKに入れ替える必要はありません。

## Step 1: FluxSocketのアカウント作成

[FluxSocket](https://fluxsocket.com) でアカウントを作成し、新しいアプリを作成します。

ダッシュボードから以下の情報を取得します：

- App ID
- Key
- Secret
- Host

## Step 2: 環境変数の変更

`.env` ファイルのPusher関連の設定を変更します。

**変更前（Pusher）:**
```env
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=123456
PUSHER_APP_KEY=abcdef1234567890
PUSHER_APP_SECRET=ghijkl1234567890
PUSHER_HOST=api-ap3.pusher.com
PUSHER_PORT=443
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=ap3
```

**変更後（FluxSocket）:**
```env
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=your-fluxsocket-app-id
PUSHER_APP_KEY=your-fluxsocket-key
PUSHER_APP_SECRET=your-fluxsocket-secret
PUSHER_HOST=your-fluxsocket-host
PUSHER_PORT=443
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=mt1
```

**`BROADCAST_DRIVER` は `pusher` のままです。** FluxSocketはPusher互換なので、Laravelから見るとPusherと同じように動きます。

## Step 3: config/broadcasting.php の確認

通常は `.env` の変更だけで動きますが、`host` の設定が直書きされていないか確認してください。

```php
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'host' => env('PUSHER_HOST'),
        'port' => env('PUSHER_PORT', 443),
        'scheme' => env('PUSHER_SCHEME', 'https'),
        'useTLS' => true,
    ],
],
```

`cluster` を指定している場合は、`host` に置き換えてください。FluxSocketは単一ホストで接続します。

## Step 4: フロントエンドの設定変更

### Laravel Echo を使っている場合

```javascript
// 変更前
window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    forceTLS: true,
});

// 変更後
window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    wsHost: import.meta.env.VITE_PUSHER_HOST,
    wsPort: 443,
    wssPort: 443,
    forceTLS: true,
    disableStats: true,
    enabledTransports: ['ws', 'wss'],
});
```

**変更点は3つだけ:**
1. `cluster` を削除
2. `wsHost` を追加
3. `disableStats` と `enabledTransports` を追加

### pusher-js を直接使っている場合

```javascript
// 変更前
const pusher = new Pusher('your-key', {
    cluster: 'ap3',
});

// 変更後
const pusher = new Pusher('your-key', {
    wsHost: 'your-fluxsocket-host',
    wsPort: 443,
    wssPort: 443,
    forceTLS: true,
    disableStats: true,
    enabledTransports: ['ws', 'wss'],
});
```

## Step 5: フロントエンドの環境変数

`.env` にフロントエンド用の値を追加：

```env
VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_HOST="${PUSHER_HOST}"
```

## Step 6: 動作確認

```bash
# キャッシュをクリア
php artisan config:clear
php artisan cache:clear

# サーバーを再起動
php artisan serve
```

FluxSocketのダッシュボードにある**デバッグコンソール**で、接続とイベントの送受信が正しく動作しているか確認できます。

## 移行チェックリスト

- [ ] FluxSocketのアカウント作成とアプリ作成
- [ ] `.env` のPusher設定を変更
- [ ] `config/broadcasting.php` の確認
- [ ] フロントエンドのEcho/Pusher設定を変更
- [ ] Publicチャンネルの動作確認
- [ ] Privateチャンネルの動作確認（使用している場合）
- [ ] Presenceチャンネルの動作確認（使用している場合）
- [ ] Webhookの設定移行（使用している場合）
- [ ] 本番環境への反映

## よくある質問

### Q: アプリケーションのコード（イベントやリスナー）は変更が必要？

**不要です。** `ShouldBroadcast` を実装したイベントクラス、`broadcastOn()` の定義、`routes/channels.php` の認証ロジック — これらは全てそのまま動きます。

### Q: pusher-php-server パッケージを入れ替える必要は？

**不要です。** `pusher/pusher-php-server` はそのまま使い続けます。FluxSocketがPusher互換のAPIを提供しているので、SDKから見ると普通のPusherサーバーと区別がつきません。

### Q: 移行中に既存のPusher環境と並行運用できる？

**可能です。** 環境変数で切り替えるだけなので、ステージング環境でFluxSocketを試しながら、本番はPusherのままにすることができます。

### Q: Pusherに戻したくなったら？

`.env` の設定を元に戻すだけです。アプリケーションのコードは何も変更していないので、リスクなく切り戻せます。

## まとめ

PusherからFluxSocketへの移行は、本質的には**環境変数とフロントエンド設定の変更だけ**です。アプリケーションのコードは一切触る必要がありません。

移行リスクが低いからこそ、まずはステージング環境で試してみてください。問題があればすぐにPusherに戻せます。

:::message
FluxSocketは現在ベータユーザーを募集しています。Pusherからの移行を検討中の方は、無料のHobbyプランでお試しいただけます。
👉 [FluxSocket公式サイト](https://fluxsocket.com)
:::

## 関連記事

- [FluxSocketで作る8つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [Pusherの料金問題と、国産WebSocket SaaSを作った理由](https://zenn.dev/brainy_software/articles/pusher-pricing-and-why-fluxsocket)
- [FluxSocketでリアルタイムチャットを実装する](https://zenn.dev/brainy_software/articles/fluxsocket-chat-tutorial)
- [WebSocket SaaSを選ぶときに見るべきポイント](https://zenn.dev/brainy_software/articles/websocket-saas-selection-guide)
