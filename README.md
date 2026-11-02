# Estimasi π dengan Monte Carlo: Pthreads vs OpenMP

Tugas Pemrograman Paralel · **Opsi C: Monte Carlo simulasi nilai π (N = 10⁹)**
Nama: Imam Naufal Mahrup · NIM: 2322105007

Satu masalah komputasi intensif diselesaikan dengan tiga implementasi yang memakai kernel perhitungan yang sama
(`src/common.h`): serial (baseline), POSIX Threads (Pthreads), dan OpenMP. Laporan tugas ada di
berkas PDF yang dikumpulkan terpisah.

## Isi repositori

| Berkas | Isi |
|---|---|
| `src/mc_serial.c`    | baseline serial |
| `src/mc_pthreads.c`  | POSIX Threads (manual; slot hasil ber-padding 64 B, tanpa mutex) |
| `src/mc_omp.c`       | OpenMP (`parallel for` + `reduction`) |
| `src/rng.h`          | RNG xoshiro256** + SplitMix64, tanpa state global |
| `src/common.h`       | kernel Monte Carlo, parsing opsi, pengukuran waktu |
| `src/accuracy.c`     | eksperimen akurasi: seed independen vs seed sama |
| `scripts/test.sh`    | uji correctness |
| `scripts/bench.sh`   | pengumpul data benchmark |
| `results/`           | data mentah benchmark (CSV) dan info mesin |
| `Makefile`           | otomatisasi build, test, dan benchmark |

## Cara menjalankan setelah clone (step by step)

Sistem yang didukung: **Linux atau macOS** (POSIX). Di Windows gunakan **WSL2 + Ubuntu**.

### Langkah 1. Clone repositori
```bash
git clone https://github.com/imam1205/paralel-2322105007-ImamNaufalMahrup.git
cd paralel-2322105007-ImamNaufalMahrup
```

### Langkah 2. Pasang prasyarat
Dibutuhkan `gcc` (dengan dukungan OpenMP), `make`, dan `python3` (hanya pustaka standar, tanpa `pip install`;
dipakai oleh `make test` untuk uji statistik):
```bash
# Ubuntu / Debian / WSL
sudo apt update
sudo apt install -y build-essential python3
```
Cek:
```bash
gcc --version
make --version
nproc            # jumlah thread logis yang tersedia
```
> macOS: gcc bawaan (Apple clang) tidak mendukung `-fopenmp`. Pasang `brew install gcc` lalu jalankan
> `make CC=gcc-14` (sesuaikan nomor versi).

### Langkah 3. Compile
```bash
make
```
Hasil: lima program di folder `bin/` (`mc_serial`, `mc_pthreads`, `mc_pthreads_fs`, `mc_omp`, `accuracy`).
Tidak boleh ada warning atau error. Untuk menghapus hasil build: `make clean`.

### Langkah 4. Verifikasi correctness
```bash
make test
```
Hasil yang diharapkan (baris terakhir):
```
== SEMUA TEST LULUS ==
```
Test membandingkan jumlah *hit* Pthreads dan OpenMP dengan versi serial (perbandingan **integer eksak**, bukan toleransi)
untuk T = 1, 2, 3, 4, 8 dan beberapa N (termasuk N yang tidak habis dibagi 256), lalu menguji bahwa estimasi π berada
dalam 5σ teoretis. Hasilnya identik pada semua jumlah thread karena **stream RNG dimiliki blok kerja
(256 blok tetap), bukan thread**.

### Langkah 5. Jalankan program (N = 10⁹, seed default 42)
```bash
./bin/mc_serial   -n 1000000000
./bin/mc_pthreads -n 1000000000 -t 4
./bin/mc_omp      -n 1000000000 -t 4
```
Contoh keluaran (waktu bergantung mesin, tetapi `hits` dan `pi` harus sama persis di semua versi):
```
[serial]   threads=1 N=1000000000 hits=785402336 pi=3.1416093440 |err|=1.669e-05 waktu=3.1241 s
[pthreads] threads=4 N=1000000000 hits=785402336 pi=3.1416093440 |err|=1.669e-05 waktu=0.7643 s
[openmp]   threads=4 N=1000000000 hits=785402336 pi=3.1416093440 |err|=1.669e-05 waktu=0.7551 s
```
Opsi program: `-n` jumlah sampel (default 10⁹), `-t` jumlah thread, `-s` seed (default 42),
`-c` keluaran CSV satu baris.

### Langkah 6. Benchmark (opsional)
```bash
make bench
```
Menjalankan serial, Pthreads, dan OpenMP pada T = 1, 2, 4, 8 dengan 5 ulangan, ditambah pengukuran overhead
thread, eksperimen false sharing, dan eksperimen akurasi RNG. Hasil ditulis ke `results/*.csv` dan
`results/env.txt` (**menimpa data yang sudah ada di repositori**). Data inilah yang menjadi dasar tabel dan grafik
pada laporan. Untuk hasil yang stabil: colokkan charger, tutup aplikasi berat, dan jangan memakai komputer
selama pengukuran. Uji cepat:
```bash
N=100000000 REPS=2 THREADS="1 2 4" bash scripts/bench.sh
```

## Ringkasan hasil (N = 10⁹, median dari 5 ulangan)

Mesin ukur: Intel Core i5-12450HX (4 P-core + 4 E-core, 12 thread logis), WSL2 Ubuntu, gcc 15.2, `-O2`.

| Thread | Pthreads (s) | OpenMP (s) | Speedup | Efficiency |
|---|---|---|---|---|
| serial | 3,287 | — | 1,00 | — |
| 2 | 1,519 | 1,516 | ≈ 2,17 | ≈ 1,08 |
| 4 | 0,757 | 0,757 | ≈ 4,34 | ≈ 1,09 |
| 8 | 0,552 | 0,551 | ≈ 5,96 | ≈ 0,75 |

Speedup dihitung terhadap median serial. Efficiency di atas 1 pada T = 2 dan 4 berasal dari baseline serial yang
lebih lambat sekitar 5,7% daripada versi paralel T = 1 (variasi frekuensi/suhu laptop), bukan efek superlinear
sejati. Efficiency turun pada T = 8 karena arsitektur hybrid dan hyper-threading. Analisis lengkap ada di
laporan PDF yang dikumpulkan terpisah.

## Eksperimen false sharing
`make` juga membangun `bin/mc_pthreads_fs` (`-DFALSE_SHARING`), yaitu versi Pthreads yang sengaja memakai slot rapat
8 byte dan menulis ke memori pada setiap hit. Hasil perbandingannya dengan versi ber-padding ada di
`results/fs.csv`.

## Pemecahan masalah

| Masalah | Solusi |
|---|---|
| `make: gcc: command not found` | pasang `build-essential` (Langkah 2) |
| Error `-fopenmp` di macOS | pakai gcc dari Homebrew (catatan Langkah 2) |
| Speedup tidak naik pada T besar | periksa `nproc`; jumlah thread melebihi core yang tersedia tidak mempercepat program |