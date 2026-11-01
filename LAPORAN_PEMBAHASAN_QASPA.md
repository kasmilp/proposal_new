# PERANCANGAN SISTEM ADAPTIVE LOAD BALANCING PADA JARINGAN HETEROGEN FIBER OPTIC DAN STARLINK BERBASIS KUALITAS LINK DAN SESSION PERSISTENCE

## Laporan Pembahasan Akademik: Titik Kontribusi dan Research Gap

**Nama kerja algoritma:** QASPA — Quality-Aware Session-Preserving Adaptive Allocator
**Studi kasus implementasi:** MikroTik Per Connection Classifier (PCC)
**Klaim generalitas:** Core algoritma device-agnostic + Southbound Adapter spesifik perangkat

---

## 1. Ringkasan Eksekutif

Penelitian ini berangkat dari keterbatasan mendasar PCC konvensional: distribusi koneksi bersifat statis berbasis hashing dan tidak mempertimbangkan kualitas aktual link. Ketika Fiber Optic (FO) atau Starlink mengalami degradasi (latency, jitter, packet loss, penurunan throughput), koneksi baru tetap diarahkan ke ISP yang terdegradasi sampai konfigurasi diubah manual. Di sisi lain, pemindahan jalur secara agresif merusak sesi aktif.

Kontribusi ilmiah penelitian ini **bukan** pada pembuatan dashboard monitoring atau script otomatis pengganti konfigurasi MikroTik. Kontribusi ada pada **fungsi keputusan adaptif** yang memetakan metrik QoS multivariat yang berfluktuasi menjadi distribusi bucket PCC integer dengan jaminan session persistence dan anti-oscillation.

Fokus penelitian (research focus) yang dibidik — dirumuskan sebagai keterbatasan pendekatan yang ada, bukan klaim ketiadaan absolut yang memerlukan Systematic Literature Review lengkap untuk dibuktikan:

> Pendekatan yang tersedia umumnya menangani sebagian aspek secara terpisah — penilaian kualitas, failover, atau distribusi statis — sehingga integrasi kelima kemampuan berikut secara bersamaan masih terbatas, khususnya pada konteks heterogen FO vs LEO-satelit dengan enforcement PCC: (1) penilaian kualitas link multivariat secara kontinu, (2) pembedaan kondisi degraded vs failed dengan aksi berbeda, (3) pengaturan alokasi khusus koneksi baru, (4) pengendalian oscillation yang terukur, (5) pemetaan ke bucket PCC integer.

---

## 2. Karakteristik FO vs Starlink dan Implikasinya pada Desain

| Aspek | Fiber Optic | Starlink (LEO) |
|---|---|---|
| Latency baseline | Rendah (1–15 ms domestik), stabil | Lebih tinggi (20–60 ms+), bervariansi |
| Jitter | Kecil | Sedang–besar, episodik (handover satelit, cuaca, obstruksi) |
| Packet loss | Rendah, jarang burst | Burst episodik |
| Throughput | Stabil, simetris relatif | Downlink baik, uplink asimetris dan berfluktuasi |
| Ketersediaan | Tinggi | Tinggi tetapi dengan micro-outage dan CGNAT |

Implikasi desain:

1. Algoritma tidak boleh mengasumsikan FO selalu baik. Harus menangani degradasi dua arah.
2. Starlink membutuhkan smoothing dan hysteresis. Tanpa itu, controller akan flapping setiap menit.
3. Pengukuran throughput aktif yang agresif akan mengganggu traffic itu sendiri. Harus dibedakan pengukuran pasif (interface counter) dan aktif periodik jarang.

---

## 3. Model Sistem dan Notasi

Himpunan ISP: `I = {1: FO, 2: Starlink}`.

Untuk tiap ISP `i` pada interval pengukuran `t` dengan periode `T_probe` (contoh 5–10 detik):

