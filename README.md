# +62 STORE Frontend

Frontend e-commerce +62 STORE berbasis **Jekyll**, Ionic Web Components, Tailwind CSS, dan vanilla JavaScript. Aplikasi menyediakan katalog streetwear, AI assistant, pilihan variasi produk, cart, serta checkout melalui WhatsApp.
![frontstore][frontstoreImage]

## Fitur

- Katalog produk dari API Cloudflare Worker yang bersumber dari D1.
- Pencarian produk dan filter berdasarkan style atau warna.
- Detail produk dengan preview gambar eksternal.
- Validasi stok, ukuran, dan variasi sebelum masuk cart.
- Cart dengan update quantity dan checkout WhatsApp.
- AI assistant berbasis Llama 3.3 70B.
- Voice input melalui Web Speech API.
- Text-to-speech melalui ElevenLabs dengan fallback browser speech synthesis.
- Dashboard admin untuk produk D1, context toko AI, dan campaign ads.
- Stok produk per kombinasi varian warna dan ukuran.
- Product detail dinamis: pilihan warna memakai swatch hex, gambar mengikuti warna aktif, dan ukuran mengikuti varian warna.
- Editor inventory grouped di admin: setiap warna memiliki daftar ukuran dan stoknya sendiri.
- Custom confirm dan alert pada dashboard admin.
- Struktur UI modular menggunakan Jekyll `_includes`.

## Arsitektur

```text
GitHub Pages / Jekyll
  |
  +-- index.html       storefront
  +-- admin.html       dashboard admin
  +-- _includes/       komponen markup Jekyll
  |
  +--> plus62ai.warpzone.workers.dev
  |      +-- Workers AI / Llama
  |      +-- D1: products dan store_settings
  |      +-- ElevenLabs TTS
  |      +-- Admin API
  |
  +--> app-core.warpzone.workers.dev
         +-- license validation
         +-- protected ads script
         +-- KV campaign ads
```

### Service utama

| Service | Fungsi |
| --- | --- |
| Jekyll + GitHub Pages | Build dan hosting frontend statis |
| `plus62ai` Worker | API katalog, AI chat, TTS, admin product, dan store settings |
| D1 `plus62-products` | Penyimpanan produk dan context toko |
| `app-core` Worker | License check, protected script, dan campaign ads |
| KV `ADS_KV` | Penyimpanan campaign iklan |
| URL gambar eksternal | Media produk dan banner tanpa upload R2 |

## Struktur Direktori

```text
frontend/
├── _config.yml
├── Gemfile
├── Gemfile.lock
├── index.html
├── admin.html
├── tailwind.css
├── _includes/
│   ├── store-header.html
│   ├── store-catalog.html
│   ├── admin-header.html
│   ├── admin-login.html
│   ├── admin-settings.html
│   ├── admin-products.html
│   ├── admin-campaign.html
│   ├── admin-system.html
│   └── admin-dialog.html
└── assets/icon/
```

`index.html` dan `admin.html` tetap menjadi entry point, sedangkan markup berulang dipisahkan ke `_includes` tanpa mengubah ID, class, atau event JavaScript yang sudah dipakai.

![dashboard][dashboardImage]

## Menjalankan Lokal

Prasyarat:

- Ruby 3.x
- Bundler
- Node.js hanya diperlukan untuk validasi JavaScript opsional

Dari direktori `frontend`:

```bash
bundle install
bundle exec jekyll serve --baseurl /store
```

Buka:

```text
http://127.0.0.1:4000/store/
http://127.0.0.1:4000/store/admin/
```

Build produksi:

```bash
bundle exec jekyll build
```

Output build berada di `_site/`.

## Konfigurasi Jekyll

Konfigurasi berada di [_config.yml](_config.yml):

```yaml
url: "https://daffadevhosting.github.io"
baseurl: "/store"
permalink: pretty
```

Karena `baseurl` adalah `/store`, link internal sebaiknya memakai filter Jekyll:

```liquid
{{ '/' | relative_url }}
```

## API Frontend

Frontend memakai endpoint berikut:

| Method | Endpoint | Keterangan |
| --- | --- | --- |
| `GET` | `/api/products` | Katalog publik dari D1 |
| `GET` | `/api/health` | Health check backend |
| `POST` | `/api/chat` | Chat AI dan action produk |
| `POST` | `/api/elevenlabs/agent-webhook` | Webhook ElevenLabs |

