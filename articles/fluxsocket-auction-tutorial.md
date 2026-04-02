---
title: "React + Express で作るリアルタイムオークション — 入札が全員に即座に届く仕組み"
emoji: "🏺"
type: "tech"
topics: ["FluxSocket", "React", "WebSocket", "リアルタイム", "Node.js"]
published: false
---

## はじめに

「オークションサイトで入札したのに、更新ボタンを押すまで最新価格が分からない」 — そんな体験をしたことはないでしょうか。

リアルタイム入札は、ECやオークション系サービスだけでなく、社内のリソース予約やイベントのチケット争奪など、**「同時に複数人が同じ対象に対してアクションする」**場面で広く必要とされます。しかしいざ実装しようとすると、WebSocket サーバーの構築・運用・スケーリングが課題になりがちです。

この記事では、**React 18 + Express + FluxSocket** を使って、入札が全員にリアルタイムで届くオークションアプリを構築します。FluxSocket は Pusher 互換の WebSocket SaaS なので、`pusher-js` ライブラリをそのまま使えます。WebSocket のインフラを自前で管理する必要がありません。

完成デモはこちらで触れます。
**デモ:** [https://auction-demo.fluxsocket.com](https://auction-demo.fluxsocket.com)

### この記事で学べること

- Public Channel を使った入札のリアルタイムブロードキャスト
- Presence Channel を使った「今何人が見ているか」の表示
- Express サーバーから FluxSocket API を呼んでイベントを発火する方法
- React の状態管理と WebSocket イベントの統合パターン

### 対象読者

- React の基本文法（`useState`, `useEffect`）を知っている方
- リアルタイム機能を Web アプリに追加してみたい方
- WebSocket を使いたいが、インフラの構築・運用は避けたい方

---

## 技術スタック

| 技術 | 役割 |
|------|------|
| **React 18**（CDN版） | フロントエンドの入札UI |
| **Express** | API サーバー + Presence Channel 認証 |
| **pusher-js** | FluxSocket への WebSocket 接続 |
| **Tailwind CSS**（CDN版） | スタイリング |
| **FluxSocket** | リアルタイム通信基盤（Pusher互換 SaaS） |

今回はビルドツールなしの構成です。CDN から React と Tailwind を読み込むので、`npm install` してすぐに動かせます。

---

## Step 1: FluxSocket の準備

FluxSocket でアプリを作成し、接続情報を取得します。

1. [FluxSocket](https://fluxsocket.com) にアカウント登録（無料の Hobby プランで利用できます）
2. ダッシュボードから「新しいアプリ」を作成
3. 以下の情報を控えておく：
   - **App ID**
   - **Key**
   - **Secret**
   - **Host**

---

## Step 2: プロジェクトのセットアップ

```bash
mkdir auction-app && cd auction-app
npm init -y
npm install express dotenv
```

`.env` ファイルを作成して、先ほど控えた接続情報を設定します。

```env
FLUX_APP_ID=your-app-id
FLUX_APP_KEY=your-app-key
FLUX_APP_SECRET=your-app-secret
FLUX_HOST=your-fluxsocket-host
FLUX_PORT=443
FLUX_USE_TLS=true
SERVER_PORT=3000
```

プロジェクト構成はシンプルです。

```
auction-app/
├── server.js          # Express サーバー
├── public/
│   └── index.html     # React オークションUI
├── .env
└── package.json
```

---

## Step 3: サーバー実装

`server.js` を作成します。主な役割は3つです。

1. **アイテムデータの管理**（今回はインメモリ）
2. **入札 API** — 入札を受け付けて FluxSocket にイベントを発火
3. **チャネル認証** — Presence Channel の認証エンドポイント

### FluxSocket へのイベント発火

FluxSocket は Pusher 互換の HTTP API を持っています。サーバーから直接 HTTP リクエストを送ってイベントを発火できます。

```js
require('dotenv').config();
const express = require('express');
const path = require('path');
const crypto = require('crypto');

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

const key = process.env.FLUX_APP_KEY;
const secret = process.env.FLUX_APP_SECRET;

// FluxSocket API でイベントを発火する関数
function triggerEvent(channel, event, data) {
    const body = JSON.stringify({
        name: event,
        channel,
        data: JSON.stringify(data),
    });
    const ts = Math.floor(Date.now() / 1000).toString();
    const md5 = crypto.createHash('md5').update(body).digest('hex');
    const apiPath = `/apps/${process.env.FLUX_APP_ID}/events`;
    const params = `auth_key=${key}&auth_timestamp=${ts}&auth_version=1.0&body_md5=${md5}`;
    const sig = crypto
        .createHmac('sha256', secret)
        .update(`POST\n${apiPath}\n${params}`)
        .digest('hex');

    const host = process.env.FLUX_HOST;
    const port = process.env.FLUX_PORT || '443';
    const protocol = process.env.FLUX_USE_TLS === 'true' ? 'https' : 'http';
    const url = `${protocol}://${host}:${port}${apiPath}?${params}&auth_signature=${sig}`;

    fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body,
    }).catch(err => console.error('Trigger error:', err.message));
}
```

HMAC-SHA256 で署名した HTTP リクエストを FluxSocket に送ります。この部分は Pusher の HTTP API と完全に同じ仕様です。`pusher` サーバー SDK を使えばこの署名処理を省略できますが、今回は仕組みを理解するために手動で実装しています。

:::message
本番環境では `pusher` npm パッケージを使うのが良いかもしれません。署名処理を自前で書く必要がなくなります。
:::

### オークションアイテムと入札 API

```js
// オークションアイテム（インメモリ）
const items = [
    {
        id: 'item-1',
        name: 'ヴィンテージ腕時計',
        image: '⌚',
        description: '1960年代の希少な機械式腕時計',
        startPrice: 50000,
        currentBid: 50000,
        bidder: null,
        bidCount: 0,
        endsAt: Date.now() + 24 * 60 * 60 * 1000, // 24時間後
    },
    // ... 他のアイテムも同様に定義
];

