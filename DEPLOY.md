# Memasang Sapecc di domain sendiri

Panduan ini memasang Sapecc di VPS Ubuntu dengan domain dan HTTPS.

Polanya sederhana: **Node.js jalan di port 3000 (hanya lokal), Nginx yang memegang domain dan HTTPS.** Aplikasi ini tidak butuh `npm install` dan tidak butuh server database terpisah.

Yang perlu disiapkan:

- VPS Ubuntu (22.04 atau 24.04), akses root atau sudo
- Domain yang panel DNS-nya bisa kamu atur
- Akun Midtrans (sandbox dulu untuk uji coba)

---

## 1. Arahkan domain ke server

Di panel DNS domain, buat dua record:

| Type | Name | Value |
|---|---|---|
| A | `@` | IP VPS |
| A | `www` | IP VPS |

Cek dari komputermu sampai hasilnya menunjuk ke IP VPS:

```bash
ping domain-kamu.com
```

Jangan lanjut ke langkah HTTPS sebelum ini benar, karena Let's Encrypt memverifikasi kepemilikan lewat domain tersebut.

## 2. Siapkan server

Node bawaan Ubuntu terlalu tua. Sapecc butuh **Node 22.13 atau lebih baru** karena memakai database bawaan `node:sqlite`.

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs nginx
node -v          # pastikan hasilnya >= v22.13
```

## 3. Unggah proyek

Taruh folder proyek di `/var/www/sapecc`, lalu buat pengguna khusus supaya aplikasi tidak jalan sebagai root:

```bash
sudo adduser --system --group --home /var/www/sapecc sapecc
sudo chown -R sapecc:sapecc /var/www/sapecc
```

Cara mengunggah, pilih salah satu dari komputermu:

```bash
scp -r sapecc root@IP-VPS:/var/www/
# atau, kalau sudah ada di Git:
sudo -u sapecc git clone URL-REPO /var/www/sapecc
```

## 4. Isi `.env` untuk production

```bash
cd /var/www/sapecc
sudo -u sapecc cp .env.example .env
sudo nano .env
```

Yang wajib diubah:

```
NODE_ENV=production
BASE_URL=https://domain-kamu.com
COOKIE_SECURE=1
SEED_DEMO=0
TRUST_PROXY=1
ADMIN_EMAIL=email-kamu@domain-kamu.com
ADMIN_PASSWORD=sandi-acak-minimal-12-karakter
PAYMENT_PROVIDER=midtrans
MIDTRANS_SERVER_KEY=SB-Mid-server-xxxxxxxx
MIDTRANS_PRODUCTION=0
```

Catatan tiap baris:

- `BASE_URL` ditulis tanpa garis miring di akhir. Alamat ini dipakai untuk mengembalikan pembeli setelah membayar.
- `COOKIE_SECURE=1` membuat cookie login hanya dikirim lewat HTTPS. Karena itu langkah 7 wajib.
- `SEED_DEMO=0` mematikan toko dan produk contoh.
- `TRUST_PROXY=1` supaya IP asli pengunjung terbaca dari Nginx, bukan `127.0.0.1`. Tanpa ini, pembatasan percobaan login akan menghitung semua orang sebagai satu IP.
- Akun admin dibuat otomatis saat server pertama kali jalan, dari `ADMIN_EMAIL` dan `ADMIN_PASSWORD`.

File ini berisi kunci pembayaran dan sandi admin, jadi kunci izin aksesnya:

```bash
sudo chown sapecc:sapecc .env && sudo chmod 600 .env
```

## 5. Jalankan sebagai service

Supaya aplikasi hidup lagi sendiri setelah crash atau VPS restart, buat `/etc/systemd/system/sapecc.service`:

```ini
[Unit]
Description=Sapecc
After=network.target

