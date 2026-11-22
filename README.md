# Tradesys Edge Dual

Web app sistem trading crypto **Trend Following + Mean Reversion** untuk **Binance Spot (long-only)**.
Satu file `index.html` tanpa build, tanpa server, tanpa aplikasi berbayar. Data diambil langsung dari API publik Binance di browser.

> Aplikasi ini hanya memberi **sinyal dan alert**. Tidak ada order yang dikirim ke exchange dan tidak butuh API key Binance.

---

## Fitur

| Tab | Isi |
|---|---|
| **Dashboard** | Ringkasan pasar (BTC, regime harian, jumlah sinyal, komposisi regime), chart dengan garis plan, **kartu Trading Plan** (Entry, SL, TP1–TP3, ukuran posisi, manajemen), papan sinyal |
| **Scanner** | Scan watchlist atau semua pair USDT (top N berdasarkan volume), filter Signals / Trend / Reversion / All |
| **Chart** | Candlestick + EMA 20/50/200 + Bollinger 20,2 + panel ADX, penanda sinyal historis, level entry/stop/target |
| **Backtest** | Router (keduanya), Trend saja, atau Reversion saja, pada 1.000–3.000 bar, dengan fee dan slippage |
| **Risk** | Kalkulator ukuran posisi (risk %, batas alokasi, target 1R/2R/3R) + aturan portofolio |
| **Alerts** | Log sinyal baru, **Kirim sinyal tes** dan **Kirim pesan tes** ke Telegram, salin semua, auto-scan 1 jam / 4 jam |
| **Config** | Semua parameter strategi, risiko, biaya, endpoint Binance, Telegram |

Timeframe scan: **1H, 4H, 1D** (ditampilkan dengan huruf kapital). Bahasa: tombol **EN / ID** di header (pilihan tersimpan di browser, juga dipakai untuk pesan Telegram).

---

## Trading plan

Setiap sinyal otomatis punya plan lengkap:

| | Trend following | Mean reversion |
|---|---|---|
| **Entry** | Close bar sinyal; zona entry = entry − 0,25×ATR s/d entry | Sama |
| **Stop loss** | Entry − 2×ATR (= 1R) | Entry − 1,5×ATR (= 1R) |
| **TP1** | +1R · ambil 33% · stop digeser ke entry | Setengah jalan ke rata-rata · ambil 50% · stop ke entry |
| **TP2** | +2R · ambil 33% | Rata-rata 20 bar (SMA 20) · tutup sisa |
| **TP3** | Sisa 34% trailing 3×ATR dari high (referensi 3R) | — (keluar juga jika RSI 2 > 70 atau setelah 10 bar) |
| **Ukuran posisi** | Modal × risiko % ÷ (entry − SL), dibatasi alokasi maks per posisi | Sama |
| **Berlaku** | 2 bar; batal jika close di bawah SL; jangan kejar di atas entry + 0,5×ATR | Sama |

Persentase ditampilkan di kartu plan, papan sinyal, scanner, chart, dan Telegram: **Sekarang → Entry** (berapa % harga harus bergerak ke entry, diperbarui live), **Entry → SL**, dan **Entry → TP1/TP2/TP3**.

Backtest memakai plan yang sama persis (scale-out di TP1/TP2, stop ke entry, trailing), jadi hasilnya mencerminkan cara plan dijalankan.

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

## Harga sekarang (live)

- Harga terbaru diambil dari Binance setiap **15 detik** selama halaman terbuka (`/api/v3/ticker/price`).
- Tampil di: KPI BTC, header chart dan **kotak biru di sumbu chart**, kartu Trading Plan, papan sinyal (kolom *Sekarang*), scanner, dan pesan Telegram.
- Kartu plan menunjukkan **status entry** berdasarkan harga sekarang:
  - *Di zona entry* / *Di atas entry, masih valid* (hijau)
  - *Di bawah zona entry* / *Terlalu jauh, jangan kejar* (kuning)
  - *Di bawah stop loss: plan batal* / *TP1 sudah tercapai: lewati* (merah)
