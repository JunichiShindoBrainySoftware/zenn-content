---
title: "FluxSocketでリアルタイム化する — 入札が全員に即座に届く"
emoji: "⚡"
type: "tech"
topics: ["FluxSocket", "React", "WebSocket", "リアルタイム", "Node.js"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-2)では、ポーリングで自動更新を実現しましたが、遅延と無駄なリクエストという問題が残りました。

今回は、ポーリングの `setInterval` を削除し、代わりに **FluxSocket**（Pusher互換の WebSocket SaaS）を導入します。サーバーから全クライアントにイベントをプッシュすることで、入札が即座に全員の画面に反映される体験を実現します。

**デモ:** [https://auction-demo.fluxsocket.com](https://auction-demo.fluxsocket.com)

複数のタブやデバイスで開いて、入札がリアルタイムに同期される様子を確かめてみてください。

### この記事で学べること

- Public Channel を使った入札のリアルタイムブロードキャスト
- Presence Channel を使った「今何人が見ているか」の表示
- Express サーバーから FluxSocket API を呼んでイベントを発火する方法
- React の状態管理と WebSocket イベントの統合パターン

### 対象読者

- [前回までの記事](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-2)のコードが手元にある方
- WebSocket を使いたいが、インフラの構築・運用は避けたい方
- Pusher 互換の API に興味がある方

---

## FluxSocket の準備

FluxSocket でアプリを作成し、接続情報を取得します。

1. [FluxSocket](https://fluxsocket.com) にアカウント登録（無料の Hobby プランで利用できます）
2. ダッシュボードから「新しいアプリ」を作成
3. 以下の情報を控えておく：
   - **App ID**
   - **Key**
   - **Secret**
   - **Host**

`.env` ファイルに追記します。

```env
FLUX_APP_ID=your-app-id
FLUX_APP_KEY=your-app-key
FLUX_APP_SECRET=your-app-secret
FLUX_HOST=your-fluxsocket-host
FLUX_PORT=443
FLUX_USE_TLS=true
SERVER_PORT=3000
```

---

## Step 1: サーバーにイベント発火機能を追加

前回までのサーバーには、入札を受け付ける API はありましたが、「他のクライアントに通知する」仕組みがありませんでした。ここに FluxSocket へのイベント発火を追加します。

### FluxSocket API でイベントを送信する関数

```js
const crypto = require('crypto');
const key = process.env.FLUX_APP_KEY;
const secret = process.env.FLUX_APP_SECRET;

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

HMAC-SHA256 で署名した HTTP リクエストを FluxSocket に送ります。この部分は Pusher の HTTP API と完全に同じ仕様です。

:::message
`pusher` npm パッケージを使えばこの署名処理を省略できます。今回は仕組みを理解するために手動で実装していますが、本番環境では SDK を使うのが良いかもしれません。
:::

### 入札 API にイベント発火を追加

前回の入札エンドポイントに、`triggerEvent` の呼び出しを1行追加します。

```js
app.post('/api/items/:id/bid', (req, res) => {
    const item = items.find(i => i.id === req.params.id);
    if (!item) return res.status(404).json({ error: 'アイテムが見つかりません' });
    if (Date.now() > item.endsAt) return res.status(400).json({ error: 'オークションは終了しました' });

    const { amount, userName } = req.body;
    if (!amount || amount <= item.currentBid) {
        return res.status(400).json({ error: `¥${(item.currentBid + 1000).toLocaleString()} 以上で入札してください` });
    }

    item.currentBid = amount;
    item.bidder = userName;
    item.bidCount++;

    // 【追加】FluxSocket で全員に通知
    triggerEvent('auction', 'bid-placed', {
        itemId: item.id,
        currentBid: item.currentBid,
        bidder: item.bidder,
        bidCount: item.bidCount,
    });

    res.json(item);
});
```

`triggerEvent('auction', 'bid-placed', ...)` — この1行が、入札情報を `auction` チャネルの全購読者にブロードキャストします。

### 設定 API と認証エンドポイントの追加

フロントエンドが WebSocket に接続するための設定 API と、Presence Channel の認証エンドポイントを追加します。

```js
// フロントエンドに接続情報を渡す
app.get('/api/config', (req, res) => res.json({
    key,
    host: process.env.FLUX_HOST || 'localhost',
    port: parseInt(process.env.FLUX_PORT || '443'),
    forceTLS: process.env.FLUX_USE_TLS === 'true',
}));

// Presence Channel の認証エンドポイント
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
```

Presence Channel では、ユーザー ID と名前を含めた `channel_data` を返します。これにより FluxSocket が「誰がこのチャネルにいるか」を管理してくれます。

---

## Step 2: フロントエンドで WebSocket を接続

HTML の `<head>` に pusher-js を追加します。

```html
<script src="https://js.pusher.com/8.2.0/pusher.min.js"></script>
```

FluxSocket は Pusher 互換なので、`pusher-js` ライブラリがそのまま使えます。

### ポーリングを削除し、WebSocket 接続に置き換える

前回の `App` コンポーネントを修正します。ポーリングの `setInterval` を削除し、代わりに FluxSocket への接続とイベント受信を実装します。

```jsx
function App() {
    const [userName, setUserName] = useState(null);
    const [items, setItems] = useState([]);
    const [bidItem, setBidItem] = useState(null);
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

        // 接続状態の管理
        pusher.connection.bind('connected', () => setConnected(true));
        pusher.connection.bind('disconnected', () => setConnected(false));

        // --- Public Channel: 入札イベントの受信 ---
        const auctionCh = pusher.subscribe('auction');
        auctionCh.bind('bid-placed', (data) => {
            // 該当アイテムの state を更新
            setItems(prev => prev.map(item =>
                item.id === data.itemId
                    ? { ...item, currentBid: data.currentBid, bidder: data.bidder, bidCount: data.bidCount }
                    : item
            ));
            // フラッシュエフェクト（1秒間ハイライト）
            setFlashIds(prev => new Set([...prev, data.itemId]));
            setTimeout(() => setFlashIds(prev => {
                const s = new Set(prev);
                s.delete(data.itemId);
                return s;
            }), 1000);
            // 最近の入札リストに追加
            setRecentBids(prev => [
                { itemId: data.itemId, bidder: data.bidder, amount: data.currentBid, time: Date.now() },
                ...prev.slice(0, 4),
            ]);
        });

        // --- Presence Channel: 閲覧者数の表示 ---
        const presenceCh = pusher.subscribe('presence-auction');
        presenceCh.bind('pusher:subscription_succeeded', (members) => setWatchers(members.count));
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

### UI に閲覧者数とフラッシュエフェクトを追加

ヘッダーに接続状態と閲覧者数を表示します。

```jsx
<div className="flex items-center gap-4">
    <span className="text-sm text-gray-500">👁 {watchers} 人が閲覧中</span>
    <span className="text-sm text-gray-600 font-medium">{userName}</span>
    <span className={`w-2 h-2 rounded-full ${connected ? 'bg-green-500' : 'bg-red-500'}`}></span>
</div>
```

商品カードにフラッシュエフェクトの props を追加します。

```jsx
function ItemCard({ item, onBidClick, flash }) {
    // ...
    return (
        <div className={`bg-white rounded-xl border border-gray-200 shadow-sm overflow-hidden
            transition-all duration-300 ${flash ? 'ring-2 ring-indigo-400 scale-[1.02]' : ''}`}>
            {/* ... */}
        </div>
    );
}
```

入札が入るとカードが一瞬拡大してインディゴ色のリングが表示されます。どのアイテムに入札があったのか視覚的に分かりやすくなります。

### サイドバーに最近の入札を表示

```jsx
<div className="lg:col-span-1">
    <div className="bg-white rounded-xl border border-gray-200 shadow-sm p-4 sticky top-20">
        <h2 className="font-bold text-gray-900 mb-3">最近の入札</h2>
        {recentBids.length === 0 ? (
            <p className="text-sm text-gray-400">まだ入札がありません</p>
        ) : (
            <div className="space-y-3">
                {recentBids.map((bid, i) => {
                    const item = items.find(it => it.id === bid.itemId);
                    return (
                        <div key={i} className="flex items-center gap-3 text-sm">
                            <span className="text-xl">{item?.image || '📦'}</span>
                            <div className="flex-1 min-w-0">
                                <p className="font-medium text-gray-800 truncate">{bid.bidder}</p>
                                <p className="text-indigo-600 font-bold">{formatPrice(bid.amount)}</p>
                            </div>
                        </div>
                    );
                })}
            </div>
        )}
    </div>
</div>
```

WebSocket でイベントを受信するたびに `recentBids` に追加されるので、サイドバーがリアルタイムで更新されます。

---

## Step 3: 動作確認

サーバーを起動して、複数タブで動かしてみます。

```bash
node server.js
```

ブラウザで http://localhost:3000 を開き、以下を試してみてください。

1. **タブ A** で名前を入力して参加
2. **タブ B** で別の名前を入力して参加 — ヘッダーの閲覧者数が「2 人が閲覧中」に増える
3. タブ A でアイテムに入札 — **タブ B にも即座に入札が反映**される
4. 入札されたカードが一瞬ハイライト（フラッシュエフェクト）
5. サイドバーの「最近の入札」にも即座に表示される

ポーリング版と比べて、遅延がほぼゼロになっていることを体感できるかと思います。

---

## 処理の流れを整理する

入札からリアルタイム更新までの全体の流れです。

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

サーバーは「イベントを発火する」だけ。WebSocket の接続管理やメッセージのルーティングは全て FluxSocket が担当してくれます。ポーリングのように無駄なリクエストも発生しません。

---

## ポーリング版との比較

| 項目 | ポーリング（前回） | WebSocket（今回） |
|:---|:---|:---|
| 更新の遅延 | 最大3秒（間隔に依存） | ほぼゼロ |
| 無駄なリクエスト | 毎回発生 | なし（イベント駆動） |
| サーバー負荷 | ユーザー数 x リクエスト/分 | イベント発生時のみ |
| 閲覧者数の表示 | 実装困難 | Presence Channel で簡単 |
| 入札の視覚フィードバック | 難しい | フラッシュエフェクトで即座に |

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

3本の記事を通して、オークションアプリの進化を段階的に体験してきました。

| 記事 | 方式 | 体験 |
|:---|:---|:---|
| 第1回 | 手動リロード | 他の人の入札が見えない |
| 第2回 | ポーリング | 自動更新されるが遅延あり・無駄なリクエスト |
| **第3回** | **WebSocket（FluxSocket）** | **即座に反映・無駄なし・閲覧者数も表示** |

FluxSocket は Pusher 互換なので、`pusher-js` をそのまま使えます。WebSocket のインフラ構築や接続管理を自前で行う必要がなく、チャネルを購読してサーバーから HTTP でイベントを発火する — というシンプルなパターンでリアルタイム機能を実現できます。

**3つのデモを比較してみる:**

- [手動リロード版](https://auction-static-demo.fluxsocket.com)
- [ポーリング版](https://auction-polling-demo.fluxsocket.com)
- [WebSocket版](https://auction-demo.fluxsocket.com)

:::message
FluxSocket は現在ベータユーザーを募集しています。無料の Hobby プランで気軽にお試しいただけます。
[FluxSocket 公式サイト](https://fluxsocket.com)
:::

## 関連記事

- [手動リロード vs ポーリング vs WebSocket — オークションアプリで体感する3つのリアルタイム実装](https://zenn.dev/brainy_software/articles/fluxsocket-auction-comparison)
- [FluxSocket で作る 8 つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [FluxSocket でリアルタイムチャットを実装する](https://zenn.dev/brainy_software/articles/fluxsocket-chat-tutorial)
- [WebSocket とは？HTTP 通信との違いと使いどころをわかりやすく解説](https://zenn.dev/brainy_software/articles/websocket-vs-http-realtime)