// 設定を返す API（フロントエンドが WebSocket 接続に使用）
app.get('/api/config', (req, res) => res.json({
    key,
    host: process.env.FLUX_HOST,
    port: parseInt(process.env.FLUX_PORT || '443'),
    forceTLS: process.env.FLUX_USE_TLS === 'true',
}));

// アイテム一覧
app.get('/api/items', (req, res) => res.json(items));

// 入札エンドポイント
app.post('/api/items/:id/bid', (req, res) => {
    const item = items.find(i => i.id === req.params.id);
    if (!item) return res.status(404).json({ error: 'アイテムが見つかりません' });
    if (Date.now() > item.endsAt) return res.status(400).json({ error: 'オークションは終了しました' });

    const { amount, userName } = req.body;
    if (!amount || amount <= item.currentBid) {
        return res.status(400).json({
            error: `¥${(item.currentBid + 1000).toLocaleString()} 以上で入札してください`,
        });
    }

    // 入札を反映
    item.currentBid = amount;
    item.bidder = userName;
    item.bidCount++;

    // FluxSocket で全員に通知
    triggerEvent('auction', 'bid-placed', {
        itemId: item.id,
        currentBid: item.currentBid,
        bidder: item.bidder,
        bidCount: item.bidCount,
    });

    res.json(item);
});
```

ポイントは `triggerEvent('auction', 'bid-placed', ...)` の部分です。入札が成功すると、`auction` チャネルに `bid-placed` イベントを発火します。このチャネルを購読している全クライアントに、入札情報がリアルタイムで届きます。

### Presence Channel の認証エンドポイント

```js
app.post('/auth/channel', (req, res) => {
    const { socket_id, channel_name } = req.body;
    const userName = req.body.user_name || 'Guest';
    const userId = req.body.user_id || crypto.randomUUID();

    if (channel_name.startsWith('presence-')) {
        // Presence Channel: ユーザー情報を含めて署名
        const userData = JSON.stringify({
            user_id: userId,
            user_info: { name: userName },
        });
        const strToSign = `${socket_id}:${channel_name}:${userData}`;
        const signature = crypto
            .createHmac('sha256', secret)
            .update(strToSign)
            .digest('hex');
        return res.json({
            auth: `${key}:${signature}`,
            channel_data: userData,
        });
    }

    // Private Channel の認証
    const strToSign = `${socket_id}:${channel_name}`;
    const signature = crypto
        .createHmac('sha256', secret)
        .update(strToSign)
        .digest('hex');
    res.json({ auth: `${key}:${signature}` });
});

const PORT = process.env.SERVER_PORT || 3000;
app.listen(PORT, () => console.log(`Auction: http://localhost:${PORT}`));
```

Presence Channel では、ユーザー ID と名前を含めた `channel_data` を返します。これにより FluxSocket が「誰がこのチャネルにいるか」を管理してくれます。

---

## Step 4: フロントエンド実装

`public/index.html` を作成します。React 18 を CDN から読み込み、Babel でブラウザ内トランスパイルします。

### WebSocket 接続と入札イベントの受信

```jsx
const { useState, useEffect, useRef, useCallback } = React;

