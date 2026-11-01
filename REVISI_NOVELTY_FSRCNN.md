# Revisi Novelty — Framework Penjadwalan Adaptif RCE untuk Pipeline FSRCNN pada big.LITTLE

## 1. Penilaian jujur: mengapa versi saat ini belum cukup untuk S2/publikasi

Rumusan saat ini:

> Bagaimana mekanisme penjadwalan CPU core asimetris untuk meminimalkan konsumsi energi dengan tetap menjamin batas minimum throughput real-time (24 FPS)?

Kelemahan struktural:

1. **RCE satu-kali ≠ adaptif.** Dokumen menyatakan biaya diukur sekali saat warm-up lalu tetap (cost table fixed), tetapi sekaligus mengklaim adaptif terhadap perubahan resolusi runtime. Keduanya kontradiktif. Jika resolusi berubah, tabel biaya basi. Reviewer pasti menolak klaim "adaptif".
2. **Minimasi energi tanpa aktuator energi.** Tidak ada DVFS, power capping, atau core on/off. Satu-satunya aktuator adalah pemetaan tugas big/little. Itu optimasi alokasi, bukan optimasi energi formal. Klaim energi butuh model energi + mekanisme slack-exploitation yang eksplisit.
3. **Throughput-constrained tapi tanpa kontroler.** Disebut throughput-oriented (Gustafson) dan target 24 FPS, tetapi tidak ada loop kontrol yang mengukur FPS aktual dan menindaklanjuti (tambah/kurangi big worker, migrasi stage). Akibatnya jaminan 24 FPS hanya harapan, bukan mekanisme.
4. **Beda vs baseline belum tajam.** Pipe-it = partisi statis berdasar deskriptor; AsyMo = rencana offline; Chronaki = critical-path kontinu (overhead tinggi); HeDPM/DynPipe = untuk terdistribusi/multi-GPU dengan biaya komunikasi eksplisit. Posisi RCE ("kalibrasi awal + cost-aware pull") ada di tengah, tetapi belum diformalkan sebagai tradeoff yang terukur: akurasi estimasi vs overhead vs energi vs throughput.

## 2. Perbaikan inti: jadikan RCE terpicu-peristiwa + kontrol berjangka (bukan sekali ukur)

Ganti "initial calibration" menjadi **event-triggered recalibration dengan drift detector**:

```
C_s(t) : estimasi biaya stage s (waktu/frame pada core referensi)
Ukur C aktual hanya jika dipicu; selain itu pakai tabel (nol syscall timing di steady-state).

Pemicu rekalibrasi (salah satu):
  a. perubahan dimensi input (resolusi/stride) terdeteksi,
  b. error prediksi |T_pred − T_aktual| > ε selama K window,
  c. drift detector (CUSUM/Page-Hinkley) pada residual antrean,
  d. FPS aktual < target − margin selama K window.

Update: C_s ← (1−β)·C_s + β·C_ukur, β∈(0,1]; catat usia estimasi + variansi.
```

Inilah reparasi pertama: **ringan seperti statis saat stabil, adaptif seperti dinamis saat rezim berubah**, dengan overhead terbatas (terbukti via hitung syscall/window).

## 3. Formalisasi masalah (pengganti rumusan deskriptif)

Variabel keputusan per window `t`: pemetaan stage→pool (`x_{s,p}`), jumlah worker aktif big/little (`b(t), l(t)`), ambang backpressure (`B_s`).

```
min   E(t) = Σ_p P_p · u_p(t)            (energi ≈ daya×utilisasi per klaster)
s.t.  FPS(t) ≥ 24 − δ                    (throughput constraint)
      Q_s(t) ≤ B_s ∀s                    (bounded queue, anti-overflow)
      starvation-free                     (tiap stage terlayani dalam H window)
      overhead_sched = O(1) per pengambilan tugas
```

Aktuator energi eksplisit: **slack-aware downshifting dengan hysteresis**:

