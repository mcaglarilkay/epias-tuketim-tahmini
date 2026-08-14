# Claude Code Komut Listesi — Faz Faz (Sürüm 2)

> Kullanım: Repo klasöründe terminali açıp `claude` yazarak oturumu başlat. Komutları sırayla, tek tek yapıştır. Her komutun sonunda Claude Code'dan "Faz X tamam" özeti bekle; hata varsa çözdürmeden bir sonrakine geçme. Toplam 14 komut (0–13).

## Başlamadan Önce (elle yapılacaklar)

- [ ] Proje klasörünü oluştur, `PLAN.md`'yi köke koy
- [ ] Notebook'u `epias_final.ipynb` adıyla köke kopyala
- [ ] CSV'leri `data/train/` ve `data/test/` altına koy (GZT, PTF, SMF, SY × 2 dönem)
- [ ] Klasörde `claude` komutuyla oturumu başlat

---

## Komut 0 — Oturum Açılışı (bağlam yükleme)

```
PLAN.md dosyasını ve epias_final.ipynb notebook'unu baştan sona oku. Projenin amacını, mevcut kodun ne yaptığını ve fazları 5-6 maddede özetle. data/ klasöründeki dosyaları listele ve PLAN.md'deki beklentiyle uyuşup uyuşmadığını kontrol et. Henüz hiçbir değişiklik yapma.
```

## Komut 1 — Faz 0: İskelet ve Docstring

```
PLAN.md Faz 0'ı uygula: git deposunu başlat, .gitignore oluştur (ipynb_checkpoints, __pycache__, .DS_Store), requirements.txt'yi plandaki listeyle oluştur. Notebook'un en başına ödev S1'in istediği açıklama hücresini ekle: projenin amacı, kullanılan kütüphaneler, çalıştırma adımları. İlk commit'i at ve "Faz 0 tamam" diyerek özetle.
```

## Komut 2 — Faz 1 + 2: Veri Yükleme ve EDA

```
PLAN.md Faz 1 ve Faz 2'yi uygula: dosya yollarını data/train ve data/test klasörlerine çevir, Kaggle'a özgü hücreleri (kagglehub, os.walk) sil. S2 için veri setinin çözdüğü problemi, S3 için hedef değişkenin "Tüketim Miktarı(MWh)" ve problem türünün regresyon olduğunu markdown hücreleriyle açıkça yaz. S4 için EDA'yı genişlet: describe(), yıllık tüketim zaman serisi grafiği, saatlik ve haftalık ortalama profil grafikleri, sayısal değişkenler korelasyon ısı haritası — her grafiğin altına 1-2 cümle yorum. Notebook'u jupyter nbconvert --execute ile baştan sona çalıştırıp hatasız olduğunu doğrula, commit at.
```

## Komut 3 — Faz 3: Ön İşleme (S5–S8)

```
PLAN.md Faz 3'ü uygula. S5: isnull().sum() tablosu göster; birleştirme kaynaklı eksik varsa saatlik interpolasyonla doldur; lag kaynaklı NaN'ların yalnızca train başından düşürüleceğini gerekçesiyle markdown'a yaz. S6: mevcut Sistem Yönü one-hot dönüşümünü "Kategorik Encoding (S6)" başlığıyla etiketle. S7: Tüketim, PTF ve SMF için IQR analizi ve boxplot'lar; tüketim piklerinin gerçek talep olduğu için silinmeyeceğini (yorumlama yaklaşımı) yaz; SMF uç değerleri için %1-%99 winsorize'ı dene, etkisini raporla ve nihai kararı gerekçelendir. S8: hangi değişkenlerin ölçekleneceğini gösteren bir bölüm ekle; StandardScaler'ın Faz 6'da Ridge/Lasso pipeline'ı içinde uygulanacağını, ağaç modellerine gerekmediğini markdown'da açıkla. Çalıştır, doğrula, commit at.
```

## Komut 4 — Faz 4: Öznitelik Mühendisliği ve Seçimi (S9–S10)

```
PLAN.md Faz 4'ü uygula. Önce sızıntı düzeltmesi: PTF (TL/USD/EUR), SMF, Sistem_Yonu_* ve PTF_SMF_Farki kolonlarının tümünü shift(1) ile en az 1 saat kaydır, eşzamanlı ham sürümlerini özellik listesinden çıkar; nedenini (tahmin anında bilinmezlik, model "bir saat öncesi" tahminci) markdown'da yaz. Sonra yeni öznitelikler: saatin sin/cos döngüsel kodlaması, holidays paketiyle Türkiye resmi tatil bayrağı (Tatil_Mi) ve Ramazan_Mi bayrağı — Diyanet takvimine göre 2025 (≈1–29 Mart) ve 2026 (≈19 Şubat–19 Mart) aralıklarını doğrulayıp sabit kodla. S10 için: hedefle korelasyon analizi + ölçeklenmiş train üzerinde LassoCV katsayılarıyla düşük katkılı özellikleri belirle; elenen özellik listesini ve gerekçesini tabloyla göster; nihai özellik listesini tek bir değişkende tut. Çalıştır, doğrula, commit at.
```

