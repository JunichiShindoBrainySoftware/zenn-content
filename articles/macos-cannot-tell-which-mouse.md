---
title: "macOS は「どのマウスのボタンか」を教えてくれない"
emoji: "🖱"
type: "tech"
topics: ["macos", "swift", "hid", "iokit", "coregraphics"]
published: false
---

マウスを2台使っている。デスクではトラックボール、持ち歩くときは小さいマウス。

同じ位置にあるボタンに、別々の動作を割り当てたかった。デスクではミッションコントロール、外では「戻る」。**それだけのつもりだった。**

できなかった。**macOS は「どのマウスのボタンか」を教えてくれない。**

## 1. まず、入力が通る道

マウスのボタンが押されてからアプリに届くまで、入力は層をいくつか通る。**どの層に立つかで、見えるものと、できることが変わる。**

```
  マウス（HID デバイス）
        │  USB / Bluetooth
        ▼
  IOKit / IOHIDFamily  ──────  IOHIDManager で覗ける
        │
        ▼
  Quartz Event Services ─────  CGEventTap で覗ける・止められる
        │              ▲
        ▼              │  作ったイベントを post して返す
   アプリケーション ───┘
```

言葉を先に置いておく。**定義として覚える必要はない。「どの層を扱う道具か」だけ分かれば足りる。**

- **HID** — Human Interface Device。マウス・キーボード・ゲームパッドが共通で使う規格。「ボタンが押された」「ホイールが回った」を決まった形で報告する
- **IOKit / IOHIDFamily** — macOS がデバイスを扱う土台。**どの機器から来たか**を知っているのはこの層
- **Quartz Event Services** — OS がアプリにイベントを配る仕組み。ここを流れるのは `CGEvent` という型で、**デバイスの情報はもう入っていない**
- **`IOHIDManager`** — IOKit の層を覗く道具。つながっている機器の一覧と、その入力が取れる
- **`CGEventTap`** — Quartz の層に割り込む道具。OS がアプリに配る**前**のイベント列に入って、書き換えたり握りつぶしたりできる

表にするとこうなる。**この表が、これから起きること全部の原因である。**

| 層 | どのマウスか分かる | 止められる | イベントを作って流せる |
|---|---|---|---|
| **IOKit**（`IOHIDManager`） | **分かる** | 占有すれば（後述） | ─ |
| **Quartz**（`CGEventTap`） | **分からない** | **できる** | **できる** |

**「誰が押したか分かる層」と「止められる層」が、別々にある。**繋ぐ公式な API は無い。

## 2. 上の層 ── 止められる。でも、誰が押したか分からない

マウスのボタンを横取りするなら `CGEventTap` を張る。定番で、情報も多い。

```swift
guard let tap = CGEvent.tapCreate(
    tap: .cgSessionEventTap,
    place: .headInsertEventTap,
    options: .defaultTap,          // consume できるタップ
    eventsOfInterest: mask,
    callback: { _, type, event, refcon in ... },
    userInfo: Unmanaged.passUnretained(self).toOpaque()
) else { return false }
```

コールバックに届いた `CGEvent` から、押されたボタンの番号が取れる。

```swift
let button = Int(event.getIntegerValueField(.mouseEventButtonNumber))
```

そして、**ここには「どのマウスか」が入っていない。**

1台しか繋いでいなければ困らない。だが「デスクのトラックボールの親指ボタンにはミッションコントロール、持ち歩いている小さいマウスの同じ位置には戻る」を割り当てたい瞬間に行き詰まる。**どちらも `buttonNumber = 3` として届く。**

`CGEvent` には他にもフィールドがあるが、ベンダー ID・プロダクト ID に相当するものは無い。1歩目でアプリの前提が崩れる。

### ついでに踏む穴 ── タップは黙って切られる

`CGEventTap` のコールバックは**同期で呼ばれる**。ここで重い処理を書くと、**macOS がタップを無効化する。**

無効化されるとき、コールバックには専用のイベント型が来る。**入れ直す実装を書いていないと、一度遅れただけで永久に効かなくなる。**

