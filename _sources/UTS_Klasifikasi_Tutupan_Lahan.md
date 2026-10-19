# Pemetaan dan Klasifikasi Tutupan Lahan Jawa Timur Menggunakan Citra Sentinel-2 dan Random Forest

**Laporan Proyek Pengganti Ujian Tengah Semester — Pengolahan Data Spasial**  
**Penyusun:** Achmad Saiful Fuadi  
**Wilayah kajian:** Provinsi Jawa Timur, termasuk Madura, pada cakupan raster komposit Sentinel-2 yang tersedia  
**Teknologi:** Google Earth Engine (GEE), Copernicus Browser, Python, GeoPandas, Rasterio, Scikit-learn, Folium/Leaflet, Streamlit, dan Jupyter Book

> **Status laporan:** diperbarui mengikuti model Random Forest *sampling Forest per poligon* yang digunakan pada website akhir. Metrik menunjukkan performa pada data uji berlabel, **bukan** validasi akurasi setiap lokasi di Jawa Timur.

---

## Ringkasan Proyek

Proyek ini mengklasifikasikan lima kelas tutupan lahan, yaitu **Built-up**, **Farm land**, **Water body**, **Forest**, dan **Mangrove**. Sebanyak **328 poligon GeoJSON** didigitasi sebagai data berlabel. Sepuluh band spektral Sentinel-2 dan empat indeks spektral digunakan untuk melatih algoritma **Random Forest**. Model hasil eksperimen diterapkan pada komposit raster wilayah Jawa Timur untuk menghasilkan peta prediksi, yang disajikan bersama poligon sampel melalui **peta Folium/Leaflet** dan **website Streamlit**.

| Indikator utama | Hasil akhir |
|---|---:|
| Poligon GeoJSON | **328** |
| Jumlah kelas | **5** |
| Poligon dengan piksel valid | **313** |
| Seluruh sampel piksel valid | **48.290** |
| Fitur Sentinel-2 | **14** (10 band + 4 indeks) |
| Poligon training / testing | **250 / 63** |
| Piksel training sebelum sampling | **43.316** |
| Piksel training setelah sampling | **6.466** |
| Piksel testing | **4.974** |
| Accuracy | **97,89%** |
| Balanced accuracy | **95,50%** |
| Macro F1-score | **93,76%** |
| Weighted F1-score | **97,96%** |

**Pembedaan penting:** poligon GeoJSON merupakan **sampel digitasi berlabel**, sedangkan layer klasifikasi pada peta berasal dari **prediksi Random Forest untuk piksel raster**. Keduanya ditampilkan sebagai layer yang terpisah.

---

## 1. Business Understanding

### 1.1 Latar Belakang

Tutupan lahan menjelaskan kondisi fisik permukaan bumi, seperti permukiman, pertanian, badan air, dan vegetasi. Jawa Timur memiliki keragaman wilayah daratan dan pesisir sehingga diperlukan cara yang efisien untuk memetakan sebaran kelas-kelas tersebut. **Sentinel-2** menyediakan informasi spektral, sedangkan **Random Forest** mempelajari hubungan fitur spektral dengan label dari poligon digitasi.

### 1.2 Rumusan Masalah

Bagaimana mengekstraksi fitur dari citra Sentinel-2 menggunakan poligon berlabel, melatih dan mengevaluasi Random Forest, memprediksi kelas tutupan lahan pada raster Jawa Timur, serta menyajikan hasil prediksi dan sampel dalam peta yang dapat difilter per kelas?

### 1.3 Tujuan

1. Menyiapkan 328 poligon GeoJSON pada lima kelas.
2. Mengolah 10 band spektral dan 4 indeks dari Sentinel-2.
3. Memisahkan training/testing berdasarkan ID poligon.
4. Melatih, menguji, dan membandingkan strategi sampling Random Forest.
5. Menghasilkan raster klasifikasi dan peta interaktif dengan layer terpisah.
6. Menyajikan hasil pada Streamlit dan laporan Jupyter Book.

### 1.4 Ruang Lingkup dan Batasan

Pemetaan dilakukan pada **piksel valid dalam cakupan raster komposit Jawa Timur**, termasuk Madura. Lima kelas yang dipakai tidak mencakup seluruh kemungkinan tutupan lahan (misalnya semak atau batuan terbuka). Nilai evaluasi berasal dari sampel testing berdasarkan poligon; tidak boleh ditafsirkan sebagai akurasi menyeluruh seluruh wilayah provinsi.

