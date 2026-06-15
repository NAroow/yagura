# yagura 櫓

Zabbix互換プロトコルを話す、シングルバイナリの監視サーバー。

- **依存ゼロ** — Go標準ライブラリのみ。外部DB不要、ランタイム不要。`CGO_ENABLED=0`ビルドなのでAlpine (musl) にバイナリ1個置くだけで動く
- **Zabbixエージェントそのまま使える** — パッシブチェック / アクティブチェック / zabbix_sender (trapper) 対応。エージェント側の設定は `Server=` / `ServerActive=` をyaguraに向けるだけ
- **Zabbixテンプレを取込可能** — 公式のJSONエクスポート形式をインポート(YAMLは `yq -o=json` で変換)
- **トリガー式はZabbix 6構文** — `avg(/host/key,5m)>2` 形式のサブセットを実装
- **組み込みTSDB** — アイテム毎のappend-onlyファイル + メモリリングバッファ。バックアップはdataディレクトリのコピーだけ
- **通知** — Discord webhook / 汎用JSON webhook
- **死活監視 (サーバー側)** — エージェント/SNMPホストが応答しなくなったら、テンプレのトリガーが無くても自動でホストダウンを検知して通知。最終ポーリング成功からの経過時間で判定
- **多言語UI** — ブラウザ言語で英語/日本語を自動切替(サイドバー下部で手動切替も可)。`internal/web/static/i18n/*.json` を足せば言語追加できる

## クイックスタート (Alpine LXC)

Releasesから自分のアーキのバイナリ(`yagura-linux-amd64` / `yagura-linux-arm64`)を取得、
または自前ビルド(`CGO_ENABLED=0 go build -ldflags='-s -w' -o yagura .`)。

```sh
apk add --no-cache ca-certificates          # HTTPS webhook (Discord等) に必要

# サービス用 user + group
addgroup -S yagura && adduser -S -D -H -G yagura -s /sbin/nologin yagura

# バイナリ配置
install -m755 yagura-linux-amd64 /usr/local/bin/yagura

# OpenRC サービス (リポジトリ同梱)
install -m755 yagura.initd /etc/init.d/yagura
rc-update add yagura default
rc-service yagura start
```

データ(`/var/lib/yagura`)とログ(`/var/log/yagura.log`)は初回起動時に自動生成される。
Web UIは `http://<host>:8086`、trapper(アクティブチェック/sender受付)は `:10051`。

認証を掛ける / オプションを上書きする場合は `/etc/conf.d/yagura`(initが起動時にsource):

```sh
# /etc/conf.d/yagura   (chmod 600 — トークンを含むため)
YAGURA_TOKEN="$(head -c24 /dev/urandom | base64)"
YAGURA_OPTS="-data /var/lib/yagura -http :8086 -trapper :10051"
```

`rc-service yagura restart` で反映。トークン設定時はUI初回アクセスで聞かれ、APIは `Authorization: Bearer <token>`。

## エージェント側設定 (zabbix-agent / agent2)

```ini
Server=<yaguraのIP>          # パッシブチェック許可
ServerActive=<yaguraのIP>    # アクティブチェック送信先 (ポート10051)
Hostname=web01               # ★yagura側のホスト名と完全一致させる
```

zabbix_senderもそのまま:

```sh
zabbix_sender -z <yaguraのIP> -s web01 -k custom.metric -o 42.5
```

(受け側アイテムは type=trapper で作成しておく)

### アクティブチェックの注意

**アクティブ**アイテム(エージェントがyaguraに接続してデータをpushする方式)では:

- `ServerActive` は yagura の **trapper ポート `:10051`** に向ける ── Web UI の `:8086` ではない。アクティブチェックと `zabbix_sender` はWebポートを一切使わない
- エージェントの `Hostname` を yagura 側のホスト名と完全一致させる(これで受信データがホストに紐付く)
- アイテムが **アクティブ型** であること ── 「… by Zabbix agent **active**」テンプレを取り込むか、アイテム種別をactiveにする。パッシブのみのホストはアクティブチェックを一切配布しない

モダンなエージェント(`zabbix_agentd` / `agent2` 4.0+)はアクティブチェックのデータをzlib圧縮して送る。yaguraは圧縮プロトコルに対応済みなので、現行エージェントがそのまま動く。

## Zabbixからの移行

1. Zabbix UIでテンプレートをエクスポート(**JSON形式**を選択)
2. YAMLでエクスポートした場合: `yq -o=json template.yaml > template.json`
3. yaguraの「テンプレ」ページでインポート
4. ホスト作成時にテンプレをリンク → アイテム/トリガーが実体化され、トリガー式の `/テンプレ名/` は自動でホスト名に書き換わる

### 対応状況

