---
title: "macOS は「どのマウスのボタンか」を教えてくれない"
emoji: "🖱"
type: "tech"
topics: ["macos", "swift", "hid", "iokit", "coregraphics"]
published: true
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
- **Quartz Event Services** — Core Graphics のイベントシステム。`CGEvent` という型がここを流れ、`CGEventTap` はこの層に割り込む。**公開されているフィールドの範囲では、どの HID デバイスから来たかが分からない**
- **`IOHIDManager`** — IOKit の層を覗く道具。つながっている機器の一覧と、その入力が取れる
- **`CGEventTap`** — Quartz の層に割り込む道具。OS がアプリに配る**前**のイベント列に入って、書き換えたり握りつぶしたりできる

表にするとこうなる。**この表が、これから起きること全部の原因である。**

| 層 | どのマウスか分かる | 止められる | イベントを作って流せる |
|---|---|---|---|
| **IOKit**（`IOHIDManager`） | **分かる** | 占有すれば（後述） | ─ |
| **Quartz**（`CGEventTap`） | **分からない** | **できる** | **できる** |

**「誰が押したか分かる層」と「止められる層」が、別々にある。**繋ぐ公式な API は無い。

なお、同じ「止める」でも**位置が違う**。seize は**イベント列に載る前**にデバイスごと押さえ、`CGEventTap` は**載ったあとに列から抜く**。後述するが、この差が選択を決めることになる。

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

「何も入っていない」わけではない。発生源に関するフィールドはいくつもあって、**イベントを作ったプロセス**（`kCGEventSourceUnixProcessID`）は取れるし、`kCGMouseEventSubtype` を見れば**タブレット由来かどうか**まで分かる。

**だが、公開されているフィールドに、ベンダー ID・プロダクト ID に相当するものは無い。**「どのプロセスが作ったか」は分かっても、「**どの機械が押したか**」は分からない。1歩目でアプリの前提が崩れる。

（**公開されていないフィールドには、手がある。**これは最後に扱う。）

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

**これは採らなかった。**ただ、採らない理由が伝聞のままなのは気持ちが悪い。**実際に seize して測った。**

### 測り方

`IOHIDManager` で**対象のマウス1台だけ**を占有し、同時に `CGEventTap` を `.listenOnly` で張る。両方にログを出して、**押下がどちらに現れるか**を見る。

```swift
// 対象を1台に絞る（他のデバイスを巻き込まないため）
let match: [String: Any] = [kIOHIDVendorIDKey: 1133, kIOHIDProductIDKey: 45085]
IOHIDManagerSetDeviceMatching(manager, match as CFDictionary)

// ここだけを切り替えて2回走らせる
let options = seize ? IOOptionBits(kIOHIDOptionsTypeSeizeDevice)
                    : IOOptionBits(kIOHIDOptionsTypeNone)
IOHIDManagerOpen(manager, options)
```

**安全策を2つ入れてある。**対象を VID/PID で1台に限定すること（内蔵トラックパッドは触らないので、失敗しても操作不能にならない）と、**5秒で自動的に解除して終了すること**。占有したまま残ると、そのマウスが戻らなくなる。

### 結果

```
=== seize しない（kIOHIDOptionsTypeNone）===
  [HID] usage=5      [TAP] button=4
  [HID] usage=1      [TAP] button=0
  [HID] usage=2      [TAP] button=1
  [HID] usage=3      [TAP] button=2
  [HID] usage=4      [TAP] button=3
  → HID 8 回 / TAP 8 回

=== seize する（kIOHIDOptionsTypeSeizeDevice）===
  [HID] usage=5
  [HID] usage=4
  [HID] usage=1
  [HID] usage=3
  [HID] usage=2
  → HID 10 回 / TAP 0 回
```

| | HID 側で観測 | OS 側（`CGEventTap`）で観測 |
|---|---|---|
| `kIOHIDOptionsTypeNone`（監視のみ） | 8 回 | **8 回** |
| **`kIOHIDOptionsTypeSeizeDevice`（占有）** | 10 回 | **0 回** |

