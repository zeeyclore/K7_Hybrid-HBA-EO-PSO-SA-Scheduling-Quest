# Laporan Scheduling Quest: Penjadwalan Task Cloud dengan Algoritma Hybrid
## Komparasi HBA-EO dan Hybrid PSO-SA terhadap Min-Min dan PSO

**Tugas Minggu 13 · Demo 1** · Aplikasi: `scheduling-quest.html`

---

## Anggota Kelompok 7

| NRP | Nama |
| :--- | :--- |
| **5027241021** | Mochkamad Maulana Syafaat |
| **5027241052** | Salsa Bil Ulla |
| **5027241049** | Khumaidi Kharis Az-zacky |
| **5027241016** | Shinta Alya Ramadani |
| **5027241041** | Raya Ahmad Syarif |

---

## Pendahuluan

Penjadwalan task di cloud (*cloud task scheduling*) adalah masalah membagi sejumlah task ke sejumlah mesin virtual (VM) agar pekerjaan selesai cepat, murah, dan hemat energi. Karena banyaknya kemungkinan pembagian tumbuh secara eksponensial, masalah ini termasuk **NP-hard** dan tidak bisa diselesaikan dengan mencoba semua kombinasi.

Pada tugas ini kami memilih **dua algoritma hybrid** dari jurnal ilmiah, mengimplementasikannya di simulator, lalu mengujinya terhadap dua metode pembanding dari tugas sebelumnya (**Min-Min** dan **PSO**):

1. **HBA-EO**: Honey Badger Algorithm + *Elite framework* (Zhang dkk., 2025).
2. **Hybrid PSO-SA**: Particle Swarm Optimization + Simulated Annealing (Khaledian dkk., 2024).

Seluruh penjelasan, simulator, dan hasil uji kami kemas dalam satu berkas web interaktif bergaya petualangan, **Scheduling Quest**, tempat maskot Madu si musang madu menjelajahi enam pulau langit (level). Aplikasi ini dilengkapi musik latar dan narasi suara Madu di setiap halaman.

> **Catatan istilah.** Pada laporan ini, **EO** di dalam nama *HBA-EO* berarti ***Elite framework*** (kerangka elite hasil hybrid di paper MOHBA-I/II), **bukan** *Equilibrium Optimizer* seperti pada laporan Min-Min / HBA / EO sebelumnya.

---

## Pemodelan Masalah

### 1. Representasi

Ada $n$ task dengan panjang $L_i$ (satuan MI) dan $m$ VM dengan kecepatan $\text{MIPS}_j$ dan tarif sewa $c_j$ (\$/detik). Setiap task hanya berjalan di satu VM, sehingga jumlah jadwal yang mungkin adalah $m^n$.

| Contoh | Jumlah jadwal | Perkiraan waktu brute force* |
| :--- | :---: | :--- |
| 6 task, 3 VM (mini game) | $3^6 = 729$ | seketika, rekor optimal bisa dihitung |
| 100 task, 10 VM (default arena) | $10^{100}$ | $\approx 3{,}2 \times 10^{83}$ tahun, atau $\approx 2{,}3 \times 10^{73}$ kali umur alam semesta |

\*Asumsi 1 miliar jadwal diperiksa per detik; umur alam semesta $\approx 1{,}38 \times 10^{10}$ tahun.

Cara tiap algoritma menerjemahkan solusinya menjadi jadwal:

| Algoritma | Bentuk solusi | Decode ke VM task $i$ |
| :--- | :--- | :--- |
| **HBA-EO** | vektor kontinu $x_i \in [0, m)$ per task | $\lfloor x_i \rfloor$ |
| **PSO / PSO-SA** | matriks $n \times m$ bernilai acak di $[-10, 10]$ | indeks dengan nilai **terbesar** pada baris task tersebut |

### 2. Model dan Metrik

$$ET_{ij} = \frac{L_i}{\text{MIPS}_j} \qquad CT_j = \sum_{i \in \text{VM}_j} ET_{ij} \qquad \text{Makespan} = \max_j CT_j$$

$$\text{Biaya} = \sum_j CT_j \cdot c_j$$

