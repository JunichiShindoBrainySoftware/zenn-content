---
title: "Claude Code のステータス行に「利用枠のリセット時刻」を映す — resets_at で“いつ回復するか”を常時表示する"
emoji: "⏳"
type: "tech"
topics: ["ClaudeCode", "Anthropic", "CLI", "生産性", "Tips"]
published: true
---

## はじめに

前回の記事「[Claude Code のステータス行に「プラン利用枠」を映す](https://zenn.dev/brainy_software/articles/claude-code-statusline-plan-usage)」では、Max / Pro のサブスクで `cost.total_cost_usd` が意味をなさない理由を整理し、代わりに **`rate_limits` でプラン利用枠の消費率**を常時表示する statusLine を作りました。

完成形はこんな表示でした。

```text
Opus 4.8  📁 my-project  ██░░░░░░░░ 21%  🪙 210.0K/1.00M  ⏳ 5h 23%  7d 6%
```

これで「今週どれくらい枠を使ったか」は分かるようになりました。ですが実際に使い続けていると、もう一つ欲しくなる情報があります。

> **「この枠、いつ回復するの？」**

`5h 88%` と出ていても、それが「あと10分で 0% に戻る」のか「あと4時間お預け」なのかで、打ち手はまったく変わります。前者なら少し休憩すればいいし、後者なら重い作業は別セッションに回す、といった判断ができる。**消費率だけでは「あとどれだけ我慢すればいいか」が読めない**のです。

そこで今回は、前回のスクリプトに **利用枠のリセット時刻**を追加して、こんな表示にします。

```text
Opus 4.8  📁 my-project  ██░░░░░░░░ 21%  🪙 210.0K/1.00M  ⏳ 5h 23% (~18:30)  7d 6% (~6/17 00:00)
```

- `5h 23% (~18:30)` … 5時間枠は **今日の 18:30** に回復
- `7d 6% (~6/17 00:00)` … 7日枠は **6/17 00:00** に回復

「いつ・どれだけ」がワンセットで見えるようになります。

:::message
本記事は前回の続編です。statusLine の基本セットアップ（`~/.claude/settings.json` と `~/.claude/statusline.js` の置き方）は前回記事を参照してください。ここでは差分だけを扱います。
:::

## カギは `resets_at` フィールド

リセット時刻は、自前で計算する必要はありません。Claude Code が statusLine に渡す JSON の `rate_limits` には、消費率（`used_percentage`）とセットで **`resets_at`** が入っています。

[公式ドキュメント](https://code.claude.com/docs/en/statusline) の記載は次のとおりです。

> `rate_limits.five_hour.resets_at`, `rate_limits.seven_day.resets_at`
> Unix epoch seconds when the 5-hour or 7-day rate limit window resets
> （5時間枠・7日枠がリセットされる Unix エポック秒）

つまり stdin の JSON はこうなっています。

```json
{
  "rate_limits": {
    "five_hour": {
      "used_percentage": 23,
      "resets_at": 1750066200
    },
    "seven_day": {
      "used_percentage": 6,
      "resets_at": 1750086000
    }
  }
}
```

`resets_at` は **Unix エポック秒**（1970年からの経過秒）。JavaScript なら `new Date(resets_at * 1000)` でそのまま `Date` に変換できます。あとはこれを読みやすい文字列に整形するだけです。

| フィールド | 内容 |
|---|---|
| `rate_limits.five_hour.used_percentage` | 5時間枠の消費率（0〜100） |
| `rate_limits.five_hour.resets_at` | 5時間枠がリセットされる Unix エポック秒 |
| `rate_limits.seven_day.used_percentage` | 7日枠の消費率（0〜100） |
| `rate_limits.seven_day.resets_at` | 7日枠がリセットされる Unix エポック秒 |

## 表示ルールを決める：当日は時刻だけ、またぐ日付は省略しない

リセット時刻を出すとき、毎回 `2026/6/14 18:30` のようにフル表示するとステータス行が長くなりすぎます。実用上ちょうどいいのは、**「今日なら時刻だけ、日付をまたぐなら月/日も付ける」**というルールです。

| リセットのタイミング | 表示 | 理由 |
|---|---|---|
| 今日のうちに回復 | `~18:30` | 日付は自明なので時刻だけで十分 |
| 明日以降に回復 | `~6/17 00:00` | 「いつの」00:00 か分からないと困る |

5時間枠は基本的に当日中（たまに日付をまたぐ）、7日枠はたいてい数日先になるので、この出し分けが自然に効きます。`~` は「だいたいこの時刻」というニュアンスの目印です。

## 実装：`resets_at` を整形する関数

前回のスクリプトに、リセット時刻を整形する `fmtReset` を足し、利用枠を組み立てる `rlPart` から呼び出します。追加・変更したのはこの部分だけです。

```js
const pad2 = (n) => String(n).padStart(2, "0");

// Unix エポック秒 → 「~HH:MM」または「~M/D HH:MM」
const fmtReset = (epoch) => {
  if (!epoch) return null;                 // resets_at が無ければ何も出さない
  const now = new Date();
  const r = new Date(epoch * 1000);        // 秒 → ミリ秒
  const hm = `${pad2(r.getHours())}:${pad2(r.getMinutes())}`;
  const sameDay =
    r.getFullYear() === now.getFullYear() &&
    r.getMonth() === now.getMonth() &&
    r.getDate() === now.getDate();
  // 当日中はリセット時刻のみ、日付をまたぐ場合は M/D を前置
  return sameDay ? `~${hm}` : `~${r.getMonth() + 1}/${r.getDate()} ${hm}`;
};

// 利用枠1つ分を組み立て（label = "5h" / "7d"）
const rlPart = (label, win) => {
  const v = win && win.used_percentage;
  if (v == null) return null;              // 消費率が無ければ枠ごと省略
  const reset = fmtReset(win.resets_at);
  const resetStr = reset ? " " + dim(`(${reset})`) : "";
  return `${dim(label)} ${paint(Math.round(v) + "%", rlColor(v))}${resetStr}`;
};
```

ポイントは3つです。

1. **`new Date(epoch * 1000)`** — `resets_at` は「秒」、JavaScript の `Date` は「ミリ秒」なので 1000 倍する
2. **`getHours()` 系はローカルタイムで返る** — Claude Code を動かしている端末のタイムゾーン（日本なら JST）でそのまま表示される。UTC 変換は不要で、むしろ手元の時計と一致するのが正解
3. **`resets_at` が無ければ時刻を出さない** — `used_percentage` はあるが `resets_at` だけ欠ける、という状態でも `(...)` 部分を省くだけで `5h 23%` は表示が崩れない

## 完成スクリプト

前回のスクリプトに上記を組み込んだ最終形です。`~/.claude/statusline.js` を丸ごと置き換えてください（`chmod +x` を忘れずに）。

```js
#!/usr/bin/env node
/**
 * Claude Code ステータス行（全プロジェクト共通）
 * stdin の JSON から モデル / ディレクトリ / コンテキスト使用率 / トークン /
 * プラン利用枠（5h・7d の消費率＋リセット時刻）を1行表示。
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
  // resets_at（Unix epoch 秒）から「いつ枠が回復するか」を併記する。
  const rl = d.rate_limits || {};
  const rlColor = (p) => (p >= 80 ? "31" : p >= 50 ? "33" : "32"); // 80%+赤 / 50%+黄 / 緑
  const pad2 = (n) => String(n).padStart(2, "0");
  const fmtReset = (epoch) => {
    if (!epoch) return null;
    const now = new Date();
    const r = new Date(epoch * 1000);
    const hm = `${pad2(r.getHours())}:${pad2(r.getMinutes())}`;
    const sameDay =
      r.getFullYear() === now.getFullYear() &&
      r.getMonth() === now.getMonth() &&
      r.getDate() === now.getDate();
    // 当日中はリセット時刻のみ、日付をまたぐ場合は M/D を前置
    return sameDay ? `~${hm}` : `~${r.getMonth() + 1}/${r.getDate()} ${hm}`;
  };
  const rlPart = (label, win) => {
    const v = win && win.used_percentage;
    if (v == null) return null;
    const reset = fmtReset(win.resets_at);
    const resetStr = reset ? " " + dim(`(${reset})`) : "";
    return `${dim(label)} ${paint(Math.round(v) + "%", rlColor(v))}${resetStr}`;
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

## 動作確認：モック入力で試す

statusLine スクリプトは **stdin から JSON を受け取って標準出力するだけ**なので、Claude Code を立ち上げなくてもターミナルから単体テストできます。`resets_at` は今の時刻を基準に作るのがポイントです。

```bash
# 5時間枠 = 今日の 18:30 / 7日枠 = 6/17 00:00 にリセットされる想定
FIVE=$(node -e 'const d=new Date(); d.setHours(18,30,0,0); console.log(Math.floor(d/1000))')
SEVEN=$(node -e 'console.log(Math.floor(new Date(2026,5,17,0,0,0)/1000))')

node ~/.claude/statusline.js <<EOF
{
  "model": { "display_name": "Opus 4.8" },
  "workspace": { "current_dir": "/path/to/my-project" },
  "context_window": { "context_window_size": 1000000, "total_input_tokens": 210000, "used_percentage": 21 },
  "rate_limits": {
    "five_hour": { "used_percentage": 23, "resets_at": $FIVE },
    "seven_day": { "used_percentage": 6,  "resets_at": $SEVEN }
  }
}
EOF
```

出力（端末では色が付きます）。

```text
Opus 4.8  📁 my-project  ██░░░░░░░░ 21%  🪙 210.0K/1.00M  ⏳ 5h 23% (~18:30)  7d 6% (~6/17 00:00)
```

念のため、欠落系のパターンも確かめておくと安心です。

| 入力 | 表示 | 期待どおりか |
|---|---|---|
| `resets_at` だけ無い | `… ⏳ 5h 23%  7d 6%` | ✅ 時刻だけ省略、消費率は残る |
| `rate_limits` ごと無い（セッション初期） | `… 🪙 210.0K/1.00M` | ✅ 利用枠ブロックごと省略 |
| 5h枠だけ・翌日 02:15 リセット | `… ⏳ 5h 88% (~6/15 02:15)` | ✅ 日付をまたぐと M/D を前置 |
| 空入力 / 壊れた JSON | `Claude  📁 ~  ░░░░░░░░░░ 0% …` | ✅ フォールバックで落ちない |

## 設計のポイントと落とし穴

- **`resets_at` は秒、`Date` はミリ秒**。`* 1000` を忘れると 1970年付近の日付になります（リセット時刻が常に大昔に見えたらこれ）
- **タイムゾーンはローカル**。`getHours()` 系は端末のタイムゾーンで返るので、手元の時計とそのまま一致します。UTC で出したい特殊事情がなければ変換は不要です
- **`resets_at` だけ欠けるケースに耐える**。`rate_limits` 自体は前回同様「あれば出す、無ければ省く」。さらに今回は **消費率はあるが `resets_at` が無い**状態でも `(...)` だけを落として崩れないようにしてあります
- **秒は出さない**。`HH:MM` までで十分で、秒まで出すと毎回チラついて目障りになります

## まとめ

- `rate_limits.five_hour.resets_at` / `seven_day.resets_at` に、利用枠が回復する時刻が **Unix エポック秒**で入っている
- `new Date(resets_at * 1000)` で `Date` に変換し、**当日は `~HH:MM`・日付をまたぐなら `~M/D HH:MM`** で出すと簡潔で読みやすい
- 消費率（いくら使ったか）に加えてリセット時刻（いつ戻るか）が並ぶと、**「今は重い作業を控える / もう少しで回復する」の判断が一目でできる**
- 欠落・初期状態・空入力でも崩れないよう、フォールバックは前回同様に効かせておく

前回の「いくら使ったか」に、今回の「いつ戻るか」が加わって、サブスクで Claude Code を回すときの“残量メーター”がほぼ完成形になりました。地味な一行ですが、作業ペースの配分がかなりやりやすくなります。
