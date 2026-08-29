# Bisiklet karar tahtası — denetim raporu

Üretim: **2026-08-29** · kaynak `bikes.csv` + `extra.json` · model `score.py` v4.3
İnteraktif sürüm: https://kasimecer.github.io/bisiklet/ (o sayfanın gövdesini JavaScript
çizer; bu dosya JS çalıştırmayan okuyucular için üretilir)
Yapısal veri: [`data.json`](data.json) — aynı kayıtların tamamı, makine okunur

## Puan ne demek, ne demek değil

Puan **ağırlıklı toplamdır, hüküm değildir.** Kalemler 0–10 arası puanlanır, profil
ağırlığıyla çarpılır, mesafe ve bütçe cezaları düşülür.

- **Profil A** (160 cm, 5,1 km, dik yokuş, arkada 3 yaş çocuk):
  beden 18 · vites 16 · ekipman 13 · foto 12 · fren 10 · çocuk koltuğu 8 · bakım 7 ·
  fiyat 8 · güven 4 · durum 2 · marka 2
- **Profil B** (181 cm, 5,8 km, dönüşte kesintisiz tırmanış, spor + pendling):
  beden 20 · vites 18 · foto 14 · fren 12 · fiyat 10 · ekipman 8 · güven 7 ·
  marka 4 · durum 3 · bakım 4

**Aralık sütunu belirsizliktir.** Bir kalem bilinmiyorsa puan onsuz hesaplanır;
alt sınır "bilinmeyenlerin hepsi 0", üst sınır "hepsi 10" varsayar. `kesin` = tüm
kalemler dolu. Geniş aralıklı bir puan **ölçüm değil, cehalet** demektir.

**Fiyat ekseni ayrıdır.** Yaşa göre çapa YOKTUR — 2026-08-29'da iptal edildi, çünkü
fiyat ile kalite arasındaki regresyon **R² = 0,09** çıktı (fiyat kaliteyi %9 açıklıyor).
Yerine ilan başına `Değer` tahmini geçti; `Fark = Değer − Fiyat`.

**Otomatik eleme:** 3 vitesli her ilan (gerekçe: vites aralığı) · satıldı · vazgeçildi.

## Bilinen kısıtlar — denetlenecek noktalar

1. **`Gün` sütunu ilk yayın tarihi DEĞİL.** Blocket ilk yayın tarihini hiçbir yerde
   vermiyor; bu sayı *son düzenlemeden* beri geçen gündür ve satıcı ilanı güncellerse
   sıfırlanır. Tek yönlü sinyal: büyük sayı kesin bayat, küçük sayı taze olmayabilir.
2. **`Değer` bir ölçüm değil, CC'nin tahminidir** — yalnız fotoğrafı fiilen incelenmiş
   ilanlarda doludur, gerisinde boştur ve `Eksik` sütununda işaretlidir.
3. **Kimlik redaksiyonu uygulanmıştır.** Satıcı adları ve ölçüm notları bu açık
   sürümde yoktur; puan/yorum/ilan sayısı gibi karar sinyalleri durur. Üçüncü
   kişilerin mahremiyeti feragat edilebilir sayılmamıştır.
4. **Fotoğraf incelenmemiş ilanlarda veri kapsaması düşüktür**; bu ilanların puanı
   karşılaştırılabilir değildir, `Eksik` sütunu bunu açıkça yazar.

## Model sözlüğü — tanımlar

**Null konvansiyonu.** CSV'de **boş string = bilinmiyor**, kalem skordan çıkarılır ve
aralığı genişletir. **`0` gerçek bir sıfır puandır**, kalem skora girer. İkisi farklıdır.
`Vites = 0` bir istisnadır ve düzeltilecek (bkz. açık maddeler).

**Kesinlik üç değerlidir** (v5, 2026-08-29):
- `ölçüldü` — tüm ağırlıklı kalemler dolu **ve hiçbiri tahmin değil**
- `tahminli` — tüm kalemler dolu **ama en az biri tahmin** (ör. beden puanı var, cm yok)
- `alt–üst` — eksik kalem var; alt sınır "eksikler 0", üst sınır "eksikler 10"

