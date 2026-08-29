# Bisiklet karar tahtası

Göteborg'da iki kişi için ikinci el bisiklet araması — ölçülmüş özellikler, değer
tahmini ve puanlama. Sayfa: **https://kasimecer.github.io/bisiklet/**

## Ne var burada

`index.html` tek dosya: veri gömülü, dış bağımlılık yok, telefon ve masaüstü uyumlu.
Sütun başlığına tıklayınca sıralar, arama ve durum filtresi var, satıra tıklayınca
bütün alanlar açılır.

**Puan bir hüküm değil, göstergedir.** Ağırlıklı toplam; veri eksikse alt–üst
aralık gösterilir ("46–88" gibi), tam veri varsa "kesin" yazar.

`Değer` sütunu o ilanın piyasa tahminidir; `Fark = Değer − Fiyat`. Artı ise
ilan değerinin altında.

## Mahremiyet

Satıcı **isimleri ve ölçüm notları bu açık sürümde yoktur** — puan, yorum sayısı,
ilan sayısı gibi karar sinyalleri kalır, kimlik kalmaz. Üçüncü kişilerin
mahremiyeti bu projede feragat edilebilir bir şey değil.

Yorumlar tarayıcının kendi deposunda (localStorage) tutulur, sunucuya gitmez.

## Üretim

Sayfa elle yazılmaz, üretilir:

```bash
PUBLIC=1 python3 tools/build_page.py && python3 tools/render.py index.html
```

Kaynak veri ve puanlama modeli bu depoda değil.
