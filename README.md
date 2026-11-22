# Tradesys Edge Dual

Web app sistem trading crypto **Trend Following + Mean Reversion** untuk **Binance Spot (long-only)**.
Satu file `index.html` tanpa build, tanpa server, tanpa aplikasi berbayar. Data diambil langsung dari API publik Binance di browser.

> Aplikasi ini hanya memberi **sinyal dan alert**. Tidak ada order yang dikirim ke exchange dan tidak butuh API key Binance.

---

## Fitur

| Tab | Isi |
|---|---|
| **Dashboard** | Regime BTC daily, jumlah sinyal trend/reversion, komposisi regime, chart setup terbaik, daftar top setup |
| **Scanner** | Scan watchlist atau semua pair USDT (top N berdasarkan volume), filter Signals / Trend / Reversion / All |
| **Chart** | Candlestick + EMA 20/50/200 + Bollinger 20,2 + panel ADX, penanda sinyal historis, level entry/stop/target |
| **Backtest** | Router (keduanya), Trend saja, atau Reversion saja, pada 1.000–3.000 bar, dengan fee dan slippage |
| **Risk** | Kalkulator ukuran posisi (risk %, batas alokasi, target 1R/2R/3R) + aturan portofolio |
| **Alerts** | Log sinyal baru, kirim ke Telegram, copy teks, auto-scan 1 jam / 4 jam |
| **Config** | Semua parameter strategi, risiko, biaya, endpoint Binance, Telegram |

Timeframe scan: **1h, 4h, 1d**.

---

## Logika strategi

### 1. Regime router (per pair, bar terakhir yang sudah close)
| Kondisi | Regime | Playbook |
|---|---|---|
| ADX ≥ 25, harga & EMA 50 di atas EMA 200 | Uptrend | Trend Following |
| ADX ≤ 20 | Range | Mean Reversion |
| ADX ≥ 25, harga & EMA 50 di bawah EMA 200 | Downtrend | Tidak ada long |
| Di antara keduanya | Transition | Hanya MR jika syarat terpenuhi |

Filter BTC: jika BTC daily di bawah struktur EMA 200, sinyal trend diberi label **BTC risk-off** (atau diblokir, bisa diatur di Config).

### 2. Trend Following
- **Breakout:** close > high tertinggi 20 bar sebelumnya (Donchian), EMA 20 > 50 > 200, ADX naik vs 3 bar lalu, volume ≥ 1,5× SMA 20 volume.
- **Pullback:** low menyentuh EMA 20 (toleransi 0,5%), close kembali di atas EMA 20, RSI 14 antara 40–60, candle hijau.
- **Stop awal:** entry − 2× ATR 14. **Target referensi:** 2R.
- **Exit:** trailing chandelier (high tertinggi sejak entry − 3× ATR) atau close di bawah EMA 50.

### 3. Mean Reversion
- Syarat: harga di atas EMA 200 (bisa dimatikan) dan ADX < 25.
- **Trigger:** Z-score(20) ≤ −2 **atau** close di bawah Bollinger bawah, **dan** RSI 2 ≤ 10.
- **Stop:** entry − 1,5× ATR. **Target:** SMA 20 (rata-rata). Minimal reward:risk 0,8.
- **Exit:** target tercapai, RSI 2 > 70, atau time stop 10 bar.

### 4. Skor 0–100
- **Trend:** kekuatan ADX (30), EMA stack (20), volume (15), relative strength vs BTC 20 bar (20), seberapa jauh harga dari EMA 20 (15). Dikurangi 15 bila BTC risk-off.
- **Reversion:** kedalaman Z-score (30), RSI 2 (25), ADX rendah (15), di atas EMA 200 (15), reward:risk ke mean (15).

### 5. Aturan backtest
- Sinyal dihitung saat close bar *i*, **entry di open bar *i+1*** (tanpa look-ahead).
- Stop dicek intrabar. Kalau harga gap di bawah stop, exit di harga open.
- Ukuran posisi: risiko 1% equity per trade, maksimal 100% equity (spot, tanpa leverage).
- Default fee 0,1% dan slippage 0,05% per sisi.
- **Hasil backtest hanya informasi, bukan syarat keputusan trading.** Di bawah 30 trade, sampel terlalu kecil untuk dinilai.

