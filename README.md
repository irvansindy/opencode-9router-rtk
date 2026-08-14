# Dev AI Environment Setup — OpenCode + 9Router + Ollama + RTK

Dokumentasi setup lengkap AI-powered development environment di Ubuntu 24.04 LTS.
Ditulis berdasarkan pengalaman setup nyata (termasuk semua error yang ditemui), supaya
kalau perlu install ulang (laptop baru / reinstall OS), tidak perlu ulang debug dari nol.

## Target Environment

- OS: Ubuntu 24.04 LTS
- Hardware referensi: Intel Core i7 Gen 11, RAM 16GB DDR4 (CPU-only, tanpa GPU diskrit)
- Shell: zsh (sesuaikan ke `~/.bashrc` kalau pakai bash)

## Arsitektur

```
OpenCode (CLI + VS Code extension)
   ├──> 9Router (localhost:20128, systemd service) ──> Kiro AI / OpenCode Free / Vertex AI (gratis, online)
   ├──> Ollama (localhost:11434) ──> qwen2.5-coder, llama3.1 (offline, lokal)
   └──> RTK plugin ──> kompres output shell command (git, ls, test runner, dst) sebelum masuk context model
```

---

## 1. Install OpenCode

```bash
curl -fsSL https://opencode.ai/install | bash
opencode --version
```

---

## 2. Install & Jalankan 9Router

### 2.1 Install

```bash
npm install -g 9router
```

### 2.2 ⚠️ PENTING — Flag yang Wajib Dipakai

9Router itu CLI **interaktif**: begitu server nyala, dia menampilkan menu pilihan
(Web UI / Terminal UI / Hide to Tray / Exit) yang menunggu keyboard. Kalau dijalankan
lewat systemd/pm2 tanpa TTY asli, dia langsung dapat EOF di stdin dan keluar bersih
(`exit code 0`) — kelihatan seperti gagal padahal sebenarnya cuma nunggu jawaban menu
yang tidak pernah datang.

**Solusinya: pakai flag `-t` (tray/background mode)** — ini skip menu itu sepenuhnya.

```bash
9router --help
```

Flag yang relevan:
| Flag | Fungsi |
|---|---|
| `-t, --tray` | Jalan headless di background, skip menu interaktif (**wajib untuk service**) |
| `-n, --no-browser` | Jangan coba buka browser otomatis |
| `--skip-update` | Skip auto-update check saat start |
| `-H, --host <host>` | Default `0.0.0.0` (exposed ke jaringan lokal!). Pakai `127.0.0.1` kalau mau localhost-only |
| `-p, --port <port>` | Default `20128` |

Test manual dulu sebelum dijadikan service:

```bash
9router -t --skip-update -n
```

Kalau muncul `Router is now running in system tray` tanpa nyangkut menu, lanjut ke service.

### 2.3 Jadikan Systemd Service (Auto-start Setelah Reboot)

**Kenapa systemd, bukan pm2?** Pm2 fork mode juga tidak menyediakan TTY asli,
sehingga 9router tetap kena masalah sama seperti di atas. Dengan flag `-t` masalah
itu sudah tidak relevan lagi, tapi systemd tetap dipilih karena lebih native untuk
Ubuntu dan lebih mudah dikontrol proteksi OOM-nya (lihat 2.4).

Cari dulu path node dan 9router:
```bash
which node
which 9router
```

Buat service file:
```bash
sudo nano /etc/systemd/system/9router.service
```

Isi (**sesuaikan path node/9router dan `User` dengan hasil `which` di atas**):

```ini
[Unit]
Description=9Router AI Proxy
After=network.target

[Service]
Type=simple
User=ictsindy
WorkingDirectory=/home/ictsindy
Environment=PATH=/home/ictsindy/.nvm/versions/node/v24.19.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/home/ictsindy/.nvm/versions/node/v24.19.0/bin/node /home/ictsindy/.nvm/versions/node/v24.19.0/bin/9router -t --skip-update -n
Restart=on-failure
RestartSec=5
OOMScoreAdjust=-500
ManagedOOMMemoryPressureLimit=0%

[Install]
WantedBy=multi-user.target
```