| 機能 | 状態 |
|---|---|
| パッシブチェック (Zabbix agent) | ✅ |
| アクティブチェック | ✅ |
| zabbix_sender / trapper | ✅ |
| テンプレJSONインポート (items/triggers) | ✅ |
| トリガー関数 last/avg/min/max/sum/count/nodata/change | ✅ |
| and/or/not・四則演算・K/M/G/T・s/m/h/d/wサフィックス | ✅ |
| {HOST.NAME}マクロ (トリガー名) | ✅ |
| ホストダウン検知 (サーバー側可用性) | ✅ テンプレのトリガー非依存 |
| SNMP v1/v2c GET アイテム (snmp_oid) | ✅ 標準ライブラリ実装・外部依存なし |
| 前処理: Change per second / Multiplier | ✅ (DISCARD_UNCHANGED系は無視して取込) |
| SNMPv3 / snmp_oid の walk[] | ❌ 無効状態で取込 |
| CALC / DEPENDENT / HTTP agent アイテム | ❌ 無効状態で取込 (senderで代替可) |
| LLD (ローレベルディスカバリ) | ❌ スキップされる |
| その他の前処理 (REGEX, JSONPATH等) | ❌ 該当アイテムは無効で取込 |
| ユーザーマクロ {$MACRO} | ❌ (delayは60sにフォールバック) |

未対応タイプのアイテムは「無効」で取り込まれるので、必要なら手動で有効化してtrapper運用(cron + zabbix_sender等)に切り替えられる。

## SNMP監視

ホストに community / port (既定 public / 161) を設定すれば、テンプレ内の
SNMP_AGENT アイテムがそのまま動く。FortiGateやスイッチの公式テンプレも
GET系アイテムは取込後すぐ収集が始まる。

- `ifHCInOctets` 等のカウンタは Change per second + Multiplier 前処理を
  そのまま適用するので bps が直接記録される(初回サンプルとカウンタ
  リセット時は破棄)
- `walk[...]` 形式のOIDとLLD依存アイテムは未対応(無効で取込まれるので、
  必要ならOID直書きのGETアイテムに手で書き換えれば動く)
- SNMPv3が要る機器は当面 v2c の read-only community で

## 死活監視 (ホストダウン検知)

公式のZabbix Linuxテンプレは可用性を内部アイテム `zabbix[host,agent,available]` で
判定するが、yaguraはこの内部アイテムを持たない。SNMPテンプレに至っては死活用の
アイテムが無いことが多い。そのため「テンプレを入れただけ」ではホストを落としても
気づけない。

そこでyaguraは**サーバー側**で各ホストの「最終ポーリング成功時刻」を監視し、一定
時間どのアイテムも値を返さなければ自動で `Unreachable` 障害を上げる(テンプレの
トリガーは不要)。パッシブ / SNMP / アクティブの全方式に効く。障害はダッシュボード・
ホスト状態(緑→赤)・webhook通知すべてに反映される。

- しきい値はUI設定の「未応答とみなす時間」(`unreachable_sec`、秒)
- `0` = 自動: そのホストの最速アイテム間隔 × 3(最低120秒)。数回のポーリング
  取りこぼしでは誤検知しない
- 一度も値を受信していないホスト(起動直後 / インポート直後)は判定対象外。
  「一度生きていたホストが落ちた」を確実に検知する設計
- 復帰すると障害は自動クローズ(RESOLVED通知)

## 言語 (i18n)

UIはブラウザの言語設定で英語/日本語を自動選択する(`ja` 以外は英語)。サイドバー
下部の `EN` / `日本語` で手動切替も可能(選択は `localStorage` に保存)。

翻訳ファイルは `internal/web/static/i18n/{en,ja}.json` の単純なキー→文字列マップ。
言語を足したいときは同じキーで `<lang>.json` を追加するだけ(ログとサーバー生成の
障害名は英語のまま — 運用上の共通言語として固定)。

## 運用メモ

- データは `-data` ディレクトリに完結(`meta.json` + `ts/`)。バックアップ = ディレクトリコピー
- 保持期間はUI設定から(デフォルト90日、毎日深夜にsweep)
- 数値1サンプル=16バイト。100アイテム×60秒間隔×90日 ≒ 200MB程度
- ポート: 8086 (UI/API) / 10051 (trapper)。パッシブチェックはyagura→エージェント:10050への接続

## アーキテクチャ

```
┌─────────────────────────── yagura (single binary) ───┐
│  poller ──→ zabbix agents (:10050 passive)           │
│  trapper (:10051) ←── active agents / zabbix_sender  │
│        │                                             │
│        ▼                                             │
│  store (meta.json) + TSDB (append-only per item)     │
│        │                                             │
│  trigger engine (15s sweep: triggers + 死活監視) ──→ Discord/webhook │
│        │                                             │
│  web UI + REST API (:8086, go:embed SPA)             │
└──────────────────────────────────────────────────────┘
```