Base URL saat ini:

```text
https://plus62ai.warpzone.workers.dev
```

Gambar produk tetap berupa URL eksternal. Field gambar dapat berupa URL absolut atau path yang diselesaikan ke sumber katalog lama.

## Dashboard Admin

Dashboard tersedia di:

```text
/store/admin/
```

Login menggunakan `ADMIN_TOKEN`. Token disimpan di `localStorage` browser dan dikirim sebagai:

```http
Authorization: Bearer <ADMIN_TOKEN>
```

### Product management

Endpoint admin produk:

| Method | Endpoint | Keterangan |
| --- | --- | --- |
| `GET` | `/api/admin/products` | Daftar semua produk |
| `POST` | `/api/admin/products` | Tambah produk |
| `PUT` | `/api/admin/products/:slug` | Perbarui produk |
| `DELETE` | `/api/admin/products/:slug` | Nonaktifkan produk |
| `POST` | `/api/admin/products/import` | Import atau update dari katalog lama |

Field utama produk:

- `slug`
- `sku`
- `title`
- `price`
- `stock`
- `description`
- `image`
- `styles`
- `sizes`
- `variants` dengan `style`, `size`, `stock`, dan `image`
- `discount`

Stok total produk dihitung dari jumlah stok seluruh varian aktif. Status tersedia atau habis tidak disimpan sebagai toggle manual:

```text
style + size dengan stock > 0 = tersedia
style + size dengan stock = 0 = habis
```

Contoh data varian:

```json
{
  "variants": [
    { "style": "Biru", "size": "S", "stock": 1, "image": "https://example.com/blue.jpg" },
    { "style": "Merah", "size": "M", "stock": 2, "image": "https://example.com/red.jpg" }
  ]
}
```

Dashboard mengelompokkan data varian menjadi:

```text
Produk
├── Kuning
│   ├── S — stok 2
│   └── M — stok 3
└── Biru
  └── M — stok 1
```

Tombol `+ Warna` menambah grup warna baru, sedangkan tombol `+ Ukuran` menambah ukuran di dalam warna yang dipilih. Backend tetap menyimpan setiap kombinasi sebagai satu row `product_variants` agar validasi stok dan checkout tetap presisi.

### AI store context

Context yang dapat diubah admin:

- Nama toko
- Alamat toko
- Link Google Maps
- Nomor WhatsApp admin
- E-mail
- Jam operasional

Endpoint:

| Method | Endpoint |
| --- | --- |
| `GET` | `/api/admin/settings` |
| `PUT` | `/api/admin/settings` |

Nilai ini dibaca backend pada setiap request `/api/chat` dan dimasukkan ke system prompt AI.

### Campaign ads

Campaign ads dikelola melalui `app-core` Worker:

| Method | Endpoint | Keterangan |
| --- | --- | --- |
| `GET` | `/admin/ads` | Membaca campaign |
| `PUT` | `/admin/ads` | Menyimpan campaign ke KV |

Dashboard mendukung title, image URL, link URL, dan interval rotasi banner.

## Backend dan D1

Schema katalog berada di:

```text
../backend/migrations/0001_products.sql
../backend/migrations/0002_store_settings.sql
../backend/migrations/0003_product_variants.sql
../backend/migrations/0004_seed_product_variants.sql
```

D1 binding dikonfigurasi di `backend/wrangler.toml` sebagai `DB`. Untuk membuat database baru:

```bash
cd ../backend
npx wrangler d1 create plus62-products --config wrangler.toml
```

Terapkan migration remote:

```bash
npx wrangler d1 migrations apply plus62-products --remote --config wrangler.toml
```

Deploy backend:

```bash
npx wrangler deploy --config wrangler.toml
```

## Secret dan Binding

Jangan menaruh secret di source code atau `[vars]`.

Backend:

```bash
cd ../backend
npx wrangler secret put ADMIN_TOKEN
npx wrangler secret put ELEVENLABS_API_KEY
```

Binding yang digunakan:

