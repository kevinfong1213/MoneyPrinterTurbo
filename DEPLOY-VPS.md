# MoneyPrinterTurbo — VPS 自架部署

Docker + nginx 反代 + HTTPS + 密碼保護。由零到出到第一條片。

**部署形態**:Docker 行兩個容器(WebUI `:8501`、API `:8080`),兩個都**只綁 `127.0.0.1`**;
對外一律經宿主機嘅 nginx,由 nginx 做 TLS 同密碼。公網任何時候都掂唔到 8501/8080。

---

## 0. 部機要幾大?

渲染影片係 CPU 密集 + 食 RAM,唔係「開得起就用得」:

| 規格 | 實際體感 |
|---|---|
| 1 core / 1G | **唔好諗**,pip 裝依賴嗰陣就已經 OOM |
| 2 core / 4G | 起到,短片(30–60 秒)可以,但慢 |
| 4 core / 8G | 舒服,建議由呢度起步 |

硬碟預 **20G 以上**:素材片段會 cache,`storage/` 會不停脹。

---

## 1. 裝底層

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 nginx apache2-utils certbot python3-certbot-nginx
sudo systemctl enable --now docker nginx
sudo usermod -aG docker $USER      # 之後要登出再登入先生效
```

## 2. 攞 code

```bash
clone 你自己個 fork(部署檔已經喺入面,唔使再搬):

```bash
git clone https://github.com/kevinfong1213/MoneyPrinterTurbo.git
cd MoneyPrinterTurbo

# 掛返上游,日後可以拉佢啲更新
git remote add upstream https://github.com/harry0703/MoneyPrinterTurbo.git
```

## 3. 填 config.toml

```bash
cp config.example.toml config.toml
nano config.toml
```

至少要填兩樣,少一樣都出唔到片:

1. **一個 LLM provider** —— 預設 `llm_provider = "moonshot"`,填 `moonshot_api_key`。
   想用第個就改 `llm_provider`,再填對應嗰條 key(`openai_api_key` / `gemini_api_key` /
   `deepseek_api_key` / `qwen_api_key` … 十幾個揀)。
2. **素材源** —— `pexels_api_keys = ["你張key"]`(免費申請,https://www.pexels.com/api/)。
   Pixabay 都得。**冇呢個攞唔到片段,generate 會直接失敗。**

## 4. 開 .env

部署檔已經喺 repo 入面(`docker-compose.prod.yml`、`nginx/moneyprinterturbo.conf`),
剩返開個 `.env`:

```bash
cp .env.example .env
nano .env                     # PUBLIC_HOST 填你嘅真域名,一定要改
```

`PUBLIC_HOST` 冇填嘅話 compose 會**故意唔肯起**(`:?` 語法),唔會俾你部署咗先發現網址錯。

## 5. 起容器

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
docker compose ps
docker compose logs -f webui
```

> 第一次 build 要拉 Python 依賴 + ffmpeg,**十幾二十分鐘好正常**,唔好以為死咗機。

本機先確認通:

```bash
curl -I http://127.0.0.1:8501/_stcore/health     # 預期 200
```

## 6. 設密碼(唔好跳)

```bash
sudo htpasswd -c /etc/nginx/.htpasswd-mpt kevin      # 會叫你打兩次密碼
```

WebUI 零認證,呢一步係**唯一**擋住陌生人用你 API key 嘅嘢。

## 7. nginx + HTTPS

```bash
sudo cp nginx/moneyprinterturbo.conf /etc/nginx/sites-available/
sudo sed -i 's/mpt.example.com/你嘅域名/g' /etc/nginx/sites-available/moneyprinterturbo.conf
sudo ln -s /etc/nginx/sites-available/moneyprinterturbo.conf /etc/nginx/sites-enabled/
```

**先出證書再 `nginx -t`** —— 順序掉轉會因為證書檔未存在而 fail:

```bash
sudo certbot --nginx -d 你嘅域名
sudo nginx -t && sudo systemctl reload nginx
```

部機冇 IPv6 嘅話,`nginx -t` 會報 `[::]:80 ... Address family not supported`,
刪走兩行 `listen [::]:...` 就得。

## 8. 防火牆

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

⚠️ **唔好**開 8501 / 8080 —— 開咗就繞過晒 nginx 個密碼,前面做嘅嘢全部白費。

## 9. 驗收

```bash
curl -I https://你嘅域名/                # 預期 401(冇密碼入唔到 = 保護生效)
curl -I -u kevin:密碼 https://你嘅域名/   # 預期 200
```

瀏覽器開 `https://你嘅域名/`,打密碼,見到 WebUI 就成功。試出一條 30 秒短片行完成個流程。

---

## 日常維運

```bash
# 睇 log
docker compose logs -f webui

# 拉自己 fork 嘅改動
git pull
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# 同步上游(harry0703)新功能
git fetch upstream
git merge upstream/main          # 部署檔係新增檔案,正常唔會有衝突
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# 清舊片(storage 脹得好快)
du -sh storage/
```

## 撞板對照

| 症狀 | 原因 |
|---|---|
| 頁面不停 "Connecting…" | nginx 少咗 `Upgrade` / `Connection "upgrade"`,Streamlit 靠 WebSocket |
| 出片出到一半 504 | `proxy_read_timeout` 太短(kit 已設 3600s) |
| 跳轉走去 127.0.0.1 | `.env` 嘅 `PUBLIC_HOST` 冇填啱 |
| 成部機卡死、SSH 都登唔入 | 冇設資源上限,渲染食晒 CPU。調細 `.env` 嘅 `WEBUI_CPUS` |
| generate 即刻失敗 | `pexels_api_keys` 空,或者 LLM key 錯 |
| `nginx: unknown directive "http2"` | nginx <1.25 要用 `listen 443 ssl http2;`(kit 已經係呢個寫法) |