function App() {
    const [userName, setUserName] = useState(null);
    const [items, setItems] = useState([]);
    const [flashIds, setFlashIds] = useState(new Set());
    const [connected, setConnected] = useState(false);
    const [watchers, setWatchers] = useState(0);
    const [recentBids, setRecentBids] = useState([]);
    const pusherRef = useRef(null);

    const join = useCallback(async (name) => {
        setUserName(name);

        // サーバーから FluxSocket の接続情報を取得
        const configRes = await fetch('/api/config');
        const config = await configRes.json();

        // アイテム一覧を取得
        const itemsRes = await fetch('/api/items');
        setItems(await itemsRes.json());

        const userId = Math.random().toString(36).slice(2, 10);

        // FluxSocket に接続（pusher-js をそのまま使用）
        const pusher = new Pusher(config.key, {
            wsHost: config.host,
            wsPort: config.port,
            wssPort: config.port,
            forceTLS: config.forceTLS || false,
            enabledTransports: ['ws', 'wss'],
            cluster: 'mt1',
            disableStats: true,
            authEndpoint: '/auth/channel',
            auth: { params: { user_name: name, user_id: userId } },
        });
        pusherRef.current = pusher;

        pusher.connection.bind('connected', () => setConnected(true));
        pusher.connection.bind('disconnected', () => setConnected(false));

        // --- Public Channel: 入札イベントの受信 ---
        const auctionCh = pusher.subscribe('auction');
        auctionCh.bind('bid-placed', (data) => {
            setItems(prev => prev.map(item =>
                item.id === data.itemId
                    ? { ...item, currentBid: data.currentBid, bidder: data.bidder, bidCount: data.bidCount }
                    : item
            ));
            // 入札されたカードをハイライト
            setFlashIds(prev => new Set([...prev, data.itemId]));
            setTimeout(() => {
                setFlashIds(prev => {
                    const s = new Set(prev);
                    s.delete(data.itemId);
                    return s;
                });
            }, 1000);
        });

        // --- Presence Channel: 閲覧者数の表示 ---
        const presenceCh = pusher.subscribe('presence-auction');
        presenceCh.bind('pusher:subscription_succeeded', (members) => {
            setWatchers(members.count);
        });
        presenceCh.bind('pusher:member_added', () => setWatchers(w => w + 1));
        presenceCh.bind('pusher:member_removed', () => setWatchers(w => Math.max(0, w - 1)));
    }, []);

    // ... UI レンダリング
}
```

ここが今回の核心部分です。2種類のチャネルを同時に使っています。

**Public Channel（`auction`）**
認証不要のチャネルです。サーバーが `bid-placed` イベントを発火すると、購読している全クライアントに届きます。入札されたアイテムのカードが一瞬ハイライトされる「フラッシュエフェクト」も実装しています。

**Presence Channel（`presence-auction`）**
認証が必要なチャネルで、「誰が今このチャネルにいるか」を FluxSocket が管理してくれます。`pusher:subscription_succeeded` でチャネル参加時の人数を取得し、`pusher:member_added` / `pusher:member_removed` で増減を反映します。ヘッダーの「N 人が閲覧中」の表示に使っています。

### 入札 UI

入札モーダルでは、最低入札額を自動計算して3つのプリセットボタンを表示しています。

```jsx
function BidModal({ item, userName, onBid, onClose }) {
    const minBid = item.currentBid + 1000;
    const [amount, setAmount] = useState(minBid);
    const [error, setError] = useState('');
    const [loading, setLoading] = useState(false);

    const submit = async () => {
        setLoading(true);
        setError('');
        try {
            const res = await fetch(`/api/items/${item.id}/bid`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ amount, userName }),
            });
            const data = await res.json();
            if (!res.ok) { setError(data.error); setLoading(false); return; }
            onBid(data);
            onClose();
        } catch { setError('通信エラー'); setLoading(false); }
    };

    return (
        <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
            <div className="bg-white rounded-xl p-6 w-96 shadow-2xl">
                <h3 className="text-lg font-bold">{item.image} {item.name}</h3>
                <p className="text-sm text-gray-500 mb-4">
                    現在の最高入札:
                    <span className="font-bold text-indigo-600">
                        ¥{item.currentBid.toLocaleString()}
                    </span>
                </p>
                {/* プリセットボタン: +1,000 / +6,000 / +11,000 */}
                <div className="flex gap-2 mb-2">
                    {[minBid, minBid + 5000, minBid + 10000].map(v => (
                        <button key={v} onClick={() => setAmount(v)}
                            className={`flex-1 py-2 rounded-lg text-sm font-bold
                                ${amount === v
                                    ? 'bg-indigo-600 text-white'
                                    : 'bg-gray-100 text-gray-700'}`}>
                            ¥{v.toLocaleString()}
                        </button>
                    ))}
                </div>
                {error && <p className="text-red-500 text-sm">{error}</p>}
                <button onClick={submit} disabled={loading}
                    className="w-full py-2 bg-indigo-600 text-white font-bold rounded-lg">
                    {loading ? '送信中...' : `¥${amount.toLocaleString()} で入札`}
                </button>
            </div>
        </div>
    );
}
```

入札 API はシンプルな POST リクエストです。成功するとサーバー側で `triggerEvent` が呼ばれ、**入札した本人も含めた全員に** `bid-placed` イベントが配信されます。

---

## Step 5: 動作確認

サーバーを起動して、複数タブで動かしてみます。

```bash
node server.js
```

ブラウザで http://localhost:3000 を開き、以下を試してみてください。

1. **タブ A** で名前を入力して参加
2. **タブ B** で別の名前を入力して参加 — ヘッダーの閲覧者数が「2 人が閲覧中」に増える
3. タブ A でアイテムに入札 — **タブ B にもリアルタイムで入札が反映**される
4. 入札されたカードが一瞬ハイライト（フラッシュエフェクト）
5. サイドバーの「最近の入札」にも即座に表示される

リロード不要で全員の画面が同期される体験を確認できるかと思います。

---

## 処理の流れを整理する

入札からリアルタイム更新までの流れを図で整理しておきます。

```
ユーザーA（入札）                     サーバー（Express）              FluxSocket
    |                                   |                              |
    |-- POST /api/items/:id/bid ------->|                              |
    |                                   |-- HTTP triggerEvent -------->|
    |                                   |    channel: "auction"        |
    |                                   |    event: "bid-placed"       |
    |                                   |                              |
    |<-- WebSocket: bid-placed ---------|------- broadcast ------------|
    |                                                                  |
