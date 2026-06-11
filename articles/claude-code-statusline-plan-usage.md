---
title: "Claude Code のステータス行に「プラン利用枠」を映す — Max/Pro でコスト表示が意味をなさない問題と解決"
emoji: "📊"
type: "tech"
topics: ["ClaudeCode", "Anthropic", "CLI", "生産性", "Tips"]
published: false
---

## はじめに

Claude Code を使っていて、こんなことを思ったことはないでしょうか。

- 「いまどれくらいコンテキストを使っている？ そろそろ自動圧縮（auto-compact）されそう？」
- 「Max プランの利用枠、今週どれくらい消費した？」

Claude Code には **statusLine**（画面下のステータス行）をカスタマイズする機能があり、これらを**常時表示**できます。

この記事では、その基本セットアップから、**サブスク（Max / Pro）ユーザーがハマりがちな「コスト表示が意味をなさない」問題と、その解決策**までを解説します。普段から Claude Code を実務で使う中で「これは出しておきたい」と感じた構成です。

:::message
本記事は Claude Code v2.1 系時点の仕様に基づきます。statusLine 自体は v1.0.71 以降で使えますが、後述の `context_window` / `rate_limits` フィールドは比較的新しいバージョンで利用できます。
:::

## statusLine とは

ステータス行は、Claude Code の画面下に常時表示されるカスタマイズ可能なバーです。設定した任意のスクリプトを実行し、その**標準出力をそのまま表示**します。スクリプトには**セッション情報が JSON で標準入力（stdin）から渡される**ので、それを整形して1行を出力すればOKです。

