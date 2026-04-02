---
title: "React + ExpressでオークションWebアプリを作る"
emoji: "🏷️"
type: "tech"
topics: ["React", "Express", "JavaScript", "オークション"]
published: false
---

## はじめに

「オークションアプリを作ってみたいけど、何から始めればいいか分からない」 — そう感じたことはないでしょうか。

この記事では、**React 18 + Express** を使って、オークションの基本機能を持つ Web アプリを一から構築します。商品一覧の表示、入札の送信、残り時間のカウントダウンなど、オークションに必要な要素を段階的に実装していきます。

ビルドツールは使いません。CDN から React と Tailwind CSS を読み込むシンプルな構成なので、`npm install` してすぐに動かせます。

### この記事で学べること

- Express で REST API（商品一覧・入札）を構築する方法
- React CDN を使ったフロントエンド UI の構築
- 入札バリデーションとエラーハンドリング
- カウントダウンタイマーの実装

### 対象読者

- React の基本文法（`useState`, `useEffect`）を知っている方
- REST API を使った Web アプリの全体像を掴みたい方
- 次のステップとして「リアルタイム機能」に興味がある方

---

## 完成イメージ

今回作るのは、6つのオークション商品が並ぶカード型の UI です。各カードには商品名、現在の最高入札額、入札数、残り時間が表示されます。「入札する」ボタンを押すとモーダルが開き、金額を選んで入札できます。

ただし、この記事の時点では **手動でページをリロードしないと、他の人の入札が画面に反映されません**。この「リアルタイムではない」状態をあえて体験することが、後続の記事への伏線になっています。

