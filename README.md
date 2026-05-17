# ATLAS セットアップガイド

**Authenticated Tunnel for Live Astrophotography Streaming**

> **カスタムドメインについて**
> `skycluster.jp` は管理者のみが使用できるドメインです。このシステムを独自に構築する場合は、**必ず自分のカスタムドメインを取得・使用してください**。この文書ではあえて `skycluster.jp` を例として記載しています。

---

## 概要

### メイン構成: ライブスタック公開

このシステムは StellarMate（天体撮影用 Raspberry Pi）で撮影したライトフレームを自動でスタック処理し、VPS 経由でリアルタイムに公開するためのものです。

StellarMate は OpenVPN でVPSに接続します。新しいFITSファイルが生成されるたびに自動検知・スタック・AVIF変換が行われ、VPSに転送されます。公開/非公開の切り替えはWebコンソールからパスワード認証後に操作できます。

```
StellarMate (Arch Linux) ← OpenVPN Client + indiserver
  ↓ 新FITSを検知
  ↓ スタック処理 (Sigma Clipping + 位置合わせ) + AVIF変換
  ↓ OpenVPN Tunnel
VPS (Oracle Cloud) ← OpenVPN Server
  ↓ rsync受信
  Webサーバー (HTTPS)
  → /login          コンソールログイン (https://atlas.skycluster.jp/login)
  → /console        公開管理
  → /               公開ページ (全世界閲覧, 5秒自動更新)
```

---

### ⚠️ 非推奨: 全世界からの望遠鏡制御

> **⛔ WARNING: 完全非推奨**
>
> ポート 7624 (INDI)、8624 (INDI Web Manager)、3000 (StellarMate Dashboard) はデフォルトでは**閉じています**。
> これらを開放すると全世界から望遠鏡を制御することが技術的に**可能**になりますが、**画像データ転送量が膨大になるため実用上は非推奨**です。
> また INDI プロトコルには認証機能がないため、セキュリティリスクも高くなっています。
> 開放方法は文書末尾の「付録: INDI系ポートの開放」を参照してください。

```
任意のデバイス (全世界)
  ↓ TCP skycluster.jp:7624   (INDI 制御)            ← ⛔ 非推奨・デフォルト閉鎖
  ↓ TCP skycluster.jp:8624   (INDI Web Manager)      ← ⛔ 非推奨・デフォルト閉鎖
  ↓ TCP skycluster.jp:3000   (StellarMate Dashboard) ← ⛔ 非推奨・デフォルト閉鎖
VPS (socat フォワード)
  ↓ OpenVPN Tunnel
StellarMate ← indiserver (望遠鏡・カメラ制御)
```

---

## GitHubフォルダ構成

```
src/
├── VPS/
│   ├── server.js              # Webサーバー
│   └── favicon.webp           # ファビコン・iOSアイコン
└── StellarMate/
    ├── stack_and_transfer.py  # スタック＆転送スクリプト
    ├── watch_and_stack.sh     # ファイル監視スクリプト
    └── healthcheck.sh         # VPN・監視プログラム自動復旧スクリプト
```