## Komut 5 — Faz 4b: Meteoroloji Entegrasyonu

```
PLAN.md Faz 4b'yi uygula. Open-Meteo Historical Archive API'den plandaki 6 şehir için (koordinatlar ve nüfus ağırlıkları PLAN.md'de) 2025-01-01 – 2026-07-22 aralığında saatlik temperature_2m çek, nüfus ağırlıklı ortalamayla Sicaklik_TR kolonunu üret ve data/weather.csv olarak kaydet. Notebook'ta varsayılan akış CSV'den okusun; API çekim kodu yorumlanmış bir hücre olarak kalsın. Tarih_Saat üzerinden ana tabloya merge et. Türev öznitelikler: HDD = max(0, 18−T), CDD = max(0, T−18), Sicaklik_Kare, 24 saatlik sıcaklık lag'i ve 24s hareketli ortalama. Notebook'un 1.3 bölümünü yeniden yaz: "neden kapsam dışı" gerekçesini kaldır, sıcaklığın talebin ana fiziksel sürücüsü olduğunu ve gerçekleşen sıcaklığın kısa vadeli hava tahmininin vekili (proxy) olarak kullanıldığını açıkla. Çalıştır, doğrula, commit at.
```

## Komut 6 — Faz 5: Veri Bölme (S11)

```
PLAN.md Faz 5'i uygula: kronolojik ayrımı netleştir — train 2025'in ilk ~%85'i, validation son %15'i, test 2026 dosyaları. Zaman serisinde shuffle ve stratify'ın neden kullanılmadığını (gelecek bilgisinin geçmişe sızmaması) S11 başlığıyla markdown'da açıkla. Üç setin boyutlarını ve tarih aralıklarını yazdır. Çalıştır, doğrula, commit at.
```

## Komut 7 — Faz 6: 5 Modelin Eğitimi ve Karşılaştırma (S12–S13)

```
PLAN.md Faz 6'yı uygula: 5 model eğit — Ridge ve Lasso Pipeline(StandardScaler → model) içinde, RandomForestRegressor, XGBoost ve LightGBM ham veriyle, hepsi makul varsayılan parametrelerle, X_tr üzerinde eğitilip X_val üzerinde değerlendirilsin. Validation karşılaştırma tablosu tek DataFrame'de: MAE, MSE, RMSE, MAPE, R². Aynı tabloya üç naif baseline satırı ekle: Lag_1h, Lag_24h, Lag_168h; ayrıca tüm modeller için skill oranını (model MAE / Lag_24h-naif MAE) hesapla ve %1 civarı MAPE'nin naif kıyas bağlamında ne ifade ettiğini markdown'da yorumla. Ek olarak sıcaklık ablasyonu: en iyi gradient boosting modelini sıcaklık öznitelikleri olmadan ve varken eğitip validation RMSE kıyasını küçük bir tabloyla göster. Sonuçları yorumlayan kısa bir markdown yaz: hangi model önde, lineer modeller neden geride. Çalıştır, doğrula, commit at.
```

## Komut 8 — Faz 7: Hiperparametre Ayarlama (S14)

```
PLAN.md Faz 7'yi uygula: validation şampiyonu modele RandomizedSearchCV uygula — cv=TimeSeriesSplit(n_splits=5), scoring="neg_root_mean_squared_error", n_iter=30, n_jobs=-1, arama X_tr üzerinde yapılsın, parametre uzayı PLAN.md'deki gibi. En iyi parametreleri ve CV skorunu yazdır; ayar öncesi/sonrası X_val performans kıyasını tabloyla göster. Mevcut Optuna hücrelerini "Ek Çalışma: Optuna ile Karşılaştırma" başlığı altına taşı ve ana akışın ödev gereği RandomizedSearchCV olduğunu markdown'da belirt. Çalıştır, doğrula, commit at.
```

## Komut 9 — Faz 8 + 9: Test Değerlendirmesi, Yorum ve Açıklanabilirlik (S15–S17)

```
PLAN.md Faz 8 ve 9'u uygula: ayarlanmış şampiyonu en iyi parametrelerle X_tr + X_val (tüm 2025) üzerinde yeniden eğit ve test setinde değerlendir. S15 metriklerini eksiksiz yazdır: MAE, MSE, RMSE, R² ve MAPE; tabloya naif baseline satırlarını (Lag_1h, Lag_24h, Lag_168h) ve skill oranını da ekle. Hata analizi grafiklerini (residual dağılımı, gerçek-tahmin serpme, son 300 saat çizgi grafiği) bu final modelle yeniden üret. Kırılımlı hata analizi ekle: saat dilimi, haftanın günü, tatil/normal ve ay bazında MAPE tabloları + en kötü 20 saatin tarih-bağlam tablosu (tatil miydi, uç sıcaklık mıydı?). Zamansal sağlamlık için test dönemi aylık MAPE çizgi grafiği ve 2025 vs 2026 tüketim/fiyat dağılım kıyasıyla kısa drift yorumu yaz. S16 için "Sonuç Yorumu ve Kısıtlar" markdown'ı yaz — PLAN.md Faz 8'deki 4 kısıt maddesini mutlaka kapsa. S17 için feature importance grafiği, SHAP summary plot ve Sicaklik_TR için SHAP dependence plot üret; U-şekilli sıcaklık-tüketim ilişkisini yorumla. Çalıştır, doğrula, commit at.
```

