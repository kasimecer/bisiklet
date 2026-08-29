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

**Arşiv kalıcıdır** (E kararı, 2026-08-29). Elenen · vetolanan · satılan · vazgeçilen
hiçbir kayıt silinmez. Her arşiv kaydı taşır: `arsiv_kod` (VETO / ELENDI / SATILDI /
VAZGECILDI) · `arsiv_sebep` · `arsiv_tarih` · satıldıysa **`son_fiyat` (zorunlu)**.
Satılan ilanların fiyatı `Değer` modelinin piyasa referansı olacak. `validate.py`
kayıt sayısını bir yüksek-su işaretinde tutar; sayı düşerse **hata verir**.

**Veri kapsaması TÜM kayıtlar üzerinden** hesaplanır, ana kuyruk üzerinden değil.

**`Gün` skorda ödül vermez** — yalnız tek yönlü bayatlık cezası: >14g −2, >30g −4,
>45g −6.


## Profil A — E (160 cm, çocuk koltuğu)

6 canlı aday · 49 toplam kayıt · veri kapsaması (TÜM kayıtlar) **%74**
Arşiv: ALINACAK 1 · elendi 14 · satıldı 6 · vazgeçildi 4

### Ana kuyruk — vetoyu geçmiş, verisi yeterli (aralık ALT SINIRINA göre sıralı)

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 63.2 | 69.2 | 63–71 | [Monark Margareta Original](https://www.blocket.se/26007926) | 3 500 | 2 900 | -600 | Boras (65 km) | 1 | 51 | 7 | jant (el freni ön+arka) | 6 | agirlik | — |
| 62.0 | 73.9 | 62–82 | [Giant X-Sport 4.0](https://www.blocket.se/25660200) | 2 400 | 2 400 | +0 | Goteborg (8 km) | 3 | ? | 24 | ? | 8 | agirlik | beden, fren |
| 51.5 | 69.8 | 52–74 | [Nishiki Trekking Master 46cm](https://www.blocket.se/24624215) | 3 500 | 2 400 | -1100 | Ojersjo (15 km) | 51 | 46 | 8 | ? | ? | fren, agirlik, guven | — |
| 50.2 | 63.9 | 50–74 | [Crescent damcykel lattvikt](https://www.blocket.se/22875816) | 2 000 | — | — | Goteborg (8 km) | 6 | ? | 9 | fotbroms | ? | agirlik, guven | beden, fren |
| 48.3 | 60.0 | 48–68 | [Apollo Limited (Sportson)](https://www.blocket.se/24910218) | 3 000 | — | — | bilinmiyor | 75 | ? | 18 | disk | ? | agirlik, guven | beden |
| 48.3 | 59.2 | 48–68 | [Servad 7-navvaxlad lattmetall](https://www.blocket.se/25892547) | 3 000 | — | — | Goteborg-Spantorget (10 km) | 30 | ? | 7 | fotbroms | 4 | agirlik | beden, fren |

### Veri yetersiz — sıralamaya girmez

Fotoğrafı incelenmemiş **veya** belirsizlik aralığı 25 puandan geniş. Tek eylem: fotoğrafı incele ya da düşür.

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 48.5 | 81.1 | 48–88 | [Vermont Chester Wave 50cm](https://www.blocket.se/26039627) | 2 000 | 2 200 | +200 | Goteborg (10 km) | 0 | 50 | 21 | jant (V) | 8 | vites, koltuk, agirlik, bakim | — |
| 42.9 | 63.5 | 43–76 | [White AX+ 290 FF hybrid](https://www.blocket.se/23264797) | 2 800 | — | — | Molndal (15 km) | 22 | 51 | 24 | ? | 4 | fren, koltuk, agirlik, foto incelenmedi | ekipman |
| 36.3 | 55.0 | 36–72 | [Monark Karin damcykel 26](https://www.blocket.se/26036785) | 1 700 | — | — | Glommen (100 km) | 1 | ? | ? | fotbroms | ? | agirlik, guven, foto incelenmedi | beden, vites, ekipman, koltuk |
| 35.9 | 55.6 | 36–71 | [Ronhill RHW5 S (+bagajlik)](https://www.blocket.se/25365906) | 2 100 | — | — | Langas (70 km) | 26 | ? | 21 | ? | 6 | fren, agirlik, foto incelenmedi | beden, ekipman, koltuk |
| 32.4 | 56.7 | 32–75 | [Crescent stadscykel 28 M svart](https://www.blocket.se/26028495) | 2 450 | — | — | Goteborg (5 km) | 1 | ? | 0 | fotbroms | ? | fren, koltuk, agirlik, guven, foto incelenmedi | beden, ekipman |
| 19.5 | 51.5 | 20–76 | [White SC LITE hybrid S](https://www.blocket.se/24938380) | 3 900 | — | — | Asa (45 km) | 0 | ? | ? | ? | ? | vites, fren, agirlik, guven, foto incelenmedi | beden, ekipman, koltuk |
| 15.4 | 55.4 | 15–76 | [Crescent hybrid lag insteg 51cm](https://www.blocket.se/25629680) | 3 900 | — | — | Huskvarna (150 km) | 0 | 51 | ? | ? | ? | vites, fren, koltuk, agirlik, bakim, guven, foto incelenmedi | ekipman |
| 15.3 | 59.8 | 15–74 | [Crescent Holma (Kallered)](https://www.blocket.se/25987112) | 5 000 | — | — | Kallered (15 km) | 2 | ? | ? | hidrolik disk | 7 | beden, vites, koltuk, agirlik, bakim | — |
| 11.1 | 49.8 | 11–80 | [Hybridcykel 53cm unisex (Gbg)](https://www.blocket.se/23090187) | 3 000 | — | — | Goteborg (8 km) | 47 | 53 | ? | ? | ? | vites, ekipman, fren, koltuk, agirlik, bakim, guven, foto incelenmedi | — |
| 10.6 | 40.7 | 11–66 | [Nishiki 422 damcykel 53cm](https://www.blocket.se/25653038) | 4 800 | — | — | Torslanda (15 km) | 14 | 53 | ? | ? | ? | vites, fren, agirlik, bakim, guven, foto incelenmedi | ekipman, koltuk |

### Arşiv — KALICI, silinmez

Elenen · vetolanan · satılan · vazgeçilen hiçbir kayıt silinmez; hiçbir temizlik, dedup veya bakım işleminde de. Satılan ilanların **son fiyatı `Değer` modelinin piyasa referansıdır**, bu yüzden fiyat alanı zorunludur.

| Kod | Tarih | Bisiklet | Son fiyat | Sebep |
|---|---|---|---|---|
| ALINACAK | 2026-08-29 | [Merida Crossway damcykel 50cm](https://www.blocket.se/26132688) | — | E alma karari verdi - islem HENUZ TAMAMLANMADI |
| ELENDI | 2026-08-26 | [KING damcykel 51cm](https://www.blocket.se/25960900) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Winther Black 5 52cm](https://www.blocket.se/26039594) | — | elle elendi |
| ELENDI | 2026-08-29 | [Stabil Crescent Damcykel](https://www.blocket.se/25992013) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Monark Karin Original 28 51cm](https://www.blocket.se/25689962) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Crescent stadscykel 48cm 21vxl](https://www.blocket.se/26045095) | — | elle elendi |
| ELENDI | 2026-08-29 | [Crescent City Sirius 2013](https://www.blocket.se/25504084) | — | elle elendi |
| ELENDI | 2026-08-29 | [Diplomat nostalgia damcykel](https://www.blocket.se/26027737) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Sjosala Lysekil 28](https://www.blocket.se/25230804) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Sjosala Mariedal 28 (ELENDI)](https://www.blocket.se/25905620) | — | elle elendi |
| ELENDI | 2026-08-29 | [Skanstull Melissa damcykel](https://www.blocket.se/25406390) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Occano J5 24 (ELENDI)](https://www.blocket.se/26011657) | — | elle elendi |
| ELENDI | 2026-08-29 | [Crescent tjejcykel 24 inc (ELENDI)](https://www.blocket.se/26053297) | — | elle elendi |
| ELENDI | 2026-08-29 | [Cykel Savedalen (ELENDI)](https://www.blocket.se/23330598) | — | elle elendi |
| ELENDI | 2026-08-29 | [Neobike 16 katlanabilir (ELENDI)](https://www.blocket.se/24088407) | — | elle elendi |
| SATILDI | 2026-08-26 | [Crescent 51cm vit dam (SATILDI)](https://www.blocket.se/25212439) | 3 000 | ilan satildi |
| SATILDI | 2026-08-25 | [Crescent dam stadscykel](https://www.blocket.se/25968826) | 2 700 | ilan satildi |
| SATILDI | 2026-08-26 | [Sjosala Carmencita](https://www.blocket.se/24556817) | 2 200 | ilan satildi |
| SATILDI | 2026-08-29 | [Crescent FEMTO (Varberg, beden bekleniyor)](https://www.blocket.se/26068818) | 2 000 | ilan satildi |
| SATILDI | 2026-08-27 | [Skeppshult stadscykel (SATILDI)](https://www.blocket.se/26053266) | 1 800 | ilan satildi |
| SATILDI | 2026-08-26 | [Skeppshult 7vxl (SATILDI)](https://www.blocket.se/25972979) | 1 500 | ilan satildi |
| VAZGECILDI | 2026-08-29 | [Crescent STREET STC600 51cm](https://www.blocket.se/23727828) | — | E vazgecti |
| VAZGECILDI | 2026-08-29 | [Apollo PRO 28 tjej-unisex](https://www.blocket.se/25965230) | — | E vazgecti |
| VAZGECILDI | 2026-08-29 | [Damcykel Yosemite](https://www.blocket.se/25992054) | — | E vazgecti |
| VAZGECILDI | 2026-08-29 | [Skeppshult stadscykel 26](https://www.blocket.se/25868007) | — | E vazgecti |

### VETO — sert kısıt ihlali, puan verilmez

| Bisiklet | Fiyat | Yer | VETO SEBEBİ |
|---|---|---|---|
| [Yosemite Silver Apron 50cm](https://www.blocket.se/24905256) | 1 500 | Goteborg | KARA LISTE: Yosemite/Biltema — guvenlik (Rad & Ron testi) |
| [Yosemite damcykel 28 (Biltema)](https://www.blocket.se/25438397) | 2 500 | Uddevalla | KARA LISTE: Yosemite/Biltema — guvenlik (Rad & Ron testi) |
| [Crescent Holma damhybrid 55cm (STOCKHOLM)](https://www.blocket.se/26044987) | 4 500 | Stockholm | MESAFE 470 km > 200 (ayni gun gidilemez) |
| [Rabeneick TS4 XS (146-165cm)](https://www.blocket.se/23076367) | 3 900 | Spanga | MESAFE 470 km > 200 (ayni gun gidilemez) |
| [Crescent hybrid (Kungsangen)](https://www.blocket.se/22534571) | 3 500 | Kungsangen | MESAFE 450 km > 200 (ayni gun gidilemez) |
| [Batavus Harlem E-Go (BATARYASIZ)](https://www.blocket.se/26089695) | 7 800 | Askim | BUTCE 7800 kr > 6000 |
| [White AX Series 28 8vxl (VALLENTUNA - cok uzak)](https://www.blocket.se/25833901) | 3 000 | Vallentuna | MESAFE 470 km > 200 (ayni gun gidilemez) |
| [Crescent hybrid M (Dalaro)](https://www.blocket.se/25541463) | 4 000 | Dalaro | MESAFE 490 km > 200 (ayni gun gidilemez) |

### Satıcı teması

- **Monark Margareta Original** — satıcı başka ilgilenen olduğunu bildirdi
- **Nishiki Trekking Master 46cm** — 2 soru soruldu — cevap yok

## Profil B — eş (181 cm, spor + pendling)

9 canlı aday · 59 toplam kayıt · veri kapsaması (TÜM kayıtlar) **%74**
Arşiv: ALINACAK 1 · elendi 7 · satıldı 1 · vazgeçildi 1

### Ana kuyruk — vetoyu geçmiş, verisi yeterli (aralık ALT SINIRINA göre sıralı)

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 76.6 | 81.7 | 77–83 | [Trek Neko SL 57 cm](https://www.blocket.se/23479269) | 2 000 | 3 000 | +1000 | Bramhult (60 km) | 11 | 57 | 27 | disk (ön+arka) | 8 | agirlik | — |
| 68.9 | 73.7 | 69–75 | [Apollo Teamrace hybridcykel](https://www.blocket.se/25933358) | 4 000 | 3 000 | -1000 | bilinmiyor | 20 | L | 24 | hidrolik disk (YENİ) | 7 | agirlik | — |
| 67.9 | 72.3 | 68–74 | [Crescent Hamra sport hybrid 58cm](https://www.blocket.se/25872191) | 2 700 | 2 600 | -100 | Ravlanda (40 km) | 5 | 58 | 24 | ? | 6 | agirlik | — |
| 67.6 | 77.8 | 68–81 | [Ghost Cross 1800 pendlarcykel](https://www.blocket.se/26045334) | 2 450 | 3 000 | +550 | Lindome (15 km) | 0 | 53 | 27 | hidrolik disk (Tektro) | ? | guven, agirlik | — |
| 63.7 | 75.7 | 64–82 | [Ridgeback Flight 56cm](https://www.blocket.se/25825943) | 3 000 | 2 700 | -300 | Falkoping (120 km) | 3 | 56 | ? | disk (ön+arka) | 5 | agirlik, bakim | vites |
| 61.6 | 69.5 | 62–72 | [Merida Crossway Striker L 55cm](https://www.blocket.se/26036967) | 5 000 | 3 400 | -1600 | Torslanda (15 km) | 2 | 55 | 24 | disk ön+arka (hidrolik DOĞRULANMADI) | 7 | agirlik, bakim | — |
| 57.6 | 68.5 | 58–77 | [Crescent Tarfek 55cm](https://www.blocket.se/25588679) | 1 600 | — | — | Hisings Backa (10 km) | 1 | 55 | ? | ? | 4 | agirlik | fren, vites |
| 56.9 | 69.2 | 57–76 | [Sjosala herrcykel 55cm 7vxl](https://www.blocket.se/25958388) | 2 000 | — | — | Ulricehamn (90 km) | 2 | 55 | 7 | ? | ? | guven, agirlik | fren |
| 53.7 | 66.9 | 54–76 | [Nishiki Cross Hybrid 522 XL](https://www.blocket.se/26081900) | 1 500 | — | — | Torslanda (12 km) | 0 | ? | 24 | ? | ? | guven, agirlik | beden |

### Veri yetersiz — sıralamaya girmez

Fotoğrafı incelenmemiş **veya** belirsizlik aralığı 25 puandan geniş. Tek eylem: fotoğrafı incele ya da düşür.

| Alt sınır | Puan | Kesinlik | Bisiklet | Fiyat | Değer | Fark | Yer | Gün | Beden | Vites | Fren | Güven | Eksik puanlanan | Tahmin olan |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 59.2 | 69.7 | 59–77 | [Crescent hybrid 56cm 2017 Deore](https://www.blocket.se/25343212) | 1 900 | — | — | Vanersborg (85 km) | 24 | 56 | ? | jant (V) | 5 | agirlik, foto incelenmedi | vites, ekipman |
| 52.5 | 69.4 | 52–78 | [Crescent Logic 175-185cm](https://www.blocket.se/23827068) | 3 400 | — | — | Goteborg (5 km) | 15 | ? | 21 | ? | ? | guven, agirlik | beden, fren |
| 50.6 | 66.2 | 51–76 | [White SC Lite FF Ane M (Lerum)](https://www.blocket.se/24835242) | 1 700 | — | — | Lerum (20 km) | 43 | ? | 18 | hidrolik disk | ? | guven, agirlik, foto incelenmedi | beden, ekipman |
| 47.4 | 66.1 | 47–73 | [Crescent Holma 24vxl 58cm UCUZ](https://www.blocket.se/24681329) | 1 800 | — | — | Jonkoping (150 km) | 0 | 58 | 24 | jant (V) | 3 | vites, agirlik | — |
| 47.4 | 61.6 | 47–72 | [Peak stadscykel 55cm](https://www.blocket.se/25995269) | 1 550 | — | — | Vallda (25 km) | 4 | 55 | ? | ? | ? | guven, agirlik, foto incelenmedi | vites, ekipman, koltuk |
| 47.1 | 67.1 | 47–80 | [Scott Sportser Hybrid](https://www.blocket.se/26014750) | 1 400 | — | — | Lerum (20 km) | 1 | ? | 30 | ? | 4 | fren, agirlik, foto incelenmedi | beden, ekipman |
| 46.7 | 66.5 | 47–80 | [Crescent stadscykel 28 M](https://www.blocket.se/26007221) | 1 999 | — | — | Goteborg (8 km) | 1 | ? | 24 | ? | 4 | fren, agirlik, foto incelenmedi | beden, ekipman |
| 45.5 | 68.0 | 46–78 | [White Hybridcykel 28 (beden yok)](https://www.blocket.se/25332612) | 1 750 | — | — | Trollhattan (80 km) | 25 | ? | 24 | hidrolik disk | 3 | beden, agirlik, foto incelenmedi | ekipman |
| 41.9 | 67.7 | 42–81 | [VeloCity ARL FF EQ](https://www.blocket.se/26027087) | 2 000 | — | — | Falkenberg (100 km) | 2 | ? | 24 | hidrolik disk | ? | fren, guven, agirlik, foto incelenmedi | beden, ekipman |
| 40.2 | 69.5 | 40–88 | [Crescent Niak hybrid 28 M](https://www.blocket.se/25956032) | 2 700 | — | — | V.Frolunda (10 km) | 2 | ? | ? | ? | ? | fren, guven, agirlik, foto incelenmedi | beden, vites, ekipman |
| 39.5 | 64.8 | 40–76 | [Aspenas Sotenas XL 58cm](https://www.blocket.se/25802935) | 3 000 | — | — | Jonkoping (150 km) | 9 | 58 | ? | jant (V) | ? | vites, guven, agirlik, foto incelenmedi | ekipman |
| 37.8 | 60.3 | 38–77 | [Focus Lost Lagoon](https://www.blocket.se/26036990) | 2 750 | — | — | Lindome (15 km) | 1 | L | ? | ? | ? | fren, guven, agirlik, foto incelenmedi | vites, ekipman |
| 37.7 | 61.2 | 38–77 | [Nakamura hybridcykel](https://www.blocket.se/22122600) | 2 699 | — | — | Goteborg (8 km) | 13 | ? | 8 | mekanik disk | ? | beden, guven, agirlik, foto incelenmedi | ekipman |
| 34.2 | 56.1 | 34–74 | [Peak Hybrid](https://www.blocket.se/26016509) | 1 500 | — | — | Kungsbacka (30 km) | 1 | ? | 21 | ? | ? | fren, guven, agirlik, foto incelenmedi | beden, ekipman |
| 32.5 | 62.1 | 32–82 | [Crescent hybrid M (Lindome)](https://www.blocket.se/24203331) | 2 000 | — | — | Lindome (20 km) | 14 | ? | ? | jant (V) | ? | vites, guven, agirlik, bakim, foto incelenmedi | beden, ekipman |
| 32.2 | 47.7 | 32–63 | [Trek Navigator T300](https://www.blocket.se/25499155) | 2 500 | — | — | Goteborg (8 km) | 19 | 60 | 24 | ? | ? | fren, guven, agirlik, foto incelenmedi | ekipman |
| 30.3 | 45.9 | 30–61 | [Crescent Angso](https://www.blocket.se/24997765) | 3 000 | — | — | Goteborg (8 km) | 35 | 61 | 8 | ? | ? | fren, guven, agirlik, foto incelenmedi | ekipman |
| 28.1 | 61.8 | 28–83 | [Merida Mats LX (MTB)](https://www.blocket.se/22282091) | 2 000 | — | — | Simlangsdalen (120 km) | 17 | ? | ? | ? | ? | fren, guven, agirlik, durum, bakim, foto incelenmedi | beden, vites, ekipman |
| 27.8 | 43.9 | 28–62 | [Merida Redwood (MTB kat.)](https://www.blocket.se/26041299) | 2 900 | — | — | Goteborg (8 km) | 0 | ? | ? | ? | ? | guven, agirlik, foto incelenmedi | beden, vites, ekipman |
| 27.7 | 44.7 | 28–62 | [Nishiki City Cross XL (koleksiyon)](https://www.blocket.se/24599946) | 2 500 | — | — | Goteborg (8 km) | 51 | ? | ? | ? | ? | guven, agirlik, foto incelenmedi | beden, vites, ekipman |
| 27.5 | 46.6 | 28–67 | [Btwin Riverside 500](https://www.blocket.se/25972194) | 1 800 | — | — | Goteborg (8 km) | 2 | ? | 9 | ? | ? | fren, guven, agirlik, foto incelenmedi | beden, ekipman |
| 21.0 | 51.8 | 21–77 | [Scott hybrid S dam (KUCUK)](https://www.blocket.se/25256218) | 2 500 | — | — | Kinnarp (110 km) | 0 | ? | ? | disk | ? | vites, ekipman, guven, agirlik, bakim, foto incelenmedi | beden |
| 19.2 | 48.6 | 19–79 | [Apollo sport XL](https://www.blocket.se/26081977) | 2 400 | — | — | Goteborg (8 km) | 0 | ? | ? | ? | ? | vites, fren, guven, agirlik, foto incelenmedi | beden, ekipman |
| 15.2 | 58.5 | 15–85 | [Merida Crossway Woodland M](https://www.blocket.se/26020702) | 2 200 | — | — | Habo (130 km) | 1 | ? | ? | ? | ? | vites, fren, ekipman, guven, agirlik, bakim, foto incelenmedi | beden |
| 15.2 | 48.9 | 15–76 | [Hybridcykel 53cm unisex (Gbg)](https://www.blocket.se/23090187) | 3 000 | — | — | Goteborg (8 km) | 47 | 53 | ? | ? | ? | vites, fren, ekipman, guven, agirlik, bakim, foto incelenmedi | — |
| 15.0 | 51.6 | 15–76 | [Crescent herrcykel 59cm](https://www.blocket.se/23605437) | 3 500 | — | — | Vanersborg (85 km) | 30 | 59 | ? | ? | ? | vites, fren, ekipman, guven, agirlik, bakim, foto incelenmedi | — |
| 11.9 | 47.1 | 12–82 | [Bianchi hybrid M](https://www.blocket.se/25787720) | 3 500 | — | — | V.Frolunda (10 km) | 9 | ? | ? | ? | ? | vites, fren, ekipman, guven, agirlik, bakim, foto incelenmedi | beden |

### Arşiv — KALICI, silinmez

Elenen · vetolanan · satılan · vazgeçilen hiçbir kayıt silinmez; hiçbir temizlik, dedup veya bakım işleminde de. Satılan ilanların **son fiyatı `Değer` modelinin piyasa referansıdır**, bu yüzden fiyat alanı zorunludur.

| Kod | Tarih | Bisiklet | Son fiyat | Sebep |
|---|---|---|---|---|
| ALINACAK | 2026-08-29 | [Merida Crossway herrcykel 55cm](https://www.blocket.se/26132911) | — | E alma karari verdi - islem HENUZ TAMAMLANMADI |
| ELENDI | 2026-08-29 | [Sjosala Nordanvind 55](https://www.blocket.se/25831322) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Crescent Koster 56cm](https://www.blocket.se/25959587) | — | 3 vites kurali |
| ELENDI | 2026-08-29 | [Scott Sportster S](https://www.blocket.se/25296469) | — | elle elendi |
| ELENDI | 2026-08-29 | [Bianchi Celvino 58cm](https://www.blocket.se/24540402) | — | elle elendi |
| ELENDI | 2026-08-29 | [Ronhill RHW5 (Stadium) S](https://www.blocket.se/25365906) | — | elle elendi |
| ELENDI | 2026-08-29 | [Marvil 26 MTB (ELENDI)](https://www.blocket.se/26005737) | — | elle elendi |
| ELENDI | 2026-08-29 | [Avenue L herr (MOTORLU)](https://www.blocket.se/25128834) | — | elle elendi |
| SATILDI | 2026-08-26 | [Merida Crossway Striker (SATILDI)](https://www.blocket.se/25968004) | 2 500 | ilan satildi |
| VAZGECILDI | 2026-08-26 | [Hybridcykel Peak 53cm](https://www.blocket.se/26036304) | — | E vazgecti |

### VETO — sert kısıt ihlali, puan verilmez

| Bisiklet | Fiyat | Yer | VETO SEBEBİ |
|---|---|---|---|
| [Ortler Chur trekking 55cm (Deore 3x9)](https://www.blocket.se/25787860) | 4 500 | Goteborg | MODEL YILI 2017 < 2019 |
| [Scott Sportster 40 herr M](https://www.blocket.se/26035210) | 2 500 | Vaggeryd | KARA LISTE: Scott |
| [Crescent Sport Cross L](https://www.blocket.se/25576096) | 3 345 | Karlstad | MESAFE 250 km > 200 (ayni gun gidilemez) |
| [Hybrid cykel 28 (Yosemite/Biltema)](https://www.blocket.se/25671871) | 1 990 | bilinmiyor | KARA LISTE: Yosemite/Biltema — guvenlik (Rad & Ron testi) |
| [Scott Sub Comfort 10 L 27vxl](https://www.blocket.se/25285342) | 5 000 | Skovde | KARA LISTE: Scott — asiri entegrasyon, bakim maliyeti |
| [Crescent Holma damhybrid 55cm (STOCKHOLM)](https://www.blocket.se/26044987) | 4 500 | Stockholm | MESAFE 470 km > 200 (ayni gun gidilemez) |
| [Skeppshult Herr ARC 7-vxl 53cm](https://www.blocket.se/25083327) | 1 200 | Ulricehamn | KARA LISTE: Skeppshult — E karari (22 kg + dar vites araligi) |
| [Specialized Vita (lag insteg)](https://www.blocket.se/25961988) | 3 490 | Molndal | KARA LISTE: Specialized — strateji karari (Merida ile ayni kadro, marka primi) |
| [Specialized hybrid M (Boras) Tourney](https://www.blocket.se/25372010) | 4 000 | Boras | KARA LISTE: Specialized — strateji karari (Merida ile ayni kadro, marka primi) |
| [Crescent Kebne 16vxl 55cm](https://www.blocket.se/25323134) | 3 500 | Vasteras | MESAFE 350 km > 200 (ayni gun gidilemez) |
| [Specialized sport disc M UCUZ](https://www.blocket.se/25909593) | 1 600 | Falkoping | KARA LISTE: Specialized — strateji karari (Merida ile ayni kadro, marka primi) |
| [Scott sportster tour M UCUZ](https://www.blocket.se/25944341) | 1 500 | Bottnaryd | KARA LISTE: Scott |
| [Trek Dual Sport 2 M 18vxl](https://www.blocket.se/21927297) | 5 000 | Grabo | KARA LISTE: Trek Dual Sport — amortisorlu catal |

### Satıcı teması

- **Merida Crossway Striker L 55cm** — temas kurulmadı — model yılı sorulmalı

## Denetçiye açık sorular

- Ağırlıklar gerekçelendirilmiş mi, yoksa keyfi mi? (A'da beden 18 vs marka 2)
- `Değer` tahminleri tutarlı mı — aynı sınıf iki bisiklete farklı mantık uygulanmış mı?
- Geniş aralıklı puanların listede yüksek sırada durması yanıltıcı mı?
- Eleme kuralları (3 vites) fazla mı katı?