設定は `~/.claude/settings.json` に書きます。`~/.claude/` 配下に置けば**すべてのプロジェクトで共通**に効きます（プロジェクト個別の `.claude/settings.json` もありますが、今回は全体に効かせます）。

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.js"
  }
}
```

あとは `~/.claude/statusline.js` を用意し、実行権限を付けるだけです。

```bash
chmod +x ~/.claude/statusline.js
```

今回は Node.js で書きますが、bash + jq でも Python でも構いません。

## 渡ってくる JSON を覗いてみる

スクリプトの stdin には、こんな JSON が渡ってきます（抜粋）。

```json
{
  "model": { "display_name": "Opus 4.8" },
  "workspace": { "current_dir": "/path/to/project" },
  "context_window": {
    "context_window_size": 1000000,
    "total_input_tokens": 210000,
    "used_percentage": 21,
    "remaining_percentage": 79
  },
  "cost": { "total_cost_usd": 1.23 },
  "rate_limits": {
    "five_hour": { "used_percentage": 23 },
    "seven_day": { "used_percentage": 6 }
  }
}
```

主に使うフィールドは次のとおりです。

| フィールド | 内容 |
|---|---|
| `model.display_name` | モデル表示名（例: Opus 4.8） |
| `workspace.current_dir` | カレントディレクトリ |
| `context_window.context_window_size` | コンテキスト上限（既定 200,000 / 拡張モデルは 1,000,000） |
| `context_window.used_percentage` | コンテキスト使用率（事前計算済み・入力トークン基準） |
| `context_window.total_input_tokens` | 現在のコンテキストの入力トークン数 |
| `cost.total_cost_usd` | セッションコスト見積り（**後述の落とし穴あり**） |
| `rate_limits.five_hour` / `seven_day` | **プラン利用枠の消費率**（サブスクのみ） |

:::message alert
`context_window.used_percentage` などは、**セッション最初の API 応答前**や **`/compact` 直後**は `null` になります。スクリプト側でフォールバックしておかないと表示が崩れるので注意してください。
:::

## まずはコンテキスト残量を表示する

ここまでで、いちばん欲しい「コンテキスト使用率」が出せます。色分けして、auto-compact が近づいたら警告色にしましょう（70% で黄、90% で赤）。

```js
#!/usr/bin/env node
let raw = "";
process.stdin.on("data", (c) => (raw += c));
process.stdin.on("end", () => {
  let d = {};
  try { d = JSON.parse(raw); } catch (_) {}

  const model = (d.model && d.model.display_name) || "Claude";
  const cw = d.context_window || {};
  const pct = Math.round(cw.used_percentage || 0);
  const color = pct >= 90 ? 31 : pct >= 70 ? 33 : 32; // 赤 / 黄 / 緑
  const bar = "█".repeat(Math.round(pct / 10)).padEnd(10, "░");

  process.stdout.write(`\x1b[36m${model}\x1b[0m  \x1b[${color}m${bar} ${pct}%\x1b[0m`);
});
```

これで `Opus 4.8  ██░░░░░░░░ 21%` のような行が出ます。Opus の拡張コンテキスト（1M）なら、200K ではなく **1M に対する使用率**で表示されます。

## 落とし穴：サブスクでは「コスト表示」が意味をなさない

多くの statusLine 例では `cost.total_cost_usd` を「$1.23」のように表示します。ですが、公式ドキュメントにはこう書かれています。

> Estimated session cost in USD, computed client-side. May differ from your actual bill.
> （USD 建てのセッションコスト見積り。クライアント側で算出。実際の請求額とは異なる場合がある）

つまりこの値は **API の従量課金単価（トークン単価）で計算した理論値**であり、**契約プランは見ていません**。

| 契約形態 | `$` 表示の意味 |
|---|---|
| **サブスク（Max / Pro / Team）** | 月額定額なので、表示は「もし API 従量だったら」の架空の金額。実際の支払いとは無関係 |
| **API 従量課金（Console）** | 実コストの近似（キャッシュや丸めでズレることあり） |

私たちは Max プランで Claude Code を回しているのですが、**定額なので `$` がいくら増えても支払いは変わりません**。気にすべきは「いくら使ったか（金額）」ではなく、「**プランの利用枠をどれだけ消費したか**」です。

## 解決：プラン利用枠（rate_limits）を表示する

statusLine の JSON には、**Claude.ai サブスク（Max / Pro）のときだけ**現れるフィールドがあります。

- `rate_limits.five_hour.used_percentage` … 直近 **5 時間**の利用枠の消費率
- `rate_limits.seven_day.used_percentage` … 直近 **7 日間**の利用枠の消費率

これは Claude Code の `/status` で見られる利用制限画面と対応しています。

| `/status` の項目 | JSON フィールド |
|---|---|
| 現在のセッション（5時間枠） | `rate_limits.five_hour.used_percentage` |
| 週間制限・すべてのモデル（7日枠） | `rate_limits.seven_day.used_percentage` |

たとえば Max (20x) で「現在のセッション 23% / 週間 6%」のとき、`⏳ 5h 23%  7d 6%` と表示できます。これなら**契約プランの実消費**がひと目で分かります。

:::message
`rate_limits` は **セッション最初の API 応答後**に現れます（それまでは存在しません）。また各枠（`five_hour` / `seven_day`）は独立して欠けることがあります。スクリプトは「あれば出す、無ければ省く」で書いておくと安全です。
:::

## 完成スクリプト

コンテキスト残量に加えて、プラン利用枠まで表示する最終形がこちらです。`~/.claude/statusline.js` に保存して `chmod +x` してください。

```js
#!/usr/bin/env node
/**
 * Claude Code ステータス行（全プロジェクト共通）
 * stdin の JSON から モデル / ディレクトリ / コンテキスト使用率 / トークン / プラン利用枠 を1行表示。
 */
"use strict";

