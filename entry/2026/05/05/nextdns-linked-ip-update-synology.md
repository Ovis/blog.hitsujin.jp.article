---
Title: NextDNSのLinked IPをSynology NASで自動更新するようにした話
Published: 2026/5/5 22:00:00
Tags:
  - "環境構築"
---

現在実家には父と母が二人で暮らしている。  
あまりネットとかPC、スマホに詳しいわけでもないので、何かあった時に私が電話でサポートしている。  
ここ数年迷惑系広告を踏んでしまって連絡をもらうことが何度かあったので、AdGuard Homeを実家で動かしているSynologyのNASで動かしていた。  

これで特段困ってはいなかったが、このNASに何かがあった時に父母がネットに接続できなくなってしまう問題があった。  
このために実家に帰るのも面倒くさい。  
最近NextDNSを契約したので、この機会に実家のネットワークでもNextDNSを利用することにした。

一点問題があり、実家で利用しているTP-LINKのDecoはDNS-over-HTTPSやDNS-over-TLSには対応していないため、NextDNSの「Linked IP」機能を使う構成にする必要がある。  
これについてはSynology NASのタスクスケジューラ機能でNextDNSが提供するグローバルIPアドレス更新のURLをCurlで叩くことで対応できる。  

今回は単純にcurlで叩くだけでは味気なかったのでもう少しスクリプト化することにした。  

なお、この記事ではSynologyのタスクスケジューラを使っているが、内容自体はcronなどでもそのまま応用できる。

<!-- more -->

## やりたいこと

- NextDNSのLinked IPを自動更新する
- IPが変わったときだけ更新する
- ログを残す  
  - ログローテートは行う

万が一Synology NASが不調でグローバルIPアドレスの更新ができなくなったとしても、最悪フィルターがかかっていない普通のDNSサーバーとして動くだけなので、インターネットにはつながる。  

## スクリプト 

適当にCodexに指示を出してシェルスクリプトを出力してもらった。  

### IPアドレス更新スクリプト
```bash
sudo tee /usr/local/bin/update-nextdns-linkip.sh > /dev/null <<'EOF'
#!/bin/bash
set -u

NEXTDNS_URL="https://link-ip.nextdns.io/hoge/fuga"

STATE_DIR="/var/lib/nextdns-linkip"
LAST_IP_FILE="${STATE_DIR}/last_ip"

LOG_FILE="/var/log/nextdns-linkip.log"

IP_CHECK_URL="https://ipv4.icanhazip.com"

MAX_RETRIES=5
RETRY_WAIT_SECONDS=10

mkdir -p "$STATE_DIR"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') $*" >> "$LOG_FILE"
}

# WAN疎通確認
if ! ping -c 1 -W 3 8.8.8.8 >/dev/null 2>&1; then
    log "WAN check failed"
    exit 1
fi

# 現在のグローバルIPv4を取得
CURRENT_IP="$(curl -4 -fsS --connect-timeout 5 --max-time 10 "$IP_CHECK_URL" | tr -d '[:space:]')" || {
    log "Failed to get current IPv4"
    exit 1
}

# IPv4形式の簡易チェック
if ! echo "$CURRENT_IP" | grep -Eq '^[0-9]+(\.[0-9]+){3}$'; then
    log "Invalid IPv4 returned: $CURRENT_IP"
    exit 1
fi

LAST_IP=""
if [ -f "$LAST_IP_FILE" ]; then
    LAST_IP="$(cat "$LAST_IP_FILE")"
fi

# IPが変わっていなければ何もしない
if [ "$CURRENT_IP" = "$LAST_IP" ]; then
    log "IP unchanged: $CURRENT_IP"
    exit 0
fi

log "IP changed: ${LAST_IP:-none} -> $CURRENT_IP"

# NextDNS Linked IP 更新
for i in $(seq 1 "$MAX_RETRIES"); do
    if curl -4 -fsS --connect-timeout 5 --max-time 10 "$NEXTDNS_URL" >> "$LOG_FILE" 2>&1; then
        echo "$CURRENT_IP" > "$LAST_IP_FILE"
        log "NextDNS Linked IP updated successfully: $CURRENT_IP"
        exit 0
    fi

    log "NextDNS update failed. retry=$i"
    sleep "$RETRY_WAIT_SECONDS"
done

log "NextDNS update failed after retries"
exit 1
EOF
```

実行権限を付与しておく必要がある。  


```bash
sudo chmod +x /usr/local/bin/update-nextdns-linkip.sh  
```


---

### ログ管理（logrotate）

```bash
sudo tee /etc/logrotate.d/nextdns-linkip > /dev/null <<'EOF'
/var/log/nextdns-linkip.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
    create 0644 root root
}
EOF
```

## Synology NAS側での設定  

Cronで設定しておいてもよいけども、必要に応じてGUI側で調整したかったのでNAS側のタスクスケジューラで設定をしておく。  

`コントロールパネル` の `タスクスケジューラー` で `作成 > 予約タスク > ユーザー指定のスクリプト`  を押下。  

`全般設定` ではタスクに適当な名前を入れておく。 `ユーザー` はroot。  
`スケジュール` では実行日数を `繰り返し：毎日` に設定。  
`時間` については開始時刻を `0時` にしたうえで、 `同日内に実行し続ける` にチェックを入れて `繰り返し` は動作間隔を設定。今回は5分ごとにした。  
`最終実行時刻` は 5分ごとに動かすなら `23:55` にする。  
この `最終実行時刻` はこのスクリプトが同一日の中で動かす中での最終的な実行時刻を定義するもの。日中にだけ動かすときなんかに使う様子。  
今回は24時間動いてほしいので `23:55` にしておく。   

* 実行日：毎日
* 開始時刻：00:00
* 同日内に実行：ON
* 繰り返し：5分ごと
* 最終実行時刻：23:55

これで動作確認としてスケジュールを実行し、ログに以下のような感じで出力されていたらOK。  

```log
2026-05-05 21:11:19 IP changed: none -> 18x.11x.21x.11
2026-05-05 21:11:19 NextDNS Linked IP updated successfully: 18x.11x.21x.11
```

今後5分おきに動作し、IPアドレスの変更がなければ

```log
2026-05-05 21:20:04 IP unchanged: 18x.11x.21x.11
```
という感じでログが残る。  

## Synology以外での実行方法

このスクリプト自体は環境非依存なので、例えば以下でも動く。

### cron

```bash
*/5 * * * * /usr/local/bin/update-nextdns-linkip.sh
```