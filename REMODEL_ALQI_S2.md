# Remodel ke Level Tesis S2 / Publikasi — ALQI + Histeresis Asimetris + PCC Dinamis

Subjek: proposal Kasmil Papalia (D082252011), RT/RW Net Waeura Buru, FO IndiHome + Starlink, MikroTik RB1100AHx4 + Server Controller eksternal.

## 1. Verdict jujur

Proposal ini **sudah layak tesis S2** dan **layak publikasi Q3/Q4/konferensi baik** apa adanya: masalah nyata, lokasi riil, H1/H2 falsifiable, desain kuasi-eksperimental + staging terkendali, SOTA 12+ rujukan. Keraguan Anda soal "terlihat standar" benar hanya untuk target **Q1/Q2**: di level itu, kombinasi SAW + hysteresis + PCC akan dibaca sebagai integrasi teknik yang sudah ada kecuali 7 penguat di bawah dipenuhi. Kabar baiknya: tidak perlu ganti topik — cukup formalkan dan buktikan 7 hal ini.

## 2. Yang sudah kuat (jangan dirombak)

- Closed-loop offloading controller (ukur via API/SNMP → filter → skor → injeksi PCC) dengan beban komputasi di luar router.
- Urutan pemrosesan eksplisit: histeresis pada metrik mentah → SAW → ALQI → rasio (subbab 1.2.10). Ini keputusan desain yang bisa dipertahankan dan diablasi.
- H1 (flapping turun) dan H2 (QoS naik) dengan dua pembanding: A PCC statis + B ALQI tanpa hysteresis. Struktur ablasi ini sudah benar.
- Staging terkendali dengan tc/netem untuk transien 15 detik + degradasi FO bertahap sebagai pelengkap validitas internal.
- Stratifikasi hujan/cerah dan sibuk/normal + effect size + uji prasyarat normalitas.

## 3. Tujuh penguat agar naik ke Q2 (tanpa keluar cakupan proposal)

### P1. Formalkan pinning sesi (SPR) — saat ini hilang
Proposal menjamin konsistensi per-koneksi khas PCC (1.1, 1.2.4) tetapi tidak pernah merumuskan aturan apa yang boleh di-rewrite. Tambahkan satu aksioma dan satu metrik:

```
Hanya aturan mangle connection-state=new yang ditulis ulang.
Aturan established/related tidak disentuh; pengecualian hanya jika jalur FAILED total.
SPR = sesi wajib-tetap yang tetap / total wajib-tetap; laporkan bersama flapping count.
```

Tanpa ini, reviewer curiga perubahan rasio merusak sesi aktif. Dengan ini, H2 tidak bisa dicapai dengan "memutus sesi jelek".

### P2. Formalkan proyeksi rasio ke slot PCC integer + error-nya
Rumus rasio persen (2.5.5) berhenti sebelum realitas RouterOS: slot PCC integer. Tambahkan:

```
n_FO = round(r_FO · N), n_SL = N − n_FO; bound |n/N − r| ≤ 1/(2N) (+koreksi N ganjil)
Laporkan N yang dipakai, error kuantisasi, dan distribusi new-conn aktual vs target r.
```

Ini kecil tetapi wajib untuk reproducibility — dan menjadi jawaban atas "dipetakan ke pecahan bilangan bulat terdekat" yang saat ini menggantung.

### P3. Ubah bobot/threshold dari "kalibrasi awal" menjadi analisis sensitivitas
Batasan 3–4 proposal jujur mengakui bobot SAW dan ambang dari literatur/kalibrasi, bukan optimasi formal. Itu dapat diterima S2 **hanya jika** dilengkapi:

- Tabel sensitivitas: ubah satu bobot ±0.1 → peringkat FO vs LEO berubah pada kondisi apa?
- Kurva tradeoff ambang: `Th_degradasi/Th_pemulihan/k` vs detection delay vs flap/jam pada data staging.
- Tanpa ini, angka contoh (bobot 0.3/0.25/0.25/0.2; ALQI 91.3 vs 53.3) terbaca ilustratif, bukan hasil.