```swift
// タップが切られたら入れ直す。これをしないと、一度遅れただけで永久に効かなくなる。
if type == .tapDisabledByTimeout || type == .tapDisabledByUserInput {
    if let tap { CGEvent.tapEnable(tap: tap, enable: true) }
    onReenabled()
    return Unmanaged.passUnretained(event)
}
```

**`kCGEventTapDisabledByTimeout`** は「コールバックが遅すぎたので切った」、**`kCGEventTapDisabledByUserInput`** は「ユーザーの操作の都合で切った」を意味する。どちらも**再登録が必須**である。

さらに厄介なことに、**切れていた間のイベントはコールバックに届いていない。**押しっぱなしのボタンがあれば、アプリが持っている「いま押している最中」の状態と実際がずれる。上の `onReenabled()` は、その状態を丸ごと捨ててもらうための通知にしてある ── **届かなかった分を推測で埋めない。**

## 3. 下の層 ── 誰か分かる。でも、止められない

デバイスが分かる層に降りる。`IOHIDManager` を使う。

```swift
let manager = IOHIDManagerCreate(kCFAllocatorDefault, IOOptionBits(kIOHIDOptionsTypeNone))

let matches: [[String: Any]] = [
    [kIOHIDDeviceUsagePageKey: kHIDPage_GenericDesktop, kIOHIDDeviceUsageKey: kHIDUsage_GD_Mouse],
    [kIOHIDDeviceUsagePageKey: kHIDPage_GenericDesktop, kIOHIDDeviceUsageKey: kHIDUsage_GD_Pointer],
]
IOHIDManagerSetDeviceMatchingMultiple(manager, matches as CFArray)
```

こちらは**どの機器のどのボタンか**が分かる。ベンダー ID とプロダクト ID が取れるので、`046D:B01D` のような識別子が作れる。

ただし、**イベントを止められない。**

止めるには **seize**（占有）が要る。`IOHIDManagerOpen` に `kIOHIDOptionsTypeSeizeDevice` を渡すと、そのデバイスの入力を独占できる。

**これは採らなかった。**占有したデバイスの入力は OS 側に流れなくなる、と理解している（**こちらで実際に試したわけではない**）。そうであれば、カーソルの移動もクリックも自分で再現しなければならない。**再現し損ねた瞬間に、利用者のマウスが死ぬ。**

「落ちてもマウスは素のマウスとして動く」を製品の約束にしている以上、マウスの生殺与奪を握る設計は選べなかった。だから `kIOHIDOptionsTypeNone`、**監視だけ**にしてある。

```swift
/// **seize しない**（`kIOHIDOptionsTypeNone`）。奪うと他のアプリからマウスが使えなくなる。
/// 監視するだけなので、入力監視の許可があれば足りる。
```

### もう1つの道 ── DriverKit

正攻法はもう1つある。**DriverKit のシステム拡張**を書けば、デバイスの層で正式に入力を扱える。

これも採らなかった。利用者に**システム拡張の許可**と**再起動**を求めることになり、配布のハードルが跳ね上がる。**入力監視とアクセシビリティの2つだけで動く**ことを要件にしていたので、合わなかった（同じ理由で root デーモンも持っていない）。

結果として、**残った手は「監視はできるが止められない層」と「止められるが誰か分からない層」を、自分で繋ぐこと**になった。

ここで手詰まりになる。**片方は「何が押されたか」しか知らず、もう片方は「誰が押したか」しか知らない。**

## 4. 縫う ── 時刻で突き合わせる

繋ぐ API が無いなら、自分で縫うしかない。**両方を並行して監視して、同じ押下を照合する。**

成立するかどうかは、**どちらが先に届くか**で決まる。

- **HID が先なら**：`CGEventTap` のコールバックが動く時点で、出所はもう分かっている。その場で判定できる
- **`CGEventTap` が先なら**：出所が分かるまで待つことになる。だがコールバックは同期なので、そこで待てば入力が固まる。**設計が破綻する**

推測で決めずに測った。両方にログを入れて、実機で親指ボタンを押す。

```
HID  down MX Ergo [046D:B01D] usage=5 → cg=4
TAP  down b=4
```

