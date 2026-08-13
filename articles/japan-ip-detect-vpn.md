---
title: "【正規利用者で見る】VPN・匿名プロキシを弾く ― その IP、“国内”ですか？（第4話）"
emoji: "🕵️"
type: "tech"
topics: ["php", "geoip", "vpn", "セキュリティ"]
published: true
---

:::message
**連載「その IP、“国内”ですか？」（全5回）**

1. [問題編：なぜ“国内判定”はつまずくのか](https://zenn.dev/brainy_software/articles/domestic-ip-allocation-vs-geolocation)
2. [割り当てで見る：APNIC の割り当てファイルから日本の CIDR を作る](https://zenn.dev/brainy_software/articles/japan-ip-by-apnic-allocation)
3. [所在で見る：GeoIP で“どこから来たか”を推定する](https://zenn.dev/brainy_software/articles/japan-ip-by-geoip)
4. **正規利用者で見る：VPN・匿名プロキシを弾く** 👈 この記事
5. [まとめ：目的別の選び方と、パナマIPの答え合わせ](https://zenn.dev/brainy_software/articles/japan-ip-check-summary)
:::

[前回（第3話）](https://zenn.dev/brainy_software/articles/japan-ip-by-geoip)、所在（GeoIP）では私のIPは「日本」と出た。割り当てでは弾かれた私が、所在では通る。だが——「日本と出れば通す」でいいとは限らない。**VPN 越しのアクセスを弾きたい**場面がある（不正対策、地域限定コンテンツ、規制対応）。今回は三つ目の視点、**「正規利用者か」**を見る。

## 「正規利用者か」で判定するとは

「そのIPが日本に見えるか」ではなく、「**そのIPが VPN・匿名プロキシ・データセンター経由でないか**」を見る方針だ。所在が日本でも、VPN 経由なら弾く——という判定になる。

方法はいくつかある。

| 方法 | 中身 | コスト |
|---|---|---|
| **匿名IP DB** | MaxMind GeoIP2 Anonymous IP（`is_anonymous_vpn` 等のフラグ） | 有料 |
| **ASN で判定** | IP の ASN（事業者）がホスティング/VPN 事業者かを見る。GeoLite2 ASN（無料）で引ける | 無料 |
| **商用 API** | IPQualityScore 等の VPN/proxy スコア | 従量 |
| **ヒューリスティック** | データセンターのレンジ・ふるまいで推測 | 脆い |

まずは無料で始められる **ASN 判定**を実装する。

## 実装（PHP・ASN 判定）

VPN 事業者は自前の IP を持たず、**ホスティング事業者（M247 など）の回線**を使うことが多い。IP の ASN と組織名を引き、ホスティング/VPN 事業者に当たれば「VPN 疑い」とする。

```php
<?php
require 'vendor/autoload.php';

use GeoIp2\Database\Reader;
use GeoIp2\Exception\AddressNotFoundException;

$asnReader = new Reader('/var/geoip/GeoLite2-ASN.mmdb');

// ホスティング/VPN 事業者の組織名（部分一致の deny-list）。運用しながら育てる。
$denyKeywords = ['M247', 'DataCamp', 'Tefincom', 'NordVPN', 'Hosting', 'Datacenter', 'Cloud', 'Server'];

function looksLikeVpn(Reader $asnReader, string $ip, array $denyKeywords): bool {
    try {
        $org = $asnReader->asn($ip)->autonomousSystemOrganization ?? '';
    } catch (AddressNotFoundException $e) {
        return false;
    }
    foreach ($denyKeywords as $kw) {
        if (stripos($org, $kw) !== false) return true;
    }
    return false;
}

// 「日本の正規利用者か」= 所在が日本 かつ VPN でない
$isDomesticUser = ($geoIso === 'JP') && !looksLikeVpn($asnReader, $ip, $denyKeywords);
```

## 精度と限界

- **いたちごっこ**：VPN 事業者は ASN を増やし・変える。deny-list は永遠に未完成。本気でやるなら**有料の匿名IP DB か商用 API**が現実的。
- **誤検出**：正規のクラウド/企業プロキシ経由の“本物の日本ユーザー”まで弾いてしまう。厳しくするほど、正規利用者を巻き込む。
- キーワード deny-list は**取っ掛かり**。実運用では ASN 番号のリスト＋匿名IP DB を併用する。

## 答え合わせ③：私のパナマIPは？

私の固定IP（NordVPN の日本サーバー経由）の ASN を引くと——

> **`AS272096` / 組織名 `PacketHub S.A.`（パナマ籍）。**

ところが、上の `$denyKeywords`（`M247` / `Tefincom` / `NordVPN` / `Hosting` …）に **`PacketHub` は入っていない**。だから `looksLikeVpn()` に通すと——**`false`**。**私のVPNは、この判定をすり抜けた。**

所在で「日本」と出た私は、素の VPN 判定でも**通ってしまった**。NordVPN の日本サーバーが使う実体は、私が想定していた M247 ではなく `PacketHub S.A.` で、私のリストがそれを知らなかっただけだ。`PacketHub` を足せば、次からは弾ける：

```php
$denyKeywords[] = 'PacketHub';   // ← 実際に食らってから足した
```

だが、これこそ**いたちごっこの正体**だ。私は「自分のIPを実際に引いて、外した」から気づけた。明日 NordVPN が別の ASN を使えば、私のリストはまた取りこぼす。**ASN 判定は「知っている VPN しか弾けない」**——リストに載っていれば弾かれ、載っていなければ通る。それだけだ。

ここまでで同じ 1 つの IP は、**弾かれる（割り当て）／通る（所在）／リスト次第（正規利用者）** と、判定ごとに違う顔を見せた。

## 次回予告（第5話・まとめ）

三方式で結果はバラバラ。**じゃあ、結局どれを使えばいいのか？** 最終回は、目的別（法規制・不正対策・配信・課金）の選び方を整理し、私のパナマIPの**答え合わせを総括**する。第1話の問い「あなたなら、このIPを“国内”と判定しますか？」に、実装から決着をつける。

> あなたなら VPN 経由の日本サーバー出口を「国内」として通しますか、弾きますか？ よければコメント欄や X（[@brainysoftware](https://x.com/brainysoftware)）で。

## 更新履歴

- **2026-08-14**：連載完結にあわせて、冒頭の連載目次に第5話へのリンクを追加しました。