```
jika FPS > 24 + m selama K_up window  → migrasikan 1 worker big→little / tidurkan 1 big worker
jika FPS < 24 − m selama K_dn window  → bangunkan/migrasikan kembali (K_dn < K_up agar protektif)
ambang naik ≠ ambang turun (anti-osilasi, analogi histeresis Tournaire)
```

## 4. Novelty yang diklaim (N1–N4, masing-masing ada persamaan + ablasi)

- **N1. Estimasi biaya terpicu-peristiwa untuk pipeline shared-memory.** Beda vs Pipe-it (deskriptor statis) dan vs Chronaki (pantau kontinu): ukur hanya saat rezim berubah. Metrik: error estimasi vs jumlah timing-syscall/window.
- **N2. Cost-aware pull scheduling terkoreksi antrean.** Prioritas = `f(C_s, Q_s, W_s)` (biaya × panjang antrean × waktu tunggu), bukan cost saja. Menjamin bounded queue + starvation-free di bawah backpressure; dibuktikan via invarian antrean + diuji pada bottleneck dekonvolusi layer-8.
- **N3. Kontrol throughput-berjangka hemat energi.** Downshift saat slack dengan hysteresis ganda (`m, K_up, K_dn`). Beda vs "pakai 4 big terus": hemat energi saat target sudah terlampaui tanpa langgar 24 FPS.
- **N4. Karakterisasi bottleneck FSRCNN sebagai artefak terukur.** Tabel biaya per-layer hasil kalibrasi pada RK3588S + distribusi waktu + titik bottleneck layer-8 dipublikasikan sebagai dataset/artefak reproduksibel, bukan sekadar narasi.

## 5. Rumusan masalah pengganti (satu utama + tiga turunan)

**Utama:**

> Bagaimana menjamin throughput minimum 24 FPS pada pipeline linear FSRCNN di ARM big.LITTLE dengan konsumsi energi minimum, memakai estimasi biaya yang hanya diukur ulang saat rezim berubah dan penjadwalan pull sadar-biaya terkoreksi antrean, sehingga diperoleh utilitas big/little yang seimbang, antrean terbatas, dan overhead penjadwalan O(1)?

**Turunan:**

1. Kapan mengukur ulang biaya agar error estimasi terbatas dengan overhead minimum? (N1)
2. Bagaimana aturan pull agar tidak terjadi backpressure/starvation pada bottleneck dekonvolusi? (N2)
3. Kapan menurunkan daya (downshift) tanpa melanggar 24 FPS dan tanpa osilasi? (N3)

## 6. Evaluasi yang membuat klaim di atas lolos reviewer

- Baseline wajib: (i) statis 4-big, (ii) statis hybrid, (iii) partisi statis ala Pipe-it (deskriptor), (iv) varian RCE tanpa N2/N3 (ablasi). Tanpa (iii)–(iv), novelty tidak terbukti.
- Variasikan: resolusi (480p→720p tiba-tiba untuk uji pemicu), worker {4,8,12}, target FPS {24,30}.
- Metrik: FPS (mean/P1), energi (Joule/frame via PowerTop + kalibrasi), utilisasi big/little, panjang antrean maks, stall/starvation count, syscall timing/window, error estimasi.
- Statistik: ≥10 ulangan, CI95%, uji beda. Satu angka "39%" atau "2,1×" tanpa variansi akan ditolak.
- Artefak: kode C/pthread + tabel biaya + skrip reproduksi dibuka.

## 7. Kalimat novelty satu paragraf untuk abstrak

> Berbeda dengan partisi statis berbasis deskriptor (Pipe-it/AsyMo) dan penjadwalan jalur-kritis kontinu (Chronaki), framework ini memakai estimasi biaya terpicu-peristiwa yang hanya mengukur ulang saat rezim berubah, dipadukan dengan penjadwalan pull sadar-biaya terkoreksi antrean dan kontrol downshift ber-hysteresis berjangka throughput, sehingga mencapai jaminan FPS dengan overhead O(1) dan konsumsi energi minimum pada pipeline FSRCNN di big.LITTLE.
