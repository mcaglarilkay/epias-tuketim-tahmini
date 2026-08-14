# EPİAŞ Elektrik Tüketimi Tahmini ve Dengesizlik Maliyeti Optimizasyonu

> Türkiye Yapay Zeka Akademisi — Makine Öğrenmesi Final Ödevi
>
> Not: Bu README, Faz 11'de gerçek sonuç rakamlarıyla doldurulacaktır. `TODO` işaretli yerler geçicidir.

## Amaç

Türkiye'nin bir sonraki saatteki toplam elektrik tüketimini (MWh) tahmin etmek ve bu tahmini gün öncesi piyasa taahhüdü olarak kullanıp EPİAŞ dengesizlik cezalarını minimize etmek. Proje iki katmandan oluşur: makine öğrenmesi tahmin katmanı ve tahminin parasal etkisini ölçen finansal simülasyon katmanı.

## Veri

- Kaynak: EPİAŞ Şeffaflık Platformu — Gerçek Zamanlı Tüketim (GZT), Piyasa Takas Fiyatı (PTF), Sistem Marjinal Fiyatı (SMF), Sistem Yönü (SY) + Open-Meteo saatlik sıcaklık (6 şehir, nüfus ağırlıklı ortalama)
- Eğitim dönemi: 01.01.2025 – 31.12.2025 · Test dönemi: 01.01.2026 – 21.07.2026
- Hedef değişken: `Tüketim Miktarı(MWh)` — Problem türü: **Regresyon**

## Kurulum ve Çalıştırma

```bash
pip install -r requirements.txt
jupyter notebook epias_final.ipynb   # ardından: Kernel → Restart & Run All
```

- EPİAŞ CSV'leri `data/train/` ve `data/test/` klasörlerine konmalıdır (klasörlerin içindeki README'lere bakın).
- `data/weather.csv` Faz 4b'de bir kez üretilir ve repoya dahil edilir; notebook internetsiz çalışır.

## Yöntem Özeti

TODO: Öznitelikler (lag, takvim, tatil, sıcaklık, gecikmeli piyasa verileri) → 5 model (Ridge, Lasso, Random Forest, XGBoost, LightGBM) → RandomizedSearchCV + TimeSeriesSplit → test değerlendirmesi → dengesizlik maliyeti simülasyonu ve alpha optimizasyonu.

## Sonuçlar

TODO: En iyi model, test metrikleri (MAE, MSE, RMSE, R², MAPE) ve finansal tasarruf (TL ve %) — Faz 8 ve Faz 10 tamamlanınca doldurulacak.

## Kısıtlar

TODO: Bir saat öncesi tahmin ufku vs gerçek gün öncesi piyasası, gerçekleşen sıcaklığın hava tahmini vekili olarak kullanımı, veri döneminin kapsamı — Faz 8'de detaylandırılacak.