（2026-08-02・MX Ergo / BLE 接続・macOS 26）

**占有すると、OS 側には1つも流れない。**押下は自分のプロセスにだけ届き、他のアプリからは**そのマウスが沈黙したように見える**。**そのデバイスではカーソルも動かなくなる**（占有していない他のマウスやトラックパッドは、当然そのまま動く）。

つまり seize を選ぶなら、**HID レポートを自分で解釈して `CGEvent` を組み立て、post し直さなければならない。**移動量の積み上げも、クリックの組み立ても、すべて自前になる。**一箇所でも取りこぼした瞬間に、利用者のマウスが死ぬ。**

「落ちてもマウスは素のマウスとして動く」を**配るものの前提**にしている以上、マウスの生殺与奪を握る設計は選べなかった。だから `kIOHIDOptionsTypeNone`、**監視だけ**にしてある。

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
- **`CGEventTap` が先なら**：出所が分かるまで待つことになる。だがコールバックは**イベント配送のパスそのもの**なので、そこで待つと**そのイベントの配送が止まり、ユーザーの操作が引っ掛かる**。しかも遅すぎれば、**OS はタップごと切ってくる**（後述）。**設計が破綻する**

推測で決めずに測った。**3章と同じ実験で分かる** ── seize しないほうのログを、今度は**順序**に注目して読み直す。

```
[HID] usage=5   ← 先に HID
[TAP] button=4  ← 後から Quartz
[HID] usage=1
[TAP] button=0
[HID] usage=2
[TAP] button=1
[HID] usage=3
[TAP] button=2
   （以下同じ）
```

**8回すべて、HID が先だった**（2026-08-02・MX Ergo / BLE 接続・macOS 26）。1回の例外もない。順序が逆なら、この方式ごと捨てる必要があった。

ただし ── **これは観測であって、仕様ではない。**Apple がこの順序を保証しているわけではないので、USB 接続でも、別のメーカーのマウスでも、別のマシンでも同じとは限らない。**環境が変わったら測り直す**という前提で使っている。

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

**50ms は大きすぎるように見える。**実際に測ると、HID と `CGEventTap` の差は **0.24〜4.10ms**（9回、平均 1.44ms）だった。**桁が1つ半違う。**

それでもこの値にしてあるのは、**下限と上限が別々のもので決まっている**からである。

- **下限は機械が決める。**BLE の揺らぎとスケジューリングの遅れを吸収できるだけの余裕が要る。詰めすぎると、たまたま遅れた1回で出所を見失う
- **上限は人間が決める。**別々のボタンを押し分ける間隔より短くないと、**直前の1件を取り違える**

**その間に収まる範囲から選んだ値であって、測って出した値ではない。**

そして、**合わなければ諦める。**

```swift
/// 分からなければ nil。呼び手は全マウス共通の設定に落とす（**取り違えるより素直に諦める**）。
```

「誰のボタンか分からない」だけで、「ボタンが押された」ことは分かっている。**間違ったマウスの割当を当てるより、共通の設定に落とすほうが害が小さい。**

### 同じボタンに、番号が2つある

上のログをもう一度見てほしい。`usage=5` の直後に `button=4` が来ている。`usage=1` の後は `button=0`。

**HID の usage は 1 始まり、`CGEvent` の `buttonNumber` は 0 始まり。**同じボタンに2つの番号が付いていて、常に1ずれる。

| | 左 | 右 | 中 | サイド |
|---|---|---|---|---|
| HID の **usage** | 1 | 2 | 3 | 4, 5 |
| `CGEvent` の `buttonNumber` | **0** | **1** | **2** | **3, 4** |

**上のログでは5つとも踏んでいる。**左（1→0）・右（2→1）・中（3→2）・サイド2つ（4→3, 5→4）が、すべてこの対応で出た。

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

## 5. ひとつだけ、抜け道がある

ここまで「公開されているフィールドには無い」と書いてきた。**公開されていないものには、ある。**