$$\text{Energi} = \sum_j \left[ CT_j \cdot P_{\text{aktif},j} + (\text{MS} - CT_j) \cdot P_{\text{idle},j} \right]$$

$$\text{DI} = \frac{CT_{\max} - CT_{\min}}{CT_{\text{rata-rata}}} \quad \text{(derajat ketidakseimbangan beban)}$$

### 3. Fungsi Fitness

Semua metode dinilai dengan fitness yang sama, dinormalisasi terhadap **Round Robin**. Semakin kecil semakin baik.

$$\text{Fitness} = w_1 \cdot \frac{\text{MS}}{\text{MS}_{RR}} + w_2 \cdot \frac{\text{Biaya}}{\text{Biaya}_{RR}} + w_3 \cdot \frac{E}{E_{RR}}, \qquad (w_1, w_2, w_3) = (0{,}5;\ 0{,}25;\ 0{,}25)$$

Round Robin selalu bernilai **1,000**. Fitness di bawah 1,000 berarti jadwal lebih baik daripada Round Robin pada gabungan ketiga kriteria.

---

## Algoritma yang Diterapkan

### Hybrid #1: HBA-EO (Honey Badger Algorithm + Elite framework)

Setiap musang adalah satu kandidat jadwal, dan **mangsa** adalah solusi terbaik. Di setiap iterasi, tiap musang memilih salah satu dari dua fase dengan peluang 50:50:

- **Digging** (menggali, eksplorasi): lompatan besar memakai intensitas bau $I$ dan pola kardioid.
- **Honey** (mengikuti burung pemandu madu, eksploitasi): bergerak mendekati mangsa.

Faktor densitas $\alpha = C \cdot e^{-t/t_{\max}}$ dengan $C = 2$ meluruh seiring iterasi, sehingga lompatan makin kecil.

$$x_{\text{new}}^{\text{digging}} = x_{\text{prey}} + F \beta I x_{\text{prey}} + F r_3 \alpha d_i \left| \cos(2\pi r_4)\left[1 - \cos(2\pi r_5)\right] \right|$$

$$x_{\text{new}}^{\text{honey}} = x_{\text{prey}} + F r_7 \alpha d_i, \qquad \beta = 6,\ F = \pm 1$$

Bagian *Elite framework* mengikuti paper (MOHBA-I/II), dan simulator kami memakai varian **MOHBA-I**:

1. Populasi lama dan baru digabung (2N solusi), lalu **elite non-dominated sorting** (front Pareto, seri dipecah dengan *crowding distance*) mengambil N terbaik.
2. **Arsip eksternal** menyimpan semua solusi non-dominated pada tiga objektif (makespan, biaya, energi).
3. **Grid indexing**: mangsa dipilih dari sel grid yang paling sepi (roulette).
4. **Circular segmentation screening**: saat arsip penuh, setiap solusi arsip menjadi pusat lingkaran berjari-jari $R$; solusi di dalam lingkaran diurutkan berdasarkan hasil kali nilai objektif dan hanya $S$ terbaik yang disimpan. Akibatnya front Pareto tetap tersebar merata.

Karena HBA-EO bersifat multi-objektif, hasil akhirnya berupa arsip Pareto, lalu kami ambil anggota arsip dengan fitness terbaik untuk dibandingkan.

### Hybrid #2: PSO-SA (Particle Swarm Optimization + Simulated Annealing)

**PSO** menjelajah secara global: tiap partikel (satu jadwal) bergerak dipengaruhi inersia, ingatan pribadi (*pbest*), dan kabar kawanan (*gbest*).

$$v \leftarrow w \cdot v + c_1 r_1 (pbest - x) + c_2 r_2 (gbest - x), \qquad x \leftarrow x + v$$

Kelemahan PSO adalah **konvergensi dini** dan mudah terjebak di optimum lokal. **SA** menutup kelemahan itu dengan memoles *gbest*: solusi yang lebih buruk masih boleh diterima dengan peluang

$$P = \begin{cases} 1, & \Delta E < 0 \\ e^{-\Delta E / T}, & \Delta E \ge 0 \end{cases} \qquad T \leftarrow 0{,}99 \cdot T,\ T_0 = 0{,}025$$

