---
title: "【FluxSocket 準備編②】チャンネル設計 — Public / Private / Presence / Client Events"
emoji: "📡"
type: "tech"
topics: ["FluxSocket", "WebSocket", "リアルタイム通信", "チュートリアル"]
published: false
---

:::message
**FluxSocket 準備編（全3回）**

1. [アカウント作成から、最初の pub/sub まで](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-setup)
2. **チャンネル設計：Public / Private / Presence / Client Events** 👈 この記事
3. 認証と本番運用：認証エンドポイント・TLS・トラブルシュート（近日公開）

この記事だけでも読めますが、接続の基礎は[第1回](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-setup)で扱っています。
:::

[第1回](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-setup)では `hello` チャンネルで 1 往復を通しました。あの `hello` は「誰でも購読できる」もっとも単純なチャンネルです。実際のアプリでは、「この人だけに届けたい」「今この画面を見ているのは誰か」「サーバーを介さず即座にやり取りしたい」といった要求が出てきます。

FluxSocket（Pusher 互換）は、**チャンネル名のプレフィックス**で 3 種類のチャンネルを、**イベント名のプレフィックス**で Client Events を表現します。設計の勘所は「**どのプレフィックスを選ぶか**」に集約されます。

| 種類 | プレフィックス | 認証 | 向いている用途 |
|------|--------------|:----:|------|
| Public | なし | 不要 | 全体配信・公開フィード |
| Private | `private-` | 必要 | ユーザー個別の配信 |
| Presence | `presence-` | 必要 | 在室表示・共同編集 |
| Client Events | `client-`（イベント名） | 必要 | タイピング・カーソル等の低遅延 |

---

## 1. Public Channel — 全員に配信する

プレフィックスなしのチャンネルです。認証不要で、誰でも購読できます。第1回で使ったのはこれです。

```javascript
// 受信側（ブラウザ）
const channel = pusher.subscribe('news');
channel.bind('posted', (data) => {
    prependToFeed(data.title);
});
```

```php
// 送信側（サーバー・pusher-php-server）
$pusher->trigger('news', 'posted', [
    'title' => '新しいお知らせが投稿されました',
]);
```

ニュース配信、公開のライブフィード、全体向けのシステム通知など、「**誰が見ても同じ情報**」に向いています。認証がない分、実装がもっともシンプルです。

:::message
Public Channel は誰でも購読できます。特定ユーザーにしか見せてはいけない情報を Public に流さないこと。個別配信は次の Private を使います。
:::

---

## 2. Private Channel — この人だけに届ける

`private-` プレフィックスを付けると、購読時に**サーバー側の認証**が走り、許可されたユーザーだけが購読できるようになります。

```javascript
// 受信側：認証エンドポイントを指定して初期化
const pusher = new Pusher('YOUR_APP_KEY', {
    wsHost: 'YOUR_FLUX_HOST',
    wsPort: 8080,
    forceTLS: false,
    enabledTransports: ['ws'],
    cluster: 'mt1',
    disableStats: true,
    authEndpoint: '/auth/channel',   // ← サーバーの認証エンドポイント
});

// 自分専用のチャンネルを購読
const channel = pusher.subscribe('private-user-123');
channel.bind('notified', (data) => {
    showToast(data.message);
});
```

```php
// 送信側：特定ユーザーのチャンネルにだけ送る
$pusher->trigger('private-user-123', 'notified', [
    'message' => 'あなた宛のメッセージが届きました',
]);
```

購読時に `authEndpoint`（`/auth/channel`）へリクエストが飛び、サーバーが「このユーザーはこのチャンネルを購読してよいか」を判定して署名を返します。**この認証エンドポイントの実装は第3回で詳しく扱います。** ここでは「Private は購読に認証が要る」ことだけ押さえてください。

ユーザーごとの通知、アカウント固有のデータ更新など、「**本人だけに見せたい**」配信に使います。

---

## 3. Presence Channel — 誰がオンラインかを共有する

`presence-` プレフィックスのチャンネルは、Private の機能に加えて、**「今このチャンネルに誰がいるか」**をリアルタイムに共有します。

```javascript
const channel = pusher.subscribe('presence-room-1');

// 参加時点のメンバー一覧
channel.bind('pusher:subscription_succeeded', (members) => {
    setOnlineCount(members.count);
    members.each((m) => addAvatar(m.id, m.info.name));
});

// 誰かが参加 / 退出
channel.bind('pusher:member_added',   (m) => addAvatar(m.id, m.info.name));
channel.bind('pusher:member_removed', (m) => removeAvatar(m.id));
```

`pusher:subscription_succeeded` で参加時の人数・メンバー、`pusher:member_added` / `pusher:member_removed` で増減を受け取れます。**サーバーに人数を数えるコードを書く必要はありません** — FluxSocket が管理してくれます。

チャットルームの在室表示、共同編集の参加者アバター、「N 人が閲覧中」といった表示に最適です。認証時にユーザー情報（`user_id` と `user_info`）を含める必要があり、これも第3回で扱います。

---

## 4. Client Events — サーバーを介さず直接送る

`client-` で始まる**イベント名**は、サーバーを経由せずにクライアント間で直接送受信されます。Private または Presence チャンネル上でのみ使用できます。

```javascript
// 送信：タイピング中であることを他の参加者へ
channel.trigger('client-typing', { user: 'Taro' });

// 受信
channel.bind('client-typing', (data) => {
    showTypingIndicator(data.user);
});
```

サーバーへの HTTP リクエストを挟まない分、**遅延が最小**です。タイピングインジケーター、カーソル位置の共有、ドラッグ中の一時的な状態など、「**サーバーに保存する必要はないが、即座に他の人へ伝えたい**」用途に向いています。

:::message
Client Events を使うには、ダッシュボードのアプリ設定で「クライアントイベント」を有効にする必要があります（第1回・手順3の項目）。また、保存が必要なデータ（確定した発言など）は Client Events ではなくサーバー経由の trigger を使ってください。届かなくても困らないものだけを Client Events で。
:::

---

## どれを選ぶか（早見表）

| やりたいこと | 選ぶチャンネル |
|------|------|
| 全員に同じお知らせを配信 | **Public** |
| ログインユーザー個別に通知 | **Private** |
| 「誰が今いるか」を出したい | **Presence** |
| タイピング・カーソルを即共有 | **Presence / Private + Client Events** |
| 確定データ（発言・入札）を全員へ | **Public / Private + サーバー trigger** |

迷ったら **Public から始めて、認証が必要になった時点で Private へ**上げるのが素直です。Presence と Client Events は「人の存在」や「低遅延の一時状態」を扱うときの専用ツール、と捉えると設計しやすくなります。

---

## 次のステップ

Private と Presence は「購読に認証が要る」と繰り返し出てきました。次回は、その**認証エンドポイント**（`/auth/channel`）を PHP / Node.js で実装し、あわせて `ws.fluxsocket.com` への**本番接続**とつまずきやすい**トラブルの対処**までをまとめます。

👉 第3回：認証と本番運用（近日公開）

---

:::message
FluxSocket は現在ベータユーザーを募集しています。無料の Hobby プランで気軽にお試しいただけます。
[FluxSocket 公式サイト](https://fluxsocket.com)
:::

## 関連記事

- [FluxSocketで作る8つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [FluxSocket 準備編① アカウント作成から、最初の pub/sub まで](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-setup)
- [FluxSocketでリアルタイムチャットを実装する](https://zenn.dev/brainy_software/articles/fluxsocket-chat-tutorial)
