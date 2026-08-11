---
title: "【所在で見る】GeoIP で“このIPはどこから来たか”を推定する ― その IP、“国内”ですか？（第3話）"
emoji: "🛰️"
type: "tech"
topics: ["php", "geoip", "maxmind", "ipアドレス"]
published: true
---

:::message
**連載「その IP、“国内”ですか？」（全5回）**

1. [問題編：なぜ“国内判定”はつまずくのか](https://zenn.dev/brainy_software/articles/domestic-ip-allocation-vs-geolocation)
2. [割り当てで見る：APNIC の割り当てファイルから日本の CIDR を作る](https://zenn.dev/brainy_software/articles/japan-ip-by-apnic-allocation)
3. **所在で見る：GeoIP で“どこから来たか”を推定する** 👈 この記事
4. 正規利用者で見る：VPN・匿名プロキシを弾く（近日公開）
5. まとめ：目的別の選び方と、パナマIPの答え合わせ（近日公開）
:::

[前回（第2話）](https://zenn.dev/brainy_software/articles/japan-ip-by-apnic-allocation)、割り当てベースは私を弾いた——戸籍がパナマだから。でも私は日本にいて、日本のサーバーから出ている。今回は視点を変えて、**「所在」で判定する**。

## 「所在」で判定するとは

割り当て（戸籍）ではなく、**そのIPが地理的にどこから来ているかの“推定”**で見る方針だ。これを担うのが **GeoIP データベース**。IP アドレスを入力すると、推定される国・地域を返す。

定番は **MaxMind** の GeoIP。無料の **GeoLite2** と、有料で精度の高い **GeoIP2** がある。まずは無料の GeoLite2 Country で十分だ。

## データソース：MaxMind GeoLite2

- MaxMind に無料アカウントを作り、ライセンスキーを取得して `GeoLite2-Country.mmdb` をダウンロードする。
- 週2回ほど更新されるので、定期的に取り直す（`geoipupdate` ツールが公式）。
- PHP からは公式パッケージ `composer require geoip2/geoip2` で読む。

## 実装（PHP）

```php
<?php
require 'vendor/autoload.php';

use GeoIp2\Database\Reader;
use GeoIp2\Exception\AddressNotFoundException;

$reader = new Reader('/var/geoip/GeoLite2-Country.mmdb');

function geoCountry(Reader $reader, string $ip): ?string {
    try {
        // 推定される国コード（'JP' など）を返す
        return $reader->country($ip)->country->isoCode;
    } catch (AddressNotFoundException $e) {
        return null; // DB に載っていない
    }
}

$iso     = geoCountry($reader, $ip);
$isJapan = ($iso === 'JP');
```

割り当てベース（第2話）が「APNIC の記録」という**台帳**を引いたのに対し、GeoIP は**推定モデル**を引く。ここが本質的な違いだ。

## 精度と限界（ここが肝）

GeoIP は**推定**なので、外れる。とくに次のケース：

- **VPN / プロキシ**：GeoIP は多くの場合「**出口サーバーの所在国**」を返す。日本サーバー経由の VPN なら「日本」と出やすい。
- **モバイル回線（CGNAT）**：キャリアのゲートウェイ所在地に寄り、実際の居場所とずれる。
- **クラウド / CDN**：データセンターの国が出る。
- **更新遅延**：再割り当て直後は古い国のまま。

そして重要な点——**無料の GeoLite2 Country は「VPN かどうか」のフラグを持たない**。「VPN を弾きたい」なら別の DB（GeoIP2 Anonymous IP・有料）か別の手段が要る。これは次回（第4話）の話。

## 答え合わせ②：私のパナマIPは？

私の固定IP（NordVPN の**日本**サーバー経由）を GeoIP に通すと——

> **`JP`（日本・東京）。** ipinfo・ip-api のどちらで引いても同じ結果だった。

これは冒頭の「精度と限界」で挙げた **VPN のケースそのもの**だ——GeoIP は**出口サーバーの所在**を返す。私の出口は東京なので、GeoIP は素直に「東京・日本」と答えた。割り当て（第2話）ではパナマ籍として弾かれた同じ IP が、所在では**日本の中にいる**ことになる。

ここで**第1話の“逆の答え”が実演される**：**割り当てでは弾かれた私が、所在では“国内”として通る**。同じ 1 つの IP。同じ私。判定の流儀が違うだけで、結論が反転する。

でも——所在で「日本」と出たからといって、本当に通していいのか？ VPN 越しの相手を、そのまま「日本の正規利用者」として扱っていいのか？

## 次回予告（第4話）

次は三つ目の視点、**「正規利用者か」**。所在で日本と出ても、VPN や匿名プロキシを弾きたい場面はある。その実装（ASN や匿名IP DB での検出）と、私のIPが**VPN として見抜かれるのか**を確かめる。答え合わせ③へ。

> あなたの「国内判定」は、割り当て・所在・正規利用者のどれで見ていますか？ よければコメント欄や X（[@brainysoftware](https://x.com/brainysoftware)）で聞かせてください。