`CGEventField` の定義を眺めると、**番号が飛んでいる**ことに気づく。

```c
kCGEventSourceStateID          = 45,
kCGScrollWheelEventIsContinuous = 88,   // ← 46〜87 は名前が付いていない
```

このうち **87 番**を読むと、値が返ってくる。マウスを2台つないで、それぞれの左ボタンを3回ずつ押した。

```
MX Ergo の左ボタン ×3    field87 = 0x1000014ac  （3回とも同じ）
bitra   の左ボタン ×3    field87 = 0x10000169a  （3回とも同じ）
```

**デバイスごとに違う値で、同じデバイスなら毎回同じ。**この値で `ioreg` を引くと、正体が分かった。

| | ノード | Product |
|---|---|---|
| `IOHIDManager` から見た ID | `IOHIDUserDevice` | MX Ergo |
| **`field 87` の値** | **`AppleUserHIDEventService`** | **MX Ergo** |

**そのデバイスのイベントを作っているドライバ**を指している。つまり ── **取れる。**縫わなくても、この1行で出所が分かる。

さらに面白いことに、**自分で `post` したイベントでは 0 になる**。割り当てのあるボタンを押して、ジェスチャーにならずクリックとして流し直されたときのログがこれである。

```
field87 = 0x1000014ac   差   0.35 ms   ← 実機から来たイベント
field87 = 0x0           差 176.17 ms   ← こちらが作って流したイベント
```

**実機由来か、合成かまで区別できる。**（2026-08-02・MX Ergo と bitra・macOS 26）

### それでも使わなかった

**公開ヘッダに無い。**名前も、意味も、Apple は何も言っていない。

- 次の macOS で**消えるかもしれない**。意味が変わるかもしれない
- 変わっても**バグではない**。もともと約束されていないのだから、文句を言う先が無い
- そして壊れたとき、**利用者の手元でマウスの挙動が変わる**

**調べ物としては使う。**デバッグ中に「いまのイベントはどこから来たのか」を知りたいとき、これほど早い手段は無い。だが**配るものには載せられない。**

「取れる」と「使える」は別である。50ms で縫うのは遠回りに見えるが、**公開されている API だけで組み立てられている**。それが選んだ理由になる。

## まとめ

これで「**どのマウスの、どのボタンが押されたか**」が分かるようになった。

- **「誰が押したか分かる層」と「止められる層」は別。**繋ぐ API は無いので、自分で縫う
- **どちらが先に届くかを、最初に測る。**順序が逆なら設計ごと成立しない
- **同じボタンに番号が2つある。**HID の usage は 1 始まり、`CGEvent` は 0 始まり
- **タップは黙って切られる。**再登録は必須
- **分からないときは諦める。**取り違えるより害が小さい
- **非公開フィールドなら1行で取れる。**ただし調べ物にしか使えない

残っているのは、受け取ったあとの話 ── **代わりのイベントを作って OS に返す**ところである。`CGEvent` を作って post するだけ、のはずだった。**そこにも穴があった。**

（後編に続きます）

---

### 用語

| 語 | 意味 |
|---|---|
| **HID** | Human Interface Device。入力機器が共通で使う規格 |
| **IOKit / IOHIDFamily** | macOS がデバイスを扱う土台。どの機器かを知っている層 |
| **Quartz Event Services** | Core Graphics のイベントシステム。`CGEvent` が流れる層。どの HID デバイスから来たかは分からない |
| **`IOHIDManager`** | IOKit の層を覗く API。機器の一覧と入力が取れる |
| **`CGEventTap`** | Quartz の層に割り込む API。書き換え・握りつぶしができる |
| **`kCGEventTapDisabledByTimeout`** | コールバックが遅く、OS がタップを切ったときに届く型。再登録が要る |
| **seize** | デバイスを占有すること。入力を独占できるが、OS 側に流れなくなる |
| **usage / Usage Page** | HID で「この値が何を意味するか」を表す番号と、その分類 |
