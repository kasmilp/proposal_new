# Glosarium Singkatan dan Istilah Penelitian

Satu file acuan singkatan untuk seluruh dokumen: proposal, `LAPORAN_PEMBAHASAN_QASPA.md`, `ALGORITMA_QASPA_FLOWCHART.md`, `ALUR_SISTEM_FO_STARLINK.md`, `TOPOLOGI_SISTEM.md`, `TAHAPAN_SKENARIO_PENGUJIAN.md`, `RUMUSAN_MASALAH.md`.

---

## A. Algoritma dan sistem inti

| Singkatan | Kepanjangan | Penjelasan |
|---|---|---|
| QASPA | Quality-Aware Session-Preserving Adaptive Allocator | Nama kerja algoritma usulan. Inti novelty: FSM 3-state + bobot non-linear + guard konjungsif + kuantisasi PCC + pinning sesi. |
| PCC | Per Connection Classifier | Metode MikroTik membagi koneksi berdasar hashing agar satu koneksi selalu lewat jalur sama. Bersifat statis tanpa input kualitas. |
| C1–C6 | Kontribusi 1–6 | C1 LQA, C2 deteksi degradasi, C3 alokasi, C4 session persistence, C5 anti-oscillation, C6 adapter PCC MikroTik. |
| P1–P7 | Blok proses 1–7 | P1 ukur, P2 nilai, P3 FSM, P4 bobot, P5 guard, P6 enforce, P7 jaga sesi. Lihat flowchart QASPA. |
| RQ1–RQ5 | Research Question 1–5 | Lima pertanyaan penelitian di proposal (karakteristik, deteksi, alokasi tanpa ganggu sesi, integrasi PCC, peningkatan vs PCC). |
| S1–S8 | Skenario uji 1–8 | S1 normal, S2 FO jelek, S3 Starlink jelek, S4 loss tinggi, S5 delay/jitter tinggi, S6 putus, S7 fluktuasi bersamaan, S8 alami panjang. |
| T0–T5 | Tahap uji 0–5 | T0 smoke, T1 karakterisasi, T2 fungsional, T3 komparatif inti, T4 ablasi, T5 sensitivitas + overhead. |
| A / B | Skenario pembanding | A = PCC statis (baseline). B = QASPA lengkap. B1–B4 = varian ablasi. |

## B. Jaringan dan perangkat

| Singkatan | Kepanjangan | Penjelasan |
|---|---|---|
| FO | Fiber Optic | ISP-1, terrestrial, latency rendah dan stabil. Masuk ether1 via ONT. |
| SL | Starlink | ISP-2, satelit LEO, latency lebih tinggi dan burst-fluktuatif (handover, cuaca, obstruksi). Masuk ether2. |
| LEO | Low Earth Orbit | Orbit satelit Starlink. Penyebab variansi dan micro-outage. |
| VSAT | Very Small Aperture Terminal | Istilah generik satelit di contoh topologi awal; dalam penelitian ini diganti Starlink LEO. |
| ONT | Optical Network Termination | Perangkat ujung FO sebelum ether1 MikroTik. |
| LAN | Local Area Network | Jaringan user di belakang MikroTik, contoh `192.168.1.0/24`. |
| CGNAT | Carrier-Grade NAT | NAT di sisi operator Starlink; alasan tidak ada IP publik langsung di ether2 pada mode standar. |
| NAT | Network Address Translation | Translasi alamat di MikroTik (masquerade per out-interface). |
| DHCP | Dynamic Host Configuration Protocol | Pembagian IP otomatis ke client via switch LAN. |
| SNMP | Simple Network Management Protocol | Protokol ambil metrik perangkat; salah satu sumber monitoring selain API. |
| API | Application Programming Interface | Antarmuka controller ke MikroTik (RouterOS API 8728/8729 TLS + SSH fallback) untuk read counter dan tulis mangle/route. |
| SDN | Software-Defined Networking | Paradigma pisah control/data plane; QASPA memakai idenya (controller eksternal + forwarding MikroTik) tanpa full SDN. |

## C. Kualitas layanan dan penilaian link

