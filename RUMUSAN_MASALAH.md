# Rumusan Masalah dan Novelty — Versi Algoritmik Device-Agnostic (Standar Tesis S2 / Publikasi)

## 1. Rumusan masalah umum (inti tesis, tanpa menyebut merek)

> Bagaimana mengalokasikan flow baru pada N path heterogen dengan kualitas bursty dan berkorelasi waktu (`Q_i(t)`) ke dalam himpunan enforcement diskret berkapasitas terbatas, dengan kendala (i) flow lama tidak boleh dipindah kecuali path-nya gagal total, dan (ii) biaya perubahan konfigurasi tidak nol, sehingga utilitas kualitas agregat maksimal dengan jumlah perpindahan konfigurasi yang terbatas dan terjamin tidak berosilasi pada noise terbatas?

Instansiasi evaluasi (bukan inti kontribusi): `N=2` (FO stabil + Starlink LEO bursty), enforcement integer `n_1+n_2=D` (PCC MikroTik sebagai satu anggota kelas Weighted Bucket Enforcer seperti ECMP/WRR).

## 2. Formalisasi matematika (masalah yang dipecahkan algoritma)

Keadaan tiap path `i` pada siklus `t`:

```
Q_i(t) in [0,1]   : kualitas komposit hasil LQA ( Bagian 3 )
V_i(t) >= 0       : indeks ketidakstabilan (variance-based, Bagian N1)
s_i(t)            : NORMAL / DEGRADED / FAILED ( Bagian N2 )
```

Keputusan: bobot kontinu `w(t)` di simplex, lalu peta diskret `n(t)` milik enforcer.

```
w(t) = argmax  Σ_i w_i · U_i(t)  −  λ · C_sw(w, w_old)  −  μ · E_q(w, n)
s.t. Σw = 1, w >= 0, n = Π(w), Σn = D, n integer
```

dengan `U_i` utilitas adaptif, `C_sw` biaya switching, `E_q` error kuantisasi, `Π` operator proyeksi enforcer. Kontribusi ada pada bentuk `U`, `s`, dan syarat update — bukan pada solver umum.

## 3. Novelty matematika (modifikasi, bukan sekadar memakai)

### N1. Utilitas sadar-stabilitas (stability-aware utility) — REPARASI atas weighted-sum statis

Weighted-sum klasik `Q = Σ c_k·q_k` dengan bobot tetap gagal pada LEO: metrik yang paling bergejolak justru paling menentukan ranking sesaat.

Perbaikan:

```
V_i(t) = Var_W(Q_i) / (Mean_W(Q_i) + ε)     # koefisien variasi pada jendela W
S_i(t) = 1 / (1 + κ·V_i(t))                  # faktor stabilitas, κ>0
U_i(t) = S_i(t) · max(0, Q_i(t) − Q_min)^γ
w_i    = U_i / Σ_j U_j
```

Sifat yang diklaim dan diuji: (a) path bursty didiskon otomatis walau mean `Q` tinggi; (b) pada `V→0` kembali ke alokasi proporsional kualitas; (c) `κ, γ` diuji sensitivitas, bukan konstanta ajaib. Inilah reparasi matematis pertama.

### N2. FSM 3-state dengan threshold adaptif (bukan threshold tetap)

Threshold tetap `0.55/0.65` adalah tuning testbed. Perbaikan: threshold mengikuti statistik baseline bergerak.

```
m_i(t) = mean_W(Q_i),  σ_i(t) = std_W(Q_i)
Tau_low_i(t)  = max(Tau_abs_low,  m_i − a·σ_i)
Tau_high_i(t) = min(Tau_abs_high, m_i − b·σ_i + h),  h>0 hysteresis gap
aturan: NORMAL→DEGRADED jika Q < Tau_low K_deg kali; pulih hanya jika Q > Tau_high K_rec kali
FAILED hanya via A_i=0 K_fail kali (jalur keras terpisah)
```