- Sinyal tetap dihitung dari **bar yang sudah close**. Harga live hanya untuk eksekusi dan monitoring.

## Probabilitas & tingkat keyakinan

Angka ini **dihitung dari data historis**, bukan tebakan:

1. Untuk setiap pair bersinyal, aplikasi mengambil 2.000 bar di timeframe yang sama.
2. Semua sinyal lampau dengan strategi yang sama disimulasikan memakai plan yang sama (entry di open berikutnya, SL, TP1, lalu stop pindah ke entry, kemudian TP2). Sinyal yang saling tumpang tindih tidak dihitung ganda.
3. Hasilnya:
   - **TP1 %**: persentase sinyal yang mencapai TP1 sebelum SL.
   - **TP2 %**: persentase sinyal yang mencapai TP2.
   - **SL duluan %**: persentase sinyal yang kena SL sebelum TP1.
   - **n**: jumlah sampel.
4. Kalau sampel di pair itu < 15, dipakai gabungan dari semua pair bersinyal dengan strategi yang sama (diberi keterangan).

**Keyakinan** memakai batas bawah konservatif (Wilson 95%) dari peluang TP1:

| Keyakinan | Syarat |
|---|---|
| Tinggi | batas bawah TP1 ≥ 50% |
| Sedang | batas bawah TP1 ≥ 35% |
| Rendah | di bawah itu |
| Rendah (sampel sedikit) | n < 10 |

Ini frekuensi masa lalu, **bukan jaminan**. Kondisi pasar bisa berubah.

---

## Multi-timeframe (MTF)

Setiap pair dicek juga di timeframe satu tingkat di atasnya:

| Timeframe scan | Timeframe besar (HTF) |
|---|---|
| 1H | 4H |
| 4H | 1D |
| 1D | 1W |

- **HTF Bullish**: close di atas EMA 50 yang sedang naik. **Bearish**: close di bawah EMA 50 yang sedang turun. Selain itu **Netral**.
- Hanya bar HTF yang **sudah close** yang dipakai, jadi tidak ada look-ahead.
- **Sinyal trend** wajib HTF Bullish. **Sinyal reversion** diblokir saat HTF Bearish.
- Filter ini juga dipakai di backtest dan perhitungan probabilitas. Bisa dimatikan di Pengaturan → Regime router.
- Status HTF tampil di papan sinyal, scanner, kartu plan, chart, dan Telegram.

## SMC & price action di chart

Layer bisa dinyalakan/dimatikan dengan tombol di bawah chart (**Struktur, Order block, FVG, Candle**):

- **Struktur**: swing high/low (fractal 3 bar) diberi label **HH / HL / LH / LL**. Garis **BOS** (break searah trend) dan **CHoCH** (break berlawanan arah, garis putus-putus). Garis **EQ 50%** = equilibrium range swing terakhir (atas = premium, bawah = discount).
- **Order block**: candle berlawanan terakhir sebelum BOS/CHoCH. Hanya OB yang belum ditembus close yang ditampilkan (hijau = bullish, merah = bearish).
- **FVG**: celah 3 candle yang belum terisi.
- **Candle**: **E** = engulfing, **P** = pin bar (hammer / shooting star). Hijau = bullish, merah = bearish. Hanya 12 bar terakhir.

Kartu Trading Plan punya kotak **SMC & price action** berisi checklist konfluensi (x/6):
1. HTF bullish
2. Struktur pasar bullish
3. Harga di area discount
4. Di dalam atau dekat order block bullish
5. Ada FVG bullish di bawah harga
6. Ada pola candle bullish di 2 bar terakhir

SMC & PA adalah **informasi konfluensi**, bukan syarat sinyal. Skor dan probabilitas tidak diubah oleh checklist ini.

---

## Audit keandalan (22 Nov 2026)

