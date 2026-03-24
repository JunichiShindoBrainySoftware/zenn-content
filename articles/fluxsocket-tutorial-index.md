---
title: "FluxSocketで作る8つのリアルタイムアプリ — チュートリアル完全ガイド"
emoji: "⚡"
type: "tech"
topics: ["FluxSocket", "WebSocket", "リアルタイム通信", "チュートリアル"]
published: false
---

## このシリーズについて

**FluxSocket** を使って、**8 つの異なる言語・フレームワーク**でリアルタイムアプリを構築するチュートリアルシリーズです。

チャット、オークション、通知、タスクボード、ダッシュボード……実務でよく求められるリアルタイム機能を、実際に手を動かしながら学べる構成になっています。各チュートリアルは独立しているので、興味のあるものから始められます。

**デモサイト:** [https://demo.fluxsocket.com](https://demo.fluxsocket.com)
全デモを実際に触って動作を確認できます。

### 対象読者

- リアルタイム機能を Web アプリに追加したい開発者
- WebSocket を使ってみたいが、インフラ構築は避けたい方
- Pusher を使ったことがあり、代替サービスを探している方

---

## まずはこちら: 準備編

https://zenn.dev/brainy_software/articles/fluxsocket-tutorial-setup

すべてのチュートリアルの前提となる **共通セットアップ** です。FluxSocket のアカウント作成、アプリキー取得、クライアント・サーバー双方の基本的な接続方法を解説しています。

**所要時間: 約 5 分**

まだ FluxSocket のアカウントをお持ちでない方は、こちらから始めてください。

---

## チュートリアル一覧

### 1. Laravel でリアルタイムチャットを作る（全 3 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | Laravel 12 + PHP + Alpine.js |
| **FluxSocket 機能** | Public Channel, Server Trigger |
| **難易度** | ⭐ 入門 |

Laravel の Broadcasting 機能と FluxSocket を組み合わせて、リアルタイムチャットアプリを段階的に構築します。

| # | タイトル | 状態 |
|---|---------|------|
| ① | 環境構築 + 最小チャット | 近日公開 |
| ② | FluxSocket 連携 + リアルタイム化 | 近日公開 |
| ③ | 認証・UI 仕上げ + デプロイ | 近日公開 |

---

### 2. React でリアルタイムオークションを作る（全 2 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | React 18 + Express |
| **FluxSocket 機能** | Private Channel, Presence Channel, Server Trigger |
| **難易度** | ⭐⭐ 中級 |

入札のリアルタイム更新と、「今誰が見ているか」の Presence 表示を実装します。認証付きチャネルの使い方を学ぶのに最適です。

| # | タイトル | 状態 |
|---|---------|------|
| ① | 商品一覧 + 入札 API | 近日公開 |
| ② | Presence + UI 仕上げ | 近日公開 |

---

### 3. Node.js でユーザー別リアルタイム通知を実装する（全 2 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | Express + Alpine.js |
| **FluxSocket 機能** | Private Channel, Server Trigger |
| **難易度** | ⭐⭐ 中級 |

ユーザーごとに異なる通知をリアルタイムで配信する仕組みを、Private Channel の認証フローとともに解説します。

| # | タイトル | 状態 |
|---|---------|------|
| ① | Express API + ユーザー画面 | 近日公開 |
| ② | Private Channel 認証 + リアルタイム通知 | 近日公開 |

---

### 4. Ruby でコラボタスクボードを作る（全 2 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | Sinatra + Alpine.js |
| **FluxSocket 機能** | Presence Channel, Client Events, Server Trigger |
| **難易度** | ⭐⭐⭐ 上級 |

複数ユーザーが同時にタスクを操作できるコラボレーションボードを構築します。Client Events を使ったサーバーを経由しないリアルタイム通信を学べます。

| # | タイトル | 状態 |
|---|---------|------|
| ① | Sinatra + タスク CRUD | 近日公開 |
| ② | Presence + Client Events + 共同編集 | 近日公開 |

---

### 5. Vue.js で SNS ライブフィードを作る（全 1 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | Vue 3 + Express |
| **FluxSocket 機能** | Public Channel, Presence Channel, Server Trigger |
| **難易度** | ⭐ 入門 |

投稿がリアルタイムでフィードに流れる SNS 風アプリを Vue 3 の Composition API で構築します。

📝 **近日公開**

---

### 6. Python でリアルタイムダッシュボードを作る（全 1 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | Flask + Chart.js |
| **FluxSocket 機能** | Public Channel, Server Trigger |
| **難易度** | ⭐ 入門 |

サーバーから定期的にデータをプッシュし、グラフがリアルタイムで更新されるダッシュボードを作ります。IoT やモニタリング用途にも応用できます。

📝 **近日公開**

---

### 7. Vanilla JS で Figma 風カーソル共有を作る（全 1 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | Vanilla JS + Canvas + Express |
| **FluxSocket 機能** | Presence Channel, Client Events |
| **難易度** | ⭐⭐⭐ 上級 |

フレームワークを使わず、素の JavaScript と Canvas で他のユーザーのカーソル位置をリアルタイム共有します。Client Events の低レイテンシ通信を体感できるサンプルです。

📝 **近日公開**

---

### 8. Pure PHP でライブ投票アプリを作る（全 1 回）

| 項目 | 内容 |
|------|------|
| **技術スタック** | Pure PHP（フレームワーク不要） |
| **FluxSocket 機能** | Public Channel, Server Trigger |
| **難易度** | ⭐ 入門 |

フレームワークなしの素の PHP だけで、投票結果がリアルタイムに反映されるライブ投票アプリを構築します。FluxSocket の導入がいかにシンプルかを実感できるチュートリアルです。

📝 **近日公開**

---

## FluxSocket 機能カバレッジ

各チュートリアルがカバーする FluxSocket の機能を一覧にまとめました。学びたい機能から逆引きでチュートリアルを選ぶこともできます。

| 機能 | Chat | Auction | Notify | Taskboard | Livefeed | Dashboard | Cursors | Voting |
|------|:----:|:-------:|:------:|:---------:|:--------:|:---------:|:-------:|:------:|
| Public Channel | ✅ | | | | ✅ | ✅ | | ✅ |
| Private Channel | | ✅ | ✅ | | | | | |
| Presence Channel | | ✅ | | ✅ | ✅ | | ✅ | |
| Client Events | | | | ✅ | | | ✅ | |
| Server Trigger | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | | ✅ |

- **Public Channel** — 認証不要。誰でも購読できるチャネル
- **Private Channel** — サーバー側で認証が必要なチャネル。ユーザー固有のデータ配信に
- **Presence Channel** — Private Channel + 「誰がオンラインか」を把握できる
- **Client Events** — サーバーを経由せず、クライアント同士で直接イベントを送受信
- **Server Trigger** — サーバーからチャネルにイベントを発火（もっとも基本的な使い方）

---

## FluxSocket について

FluxSocket は、日本発の **Pusher 互換リアルタイム通信 SaaS** です。Pusher のクライアントライブラリやサーバー SDK がそのまま使えるため、既存の知識やコードをそのまま活かせます。

- **公式サイト:** [https://fluxsocket.com](https://fluxsocket.com)
- **ドキュメント:** [https://fluxsocket.com/docs](https://fluxsocket.com/docs)

現在ベータユーザーを募集中です。無料プランもありますので、このチュートリアルシリーズを機にぜひお試しください。