---

## 2. Data Understanding

### 2.1 Sumber Data

Dua jalur data Sentinel-2 digunakan dengan fungsi berbeda:

- **Copernicus Browser:** citra Sentinel-2 Level-2A tanggal **1 Oktober 2026** untuk pengamatan dan referensi visual/digitasi.
- **Google Earth Engine (GEE):** komposit Sentinel-2 Surface Reflectance Harmonized dari **1 Januari sampai sebelum 1 Oktober 2026**, disiapkan menjadi 14 band dan diekspor pada skala **100 meter** untuk ekstraksi sampel dan prediksi wilayah kajian.
- **GeoJSON:** 328 poligon berlabel hasil digitasi; koordinat diselaraskan dengan CRS raster saat pengolahan.

Raster utama prediksi `sentinel2_jatim_14fitur_100m.tif` berdimensi **5.933 × 4.124 piksel**, **EPSG:32749**, resolusi **100 × 100 meter**. Raster `sentinel2_jatim_training_buffer3km_14fitur_100m.tif` digunakan untuk menjangkau sampel training di sekitar pesisir yang kurang tercakup oleh raster administrasi tanpa buffer. File asli Copernicus dan komposit GEE berbeda tujuan dan tidak boleh disamakan resolusinya.

### 2.2 Definisi Operasional dan Distribusi Poligon

| Kelas | Definisi | Poligon |
|---|---|---:|
| Built-up | Permukiman dan kawasan terbangun | 72 |
| Farm land | Lahan pertanian termasuk sawah | 80 |
| Water body | Perairan seperti sungai, danau, waduk, laut | 53 |
| Forest | Hutan daratan non-mangrove | 50 |
| Mangrove | Vegetasi mangrove pada lingkungan pesisir | 73 |
| **Total** | | **328** |

### 2.3 Band dan Indeks

**10 band:** `B02`, `B03`, `B04`, `B05`, `B06`, `B07`, `B08`, `B8A`, `B11`, `B12`.

**4 indeks:**

| Indeks | Rumus | Informasi umum |
|---|---|---|
| NDVI | `(B08 - B04) / (B08 + B04)` | Vegetasi / kehijauan |
| NDMI | `(B08 - B11) / (B08 + B11)` | Kelembapan vegetasi |
| MNDWI | `(B03 - B11) / (B03 + B11)` | Informasi badan air |
| NDBI | `(B11 - B08) / (B11 + B08)` | Informasi permukaan terbangun |

Band **B8A, B11, dan B04** juga relevan untuk mendiskusikan vegetasi Forest dan Mangrove. Namun **MVI belum menjadi fitur dalam model akhir**; model menggunakan 14 fitur di atas. Tidak ada ambang NDVI/MVI yang secara otomatis menetapkan kelas Mangrove.

---

## 3. Data Preparation

### 3.1 Validasi dan Ekstraksi Piksel

Tahap `scripts/02_preprocessing.py` mencocokkan CRS/geometri poligon dengan raster, mengambil nilai 14 fitur pada piksel di dalam poligon, lalu membuang nilai yang tidak valid atau tidak memenuhi kriteria filter piksel. Sebanyak **313 dari 328 poligon** menghasilkan piksel valid; **15 poligon** tidak memberikan piksel valid menurut kriteria pengolahan. Total sampel valid **48.290 piksel**.

Output utama:

- `outputs/reports/training.csv`
- `outputs/reports/testing.csv`

Setiap baris data berisi 14 nilai fitur, `polygon_id`, dan `label`.

### 3.2 Pembagian Training dan Testing

Pembagian dilakukan secara terstratifikasi pada **tingkat poligon**, sehingga ID poligon training tidak digunakan kembali pada testing. Proporsi banyaknya poligon sekitar 80:20, sedangkan jumlah piksel berbeda karena luas setiap poligon berbeda.

| Kelompok | Poligon | Piksel |
|---|---:|---:|
| Training | 250 | 43.316 |
| Testing | 63 | 4.974 |
| **Total** | **313** | **48.290** |

Pemisahan berdasarkan poligon mengurangi kebocoran data antar piksel dalam satu poligon, tetapi piksel dari poligon berbeda masih mungkin mempunyai kemiripan spasial.