- `L_i(t)`: rata-rata RTT hasil active probe (detik/ms)
- `J_i(t)`: jitter, didefinisikan sebagai mean absolute difference atau stddev RTT
- `P_i(t)`: packet loss dalam persen [0,100]
- `Th_i(t)`: estimasi throughput tersedia (Mbps)
- `A_i(t)`: ketersediaan biner {0,1} dari probe gateway + DNS/HTTP

Himpunan koneksi dibagi dua:

- `C_active(t)`: koneksi dengan connection-mark yang sudah terbentuk
- `C_new(t)`: koneksi baru yang belum punya mark dan akan diklasifikasikan

Variabel keputusan:

- `w_i(t)` dalam [0,1]: bobot kontinu alokasi koneksi baru, `sum w_i = 1`
- `n_i(t)` integer: jumlah bucket PCC untuk ISP `i`, `sum n_i = D` dengan `D` denominator PCC
- `s_i(t)` dalam {NORMAL, DEGRADED, FAILED}: state kualitas ISP

Tujuan umum (informal, diformalkan per modul di bawah):

> Maksimalkan kualitas terbobot untuk koneksi baru, minimalkan gangguan sesi aktif, dan minimalkan frekuensi perubahan konfigurasi.

---

## 4. Titik Kontribusi Ilmiah dan Formulasi

### Kontribusi C1: Link Quality Assessment (LQA) Multivariat dengan Smoothing

**Gap yang ditutup:** PCC tidak punya input kualitas. Threshold tunggal pada satu metrik (misalnya hanya ping) tidak cukup untuk bedakan degradasi throughput vs degradasi delay/loss.

**Formulasi:**

Langkah 1 — Smoothing dengan EWMA untuk setiap metrik mentah X:

```
S_i(t) = alpha * X_i(t) + (1 - alpha) * S_i(t-1),  0 < alpha < 1
```

Rekomendasi awal `alpha = 0.2 - 0.4`. Nilai kecil = lebih halus, tahan terhadap spike Starlink, tetapi lebih lambat merespons.

Langkah 2 — Normalisasi ke [0,1], semakin besar semakin baik:

```
qL_i = 1 - min(L_i / L_max, 1)
qJ_i = 1 - min(J_i / J_max, 1)
qP_i = 1 - P_i / 100
qT_i = min(Th_i / Th_ref, 1)
```

`L_max, J_max, Th_ref` diturunkan dari fase baseline pengukuran, bukan konstanta global. Contoh: `L_max` = persentil-95 RTT gabungan selama baseline.

Langkah 3 — Skor komposit:

```
Q_i(t) = wL*qL_i + wJ*qJ_i + wP*qP_i + wT*qT_i
dengan wL + wJ + wP + wT = 1, masing-masing >= 0
```

**Titik kontribusi C1:** prosedur kalibrasi `L_max, J_max, Th_ref`, pemilihan `alpha`, dan analisis sensitivitas bobot `w` untuk profil FO vs LEO. Bobot tidak boleh asal. Tesis harus memuat tabel sensitivitas: perubahan bobot mengubah ranking ISP pada skenario apa.

**Cara menggugurkan (falsifikasi):** ablasi tanpa EWMA vs dengan EWMA pada skenario fluktuasi Starlink. Hipotesis: tanpa smoothing, jumlah flap dan false-positive degradasi naik signifikan.

---

### Kontribusi C2: ISP Degradation Detection 3-State dengan Hysteresis

**Gap yang ditutup:** Failover biner up/down tidak bisa bedakan ISP lambat-tapi-masih-bisa-jalan vs ISP putus. Akibatnya pilihan hanya dua ekstrem: tetap bagi rata atau putus total.

**Formulasi:**

State `s_i(t)` ditentukan dua lapis:

Lapis keras (hard failure):

```
jika A_i = 0 selama K_fail probe berturut-turut -> s_i = FAILED
```

Lapis lunak (soft degradation):

```
jika Q_i < Tau_low selama K_deg window berturut-turut -> s_i = DEGRADED
atau jika (Q_i(t-1) - Q_i(t)) > Delta_drop tiba-tiba -> s_i = DEGRADED
```

Recovery dengan hysteresis (threshold naik berbeda dengan threshold turun):