`tahmin` sütunu hangi **puanlanan** kalemin ölçüm olmadığını sayar: beden (cm yok) ·
fren (dayanak yok) · vites (sayı yok) · ekipman/koltuk (fotoğraf incelenmemiş).

**Kovalar.** `aday` = vetoyu geçti + verisi yeterli · `veri-yetersiz` = fotoğraf
incelenmemiş **veya** aralık > 25 · `veto` = sert kısıt ihlali, **puan verilmez** ·
`elendi/satıldı/vazgeçildi` = arşiv.

**Veto kuralları** (sert, ceza değil): beden bandı A 44–52 cm, B 54–60 cm — **yalnız cm
biliniyorsa** uygulanır · bütçe > 5.000 kr · profil A'da ilan "bagaj yok" diyorsa
(arka çocuk koltuğu takılamaz).

**Eleme kuralları:** 3 vitesli her ilan (E'nin kararı, gerekçe vites aralığı) ·
satıldı · vazgeçildi.

**Sıralama:** varsayılan **aralık alt sınırı** (maximin). Nokta tahmini ayrı sütun.

**`Değer`in işletimsel tanımı:** bkz. denetim cevabı — bu sürümde tanım tartışmalıdır
ve rapora yazılmadan önce netleştirilecektir.

**`foto` kalemi belge kalitesidir**, bisikletin fiziksel durumu değil (o `durum`).
Ağırlığının skorda kalması denetimde sorgulandı; karar bekliyor.

**`Gün` skorda ödül vermez** — yalnız tek yönlü bayatlık cezası: >14g −2, >30g −4,
>45g −6.


## Profil A — E (160 cm, çocuk koltuğu)

8 canlı aday · 49 toplam kayıt · ortalama veri kapsaması **%97**
Arşiv: elendi 14 · satıldı 6 · vazgeçildi 4

### Ana kuyruk — vetoyu geçmiş, verisi yeterli (aralık ALT SINIRINA göre sıralı)

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 75.7 | 81.4 | 76–83 | [Merida Crossway damcykel 50cm](https://www.blocket.se/26132688) | 1 650 | 2 200 | +550 | Torslanda (15 km) | 0 | 50 | 21 | disk ön+arka (satıcı teyit etti) | 7 | bakim | — |
| 74.6 | 74.6 | tahminli | [Giant X-Sport 4.0](https://www.blocket.se/25660200) | 2 400 | 2 400 | +0 | Goteborg (8 km) | 3 | ? | 24 | ? | 8 | — | beden, fren |
| 72.0 | 72.0 | ölçüldü | [Monark Margareta Original](https://www.blocket.se/26007926) | 3 500 | 2 900 | -600 | Boras (65 km) | 1 | 51 | 7 | jant (el freni ön+arka) | 6 | — | — |
| 67.7 | 67.7 | tahminli | [Yosemite Silver Apron 50cm](https://www.blocket.se/24905256) | 1 500 | — | — | Goteborg (8 km) | 40 | 50 | 24 | ? | 9 | — | fren |
| 61.4 | 61.4 | tahminli | [Servad 7-navvaxlad lattmetall](https://www.blocket.se/25892547) | 3 000 | — | — | Goteborg-Spantorget (10 km) | 30 | ? | 7 | fotbroms | 4 | — | beden, fren |
| 61.1 | 69.0 | 61–71 | [Nishiki Trekking Master 46cm](https://www.blocket.se/24624215) | 3 500 | 2 400 | -1100 | Ojersjo (15 km) | 51 | 46 | 8 | ? | 7 | fren | — |
| 58.9 | 61.6 | 59–63 | [Apollo Limited (Sportson)](https://www.blocket.se/24910218) | 3 000 | — | — | bilinmiyor | 75 | ? | 18 | disk | ? | guven | beden |
| 55.3 | 58.0 | 55–59 | [Yosemite damcykel 28 (Biltema)](https://www.blocket.se/25438397) | 2 500 | — | — | Uddevalla (85 km) | 70 | 51 | 21 | ? | ? | guven | fren |

### Veri yetersiz — sıralamaya girmez

Fotoğrafı incelenmemiş **veya** belirsizlik aralığı 25 puandan geniş. Tek eylem: fotoğrafı incele ya da düşür.

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 54.5 | 79.2 | 54–86 | [Vermont Chester Wave 50cm](https://www.blocket.se/26039627) | 2 000 | 2 200 | +200 | Goteborg (10 km) | 0 | 50 | 21 | jant (V) | 8 | vites, koltuk, bakim | — |
| 45.9 | 55.5 | 46–62 | [Monark Karin damcykel 26](https://www.blocket.se/26036785) | 1 700 | — | — | Glommen (100 km) | 1 | ? | ? | fotbroms | ? | foto, guven, foto incelenmedi | beden, vites, ekipman, koltuk |
| 43.8 | 63.6 | 44–74 | [White AX+ 290 FF hybrid](https://www.blocket.se/23264797) | 2 800 | — | — | Molndal (15 km) | 22 | 51 | 24 | ? | 4 | foto, fren, koltuk, foto incelenmedi | ekipman |
| 42.4 | 55.7 | 42–64 | [Ronhill RHW5 S (+bagajlik)](https://www.blocket.se/25365906) | 2 100 | — | — | Langas (70 km) | 26 | ? | 21 | ? | 6 | foto, fren, foto incelenmedi | beden, ekipman, koltuk |
| 37.3 | 56.7 | 37–71 | [Crescent stadscykel 28 M svart](https://www.blocket.se/26028495) | 2 450 | — | — | Goteborg (5 km) | 1 | ? | 0 | fotbroms | ? | foto, fren, koltuk, guven, foto incelenmedi | beden, ekipman |
| 28.2 | 52.1 | 28–62 | [Rabeneick TS4 XS (146-165cm)](https://www.blocket.se/23076367) | 3 900 | — | — | Spanga (470 km) | 1 | ? | 10 | ? | ? | foto, fren, koltuk, guven, foto incelenmedi | beden, ekipman |
| 18.3 | 50.9 | 18–67 | [Crescent Holma (Kallered)](https://www.blocket.se/25987112) | 5 000 | — | — | Kallered (15 km) | 2 | ? | ? | hidrolik disk | 7 | beden, vites, koltuk, bakim | — |
| 17.1 | 55.7 | 17–74 | [White AX Series 28 8vxl (VALLENTUNA - cok uzak)](https://www.blocket.se/25833901) | 3 000 | 1 500 | -1500 | Vallentuna (470 km) | 0 | ? | 8 | ? | ? | vites, foto, fren, koltuk, bakim, guven, foto incelenmedi | beden, ekipman |
| 17.0 | 58.6 | 17–72 | [Crescent hybrid (Kungsangen)](https://www.blocket.se/22534571) | 3 500 | — | — | Kungsangen (450 km) | 22 | ? | 16 | hidrolik disk | ? | beden, ekipman, foto, koltuk, guven, foto incelenmedi | — |
| 16.8 | 55.2 | 17–74 | [Crescent hybrid lag insteg 51cm](https://www.blocket.se/25629680) | 3 900 | — | — | Huskvarna (150 km) | 0 | 51 | ? | ? | ? | vites, foto, fren, koltuk, bakim, guven, foto incelenmedi | ekipman |

### VETO — sert kısıt ihlali, puan verilmez

| Bisiklet | Fiyat | Yer | VETO SEBEBİ |
|---|---|---|---|
| [Crescent damcykel lattvikt](https://www.blocket.se/22875816) | 2 000 | Goteborg | ARKA BAGAJ YOK (cocuk koltugu takilamaz) |
| [Crescent Holma damhybrid 55cm (STOCKHOLM)](https://www.blocket.se/26044987) | 4 500 | Stockholm | BEDEN 55 cm bant disi (44-52) |
| [White SC LITE hybrid S](https://www.blocket.se/24938380) | 3 900 | Asa | ARKA BAGAJ YOK (cocuk koltugu takilamaz) |
| [Batavus Harlem E-Go (BATARYASIZ)](https://www.blocket.se/26089695) | 7 800 | Askim | BUTCE 7800 kr > 5000 |
| [Crescent hybrid M (Dalaro)](https://www.blocket.se/25541463) | 4 000 | Dalaro | ARKA BAGAJ YOK (cocuk koltugu takilamaz) |
| [Nishiki 422 damcykel 53cm](https://www.blocket.se/25653038) | 4 800 | Torslanda | BEDEN 53 cm bant disi (44-52) |
| [Hybridcykel 53cm unisex (Gbg)](https://www.blocket.se/23090187) | 3 000 | Goteborg | BEDEN 53 cm bant disi (44-52) |

### Satıcı teması

- **Merida Crossway damcykel 50cm** — satıcı cevapladı: arka disk var, yıl ~2017/18, bagajlık yükü bilinmiyor
- **Monark Margareta Original** — satıcı başka ilgilenen olduğunu bildirdi
- **Nishiki Trekking Master 46cm** — 2 soru soruldu — cevap yok

## Profil B — eş (181 cm, spor + pendling)

15 canlı aday · 58 toplam kayıt · ortalama veri kapsaması **%97**
Arşiv: elendi 7 · satıldı 1 · vazgeçildi 1

### Ana kuyruk — vetoyu geçmiş, verisi yeterli (aralık ALT SINIRINA göre sıralı)

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 81.2 | 81.2 | ölçüldü | [Trek Neko SL 57 cm](https://www.blocket.se/23479269) | 2 000 | 3 000 | +1000 | Bramhult (60 km) | 11 | 57 | 27 | disk (ön+arka) | 8 | — | — |
| 78.7 | 78.7 | ölçüldü | [Ortler Chur trekking 55cm (Deore 3x9)](https://www.blocket.se/25787860) | 4 500 | 3 100 | -1400 | Goteborg (10 km) | 12 | 55 | 27 | hidrolik disk (Shimano, 160mm) | 8 | — | — |
| 74.7 | 74.7 | ölçüldü | [Apollo Teamrace hybridcykel](https://www.blocket.se/25933358) | 4 000 | 3 000 | -1000 | bilinmiyor | 20 | L | 24 | hidrolik disk (YENİ) | 7 | — | — |
| 71.9 | 75.0 | 72–76 | [Ridgeback Flight 56cm](https://www.blocket.se/25825943) | 3 000 | 2 700 | -300 | Falkoping (120 km) | 3 | 56 | ? | disk (ön+arka) | 5 | bakim | vites |
| 71.7 | 71.7 | ölçüldü | [Crescent Hamra sport hybrid 58cm](https://www.blocket.se/25872191) | 2 700 | 2 600 | -100 | Ravlanda (40 km) | 5 | 58 | 24 | ? | 6 | — | — |
| 71.5 | 71.5 | tahminli | [Crescent Tarfek 55cm](https://www.blocket.se/25588679) | 1 600 | — | — | Hisings Backa (10 km) | 1 | 55 | ? | ? | 4 | — | fren, vites |
| 67.8 | 67.8 | tahminli | [Nishiki Cross Hybrid 522 XL](https://www.blocket.se/26081900) | 1 500 | — | — | Torslanda (12 km) | 0 | ? | 24 | ? | 7 | — | beden |
| 66.1 | 66.1 | tahminli | [Sjosala herrcykel 55cm 7vxl](https://www.blocket.se/25958388) | 2 000 | — | — | Ulricehamn (90 km) | 2 | 55 | 7 | ? | 5 | — | fren |
| 63.2 | 66.2 | 63–67 | [Merida Crossway Striker L 55cm](https://www.blocket.se/26036967) | 5 000 | 3 400 | -1600 | Torslanda (15 km) | 2 | 55 | 24 | disk ön+arka (hidrolik DOĞRULANMADI) | 7 | bakim | — |
| 61.9 | 67.0 | 62–69 | [Crescent Logic 175-185cm](https://www.blocket.se/23827068) | 3 400 | — | — | Goteborg (5 km) | 15 | ? | 21 | ? | ? | guven | beden, fren |
| 60.6 | 63.4 | 61–65 | [Scott Sportster 40 herr M](https://www.blocket.se/26035210) | 2 500 | 2 400 | -100 | Vaggeryd (145 km) | 2 | M | 24 | disk ön+arka (mekanik gibi) | 5 | bakim | — |
| 53.6 | 53.6 | tahminli | [Scott Sub Comfort 10 L 27vxl](https://www.blocket.se/25285342) | 5 000 | 2 200 | -2800 | Skovde (115 km) | 2 | ? | 27 | jant (V) — disk rotoru hiçbir fotoda yok | 7 | — | beden |
| 51.8 | 64.4 | 52–70 | [Crescent Holma 24vxl 58cm UCUZ](https://www.blocket.se/24681329) | 1 800 | — | — | Jonkoping (150 km) | 0 | 58 | 24 | jant (V) | 3 | vites | — |
| 50.4 | 55.9 | 50–57 | [Crescent Holma damhybrid 55cm (STOCKHOLM)](https://www.blocket.se/26044987) | 4 500 | — | — | Stockholm (470 km) | 0 | 55 | 16 | disk | ? | guven | — |
| 47.5 | 50.0 | 48–52 | [Specialized hybrid M (Boras) Tourney](https://www.blocket.se/25372010) | 4 000 | 2 200 | -1800 | Boras (65 km) | 23 | ? | ? | mekanik disk (ön+arka) | 5 | bakim | beden, vites |

### Veri yetersiz — sıralamaya girmez

Fotoğrafı incelenmemiş **veya** belirsizlik aralığı 25 puandan geniş. Tek eylem: fotoğrafı incele ya da düşür.

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 59.2 | 69.7 | 59–73 | [Crescent hybrid 56cm 2017 Deore](https://www.blocket.se/25343212) | 1 900 | — | — | Vanersborg (85 km) | 24 | 56 | ? | jant (V) | 5 | foto, foto incelenmedi | vites, ekipman |
| 51.4 | 66.3 | 51–72 | [White SC Lite FF Ane M (Lerum)](https://www.blocket.se/24835242) | 1 700 | — | — | Lerum (20 km) | 43 | ? | 18 | hidrolik disk | ? | foto, guven, foto incelenmedi | beden, ekipman |
| 50.1 | 68.0 | 50–76 | [Scott Sportser Hybrid](https://www.blocket.se/26014750) | 1 400 | — | — | Lerum (20 km) | 1 | ? | 30 | ? | 4 | foto, fren, foto incelenmedi | beden, ekipman |
| 49.7 | 67.3 | 50–76 | [Crescent Niak hybrid 28 M](https://www.blocket.se/25956032) | 2 700 | — | — | V.Frolunda (10 km) | 2 | ? | ? | ? | 5 | foto, fren, foto incelenmedi | beden, vites, ekipman |
| 49.1 | 66.5 | 49–75 | [Crescent stadscykel 28 M](https://www.blocket.se/26007221) | 1 999 | — | — | Goteborg (8 km) | 1 | ? | 24 | ? | 4 | foto, fren, foto incelenmedi | beden, ekipman |
| 49.0 | 65.2 | 49–70 | [Crescent Sport Cross L](https://www.blocket.se/25576096) | 3 345 | — | — | Karlstad (250 km) | 6 | ? | 24 | disk | ? | foto, guven, foto incelenmedi | beden, ekipman |
| 48.8 | 62.1 | 49–70 | [Peak stadscykel 55cm](https://www.blocket.se/25995269) | 1 550 | — | — | Vallda (25 km) | 4 | 55 | ? | ? | ? | foto, guven, foto incelenmedi | vites, ekipman, koltuk |
| 43.8 | 67.3 | 44–77 | [VeloCity ARL FF EQ](https://www.blocket.se/26027087) | 2 000 | — | — | Falkenberg (100 km) | 2 | ? | 24 | hidrolik disk | ? | foto, fren, guven, foto incelenmedi | beden, ekipman |
| 43.1 | 68.0 | 43–77 | [White Hybridcykel 28 (beden yok)](https://www.blocket.se/25332612) | 1 750 | — | — | Trollhattan (80 km) | 25 | ? | 24 | hidrolik disk | 3 | beden, foto, foto incelenmedi | ekipman |
| 40.4 | 60.6 | 40–73 | [Focus Lost Lagoon](https://www.blocket.se/26036990) | 2 750 | — | — | Lindome (15 km) | 1 | L | ? | ? | ? | foto, fren, guven, foto incelenmedi | vites, ekipman |
| 38.9 | 59.7 | 39–72 | [Specialized Vita (lag insteg)](https://www.blocket.se/25961988) | 3 490 | — | — | Molndal (10 km) | 3 | ? | 24 | hidrolik disk | ? | foto, ekipman, guven, bakim, foto incelenmedi | beden |
| 37.8 | 57.0 | 38–71 | [Peak Hybrid](https://www.blocket.se/26016509) | 1 500 | — | — | Kungsbacka (30 km) | 1 | ? | 21 | ? | ? | foto, fren, guven, foto incelenmedi | beden, ekipman |
| 36.9 | 64.2 | 37–76 | [Aspenas Sotenas XL 58cm](https://www.blocket.se/25802935) | 3 000 | — | — | Jonkoping (150 km) | 9 | 58 | ? | jant (V) | ? | vites, foto, guven, foto incelenmedi | ekipman |
| 35.9 | 61.0 | 36–77 | [Nakamura hybridcykel](https://www.blocket.se/22122600) | 2 699 | — | — | Goteborg (8 km) | 13 | ? | 8 | mekanik disk | ? | beden, foto, guven, foto incelenmedi | ekipman |
| 35.2 | 62.4 | 35–78 | [Crescent hybrid M (Lindome)](https://www.blocket.se/24203331) | 2 000 | — | — | Lindome (20 km) | 14 | ? | ? | jant (V) | ? | vites, foto, guven, bakim, foto incelenmedi | beden, ekipman |
| 35.0 | 63.0 | 35–75 | [Merida Mats LX (MTB)](https://www.blocket.se/22282091) | 2 000 | — | — | Simlangsdalen (120 km) | 17 | ? | ? | ? | ? | foto, fren, guven, durum, bakim, foto incelenmedi | beden, vites, ekipman |
| 35.0 | 44.4 | 35–56 | [Merida Redwood (MTB kat.)](https://www.blocket.se/26041299) | 2 900 | — | — | Goteborg (8 km) | 0 | ? | ? | ? | ? | foto, guven, foto incelenmedi | beden, vites, ekipman |
| 34.7 | 45.6 | 35–56 | [Nishiki City Cross XL (koleksiyon)](https://www.blocket.se/24599946) | 2 500 | — | — | Goteborg (8 km) | 51 | ? | ? | ? | ? | foto, guven, foto incelenmedi | beden, vites, ekipman |
| 31.9 | 47.8 | 32–65 | [Btwin Riverside 500](https://www.blocket.se/25972194) | 1 800 | — | — | Goteborg (8 km) | 2 | ? | 9 | ? | ? | foto, fren, guven, foto incelenmedi | beden, ekipman |
| 31.6 | 48.3 | 32–65 | [Trek Navigator T300](https://www.blocket.se/25499155) | 2 500 | — | — | Goteborg (8 km) | 19 | 60 | 24 | ? | ? | foto, fren, guven, foto incelenmedi | ekipman |
| 24.7 | 60.3 | 25–80 | [Scott sportster tour M UCUZ](https://www.blocket.se/25944341) | 1 500 | — | — | Bottnaryd (110 km) | 0 | ? | ? | ? | ? | vites, foto, fren, guven, bakim, foto incelenmedi | beden, ekipman |
| 24.5 | 65.4 | 24–83 | [Specialized sport disc M UCUZ](https://www.blocket.se/25909593) | 1 600 | — | — | Falkoping (100 km) | 0 | ? | 27 | disk | ? | vites, foto, fren, ekipman, guven, foto incelenmedi | beden |
| 24.0 | 49.4 | 24–75 | [Apollo sport XL](https://www.blocket.se/26081977) | 2 400 | — | — | Goteborg (8 km) | 0 | ? | ? | ? | ? | vites, foto, fren, guven, foto incelenmedi | beden, ekipman |
| 23.0 | 51.5 | 23–74 | [Scott hybrid S dam (KUCUK)](https://www.blocket.se/25256218) | 2 500 | — | — | Kinnarp (110 km) | 0 | ? | ? | disk | ? | vites, foto, ekipman, guven, bakim, foto incelenmedi | beden |
| 19.9 | 50.1 | 20–65 | [Crescent Kebne 16vxl 55cm](https://www.blocket.se/25323134) | 3 500 | — | — | Vasteras (350 km) | 24 | 55 | 16 | ? | ? | foto, fren, ekipman, guven, bakim, foto incelenmedi | — |
| 18.6 | 59.3 | 19–82 | [Merida Crossway Woodland M](https://www.blocket.se/26020702) | 2 200 | — | — | Habo (130 km) | 1 | ? | ? | ? | ? | vites, foto, fren, ekipman, guven, bakim, foto incelenmedi | beden |
| 15.3 | 47.1 | 15–78 | [Bianchi hybrid M](https://www.blocket.se/25787720) | 3 500 | — | — | V.Frolunda (10 km) | 9 | ? | ? | ? | ? | vites, foto, fren, ekipman, guven, bakim, foto incelenmedi | beden |
| 13.6 | 51.1 | 14–77 | [Crescent herrcykel 59cm](https://www.blocket.se/23605437) | 3 500 | — | — | Vanersborg (85 km) | 30 | 59 | ? | ? | ? | vites, foto, fren, ekipman, guven, bakim, foto incelenmedi | — |
| 8.5 | 38.4 | 8–64 | [Trek Dual Sport 2 M 18vxl](https://www.blocket.se/21927297) | 5 000 | — | — | Grabo (25 km) | 0 | ? | 18 | ? | ? | vites, foto, fren, guven, bakim, foto incelenmedi | beden, ekipman |

### VETO — sert kısıt ihlali, puan verilmez

| Bisiklet | Fiyat | Yer | VETO SEBEBİ |
|---|---|---|---|
| [Ghost Cross 1800 pendlarcykel](https://www.blocket.se/26045334) | 2 450 | Lindome | BEDEN 53 cm bant disi (54-60) |
| [Hybrid cykel 28 (Yosemite/Biltema)](https://www.blocket.se/25671871) | 1 990 | bilinmiyor | BEDEN 53 cm bant disi (54-60) |
| [Skeppshult Herr ARC 7-vxl 53cm](https://www.blocket.se/25083327) | 1 200 | Ulricehamn | BEDEN 53 cm bant disi (54-60) |
| [Crescent Angso](https://www.blocket.se/24997765) | 3 000 | Goteborg | BEDEN 61 cm bant disi (54-60) |
| [Hybridcykel 53cm unisex (Gbg)](https://www.blocket.se/23090187) | 3 000 | Goteborg | BEDEN 53 cm bant disi (54-60) |

### Satıcı teması

- **Merida Crossway Striker L 55cm** — temas kurulmadı — model yılı sorulmalı

## Denetçiye açık sorular

- Ağırlıklar gerekçelendirilmiş mi, yoksa keyfi mi? (A'da beden 18 vs marka 2)
- `Değer` tahminleri tutarlı mı — aynı sınıf iki bisiklete farklı mantık uygulanmış mı?
- Geniş aralıklı puanların listede yüksek sırada durması yanıltıcı mı?
- Eleme kuralları (3 vites) fazla mı katı?
