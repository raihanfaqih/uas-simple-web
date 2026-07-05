# uas-simple-web

Aplikasi web sederhana yang di-deploy ke production VPS dengan HTTPS, CI/CD otomatis, dan monitoring — dibuat untuk UAS Sistem Operasi + Jaringan Komputer, STMIK Tazkia.

- **Domain utama:** https://mainn.web.id

---

## Architecture Diagram

```
                         ┌─────────────────┐
                         │  User Browser    │
                         └────────┬─────────┘
                                  │ HTTPS request
                                  ▼
                         ┌─────────────────┐
                         │   DNS (A record) │
                         │  mainn.web.id     │
                         │  monitor.mainn... │──────► IP VPS
                         └────────┬─────────┘
                                  │
                                  ▼
                    ┌───────────────────────────┐
                    │        VPS Server          │
                    │                             │
                    │  ┌───────────────────────┐  │
                    │  │  nginx-proxy (:80,:443)│  │
                    │  │  - reverse proxy        │  │
                    │  │  - SSL via certbot       │  │
                    │  └──────────┬─────────────┘  │
                    │             │                 │
                    │   ┌─────────┴──────────┐      │
                    │   ▼                    ▼      │
                    │ ┌────────────┐  ┌─────────────┐│
                    │ │ web-app     │  │ uptime-kuma  ││
                    │ │ (:3000)     │  │ (:3001)      ││
                    │ └────────────┘  └─────────────┘│
                    └───────────────────────────┘
                                  ▲
                                  │ SSH deploy (private key)
                                  │
                    ┌───────────────────────────┐
                    │      GitHub Actions        │
                    │  build → test → deploy     │
                    └─────────────▲──────────────┘
                                  │ push ke branch main
                    ┌───────────────────────────┐
                    │      Developer Laptop      │
                    └───────────────────────────┘
```

### Alur singkat
1. Developer `git push` ke branch `main` di GitHub
2. GitHub Actions otomatis: checkout code → SSH ke VPS (pakai private key) → `git pull` → `docker compose up -d --build`
3. User browser resolve domain lewat DNS → HTTPS request ke `nginx-proxy` → nginx terminate SSL (sertifikat dari certbot) → reverse proxy ke container `web-app` (port 3000)
4. Subdomain `monitor.mainn.web.id` diarahkan nginx ke container `uptime-kuma` (port 3001)
5. `uptime-kuma` melakukan health check ke `https://mainn.web.id` setiap 60 detik
6. Cron job harian menjalankan backup folder project + data Uptime Kuma ke `/var/backups/uas-web`