```
jika s_i = DEGRADED dan Q_i > Tau_high selama K_rec -> s_i = NORMAL
dengan Tau_high > Tau_low (contoh Tau_low=0.55, Tau_high=0.65)
jika s_i = FAILED dan A_i=1 dan Q_i > Tau_high selama K_rec -> s_i = NORMAL atau DEGRADED
```

**Titik kontribusi C2:** pemisahan DEGRADED vs FAILED dengan aksi berbeda (lihat C3 dan C4). Threshold ganda + `K` siklus stabil adalah inti anti-flap di level deteksi.

**Cara menggugurkan:** ukur detection delay, false positive rate, dan flapping state per jam pada skenario loss 2% vs 10% vs putus total. Tanpa hysteresis, state akan berosilasi di sekitar threshold.

---

### Kontribusi C3: Adaptive Traffic Allocation Proporsional Kualitas

**Gap yang ditutup:** MCDM / fuzzy path selection umumnya menghasilkan bobot kontinu dan mengasumsikan flow bebas dipindah. Tidak menjawab kendala PCC integer dan persistensi sesi.

**Formulasi:**

Bobot mentah dari kualitas (hanya untuk NORMAL dan DEGRADED):

```
W_i = 0                          jika s_i = FAILED
W_i = max(0, Q_i - Q_min)^gamma  jika s_i = NORMAL atau DEGRADED
```

`Q_min` adalah kualitas minimum agar masih layak terima new-conn (contoh 0.30). `gamma > 1` (contoh 1.5–2.0) menekan ISP berkualitas rendah secara non-linear, bukan proporsional linear.

Normalisasi:

```
w_i = W_i / sum_j(W_j)
```

Kasus khusus:

- Satu FAILED: `w_failed = 0, w_lain = 1`
- Keduanya DEGRADED: tetap proporsional, bukan failover
- Keduanya FAILED: fallback ke rute default / policy darurat yang didefinisikan eksplisit

**Titik kontribusi C3:** fungsi `f(Q) = max(0, Q - Q_min)^gamma` dengan analisis pengaruh `gamma` dan `Q_min`. Ini yang membuat DEGRADED ringan masih dapat 20–30% new-conn, sedangkan DEGRADED berat mendekati 0% tanpa langsung failover.

**Cara menggugurkan:** bandingkan alokasi linear (`gamma=1`) vs non-linear pada skenario degradasi asimetris. Ukur persentase new-conn yang salah jalan ke ISP buruk dan dampaknya pada latency/jitter agregat.

---

### Kontribusi C4: Session Persistence sebagai Constraint Eksplisit

**Gap yang ditutup:** Optimasi path yang memindah flow aktif merusak TCP, TLS session, VoIP call, dan transaksi perbankan. Di PCC + NAT, pemindahan membutuhkan re-mark dan berpotensi reset koneksi.

**Formulasi:**

```
C_active(t) tidak dievaluasi ulang terhadap w_i(t) yang baru,
kecuali s_i = FAILED untuk ISP tempat koneksi tersebut berada.

C_new(t) dialokasikan sesuai w_i(t) / n_i(t) terbaru.
```

Metrik utama:

```
SPR (Session Persistence Ratio) =
  jumlah koneksi yang tetap di ISP awal / jumlah koneksi yang wajib tetap
```

Koneksi yang berada di ISP FAILED dan memang harus pindah tidak dihitung sebagai pelanggaran SPR. Jumlah koneksi terganggu dan recovery time dilaporkan terpisah.

**Titik kontribusi C4:** menjadikan SPR sebagai constraint optimasi berdampingan dengan QoS, bukan metrik tambahan. Di implementasi MikroTik, ini berarti hanya rule untuk `connection-state=new` yang di-rewrite. Rule `established,related` tidak disentuh.

**Cara menggugurkan:** ukur SPR dan jumlah sesi terputus pada tiap skenario. Hipotesis: QASPA unggul besar atas PCC saat degradasi, tetapi imbang saat normal. Jika adaptif merusak SPR saat normal, desain gagal.