Alur hybrid: jalankan PSO, dan **setiap kali *gbest* membaik**, jalankan SA selama 15 langkah dari *gbest* tersebut, lalu suntikkan hasilnya kembali ke kawanan. Tetangga SA yang kami pakai: 50% memindahkan task dari VM tersibuk, 25% memindahkan task acak, 25% menukar dua task.

### Metode Pembanding

| Metode | Sifat | Cara kerja singkat |
| :--- | :--- | :--- |
| **Min-Min** | heuristik greedy, deterministik | Pada tiap langkah pilih task dengan waktu selesai tercepat secara global, tempatkan di VM yang membuatnya selesai paling awal. |
| **PSO** | metaheuristik populasi | PSO murni tanpa SA (inersia $w$ turun linear $0{,}9 \to 0{,}4$, $c_1 = c_2 = 2$). |
| **Round Robin** | heuristik sederhana | Task dibagi bergiliran antar VM; dipakai sebagai acuan normalisasi fitness. |

---

## Spesifikasi Pengujian

| Parameter | Nilai |
| :--- | :--- |
| Dataset | Dataset contoh bawaan simulator (`sampleData`): panjang task acak 1.000 sampai 20.000 MI; VM acak dari kelas 500 sampai 3.000 MIPS dengan tarif mengikuti $0{,}002 \cdot (\text{MIPS}/1000)^{1{,}7}$ |
| Ukuran default | 100 task, 10 VM, seed 42 |
| Bobot fitness | 0,5 : 0,25 : 0,25 (makespan : biaya : energi) |
| Populasi / iterasi | 40 / 150 |
| HBA-EO | $C = 2$, $\beta = 6$, ukuran arsip 60, radius $R = 0{,}05$, selection factor $S = 2$, crossover rate 0,1 |
| PSO-SA | $T_0 = 0{,}025$, laju pendinginan 0,99, SA 15 langkah per perbaikan *gbest* |
| Pembanding | Min-Min dan PSO |

> **Penting.** Slot "Dataset minggu 3" di Arena berisi masukan CSV khusus, sedangkan hasil di laporan ini memakai dataset contoh bawaan. Setelah dataset minggu 3 dimasukkan, angka pada bagian hasil perlu dijalankan ulang dan diperbarui.

---

## Hasil Eksperimen

### 1. Satu Kali Uji (konfigurasi default, seed 42)

Ini adalah angka yang tampil ketika Boss Battle dibuka dengan pengaturan bawaan.

| Metode | Makespan (s) | Biaya (\$) | Energi (kJ) | DI | Fitness |
| :--- | :---: | :---: | :---: | :---: | :---: |
| Round Robin (acuan) | 138,7 | 2,785 | 155,0 | 1,338 | 1,0000 |
| Min-Min | 67,4 | 3,391 | 93,2 | 0,382 | 0,6978 |
| PSO | 66,1 | 3,310 | 93,0 | 0,247 | 0,6853 |
| **Hybrid PSO-SA** | **63,9** | 3,301 | **91,5** | **0,157** | **0,6740** |
| HBA-EO | 75,6 | **3,228** | 101,1 | 0,603 | 0,7255 |

### 2. Rata-rata 30 Seed (100 task, 10 VM, bobot default)

Untuk memastikan hasil satu seed tidak kebetulan, kami menjalankan mesin simulasi yang sama (kode di dalam berkas HTML) pada seed 1 sampai 30, masing-masing dengan dataset berbeda.

| Metode | Fitness (rata-rata ± simpangan) | Makespan (s) | Biaya (\$) | Energi (kJ) | DI |
| :--- | :---: | :---: | :---: | :---: | :---: |
| Round Robin | 1,0000 | 209,5 | 2,757 | 211,7 | 1,882 |
| Min-Min | 0,6083 ± 0,0790 | 74,9 | 3,330 | 100,3 | 0,389 |
| PSO | 0,6079 ± 0,0804 | 77,8 | 3,206 | 103,8 | 0,375 |
| **Hybrid PSO-SA** | **0,6004 ± 0,0754** | 75,7 | 3,209 | 102,3 | **0,322** |
| HBA-EO | 0,6413 ± 0,0856 | 90,1 | **3,104** | 114,3 | 0,671 |

