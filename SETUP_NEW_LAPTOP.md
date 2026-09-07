# Dựng lại môi trường GRC Assistant trên máy mới

Viết ngày 2026-09-07, trước khi đổi laptop công ty. Checklist này giả định máy mới là macOS.

---

## Phần 0 — LÀM TRƯỚC KHI RỜI MÁY CŨ

Những thứ dưới đây **không nằm trong git**. Clone lại repo sẽ KHÔNG có chúng.

### 0.1 Secrets (bắt buộc — không thể tạo lại)

Đưa 3 file này vào **password manager hoặc kho secret của công ty**. Không commit, không gửi qua chat/email.

| File | Nội dung |
|---|---|
| `.env` | 52 biến, gồm `AI_PLATFORM_API_KEY`, `MICROSOFT_APP_PASSWORD`, `MS_CLIENT_SECRET`, `GREENNODE_CLIENT_SECRET` |
| `.env.deploy` | bản dùng cho deploy (= `.env` trừ một số biến, xem `DEPLOYMENT.md`) |
| `.greennode.json` | `client_id` + `client_secret` của GreenNode |

Nếu mất `.env`, phải xin/tạo lại từng cái: API key VNG MaaS, secret của Azure Bot, secret của SharePoint Graph app, credentials GreenNode. Rất mất thời gian.

### 0.2 SSH key

```bash
ls ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

Key này đang dùng cho các repo khác qua SSH (`git@github.com:…`). Hai lựa chọn:

- **Copy** cặp key sang máy mới (giữ `chmod 600` cho private key), hoặc
- **Tạo key mới** trên máy mới rồi thêm public key vào GitHub → Settings → SSH and GPG keys.

### 0.3 Dữ liệu tải lại được nhưng mất thời gian

| Thư mục | Dung lượng | Ghi chú |
|---|---|---|
| `sharepoint_downloads/ISMS-Docs/` | ~30 MB | Tài liệu ISMS gốc. Tải lại được bằng `sharepoint_sync.py` (cần creds SharePoint). |
| `vector_db/` | ~4 MB | FAISS index (1664 chunk). Rebuild được bằng `ingest.py`, mất thời gian + tốn embedding. |

Copy sang ổ ngoài sẽ nhanh hơn nhiều so với ingest lại.

### 0.4 Memory của Claude Code

```
~/.claude/projects/-Users-lap13986-grc-assistant/memory/
```

4 file ghi lại các gotcha đã trả giá đắt: cấu hình deploy, auth Teams bot, insight về archive-pollution trong retrieval. Copy cả thư mục.

### 0.5 File rời không nằm trong repo nào

- `~/Downloads/ai-chat-agent-claw-a-thon/ZION_ISMS_2026_CEO_Review.pptx` — untracked, sẽ mất.
- Kiểm tra lại 2 repo khác trước khi rời máy:

```bash
git -C ~/Documents/clawathon status --short && git -C ~/Downloads/ai-chat-agent-claw-a-thon status --short
```

### 0.6 Đảm bảo đã push hết

```bash
git -C ~/grc-assistant status -sb
```

Phải thấy `## main...origin/main` **không kèm** `[ahead N]`.

---

## Phần 1 — CÀI TRÊN MÁY MỚI

### 1.1 Công cụ nền

```bash
xcode-select --install
```

Cài Homebrew (nếu chưa có), rồi:

```bash
brew install nvm git
```

Thêm nvm vào `~/.zshrc`:

```bash
mkdir -p ~/.nvm && printf '\nexport NVM_DIR="$HOME/.nvm"\n[ -s "$(brew --prefix nvm)/nvm.sh" ] && \\. "$(brew --prefix nvm)/nvm.sh"\n' >> ~/.zshrc
```

Mở terminal mới, cài Node LTS:

```bash
nvm install --lts && nvm alias default 'lts/*'
```

Docker Desktop: tải từ docker.com, cài, mở app, bật *Start Docker Desktop when you sign in*.

### 1.2 Git identity

Trên máy cũ chưa set nên commit bị gán sai tác giả. Set ngay trên máy mới:

```bash
git config --global user.name "ttkt12" && git config --global user.email "thittk2@vng.com.vn"
```

### 1.3 GitHub CLI

```bash
brew install gh && gh auth login
```

### 1.4 Clone repo

Tên thư mục **phải là `grc-assistant`** — AgentBase lấy tên app từ tên thư mục, và các doc trong repo đều giả định đường dẫn này.

```bash
cd ~ && git clone https://github.com/ttkt12/grc-assistant.git && cd grc-assistant
```

### 1.5 Python

Máy cũ dùng Python 3.9.6 (bản kèm macOS). Nên dùng bản mới hơn + venv riêng:

```bash
brew install python@3.12
```

```bash
cd ~/grc-assistant && python3.12 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt
```

Nếu gặp lỗi phiên bản, có `constraints.txt`:

```bash
pip install -r requirements.txt -c constraints.txt
```

### 1.6 Restore secrets và dữ liệu

Copy vào thư mục gốc repo: `.env`, `.env.deploy`, `.greennode.json`, `vector_db/`, `sharepoint_downloads/`.

Copy memory Claude Code vào `~/.claude/projects/-Users-lap13986-grc-assistant/memory/` (tên thư mục theo đường dẫn repo trên máy mới — nếu username khác `lap13986` thì tên thư mục sẽ khác theo).

### 1.7 Nếu không có bản backup vector_db

```bash
python3 sharepoint_sync.py
```

```bash
python3 ingest.py
```

Xem `LOCAL_KNOWLEDGE_UPDATE.md` để biết chi tiết. Lưu ý `ingest.py` loại thư mục `Archives/` và dedupe theo tên file — đừng bỏ bước này, index bị lẫn bản 2022-2023 sẽ làm câu trả lời sai.

---

## Phần 2 — KIỂM TRA HOẠT ĐỘNG

```bash
python3 predeploy_check.py
```

Chạy app local:

```bash
python3 teams_bot.py
```

Khởi động chậm (~60-90s vì phải load FAISS index). Sau đó kiểm tra:

```bash
curl -s localhost:3978/health && curl -s localhost:3978/documents/count
```

Kỳ vọng: `{"status": "ok"}` và `total_documents: 52`.

Mở http://localhost:3978 — tiêu đề phải là **GRC Assistant — Zalopay Compliance / GRC**.

---

## Phần 3 — LƯU Ý VỀ DEPLOY

Tính tới 2026-09-07, deploy đang chạy trên **GreenNode AgentBase**, nhưng kế hoạch là **chuyển hẳn sang hạ tầng Zalopay**. Xác nhận lại target trước khi deploy.

Các ràng buộc dưới đây thuộc về chính app, mang theo được sang bất kỳ hạ tầng nào:

1. **Container phải listen port 8080.** `.env.deploy` **không được chứa** `PORT` hoặc `TEAMS_BOT_PORT` — app đọc 2 biến này và sẽ listen sai port, health check fail.
2. **Biến auth Teams bot** phải có giá trị thật (không phải placeholder `your_*`): `MICROSOFT_APP_ID`, `MICROSOFT_APP_PASSWORD`, `MICROSOFT_APP_TENANT_ID`, `MICROSOFT_APP_TYPE=SingleTenant`.
3. `MS_CLIENT_ID` / `MS_CLIENT_SECRET` là **app SharePoint Graph, KHÔNG phải bot** — đừng dùng lẫn.
4. `vector_db/` được `COPY` vào Docker image, nên máy build phải có index sẵn.

Chi tiết đường deploy GreenNode (sẽ bỏ dần): xem `DEPLOYMENT.md`.

### Tên tài nguyên phía nền tảng vẫn là brand cũ

Repo đã rebrand SecureMind RAG → GRC Assistant, nhưng các tài nguyên đã đăng ký thì **giữ nguyên tên cũ có chủ ý** — đổi sẽ làm hỏng chúng:

- Runtime GreenNode: `securemind-rag`
- Azure Bot: `SecureMind_RAG`
- Teams manifest `id` và `botId` (GUID) — tuyệt đối không đổi, nếu đổi thì app đã cài trong Teams sẽ hỏng.
