# Prediksi Customer Lifetime Value (CLV)

Capstone project regresi untuk memprediksi **Customer Lifetime Value** pelanggan asuransi mobil, menggunakan data polis, klaim, dan demografi — agar tim Marketing & Retention bisa memprioritaskan budget ke pelanggan yang paling bernilai, **sebelum** nilai tersebut sepenuhnya terealisasi.

## Business Problem

Saat ini, tim Marketing & Customer Retention baru mengetahui pelanggan mana yang bernilai tinggi **setelah kejadiannya** — dari data billing dan klaim historis yang sudah terakumulasi. Hal ini menyulitkan pengambilan keputusan, baik pada saat akuisisi maupun renewal, mengenai pelanggan mana yang layak mendapat budget retensi.

**Goal:** Membangun model regresi yang memprediksi `Customer Lifetime Value` berdasarkan atribut polis, klaim, dan demografi pelanggan, dengan error (MAE) yang cukup rendah — relatif terhadap sebaran CLV pada data — sehingga bisa diandalkan untuk mengelompokkan pelanggan ke dalam tier nilai (misalnya top 20% vs bottom 20%) untuk prioritisasi marketing.

Goal dianggap tercapai jika R² model jelas mengungguli baseline naif (memprediksi rata-rata), dan MAE jauh lebih kecil dibanding standar deviasi CLV.

## Dataset

- **Ukuran awal:** 5.669 baris, 11 kolom, tanpa missing value
- **Duplikat:** 618 baris (10.9%) ditemukan dan dihapus → **5.051 baris final** digunakan untuk modeling
- **Target:** `Customer Lifetime Value` (numerik, kontinu, right-skewed)
- **Unit analisis:** satu baris per pelanggan

| Kolom | Arti dari Sisi Bisnis |
|---|---|
| `Vehicle Class` | Tipe/tier kendaraan yang diasuransikan — proxy untuk nilai aset dan profil risiko pelanggan |
| `Coverage` | Tier paket asuransi (Basic / Extended / Premium) — proxy kesediaan membayar |
| `Renew Offer Type` | Jenis penawaran renewal yang ditawarkan — mencerminkan histori engagement marketing |
| `EmploymentStatus` | Status pekerjaan pelanggan — proxy stabilitas finansial |
| `Marital Status` | Married, Single, atau Divorced |
| `Education` | Tingkat pendidikan tertinggi — segmen sosioekonomi |
| `Number of Policies` | Jumlah polis yang dimiliki — sinyal kedalaman relasi/cross-sell |
| `Monthly Premium Auto` | Nominal yang dibayar per bulan — sinyal revenue langsung |
| `Total Claim Amount` | Total nominal klaim yang diajukan — sinyal cost langsung |
| `Income` | Pendapatan tahunan — proxy daya beli |
| `Customer Lifetime Value` | **Target** — total expected profit dari pelanggan selama masa hubungannya dengan perusahaan |

## Metodologi

1. **EDA** — distribusi CLV (right-skewed), korelasi fitur numerik terhadap CLV, dan sebaran CLV per segmen kategorikal.
   - `Monthly Premium Auto` adalah prediktor linear individual terkuat (r ≈ 0.40).
   - `Total Claim Amount` berkorelasi lemah (r ≈ 0.22) — sebagian besar karena kolinearitasnya dengan premium.
   - `Income` (r ≈ 0.03) dan `Number of Policies` (r ≈ 0.02) nyaris tidak berkorelasi linear dengan CLV secara individual — namun ini berubah drastis saat dilihat lewat model non-linear (lihat Feature Importance).
   - `Vehicle Class` dan `Coverage` jelas memengaruhi CLV (median lebih tinggi untuk vehicle class mewah dan tier coverage lebih lengkap); `Renew Offer Type`, `EmploymentStatus`, `Marital Status`, dan `Education` hampir tidak berpengaruh.
2. **Data cleaning** — 618 baris duplikat (10.9%) dihapus sebelum feature engineering.
3. **Feature engineering** — dibuat fitur rasio yang lebih merepresentasikan hubungan bisnis:
   - `Claim_to_Premium_Ratio` — proxy loss-ratio
   - `Premium_to_Income_Ratio` — proxy affordability
   - `Policies_per_Claim_Dollar` — kedalaman relasi relatif terhadap cost
   - `Is_High_Value_Vehicle` — flag untuk vehicle class luxury/sports
   - `Coverage_Level` — encoding ordinal untuk tier coverage (Basic < Extended < Premium)
4. **Modeling** — membandingkan tiga algoritma terhadap baseline prediksi rata-rata:
   - Linear Regression
   - Random Forest Regressor
   - Gradient Boosting Regressor
5. **Evaluasi** — MAE, RMSE, R², analisis error per kuartil CLV, feature importance, dan model terbaik disimpan sebagai file `.pkl` untuk reuse.

### Alasan Pemilihan Metrics

- **MAE** — rata-rata error dalam dolar, paling mudah dikomunikasikan ke stakeholder non-teknis
- **RMSE** — memberi penalti lebih besar untuk error besar, penting karena salah estimasi signifikan pada pelanggan bernilai tinggi lebih costly dibanding sedikit meleset pada pelanggan rata-rata
- **R²** — seberapa besar variansi CLV yang bisa dijelaskan model dibanding baseline