Catatan tiap baris penting:
- `Environment=PATH=...` — **wajib**, karena systemd tidak mewarisi PATH shell interaktif kamu.
  Tanpa ini, error yang muncul adalah `env: 'node': No such file or directory` (status 127).
- `ExecStart` memanggil `node` secara eksplisit (bukan langsung `9router`) — supaya tidak
  bergantung pada shebang `#!/usr/bin/env node` yang bisa gagal resolve di environment systemd.
- `OOMScoreAdjust=-500` + `ManagedOOMMemoryPressureLimit=0%` — melindungi 9router dari
  `systemd-oomd` (userspace OOM killer Ubuntu) yang bisa membunuhnya kalau RAM sistem mepet.
  Ini penting di RAM 16GB kalau banyak aplikasi lain (Docker Desktop, Chrome, VS Code) jalan bareng.

Aktifkan:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now 9router.service
sudo systemctl status 9router.service
curl http://localhost:20128/v1/models
```

Pastikan statusnya `active (running)`, bukan `activating (auto-restart)`.

### 2.4 Verifikasi Tahan Reboot

```bash
sudo reboot
```

Setelah nyala lagi, tanpa buka terminal 9router manual apapun:
```bash
sudo systemctl status 9router.service
curl http://localhost:20128/v1/models
```

---

## 3. Connect Provider Gratis di Dashboard 9Router

Buka `http://localhost:20128` (atau `http://localhost:20128/dashboard`) → **Providers**:

| Provider | Cara connect | Catatan |
|---|---|---|
| **Kiro AI** | OAuth (AWS Builder ID / Google / GitHub), tanpa API key | ±50 kredit gratis/bulan |
| **OpenCode Free** | Klik connect langsung | Auto-fetch model dari opencode.ai |
| **Vertex AI** | Upload Service Account JSON dari GCP | Dari kredit gratis $300 akun GCP baru |

### 3.1 Buat Combo (Auto-Fallback)

Dashboard → **Combos** → Create New. Susun model dari prioritas tertinggi ke fallback:

```
Nama: auto-fallback
1. Model kualitas terbaik (misal Llama 3.1 70B)
2. Model backup (misal Qwen 32B)
3. Model paling longgar limitnya (misal Gemini Flash Lite)
```

9Router otomatis pindah ke baris berikutnya kalau provider kena rate limit/quota habis.
Pilih `auto-fallback` sebagai model di OpenCode supaya tidak perlu ganti model manual
tiap kali kena limit.

---

## 4. Hubungkan OpenCode ke 9Router

Edit `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["@vheins/opencode-9router@latest"]
}
```

Restart OpenCode total (quit penuh dari tray, bukan cuma tutup window), lalu cek `/models`
di dalam OpenCode — model 9Router harus muncul otomatis.

**Kalau tidak muncul:** plugin cache selama 3 jam. Hapus cache dan restart:
```bash
rm -rf ~/.cache/opencode-9router/
```

---

## 5. AI Offline dengan Ollama

Untuk hardware CPU-only 16GB RAM, model 7B–8B kuantisasi Q4 adalah sweet spot
(pakai RAM ~5-6GB saat jalan, sisa cukup untuk aplikasi lain).

```bash
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama   # installer resmi Ollama otomatis setup sebagai service

ollama pull qwen2.5-coder:7b      # ±4.7GB, model coding utama
ollama pull qwen2.5-coder:1.5b    # ±2GB, model ringan/cepat
ollama pull llama3.1:8b           # ±4.9GB, general reasoning
```

**Batasan realistis:** jangan coba model 13B+ di CPU-only 16GB RAM — RAM ketat dan
inference lambat. Kecepatan token 7B di CPU i7 Gen 11 sekitar 5-15 token/detik.