**デモ:** [https://auction-static-demo.fluxsocket.com](https://auction-static-demo.fluxsocket.com)

---

## 技術スタック

| 技術 | 役割 |
|------|------|
| **React 18**（CDN版） | フロントエンドの入札 UI |
| **Express** | API サーバー |
| **Tailwind CSS**（CDN版） | スタイリング |

---

## Step 1: プロジェクトのセットアップ

```bash
mkdir auction-app && cd auction-app
npm init -y
npm install express dotenv
```

プロジェクト構成はシンプルです。

```
auction-app/
├── server.js          # Express サーバー
├── public/
│   └── index.html     # React オークション UI
├── .env
└── package.json
```

---

## Step 2: サーバー実装

`server.js` を作成します。役割は2つだけです。

1. **アイテムデータの管理**（今回はインメモリ）
2. **入札 API** — 入札を受け付けてデータを更新

```js
require('dotenv').config();
const express = require('express');
const path = require('path');

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

// オークションアイテム（インメモリ）
// 当日24時（翌日0:00:00 JST）を終了時刻とする
function getTodayMidnightJST() {
    const now = new Date();
    const jstOffset = 9 * 60 * 60 * 1000;
    const nowJST = new Date(now.getTime() + jstOffset);
    const midnightJST = new Date(
        Date.UTC(nowJST.getUTCFullYear(), nowJST.getUTCMonth(), nowJST.getUTCDate() + 1, 0, 0, 0)
    );
    return midnightJST.getTime() - jstOffset;
}

const endsAtMidnight = getTodayMidnightJST();
const items = [
    { id: 'item-1', name: 'ヴィンテージ腕時計', image: '⌚', description: '1960年代の希少な機械式腕時計', startPrice: 50000, currentBid: 50000, bidder: null, bidCount: 0, endsAt: endsAtMidnight },
    { id: 'item-2', name: 'アンティーク花瓶', image: '🏺', description: '明治時代の有田焼花瓶', startPrice: 30000, currentBid: 30000, bidder: null, bidCount: 0, endsAt: endsAtMidnight },
    { id: 'item-3', name: 'レトロカメラ', image: '📷', description: 'ライカ M3 ボディ（動作品）', startPrice: 120000, currentBid: 120000, bidder: null, bidCount: 0, endsAt: endsAtMidnight },
    { id: 'item-4', name: '初版本コレクション', image: '📚', description: '夏目漱石「こころ」初版', startPrice: 80000, currentBid: 80000, bidder: null, bidCount: 0, endsAt: endsAtMidnight },
    { id: 'item-5', name: 'アートプリント', image: '🎨', description: '草間彌生 限定リトグラフ', startPrice: 200000, currentBid: 200000, bidder: null, bidCount: 0, endsAt: endsAtMidnight },
    { id: 'item-6', name: 'ギター', image: '🎸', description: 'Gibson Les Paul 1959 Reissue', startPrice: 350000, currentBid: 350000, bidder: null, bidCount: 0, endsAt: endsAtMidnight },
];

// アイテム一覧
app.get('/api/items', (req, res) => res.json(items));

// 入札エンドポイント
app.post('/api/items/:id/bid', (req, res) => {
    const item = items.find(i => i.id === req.params.id);
    if (!item) return res.status(404).json({ error: 'アイテムが見つかりません' });

    if (Date.now() > item.endsAt) {
        return res.status(400).json({ error: 'オークションは終了しました' });
    }

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

    res.json(item);
});

const PORT = process.env.SERVER_PORT || 3000;
app.listen(PORT, () => console.log(`Auction: http://localhost:${PORT}`));
```

入札 API のポイントは以下の通りです。

- オークション終了時刻を過ぎていないかチェック
- 入札額が現在の最高入札額より高いかバリデーション
- 条件を満たせばインメモリのデータを更新

シンプルですが、オークションの基本的なビジネスロジックが入っています。

:::message
本番環境ではデータベースが必要です。今回は学習用途なのでインメモリで進めます。
:::

---

## Step 3: フロントエンド実装

`public/index.html` を作成します。React 18 を CDN から読み込み、Babel でブラウザ内トランスパイルします。

### アプリの骨格

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>オークションアプリ</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js" crossorigin></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body class="bg-gray-50 min-h-screen">
<div id="root"></div>

<script type="text/babel">
const { useState, useEffect, useCallback } = React;

const formatPrice = (n) => `¥${n.toLocaleString()}`;

function timeLeft(endsAt) {
    const diff = endsAt - Date.now();
    if (diff <= 0) return '終了';
    const m = Math.floor(diff / 60000);
    const s = Math.floor((diff % 60000) / 1000);
    return `${m}:${s.toString().padStart(2, '0')}`;
}
```

`formatPrice` は金額のフォーマット、`timeLeft` は残り時間の計算です。1秒ごとにカウントダウンを更新するために使います。

### 名前入力モーダル

```jsx
function NameModal({ onJoin }) {
    const [name, setName] = useState('');
    const submit = () => { if (name.trim()) onJoin(name.trim()); };
    return (
        <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
            <div className="bg-white rounded-xl p-8 w-80 shadow-2xl">
                <h2 className="text-xl font-bold text-gray-900 mb-2">オークション</h2>
                <p className="text-sm text-gray-500 mb-4">名前を入力して参加</p>
                <input type="text" value={name} onChange={e => setName(e.target.value)}
                       onKeyDown={e => e.key === 'Enter' && submit()}
                       placeholder="あなたの名前"
                       className="w-full border border-gray-300 rounded-lg px-4 py-2 mb-4 focus:ring-2 focus:ring-indigo-500 focus:border-transparent outline-none" />
                <button onClick={submit}
                        className="w-full bg-indigo-600 text-white py-2 rounded-lg font-bold hover:bg-indigo-500 transition">
                    参加する
                </button>
            </div>
        </div>
    );
}
```

### 入札モーダル

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
        <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50" onClick={onClose}>
            <div className="bg-white rounded-xl p-6 w-96 shadow-2xl" onClick={e => e.stopPropagation()}>
                <h3 className="text-lg font-bold text-gray-900 mb-1">{item.image} {item.name}</h3>
                <p className="text-sm text-gray-500 mb-4">
                    現在の最高入札: <span className="font-bold text-indigo-600">{formatPrice(item.currentBid)}</span>
                </p>
                <div className="mb-4">
                    <label className="block text-sm font-medium text-gray-700 mb-1">入札額</label>
                    <div className="flex gap-2">
                        {[minBid, minBid + 5000, minBid + 10000].map(v => (
                            <button key={v} onClick={() => setAmount(v)}
                                    className={`flex-1 py-2 rounded-lg text-sm font-bold transition
                                        ${amount === v
                                            ? 'bg-indigo-600 text-white'
                                            : 'bg-gray-100 text-gray-700 hover:bg-gray-200'}`}>
                                {formatPrice(v)}
                            </button>
                        ))}
                    </div>
                    <input type="number" value={amount} onChange={e => setAmount(parseInt(e.target.value) || 0)}
                           min={minBid} step={1000}
                           className="w-full mt-2 border border-gray-300 rounded-lg px-4 py-2 focus:ring-2 focus:ring-indigo-500 outline-none" />
                </div>
                {error && <p className="text-red-500 text-sm mb-3">{error}</p>}
                <div className="flex gap-2">
                    <button onClick={onClose}
                            className="flex-1 py-2 rounded-lg border border-gray-300 text-gray-700 hover:bg-gray-50 transition">
                        キャンセル
                    </button>
                    <button onClick={submit} disabled={loading}
                            className="flex-1 py-2 rounded-lg bg-indigo-600 text-white font-bold hover:bg-indigo-500 transition disabled:opacity-50">
                        {loading ? '送信中...' : `${formatPrice(amount)} で入札`}
                    </button>
                </div>
            </div>
        </div>
    );
}
```

3つのプリセットボタン（+1,000円 / +6,000円 / +11,000円）を用意しているので、ユーザーは金額を手入力しなくても素早く入札できます。

### 商品カード

```jsx
function ItemCard({ item, onBidClick }) {
    const [remaining, setRemaining] = useState(timeLeft(item.endsAt));
    const ended = Date.now() > item.endsAt;

    useEffect(() => {
        const timer = setInterval(() => setRemaining(timeLeft(item.endsAt)), 1000);
        return () => clearInterval(timer);
    }, [item.endsAt]);

    return (
        <div className="bg-white rounded-xl border border-gray-200 shadow-sm overflow-hidden">
            <div className="p-5">
                <div className="flex items-start justify-between mb-3">
                    <div className="text-4xl">{item.image}</div>
                    <span className={`text-xs font-bold px-2 py-1 rounded-full
                        ${ended ? 'bg-gray-100 text-gray-500' : 'bg-green-50 text-green-700'}`}>
                        {ended ? '終了' : `残り ${remaining}`}
                    </span>
                </div>
                <h3 className="font-bold text-gray-900 mb-1">{item.name}</h3>
                <p className="text-sm text-gray-500 mb-4">{item.description}</p>
                <div className="flex items-end justify-between mb-3">
                    <div>
                        <p className="text-xs text-gray-400">現在の最高入札</p>
                        <p className="text-2xl font-bold text-indigo-600">{formatPrice(item.currentBid)}</p>
                    </div>
                    <div className="text-right">
                        <p className="text-xs text-gray-400">入札数</p>
                        <p className="text-lg font-bold text-gray-700">{item.bidCount}</p>
                    </div>
                </div>
                {item.bidder && (
                    <p className="text-xs text-gray-500 mb-3">
                        最高入札者: <span className="font-medium text-gray-700">{item.bidder}</span>
                    </p>
                )}
                <button onClick={() => onBidClick(item)} disabled={ended}
                        className="w-full py-2.5 rounded-lg bg-indigo-600 text-white font-bold hover:bg-indigo-500 transition disabled:opacity-40 disabled:cursor-not-allowed">
                    {ended ? 'オークション終了' : '入札する'}
                </button>
            </div>
        </div>
    );
}
```

`useEffect` で1秒ごとに `setInterval` を回し、残り時間をカウントダウンしています。

### メインの App コンポーネント

```jsx
function App() {
    const [userName, setUserName] = useState(null);
    const [items, setItems] = useState([]);
    const [bidItem, setBidItem] = useState(null);

    const join = useCallback(async (name) => {
        setUserName(name);
        const res = await fetch('/api/items');
        setItems(await res.json());
    }, []);

    if (!userName) return <NameModal onJoin={join} />;

    return (
        <div className="min-h-screen">
            {/* ヘッダー */}
            <div className="border-b border-gray-200 bg-white/80 backdrop-blur sticky top-0 z-40">
                <div className="max-w-6xl mx-auto px-4 py-3 flex items-center justify-between">
                    <div>
                        <h1 className="text-lg font-bold text-gray-900">オークション</h1>
                        <p className="text-xs text-gray-500">入札するには各商品の「入札する」ボタンを押してください</p>
                    </div>
                    <span className="text-sm text-gray-600 font-medium">{userName}</span>
                </div>
            </div>

            {/* 商品グリッド */}
            <div className="max-w-6xl mx-auto px-4 py-6">
                <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                    {items.map(item => (
                        <ItemCard key={item.id} item={item} onBidClick={setBidItem} />
                    ))}
                </div>
            </div>

            {/* 入札モーダル */}
            {bidItem && (
                <BidModal item={bidItem} userName={userName}
                          onBid={(updated) => setItems(prev => prev.map(i => i.id === updated.id ? updated : i))}
                          onClose={() => setBidItem(null)} />
            )}
        </div>
    );
}

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
</script>
</body>
</html>
```

`join` 関数は名前入力後に `GET /api/items` でアイテム一覧を取得し、state にセットします。入札成功時は `onBid` コールバックで該当アイテムだけを更新しています。

---

## Step 4: 動作確認

サーバーを起動します。

```bash
node server.js
```

ブラウザで http://localhost:3000 を開いてみてください。

1. 名前を入力して「参加する」をクリック
2. 商品カードの「入札する」ボタンをクリック
3. 金額を選んで入札
4. 入札が成功すると、**自分の画面では**価格が更新される

ここまでは問題なく動作するかと思います。

---

## 試してみてください: 2つのタブで開く

では、ブラウザで**もう1つタブ**を開いて、別の名前で参加してみてください。

1. **タブ A** で「太郎」として参加し、ヴィンテージ腕時計に入札
2. **タブ B** で「花子」として参加 — タブ A の入札は**反映されていない**
3. タブ B で**ページをリロード**すると、太郎の入札が反映される

お気づきの通り、**手動でリロードしないと他の人の入札が見えない**状態です。

これは当然の動作です。タブ A の入札はサーバーのインメモリデータを更新していますが、タブ B にはそのことが伝わっていません。タブ B は最初に `GET /api/items` で取得したデータをそのまま表示し続けているだけです。

```
タブA（入札）                     サーバー（Express）              タブB（閲覧中）
    |                                   |                              |
    |-- POST /api/items/:id/bid ------->|                              |
    |                                   |（データ更新）                 |
    |<-- 200 OK（更新後のアイテム）------|                              |
    |                                   |                              |
    | タブAの画面は更新される            |        タブBは何も知らない     |
    |                                   |                              |
    |                                   |     リロードしないと          |
    |                                   |     最新データが見えない       |
```

実際のオークションサービスでこの状態では困りますよね。他の人がいくらで入札したか分からないまま、自分の入札額を決めなければなりません。

---

## この問題をどう解決するか

次の記事では、この「他の人の入札が見えない」問題に対して、最初に思いつく解決策 — **ポーリング**（定期的にサーバーにデータを取りに行く方式）を実装してみます。ポーリングで自動更新はできるようになりますが、そこにもトレードオフがあります。

次回: [ポーリングで自動更新する — そのメリットと限界](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-2)

---

## 関連記事

- [FluxSocket で作る 8 つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [WebSocket とは？HTTP 通信との違いと使いどころをわかりやすく解説](https://zenn.dev/brainy_software/articles/websocket-vs-http-realtime)
