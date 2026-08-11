---
title: "FluxSocketでリアルタイムチャットを実装する"
emoji: "💬"
type: "tech"
topics: ["FluxSocket", "Laravel", "websocket", "リアルタイム通信"]
published: false
---

この記事では、FluxSocketとLaravelを使って、リアルタイムチャット機能を最短で構築する方法を解説します。Pusher互換APIなので、LaravelのBroadcasting機能がそのまま使えます。

## 前提

- Laravel 10以上
- Composerが使える環境
- FluxSocketのアカウント（[無料のHobbyプランで登録](https://fluxsocket.com)）

## Step 1: FluxSocketでアプリを作成

FluxSocketにログインし、ダッシュボードから新しいアプリを作成します。

作成すると以下の情報が発行されます：

- **App ID**
- **Key**
- **Secret**
- **Host**（FluxSocketのWebSocketエンドポイント）

## Step 2: Laravel側の設定

### パッケージのインストール

```bash
composer require pusher/pusher-php-server
npm install pusher-js laravel-echo
```

FluxSocketはPusher互換なので、Pusherの公式SDKがそのまま使えます。

### 環境変数の設定

`.env` ファイルを編集します：

```env
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=your-app-id
PUSHER_APP_KEY=your-app-key
PUSHER_APP_SECRET=your-app-secret
PUSHER_HOST=your-fluxsocket-host
PUSHER_PORT=443
PUSHER_SCHEME=https
```

### config/broadcasting.php の確認

Laravelのデフォルト設定で動きますが、`host` の設定を追加します：

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

## Step 3: イベントの作成

チャットメッセージを送信するイベントを作成します。

```bash
php artisan make:event ChatMessageSent
```

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class ChatMessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public string $username,
        public string $message,
    ) {}

    public function broadcastOn(): Channel
    {
        return new Channel('chat');
    }

    public function broadcastAs(): string
    {
        return 'message.sent';
    }
}
```

## Step 4: メッセージ送信のエンドポイント

```php
// routes/api.php
use App\Events\ChatMessageSent;

Route::post('/chat/send', function (Request $request) {
    $request->validate([
        'username' => 'required|string|max:50',
        'message' => 'required|string|max:500',
    ]);

    event(new ChatMessageSent(
        username: $request->username,
        message: $request->message,
    ));

    return response()->json(['status' => 'sent']);
});
```

## Step 5: フロントエンドでメッセージを受信

### Laravel Echo の設定

```javascript
// resources/js/bootstrap.js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

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

### チャットUIの実装

```javascript
// チャンネルを購読してメッセージを受信
window.Echo.channel('chat')
    .listen('.message.sent', (event) => {
        appendMessage(event.username, event.message);
    });

function appendMessage(username, message) {
    const chatArea = document.getElementById('chat-messages');
    const div = document.createElement('div');
    div.innerHTML = `<strong>${username}</strong>: ${message}`;
    chatArea.appendChild(div);
    chatArea.scrollTop = chatArea.scrollHeight;
}

// メッセージ送信
document.getElementById('chat-form').addEventListener('submit', async (e) => {
    e.preventDefault();
    const input = document.getElementById('message-input');

    await fetch('/api/chat/send', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
        },
        body: JSON.stringify({
            username: 'ユーザー名',
            message: input.value,
        }),
    });

    input.value = '';
});
```

## Step 6: 動作確認

```bash
php artisan serve
```

ブラウザで2つのタブを開き、片方からメッセージを送信すると、もう片方にリアルタイムでメッセージが表示されます。

## Privateチャンネルを使う場合

上記の例はPublicチャンネルですが、認証が必要なPrivateチャンネルも簡単に使えます。

```php
// ChatMessageSent.php の broadcastOn を変更
public function broadcastOn(): Channel
{
    return new PrivateChannel('chat.room.' . $this->roomId);
}
```

```php
// routes/channels.php
Broadcast::channel('chat.room.{roomId}', function ($user, $roomId) {
    return $user->belongsToRoom($roomId);
});
```

フロントエンド側も `private` メソッドに変更するだけです：

```javascript
window.Echo.private(`chat.room.${roomId}`)
    .listen('.message.sent', (event) => {
        appendMessage(event.username, event.message);
    });
```

## まとめ

FluxSocketはPusher互換なので、Laravelの標準的なBroadcasting機能がそのまま使えます。特別なSDKの学習は不要で、既存のLaravelの知識だけでリアルタイム通信が実装できます。

**ポイント:**
- `pusher/pusher-php-server` と `pusher-js` がそのまま使える
- `.env` のホスト設定を変えるだけでFluxSocketに接続
- Public/Private/Presenceチャンネル全てに対応

> FluxSocket には純正のネイティブSDK（`flux-socket-js` など）もあります。本記事は「既存の Laravel Broadcasting 資産をそのまま活かす」ことを重視して Pusher 互換の道を採りました。ゼロから作るなら純正SDKも選べます。

:::message
FluxSocketは現在ベータユーザーを募集しています。無料のHobbyプランで気軽にお試しいただけます。
👉 [FluxSocket公式サイト](https://fluxsocket.com)
:::

## 関連記事

- [FluxSocketで作る8つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [WebSocketとは？HTTP通信との違いと使いどころをわかりやすく解説](https://zenn.dev/brainy_software/articles/websocket-vs-http-realtime)
- [PusherからFluxSocketへの移行ガイド](https://zenn.dev/brainy_software/articles/pusher-to-fluxsocket-migration)