Hitungan kemenangan dari 30 dataset:

- PSO-SA mengalahkan HBA-EO pada fitness di **30 dari 30** dataset.
- PSO-SA mengalahkan Min-Min di **22 dari 30** dataset dan PSO di **23 dari 30**.
- HBA-EO mengalahkan Min-Min hanya di **2 dari 30** dataset pada pengaturan ini.

### 3. Pengaruh Pengaturan

Kami juga menjalankan variasi untuk melihat sensitivitas hasil (30 seed, kecuali dinyatakan lain).

| Skenario | Min-Min | PSO | PSO-SA | HBA-EO | Catatan |
| :--- | :---: | :---: | :---: | :---: | :--- |
| Default (bobot 0,5 : 0,25 : 0,25) | 0,6083 | 0,6079 | **0,6004** | 0,6413 | PSO-SA terbaik |
| Seed 1 individu dengan Min-Min | 0,6083 | 0,5960 | **0,5901** | 0,6015 | HBA-EO mengalahkan Min-Min di 28/30, PSO-SA di 30/30 |
| Utamakan biaya (0,2 : 0,6 : 0,2) | 0,8969 | **0,8436** | 0,8455 | 0,8489 | Ketiganya lebih baik daripada Min-Min; PSO-SA menang atas HBA-EO di 19/30 |
| 200 task, 10 VM (20 seed) | **0,5911** | 0,6085 | 0,6022 | 0,6656 | Min-Min sulit dikalahkan pada beban besar dengan iterasi tetap |

---

## Pembahasan

1. **Hybrid PSO-SA menjadi juara pada pengaturan default.** SA menyerang VM yang paling sibuk, sehingga makespan dan ketimpangan beban (DI) turun lebih jauh daripada PSO murni. Pada seed 42, PSO-SA mengungguli Min-Min dan PSO pada makespan, energi, dan DI. Rata-rata 30 seed menunjukkan keunggulannya konsisten tetapi tipis (fitness 0,6004 vs 0,6083 untuk Min-Min).

2. **HBA-EO unggul di biaya dan keragaman solusi, bukan di fitness gabungan.** Pada pengaturan default, HBA-EO menghasilkan biaya paling rendah (rata-rata \$3,104), begitu juga pada skenario utamakan biaya dan 200 task, karena ia menyimpan arsip Pareto (rata-rata 59 solusi) dan bisa memilih trade-off yang condong ke biaya. Namun anggota arsip dengan fitness terbaik masih memiliki makespan lebih panjang (90,1 s) dan energi lebih tinggi daripada hasil PSO-SA. Jika pembuat keputusan lebih mementingkan biaya, arsip Pareto HBA-EO justru nilai tambahnya.

3. **Min-Min adalah baseline yang kuat.** Pada makespan, Min-Min sudah sangat baik (greedy yang efektif). Kelemahannya ada pada biaya karena cenderung menumpuk task di VM cepat yang tarifnya mahal. Ketiga metaheuristik jelas melampauinya ketika bobot biaya dinaikkan (fitness 0,84 vs 0,90).

4. **Min-Min sebagai bibit awal membantu semua metaheuristik.** Dengan satu individu awal berupa solusi Min-Min, HBA-EO naik dari mengalahkan Min-Min di 2/30 dataset menjadi 28/30, dan PSO-SA mengalahkan Min-Min di 30/30. Ini menunjukkan populasi acak murni dengan 150 iterasi belum cukup untuk menemukan solusi sebagus heuristik klasik, dan menggabungkan keduanya (heuristik sebagai titik awal) adalah strategi praktis.

5. **Harga komputasi.** Heuristik sederhana selesai di bawah 1 ms, sedangkan PSO-SA dan HBA-EO butuh kira-kira 100 sampai 350 ms pada ukuran di atas. Kualitas jadwal bisa lebih baik, tetapi tidak gratis.

---

## Level pada Aplikasi Web