[Service]
User=sapecc
Group=sapecc
WorkingDirectory=/var/www/sapecc
ExecStart=/usr/bin/node --disable-warning=ExperimentalWarning server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Nyalakan dan cek:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sapecc
sudo systemctl status sapecc
curl -I http://127.0.0.1:3000      # harusnya menjawab 200
```

Melihat log kalau ada masalah:

```bash
sudo journalctl -u sapecc -f
```

## 6. Nginx sebagai pintu domain

Buat `/etc/nginx/sites-available/sapecc`:

```nginx
server {
    listen 80;
    server_name domain-kamu.com www.domain-kamu.com;
    client_max_body_size 2M;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Aktifkan:

```bash
sudo ln -s /etc/nginx/sites-available/sapecc /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Dua baris yang sering jadi masalah kalau salah:

- **`proxy_set_header Host $host;` jangan dihapus.** Perlindungan CSRF di `server.js` membandingkan header `Origin` dengan header `Host`. Kalau Nginx mengganti Host menjadi `127.0.0.1:3000`, semua aksi simpan, posting, dan bayar akan ditolak dengan pesan "Permintaan ditolak."
- **`client_max_body_size 2M`** diperlukan karena upload logo di panel admin bisa mencapai sekitar 1,6 MB. Bawaan Nginx hanya 1 MB.

## 7. Pasang HTTPS

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d domain-kamu.com -d www.domain-kamu.com
```

Certbot menyunting sendiri file Nginx di atas, menambahkan pengalihan dari HTTP ke HTTPS, dan memasang perpanjangan otomatis. Cek perpanjangannya bisa jalan:

```bash
sudo certbot renew --dry-run
```

HTTPS di sini bukan pilihan. Dengan `COOKIE_SECURE=1`, cookie login hanya dikirim lewat HTTPS, jadi tanpa sertifikat kamu tidak akan bisa masuk sama sekali.

Kalau memakai Cloudflare, set SSL/TLS mode ke **Full (strict)**. Mode Flexible akan membuat login gagal berputar-putar.

Buka `https://domain-kamu.com` dan masuk dengan akun admin dari `.env`. Panel admin ada di `/admin`.

## 8. Sambungkan Midtrans ke domain

1. Di dashboard Midtrans, buka **Settings, Configuration**.
2. Isi **Payment Notification URL**:
   ```
   https://domain-kamu.com/api/payments/notify
   ```
3. Coba satu transaksi dengan simulator sandbox. Status pesanan berubah setelah notifikasi masuk.
4. Setelah akun production disetujui, ganti `MIDTRANS_SERVER_KEY` ke kunci production, set `MIDTRANS_PRODUCTION=1`, lalu `sudo systemctl restart sapecc`.

Server memverifikasi tiap notifikasi dua kali: mencocokkan `signature_key`, lalu menanyakan ulang status ke API Midtrans. Jadi notifikasi palsu tidak bisa menandai pesanan lunas.

Sesuaikan juga `QRIS_FEE_PERCENT` dan `VA_FEE` di `.env` dengan tarif MDR di kontrak Midtrans kamu, karena itu biaya yang ditagihkan ke pembeli.

## 9. Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

Port 3000 tidak perlu dibuka ke publik. Nginx mengaksesnya lewat `127.0.0.1`.

## 10. Backup

Semua data ada di dua tempat, keduanya di dalam folder `data/`:

| Isi | Lokasi |
|---|---|
| Akun, saldo, pesanan, chat | `data/sapecc.db` |
| Logo yang diunggah admin | `data/uploads/` |

Backup harian dengan cron. Jalankan `sudo crontab -e`, lalu tambahkan:

```
0 2 * * * tar -czf /var/backups/sapecc-$(date +\%F).tar.gz -C /var/www/sapecc data
```

Buat dulu foldernya: `sudo mkdir -p /var/backups`.

Dua hal yang sering terlewat:

- Salinan `sapecc.db` saat ada transaksi berjalan bisa tidak konsisten. Kalau `sqlite3` terpasang, lebih aman memakai `sqlite3 data/sapecc.db ".backup /tmp/sapecc.db"` lalu arsipkan hasilnya.
- Backup yang hanya ada di VPS yang sama tidak menolong kalau VPS-nya hilang. Salin ke tempat lain, misalnya object storage atau komputer sendiri.

---

## Memperbarui aplikasi

```bash
sudo tar -czf /var/backups/sapecc-sebelum-update.tar.gz -C /var/www/sapecc data
sudo systemctl stop sapecc
# salin file baru, jangan menimpa .env dan folder data/
sudo chown -R sapecc:sapecc /var/www/sapecc
sudo systemctl start sapecc
sudo journalctl -u sapecc -n 50
```

## Alternatif: Docker

Proyek ini sudah punya `Dockerfile`. Yang penting, pasang volume ke `/app/data` supaya database tidak hilang saat container dibuat ulang:

```bash
docker build -t sapecc .
docker run -d --name sapecc \
  -p 127.0.0.1:3000:3000 \
  -v /var/lib/sapecc:/app/data \
  --env-file .env \
  --restart unless-stopped \
  sapecc
```

Nginx dan HTTPS tetap seperti langkah 6 dan 7.

## Kalau ada masalah

| Gejala | Kemungkinan penyebab |
|---|---|
| "Permintaan ditolak." saat simpan atau bayar | `proxy_set_header Host $host;` hilang di Nginx |
| Login selalu kembali ke halaman masuk | Situs diakses lewat HTTP padahal `COOKIE_SECURE=1`, atau Cloudflare masih mode Flexible |
| 502 Bad Gateway | Service mati. Cek `sudo journalctl -u sapecc -n 50` |
| 413 saat unggah logo | `client_max_body_size` di Nginx kurang besar |
| Status pesanan tidak berubah setelah bayar | Notification URL di Midtrans salah, atau `BASE_URL` masih `localhost` |
| Error soal `node:sqlite` saat start | Versi Node di bawah 22.13. Cek `node -v` |
| Semua pengunjung kena batas login | `TRUST_PROXY=1` belum diisi di `.env` |

## Daftar periksa sebelum dibuka untuk umum

- [ ] `NODE_ENV=production` dan `SEED_DEMO=0`, data contoh sudah hilang dari situs
- [ ] `ADMIN_PASSWORD` sudah diganti dengan sandi acak, dan `.env` sudah `chmod 600`
- [ ] HTTPS jalan, `certbot renew --dry-run` sukses
- [ ] Satu transaksi uji coba berhasil sampai dana cair ke penjual
- [ ] Cron backup sudah jalan, dan hasilnya sudah pernah dicoba dipulihkan
- [ ] Halaman Ketentuan Layanan dan Kebijakan Privasi sudah ada

## Catatan hukum

Menahan dana pengguna (rekber dan saldo) di Indonesia bisa termasuk kegiatan yang diatur Bank Indonesia dan OJK. Sebelum menerima transaksi sungguhan, konsultasikan dengan ahli hukum, dan pertimbangkan memakai layanan escrow atau e-money dari penyedia pembayaran berlisensi. Kebijakan Privasi juga perlu menyesuaikan UU Pelindungan Data Pribadi.