---

## Cara upload (GitHub Pages)

1. Login ke GitHub, lalu **New repository**, misalnya `tradesys-dual`, set **Public**.
2. **Add file → Upload files**: upload `index.html` dan `README.md`, lalu **Commit changes**.
3. Buka **Settings → Pages**. Di **Source** pilih *Deploy from a branch*, branch `main`, folder `/ (root)`, lalu **Save**.
4. Tunggu 1–2 menit. Aplikasi akan tersedia di:
   `https://<username>.github.io/tradesys-dual/`
5. Di HP: buka link tersebut di Chrome, lalu menu **⋮ → Add to Home screen** supaya bisa dibuka seperti aplikasi.

Hosting statis lain juga bisa (Netlify Drop, Vercel, Cloudflare Pages): cukup drag-and-drop folder berisi `index.html`.

Untuk update, upload ulang `index.html` di repo yang sama, lalu refresh browser (Ctrl+F5).

---

## Setup Telegram

1. Di Telegram, chat **@BotFather**, kirim `/newbot`, ikuti langkahnya, lalu salin **bot token**.
2. Kirim pesan apa saja ke bot kamu, lalu buka
   `https://api.telegram.org/bot<TOKEN>/getUpdates` dan salin angka `"chat":{"id": ... }`. Itu **chat ID**. Untuk grup/channel, tambahkan bot ke grup dulu; ID grup diawali `-100`.
3. Di aplikasi, buka **Config → Data and alerts**: isi token dan chat ID, centang **Send new live signals to Telegram**, lalu **Save and rescan**.
4. Buka tab **Alerts → Send test to Telegram** untuk mengecek.

Contoh format alert:
```
[Tradesys Edge Dual] TREND Breakout · SOLUSDT 4h · Entry 145.20 · Stop 139.80 · Target 156.00 · R:R 2.00 · Score 78
```

**Keamanan:** token hanya disimpan di localStorage browser kamu dan tidak ikut ter-upload ke GitHub. Jangan menulis token di dalam `index.html`, karena repo publik bisa dibaca siapa saja. Pakai bot khusus untuk alert.

---

## Batasan yang perlu diketahui

- **Scan otomatis hanya jalan selama halaman terbuka.** Halaman statis tidak bisa berjalan di background. Di HP, tab yang tertidur akan berhenti scan. Untuk scan 24 jam tanpa laptop atau HP menyala, perlu scheduler terpisah, misalnya GitHub Actions cron yang menjalankan engine yang sama lalu mengirim ke Telegram.
- **Akses Binance:** default endpoint `data-api.binance.vision` (khusus market data). Kalau status menunjukkan *Demo data · Binance unreachable*, ganti endpoint di Config (`api.binance.com` / `api1.binance.com`). Untuk pengguna di AS, pakai `api.binance.us`.
- **Mode demo** memakai harga sintetis dan diberi label jelas. Sinyal demo tidak pernah dikirim ke Telegram.
- Mode *All USDT pairs* mengabaikan stablecoin dan leveraged token, lalu mengambil top N berdasarkan volume 24 jam (default 40 pair, min. $5 juta).
- Rate limit Binance cukup longgar untuk scan ini (4 request paralel). Jangan set auto-scan terlalu sering di banyak tab sekaligus.

---

## Struktur kode (`index.html`)

- `ENGINE-START … ENGINE-END`: indikator (SMA, EMA, Wilder RSI/ATR/ADX, Bollinger, Donchian), `evalAt()` (regime + sinyal), `backtest()`, generator data demo. Bagian ini bisa dipakai ulang di Node.js untuk scheduler.
- Data layer: `api()`, `getKlines()` (paging hingga 3.000 bar, membuang candle yang belum close), `universe()`.
- UI: render tiap tab, chart SVG, kalkulator risiko, alerts, config (disimpan di localStorage).

---

*Bukan nasihat keuangan. Uji setiap strategi dengan data historis dan forward test dengan nominal kecil sebelum dipakai dengan modal sungguhan.*