Nilai kebaruan: ambang mengikuti rezim tiap path (siang/malam, cuaca), sehingga klaim tidak overfit satu angka. `a,b,h` dilaporkan + diablasi.

### N3. Guard konjungsif dengan jaminan terbatas (bounded switching)

Guard tunggal (cooldown saja) sudah umum. Perbaikan: tiga syarat konjungsif yang memberi bound keras.

```
update(t) =  [max|w_new − w_old| > δw] ∧ [t − t_last > Tc] ∧ [s stabil K_stable siklus]
```

Jaminan yang dibuktikan (bukan klaim): untuk noise `|Q(t)−Q(t−1)| ≤ η`, frekuensi update ≤ `3600/Tc` per jam apa pun yang terjadi, dan tidak ada update pada osilasi amplitudo `< δw/(L·η)` dengan `L` konstanta Lipschitz pemetaan `Q→w`. Kurva tradeoff `δw,Tc` vs detection delay vs flap diukur — inilah yang membuat C5 menjadi hasil, bukan parameter.

### N4. Proyeksi kuantisasi sadar-error dengan bound

`round(w·D)` sudah umum. Perbaikan: proyeksi dengan proteksi minimum + bound eksplisit.

```
n = Π(w): n_1 = round(w_1·D), n_2 = D − n_1, n_i ≥ n_min kecuali FAILED
bound: |n_i/D − w_i| ≤ 1/(2D) + 1[D ganjil dan w≈0.5]
```

Implikasi: pemilihan `D` bukan selera — tesis memberi tabel error vs granularitas vs jumlah rule `O(D)`, dan menunjukkan kapan `D=10` cukup dan kapan perlu `D=20`. Berlaku untuk kelas enforcer apa pun (`Π` diganti ECMP/WRR tanpa ubah N1–N3).

## 4. Mengapa ini bukan "algoritma dipakai begitu saja"

| Algoritma dasar yang dipakai | Reparasi pada penelitian ini | Bukti yang dituntut |
|---|---|---|
| EWMA + weighted sum | Bobot dikali faktor stabilitas `S_i(V_i)` + threshold adaptif `m±kσ` | Ablasi tanpa `S_i` vs dengan `S_i` pada S7; sensitivitas `κ` |
| Threshold failover | FSM 3-state aksi beda + hysteresis gap `h` | DEGRADED vs FAILED punya distribusi dan aksi optimal berbeda |
| WRR proporsional | Supresi non-linear `γ` + `Q_min` | `γ=1` vs `γ>1`: % new-flow salah jalan |
| Cooldown | Guard 3-syarat + bound switching + kurva tradeoff | Tanpa guard vs dengan guard: flap/jam vs delay |
| round() | Proyeksi `Π` + bound error + analisis `D` | Error kuantisasi vs distribusi aktual |

Setiap baris punya persamaan yang diubah + eksperimen yang menggugurkannya. Tanpa keduanya, novelty gugur.

## 5. Kriteria keberhasilan (falsifiable, untuk publikasi)

Pada ≥10 repetisi/skenario, CI95%, baselines {statis, threshold-failover, SAW/MCDM sederhana}:

1. S2–S6: utilitas agregat dan % flow salah jalan lebih baik signifikan; `SPR > 0.95` kecuali FAILED.
2. S1: tidak lebih buruk signifikan vs statis; flap ≈ 0.
3. S7: flap/jam dengan N3 jauh < tanpa N3 dengan tambahan delay yang terukur (bukan nol).
4. N1–N4 masing-masing lolos ablasi pada metriknya sendiri.
5. Overhead `O(1)` per paket di data plane (hanya enforcement), `O(N)` per siklus di control plane; waktu respons `< 2·T_probe + API`.

Kegagalan apa pun wajib dilaporkan sebagai batasan — itu justru yang membuat paper dipercaya reviewer.
