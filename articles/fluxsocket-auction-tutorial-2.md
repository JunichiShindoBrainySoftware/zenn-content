---
title: "ポーリングで自動更新する — そのメリットと限界"
emoji: "🔄"
type: "tech"
topics: ["React", "Express", "JavaScript", "ポーリング"]
published: true
---

## はじめに

[前回の記事](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-1)では、React + Express でオークションアプリの基本機能を構築しました。入札の送信と表示は動いていましたが、**他の人の入札が自分の画面に反映されない**という問題がありました。

今回はこの問題を「ポーリング」で解決してみます。数秒おきにサーバーにデータを取りに行くことで、画面を自動更新する方式です。

実装してみると自動更新はできるようになりますが、そこには見過ごせないトレードオフがあることも体感できるかと思います。

**デモ:** [https://auction-polling-demo.fluxsocket.com](https://auction-polling-demo.fluxsocket.com)

### 対象読者

- [前回の記事](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-1)のコードが手元にある方
- ポーリングの仕組みとトレードオフを理解したい方
- 「リアルタイム」と「定期的な更新」の違いを体感したい方

---

## ポーリングとは

ポーリングは、クライアントが一定間隔でサーバーにリクエストを送り、最新データを取得する方式です。

```
クライアント                  サーバー
    |                           |
    |-- GET /api/items -------->|
    |<-- 200 OK（アイテム一覧）--|
    |                           |
    |  （30秒待つ）              |
    |                           |
    |-- GET /api/items -------->|
    |<-- 200 OK（アイテム一覧）--|
    |                           |
    |  （30秒待つ）              |
    |                           |
    |-- GET /api/items -------->|  ← データに変更がなくても毎回リクエスト
    |<-- 200 OK（アイテム一覧）--|
```

Webの黎明期から使われてきた古典的な手法で、実装がシンプルなのが大きな利点です。

---

## Step 1: サーバー側の変更

サーバー側は**変更不要**です。前回作った `GET /api/items` と `POST /api/items/:id/bid` がそのまま使えます。

ポーリングはクライアント側の実装だけで完結する — これもポーリングの利点の一つです。

---

## Step 2: ポーリングの実装

前回の `App` コンポーネントに `setInterval` を追加するだけです。変更箇所をハイライトします。

```jsx
function App() {
    const [userName, setUserName] = useState(null);
    const [items, setItems] = useState([]);
    const [bidItem, setBidItem] = useState(null);
    // 【追加】ポーリングのリクエスト数をカウント（デバッグ用）
    const [pollCount, setPollCount] = useState(0);

    const join = useCallback(async (name) => {
        setUserName(name);
        const res = await fetch('/api/items');
        setItems(await res.json());
    }, []);

    // 【追加】ポーリング: 30秒おきに最新データを取得
    useEffect(() => {
        if (!userName) return;

        const interval = setInterval(async () => {
            try {
                const res = await fetch('/api/items');
                const data = await res.json();
                setItems(data);
                setPollCount(prev => prev + 1);
            } catch (err) {
                console.error('ポーリングエラー:', err);
            }
        }, 30000); // 30秒間隔

        return () => clearInterval(interval);
    }, [userName]);

    if (!userName) return <NameModal onJoin={join} />;

    return (
        <div className="min-h-screen">
            {/* ヘッダー */}
            <div className="border-b border-gray-200 bg-white/80 backdrop-blur sticky top-0 z-40">
                <div className="max-w-6xl mx-auto px-4 py-3 flex items-center justify-between">
                    <div>
                        <h1 className="text-lg font-bold text-gray-900">オークション（ポーリング版）</h1>
                        <p className="text-xs text-gray-500">30秒おきにサーバーからデータを取得しています</p>
                    </div>
                    <div className="flex items-center gap-4">
                        {/* 【追加】ポーリング回数の表示 */}
                        <span className="text-xs text-gray-400">
                            リクエスト: {pollCount}回
                        </span>
                        <span className="text-sm text-gray-600 font-medium">{userName}</span>
                    </div>
                </div>
            </div>

            {/* 商品グリッドは前回と同じ */}
            <div className="max-w-6xl mx-auto px-4 py-6">
                <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                    {items.map(item => (
                        <ItemCard key={item.id} item={item} onBidClick={setBidItem} />
                    ))}
                </div>
            </div>

            {bidItem && (
                <BidModal item={bidItem} userName={userName}
                          onBid={(updated) => setItems(prev => prev.map(i => i.id === updated.id ? updated : i))}
                          onClose={() => setBidItem(null)} />
            )}
        </div>
    );
}
```

追加したのは主に2箇所です。

1. **`useEffect` 内の `setInterval`** — 30秒ごとに `GET /api/items` を呼び、state を更新
2. **`pollCount`** — リクエスト回数を表示して、ポーリングが動いていることを可視化

`NameModal`, `BidModal`, `ItemCard` は前回のコードをそのまま使えます。

---

## Step 3: 動作確認

サーバーを起動して、2つのタブで開いてみてください。

```bash
node server.js
```

1. **タブ A** で「太郎」として参加し、ヴィンテージ腕時計に入札
2. **タブ B** で「花子」として参加 — **数秒後に**太郎の入札が反映される
3. ヘッダーのリクエスト回数が増え続けていることを確認

前回は手動リロードが必要でしたが、今回は自動で更新されるようになりました。

---

## ポーリングの問題点を体感する

自動更新ができるようになったのは良いのですが、しばらくデモを眺めていると気になる点がいくつか出てきます。

### 問題 1: 遅延がある

ポーリング間隔を30秒に設定しているので、入札してから他の人の画面に反映されるまで**最大30秒のラグ**があります。

オークションの終了間際に30秒の遅延があると、「自分が最高額で入札したと思ったら、実は別の人がその間に入札していた」という事態が起こり得ます。

### 問題 2: 無駄なリクエストが大量に発生する

ヘッダーのリクエスト回数を見てみてください。**誰も入札していなくても**、30秒ごとにリクエストが飛び続けています。

計算してみると分かりやすいかもしれません。

```
1ユーザーのポーリング: 2リクエスト/分（30秒間隔）

ユーザー数ごとの負荷:
  10人  →    20 リクエスト/分
  100人 →   200 リクエスト/分
  1000人 → 2,000 リクエスト/分
```

データに変化がない場合でも同じ量のリクエストが発生します。サーバーのリソースを無駄に消費している状態です。

### 問題 3: 間隔のジレンマ

ポーリング間隔には根本的なトレードオフがあります。

| 間隔を短くすると | 間隔を長くすると |
|:---|:---|
| リアルタイムに近づく | 遅延が大きくなる |
| サーバー負荷が増える | サーバー負荷は軽くなる |
| ネットワーク帯域を消費する | 帯域の消費は抑えられる |

どちらに寄せても何かを犠牲にします。「リアルタイムに近い体験」と「効率的なリソース使用」を両立する方法が、ポーリングにはありません。

---

## 処理の流れを比較する

ここまでの2つの方式を図で比較してみます。

### 手動リロード方式（前回）

```
タブA（入札）                  サーバー                タブB（閲覧中）
    |                           |                        |
    |-- POST（入札）----------->|                        |
    |<-- 200 OK----------------|                        |
    |                           |                        |
    |   タブAは更新される        |    タブBは何も知らない   |
    |                           |                        |
    |                           |    ユーザーがリロード    |
    |                           |<-- GET /api/items -----|
    |                           |--- 200 OK ------------>|
    |                           |     ようやく反映される   |
```

### ポーリング方式（今回）

```
タブA（入札）                  サーバー                タブB（閲覧中）
    |                           |                        |
    |                           |<-- GET（ポーリング）----|  ← 変化なし
    |                           |--- 200 OK ------------>|
    |                           |                        |
    |-- POST（入札）----------->|                        |
    |<-- 200 OK----------------|                        |
    |                           |                        |
    |                           |<-- GET（ポーリング）----|  ← 変化なし
    |                           |--- 200 OK ------------>|
    |                           |                        |
    |                           |<-- GET（ポーリング）----|  ← ここでやっと反映
    |                           |--- 200 OK ------------>|
```

ポーリング方式では、入札のタイミングとポーリングのタイミングにずれがあるため、反映までにラグが生じます。そして変化がないタイミングでも常にリクエストが飛んでいます。

---

## ポーリングが適切な場面

ポーリングには問題がありますが、すべての場面で不適切というわけではありません。

以下のような条件に当てはまる場合は、ポーリングでも十分かもしれません。

- [ ] リアルタイム性の要求が低い（数秒〜数十秒の遅延が許容される）
- [ ] 同時接続ユーザー数が少ない（サーバー負荷が問題にならない）
- [ ] データの更新頻度が低い（ほとんどのリクエストが「変化なし」でも許容できる）
- [ ] 実装のシンプルさを最優先したい（`setInterval` だけで完結する）
- [ ] WebSocket をサポートしないインフラ環境

天気予報の表示や、数分おきに更新するダッシュボードなどでは、ポーリングが合理的な選択肢になることもあります。

一方、オークションのように**「他の人のアクションを即座に反映したい」**場面では、ポーリングの限界が見えてきます。

---

## 次回: WebSocket でリアルタイム化する

次の記事では、ポーリングの `setInterval` を削除し、代わりに **WebSocket**（FluxSocket）を導入します。

サーバーからイベントをプッシュする方式に切り替えることで、「入札が全員に即座に届く」体験を実現します。無駄なリクエストもなくなります。

```
タブA（入札）           サーバー           FluxSocket           タブB（閲覧中）
    |                    |                   |                      |
    |-- POST（入札）---->|                   |                      |
    |                    |-- イベント発火 --->|                      |
    |                    |                   |--- WebSocket Push -->|
    |<-- WebSocket Push -|-------------------| 即座に反映される      |
```

ポーリングとの体験の違いを、ぜひデモで比較してみてください。

次回: [FluxSocketでリアルタイム化する — 入札が全員に即座に届く](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-3)

---

## 関連記事

- [前回: React + ExpressでオークションWebアプリを作る](https://zenn.dev/brainy_software/articles/fluxsocket-auction-tutorial-1)
- [FluxSocket で作る 8 つのリアルタイムアプリ — チュートリアル完全ガイド](https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-index)
- [WebSocket とは？HTTP 通信との違いと使いどころをわかりやすく解説](https://zenn.dev/brainy_software/articles/websocket-vs-http-realtime)