---

### Kontribusi C5: Anti-Oscillation Controller dengan Bound

**Gap yang ditutup:** Controller reaktif tanpa guard akan rewrite mangle setiap ada spike Starlink. Ini menambah churn, memenuhi log, dan berisiko transient mis-routing.

**Formulasi:**

Update konfigurasi PCC hanya jika ketiga syarat terpenuhi secara konjungsi:

```
1. max_i |w_new_i - w_old_i| > delta_w   (contoh 0.10)
2. t - t_last_update > Tc                (contoh 30-60 detik)
3. state s_i stabil selama K_stable siklus (contoh 2-3 siklus)
```

**Titik kontribusi C5:** tradeoff curve yang terukur antara responsivitas dan stabilitas: `delta_w` dan `Tc` kecil = cepat tetapi flap; besar = stabil tetapi lambat. Tesis harus memuat kurva ini, bukan satu titik parameter.

**Cara menggugurkan (ablasi wajib):** QASPA tanpa C5 vs dengan C5 pada skenario fluktuasi bersamaan. Metrik: flap count per jam, jumlah rewrite, detection delay, dan QoS agregat. Tanpa eksperimen ini, C5 dianggap tempelan.

---

### Kontribusi C6: Dynamic PCC Configuration (Adapter Spesifik MikroTik)

**Gap yang ditutup:** Bobot kontinu `w_i` tidak bisa langsung dipakai PCC yang bekerja pada bucket integer.

**Formulasi:**

Diberikan denominator `D` (contoh 10, 12, atau 20):

```
n_1 = round(w_1 * D)
n_2 = D - n_1
dengan n_i >= n_min kecuali s_i = FAILED (contoh n_min = 1 agar probe tetap jalan)
```

Contoh `D=10`: `w=[0.6,0.4]` menjadi `n=[6,4]`, artinya 6 PCC classifier ke FO dan 4 ke Starlink.

Prosedur update atomik:

1. Hitung `n_i` baru dari `w_i`
2. Jika sama dengan konfigurasi berjalan, tidak ada aksi
3. Jika berbeda dan lolos guard C5, generate ulang rule mangle prerouting PCC `n/D` untuk `connection-state=new` beserta connection-mark, routing-mark, dan tabel routing terkait
4. Verifikasi via read-back API sebelum menandai `t_last_update`

Kompleksitas: O(D) saat update, O(1) per paket setelah itu. Overhead controller didominasi probing, bukan forwarding.

**Titik kontribusi C6:** kuantisasi sadar-error: laporkan error kuantisasi `|n_i/D - w_i|` untuk tiap `D` dan pengaruhnya pada distribusi aktual. Ini kecil tetapi perlu agar klaim reproducible.

---

## 5. Arsitektur Sistem

```
[ISP FO] [ISP Starlink]
    \      /
[Monitor: active probe + passive interface counter] -> Time-series DB
              |
      [C1 LQA + C2 FSM degradasi]
              |
      [C3 Weight controller + C5 Anti-oscillation]
              |
      [Enforcer API abstrak: setWeights(w1,w2)]
         /              \
MikroTik Adapter   Adapter generik (OpenWrt/SDN, definisi interface saja)
         |
[C4 Session tracker: SPR, flap, detection time, response time]
```

Prinsip pemisahan:

- Core (C1–C5): murni algoritma, tidak menyebut sintaks MikroTik. Ini yang diuji dan diklaim device-agnostic.
- Adapter (C6): satu-satunya modul yang menyebut `/ip firewall mangle`, `connection-mark`, `routing-mark`, RouterOS API/SSH.

Controller ditempatkan di luar MikroTik (mini-PC / RPi / VM) agar logika, logging, dan analisis statistik reproducible. MikroTik tetap sebagai forwarding plane.

---

## 6. Metodologi Eksperimen

### 6.1 Skenario pembanding

