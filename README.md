# Ollama + Open WebUI + SearXNG on Ubuntu Server

Ubuntu Server 上で `ollama`、`Open WebUI`、`SearXNG` を Docker Compose でまとめて動かす構成です。

確認しているイメージタグ:

- `ollama/ollama:0.18.2`
- `ghcr.io/open-webui/open-webui:v0.8.10`
- `ghcr.io/searxng/searxng:2026.3.17-2bb8ac17c`

構成の役割:

- `ollama`: ローカル LLM 実行
- `open-webui`: ブラウザ UI
- `searxng`: Web 検索エンジン

## 1. 前提

- OS: Ubuntu Server
- Docker Engine と Docker Compose Plugin が入っていること
- 使用ポート:
  - `3000`: Open WebUI
  - `8080`: SearXNG
  - `11434`: Ollama API

## 2. Docker のインストール

未導入なら以下を実行します。

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"
```

`usermod` 後はログインし直してください。

## 3. 配置するファイル

このディレクトリには次のファイルがあります。

- `docker-compose.yml`
- `docker-compose.rx570.yml`
- `README.md`
- `searxng/settings.yml`

例:

```bash
mkdir -p ~/ai-stack/searxng
cd ~/ai-stack
```

## 4. 通常起動

```bash
docker compose up -d
```

確認:

```bash
docker compose ps
docker compose logs -f
```

アクセス先:

- Open WebUI: `http://<server-ip>:3000`
- SearXNG: `http://<server-ip>:8080`
- Ollama API: `http://<server-ip>:11434`

## 5. モデル追加

```bash
docker compose exec ollama ollama pull llama3.1
docker compose exec ollama ollama list
```

## 6. Open WebUI と SearXNG の連携

この構成では Open WebUI から SearXNG へ以下の URL で検索します。

```text
http://searxng:8080/search?q=<query>&format=json
```

`docker-compose.yml` では次を設定済みです。

- `WEBUI_SECRET_KEY=change-this-webui-secret-key`
- `ENABLE_RAG_WEB_SEARCH=True`
- `RAG_WEB_SEARCH_ENGINE=searxng`
- `SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>&format=json`

本番利用前に `WEBUI_SECRET_KEY` は変更してください。

## 7. SearXNG の基本設定

`searxng/settings.yml` で次を調整できます。

- `server.secret_key`
- `search.safe_search`
- 利用する検索エンジン

変更後は再起動します。

```bash
docker compose restart searxng
```

## 8. RX570 向けの使い方

RX570 は AMD Polaris 世代です。2026-03-20 時点で、Ollama の公式ハードウェア対応表に `RX 570` は載っていません。Linux の AMD GPU 公式対応として列挙されているのは `RX 5500 XT` 以上の一部で、`RX 570` は ROCm の正式対応外です。

そのため、この構成では RX570 用に `docker-compose.rx570.yml` を追加し、ROCm ではなく Ollama の `Vulkan` 実験機能を使う前提にしています。

重要:

- `RX570で必ず動く` ことを保証する設定ではありません
- `ROCm正式対応` ではなく `Vulkan実験機能` を使います
- うまくいかない場合は CPU 実行にフォールバックする可能性があります

### RX570 手順

1. GPU が見えているか確認

```bash
lspci | grep VGA
ls -l /dev/kfd
ls -l /dev/dri
```

2. Vulkan 関連を入れる

```bash
sudo apt update
sudo apt install -y mesa-vulkan-drivers vulkan-tools
```

3. ホスト側で Vulkan が見えるか確認

```bash
vulkaninfo | less
```
3. 設定

- まず `docker compose logs -f ollama` を確認
- `docker compose exec ollama env | grep OLLAMA` で `OLLAMA_VULKAN=1` を確認
- コンテナ内のデバイス確認:

```bash
docker compose exec ollama ls -l /dev/kfd
docker compose exec ollama ls -l /dev/dri
```

コンテナ内のデバイスを確認したら多分数字が出てくると思うのでそこをdocker-compose.rx570.ymlの中身確認して編集をしていきます。

もし出ない場合
```bash
getent group render
getent group video
```
こっちで確認します。
ここに出てきた数字を入れていきます。
僕の場合は、こんな感じに出てきました。

```log
render:x:993:ユーザー名
video:x:44:ユーザー名

```
これをdocker-compose.rx570.ymlの中を書き換えていきます。

```bash
    group_add:
      - video
      - render
```
僕の場合は、renderを消して"993"videoを"44"に変えていきます。
この数字は、環境によって変わるので必ず確認してください。

- それでも GPU を使えない場合は、RX570 が Ollama の Vulkan 実装で安定しない可能性があります

4. コンテナを RX570 用 override 付きで起動

```bash
docker compose -f docker-compose.yml -f docker-compose.rx570.yml up -d
```

5. Ollama ログを確認

```bash
docker compose logs -f ollama
```

6. モデルを取得して動作確認

```bash
docker compose exec ollama ollama pull llama3.2:3b
docker compose exec ollama ollama run llama3.2:3b
```


10. データ保存

- `ollama` モデル: Docker volume `ollama`
- `open-webui` データ: Docker volume `open-webui`

`docker compose down` では volume は消えません。

## 11. 動作確認コマンド

SearXNG 確認:

```bash
curl "http://localhost:8080/search?q=test&format=json"
```

Ollama API 確認:

```bash
curl http://localhost:11434/api/tags
```

## 12. 参考にした公式情報

- Ollama Docker: https://docs.ollama.com/docker
- Ollama Hardware support: https://docs.ollama.com/gpu
- Ollama Linux install: https://docs.ollama.com/linux
- AMD ROCm system requirements: https://rocm.docs.amd.com/projects/install-on-linux/en/docs-7.1.1/reference/system-requirements.html