MAPE sengaja tidak digunakan karena metric ini menjadi tidak stabil pada target moneter yang right-skewed seperti CLV.

## Hasil Model

| Model | MAE | RMSE | R² |
|---|---:|---:|---:|
| **Gradient Boosting (terbaik)** | **1.693,56** | **3.948,63** | **0,672** |
| Random Forest | 1.626,77 | 4.069,63 | 0,652 |
| Linear Regression | 3.859,06 | 6.346,19 | 0,154 |
| Baseline (mean) | 4.380,39 | 6.909,99 | -0,003 |

Gradient Boosting dipilih sebagai model terbaik (R² tertinggi), meski Random Forest sedikit lebih unggul di MAE — keduanya jauh mengungguli baseline dan Linear Regression, mengonfirmasi bahwa hubungan antar fitur dengan CLV memang bersifat non-linear.

### Error per Kuartil CLV

| Kuartil | Rata-rata Absolute Error |
|---|---:|
| Q1 (terendah) | ~187 |
| Q2 | ~299 |
| Q3 | ~1.792 |
| Q4 (tertinggi) | ~5.049 |

Error meningkat tajam pada kuartil CLV tertinggi — model paling kurang andal justru pada segmen pelanggan paling bernilai bagi bisnis.

### Feature Importance (Top 5, model terbaik)

1. `Number of Policies` (0,64) — meski korelasi linearnya nyaris nol di EDA, fitur ini justru paling penting di model non-linear, kemungkinan karena efeknya multiplikatif/berinteraksi dengan fitur lain
2. `Monthly Premium Auto` (0,30)
3. `Income` (0,015)
4. `Premium_to_Income_Ratio` (0,012)
5. `Claim_to_Premium_Ratio` (0,006)

## Temuan Utama

- Ketiga model jelas mengungguli baseline mean-only, mengonfirmasi bahwa goal di business problem tercapai.
- **Keandalan bervariasi per segmen:** error prediksi paling besar pada kuartil CLV tertinggi — justru segmen yang paling penting bagi bisnis. Ini efek umum pada target yang right-skewed, di mana contoh pelanggan bernilai tinggi lebih sedikit tersedia untuk training.
- **`Number of Policies` dan `Monthly Premium Auto` mendominasi feature importance**, sementara fitur demografis (marital status, education, employment) kontribusinya sangat kecil.
- **Implikasi praktis:** model lebih cocok digunakan untuk **ranking/segmentasi** pelanggan ke dalam tier nilai, dibanding untuk forecasting nilai dolar yang presisi pada satu pelanggan bernilai tinggi secara individual.

## Rekomendasi

1. Hyperparameter tuning (GridSearchCV / RandomizedSearchCV) pada Gradient Boosting untuk meningkatkan akurasi lebih lanjut.
2. Mencoba XGBoost / LightGBM, yang seringkali mengungguli Gradient Boosting bawaan sklearn pada data tabular.
3. Melatih model pada `log1p(CLV)` untuk berpotensi meningkatkan akurasi pada segmen CLV tinggi (Q4).
4. Menambahkan dimensi waktu (tenure, histori renewal) jika tersedia, untuk beralih dari estimasi CLV statis menuju model trajectory lifetime yang sesungguhnya.
5. Mengoperasionalkan sebagai tier nilai (High/Medium/Low), bukan angka dolar eksak, mengingat error yang besar pada segmen CLV tinggi.

## Tech Stack

- Python, pandas, numpy
- scikit-learn (`Pipeline`, `ColumnTransformer`, `LinearRegression`, `RandomForestRegressor`, `GradientBoostingRegressor`, `DummyRegressor`)
- matplotlib, seaborn
- pickle (menyimpan/memuat model terbaik)

## Struktur Repository

```
.
├── CLV_Prediction_Assignment.ipynb   # Analisis lengkap: EDA, feature engineering, modeling, evaluasi
├── data_customer_lifetime_value.csv  # Dataset (tambahkan file kamu sendiri)
├── clv_model.pkl                     # Model terbaik (Gradient Boosting) hasil training, disimpan via pickle
└── README.md
```

## Cara Menjalankan

```bash
pip install pandas numpy matplotlib seaborn scikit-learn
jupyter notebook CLV_Prediction_Assignment.ipynb
```

Pastikan `data_customer_lifetime_value.csv` berada di folder yang sama dengan notebook sebelum dijalankan.

Untuk menggunakan model yang sudah dilatih tanpa menjalankan ulang seluruh notebook:

```python
import pickle

with open("clv_model.pkl", "rb") as f:
    model = pickle.load(f)

predictions = model.predict(X_new)  # X_new harus punya kolom fitur yang sama seperti data training
```

## Isi Notebook

1. Business Problem & Data Understanding
2. Import Library
3. Load Data
4. Exploratory Data Analysis (EDA)
5. Data Cleaning, Feature Selection & Feature Engineering
6. Analytics — Algorithm & Evaluation Metrics
7. Analisis Keandalan (error per segmen CLV)
8. Feature Importance
9. Conclusion and Recommendation
10. Appendix: Save Model in Pickle
