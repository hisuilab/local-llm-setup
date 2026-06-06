# Local LLM Setup (Ollama + Open WebUI on Ubuntu)

Ubuntu 26.04 環境に、ローカルLLM実行環境である Ollama と、グラフィカルなウェブUIを提供する Open WebUI を構築し、Tailscale（VPN）経由で安全に外部からアクセス可能にするためのセットアップガイドです。

---

## サーバー環境構成

本ガイドは、以下のプライベート環境での動作実績に基づいています。

| 項目 | 構成・スペック |
| :--- | :--- |
| **HW** | 自作デスクトップPC |
| **OS** | Ubuntu 26.04 LTS |
| **CPU** | AMD Ryzen 5 5600X |
| **GPU** | NVIDIA GeForce RTX 3070 (VRAM 8GB) |
| **RAM** | DDR4 64GB |
| **ネットワーク** | Tailscale VPN (SSH / MagicDNS 有効) |

---

## 環境構築手順

### Step 1: Ollama のインストールと初期確認

ホストOS（Ubuntu）に Ollama を直接インストールします。

1. **Ollamaのインストールスクリプトを実行**
   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```

2. **Ollama サービスのステータス確認**
   ```bash
   sudo systemctl status ollama
   ```

3. **リアルタイムログの確認**
   ```bash
   sudo journalctl -u ollama -f -n 20
   ```

4. **モデルのテスト起動**
   軽量なLLM（ここでは `qwen2.5:3b` を例にします）をダウンロードして動作を確認します。
   ```bash
   ollama run qwen2.5:3b
   ```

   > [!NOTE]
   > 起動すると対話モードになります。終了するには `/exit` と入力します。

---

### Step 2: Ollama の外部アクセス許可

初期状態では、Ollamaは `127.0.0.1`（ローカルホスト）からの接続のみを受け付けます。Dockerコンテナや外部デバイス（Tailscale）から接続できるように、リッスンアドレスを `0.0.0.0` に変更します。

1. **systemd サービス設定の編集**
   ```bash
   sudo systemctl edit ollama.service
   ```
   エディタが開くので、上部のコメント行の間に以下の記述を追加して保存します。
   ```ini
   [Service]
   Environment="OLLAMA_HOST=0.0.0.0"
   ```

2. **設定の反映とサービスの再起動**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart ollama
   ```

3. **サービスの確認**
   ```bash
   sudo systemctl status ollama
   ```

4. **ネットワーク経由での接続テスト**
   別の同一ネットワーク（またはTailscale経由の別端末）から、MagicDNSやTailscale IPを指定してアクセスできるか確認します。
   ```bash
   # 例: 自身のホスト名が 'hisuilab-ubuntu-dt' の場合
   curl http://hisuilab-ubuntu-dt:11434/
   ```
   以下のように出力されれば成功です。
   ```text
   Ollama is running
   ```

---

### Step 3: Docker と NVIDIA Container Toolkit のセットアップ

Open WebUI をコンテナで動かすために Docker を用意し、さらにコンテナ内から GPU を利用できるよう NVIDIA Container Toolkit を導入します。

1. **Docker のインストール**
   [Docker Engine 公式ドキュメント (Ubuntu)](https://docs.docker.com/engine/install/ubuntu/) に従って Docker をインストールします。

2. **sudo なしで Docker コマンドを実行できるようにユーザーを追加**
   ```bash
   sudo usermod -aG docker $USER
   ```

   > [!IMPORTANT]
   > 設定を反映させるために、一度 **ログアウトしてログインし直す** か、または `newgrp docker` コマンドを実行してください。これを怠ると、Dockerコマンドの実行に `sudo` が必要になります。

3. **NVIDIA Container Toolkit のインストール**
   コンテナ側でGPUを利用可能にするために必要です。
   ```bash
   # リポジトリの登録
   curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
     && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
       sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
       sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

   # パッケージの更新とインストール
   sudo apt-get update
   sudo apt-get install -y nvidia-container-toolkit

   # Dockerの設定と再起動
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

---

### Step 4: Docker Compose による Open WebUI の構築

本リポジトリに含まれる `docker-compose.yml` を利用して、Open WebUI をバックグラウンドで起動します。

1. **コンテナのビルド・起動**
   ```bash
   docker compose up -d
   ```

2. **接続確認**
   Tailscale 接続された任意のデバイスのブラウザから、以下のURLにアクセスします。
   ```text
   http://hisuilab-ubuntu-dt:3000
   ```
   初回アクセス時に管理者アカウントの登録画面が表示されれば、環境構築は完了です。

   > [!WARNING]
   > 初回に作成したユーザーが管理者権限を持ちます。パブリックなTailscale環境や複数人が共有する環境にデプロイする場合は、速やかに管理ユーザーを作成し、不要なサインアップ設定をOpen WebUI上の管理パネルから無効化することを推奨します。

---

## 補足事項

### docker-compose.yml のネットワーク設定について

本リポジトリの `docker-compose.yml` では、ホスト側で動作している Ollama を参照するために以下の設定を採用しています。

```yaml
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

* **`host.docker.internal`**: Dockerコンテナ内からホストマシン（Ubuntu）自体を指す特別なホスト名です。
* **`host-gateway`**: コンテナゲートウェイのIPアドレス（通常 `172.17.0.1` 等）を自動的にマッピングしてくれます。

> [!NOTE]
> これにより、コンテナ内の Open WebUI からホストOS上の Ollama サービスに安全かつスマートにアクセス可能となっています。

---

## トラブルシューティング

### Open WebUI でモデルが認識されない場合（UFW設定）

Ubuntu のファイアウォール（UFW）がアクティブになっている場合、コンテナ内部からホスト上の Ollama サービス（11434ポート）への通信がデフォルトでブロックされます。

#### 1. 通信が遮断されているかの検証
ホストマシンのターミナルから、コンテナ内部を介してOllama APIへの疎通を確認します。
```bash
docker compose exec open-webui curl http://host.docker.internal:11434/api/tags
```
応答が返ってこない（タイムアウトする）場合は、UFWによってブロックされています。

#### 2. UFW のルール追加による解決
安全性を維持したまま、Dockerコンテナからのアクセスのみを許可するルールを追加します。
```bash
# Dockerの内部ネットワーク（デフォルト 172.16.0.0/12）からホストの11434ポートへの通信を許可
sudo ufw allow from 172.16.0.0/12 to any port 11434 proto tcp
```

> [!IMPORTANT]
> このルールを適用することで、外部（インターネット）に対してOllamaポートを公開することなく、ローカルのコンテナからのみ安全に接続できるようになります。
