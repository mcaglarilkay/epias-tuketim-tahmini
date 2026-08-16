# EPİAŞ Elektrik Tüketimi Tahmini ve Dengesizlik Maliyeti Optimizasyonu

> Türkiye Yapay Zeka Akademisi — Makine Öğrenmesi Final Ödevi
>
> 📓 **Notebook'u çıktılarıyla görüntülemek için:** GitHub önizlemesi açılmazsa [nbviewer üzerinden görüntüleyin](https://nbviewer.org/github/mcaglarilkay/epias-tuketim-tahmini/blob/main/epias_final.ipynb) — tüm tablolar ve grafikler eksiksiz render edilir.

## Amaç

Türkiye'nin saatlik toplam elektrik tüketimini (MWh) tahmin etmek ve bu tahmini Gün Öncesi Piyasası teklifi olarak kullanıp EPİAŞ dengesizlik cezalarını en aza indirmek. Proje iki katmandan oluşur: makine öğrenmesi tahmin katmanı (5 model + naif kıyaslar + hiperparametre araması) ve tahminin maliyet etkisini **göreli endeksle** ölçen finansal simülasyon katmanı (dengesizlik cezası mekanizması + newsvendor-optimal teklif stratejileri; tüm sonuçlar naif teklif = 100 endeksiyle raporlanır, mutlak tutar verilmez).

## Veri

- Kaynak: EPİAŞ Şeffaflık Platformu — Gerçek Zamanlı Tüketim (GZT, hedef), Piyasa Takas Fiyatı (PTF), Sistem Marjinal Fiyatı (SMF), Sistem Yönü (SY); `;` ayraçlı CSV
- Ek kaynak: Open-Meteo Historical Archive — 6 şehrin (İstanbul, Ankara, İzmir, Bursa, Antalya, Adana) saatlik sıcaklığı, nüfus ağırlıklı ortalama (`data/weather.csv`, tek seferlik indirilip repoya kaydedildi; notebook internetsiz çalışır)
- Eğitim dönemi: 01.01.2025 – 31.12.2025 · Test dönemi: 01.01.2026 – 14.08.2026
- Hedef değişken: `Tüketim Miktarı(MWh)` — Problem türü: **Regresyon**

## Kurulum ve Çalıştırma

```bash
pip install -r requirements.txt
jupyter notebook epias_final.ipynb   # ardından: Kernel → Restart & Run All
```

Veri dosyaları repo içindedir (`data/train/`, `data/test/`, `data/weather.csv`); internet bağlantısı gerekmez.

## Yöntem Özeti

1. **Ön işleme:** tip dönüşümü, eksik değer kontrolü (saatlik interpolasyon hazır), Sistem Yönü one-hot, IQR aykırı değer analizi (tüketim pikleri gerçek talep — silinmez), ölçekleme yalnızca doğrusal model pipeline'ında
2. **Öznitelikler:** lag (1/2/24/168 s), rolling istatistikler, takvim + sin/cos, Türkiye resmi tatil ve Ramazan bayrakları, sıcaklık türevleri (HDD/CDD), **sızıntı düzeltmesi** (tüm piyasa değişkenleri ≥1 saat gecikmeli)
3. **Öznitelik seçimi:** hedefle korelasyon + LassoCV(TimeSeriesSplit) — doğrulama korumalı eleme (29 → 20 özellik)
4. **Modeller:** Ridge, Lasso (StandardScaler pipeline), RandomForest, XGBoost, LightGBM + 3 naif baseline (Lag 1/24/168 s) ve skill oranı
5. **Hiperparametre:** şampiyona (LightGBM) `RandomizedSearchCV(cv=TimeSeriesSplit(5), n_iter=30)`
6. **Senaryolar:** A = saat-öncesi (Lag_1h'li), B = gerçek gün-öncesi (yalnızca teklif anında bilinen özellikler)
7. **Finansal katman:** dengesizlik cezası simülasyonu; newsvendor τ* kantili ve alpha çarpanı (yalnızca validation'da ayarlanır); kantil LightGBM modelleri ve P10–P90 bandı

## Sonuçlar

**En iyi model:** RandomizedSearchCV ile ayarlanmış **LightGBM** (tüm 2025 ile yeniden eğitilip test'te değerlendirildi).

| Test (Oca–Ağu 2026) | MAE (MWh) | RMSE (MWh) | MAPE | R² | Skill (MAE/Lag24-naif) |
|---|---|---|---|---|---|
| Senaryo A — saat öncesi | 427 | 603 | %1,06 | 0,992 | 0,183 |
| Senaryo B — gün öncesi | 940 | 1.440 | %2,37 | 0,955 | 0,403 |

**Finansal simülasyon (test dönemi toplam dengesizlik maliyeti — göreli endeks, naif = 100):**

| Strateji | Maliyet Endeksi (Naif = 100) | Naife göre tasarruf |
|---|---|---|
| Naif teklif (dün aynı saat) | 100,0 | — |
| A — ortalama / alpha / kantil(τ*) | 19,4 / 19,0 / 20,3 | %80,6 / **%81,0** / %79,7 |
| B — ortalama / alpha / kantil(τ*) | 44,2 / 38,5 / 45,5 | %55,8 / **%61,5** / %54,5 |

Gerçekçi iş rakamı **B satırlarıdır** (gün-öncesi kurallarına uygun): en iyi strateji **B-alpha** ile maliyet endeksi 38,5'e iner (%61,5 tasarruf). A satırları modelin tavanını gösterir. τ* = 0,503 (validation'da tavan-fiyat saatlerinde Cu=Co simetrisi) olduğundan kantil strateji bu dönemde ek avantaj sağlamadı; tüm strateji parametreleri yalnızca validation'dan gelir, test hiçbir ayarda kullanılmamıştır.

## Kısıtlar

- **Raporlama ilkesi:** Maliyet sonuçları mutlak tutar olarak değil, naif teklif = 100 endeksiyle raporlanır; endeksler herhangi bir piyasa büyüklüğünü veya katılımcı maliyetini ölçmez. Çalışma eğitim amaçlıdır; teklif, işlem veya strateji tavsiyesi değildir.

- **Tahmin ufku:** Senaryo A "bir saat öncesi" tahmincisidir ve üst sınırdır; gerçek GÖP ufku 13–35 saattir (Senaryo B bunu modeller, tekdüze ≥24 saat sadeleştirmesiyle).
- **Sıcaklık vekili:** Gerçekleşen sıcaklık, teklif anındaki hava tahmininin vekili (proxy) olarak kullanılır; canlı sistemde meteorolojik tahmin verisi gerekir.
- **Veri dönemi:** Model 2025 ile eğitilmiştir; 2026'da fiyat rejimi belirgin kaydı (PTF ort. −%29). Fiyat özelliklerinin azaltılması modeli korudu, ancak tüketim tarafında yapısal kırılma olursa düzenli yeniden eğitim gerekir.
- **P10–P90 kapsaması** test'te %44 (hedef %80): kalibrasyonsuz kantiller drift altında dar kalır; üretimde conformal kalibrasyon önerilir.
