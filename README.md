# Mailpit Email Testing

Setup [Mailpit](https://mailpit.axllent.org/) berbasis Docker untuk menangkap email dari aplikasi saat development dan testing di staging, tanpa mengirim email ke inbox asli.

Mailpit adalah SMTP server untuk testing. Semua email yang diterimanya disimpan dan ditampilkan di Web UI: isi HTML/teks, header, lampiran, dan source email. Mailpit juga punya API untuk automated test.

## Isi repo

| Path | Fungsi |
|---|---|
| `docker-compose.yml` | Mailpit untuk **development lokal**. Aplikasi lokal mengirim email ke `localhost:1025`. |
| `vm/docker-compose.yml` | Mailpit di **VM publik** sebagai mail server penerima (MX) untuk domain dummy staging, dengan Web UI HTTPS lewat Caddy. |
| `vm/Caddyfile` | Reverse proxy HTTPS (Let's Encrypt otomatis) untuk Web UI. |
| `vm/.env.example` | Template konfigurasi VM: domain, login UI, dan domain dummy. |

---

## 1. Development lokal

```bash
docker compose up -d
```

- Web UI: http://localhost:8025
- SMTP: `localhost:1025`, tanpa TLS, username/password bebas
- Dari container lain: host `host.docker.internal`, port `1025`

Contoh config Laravel `.env`:

```
MAIL_MAILER=smtp
MAIL_HOST=127.0.0.1
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
```

---

## 2. Staging (VM): menangkap email tanpa mengubah backend

Mailpit dipasang di VM sebagai **mail server penerima** untuk sebuah subdomain dummy. Backend staging tetap mengirim email lewat provider biasa (SendGrid, SES, dan lain-lain). Email yang ditujukan ke alamat di subdomain dummy otomatis diarahkan ke Mailpit lewat MX record.

```
Daftar akun di staging pakai user1@mail.domainkamu.com
  -> backend kirim email via provider biasa
  -> provider cek MX record mail.domainkamu.com -> IP VM
  -> email masuk ke Mailpit (port 25)
```

Email ke alamat asli tetap terkirim normal. Hanya email ke subdomain dummy yang masuk ke Mailpit. Mailpit tidak butuh akun, jadi alamat apa pun di subdomain dummy bisa langsung dipakai.

### Kebutuhan

- VM dengan IP publik dan Docker terpasang
- Port **25, 80, 443** inbound terbuka di firewall
- Akses DNS ke sebuah domain

### Langkah

1. Atur DNS:

   ```
   mailpit.domainkamu.com.   A    <IP_VM>
   mail.domainkamu.com.      MX   10 mailpit.domainkamu.com.
   ```

2. Di VM:

   ```bash
   git clone <url-repo-ini> mailpit
   cd mailpit/vm
   cp .env.example .env
   nano .env            # isi domain, password UI, regex domain dummy
   docker compose up -d
   ```

3. Tes: kirim email dari Gmail ke `tes@mail.domainkamu.com`, lalu buka `https://mailpit.domainkamu.com`.

4. Daftar akun dummy di staging pakai `user1@mail.domainkamu.com`, `user2@mail.domainkamu.com`, dan seterusnya.

Untuk update, jalankan `git pull && docker compose up -d` di folder `vm/`.

### Catatan keamanan

- **Jangan commit `.env`.** File ini berisi password Web UI dan sudah di-ignore oleh `.gitignore`.
- Web UI berisi OTP dan link reset password dari staging. Pakai password yang kuat. Sebaiknya akses UI juga dibatasi ke IP tertentu lewat firewall.
- `MP_SMTP_ALLOWED_RECIPIENTS` membatasi Mailpit supaya hanya menerima email untuk subdomain dummy, sehingga tidak dibanjiri spam.
- Kalau provider email staging masih dalam **sandbox mode** (misalnya AWS SES sandbox), email ke alamat yang belum diverifikasi tidak akan dikirim.

---

## API

```
GET    /api/v1/messages        # daftar email
GET    /api/v1/message/{ID}    # detail email
DELETE /api/v1/messages        # hapus semua
```

Dokumentasi lengkap tersedia di `/api/v1/` pada instance Mailpit, atau di https://mailpit.axllent.org/docs/api-v1/.