| Level | Pulau | Isi |
| :---: | :--- | :--- |
| I | **Pulau Masalah** | Mengapa penjadwalan sulit: kalkulator ledakan kombinasi ($m^n$ vs umur alam semesta), mini game 6 task ke 3 VM (dengan tombol Min-Min, Round Robin, Acak), rumus model dan metrik, dua hybrid pilihan, dan checklist ketentuan tugas. |
| II | **Sarang Madu** | HBA-EO: animasi fase digging/honey di lanskap fitness, kerangka elite (non-dominated sorting, grid, arsip), animasi circular segmentation screening, alur algoritma, dan detail paper asli. |
| III | **Hutan Partikel** | PSO (slider $w$, $c_1$, $c_2$ dan animasi partikel), SA (dadu peluang penerimaan dan kurva pendinginan), alur Hybrid PSO-SA, dan detail paper asli. |
| IV | **Arena Simulasi** | Balapan 4 metode pada dataset contoh atau CSV sendiri (task dan VM), pengaturan populasi, iterasi, bobot, dan seed; grafik konvergensi fitness, grafik makespan vs biaya dengan arsip Pareto, dan beban tiap VM. |
| V | **Boss Battle** | Hybrid terunggul vs 2 metode pembanding: batang perbandingan metrik, persentase perbaikan, kesimpulan otomatis, dan penjelasan cara kerja sang juara. |
| VI | **Menara Referensi** | Kesimpulan yang terisi otomatis dari hasil Arena, rencana Demo 2 (minggu 14), dan daftar referensi. |

Pada Arena, kolom **Metode A** dan **Metode B** dapat diganti dengan Round Robin, FCFS, SJF, MCT, Min-Min, Max-Min, PSO, GA, atau SA (default: Min-Min dan PSO).

---

## Musik dan Narasi Suara

Setiap halaman level dinarasikan oleh Madu, ditemani musik latar.

| Fitur | Keterangan |
| :--- | :--- |
| **Narasi Madu** | 19 klip suara (pembuka, peta, tiap halaman level, selesai balapan, dan naik level) dibuat dengan ElevenLabs v3 dan **sudah tertanam** di berkas HTML, jadi tidak perlu mengunduh apa pun. Mulut maskot ikut bergerak saat bicara. |
| **Suara cadangan** | Jika sebuah baris tidak punya klip, narasi dibacakan oleh suara bahasa Indonesia bawaan peramban. |
| **Musik latar** | Waltz bergaya fantasi yang disintesis langsung dengan Web Audio API (piano, flute, pad, lonceng). Ada tiga suasana: peta, tenang (Level I, II, III, dan VI), dan *battle* (Level IV dan V). Volume musik otomatis turun saat narator bicara. |
| **Studio Suara** | Panel untuk mengganti klip narasi per baris, menyeret banyak file MP3 sekaligus (dicocokkan lewat nama file `01_title.mp3` dan seterusnya, atau berdasarkan urutan), memakai musik sendiri (`music.mp3`), mengatur volume, menyalin seluruh naskah, dan menghapus file tambahan. |
| **Penyimpanan** | File yang ditambahkan lewat Studio Suara tersimpan di peramban (IndexedDB) pada perangkat tersebut saja. Jika presentasi memakai laptop lain, file perlu diseret ulang. |

Audio baru mulai berbunyi setelah tombol **Mulai petualangan** diklik (aturan autoplay peramban).

---

## Panduan Menjalankan

Aplikasi bersifat satu berkas (sekitar 6,3 MB karena narasi suara tertanam) dan tidak membutuhkan server atau instalasi. Buka `scheduling-quest.html` di peramban modern. Dunia 3D memerlukan **WebGL2**; jika tidak didukung, semua level tetap dapat dibuka lewat tombol angka.

| Kontrol | Fungsi |
| :--- | :--- |
| **→** / **←** | Level berikutnya / sebelumnya |
| **1** sampai **6** | Lompat langsung ke level |
| **M** | Buka peta |
| **Seret mouse** / **scroll** | Memutar / zoom pemandangan 3D |
| **Klik label pulau** | Terbang ke level tersebut |
| Tombol **Musik** / **Narator** | Menyalakan atau mematikan musik latar / narasi suara |
| Tombol **Studio Suara** | Membuka panel untuk mengganti klip narasi dan musik |
| **Esc** | Menutup Studio Suara |
| Tombol **Layar penuh** | Mode layar penuh |

