# Traefik + Cloudflare Tunnel (apps.burhanfs.my.id)

Dokumentasi singkat untuk setup Traefik + Cloudflare Tunnel di server ini.

## Ringkas konfigurasi

- Traefik listen di port 80 dan 443 di dalam container.
- Public access via Cloudflare Tunnel ke service:
  - `http://traefik:80` (tanpa TLS) **atau**
  - `https://traefik:443` (dengan TLS; butuh SNI/Origin config yang benar)
- Router aplikasi menggunakan `PathPrefix` (contoh `/app`).

## File penting

- `docker-compose.yml` (Traefik)
- `config/traefik.yml` (entrypoints, redirect, provider docker)
- `letsencrypt/acme.json` (sertifikat ACME)

## Requirement

- Docker Engine + Docker Compose v2
- Network Docker external bernama `web`
  - Contoh: `docker network create web`
- Port 443 terbuka di host (jika akses langsung HTTPS)
- Cloudflare Tunnel (container `cloudflared`) untuk akses via tunnel
- Kredensial Cloudflare (DNS API token) jika memakai ACME DNS-01

## Catatan penting (hasil troubleshooting)

1) **Port 80 di Traefik**
   - Wajib aktif untuk Cloudflare Tunnel jika service diarahkan ke `http://traefik:80`.
   - Pastikan ada:
     - `--entrypoints.web.address=:80`

2) **Redirect loop (ERR_TOO_MANY_REDIRECTS)**
   - Terjadi jika Cloudflare/Tunnel pakai HTTP, sementara Traefik redirect ke HTTPS.
   - Solusi:
     - Ubah SSL/TLS mode Cloudflare ke **Full**/**Full (strict)**, atau
     - Arahkan Tunnel ke `https://traefik:443`, atau
     - Matikan redirect HTTP→HTTPS di Traefik untuk host tertentu.

3) **502 Bad Gateway dari Cloudflare**
   - Jika Tunnel ke `https://traefik:443`, biasanya karena TLS/SNI mismatch.
   - Solusi:
     - Set **Origin Server Name (SNI)** ke `apps.burhanfs.my.id`, atau
     - Aktifkan **No TLS Verify** (opsi cepat).

4) **403 challenge dari Cloudflare**
   - Berasal dari WAF/Managed Challenge Cloudflare.
   - Buat rule **Skip** untuk `apps.burhanfs.my.id` (Managed Challenge/Bot Fight Mode).

## Verifikasi cepat

Tes akses origin langsung (bypass Cloudflare):

```bash
curl -I -k --resolve apps.burhanfs.my.id:443:127.0.0.1 https://apps.burhanfs.my.id/chat
```

Jika balas `302` ke `/chat/login`, berarti Traefik + app OK.

## Troubleshooting cepat

- **502**: cek origin service di tunnel + TLS/SNI.
- **Loop redirect**: cek SSL/TLS mode Cloudflare + redirect Traefik.
- **404**: pastikan router Traefik match `Host` + `PathPrefix`.
