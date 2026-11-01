# Rumusan Masalah Utama (Spesifik + Nilai Novelty)

## Rumusan masalah (1 kalimat utama)

> Bagaimana merancang mekanisme alokasi koneksi baru yang adaptif terhadap kualitas link heterogen FO–Starlink (`Q_FO(t)`, `Q_SL(t)`) dengan kendala bucket PCC integer (`n_FO + n_SL = D`), pinning sesi aktif (`SPR`), dan guard anti-oscillation konjungsif (`Δw`, `Tc`, `K_stable`), sehingga pada kondisi degradasi asimetris persentase koneksi baru yang salah jalan dan gangguan sesi menurun signifikan dibanding PCC statis, tanpa menimbulkan flapping konfigurasi pada kondisi fluktuasi bersamaan?

## Mengapa rumusan ini spesifik (bukan umum)

- Objek: **koneksi baru** saja (`C_new`), bukan semua paket. Koneksi aktif (`C_active`) dikecualikan kecuali `FAILED`.
- Input: skor kualitas multivariat kontinu `Q_i(t)` dari `L, J, P, Th, A` dengan smoothing EWMA.
- Kendala keras: output harus integer `n_i` dengan `Σn_i = D` (PCC), bukan bobot kontinu bebas.
- Constraint: `SPR` dipertahankan + frekuensi update dibatasi guard.
- Kondisi uji: degradasi asimetris (satu ISP jelek) dan fluktuasi bersamaan (keduanya naik-turun).
- Pembanding: PCC statis + varian ablasi (tanpa guard / linear / tanpa EWMA).

## Nilai novelty yang terkunci di rumusan ini

1. **FSM 3-state dengan aksi berbeda** (`NORMAL` bagi proporsional, `DEGRADED` tekan new-conn tapi tahan active, `FAILED` baru failover) — bukan failover biner.
2. **Alokasi non-linear** `W = max(0, Q − Q_min)^γ` — menekan ISP jelek lebih agresif daripada proporsional linear, dengan `γ` dan `Q_min` yang diuji sensitivitasnya.
3. **Guard konjungsif tiga syarat** (`Δw` + `Tc` + `K_stable`) — update PCC hanya jika ketiganya lolos, sehingga ada tradeoff terukur antara kecepatan deteksi dan stabilitas.
4. **Kuantisasi sadar-error** `n = round(w·D)` dengan `n_min` + laporan error `|n/D − w|` untuk tiap `D` — jembatan core generik ke PCC spesifik.
5. **SPR sebagai constraint optimasi**, bukan metrik tambahan — adaptif tidak boleh merusak sesi lama.

## Kriteria jawaban dinyatakan berhasil (falsifiable)

Rumusan dinyatakan terjawab jika dan hanya jika, pada ≥10 repetisi per skenario dengan CI95%:

1. Di S2–S6 (degradasi/putus satu sisi): `% new-conn salah jalan` dan `sesi terganggu` QASPA < PCC statis secara signifikan, `SPR > 0.95` (kecuali sisi FAILED).
2. Di S1 (normal): QoS agregat QASPA imbang dengan PCC (tidak lebih buruk signifikan) dan `flap ≈ 0`.
3. Di S7 (fluktuasi bersamaan): `flap/jam` dan `rewrite/jam` QASPA dengan guard jauh < QASPA tanpa guard, dengan tambahan detection delay yang terukur dan dilaporkan.
4. Ablasi menunjukkan tiap komponen (EWMA, `γ>1`, guard, pemisahan DEGRADED/FAILED) berkontribusi pada metrik berbeda — bukan tempelan.

Jika salah satu gagal, klaim novelty gugur sebagian dan harus dilaporkan jujur sebagai batasan.