Untuk mereproduksi angka bagian "Satu Kali Uji": buka Arena, biarkan semua nilai default (100 task, 10 VM, seed 42, populasi 40, iterasi 150, bobot 0,5 : 0,25 : 0,25, Metode A Min-Min, Metode B PSO), lalu jalankan balapan dan buka Boss Battle.

---

## Keterbatasan

- Naskah narasi bersifat tetap. Ia tidak ikut berubah mengikuti hasil Arena, sedangkan kesimpulan di Level VI dan Boss Battle terisi otomatis dari angka uji. Kalimat penutup narasi (jadwal hybrid "jauh lebih bagus") lebih kuat daripada selisih yang kami ukur pada pengaturan default.
- Dataset minggu 3 belum dimasukkan ke dalam perhitungan pada laporan ini (lihat catatan di bagian Spesifikasi Pengujian).
- Simulator memodelkan **task independen** pada VM, sedangkan paper PSO-SA asli menangani workflow DAG di lingkungan fog-cloud dengan fitness berbasis pelanggaran *deadline* dan energi. Fitness kami adalah gabungan makespan, biaya, dan energi yang dinormalisasi terhadap Round Robin.
- Implementasi HBA-EO mengikuti varian MOHBA-I (grid indexing) dan tidak mencakup varian dekomposisi MOHBA-II. Parameter paper asli (populasi 200, 300 iterasi, 15 kali run) juga lebih besar daripada default simulator (populasi 40, 150 iterasi).
- Hasil sangat bergantung pada bobot fitness, ukuran masalah, dan jumlah iterasi. Kesimpulan di bawah berlaku untuk pengaturan yang kami uji.

---

## Kesimpulan

1. Penjadwalan task ke VM adalah masalah kombinatorial yang tidak bisa di-brute-force ($10^{100}$ kemungkinan pada 100 task dan 10 VM), sehingga pendekatan metaheuristik dan hybrid diperlukan.
2. Pada pengaturan default, **Hybrid PSO-SA** menjadi yang terbaik pada fitness gabungan (rata-rata 0,6004 pada 30 dataset, lebih baik dari Min-Min 0,6083 dan PSO 0,6079) sekaligus paling seimbang bebannya (DI 0,322).
3. **HBA-EO** paling hemat biaya (rata-rata \$3,104) dan menghasilkan arsip Pareto yang beragam, tetapi pada fitness gabungan belum mengalahkan PSO-SA ataupun Min-Min kecuali bila diberi bibit awal Min-Min.
4. Menggabungkan heuristik klasik sebagai titik awal dengan pencarian metaheuristik adalah strategi yang konsisten memperbaiki hasil. Pemilihan algoritma sebaiknya mengikuti prioritas: PSO-SA untuk fitness gabungan, HBA-EO bila banyak pilihan trade-off biaya dibutuhkan.

---

## Referensi

1. Zhang, S.-W., Wang, J.-S., Zhang, S.-H., Xing, Y.-X., Sui, X.-F., & Zhang, Y.-H. (2025). *Task scheduling in cloud computing systems using multi-objective honey badger algorithm with two hybrid elite frameworks and circular segmentation screening*. Artificial Intelligence Review, 58(2), 48. https://doi.org/10.1007/s10462-024-11032-6
2. Khaledian, N., Khamforoosh, K., Akraminejad, R., Abualigah, L., & Javaheri, D. (2024). *An energy-efficient and deadline-aware workflow scheduling algorithm in the fog and cloud environment*. Computing, 106(1), 109–137. https://doi.org/10.1007/s00607-023-01215-4
3. Hashim, F. A., Houssein, E. H., Hussain, K., Mabrouk, M. S., & Al-Atabany, W. (2022). *Honey Badger Algorithm: New metaheuristic algorithm for solving optimization problems*. Mathematics and Computers in Simulation, 192, 84–110.
4. Kennedy, J., & Eberhart, R. (1995). *Particle swarm optimization*. Proceedings of ICNN'95, 4, 1942–1948.
5. Kirkpatrick, S., Gelatt, C. D., & Vecchi, M. P. (1983). *Optimization by simulated annealing*. Science, 220(4598), 671–680.