---

## 4. Modeling — Random Forest

### 4.1 Algoritma dan Parameter

Random Forest menggabungkan keputusan banyak pohon (*ensemble*) untuk memprediksi kelas tutupan lahan berdasarkan 14 fitur spektral.

| Parameter | Nilai |
|---|---|
| `n_estimators` | `200` |
| `max_features` | `sqrt` |
| `min_samples_leaf` | `1` |
| `random_state` | `42` |
| `class_weight` | `balanced` |
| `n_jobs` | `-1` |
| Fitur | 14 |
| Kelas | 5 |

### 4.2 Sampling Data Training

Data training tidak seimbang: Forest memiliki **38.850 piksel** sebelum pembatasan. Seluruh kelas dibatasi maksimal **2.000 piksel** hanya pada training; data testing tidak diubah. Hasilnya:

| Kelas | Piksel training akhir |
|---|---:|
| Built-up | 932 |
| Farm land | 1.590 |
| Water body | 1.146 |
| Forest | 2.000 |
| Mangrove | 798 |
| **Total** | **6.466** |

Pada `03_training_rf.py`, sampling Forest awal dilakukan acak dari kumpulan seluruh piksel kelas. Pada `03_training_rf_uji_sampling_poligon.py`, **2.000 piksel Forest diambil lebih merata dari 40 poligon Forest training** (kebanyakan sekitar 51–52 piksel per poligon, dibatasi ketersediaan piksel). Empat kelas lain mengikuti sampling semula. Tujuannya memperluas keterwakilan poligon, bukan mengubah label atau memakai aturan koreksi peta.

### 4.3 Training dan Penyimpanan Model

Pola inti pelatihan:

```python
rf_model = RandomForestClassifier(
    n_estimators=200,
    max_features="sqrt",
    min_samples_leaf=1,
    random_state=42,
    class_weight="balanced",
    n_jobs=-1,
)
rf_model.fit(X_train, y_train)
y_pred = rf_model.predict(X_test)
```

Model eksperimen disimpan pada `outputs/models/random_forest_model_sampling_poligon.joblib`, sedangkan laporan eksperimen disimpan di `outputs/reports/eksperimen_sampling_poligon/`. Model utama sebelum eksperimen tidak ditimpa.

### 4.4 Alur Kerja

**Sentinel-2 dan GeoJSON → ekstraksi 14 fitur → pemisahan berdasarkan poligon → sampling training → Random Forest → evaluasi testing → prediksi raster Jawa Timur.**

---

## 5. Evaluation — Hasil Model Eksperimen

### 5.1 Perbandingan Model Awal dan Eksperimen

| Metrik | Model awal | Model sampling per poligon |
|---|---:|---:|
| Accuracy | 97,77% | **97,89%** |
| Balanced accuracy | **95,95%** | 95,50% |
| Macro F1 | 93,75% | **93,76%** |
| Weighted F1 | 97,84% | **97,96%** |

Model eksperimen dipakai untuk peta akhir sebagai kandidat yang lebih baik secara kewajaran geografis pada sebagian lokasi; balanced accuracy sedikit turun sehingga **tidak semua metrik mengalami peningkatan**.

### 5.2 Classification Report — Model Eksperimen

| Kelas | Precision | Recall | F1-score | Support |
|---|---:|---:|---:|---:|
| Built-up | 0,9922 | 0,9697 | 0,9808 | 264 |
| Farm land | 0,8289 | 0,9594 | 0,8894 | 394 |
| Water body | 0,8246 | 0,9216 | 0,8704 | 51 |
| Forest | 0,9971 | 0,9831 | 0,9900 | 4.146 |
| Mangrove | 0,9739 | 0,9412 | 0,9573 | 119 |

### 5.3 Confusion Matrix — Model Eksperimen

Baris menunjukkan label aktual, kolom menunjukkan hasil prediksi.

| Aktual \ Prediksi | Built-up | Farm land | Water | Forest | Mangrove |
|---|---:|---:|---:|---:|---:|
| Built-up | **256** | 8 | 0 | 0 | 0 |
| Farm land | 1 | **378** | 3 | 12 | 0 |
| Water body | 0 | 1 | **47** | 0 | 3 |
| Forest | 1 | 69 | 0 | **4.076** | 0 |
| Mangrove | 0 | 0 | 7 | 0 | **112** |

