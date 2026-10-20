# UTS-Graf-Sains-Data-Graf-Rekomendasi-RetailRocket-dengan-Neo4j
Ujian Tengah Semester Graf Sains Data, Prodi S1 Sains Data, Universitas Negeri Surabaya (Gasal 2026/2027).  
- Nama: Refochella Florentine Gwacevo Pertiwi
- NIM:24031554045
- Kelas:2024 A

**Ringkasan**
Proyek ini memodelkan data interaksi e-commerce RetailRocket (Kaggle, Roman Zykov) sebagai graf di Neo4j, lalu membangun rekomendasi berbasis KNN. Alurnya: filter periode, impor ke graf, Vector Space Modelling (VSM), perhitungan similarity, dan evaluasi dengan Hit@5 dan NDCG@5.

**Data dan periode**
Sumber: events.csv (2.756.101 baris, 3 Mei sampai 18 September 2015). Kolom timestamp berupa unix epoch milidetik.
| | Rentang | Baris | Transaksi | Visitor unik |
|---|---|---|---|---|
| P1 (training) | 1 - 16 Agustus 2015 | 280.001 | 2.169 | 162.337 |
| P2 (testing) | 17 - 20 Agustus 2015 | 80.912 | 837 | 47.074 |
- Rasio 80/20 berdasarkan hari (16 : 4). Berdasarkan baris menjadi 77,6% : 22,4%.
- Pemotongan dilakukan kronologis (bukan acak) agar tidak ada kebocoran data dari masa depan.
- Hanya 3.318 visitor P2 yang juga ada di P1. Sisanya tidak punya riwayat (cold start).
- Hanya P1 yang diimpor ke Neo4j. P2 dipakai sebagai ground truth evaluasi di Python.

**Model Graf**
Graf bipartit: (:Visitor)-[:VIEWED | :ADDED_TO_CART | :BOUGHT]->(:Item)
## Komposisi graf (P1)
| Jenis | Nama | Jumlah |
|---|---|---|
| Simpul | Visitor | 162.337 |
| Simpul | Item | 78.365 |
| Relasi | VIEWED | 270.404 |
| Relasi | ADDED_TO_CART | 7.428 |
| Relasi | BOUGHT | 2.169 |
Total relasi 280.001, sama dengan jumlah baris events_p1.csv. Constraint keunikan dibuat pada Visitor.visitorId dan Item.itemId.

**Metodologi singkat**
VSM. Matriks User x Item (162.337 x 78.365). Nilai sel = bobot interaksi terkuat per pasangan: VIEWED=1, ADDED_TO_CART=3, BOUGHT=5 (diambil maksimum). Bobot adalah asumsi: urutannya berdasar logika, angkanya tidak diturunkan dari data.

Karakteristik matriks. 224.732 elemen tidak nol, density 0,001767%. Median item per visitor = 1 (75% visitor hanya menyentuh 1 item). Ada hub ekstrem: visitor dengan 910 item dan item dengan 1.402 visitor.

## Perbandingan proyeksi
| | User-User | Item-Item |
|---|---|---|
| Simpul | 162.337 | 78.365 |
| Pasangan unik | 2.333.273 | 1.390.845 |
Item-Item dipilih: sekitar 1,7x lebih ringan, dan vektor Item lebih padat (rata-rata 2,87 visitor per item vs 1,38 item per visitor). Selisih efisiensinya tidak besar, dan klaim efektivitas masih dugaan analitis. Similarity (no. 7). Cosine pada proyeksi Item-Item, dihitung di Python (scikit-learn, scipy), item acuan 142466. Hasil ditulis kembali ke Neo4j sebagai relasi SIMILAR_TO {cosine} memakai MERGE. Tie-break: cosine turun, lalu itemId naik. Rekomendasi (2.2). KNN antar-user (cosine, k = 5). Skor item = jumlah (cosine tetangga x bobot interaksi tetangga), hanya untuk item yang belum disentuh user di P1. Pemilihan user: numpy.random.default_rng(seed=42) dari visitor yang ada di P1, membeli di P2, dan punya 5 tetangga serta 3 rekomendasi (12 dari 67 kandidat). Ground truth: item yang dibeli di P2.

## Hasil evaluasi (2.2)
| | Nilai |
|---|---|
| User terpilih (seed 42) | 152963 |
| Kandidat awal / layak | 67 / 12 |
| Hit@5 | 0 |
| NDCG@5 | 0,0000 |
| Rata-rata 12 user layak | Hit@5 = 0, NDCG@5 = 0 |

Hasil nol di semua user layak. Dugaan penyebab: data sangat jarang, tetangga hanya berbagi 1-2 item sehingga kandidat rekomendasi sedikit, jendela P2 hanya 4 hari, dan hanya 3 item direkomendasikan dari 78 ribu item. Ini keterbatasan data dan desain, bukan bukti bahwa KNN selalu gagal.

**Keterbatasan**
- Bobot 1/3/5 subjektif; pengambilan maksimum membuang frekuensi interaksi.
- Pool user acak dibatasi (ada di P1, membeli di P2, punya 5 tetangga), jadi bukan acak dari seluruh visitor.
- Soal meminta 3 item tetapi metrik @5; versi 5 item dilaporkan sebagai pembanding.
- Kesimpulan "Item-Item lebih baik" (no. 6) diterapkan pada similarity no. 7, sedangkan 2.2 memakai KNN antar-user sesuai bunyi soal.
- Plot networkx hanya sampel (bukan seluruh graf) dan bias ke visitor aktif / item berderajat tinggi.
- GDS (Graph Data Science) tidak terpasang, jadi similarity dihitung di Python.

Sumber
Dataset: RetailRocket E-commerce Dataset, Roman Zykov.
Neo4j Python Driver.