let raw = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => (raw += chunk));
process.stdin.on("end", () => {
  let d = {};
  try { d = JSON.parse(raw); } catch (_) { /* 空/不正でも落とさない */ }

  const ESC = "\x1b[";
  const paint = (s, code) => `${ESC}${code}m${s}${ESC}0m`;
  const dim = (s) => paint(s, "2");
  const accent = (s) => paint(s, "38;5;39");

  // モデル / ディレクトリ
  const model = (d.model && d.model.display_name) || "Claude";
  const cwd = (d.workspace && d.workspace.current_dir) || d.cwd || "";
  const dir = cwd.split("/").filter(Boolean).pop() || "~";

  // コンテキスト使用率（used_percentage は input ベースの事前計算値）
  const cw = d.context_window || {};
  const size = cw.context_window_size || 200000;
  const inputTokens = cw.total_input_tokens || 0;
  let pct = cw.used_percentage;
  if (pct == null) pct = size ? (inputTokens / size) * 100 : 0;
  pct = Math.max(0, Math.min(100, Math.round(pct)));

  // コンテキスト色: 90%+ 赤 / 70%+ 黄 / それ未満 緑
  const code = pct >= 90 ? "31" : pct >= 70 ? "33" : "32";
  const filled = Math.round(pct / 10);
  const bar = paint("█".repeat(filled), code) + dim("░".repeat(Math.max(0, 10 - filled)));

  // トークンを K/M 整形
  const fmt = (n) =>
    n >= 1e6 ? (n / 1e6).toFixed(2) + "M" : n >= 1e3 ? (n / 1e3).toFixed(1) + "K" : String(n);

  // プラン利用枠（サブスク = Max/Pro のとき rate_limits が届く。最初の API 応答後に出現）
  //   five_hour … /status の「現在のセッション(5時間枠)」
  //   seven_day … /status の「週間制限・すべてのモデル(7日枠)」
  const rl = d.rate_limits || {};
  const rlColor = (p) => (p >= 80 ? "31" : p >= 50 ? "33" : "32"); // 80%+赤 / 50%+黄 / 緑
  const rlPart = (label, win) => {
    const v = win && win.used_percentage;
    return v == null ? null : `${dim(label)} ${paint(Math.round(v) + "%", rlColor(v))}`;
  };
  const planParts = [rlPart("5h", rl.five_hour), rlPart("7d", rl.seven_day)].filter(Boolean);
  const planStr = planParts.length ? "  " + dim("⏳") + " " + planParts.join("  ") : "";

  const line =
    `${accent(model)}  ${dim("📁")} ${dir}` +
    `  ${bar} ${paint(pct + "%", code)}` +
    `  ${dim("🪙")} ${fmt(inputTokens)}/${fmt(size)}${planStr}`;

  process.stdout.write(line);
});
```

完成形の表示はこんな雰囲気です（端末では色が付きます）。

```text
Opus 4.8  📁 my-project  ██░░░░░░░░ 21%  🪙 210.0K/1.00M  ⏳ 5h 23%  7d 6%
```

設計のポイント:

- `used_percentage` が `null` のときは `total_input_tokens / context_window_size` で手計算フォールバック
- コンテキスト使用率は **70% 黄 / 90% 赤**、プラン利用枠は **50% 黄 / 80% 赤**で色分け
- `rate_limits` が無いセッション初期は枠表示を省略（誤った値を出さない）

## すべてのプロジェクトで効く

`~/.claude/settings.json` と `~/.claude/statusline.js`（絶対パス参照）に置いたので、**どのプロジェクトでセッションを開いても同じステータス行**が出ます。しかも表示内容は毎回 stdin で渡される情報を元に描画されるため、「いま開いているプロジェクト」の情報が動的に反映されます。**設定は1か所、表示は各プロジェクトに追従**します。

なお設定は**起動時に読み込まれる**ため、反映は次に開く新しいセッションからになります。

## まとめ

- Claude Code の statusLine で、コンテキスト残量とプラン利用枠を常時可視化できる
- `cost.total_cost_usd` は **API 従量単価ベースの見積り**で、**サブスクでは意味をなさない**
- サブスク（Max / Pro）なら `rate_limits.five_hour` / `seven_day` を出すのが契約に即した指標
- スクリプトと設定を `~/.claude/` に置けば、**全プロジェクト共通**で効く

地味ですが、毎セッションの安心感がかなり変わるカスタマイズです。同じようにサブスクで Claude Code を使っている方の参考になれば幸いです。