### Tech stack
| Komponen | Teknologi |
|---|---|
| Aplikasi web | Node.js (`http-server`) |
| Reverse proxy | nginx (Alpine, dalam Docker) |
| SSL/HTTPS | certbot (Let's Encrypt), mode standalone |
| Monitoring | Uptime Kuma |
| CI/CD | GitHub Actions + `appleboy/ssh-action` (autentikasi SSH key) |
| Orchestration | Docker Compose |
| Backup | cron + tar, disimpan lokal di partition terpisah VPS |

---

## Runbook

### 1. Restart aplikasi (kalau container down/error)
```bash
cd /app/uas-web
docker compose ps                    # cek status semua container
docker compose restart web-app       # restart 1 service spesifik
# atau restart semuanya:
docker compose down
docker compose up -d
```

### 2. Rollback (kalau ada deploy yang bikin rusak)
```bash
cd /app/uas-web
git log --oneline -5                 # lihat commit terakhir
git revert <commit_hash>             # buat commit baru yang membatalkan perubahan
git push origin main                 # otomatis trigger GitHub Actions untuk re-deploy
```
Alternatif cepat (langsung di VPS tanpa lewat CI/CD, untuk keadaan darurat):
```bash
git reset --hard <commit_hash_sebelumnya>
docker compose up -d --build
```

### 3. Restore dari backup
```bash
# Lihat daftar backup yang tersedia
ls -la /var/backups/uas-web/

# Extract backup project ke lokasi sementara untuk dicek dulu
mkdir -p /tmp/restore-check
tar -xzf /var/backups/uas-web/project-<TANGGAL>.tar.gz -C /tmp/restore-check

# Kalau sudah yakin, replace folder project (HATI-HATI, ini akan menimpa)
docker compose down
tar -xzf /var/backups/uas-web/project-<TANGGAL>.tar.gz -C /
docker compose up -d

# Restore data Uptime Kuma ke docker volume
docker run --rm -v uas-web_kuma-data:/data -v /var/backups/uas-web:/backup alpine \
  sh -c "rm -rf /data/* && tar -xzf /backup/kuma-data-<TANGGAL>.tar.gz -C /"
docker compose restart uptime-kuma
```

### 4. Perpanjang / re-issue sertifikat SSL (kalau certbot bermasalah)
```bash
docker compose stop nginx-proxy
sudo certbot renew --force-renewal
# atau kalau perlu re-issue dari nol:
sudo certbot certonly --standalone -d mainn.web.id -d www.mainn.web.id -d monitor.mainn.web.id
docker compose up -d
```
Certbot sudah auto-renew via scheduled task bawaan, biasanya tidak perlu manual kecuali ada error.

---

## Catatan Operasional

### Di mana log aplikasi?
```bash
docker logs uas_web_app --tail 50      # log aplikasi web
docker logs nginx_proxy --tail 50      # log akses/error nginx
docker logs uptime_kuma --tail 50      # log monitoring
```
Semua log tercetak ke stdout container, tidak ada file log terpisah yang perlu dicari manual.

### Cara cek monitoring
Buka dashboard: `https://monitor.mainn.web.id`. Monitor `https://mainn.web.id` sudah terdaftar dengan interval cek 60 detik. Status hijau = aktif/up, merah = down.

### Struktur folder di VPS
```
/app/uas-web/
├── docker-compose.yml    # definisi 3 service: web-app, uptime-kuma, nginx-proxy
├── nginx.conf            # konfigurasi reverse proxy + SSL untuk 2 domain
└── html/                 # (jika ada) source aplikasi

/etc/letsencrypt/live/mainn.web.id/   # sertifikat SSL (dikelola certbot)
/var/backups/uas-web/                 # hasil backup harian
/root/scripts/backup.sh               # script backup
```

### Variabel/konfigurasi sensitif
- **GitHub Secrets:** `SSH_HOST`, `SSH_USERNAME`, `SSH_PRIVATE_KEY` — dipakai workflow `.github/workflows/deploy.yml` untuk autentikasi ke VPS, tidak pernah muncul di kode.

### Kalau VPS hilang/rusak total
1. Provision VPS baru, install Docker + Docker Compose
2. Arahkan ulang DNS (`@`, `www`, `monitor`) ke IP VPS baru
3. Clone ulang repo: `git clone <url_repo> /app/uas-web`
4. Setup ulang SSH key untuk GitHub Actions (generate baru, update secret `SSH_PRIVATE_KEY`)
5. Generate ulang sertifikat SSL: `sudo certbot certonly --standalone -d mainn.web.id -d www.mainn.web.id -d monitor.mainn.web.id`
6. Restore data dari backup terakhir (lihat bagian Restore di atas, kalau backup sempat diamankan di luar VPS lama)
7. `docker compose up -d`

### Kenapa backup disimpan lokal, bukan cloud storage?
Sempat dicoba Cloudflare R2, tapi terkendala verifikasi kartu kredit internasional. Dokumen UAS mengizinkan backup ke "cloud storage **atau** partition terpisah" sebagai opsi yang setara, sehingga dipilih backup ke folder terpisah (`/var/backups/uas-web`) di VPS yang sama, dipisahkan dari folder aplikasi produksi.

---

## Lessons Learned (ringkasan troubleshooting selama development)

| Masalah | Solusi |
|---|---|
| SSH password auth ditolak GitHub Actions | Pindah ke autentikasi SSH key |
| `certbot --nginx` bentrok port dengan nginx di Docker | Pakai `certbot certonly --standalone`, matikan container dulu |
| Volume sertifikat pakai named volume kosong | Ganti ke bind mount `/etc/letsencrypt:/etc/letsencrypt:ro` |
| `nginx.conf` yang jalan ternyata masih versi lama | Overwrite langsung via `cat > file << EOF`, verifikasi isi dengan `docker exec` |