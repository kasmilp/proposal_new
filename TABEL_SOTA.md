# Tabel SOTA — Acuan Penelitian QASPA

Disusun dari daftar bacaan yang diberikan peneliti. Isi tabel dirumuskan ulang agar konsisten (objek, metode, hasil, celah) tanpa menambah klaim di luar ringkasan yang diberikan. Verifikasi DOI/venues wajib dilakukan ulang sebelum submit (cek Crossref/IEEE Xplore).

| No | Penulis, Tahun, DOI | Objek / Masalah | Metode | Hasil | Celah / Keterbatasan |
|---|---|---|---|---|---|
| 1 | Ali, Abouelatta & Youssef, 2025. 10.1109/ACCESS.2025.3529486 | Agregasi banyak ISP heterogen (FO, 4G/5G, LEO, GEO) untuk konektivitas IoT yang andal dan hemat biaya. | Dynamic Connectivity Hub berbasis ANN yang mengalokasikan data rate ke ISP terbaik real-time dari latensi, data rate, kongesti, biaya; testbed Linux. | Distribusi trafik adaptif lintas ISP terestrial–satelit; efisiensi alokasi naik ~17% vs baseline tanpa load balancing. | Butuh pelatihan ANN dan komputasi besar; tidak untuk SOHO/MikroTik; tidak membahas konsistensi per-koneksi dan peredaman osilasi keputusan. |
| 2 | Amalia dkk., 2022. 10.1016/j.procs.2022.12.116 | Koneksi putus dan beban timpang pada jaringan instansi dua ISP. | PCC + failover RouterOS MikroTik; perancangan NDLC; evaluasi QoS TIPHON. | Skor QoS 3,75 (sibuk) dan 4,00 (normal); kepuasan pengguna 0,4 → 0,89. | Rasio PCC statis; tanpa umpan balik link real-time; dua jalur terestrial homogen. |
| 3 | Arfianita dkk., 2024. 10.31763/iota.v4i4.810 | Responsivitas jaringan kampus/kantor tiga ISP. | PCC tiga jalur; simulasi GNS3; ukur throughput, delay, jitter, loss. | Throughput +9,32 Mbps; delay −43,54 ms; jitter −74,17 ms; loss stabil 0%. | Lab stabil; tanpa skenario degradasi satu ISP; rasio tetap tanpa penyesuaian otomatis. |
| 4 | Mirsal & Agtriadi, 2026. 10.37339/e-komtek.v10i1.3170 | Perbandingan PCC vs QoS Queue Tree untuk optimasi bandwidth. | Studi komparatif tiga kondisi beban; metrik throughput, latensi, loss, stabilitas. | PCC unggul agregasi kapasitas dan stabilitas koneksi; Queue Tree unggul penjaminan per pengguna. | Keduanya statis; tanpa mekanisme adaptif gabungan. |
| 5 | Ma dkk., 2023. 10.1109/INFOCOM53939.2023.10228912 | Karakteristik konstelasi LEO (Starlink) dari sisi end-user. | Pengukuran end-to-end throughput, latensi, outage, daya dari terminal pengguna. | Dinamika throughput/latensi LEO jauh lebih besar vs terestrial + outage sering; dipengaruhi medan, cuaca, badai matahari, suhu. | Deskriptif; tanpa mitigasi/alokasi trafik berbasis hasil ukur. |
| 6 | Lanfer dkk., 2024 (Boeckenholt dkk.). — | Ketidakpastian throughput Starlink akibat cuaca. | Perbandingan model ML (termasuk Random Forest) dari parameter cuaca; feature importance. | Random Forest terbaik (R² 0,47 downlink; 0,13 uplink); hujan prediktor terpenting. | Uplink lemah; iklim sedang (belum tropis); prediksi belum masuk kontrol routing. |
| 7 | Garcia, Sundberg & Brunstrom, 2025. 10.1145/3748749.3749090 | Karakterisasi one-way delay Starlink dan sumber spike. | Analisis >500 juta paket probe 10 hari; pisah uplink/downlink. | Uplink paling terdampak siklus rekonfigurasi periodik 15 detik Starlink. | Pengukuran/pemodelan saja; tanpa filter/penjadwalan peredam transien. |
| 8 | Hammer dkk., 2024. 10.1145/3697253.3697273 | Perilaku Starlink dari sisi edge router (bukan host tunggal). | Pengukuran pasif edge router + analisis periodisitas dan reordering. | Pola periodik penurunan throughput pada batas 15 detik terkait reordering/loss. | Belum dikaitkan ke kebijakan multi-WAN; tanpa analisis dampak ke keputusan real-time. |
| 9 | Wang & Kuo, 2013. 10.1109/SURV.2012.010912.00044 | Pemodelan matematis pemilihan jaringan heterogen nirkabel. | Tinjauan: utility theory, MADM (SAW, TOPSIS), fuzzy, game theory, optimasi kombinatorial. | Peta kesesuaian teori vs tujuan/kompleksitas; MADM paling cocok untuk seleksi. | Tutorial domain seluler; belum diterjemahkan ke rasio load balancing router kabel/satelit. |
| 10 | Vafaei, Ribeiro & Camarinha-Matos, 2022. 10.1016/j.procs.2022.01.156 | Efek teknik normalisasi pada peringkat SAW. | Komparasi normalisasi (linear max, max-min, vector, sum) dalam SAW. | Normalisasi memengaruhi peringkat signifikan; perlu teknik sesuai tipe cost/benefit. | Dataset MCDM generik statis; belum diuji pada aliran metrik real-time ber-noise/outlier. |
| 11 | Tournaire dkk., 2023. 10.1145/3603532 | Ambang optimal auto-scaling dengan tradeoff SLA vs resource. | Teori antrean + local search (rantai Markov); MDP multi-threshold histeresis. | Komputasi ambang maju/mundur efisien; ambang tunggal sebabkan osilasi, histeresis menghindarinya. | Domain cloud; ambang simetris okupansi antrean; bukan skor kualitas link multikriteria. |
| 12 | Lilhore dkk., 2025. 10.1038/s41598-025-96364-1 | Load balancing multi-objektif di cloud dengan kriteria konflik. | Integrasi ACO + Water Wave Optimization untuk penjadwalan. | Perbaikan makespan, utilisasi, dan balance vs pembanding. | Metaheuristik berat untuk kontrol detik di perangkat jaringan; tanpa session consistency dan stabilitas antarsiklus. |