**HID が先だった**（2026-07-27・MX Ergo / BLE 接続・macOS 26）。順序が逆なら、この方式ごと捨てる必要があった。

実装は素直になる。HID 側は直前の1件を上書きで覚える。

```swift
private let toleranceSec: CFAbsoluteTime = 0.05

func deviceKey(forButton button: Int) -> String? {
    guard let recent,
          recent.button == button,
          CFAbsoluteTimeGetCurrent() - recent.at < toleranceSec
    else { return nil }
    return recent.deviceKey
}
```

**50ms** は実測から出した値ではなく、2つの条件を満たす範囲から選んだ値である ── 人間が別々のボタンを押し分ける間隔よりずっと短く、機械（BLE を含む）の伝搬遅延よりは長い。実機の差はミリ秒未満だったので、桁で余裕がある。

そして、**合わなければ諦める。**

```swift
/// 分からなければ nil。呼び手は全マウス共通の設定に落とす（**取り違えるより素直に諦める**）。
```

「誰のボタンか分からない」だけで、「ボタンが押された」ことは分かっている。**間違ったマウスの割当を当てるより、共通の設定に落とすほうが害が小さい。**

### 同じボタンに、番号が2つある

上のログをもう一度見てほしい。`usage=5 → cg=4` と書いてある。

**HID の usage は 1 始まり、`CGEvent` の `buttonNumber` は 0 始まり。**同じボタンに2つの番号が付いていて、常に1ずれる。

| | 左 | 右 | 中 | サイド |
|---|---|---|---|---|
| HID の **usage** | 1 | 2 | 3 | 4, 5 |
| `CGEvent` の `buttonNumber` | **0** | **1** | **2** | **3, 4** |

**usage** は HID の用語で、「この値は何を意味するか」を表す番号である。ボタンなら「何番目のボタンか」を指す。

```swift
struct ButtonEvent {
    /// HID の usage。1=左 2=右 3=中 …
    let usage: Int
    /// CGEvent の buttonNumber に合わせた番号（usage - 1）
    var cgButton: Int { usage - 1 }
}
```

これは**気づきにくい壊れ方**をする。全部が1つずれるので、「右を押したのに中ボタンの割当が出る」という形で現れる。動いてはいるので、パッと見は設定ミスに見える。**変換は入り口で1回だけ行う。**あちこちで `-1` すると、必ずどこかで2回引く。

## まとめ

これで「**どのマウスの、どのボタンが押されたか**」が分かるようになった。

- **「誰が押したか分かる層」と「止められる層」は別。**繋ぐ API は無いので、自分で縫う
- **どちらが先に届くかを、最初に測る。**順序が逆なら設計ごと成立しない
- **同じボタンに番号が2つある。**HID の usage は 1 始まり、`CGEvent` は 0 始まり
- **タップは黙って切られる。**再登録は必須
- **分からないときは諦める。**取り違えるより害が小さい

残っているのは、受け取ったあとの話 ── **代わりのイベントを作って OS に返す**ところである。`CGEvent` を作って post するだけ、のはずだった。**そこにも穴があった。**

（後編に続きます）

---

### 用語

| 語 | 意味 |
|---|---|
| **HID** | Human Interface Device。入力機器が共通で使う規格 |
| **IOKit / IOHIDFamily** | macOS がデバイスを扱う土台。どの機器かを知っている層 |
| **Quartz Event Services** | OS がアプリにイベントを配る仕組み。デバイス情報は入っていない |
| **`IOHIDManager`** | IOKit の層を覗く API。機器の一覧と入力が取れる |
| **`CGEventTap`** | Quartz の層に割り込む API。書き換え・握りつぶしができる |
| **`kCGEventTapDisabledByTimeout`** | コールバックが遅く、OS がタップを切ったときに届く型。再登録が要る |
| **seize** | デバイスを占有すること。入力を独占できるが、OS 側に流れなくなる |
| **usage / Usage Page** | HID で「この値が何を意味するか」を表す番号と、その分類 |

---

この突き合わせは、メーカーが違うマウスを1つのアプリで設定するために書いたものです。作ったものは **[MouseForge](https://mouseforge.jp)** という常駐アプリで、macOS 26 で動いています。
