# Analisis Promo JSM pada Retail Minimarket Alfamart

Analisis data transaksi POS Alfamart untuk mengidentifikasi pola pembelian produk Promo JSM (Jumat, Sabtu, Minggu), mendeteksi transaksi dengan kuantitas tidak wajar, mengestimasi dampak finansialnya, dan merumuskan rekomendasi kebijakan pembatasan kuantitas pembelian per SKU.

## Latar Belakang

Promo JSM adalah program potongan harga akhir pekan dari Alfamart untuk produk tertentu. Harga promo berpotensi mendorong pembelian dalam jumlah besar oleh sebagian kecil pembeli (diduga untuk dijual kembali), sehingga mengurangi ketersediaan produk bagi konsumen lain dan memengaruhi profitabilitas promosi.

Kasus serupa pernah terjadi pada Januari 2022, ketika minyak goreng seharga Rp14.000/liter cepat habis di banyak gerai dan Alfamart menerapkan batas pembelian per orang. Preseden ini menjadi acuan dalam proyek ini.

## Rumusan Masalah

1. **Dampak finansial** — Berapa besar potensi nilai penjualan dari pembelian berlebih pada produk Promo JSM?
2. **Kebijakan pembatasan** — Berapa batas kuantitas pembelian per transaksi yang tepat untuk setiap SKU promo JSM?

## Stakeholder

- **Manajemen Alfamart** — informasi dampak finansial dan rekomendasi kebijakan.
- **Tim Data Analyst** — pengolahan data dan penyusunan insight.
- **Tim Operasional Gerai** — penerapan pengawasan transaksi dan pengelolaan stok promo.
- **Pelanggan** — pihak yang terdampak oleh ketersediaan produk promo.

## Dataset

Data diambil dari BigQuery dan juga tersedia dalam bentuk CSV pada folder `data/`.

| Dataset | Baris | Kolom | Deskripsi |
|---|---:|---:|---|
| `alfa_pos_transactions` | 455.148 | 12 | Transaksi POS: struk, gerai, SKU, member, waktu, qty, harga, diskon, metode pembayaran |
| `alfa_products` | 507 | 11 | Master produk: kategori, merek, harga, masa simpan, status dan persentase diskon JSM |
| `alfa_stores` | 1.003 | 10 | Master gerai: lokasi, tipe, format, luas, fasilitas self-checkout |
| `alfa_members` | 48.214 | 10 | Data member: registrasi, kota, tier, kanal akuisisi, saldo poin |
| `alfa_complaint_tickets` | 10.759 | 9 | Tiket komplain: kategori, kanal, status penyelesaian, CSAT, nilai sengketa |

Relasi antar tabel:

- `transactions.sku_id` → `products.sku_id`
- `transactions.store_id` → `stores.store_id`
- `transactions.member_id` → `members.member_id`
- `transactions.receipt_id` → `complaint_tickets.receipt_id` (diagregasi per struk sebelum digabung)

Deskripsi lengkap setiap kolom tersedia di dalam notebook.

## Alur Analisis

1. **Data Extract** — Mengambil lima tabel dari BigQuery menggunakan `pandas_gbq`.
2. **Data Merging** — Menggabungkan transaksi dengan produk, gerai, member, dan ringkasan komplain (left join, validasi many-to-one).
3. **Data Cleaning**
   - Penyeragaman format: variasi penulisan `payment_method` (mis. `CASH`/`Cash`, `debit bca`/`BCA Debit`) dan `product_name` (mis. `Indm Grg`, `INDOMIE MIE GORENG` → `Indomie Goreng`).
   - Missing values: `member_id` kosong 33,76% (transaksi non-member), `shelf_life_days` kosong 19,73% (produk rokok).
   - Penyesuaian tipe data kolom tanggal dan `complaint_count`.
   - Pengecekan duplikat (tidak ditemukan).
   - Range constraints: 1.483 record dengan `qty` dan `total_price` negatif, serta 104 record dengan diskon negatif. Setelah divalidasi terhadap harga master produk, seluruhnya konsisten sehingga dikoreksi menjadi nilai absolut.
   - Outlier: deteksi dengan IQR global dan IQR per SKU.
4. **Exploratory Data Analysis** — Perbandingan JSM vs Non-JSM, tren bulanan, pola harian, weekday vs weekend, identifikasi transaksi bulk, simulasi batas pembelian, serta analisis per kategori, pelanggan, interval transaksi, dan gerai.

## Metodologi Penentuan Batas

Batas kuantitas kandidat dihitung per SKU menggunakan metode IQR:

```
upper_bound = Q3 + 1.5 × IQR
```