- Skenario A (baseline): PCC statis, tanpa penilaian kualitas adaptif
- Skenario B (usulan): QASPA lengkap (C1–C6)
- Skenario C (ablasi, wajib untuk klaim ilmiah):
  - B tanpa C5 (tanpa anti-oscillation)
  - B dengan `gamma=1` (alokasi linear)
  - B tanpa EWMA

### 6.2 Kondisi uji

1. Kedua ISP normal
2. FO degradasi (inject delay/jitter/loss terkontrol)
3. Starlink degradasi
4. Salah satu loss tinggi (5–10%)
5. Salah satu latency/jitter tinggi
6. Salah satu putus total
7. Kedua fluktuasi bersamaan (pola sinusoidal/acak — uji C5)
8. Baseline karakterisasi alami Starlink jangka panjang (untuk RQ1, bukan untuk uji terkontrol)

Impairment terkontrol menggunakan `tc-netem` pada Linux bridge sebelum MikroTik agar reproducible. Menunggu Starlink buruk secara alami tidak reproducible dan tidak boleh jadi satu-satunya bukti.

### 6.3 Beban traffic

- `iperf3`: throughput
- Traffic UDP seperti VoIP + ICMP: latency/jitter/loss
- HTTP short-lived connections: distribusi new-conn
- Streaming session panjang: SPR dan disruption

### 6.4 Repetisi dan statistik

Minimal 10–30 repetisi per skenario. Laporkan mean, confidence interval, dan uji beda (t-test atau Wilcoxon). Grafik single-run tidak diterima untuk Q1/Q2.

---

## 7. Parameter Evaluasi dan Hipotesis

| Parameter | Hipotesis QASPA vs PCC |
|---|---|
| Latency agregat | Turun signifikan saat degradasi asimetris, imbang saat normal |
| Jitter | Turun saat ISP jitter-tinggi dihindari untuk new-conn |
| Packet loss | Turun karena new-conn tidak dikirim ke ISP loss-tinggi |
| Throughput agregat | Naik atau imbang, tidak turun saat normal |
| Distribusi trafik antar-ISP | Mendekati `w_i`, bukan 50:50 buta |
| Jumlah perubahan konfigurasi PCC | Rendah dan bounded berkat C5 |
| Waktu deteksi degradasi | Terukur, tradeoff dengan `Tc` dan `K` dilaporkan |
| Waktu respons controller | Terukur dari probe sampai enforce |
| Session Persistence Ratio | >0.95 kecuali FAILED |
| Jumlah koneksi terganggu | <50% dibanding failover agresif |
| Recovery time | Terukur setelah ISP pulih, tanpa flap |

Kegagalan yang harus dilaporkan jujur jika terjadi: overhead saat normal, false positive pada Starlink, dan error kuantisasi pada `D` kecil.

---

## 8. Peta Jalan ke Q1/Q2 dan Risiko

Pecah menjadi dua paper:

1. Paper karakterisasi: performa FO vs Starlink heterogen. Target jurnal measurement / telecommunication systems.
2. Paper algoritma: formulasi QASPA + FSM + kuantisasi PCC + anti-oscillation + evaluasi komparatif dan ablasi. Target Computer Networks / JNCA / IEEE Access / TNSM.

Risiko desk-reject dan mitigasi:

- Bobot dan threshold asal pilih -> mitigasi dengan sensitivity analysis dan tradeoff curve
- Satu testbed -> mitigasi dengan abstraksi Enforcer API dan definisi adapter generik
- Throughput aktif mengganggu pengukuran -> mitigasi dengan probe jarang + counter pasif
- Klaim novelty berlebihan atas EWMA/hysteresis saja -> klaim novelty pada kombinasi FSM 3-state + kuantisasi PCC + pinning sesi + guard konjungsif yang dievaluasi bersama

---

## Lampiran: Daftar Notasi

`L_i, J_i, P_i, Th_i, A_i, Q_i, qL, qJ, qP, qT, wL, wJ, wP, wT, alpha, L_max, J_max, Th_ref, Tau_low, Tau_high, K_fail, K_deg, K_rec, Q_min, gamma, w_i, W_i, D, n_i, n_min, delta_w, Tc, K_stable, s_i, SPR`