### P4. Buktikan klaim urutan "histeresis dulu, SAW kemudian"
Subbab 1.2.10 mengklaim transien lebih mudah terdeteksi sebelum agregasi SAW. Klaim ini orisinal dan bisa jadi novelty metodologis — tetapi harus diablasi:

```
Varian X (usul): hysteresis(metrik) → SAW → ALQI
Varian Y: SAW(metrik mentah) → hysteresis(skor) → ALQI
Bandingkan pada transien 15 detik sintetis: false-positive, detection delay, flap.
```

Siapa pun yang menang, paper mendapat hasil. Jika X menang, itu kontribusi. Jika seri, klaim urutan dicabut jujur.

### P5. Beri jaminan terbatas pada histeresis (bound, bukan sekadar prinsip)
Tournaire dikutip sebagai prinsip; jadikan jaminan terukur pada sistem Anda:

```
Update rasio HANYA saat transisi status (bukan tiap polling) + dwell k sampel.
Konsekuensi: flap/jam ≤ 3600/(interval_polling · k) apa pun noise-nya; zona [Th_deg, Th_pem] = dead zone tanpa keputusan.
Laporkan bound teoritis vs flap aktual lapangan + staging.
```

### P6. Pisahkan dua lapis ketidakstabilan LEO dalam evaluasi
Proposal sudah mengidentifikasi dengan tepat: rain fade (menit) vs handover 15 detik (detik). Evaluasi harus memisahkan keduanya:

- Transien cepat → uji H1 (flapping) pada staging 15 detik.
- Degradasi lambat → uji H2 (QoS) pada lapangan hujan vs cerah.
- Klaim "Starlink sebagai upper bound" (Bab I) hanya sah jika filter yang lolos uji 15 detik juga diuji pada degradasi FO lambat — rencanakan eksplisit pasangan uji ini.

### P7. Framing generalisasi tanpa keluar batasan
Batasan 1,5,7 (dual-ISP, satu site, MikroTik-only) akan dipakai reviewer untuk menolak generalisasi. Netralkan dengan satu paragraf + satu artefak:

- Klaim generik: kelas masalah = "alokasi new-flow stateful pada 2 path heterogen dengan enforcement bucket integer + update cost".
- MikroTik = satu instansiasi `Π(r)`; ECMP/WRR = instansiasi lain yang formulanya ditulis (tanpa harus diimplementasikan).
- Artefak terbuka: skema DB + log `timestamp, metrik, skor, status, rasio` + skrip tc/netem staging agar site lain bisa replikasi.

## 4. Revisi rumusan masalah minimal (tetap ALQI, naik level)

> Bagaimana merancang ALQI berbasis SAW dengan histeresis asimetris dan dwell count yang menilai kualitas FO–Starlink dari delay, jitter, loss, throughput, lalu memproyeksikannya menjadi rasio PCC integer yang hanya mengatur koneksi baru, sehingga QoS agregat naik vs PCC statis dan flapping turun vs varian tanpa hysteresis, dengan bound switching terbatas dan sensitivitas bobot/ambang yang terukur?

## 5. Struktur bukti untuk H1/H2 (tidak berubah, hanya ditambah)

- H1: B vs C pada staging 15 detik (presisi) + lapangan (realisme); metrik flap/satuan waktu + bound P5 + visual garis waktu berdampingan.
- H2: A vs C pada degradasi nyata; metrik QoS agregat + ALQI + SPR (P1) + error kuantisasi (P2); stratifikasi hujan/cerah, sibuk/normal.
- Tambahan wajib: sensitivitas bobot (P3) + ablasi urutan filter (P4). Empat analisis ini yang mengubah tesis "sistem berjalan" menjadi "sistem terbukti".

## 6. Peta bab tesis hasil remodel

- Bab I: tetap, perkuat paragraf upper-bound Starlink + kalimat kontribusi 3-elemen yang sudah ada.
- Bab II: tambah P1 (aturan rewrite new-only + SPR), P2 (proyeksi integer + bound), P5 (bound switching) sebagai subbab perancangan; tambah P4 sebagai skenario staging ke-2.
- Bab III (hasil): H1, H2, sensitivitas, ablasi urutan, stratifikasi, ancaman validitas (satu site, K pelanggan, API failure rate dilaporkan).