## Komut 10 — Faz 9b: Senaryo B — Gerçek Gün-Öncesi Modeli

```
PLAN.md Faz 9b'yi uygula: yalnızca teklif anında bilinen özelliklerle Senaryo B veri setini kur — hedef lag'leri 24/48/168, 24 saat kaydırılmış rolling istatistikler, shift(24) piyasa değişkenleri, takvim/sin-cos/tatil/Ramazan bayrakları ve sıcaklık (hava tahmini vekili). Faz 7'nin en iyi parametreleriyle aynı model mimarisini eğit (yeniden arama yapma, nedenini not düş), X_val ve test'te tüm metriklerle değerlendir. Senaryo A ile yan yana kıyas tablosu oluştur ve gerçek GÖP zamanlamasını (teklifler D günü ~12:00'de, ufuk 13–35 saat) açıklayan, A→B performans düşüşünü bulgu olarak yorumlayan bir markdown yaz. Çalıştır, doğrula, commit at.
```

## Komut 11 — Faz 10: Finansal Katman (Alpha + Kantil Teklif)

```
PLAN.md Faz 10'u uygula: dengesizlik maliyet fonksiyonunu koru (pozitif min(PTF,SMF)×0.93, negatif max(PTF,SMF)×1.07). Validation saatlerinde Cu = 1.07·max(PTF,SMF) − PTF ve Co = PTF − 0.93·min(PTF,SMF) hesapla, τ* = mean(Cu)/(mean(Cu)+mean(Co)) değerini yazdır ve newsvendor mantığını (optimal teklif = tüketim dağılımının τ* kantili) markdown'da 3-4 cümleyle açıkla. LightGBM quantile modellerini (alpha = 0.10, 0.50, 0.90 ve τ*) her iki senaryonun özellik setiyle eğit. Her iki senaryo için üç teklif stratejisini hesapla: (a) ortalama tahmin, (b) validation'da scipy.optimize.minimize ile bulunan alpha çarpanlı tahmin, (c) optimal kantil (τ*) teklifi. Senaryo B için örnek bir haftanın P10–P90 bant grafiğini ve bant kapsama oranını raporla. Nihai tablo: Naif teklif (Lag_24h) + 2 senaryo × 3 strateji = 7 satır, toplam ceza (TL) ve naife göre tasarruf yüzdesi + bar grafik. Alpha ve τ*'ın yalnızca validation'dan geldiğini, testin strateji ayarında kullanılmadığını markdown'da vurgula; gerçekçi iş rakamının B-kantil satırı olduğunu yorumla. Çalıştır, doğrula, commit at.
```

## Komut 12 — Faz 11: Teslim Hazırlığı

```
PLAN.md Faz 11'i uygula: README.md'yi plandaki iskelete göre gerçek sonuç rakamlarıyla doldur (en iyi model, test metrikleri, senaryo × strateji finans tablosu). pip freeze ile requirements.txt sürümlerini sabitle. Notebook'u jupyter nbconvert --execute ile baştan sona temiz çalıştır ve tüm hücre çıktıları kayıtlı halde bırak. Son commit'i at. GitHub'da repo oluşturup push'la (gh CLI varsa "gh repo create" ile, yoksa bana remote ekleme adımlarını söyle).
```

## Komut 13 — Final Denetim (teslimden hemen önce)

```
PLAN.md bölüm 6'daki 17 maddelik kabul listesini VE bölüm 6b'deki farklılaştırıcı ekleri tek tek denetle: her madde için notebook'ta hangi bölümde/hücrede karşılandığını belirt, eksik veya zayıf olanları listele ve düzeltme öner. Ayrıca teslim kurallarını kontrol et: README, requirements.txt, veri dosyaları ve çalışır notebook repoda mı, hücre çıktıları kayıtlı mı?
```

---

## İpuçları

- Büyük fazlarda (özellikle 4, 9, 10, 11) komutun başına "Önce yapacaklarını maddeler halinde söyle, onayımdan sonra uygula." ekleyebilirsin.
- Bir şey ters giderse: "Son değişikliği geri al" veya "git diff ile ne değiştiğini göster" de.
- Komut 8'deki arama birkaç dakika sürebilir; Komut 11'deki kantil modeller hafiftir, hızlı biter.
- Sonuç tabloları çıktıkça (Komut 7, 9, 10, 11 sonrası) rakamları buraya getir, birlikte yorumlayalım.