Tambahkan ke `opencode.json` (gabung dengan plugin 9Router, **jangan sampai menimpa**):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["@vheins/opencode-9router@latest"],
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "http://localhost:11434/v1" },
      "models": {
        "qwen2.5-coder:7b": {},
        "qwen2.5-coder:1.5b": {},
        "llama3.1:8b": {}
      }
    }
  }
}
```

Restart OpenCode, cari model di `/models` dengan ketik nama model (misal `qwen2.5-coder`),
bukan kata "ollama" — 9Router juga punya model dengan prefix nama "ollama/..." (cloud-hosted)
yang bisa bikin bingung karena namanya mirip.

---

## 6. Integrasi VS Code

Buka **Integrated Terminal** VS Code (`Ctrl+`` `), jalankan:

```bash
opencode
```

Extension VS Code ter-install otomatis (identifier: `sst-dev.opencode`). Config dan
provider yang sudah di-setup (9Router + Ollama) otomatis kepakai, tidak perlu setup ulang.

| Shortcut | Fungsi |
|---|---|
| `Ctrl+Esc` | Buka OpenCode di split terminal |
| `Ctrl+Shift+Esc` | Mulai sesi baru |
| `Ctrl+Alt+K` | Insert referensi file ke prompt |

Catatan: ini bukan autocomplete inline seperti Copilot — OpenCode berbasis chat/agent.

---

## 7. RTK — Efisiensi Token

RTK memampatkan output command shell (`git status`, `ls`, test runner, dst) sebelum
masuk ke context window model — menghemat token, terutama berguna untuk kuota gratis
9Router dan context window model Ollama lokal yang terbatas.

```bash
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc   # sesuaikan shell
source ~/.zshrc

rtk --version
rtk init -g --opencode
```

`rtk init -g --opencode` menaruh file plugin di `~/.config/opencode/plugins/rtk.ts`.
**OpenCode otomatis load semua file di folder `plugins/`** — tidak perlu didaftarkan
manual di `opencode.json`, dan tidak akan menimpa plugin 9Router yang sudah ada.

Restart OpenCode total, test dari dalam folder git project:
```
tolong cek git status di project ini
```

Cek hasilnya:
```bash
rtk gain              # statistik global
rtk gain --graph       # grafik 30 hari
rtk discover --all --since 30   # cari command yang belum ke-cover
```

---

## 8. Troubleshooting Umum

| Gejala | Penyebab | Solusi |
|---|---|---|
| 9router exit langsung, `code=exited status=0` | Menu interaktif nunggu keyboard, stdin EOF | Pakai flag `-t` |
| `env: 'node': No such file or directory` (status 127) | Systemd tidak punya PATH ke node/nvm | Tambah `Environment=PATH=...` di service file |
| `code=killed, signal=KILL` berulang | systemd-oomd / OOM killer, RAM sistem mepet | `OOMScoreAdjust=-500` + `ManagedOOMMemoryPressureLimit=0%`, dan turunkan alokasi RAM Docker Desktop |
| Model 9Router/Ollama tidak muncul di `/models` | Plugin cache 3 jam, atau config `opencode.json` ke-overwrite | `rm -rf ~/.cache/opencode-9router/`, cek isi `opencode.json` masih ada `plugin` + `provider` |
| pm2 mati setelah reboot | Proses jalan manual, bukan lewat `pm2 start`, atau `pm2 startup` belum dieksekusi | Sudah tidak relevan — 9router dipindah ke systemd |
| Cari "ollama" di `/models` malah muncul model 9Router | 9Router punya model cloud dengan prefix nama "ollama/..." | Cari nama model spesifik yang sudah di-pull (misal `qwen2.5-coder`), bukan kata "ollama" |

---

## 9. Perawatan Rutin (Disarankan)

```bash
free -h                          # cek RAM, kalau available < 2-3Gi, tutup aplikasi berat
ps aux --sort=-%mem | head -10   # cari proses paling boros RAM

sudo systemctl status 9router.service   # pastikan masih active (running)
rtk gain                                # cek penghematan token mingguan
```

Kalau RAM sering mepet: turunkan alokasi Docker Desktop VM (Settings → Resources →
Memory, dari 6GB ke 3-4GB), atau pertimbangkan upgrade RAM fisik ke 32GB kalau
workload AI-nya makin berat.

---

## 10. File Penting yang Di-backup di Repo Ini

- `opencode.json` — config provider (9Router plugin + Ollama)
- `9router.service` — systemd service file
- File ini (`README.md`)

Kalau install ulang OS / laptop baru: copy 2 file config tersebut ke path aslinya
(`~/.config/opencode/opencode.json` dan `/etc/systemd/system/9router.service`),
sesuaikan path `node`/`9router`/`User` di service file kalau versi Node berbeda,
lalu ikuti ulang langkah `daemon-reload` + `enable --now` di bagian 2.3.