## Sintesis gap (defensible, bukan klaim absolut)

1. **PCC statis vs link dinamis:** Baris 2–4 menunjukkan PCC efektif untuk agregasi/stabilitas, tetapi rasio ditetapkan di konfigurasi dan diuji pada jalur homogen atau lab stabil. Integrasi umpan balik kualitas real-time + degradasi asimetris LEO masih terbatas.
2. **Karakteristik LEO tersedia, kebijakan mitigasi belum:** Baris 5–8 memberi fondasi empiris (variansi besar, periodisitas 15 detik, sensitivitas cuaca) tetapi berhenti pada pengukuran/prediksi; belum diterjemahkan menjadi kebijakan alokasi new-flow dengan pinning sesi dan guard anti-osilasi.
3. **Keputusan multikriteria belum untuk aliran real-time ber-noise:** Baris 9–10 menempatkan MADM/SAW sebagai kandidat seleksi, tetapi normalisasi diuji statis. Pembobotan yang sadar-stabilitas (N1) + threshold adaptif (N2) untuk metrik jaringan real-time masih terbuka.
4. **Histeresis terbukti di domain lain, belum untuk skor link:** Baris 11 membuktikan ambang tunggal sebabkan osilasi dan histeresis mengatasinya di cloud. Penerapan guard konjungsif + bound switching (N3) pada skor kualitas link multikriteria + enforcement bucket integer (N4) adalah ruang kontribusi.
5. **Komputasi ringan vs berat:** Baris 1 dan 12 menunjukkan pendekatan ANN/metaheuristik memberi gain tetapi berat untuk SOHO/siklus detik. Mekanisme `O(1)` per paket + `O(N)` per siklus yang tetap adaptif adalah diferensiasi QASPA.

## Pemetaan ke novelty QASPA (N1–N4 di RUMUSAN_MASALAH.md)

- Baris 5–8 → justifikasi **N1** (faktor stabilitas `S(V)`) dan **N2** (threshold adaptif `m±kσ`): LEO bursty + periodik tidak bisa ditangani threshold tetap.
- Baris 9–10 → justifikasi **N1/N2**: SAW statis diperbaiki menjadi utilitas sadar-stabilitas untuk aliran ber-noise.
- Baris 11 → justifikasi **N3**: histeresis cloud diadopsi menjadi guard 3-syarat + bound untuk skor link.
- Baris 2–4 → justifikasi **N4 + pinning sesi**: rasio PCC statis diganti proyeksi `Π(w)` + `SPR` sebagai constraint; evaluasi vs baseline statis + threshold-failover + SAW sederhana.
- Baris 1, 12 → diferensiasi kompleksitas: QASPA ringan vs ANN/metaheuristik berat.

## Catatan verifikasi sebelum publikasi

- Samakan tahun Amalia (2022 pada DOI/Procedia, bukan 2023) dan nama lengkap Lanfer/Boeckenholt sesuai prosiding.
- DOI baris 6–8 dilengkapi dari IEEE Xplore/ACM DL (jangan pakai URL ieeexplore polos).
- Referensi tambahan di daftar Anda (Chen 2025, Ji 2026, Mohan 2024, Ullah 2025, dsb.) belum masuk tabel; masukkan hanya jika benar dibaca dan dipakai untuk argumen spesifik, bukan sekadar daftar.
