# EPİAŞ Elektrik Tüketimi Tahmini — Final Ödevi Uygulama Planı

> Bu dosya, Türkiye Yapay Zeka Akademisi "Makine Öğrenmesi Final Ödevi" gereksinimleri ile mevcut EPİAŞ projesini birleştiren yol haritasıdır. Claude Code ile fazları sırayla uygula. Her faz sonunda notebook "Restart & Run All" ile baştan sona hatasız çalışmalıdır. Mevcut notebook'taki çalışan kod korunur; bu plan üzerine ekleme ve düzeltme yapar.
>
> **Sürüm 2** — Eklenenler: naif baseline'lar + skill oranı, Ramazan bayrağı, kırılımlı hata ve drift analizi, SHAP dependence, **Senaryo B (gerçek gün-öncesi modeli)** ve **kantil regresyon + optimal teklif (newsvendor)** katmanı.

## 0. Proje Kimliği

- Problem türü: **Regresyon** — saatlik elektrik tüketimi tahmini (MWh)
- Hedef değişken: `Tüketim Miktarı(MWh)` (GZT)
- Veri: EPİAŞ Şeffaflık Platformu — GZT, PTF, SMF, SY (Train: 2025 tam yıl, Test: Oca–Tem 2026), `;` ayraçlı CSV
- Farklılaştırıcı katmanlar: dengesizlik maliyeti simülasyonu, optimal kantil teklif (newsvendor) ve saat-öncesi (A) vs gün-öncesi (B) senaryo kıyası

## 1. Depo Yapısı

```
epias-tuketim-tahmini/
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   ├── train/   (2025 CSV'leri: GZT, PTF, SMF, SY)
│   └── test/    (2026 CSV'leri)
├── epias_final.ipynb        # ana teslim dosyası
└── outputs/figures/          # kaydedilen grafikler (opsiyonel)
```

Not: CSV'ler Kaggle'daki `proje-analiz-verisi` setinden lokale kopyalanır. Dosya yolları notebook'ta `/kaggle/input/...` yerine `data/train/...` ve `data/test/...` olarak güncellenir. Kaggle'a özgü hücreler (kagglehub, os.walk) silinir.

## 2. requirements.txt

```
pandas
numpy
matplotlib
seaborn
scikit-learn
xgboost
lightgbm
scipy
shap
holidays
requests       # Open-Meteo veri çekimi (tek seferlik)
jupyter
optuna        # sadece "ek çalışma" bölümü tutulacaksa
```

Son adımda `pip freeze` ile kullanılan sürümler sabitlenir.

## 3. Fazlar