Contoh: dari 119 piksel testing Mangrove, 112 diprediksi dengan benar sebagai Mangrove dan 7 keliru menjadi Water body. Evaluasi seperti ini harus dibedakan dari hasil peta piksel provinsi.

### 5.4 Feature Importance

Empat fitur dengan importance tertinggi pada model eksperimen:

| Fitur | Importance |
|---|---:|
| MNDWI | 0,156623 |
| B12 | 0,140924 |
| NDVI | 0,104143 |
| B11 | 0,101197 |

Nilai *impurity-based importance* ini menunjukkan kontribusi relatif fitur menurut model, bukan bukti sebab-akibat atau akurasi geografis untuk satu kelas tertentu.

### 5.5 Grafik Evaluasi

Grafik dalam folder `uts_assets/` pada versi laporan sebelumnya berasal dari **eksperimen lama dengan akurasi 86,93%**. Grafik tersebut **tidak ditampilkan** di versi terbaru agar tidak menimbulkan kontradiksi. Jika ingin menampilkan grafik model 97,89%, ekspor ulang visualisasi dari laporan `outputs/reports/eksperimen_sampling_poligon/` ke `materi/uts_assets/`, lalu tambahkan tautan gambar yang baru.

---

## 6. Visualization — Peta Klasifikasi dan Sampel

### 6.1 Prediksi Raster Seluruh Jawa Timur

Model eksperimen diterapkan menggunakan `scripts/14_prediksi_sampling_poligon.py` pada raster Sentinel-2 14 band. Raster diproses per blok maksimal **512 × 512 piksel**, menghindari pemuatan keseluruhan data secara berlebihan dalam RAM. Hasil prediksi diberi kode: `0` NoData, `1` Built-up, `2` Farm land, `3` Water body, `4` Forest, `5` Mangrove.

| Kelas | Piksel prediksi |
|---|---:|
| Built-up | 273.827 |
| Farm land | 1.246.484 |
| Water body | 234.820 |
| Forest | 2.993.284 |
| Mangrove | 47.422 |
| **Total** | **4.795.837** |

Hasil disimpan sebagai `outputs/maps/klasifikasi_rf_jatim_sampling_poligon.tif` dan PNG preview. Jumlah piksel ini adalah **prediksi**, bukan label referensi yang sudah diverifikasi di lapangan.

### 6.2 Peta Interaktif Leaflet

File `scripts/10_peta_interaktif_rf.py` mereproyeksikan raster ke EPSG:4326, mengubah kode kelas menjadi lima overlay warna transparan, dan menambahkan **lima layer GeoJSON** yang terpisah. Peta mendukung zoom, pan, pilihan basemap Esri/Sentinel-2, legenda, checkbox, dan tooltip sampel.

**Klasifikasi** menunjukkan hasil prediksi piksel Random Forest. **Sampel** menunjukkan geometri poligon digitasi. Keduanya jangan ditukar dalam penjelasan laporan.

#### Peta model eksperimen (HTML)

```{raw} html
<iframe src="uts_assets/peta_interaktif_rf_sampling_poligon.html" width="100%" height="700" style="border:1px solid #d3d9df; border-radius:8px;" title="Peta interaktif klasifikasi Random Forest Jawa Timur"></iframe>
```

> **Berkas wajib sebelum build Jupyter Book:** salin `outputs/maps/peta_interaktif_rf_sampling_poligon.html` dari repository UTS ke `materi/uts_assets/peta_interaktif_rf_sampling_poligon.html` pada repository PSD. Jika tidak disalin, iframe akan kosong/404. File HTML tersebut perlu diterbitkan bersama hasil build GitHub Pages.

**[Buka peta interaktif di halaman penuh](uts_assets/peta_interaktif_rf_sampling_poligon.html)**

### 6.3 Pemeriksaan Kewajaran Geografis

Dibandingkan model awal, piksel yang diprediksi **Mangrove pada elevasi lebih dari 300 meter** berkurang dari **11.284 menjadi 6.806** (sekitar 39,7%). Prediksi Mangrove di luar mask pesisir perkiraan 2 km turun dari **43.774 menjadi 35.854**. Angka ini menunjukkan perbaikan kewajaran pada sebagian piksel, tetapi mask pesisir hanya pendekatan kasar dan masih terdapat kemungkinan salah klasifikasi.

