---
title: "リアルタイム通知をWebアプリに組み込む方法まとめ"
emoji: "🔔"
type: "tech"
topics: ["websocket", "リアルタイム通信", "web開発", "通知"]
published: false
---

「新しいメッセージが届きました」「注文のステータスが更新されました」— こういったリアルタイム通知は、今やWebアプリのユーザー体験に欠かせない要素です。

でも実装方法は一つではありません。この記事では、リアルタイム通知を実現する主要な4つの方法を比較し、プロジェクトに最適な手段を選ぶための判断基準を整理します。

## 方法1: ポーリング（Polling）

最もシンプルな方法。JavaScriptの`setInterval`でサーバーに定期的に問い合わせます。

```javascript
// 5秒ごとに新しい通知がないか確認
setInterval(async () => {
  const res = await fetch('/api/notifications/unread');
  const data = await res.json();
  if (data.length > 0) {
    showNotification(data);
  }
}, 5000);
```

### メリット
- 実装が非常に簡単。HTTPのGETリクエストだけ
- サーバー側に特別な仕組みが不要
- どんなホスティング環境でも動く

### デメリット
- **リアルタイム性が低い。** 5秒間隔なら最大5秒の遅延
- **無駄なリクエストが多い。** 通知がなくても毎回リクエストが発生
- **ユーザー数 × ポーリング間隔 = サーバー負荷。** スケールしにくい

### 向いているケース
- プロトタイプや社内ツールなど、ユーザー数が少ない場面
- 数秒の遅延が許容されるケース
- とにかく早く実装したい場面

## 方法2: ロングポーリング（Long Polling）

ポーリングの改良版。サーバーが新しいデータを持つまでレスポンスを返さず、接続を保持します。

```javascript
async function longPoll() {
  try {
    const res = await fetch('/api/notifications/wait');
    const data = await res.json();
    showNotification(data);
  } catch (e) {
    // エラー時は少し待ってからリトライ
    await new Promise(r => setTimeout(r, 3000));
  }
  // 次のロングポーリングを開始
  longPoll();
}

longPoll();
```

### メリット
- ポーリングより即時性が高い
- 無駄なリクエストが減る
- HTTP環境で動くので、WebSocketが使えない環境でも可

### デメリット
- **サーバー側の接続管理が複雑。** タイムアウトの設定が必要
- **接続の張り直しが頻繁。** レスポンスを返すたびに再接続
- **HTTP接続を長時間占有。** プロキシやロードバランサーのタイムアウトに引っかかりやすい

### 向いているケース
- WebSocketが使えない環境（古いプロキシ、ファイアウォール制限）
- ポーリングよりはリアルタイムにしたいが、WebSocketの導入は重い場面

## 方法3: Server-Sent Events（SSE）

サーバーからクライアントへの**一方向ストリーミング**。HTTPの仕組みの上で動くので導入しやすいです。

```javascript
const source = new EventSource('/api/notifications/stream');

source.onmessage = (event) => {
  const data = JSON.parse(event.data);
  showNotification(data);
};

source.onerror = () => {
  // ブラウザが自動的に再接続を試みる
  console.log('接続が切れました。再接続中...');
};
```

### メリット
- **自動再接続が組み込み。** ブラウザが勝手にリトライしてくれる
- HTTPの上で動くので、既存のインフラ設定がそのまま使える
- 実装がWebSocketより簡単

### デメリット
- **サーバー→クライアントの一方向のみ。** クライアントからの送信はできない
- **同時接続数に制限。** HTTP/1.1ではブラウザあたり6接続まで
- **バイナリデータの送信が不可。** テキストのみ

### 向いているケース
- サーバーからの一方向通知で十分な場面（ニュースフィード、株価更新）
- 双方向通信が不要なダッシュボードの自動更新
- WebSocketの導入が難しいが、ポーリングよりは改善したい場面

## 方法4: WebSocket（+ SaaS）

双方向のリアルタイム通信。クライアントとサーバーが自由にデータをやりとりできます。

```javascript
// WebSocket SaaSを使った例（Pusher互換）
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Echo = new Echo({
  broadcaster: 'pusher',
  key: 'your-app-key',
  wsHost: 'your-fluxsocket-host',
  wsPort: 443,
  forceTLS: true,
});

// プライベートチャンネルで通知を受信
window.Echo.private(`user.${userId}`)
  .listen('NewNotification', (event) => {
    showNotification(event.notification);
  });
```

### メリット
- **双方向リアルタイム。** 送信も受信も即座
- **低レイテンシ。** ミリ秒単位でデータが届く
- **効率的。** 1本の接続で複数のチャンネルを扱える
- **SaaSを使えば運用の負荷がゼロ**

### デメリット
- **自前運用は複雑。** スケーリング、認証、再接続の実装が必要
- **インフラの設定が必要。** WebSocket対応のロードバランサー、プロキシ
- **SaaS利用のコスト。** 接続数に応じた月額費用

### 向いているケース
- チャット、共同編集、オンラインゲームなど双方向通信が必要
- 大量のユーザーに即座に通知を届けたい
- プロダクションレベルの信頼性が求められる

## 4つの方法を比較

| 項目 | ポーリング | ロングポーリング | SSE | WebSocket |
|------|-----------|----------------|-----|-----------|
| **即時性** | △（間隔依存） | ○ | ○ | ◎ |
| **通信方向** | 一方向 | 一方向 | 一方向 | **双方向** |
| **実装の簡単さ** | ◎ | ○ | ○ | △（自前）/ ◎（SaaS） |
| **サーバー負荷** | × | △ | ○ | ○ |
| **スケーラビリティ** | × | △ | ○ | ◎（SaaS） |
| **再接続** | 不要 | 手動実装 | 自動 | 手動実装 / SaaS側で対応 |

## どう選ぶべきか

**判断フローチャート:**

1. **双方向通信が必要？** → Yes → **WebSocket**
2. **リアルタイム性はどのくらい必要？**
   - 数秒の遅延でOK → **ポーリング**
   - できるだけ即時 → 3へ
3. **サーバーからの一方向でOK？**
   - Yes → **SSE**
   - No → **WebSocket**
4. **ユーザー数が多い？運用コストを抑えたい？**
   - → **WebSocket SaaS** の利用を検討

多くのモダンWebアプリでは、最終的にWebSocketに行き着きます。ただし、自前で運用するのは大変なので、PusherやAbly、あるいは私たちが開発している [FluxSocket](https://fluxsocket.com) のようなWebSocket SaaSを利用するのが現実的な選択肢です。

FluxSocketはPusher互換APIで、日本語ドキュメント完備・日本円決済に対応しています。

:::message
FluxSocketは現在ベータユーザーを募集しています。無料のHobbyプランで気軽にお試しいただけます。
👉 [FluxSocket公式サイト](https://fluxsocket.com)
:::

## 関連記事

- [WebSocketとは？HTTP通信との違いと使いどころをわかりやすく解説](https://zenn.dev/brainy_software/articles/websocket-vs-http-realtime)
- [WebSocket SaaSを選ぶときに見るべきポイント](https://zenn.dev/brainy_software/articles/websocket-saas-selection-guide)