ユーザーB（閲覧中）                                                    |
    |<-- WebSocket: bid-placed --------|------- broadcast -------------|
```

1. ユーザー A が入札 API を叩く
2. サーバーが入札を検証・反映し、FluxSocket に HTTP でイベントを送信
3. FluxSocket が `auction` チャネルの全購読者に WebSocket でイベントを配信
4. 各クライアントの React が `bid-placed` イベントを受け取り、state を更新

サーバーは「イベントを発火する」だけ。WebSocket の接続管理やメッセージのルーティングは全て FluxSocket が担当してくれます。

---

## 応用のヒント

今回のデモは学習用にシンプルな構成にしていますが、本番で使う場合はいくつか検討すべき点があります。

### 自分のユースケースに合っているか確認してみてください

- [ ] 入札データの永続化が必要か（今回はインメモリ。本番では DB が必須です）
- [ ] 認証の強化が必要か（今回は名前入力のみ。本番ではログイン連携が望ましいです）
- [ ] 楽観的 UI 更新を入れるか（入札ボタン押下時にすぐ UI を更新し、失敗時にロールバック）
- [ ] 入札の競合制御（同時入札への対処。DB のトランザクションやバージョンロック等）
- [ ] 残り時間の同期（サーバー時刻を基準にするとクライアント間のずれを防げます）

---

## まとめ

この記事では、React + Express + FluxSocket を使って、リアルタイムオークションアプリを構築しました。

**実装した機能:**

- Public Channel による入札のリアルタイムブロードキャスト
- Presence Channel による「N 人が閲覧中」表示
- フラッシュエフェクトによる入札の視覚フィードバック
- 最近の入札履歴のリアルタイム表示

WebSocket のインフラ構築や接続管理を自前で行う必要がなく、`pusher-js` を使ってチャネルを購読し、サーバーから HTTP でイベントを発火する — というシンプルなパターンでリアルタイム機能を実現できることを体感いただけたのではないかと思います。

**デモを触ってみる:** [https://auction-demo.fluxsocket.com](https://auction-demo.fluxsocket.com)

:::message
FluxSocket は現在ベータユーザーを募集しています。無料の Hobby プランで気軽にお試しいただけます。
[FluxSocket 公式サイト](https://fluxsocket.com)
:::

## 関連記事

- [FluxSocket で作る 8 つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [FluxSocket でリアルタイムチャットを実装する](https://zenn.dev/brainy_software/articles/fluxsocket-chat-tutorial)
- [WebSocket とは？HTTP 通信との違いと使いどころをわかりやすく解説](https://zenn.dev/brainy_software/articles/websocket-vs-http-realtime)