| Singkatan | Kepanjangan | Penjelasan |
|---|---|---|
| QoS | Quality of Service | Kualitas layanan: latency, jitter, loss, throughput, ketersediaan. |
| LQA | Link Quality Assessment | Modul C1: smoothing + normalisasi + skor komposit `Q`. |
| L / J / P / Th / A | Latency / Jitter / Packet loss / Throughput / Availability | Lima metrik mentah per ISP per siklus `T_probe`. `A` biner {0,1}. |
| Q / Q_i | Link Quality Score | Skor komposit [0,1] per ISP; makin besar makin baik. `Q_i = wL·qL + wJ·qJ + wP·qP + wT·qT`. |
| qL,qJ,qP,qT | Skor ternormalisasi | Komponen [0,1] hasil normalisasi tiap metrik sebelum dibobot. |
| wL,wJ,wP,wT | Bobot metrik | Bobot skor komposit, jumlah = 1. Dipilih via baseline + sensitivity, bukan asal. |
| L_max,J_max,Th_ref | Parameter normalisasi | Batas kalibrasi dari baseline (mis. P95 RTT, P95 jitter, median throughput sehat). |
| EWMA / α | Exponentially Weighted Moving Average / alpha | Perataan `S(t) = α·X(t) + (1−α)·S(t−1)`. `α` 0.2–0.4; kecil = halus tapi lambat. |
| MCDM | Multi-Criteria Decision Making | Keluarga metode keputusan multikriteria (SAW, TOPSIS, AHP). Pembanding konseptual, bukan inti QASPA. |
| SAW | Simple Additive Weighting | Varian MCDM penjumlahan terbobot; dipakai di contoh topologi awal, diganti LQA QASPA. |
| ALQI | Adaptive Link Quality Indicator | Istilah skor kualitas di contoh awal; dalam QASPA diganti `Q` komposit + `s, w, n`. |
| T_probe | Periode probe | Interval siklus kontrol, contoh 5–10 detik. |

## D. Status, bobot, dan parameter keputusan

| Singkatan | Kepanjangan | Penjelasan |
|---|---|---|
| FSM | Finite State Machine | Mesin keadaan hingga untuk status ISP di P3. |
| NORMAL / DEGRADED / FAILED | Status ISP | NORMAL layak penuh; DEGRADED kurangi new-conn + tahan active; FAILED failover + boleh pindah active. |
| Tau_low / Tau_high | Threshold hysteresis | Ambang bawah/atas `Q` (contoh 0.55/0.65). `Tau_high > Tau_low` agar tidak flap di batas. |
| K_fail,K_deg,K_rec,K_stable | Jumlah siklus stabil | Syarat berturut-turut untuk nyatakan gagal, degradasi, pulih, dan stabil sebelum update. |
| Delta_drop | Penurunan tiba-tiba | Selisih jatuh `Q` antar siklus yang langsung memicu DEGRADED meski masih di atas `Tau_low`. |
| Q_min / γ (gamma) | Batas minimum / eksponen supresi | `W = max(0, Q − Q_min)^γ`. `γ>1` menekan ISP jelek non-linear. Contoh `Q_min=0.30, γ=1.8`. |
| W_i / w_i | Bobot mentah / bobot normalisasi | `W` per ISP lalu `w_i = W_i/ΣW`, jumlah = 1. Hanya untuk new-conn. |
| D / n_i / n_min | Denominator / bucket / minimal bucket | `D` jumlah bucket PCC (contoh 10). `n_1 = round(w_1·D)`, `Σn = D`. `n_min` (mis. 1) agar probe tetap jalan kecuali FAILED. |
| δw (delta_w) / Tc | Ambang perubahan / cooldown | Guard P5: update hanya jika `max|w_new−w_old| > δw` (mis. 0.10) dan `t−t_last > Tc` (mis. 45 dtk). |
| s_i | State ISP-i | Nilai FSM per ISP: NORMAL/DEGRADED/FAILED. |
| SPR | Session Persistence Ratio | `sesi wajib-tetap yang tetap / total wajib-tetap`. Target >0.95 kecuali FAILED. Sesi di ISP FAILED yang pindah tidak dihitung pelanggaran. |
| CI | Confidence Interval | Rentang kepercayaan statistik (dilaporkan CI95% dari ≥10 repetisi, bukan single-run). |

## E. Data plane vs control plane di MikroTik

| Istilah | Penjelasan |
|---|---|
| Data plane | MikroTik meneruskan tiap paket (mangle, routing, NAT). Real-time per-paket. |
| Control plane | Controller QASPA tiap `T_probe`: ukur → nilai → FSM → bobot → guard → enforce via API. |
| connection-mark / routing-mark | Penanda koneksi dan penanda rute di mangle. `via-FO/to-FO` dan `via-SL/to-SL`. |
| connection-state new / established,related | `new` = koneksi baru (lewat PCC). `established,related` = koneksi lama (langsung pakai mark lama, tidak lewat PCC). |
| Tabel to-FO / to-SL | Tabel routing policy per ISP dengan gateway dan `check-gateway` masing-masing. |
| Fail-safe | Jika controller mati, MikroTik tetap jalan dengan konfigurasi PCC terakhir + `check-gateway`/distance. |
| tc-netem | Traffic control network emulator di Linux bridge untuk injeksi delay/jitter/loss terkontrol saat uji (bukan Starlink alami). |
| Read-back | Verifikasi baca-ulang via API setelah tulis mangle/route sebelum menandai `t_last_update`. |