Mangrove tidak dapat ditentukan dari NDVI saja. Model akhir memakai 14 fitur spektral dan **belum** memasukkan MVI, DEM, atau jarak pesisir sebagai fitur training. Sebagian piksel Water body dan Built-up di pegunungan juga masih perlu validasi independen. Tidak ada kelas koreksi merah tambahan dalam hasil akhir.

---

## 7. Deployment dan Penyajian Informasi

### 7.1 Jupyter Book di GitHub Pages

Jupyter Book merupakan **laporan statis** yang berisi metodologi, hasil numerik, serta peta HTML yang bisa ditampilkan langsung. Berkas Markdown ini ditempatkan di `materi/UTS_Klasifikasi_Tutupan_Lahan.md`, sedangkan aset peta ditempatkan di `materi/uts_assets/`.

### 7.2 Streamlit sebagai Aplikasi Interaktif

Aplikasi Streamlit menggunakan `app.py` dan menyediakan **delapan halaman**: Beranda, Business Understanding, Data Understanding, Data Preparation, Random Forest, Evaluation, Peta Interaktif, dan Kesimpulan. Berkas HTML Folium yang telah dibuat digunakan kembali sehingga situs tidak perlu melakukan training/prediksi saat dibuka.

**Repository kode UTS:** [Fu4di4chmd/UTS-Klasifikasi-Tutupan-Lahan](https://github.com/Fu4di4chmd/UTS-Klasifikasi-Tutupan-Lahan)

**Website Streamlit:** tambahkan URL `.streamlit.app` milik proyek setelah memastikan alamat pastinya. Tidak perlu menuliskan alamat perkiraan yang belum diverifikasi.

### 7.3 Urutan Script Reproduksi

Jika **raster sumber lokal** dan dependensi sudah tersedia:

```powershell
py -3.9 scripts/01_data_understanding.py
py -3.9 scripts/02_preprocessing.py
py -3.9 scripts/03_training_rf_uji_sampling_poligon.py
py -3.9 scripts/14_prediksi_sampling_poligon.py
py -3.9 scripts/10_peta_interaktif_rf.py
py -3.9 -m streamlit run app.py
```

Script `04_evaluation.py` dan `05_create_map.py` merupakan bagian alur awal; evaluasi akhir tersedia di folder laporan eksperimen. Raster besar dari GEE tidak disertakan di repository GitHub biasa, sehingga proses dari nol memerlukan data lokal tersebut.

---

## 8. Kesimpulan

Proyek UTS telah menghasilkan dataset **328 poligon GeoJSON**, dengan **313 poligon** menghasilkan **48.290 piksel valid**. Random Forest memakai **14 fitur Sentinel-2** dan lima kelas tutupan lahan. Model eksperimen sampling Forest per poligon menggunakan **6.466 piksel training** dan dievaluasi pada **4.974 piksel testing**. Metrik model akhir ialah **Accuracy 97,89%**, **Balanced Accuracy 95,50%**, **Macro F1 93,76%**, dan **Weighted F1 97,96%**.

Model dipakai untuk menghasilkan raster prediksi Jawa Timur, kemudian disajikan sebagai peta interaktif dengan **lima checkbox klasifikasi dan lima checkbox sampel** pada Folium/Leaflet maupun Streamlit.

**Keterbatasan:** resolusi grid 100 meter, lima kelas tidak mewakili seluruh jenis tutupan, potensi piksel campuran, keterbatasan distribusi dan kedekatan sampel, serta kemungkinan kesalahan spasial Mangrove/Water body/Built-up. Accuracy testing bukan jaminan akurasi peta seluruh provinsi. Pengembangan lebih lanjut dapat mencakup verifikasi lapangan atau data referensi independen, perbaikan training, dan pengujian fitur topografi maupun indeks khusus Mangrove.

---

**Sumber data dan hasil:** Citra Sentinel-2 Level-2A melalui Copernicus Browser; komposit Sentinel-2 melalui Google Earth Engine; poligon digitasi; keluaran preprocessing, model dan laporan Random Forest, serta peta Leaflet dari proyek UTS. Angka pada halaman ini merujuk pada model eksperimen sampling Forest per poligon.