Pendekatan per SKU dipilih karena setiap produk memiliki pola pembelian yang berbeda. Sebagai pembanding, simulasi batas tetap 10, 20, dan 30 unit untuk semua SKU berdampak pada 29,77–41,63% transaksi JSM, sedangkan IQR per SKU hanya menandai 0,54% transaksi.

Cakupan analisis utama: transaksi Promo JSM pada Jumat–Minggu dengan status `COMPLETED` dan `qty` positif (54.717 transaksi).

## Temuan Utama

- Penjualan produk JSM naik tajam pada akhir pekan. Total quantity naik 160,66% dan rata-rata quantity per record naik 121,65% dibanding hari kerja, sementara produk Non-JSM relatif stabil.
- **296 transaksi (0,54%)** melebihi batas IQR per SKU, dengan **76.760 unit** quantity berlebih.
- Estimasi nilai penjualan dari quantity berlebih setelah diskon sebesar **Rp1.175.916.383**, atau **5,54%** dari total penjualan JSM weekend.
- Per kategori: Food & Staples 72,18%, Beverage 18,87%, Personal Care 8,95%.
- Terdapat 219 pelanggan teridentifikasi dan 77 transaksi tanpa member ID. Tidak ditemukan pelanggan yang melakukan transaksi berlebih secara berulang, dan 94,44% interval transaksi antar pelanggan berjarak lebih dari 30 hari.
- Sepuluh pelanggan teratas menyumbang 84,87% quantity berlebih dari kelompok pelanggan teridentifikasi, sehingga dampaknya didominasi oleh sedikit transaksi ekstrem.
- Transaksi terdampak tersebar di 96 gerai. Alfamart Pahlawan 345 (Semarang) mencatat quantity berlebih tertinggi, yaitu 8.997 unit.

Catatan: pola ini belum membuktikan adanya praktik reselling. Quantity yang sangat besar juga perlu divalidasi terhadap kemungkinan salah input POS atau perbedaan satuan. Estimasi nilai penjualan juga tidak dapat langsung dianggap sebagai kerugian margin aktual.

## Rekomendasi

1. **Prioritaskan kategori Food & Staples**, terutama tiga SKU dengan dampak tertinggi:
   - Minyak Goreng Bimoli (SKU000486): Rp169,74 juta
   - Minyak Bimoli 2L (SKU000022): Rp144,86 juta
   - Indomie Goreng (SKU000406): Rp138,67 juta
2. **Gunakan IQR per SKU sebagai kandidat batas awal**, lalu validasi dengan kebutuhan wajar konsumen dan satuan produk sebelum ditetapkan sebagai kebijakan final.
3. **Perketat pengawasan pada 96 gerai terdampak**, dimulai dari gerai dengan quantity berlebih tertinggi.
4. **Perkuat identifikasi pelanggan** (penggunaan member ID) agar pembelian berlebih yang berulang dapat dipantau.
5. **Lakukan monitoring berkala** setelah kebijakan diterapkan agar pembatasan tetap efektif tanpa menghambat pembelian wajar.

## Struktur Repository

```
.
├── Analisis_Promo_JSM_pada_Retail_Minimarket_Alfamart.ipynb
├── data/
│   ├── alfa_pos_transactions.csv
│   ├── alfa_products.csv
│   ├── alfa_stores.csv
│   ├── alfa_members.csv
│   └── alfa_complaint_tickets.csv
└── README.md
```

## Tools

- Python: pandas, numpy, matplotlib, seaborn, missingno
- Google BigQuery (`pandas-gbq`, `google-cloud-bigquery`)
- Google Colab

## Cara Menjalankan

Notebook dibuat di Google Colab dan mengambil data langsung dari BigQuery. Untuk menjalankannya secara lokal tanpa BigQuery, ganti sel ekstraksi data dengan pembacaan CSV:

```python
import pandas as pd

df_transactions = pd.read_csv('data/alfa_pos_transactions.csv')
df_products     = pd.read_csv('data/alfa_products.csv')
df_stores       = pd.read_csv('data/alfa_stores.csv')
df_members      = pd.read_csv('data/alfa_members.csv')
df_complaints   = pd.read_csv('data/alfa_complaint_tickets.csv')
```

Instalasi dependensi:

```bash
pip install pandas numpy matplotlib seaborn missingno pandas-gbq google-cloud-bigquery
```

Saat membaca dari CSV, kolom `trx_time` dan `ticket_date` perlu dikonversi dengan `pd.to_datetime`, karena BigQuery sudah mengembalikannya sebagai datetime. Format `trx_time` di CSV adalah `MM/DD/YYYY HH:MM`.
