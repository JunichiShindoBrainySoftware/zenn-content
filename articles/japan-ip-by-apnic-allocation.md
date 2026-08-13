---
title: "【割り当てで見る】APNIC の割り当てファイルから“日本のIP”を判定する ― その IP、“国内”ですか？（第2話）"
emoji: "🗾"
type: "tech"
topics: ["php", "ネットワーク", "ipアドレス", "apnic"]
published: true
---

:::message
**連載「その IP、“国内”ですか？」（全5回）**

1. [問題編：なぜ“国内判定”はつまずくのか](https://zenn.dev/brainy_software/articles/domestic-ip-allocation-vs-geolocation)
2. **割り当てで見る：APNIC の割り当てファイルから日本の CIDR を作る** 👈 この記事
3. [所在で見る：GeoIP で“どこから来たか”を推定する](https://zenn.dev/brainy_software/articles/japan-ip-by-geoip)
4. [正規利用者で見る：VPN・匿名プロキシを弾く](https://zenn.dev/brainy_software/articles/japan-ip-detect-vpn)
5. [まとめ：目的別の選び方と、パナマIPの答え合わせ](https://zenn.dev/brainy_software/articles/japan-ip-check-summary)
:::

[前回（第1話）](https://zenn.dev/brainy_software/articles/domestic-ip-allocation-vs-geolocation)、私の固定IPは「日本を名乗り、戸籍はパナマ」だった。今回はまず、**その“戸籍”で判定する方法**——IP の**割り当て国**で「日本かどうか」を見る実装をやる。

## 「割り当て」で判定するとは

インターネットの IP アドレスは、地域ごとの管理団体（RIR）が国・組織に**割り当て**ている。日本を含むアジア太平洋は **APNIC** が管理している。

「割り当てで国内判定する」とは、**APNIC が「日本に割り当てた」と記録している IP レンジ**を許可リストにして、そこに含まれる IP だけを国内とみなす、という方針だ。一次情報がはっきりしているのが強み。

## データソース：RIR の割り当てファイル

APNIC は、どの IP レンジがどの国に割り当てられているかを**毎日**公開している。

- ファイル：`https://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest`
- 1 行 1 レコード、`|` 区切り：

```text
apnic|JP|ipv4|1.0.16.0|4096|20110412|allocated
apnic|JP|ipv4|1.0.64.0|8192|20110412|allocated
apnic|US|ipv4|1.1.1.0|256|20110811|assigned
```

`JP` の `ipv4` 行を拾えば、「日本に割り当てられた IPv4 レンジ」が全部手に入る。ポイントは 5 列目の `value`——これは**アドレス数**であって、プレフィックス長ではない（`4096` = 4096 個のアドレス）。

## 実装（PHP）

割り当てファイルを読み、日本の IPv4 レンジを `[開始, 終了]` の整数区間にして、二分探索で含有判定する。

```php
<?php
// 1. APNIC の割り当てファイル（日次更新）を取得
$url = 'https://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest';
$lines = file($url, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

// 2. 日本(JP)の IPv4 割り当てを [開始, 終了] の整数レンジへ
$ranges = [];
foreach ($lines as $line) {
    if ($line === '' || $line[0] === '#') continue;
    $f = explode('|', $line);
    // 例: apnic|JP|ipv4|1.0.16.0|4096|20110412|allocated
    if (count($f) < 7 || $f[1] !== 'JP' || $f[2] !== 'ipv4') continue;
    if (!in_array($f[6], ['allocated', 'assigned'], true)) continue;
    $start = ip2long($f[3]);   // 開始IP → 整数
    $count = (int) $f[4];      // アドレス数（プレフィックス長ではない）
    $ranges[] = [$start, $start + $count - 1];
}
usort($ranges, fn($a, $b) => $a[0] <=> $b[0]);

// 3. 判定：IP が日本割り当てレンジに含まれるか（二分探索）
function isJapanAllocated(string $ip, array $ranges): bool {
    $n = ip2long($ip);
    if ($n === false) return false;   // IPv4 以外
    $lo = 0; $hi = count($ranges) - 1;
    while ($lo <= $hi) {
        $mid = intdiv($lo + $hi, 2);
        [$s, $e] = $ranges[$mid];
        if     ($n < $s) $hi = $mid - 1;
        elseif ($n > $e) $lo = $mid + 1;
        else             return true;
    }
    return false;
}
```

使ってみる：

```php
var_dump(isJapanAllocated('133.11.0.0', $ranges));   // 東京大学のレンジ → true
var_dump(isJapanAllocated($myNordVpnIp, $ranges));   // 私の固定IP     → false
```

## 実運用の注意

- **日次更新**：ファイルは毎日変わる。起動時に取得してキャッシュし、1 日 1 回更新する。
- **IPv6 も同様**。ただし `ipv6` 行の `value` は**プレフィックス長**（アドレス数ではない）——ここが IPv4 と違う。
- **`allocated` / `assigned` のみ**採用し、`reserved` / `available` は除外する。
- **64bit PHP 前提**（`ip2long` の戻り値の符号）。32bit 環境なら `sprintf('%u', ...)` で符号なしに直す。
- ファイアウォールや nginx の allowlist にしたいなら、レンジを **CIDR に変換**して出力する（区間→CIDR 変換）。
- そして最大の注意——**これはあくまで「割り当て」**。所在地ではない（第1話の話）。

## 答え合わせ①：私のパナマIPは？

私の固定IP（NordVPN の日本サーバー経由）を `isJapanAllocated()` に通すと——**`false`**。
APNIC の日本レンジに、パナマ割り当ての私のIPは入っていない。**割り当てベースは、日本サーバー経由だろうと私を通さない。** 第1話で私を弾いたのは、まさにこの判定だ。

でも、これで「私＝国外ユーザー」と結論していいのだろうか。私は日本にいて、日本のサーバーから出ているのに。

## 次回予告（第3話）

次は視点を変えて**所在ベース**——GeoIP で「このIPは地理的にどこから来たか」を推定する。同じ私のIPが、GeoIP では**日本**と出るのか、それとも**VPN**と見抜かれるのか。答え合わせ②へ。

> あなたの環境の「国内判定」は、割り当て・所在・正規利用者のどれで見ていますか？ よければコメント欄や X（[@brainysoftware](https://x.com/brainysoftware)）で聞かせてください。

## 更新履歴

- **2026-08-13**：第3話・第4話の公開にあわせて、冒頭の連載目次から各記事へのリンクを追加しました（第5話は近日公開）。
- **2026-08-14**：連載完結にあわせて、冒頭の連載目次に第5話へのリンクを追加しました。