[ダウンロードはこちらから](https://github.com/mahiromiyake/a.t.l.a.s./releases)


---

## Part 1: VPS セットアップ (Oracle Cloud)

### 1-1. Oracle Cloud インスタンス作成

1. https://www.oracle.com/cloud/free/ でアカウントを作成してください。
2. ホームリージョンを **Japan East (東京)** または **Japan West (大阪)** に設定してください（変更不可）。
3. Compute → Instances → Create Instance に進み、以下の設定でインスタンスを作成してください。
   - Shape: VM.Standard.E2.1.Micro (Always Free)
   - OS: Ubuntu 22.04
   - SSHキーペアを作成しダウンロードしてください（紛失厳禁）。

### 1-2. VCN・Subnet作成

Networking → Virtual Cloud Networks → Create VCN に進み、以下を設定してください。

- IPv4 CIDR: `10.0.0.0/16`

次に VCN → Create Subnet で以下を設定してください。

- Subnet Type: Regional
- IPv4 CIDR: `10.0.0.0/24`
- Subnet Access: **Public Subnet**

続いて VCN → Internet Gateways → Create Internet Gateway でInternet Gatewayを作成してください。

VCN → Route Tables → Default Route Table → Add Route Rule に以下を追加してください。

- Destination: `0.0.0.0/0`
- Target: 作成したInternet Gateway

### 1-3. Security List (ファイアウォール)

VCN → Subnet → Security List → Ingress Rules に以下のルールを追加してください。

| Protocol | Source CIDR | Port | 用途 |
|---|---|---|---|
| TCP | 0.0.0.0/0 | 22 | SSH |
| UDP | 0.0.0.0/0 | 1194 | OpenVPN |
| TCP | 0.0.0.0/0 | 80 | HTTP (HTTPSリダイレクト) |
| TCP | 0.0.0.0/0 | 443 | HTTPS |
| TCP | 0.0.0.0/0 | 2222 | SFTP |

> **注意**: ポート 3000、7624、8624 はデフォルトでは開放しません。全世界からの望遠鏡制御が必要な場合のみ、付録の手順に従って開放してください。

### 1-4. SSH接続

以下のコマンドでVPSにSSH接続してください。

```bash
chmod 600 ~/Downloads/ssh-key.key
ssh -i ~/Downloads/ssh-key.key ubuntu@<VPS Public IP>
```

### 1-5. OS側ファイアウォール設定

以下のコマンドでOS側のファイアウォールを設定してください。

```bash
sudo apt update && sudo apt upgrade -y
sudo iptables -I INPUT -p udp --dport 1194 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 2222 -j ACCEPT
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

### 1-6. OpenVPNサーバー構築

以下のコマンドでOpenVPNサーバーを構築してください。

```bash
sudo apt install openvpn easy-rsa -y
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey secret ~/openvpn-ca/pki/ta.key
```

必要なファイルをコピーしてください。

```bash
sudo cp ~/openvpn-ca/pki/ca.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/private/server.key /etc/openvpn/
sudo cp ~/openvpn-ca/pki/dh.pem /etc/openvpn/
sudo cp ~/openvpn-ca/pki/ta.key /etc/openvpn/
```

`/etc/openvpn/server.conf` を以下の内容で作成してください。

```
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0
server 10.8.0.0 255.255.255.0
keepalive 10 120
cipher AES-256-CBC
persist-key
persist-tun
status /var/log/openvpn-status.log
verb 3
```

以下のコマンドでOpenVPNを起動してください。

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server
```

### 1-7. クライアント証明書作成 (StellarMate用)

以下のコマンドでStellarMate用のクライアント証明書を作成してください。

```bash
cd ~/openvpn-ca
./easyrsa gen-req stellarmate nopass
./easyrsa sign-req client stellarmate
```

`stellarmate.ovpn` をVPS上に生成してください。この時点ではまだStellarMateとVPSが接続されていないため、転送はPart 3で直接ファイルを作成する形で行います。

```bash
cat > ~/stellarmate.ovpn << EOF
client
dev tun
proto udp
remote skycluster.jp 1194
resolv-retry infinite
nobind
persist-key
persist-tun
cipher AES-256-CBC
verb 3
key-direction 1
<ca>
$(cat ~/openvpn-ca/pki/ca.crt)
</ca>
<cert>
$(cat ~/openvpn-ca/pki/issued/stellarmate.crt)
</cert>
<key>
$(cat ~/openvpn-ca/pki/private/stellarmate.key)
</key>
<tls-auth>
$(cat ~/openvpn-ca/pki/ta.key)
</tls-auth>
EOF
```

生成された内容を確認してください。Part 3でこの内容をStellarMate上に直接貼り付けます。

```bash
cat ~/stellarmate.ovpn
```

### 1-8. socatでポートフォワード設定

以下のコマンドでsocatをインストールし、systemdサービスを作成してください。

> **注意**: ここではSFTP（2222）のみ有効化します。INDI系ポートのsocatサービスは付録を参照してください。

```bash
sudo apt install socat -y

sudo tee /etc/systemd/system/socat-sftp.service << EOF
[Unit]
Description=socat SFTP forwarder
After=network.target openvpn@server.service

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:2222,fork,reuseaddr TCP:10.8.0.6:22
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable socat-sftp
sudo systemctl start socat-sftp
```

### 1-9. Webサーバーセットアップ

以下のコマンドでNode.jsとnpmパッケージをインストールしてください。

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt remove nodejs libnode-dev libnode72 -y
sudo apt install nodejs -y

mkdir -p ~/astro/public
cd ~/astro
npm init -y
npm install express express-session bcryptjs
```

`src/VPS/server.js` を `~/astro/server.js` にコピーしてください。
`src/VPS/favicon.webp` を `~/astro/favicon.webp` にコピーしてください。

> **⚠️ 重要**: `server.js` の **9行目** にある `'change me!!'` を任意のパスワードに変更してください。

#### SSL証明書の取得

Let's Encryptで証明書を取得してください。

```bash
sudo apt install certbot -y
sudo certbot certonly --standalone -d atlas.your-domain.com --email your@email.com --agree-tos --non-interactive
```

証明書は90日で期限切れになるため、自動更新を設定してください。

```bash
sudo crontab -e
```

以下を追加してください：

```
0 3 * * * certbot renew --quiet --pre-hook "systemctl stop skycluster-web" --post-hook "systemctl start skycluster-web"
```

以下のコマンドでsystemdサービスを作成し、起動してください。

```bash
sudo tee /etc/systemd/system/skycluster-web.service << EOF
[Unit]
Description=Skycluster Web Server
After=network.target

[Service]
User=root
WorkingDirectory=/home/ubuntu/astro
ExecStart=/usr/bin/node /home/ubuntu/astro/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable skycluster-web
sudo systemctl start skycluster-web
```

### 1-10. VPS→StellarMate SSH鍵設定

以下のコマンドでSSH鍵を生成してください。

```bash
ssh-keygen -t ed25519 -f ~/.ssh/stellarmate_key -N ""
```

生成した公開鍵をStellarMateの `~/.ssh/authorized_keys` に追加してください（Part 3参照）。

---

## Part 2: DNS設定

ドメインのDNS管理画面でAレコードを以下のように設定してください。

| ホスト名 | タイプ | VALUE |
|---|---|---|
| `@` | A | VPSのグローバルIP |
| `atlas` | A | VPSのグローバルIP |

`atlas.skycluster.jp` はWebコンソール・公開ページのアクセスに使用します。

---

## Part 3: StellarMate セットアップ

StellarMateへのSSH接続は以下のコマンドで行ってください。StellarMateと同じネットワーク（ローカル）に接続されている必要があります。

```bash
ssh -p 5624 stellarmate@stellarmate.local
```

IPアドレスで接続する場合は以下のように指定してください。

```bash
ssh -p 5624 stellarmate@<StellarMate IP>
```

デフォルトのパスワードは `stellarmate` です。

### 3-1. OpenVPNクライアント設定

VPSで生成した `stellarmate.ovpn` の内容（手順1-7参照）を以下のコマンドでStellarMate上に直接作成してください。

```bash
sudo mkdir -p /etc/openvpn/client
sudo nano /etc/openvpn/client/stellarmate.conf
```

エディタが開いたら、VPSで `cat ~/stellarmate.ovpn` を実行して表示された内容をそのまま貼り付けてください。保存後、以下のコマンドを実行してください。

```bash
# atomic updatesを一時無効化
sudo /etc/stellarmate/atomic-updates.sh --disable

sudo pacman -S openvpn inotify-tools --noconfirm

# atomic updatesを再有効化
sudo /etc/stellarmate/atomic-updates.sh --enable

sudo systemctl enable openvpn-client@stellarmate
sudo systemctl start openvpn-client@stellarmate
```

### 3-2. Pythonライブラリのインストール

以下のコマンドで必要なPythonライブラリをインストールしてください。

```bash
sudo /etc/stellarmate/atomic-updates.sh --disable
sudo pacman -S python-astropy python-pillow --noconfirm
sudo /etc/stellarmate/atomic-updates.sh --enable

pip3 install astroalign scikit-image opencv-python-headless --break-system-packages
```

### 3-3. スタック＆転送スクリプト配置

以下のコマンドで作業ディレクトリを作成してください。

```bash
mkdir -p ~/stacked
```

`src/StellarMate/stack_and_transfer.py` を `~/stack_and_transfer.py` にコピーしてください。
`src/StellarMate/watch_and_stack.sh` を `~/watch_and_stack.sh` にコピーしてください。

```bash
chmod +x ~/watch_and_stack.sh
```

### 3-4. StellarMate→VPS SSH鍵設定

以下のコマンドでSSH鍵を生成し、VPSに登録してください。

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vps_key -N ""
ssh-copy-id -i ~/.ssh/vps_key.pub ubuntu@skycluster.jp
```

また、VPSで生成した公開鍵（`~/.ssh/stellarmate_key.pub`）を `~/.ssh/authorized_keys` に追加してください。

```bash
echo "<VPSの公開鍵>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 3-5. 監視サービスの自動起動

以下のコマンドでsystemdサービスを作成し、起動してください。

```bash
sudo tee /etc/systemd/system/astro-stack.service << EOF
[Unit]
Description=Astro Stack and Transfer
After=network.target openvpn-client@stellarmate.service

[Service]
User=stellarmate
ExecStart=/home/stellarmate/watch_and_stack.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable astro-stack
sudo systemctl start astro-stack
```

### 3-6. ヘルスチェック・自動復旧の設定

`src/StellarMate/healthcheck.sh` を `~/healthcheck.sh` にコピーしてください。

VPN接続と監視プログラム (`astro-stack`) の死活監視を5分ごとに行い、停止していれば自動的に復旧します。ログは `~/healthcheck.log` に記録されます。

```bash
chmod +x ~/healthcheck.sh

# cronをインストール
sudo /etc/stellarmate/atomic-updates.sh --disable
sudo pacman -S cronie --noconfirm
sudo /etc/stellarmate/atomic-updates.sh --enable

sudo systemctl enable cronie
sudo systemctl start cronie

# 5分ごとに実行するよう登録
(crontab -l 2>/dev/null; echo "*/5 * * * * /home/stellarmate/healthcheck.sh") | crontab -
```

ログのローテーションを設定してください。これにより毎日自動でローテーションされ、7日分を保持して古いものは自動削除されます。

```bash
sudo tee /etc/logrotate.d/atlas-healthcheck << EOF
/home/stellarmate/healthcheck.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

ログを確認する場合は以下のコマンドを使用してください。

```bash
tail -f ~/healthcheck.log
```

---

## Part 4: 動作確認

| 確認項目 | URL/コマンド |
|---|---|
| INDI Web Manager | http://atlas.skycluster.jp:8624 |
| コンソールログイン | https://atlas.skycluster.jp/login |
| 公開ページ | https://atlas.skycluster.jp |
| VPN疎通確認 | `ping 10.8.0.1` (StellarMateから) |
| 手動スタックテスト | `python3 ~/stack_and_transfer.py <fits_path>` |
| ヘルスチェック手動実行 | `~/healthcheck.sh` |

---

## コンソール機能

| 機能 | 説明 |
|---|---|
| 公開/非公開 | セッション（天体）ごとに個別に切り替えられます。状態は再起動後も維持されます。 |
| EXCLUDE_FRAME | フレーム番号を指定して除外できます。最低1フレームは維持されます。スタック処理中はボタンが無効化されます。 |
| CLEAR_EXCLUDED & RESTACK | 除外リストをクリアして全フレームで再スタックします。処理中はロックされます。 |
| DELETE_THIS_PHOTOGRAPHY | そのセッションの画像とフォルダをVPSから完全に削除します。 |

---

## 注意事項

- atomic updatesを無効にしてパッケージをインストールした場合、OSアップデート後にパッケージが消える可能性があります。
- StellarMateのSSHポートは **5624** です。
- socatのフォワード先IPはStellarMateのVPN IP (`10.8.0.6`) に固定されています。変更した場合は各socatサービスファイルを更新してください。
- `server.js` の9行目のパスワードは必ず変更してください。

---

## 付録: INDI系ポートの開放（非推奨）

> **⛔ セキュリティリスクを十分に理解した上で実施してください。**
> INDI プロトコルには認証機能がなく、開放すると全世界から望遠鏡を制御できる状態になります。
> 使用後は必ず閉じることを推奨します。

### Oracle Cloud Security List への追加

VCN → Subnet → Security List → Ingress Rules に以下を追加してください。

| Protocol | Source CIDR | Port | 用途 |
|---|---|---|---|
| TCP | 0.0.0.0/0 | 3000 | StellarMate Dashboard |
| TCP | 0.0.0.0/0 | 7624 | INDI |
| TCP | 0.0.0.0/0 | 8624 | INDI Web Manager |

### OS側ファイアウォールの開放

```bash
sudo iptables -I INPUT -p tcp --dport 7624 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 8624 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 3000 -j ACCEPT
sudo netfilter-persistent save
```

### socatサービスの作成・起動

```bash
sudo tee /etc/systemd/system/socat-indi.service << EOF
[Unit]
Description=socat INDI forwarder
After=network.target openvpn@server.service

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:7624,fork,reuseaddr TCP:10.8.0.6:7624
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/socat-indiweb.service << EOF
[Unit]
Description=socat INDI Web Manager forwarder
After=network.target openvpn@server.service

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:8624,fork,reuseaddr TCP:10.8.0.6:8624
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/socat-stellarmate.service << EOF
[Unit]
Description=socat StellarMate Dashboard forwarder
After=network.target openvpn@server.service

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:3000,fork,reuseaddr TCP:10.8.0.6:3000
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable socat-indi socat-indiweb socat-stellarmate
sudo systemctl start socat-indi socat-indiweb socat-stellarmate
```

### 使用後の閉鎖

使用が終わったら以下のコマンドでサービスを停止し、ファイアウォールを閉じてください。

```bash
sudo systemctl stop socat-indi socat-indiweb socat-stellarmate
sudo systemctl disable socat-indi socat-indiweb socat-stellarmate

sudo iptables -D INPUT -p tcp --dport 7624 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 8624 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 3000 -j ACCEPT
sudo netfilter-persistent save
```

Oracle Cloud の Security List からも該当ルールを削除してください。