### Faz 0 — İskelet ve Docstring (Ödev S1)
- [ ] Depo yapısını kur, `.gitignore` ekle (checkpoint'ler, `__pycache__` vb.)
- [ ] Notebook'un en başına docstring/markdown hücresi: projenin amacı, kullanılan kütüphaneler, çalıştırma adımları (`pip install -r requirements.txt` → `jupyter notebook epias_final.ipynb` → Run All)

### Faz 1 — Veri Yükleme ve Problem Tanımı (S2, S3)
- [ ] Mevcut yükleme + `tarih_saat_birlestir` + merge kodu korunur, yollar `data/` altına çevrilir
- [ ] Markdown hücresi: veri setinin hangi problemi çözdüğü (dengesizlik cezası riski) — S2
- [ ] Markdown hücresi: hedef değişken `Tüketim Miktarı(MWh)`, problem türü açıkça **regresyon** — S3

### Faz 2 — Veri İnceleme / EDA (S4)
- [ ] `head()`, `shape`, `info()`, `describe()` çıktıları ve kısa yorumları
- [ ] Grafikler: tüm yılın tüketim zaman serisi, ortalama günlük profil (saat bazında), haftalık profil (haftanın günü), sayısal değişkenler korelasyon ısı haritası

### Faz 3 — Ön İşleme (S5–S8)
- [ ] **S5 Eksik değer:** `isnull().sum()` tablosu; birleştirme sonrası eksikler varsa saatlik interpolasyon; lag kaynaklı NaN'ların yalnızca train başından düşürüldüğü gerekçesiyle yazılır
- [ ] **S6 Encoding:** Sistem Yönü one-hot (mevcut kod) — "Kategorik Dönüşüm" başlığıyla açıkça etiketlenir
- [ ] **S7 Aykırı değer:** Tüketim, PTF, SMF için IQR analizi + boxplot. Karar: tüketim pikleri gerçek talep olduğundan silinmez, **yorumlama** yaklaşımı; SMF uç sıçramaları için isteğe bağlı winsorize (%1–%99) denenir ve etkisi raporlanır. Karar gerekçesi markdown'da yazılır
- [ ] **S8 Ölçekleme:** `Pipeline(StandardScaler → model)` yalnızca Ridge/Lasso için; ağaç tabanlı modellerin ölçeklemeye ihtiyaç duymadığı gerekçesiyle belirtilir. Scaler yalnızca train'e fit edilir (pipeline bunu garanti eder)

### Faz 4 — Öznitelik Mühendisliği ve Seçimi (S9, S10)
- [ ] Mevcut öznitelikler korunur: takvim (Saat, Gün, Ay, Haftanın Günü, Hafta Sonu), lag (1, 2, 24, 168 saat), rolling mean/std (24s), PTF−SMF farkı
- [ ] **Sızıntı düzeltmesi:** aynı saatin PTF, SMF, SY ve türevleri tahmin anında bilinmez → tüm piyasa değişkenleri en az `shift(1)` ile kaydırılır (model "bir saat öncesi" tahminci olarak çerçevelenir)
- [ ] Ek öznitelikler: saatin sin/cos döngüsel kodlaması, `holidays` paketiyle Türkiye resmi tatil bayrağı (bayramlar tüketimi belirgin düşürür)
- [ ] **`Ramazan_Mi` bayrağı:** oruç ayında saatlik tüketim deseni kayar (sahur/iftar); Diyanet takvimine göre aralıklar sabit kodlanır (2025: ≈1–29 Mart, 2026: ≈19 Şubat–19 Mart — uygulanmadan önce doğrulanır)
- [ ] **S10 Öznitelik seçimi:** (a) hedefle ve birbiriyle korelasyon analizi, (b) LassoCV katsayıları veya RF feature importance ile düşük katkılı değişkenlerin elenmesi. Seçim öncesi/sonrası özellik listesi ve validation etkisi kısa tabloyla raporlanır

### Faz 4b — Meteoroloji Entegrasyonu (KARAR: DAHİL — S9'u güçlendirir)
- [ ] Open-Meteo Historical Archive API'den (ücretsiz, anahtarsız) saatlik `temperature_2m` çekilir:
  `https://archive-api.open-meteo.com/v1/archive?latitude={lat}&longitude={lon}&start_date=2025-01-01&end_date=2026-07-22&hourly=temperature_2m&timezone=Europe/Istanbul`
- [ ] Şehirler, koordinatlar ve nüfus ağırlıkları (ulusal talebin vekili):
  İstanbul (41.01, 28.95 — 0.46) · Ankara (39.93, 32.85 — 0.17) · İzmir (38.42, 27.14 — 0.13) · Bursa (40.18, 29.07 — 0.09) · Antalya (36.90, 30.70 — 0.08) · Adana (37.00, 35.32 — 0.07)
- [ ] `Sicaklik_TR` = ağırlıklı ortalama sıcaklık; **tek seferlik** indirilip `data/weather.csv` olarak repoya kaydedilir (notebook API'ye bağımlı kalmaz, değerlendiren kişi internetsiz çalıştırabilir). İndirme kodu notebook'ta yorumlu bir hücre olarak bırakılır
- [ ] Türev öznitelikler: `HDD = max(0, 18 − T)`, `CDD = max(0, T − 18)`, `Sicaklik_Kare` (U-şekilli sıcaklık-tüketim ilişkisi; lineer modeller için kritik), 24 saatlik sıcaklık lag'i ve 24s hareketli ortalama
- [ ] `Tarih_Saat` üzerinden ana tabloya merge — Open-Meteo `timezone=Europe/Istanbul` ile EPİAŞ'la aynı yerel saatte döner, ekstra dönüşüm gerekmez
- [ ] Notebook'un 1.3 bölümü yeniden yazılır: "neden kapsam dışı" gerekçesi kaldırılır; yerine sıcaklığın talebin ana fiziksel sürücüsü olduğu (ısıtma/soğutma yükü) ve modele nasıl katıldığı anlatılır
- [ ] Etki ölçümü: sıcaklık öznitelikleri olmadan/varken validation RMSE kıyası kısa tabloyla raporlanır (S10 seçim analizine de girdi olur)

### Faz 5 — Veri Bölme (S11)
- [ ] Kronolojik ayrım: train 2025'in ilk ~%85'i, validation son %15'i (mevcut mantık), test = 2026 dosyaları
- [ ] Markdown: zaman serisinde shuffle/stratify neden kullanılmadığı (gelecek bilgisinin geçmişe sızmaması) açıklanır

### Faz 6 — Model Eğitimi ve Karşılaştırma (S12, S13)
- [ ] **5 model eğitilir:** Ridge, Lasso, RandomForestRegressor (ödev listesinden) + XGBoost, LightGBM (ileri düzey — varsayılan/temel parametrelerle başlanır)
- [ ] Validation karşılaştırma tablosu: MAE, MSE, RMSE, MAPE, R² — hepsi tek DataFrame'de
- [ ] **Naif baseline satırları:** Lag_1h ("son saat"), Lag_24h ("dün aynı saat"), Lag_168h ("geçen hafta aynı saat") aynı tabloya eklenir; MASE benzeri **skill oranı** (model MAE / Lag_24h-naif MAE) tüm modeller için raporlanır; %1 civarı MAPE'nin naif kıyas bağlamında ne ifade ettiği yorumlanır
- [ ] Kısa yorum: hangi model önde, lineer modeller neden geride (doğrusal olmayan saatlik desenler)

### Faz 7 — Hiperparametre Ayarlama (S14 + çapraz doğrulama)
- [ ] Validation şampiyonuna `RandomizedSearchCV(cv=TimeSeriesSplit(n_splits=5), scoring="neg_root_mean_squared_error", n_iter=25-40)` — ödevin istediği Random Search + çapraz doğrulama tek adımda karşılanır
- [ ] Aranan parametreler (ör. XGBoost için): n_estimators, learning_rate, max_depth, min_child_weight, subsample, colsample_bytree, reg_alpha, reg_lambda
- [ ] En iyi parametreler ve CV skoru yazdırılır; ayar öncesi/sonrası validation farkı gösterilir
- [ ] Mevcut Optuna hücreleri istenirse "Ek Çalışma: Optuna ile Karşılaştırma" başlığı altında korunabilir (ödev gereksinimi RandomizedSearchCV ile karşılanmış olur)

### Faz 8 — Test Değerlendirmesi ve Yorum (S15, S16)
- [ ] Ayarlanmış final model tüm train+validation üzerinde yeniden eğitilir, test'te değerlendirilir
- [ ] **S15 metrikleri eksiksiz:** MAE, MSE, RMSE, R² (+MAPE) yazdırılır
- [ ] Test tablosuna naif baseline satırları (Lag_1h, Lag_24h, Lag_168h) ve skill oranı eklenir
- [ ] Mevcut hata analizi korunur: residual dağılımı, gerçek-tahmin serpme grafiği, son 300 saat çizgi grafiği
- [ ] **Kırılımlı hata analizi:** saat dilimi, haftanın günü, tatil/normal ve ay bazında MAPE tabloları; en kötü 20 saatin tarih + bağlam tablosu (tatil miydi, uç sıcaklık mıydı?)
- [ ] **Zamansal sağlamlık:** test dönemi aylık MAPE çizgi grafiği; 2025 vs 2026 tüketim ve fiyat dağılım kıyasıyla kısa drift yorumu
- [ ] **S16 yorum:** hangi model neden kazandı, önemli değişkenler, **kısıtlar** — özellikle: (1) bir saat öncesi ufuk vs gerçek GÖP'ün 24 saat+ ufku (Senaryo B ile köprü kurulur), (2) 2026'daki yapısal değişikliklere (fiyat rejimi, ekonomi) duyarlılık, (3) uç piklerde hafif underestimation eğilimi, (4) gerçekleşen sıcaklığın, tahmin anında elde olacak kısa vadeli hava tahmininin **vekili (proxy)** olarak kullanıldığı — 1 saatlik ufukta fark ihmal edilebilir, ancak canlı sistemde meteorolojik tahmin verisi kullanılırdı

### Faz 9 — Açıklanabilirlik (S17 bonus)
- [ ] Mevcut feature importance grafiği + SHAP summary plot korunur, final modele göre yeniden üretilir
- [ ] **SHAP dependence plot (`Sicaklik_TR`):** modelin öğrendiği U-şekilli sıcaklık-tüketim ilişkisi gösterilir ve HDD/CDD tasarımıyla ilişkilendirilir
- [ ] SHAP yorumu markdown'da (Lag ve Saat dominansı)

### Faz 9b — Senaryo B: Gerçek Gün-Öncesi Modeli (Farklılaştırıcı)
- [ ] **Özellik kuralı:** yalnızca teklif anında bilinenler — hedef lag'leri {24, 48, 168}, 24 saat kaydırılmış rolling istatistikler, `shift(24)` piyasa değişkenleri, takvim/sin-cos/tatil/Ramazan bayrakları, sıcaklık (hava tahmini vekili — teklif anında tahmin mevcuttur)
- [ ] Model: Faz 7 şampiyonunun mimarisi ve en iyi parametreleri aynen kullanılır (yeniden arama yapılmaz; gerekçesi not düşülür)
- [ ] Değerlendirme: X_val ve test'te tüm metrikler; **Senaryo A ile yan yana kıyas tablosu**
- [ ] Markdown: gerçek GÖP zamanlaması (teklifler D günü ~12:00'de verilir, ufuk 13–35 saat) ve buradaki "≥24 saat gecikme" sadeleştirmesi açıklanır; A→B performans düşüşünün kendisi raporlanacak bir bulgudur

### Faz 10 — Finansal Katman (Farklılaştırıcı)
- [ ] Dengesizlik maliyet fonksiyonu korunur: pozitif dengesizlik `min(PTF,SMF)×0.93`, negatif `max(PTF,SMF)×1.07`
- [ ] **Asimetri katsayıları (yalnızca validation döneminde):** `Cu = 1.07·max(PTF,SMF) − PTF` (eksik kalmanın birim maliyeti), `Co = PTF − 0.93·min(PTF,SMF)` (fazla almanın birim maliyeti); `τ* = mean(Cu) / (mean(Cu) + mean(Co))` hesaplanıp yazdırılır
- [ ] **Newsvendor mantığı markdown'da:** asimetrik maliyet altında optimal teklif ortalama tahmin değil, tüketim dağılımının τ* kantilidir — "alpha > 1" bulgusunun teorik açıklaması budur
- [ ] **Kantil modeller:** LightGBM `objective='quantile'`, alpha ∈ {0.10, 0.50, 0.90, τ*}; her iki senaryonun (A ve B) özellik setiyle eğitilir
- [ ] **Üç teklif stratejisi × iki senaryo:** (a) ortalama tahmin, (b) +alpha (her senaryo için ayrı; yalnızca validation'da `scipy.optimize.minimize` ile), (c) optimal kantil (τ*) teklifi
- [ ] Senaryo B için örnek hafta **P10–P90 bant grafiği** + bant kapsama oranı (hedef ≈ %80)
- [ ] **Nihai tablo (7 satır):** Naif teklif (Lag_24h) + {A, B} × {ortalama, +alpha, kantil}; toplam ceza (TL) ve naife göre tasarruf %; bar grafik
- [ ] Kısa iş yorumu: gerçekçi rakam **B-kantil** satırıdır; A satırları modelin tavanını gösterir

### Faz 11 — Teslim Hazırlığı
- [ ] README.md doldurulur (aşağıdaki iskelet)
- [ ] `pip freeze` ile requirements sürümleri sabitlenir
- [ ] "Restart & Run All" ile uçtan uca temiz çalıştırma; tüm hücre çıktıları kayıtlı halde commit
- [ ] GitHub'a push; repo linki `info@turkiyeyapayzekaakademisi.com` adresine, konu: `Makine Öğrenmesi Final Ödev – Ad Soyad`

## 4. Metodoloji Kuralları (her fazda geçerli)

1. **Zaman sızıntısı yok:** t anında bilinmeyen hiçbir bilgi (aynı saatin SMF/SY/PTF'si) özellik olamaz; tüm piyasa değişkenleri en az 1 saat gecikmeli
2. **Strateji parametreleri validation'da:** alpha ve optimal kantil seviyesi (τ*) yalnızca validation verisiyle belirlenir; test asla strateji ayarında kullanılmaz
3. **CV daima TimeSeriesSplit:** hiçbir aramada KFold/shuffle kullanılmaz
4. **Scaler yalnızca train'e fit:** pipeline dışında manuel transform yapılmaz
5. **dropna yalnızca lag-NaN için:** train'in ilk 168 satırı; test'in başı df_full birleşimi sayesinde train kuyruğundan beslenir, düşürülmez
6. **Senaryo B'de tazelik sınırı:** hiçbir özellik hedef saate 24 saatten yakın bilgi içeremez (sıcaklık hariç — tahmin vekili gerekçesiyle)
7. Her ana adımın başında ödev soru numarasına atıf yapan kısa markdown açıklaması bulunur (değerlendirme kolaylığı)

## 5. README.md İskeleti

```
# EPİAŞ Elektrik Tüketimi Tahmini ve Dengesizlik Maliyeti Optimizasyonu
- Projenin amacı (2-3 cümle: tahmin + finansal optimizasyon)
- Veri seti: EPİAŞ Şeffaflık Platformu (4 dosya, tarih aralıkları, hedef değişken) + Open-Meteo saatlik sıcaklık (6 şehir, nüfus ağırlıklı)
- Problem türü: Regresyon
- Kurulum ve çalıştırma: pip install -r requirements.txt → notebook'u aç → Run All
- Yöntem özeti: öznitelikler → 5 model → RandomizedSearchCV(TimeSeriesSplit) → test → Senaryo A/B + kantil teklif finans simülasyonu
- Sonuç özeti: en iyi model, test metrikleri, senaryo × strateji finans tablosu (TL ve %)
- Kısıtlar: tahmin ufku, sıcaklık proxy, veri dönemi
```

## 6. Kabul Kontrol Listesi (Ödevin 17 Maddesi)

| # | Gereksinim | Karşılayan Faz |
|---|---|---|
| 1 | Docstring | Faz 0 |
| 2 | Veri okuma + problem açıklaması | Faz 1 |
| 3 | Hedef değişken + problem türü | Faz 1 |
| 4 | Head, boyut, tipler, istatistikler | Faz 2 |
| 5 | Eksik değer kontrolü | Faz 3 |
| 6 | Kategorik encoding | Faz 3 |
| 7 | Aykırı değer analizi | Faz 3 |
| 8 | Ölçekleme | Faz 3 |
| 9 | ≥2 anlamlı öznitelik | Faz 4 |
| 10 | ≥1 öznitelik seçimi | Faz 4 |
| 11 | Train/val/test ayrımı | Faz 5 |
| 12 | ≥3 model (Ridge, Lasso, RF + XGB, LGBM) | Faz 6 |
| 13 | Validation karşılaştırması | Faz 6 |
| 14 | Grid/Random Search | Faz 7 |
| 15 | Test: MAE, MSE, RMSE, R² | Faz 8 |
| 16 | Sonuç yorumu + kısıtlar | Faz 8 |
| 17 | Bonus: açıklanabilirlik (SHAP) | Faz 9 |
| — | README, requirements, çalışır kod, GitHub | Faz 11 |

### 6b. Farklılaştırıcı Ekler (puan üstü kontrol)

| Ek | Faz |
|---|---|
| Naif baseline'lar (Lag_1h/24h/168h) + skill oranı | Faz 6, 8 |
| Ramazan bayrağı | Faz 4 |
| Kırılımlı hata analizi + en kötü 20 saat | Faz 8 |
| Aylık MAPE + 2025→2026 drift kıyası | Faz 8 |
| SHAP dependence (sıcaklık U-eğrisi) | Faz 9 |
| Senaryo B — gerçek gün-öncesi modeli | Faz 9b |
| Kantil regresyon + optimal teklif (τ*) | Faz 10 |
| P10–P90 bant kapsaması | Faz 10 |