**Uji teknis (lulus):**
- Look-ahead: 2.160 titik diuji ulang dengan data masa depan dipotong, termasuk 43 sinyal. Hasilnya sama persis, 0 selisih.
- Invarian backtest: 398 trade, 0 pelanggaran (urutan waktu, posisi tidak tumpang tindih, R tidak melewati −1R kecuali karena gap).
- Probabilitas sekarang dihitung dari mesin backtest yang sama, jadi angkanya identik dengan backtest.

**Perbaikan dari audit:**
- Ganti timeframe saat scan berjalan sebelumnya bisa mencampur data dua timeframe. Sekarang pengaturan dikunci per scan, dan scan baru diantrekan.
- Satu simbol salah di tab Chart sebelumnya membuat harga live berhenti update (Binance menolak seluruh batch). Sekarang hanya simbol valid yang diminta.
- Pair yang di-halt atau datanya basi sekarang dilewati.
- Request yang gagal (timeout, 5xx, 429) dicoba ulang sekali, dengan menghormati Retry-After.
- Ukuran posisi dibulatkan ke lot size Binance, dan ada peringatan kalau di bawah minimum notional.
- Telegram diberi jeda 1,1 detik antar pesan, sesuai limit per chat.
- Filter MTF lebih ketat: sinyal trend hanya lolos jika HTF terkonfirmasi bullish. Data HTF yang kurang tidak lagi diloloskan.
- Input simbol disanitasi.
- Stablecoin varian USD (mis. RWUSD) dikeluarkan dari universe.

**Uji strategi dengan data Binance asli** (10 pair besar, 3.000 bar, fee 0,1% + slippage 0,05% per sisi, risiko 1%/trade):

| Timeframe | Strategi | Trade | Win | Profit factor | Ekspektansi | t-stat |
|---|---|---|---|---|---|---|
| 4H | Trend | 164 | 43% | 0,85 | −0,08R | −0,87 |
| 4H | Reversion | 75 | 59% | 0,73 | −0,10R | −1,12 |
| 1D | Trend | 224 | 44% | 1,31 | +0,14R | 1,37 |
| 1D | Reversion | 82 | 68% | 0,97 | −0,01R | −0,13 |

Kesimpulan: di **4H** kedua strategi belum punya edge, ekspektansinya negatif setelah biaya. Di **1D**, trend following positif tapi belum signifikan secara statistik (t < 2). Mean reversion mendekati impas. Gunakan sebagai alat bantu analisis dan forward test dengan nominal kecil, bukan sebagai sinyal otomatis.

**Batasan yang masih ada:**
- Auto-scan hanya berjalan selama halaman terbuka.
- Backtest dan probabilitas tidak memakai filter BTC risk-off, hanya filter MTF per pair.
- Hasil di atas bisa berubah setiap kali data bertambah.

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
4. Buka tab **Alert**:
   - **Kirim pesan tes**: cek koneksi bot.
   - **Kirim sinyal tes**: mengirim trading plan lengkap dari sinyal yang sedang dipilih (atau contoh dari BTC bila belum ada sinyal), diberi label *SINYAL TES*.
5. Dari kartu Trading Plan di Dashboard, tombol **Kirim ke Telegram** mengirim plan pair yang sedang dipilih.

Contoh pesan (Bahasa Indonesia):
```
🔵 TREND · BREAKOUT
SOLUSDT · 4H · Skor 78/100

Entry: 145.20 (zona 144.53 – 145.20)
Stop loss: 139.80 (-3.7%)
TP1: 150.60 (+3.7%) · ambil 33%
TP2: 156.00 (+7.4%) · ambil 33%
TP3: sisanya trailing 3×ATR (ref 161.40)
R:R ke TP2: 2.00
Jumlah: 18.52 SOL ≈ 2,500 USDT · risiko 100 USDT

Plan
1. Di TP1 ambil 1/3 dan geser stop ke harga entry.
2. Di TP2 ambil 1/3 lagi.
3. Sisa 1/3 trailing 3×ATR di bawah high; keluar jika close di bawah EMA 50.

Berlaku 2 bar · batal jika close di bawah SL
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