- `AI`: Workers AI
- `DB`: D1 `plus62-products`
- `ADS_KV`: KV campaign ads
- `ELEVENLABS_VOICE_ID`: variable non-secret
- `ADMIN_TOKEN`: secret
- `ELEVENLABS_API_KEY`: secret

Ads worker memakai binding `ADS_KV` dan `ADMIN_TOKEN`. Deploy dari root workspace dengan konfigurasi eksplisit agar tidak salah memilih project:

```bash
npx wrangler deploy --config ads-engine/wrangler.toml
```

## Deployment Frontend

Frontend dapat dipublish sebagai GitHub Pages dari hasil Jekyll build. Pastikan workflow atau Pages build memakai direktori `frontend` dan menjalankan:

```bash
bundle install
bundle exec jekyll build
```

Untuk local preview, gunakan `bundle exec jekyll serve --baseurl /store`.

## Validasi

Validasi dasar yang digunakan project:

```bash
bundle exec jekyll build --trace
node --check ../backend/src/worker.js
```

Smoke test API admin perlu menyertakan token valid. Preflight CORS untuk CRUD harus mengizinkan:

```http
GET, POST, PUT, DELETE, OPTIONS
```

## TODO Next Version

### Notif BOT TELEGRAM

- Kirim Notifikasi order + success payment ke BOT TELEGRAM admin / user toko.

### Stock dinamis saat checkout WhatsApp

Saat ini checkout WhatsApp membuat pesan pesanan dan membuka WhatsApp, tetapi belum mengurangi stok D1 secara otomatis. Implementasi berikutnya perlu membuat alur order terkontrol:

- Buat endpoint order, misalnya `POST /api/orders`.
- Kirim snapshot cart berisi `productSlug`, `style`, `size`, dan `quantity`.
- Validasi ulang stok setiap varian di server, bukan hanya di browser.
- Kurangi `product_variants.stock` menggunakan transaksi atau conditional update:

```sql
UPDATE product_variants
SET stock = stock - ?
WHERE id = ? AND stock >= ?;
```

- Tolak order jika salah satu varian tidak cukup stok.
- Simpan order dan item order di D1 sebelum membuka WhatsApp.
- Buat nomor order unik untuk dimasukkan ke pesan WhatsApp.
- Kurangi stok hanya sekali dengan idempotency key agar refresh atau retry tidak mengurangi stok dua kali.
- Sediakan status order seperti `pending`, `confirmed`, `cancelled`, dan `fulfilled`.
- Tambahkan mekanisme release atau restore stok jika order dibatalkan atau tidak dikonfirmasi.
- Setelah order berhasil, frontend melakukan refresh katalog agar stok terbaru terlihat user lain.
- Integrasikan payment otomatis Midtrans dengan Snap atau Core API.
- Simpan `order_id`, `transaction_id`, `payment_type`, `gross_amount`, dan `transaction_status` di D1.
- Buat webhook Midtrans yang memverifikasi signature dan memperbarui status pembayaran secara idempotent.
- Kurangi atau reservasi stok berdasarkan status pembayaran yang dikonfirmasi server, bukan callback dari browser.
- Sediakan halaman atau panel status pembayaran untuk user dan admin.
- Tangani status `pending`, `settlement`, `capture`, `expire`, `cancel`, dan `deny` sesuai tipe pembayaran.
- Simpan credential Midtrans sebagai Worker secret, bukan di frontend atau repository.

Pengurangan stok tidak sebaiknya dilakukan langsung dari frontend atau berdasarkan keberhasilan `window.open()` WhatsApp, karena browser tidak dapat menjamin pesan benar-benar terkirim atau order benar-benar dikonfirmasi admin.

## Catatan Keamanan

- `ADMIN_TOKEN` hanya digunakan untuk endpoint admin.
- API key ElevenLabs hanya berada di secret Worker.
- Input markdown chat disanitasi dengan DOMPurify.
- Endpoint admin tidak dapat dipakai tanpa header `Authorization`.
- URL gambar eksternal harus berasal dari sumber HTTPS yang dipercaya.
- CORS backend mengizinkan request frontend dan method CRUD yang diperlukan.

## Lisensi

Project ini menggunakan lisensi MIT. Lihat [LICENSE](LICENSE).


[frontstoreImage]: ./frontstore.png
[dashboardImage]: ./dashboard.png