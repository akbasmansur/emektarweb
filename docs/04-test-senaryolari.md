# 4/b Ne Zaman Emekli Olabilirim? Uygulaması
## 04 – Test Senaryoları

| | |
| --- | --- |
| Doküman sürümü | 0.1 (taslak) |
| Tarih | 14.10.2026 |
| Senaryo sayısı | 61 otomatik (TS-01 – TS-61) + 18 elle yapılacak (MS-01 – MS-18) + 21 tarih aritmetiği birim vakası (BT-01 – BT-21) |
| Makinece okunur sürüm | `04-test-senaryolari.csv` |
| İlgili dokümanlar | 02 İş Kuralları (KR-xx, T-xx), 03 Teknik Analiz (H-xx) |

Bu doküman, yeni Spring Boot uygulamasının eski uygulamayla **aynı sonucu ürettiğini** kanıtlamak için kullanılacak senaryoları içerir. Her senaryo 02'deki bir veya daha fazla kurala bağlıdır.

---

## 1. Beklenen Sonuçlar Nasıl Üretildi

Beklenen sonuçlar elle hesaplanmadı; **eski uygulamanın kendi kodu çalıştırılarak** üretildi:

- `HakKazanmaService`, `HizmetBilgileriService`, `EmeklilikSartlari`, `GetTarih`, `Sabit` ve form sınıfları **değiştirilmeden** derlendi. Struts, commons-lang, slf4j gibi kütüphaneler için yalnızca iş mantığına dokunmayan küçük yer tutucular kullanıldı.
- Her senaryo uygulamadaki akışı izler: emeklilik bilgileri kaydedilir; her hizmet için önce ekleme sayfası kontrolü, sonra `HizmetBilgileriForm.validate()` içindeki Java kuralları çalışır ve kayıt listeye eklenir; son olarak `EmeklilikHesaplaAction` içindeki kontrol ve hesaplama adımları birebir uygulanır.
- Sistem tarihi sabitlendi (libfaketime). **Hesaplama tarihi TS-01 için 12/10/2026, diğerleri için 14/10/2026'dır.** Saat dilimi Europe/Istanbul.
- Doğrulama: TS-01, uygulamanın ekran görüntüsündeki sonucu (56 yıl 9 ay 11 gün, 26 yıl 9 ay 12 gün, 1800 gün, 12/10/2031) **birebir** üretmiştir.

**Kapsam dışı kalanlar:** `validation.xml` kuralları (zorunlu alan, tarih sırası, > 0), tarayıcıdaki JavaScript kuralları ve ekran davranışları bu yöntemle çalıştırılamaz. Bunlar bölüm 5'te elle yapılacak senaryolar olarak verilmiştir.

## 2. Nasıl Kullanılır

**Eski sistemle karşılaştırma (önerilen: en az 10 senaryo)**
Yöntemin güvenilirliğini görmek için birkaç senaryoyu eski uygulamada elle gir. Hesaplama tarihi farklı olacağı için sonuçlar da farklı çıkar (KR-03); iki seçenek var:
1. Senaryoyu **14/10/2026'da** çalıştır, sonuçlar doğrudan karşılaştırılabilir.
2. Başka bir gün çalıştırıyorsan, düzenekle aynı günün sonuçlarını yeniden üret (bölüm 8) ve onlarla karşılaştır.

Hesaplama tarihinden etkilenmeyen senaryolar (bitişi geçmişte olan ve kalan hizmeti 0 olanlar, ör. TS-03, TS-09, TS-15, TS-28) her gün aynı sonucu verir; ilk kontrol için bunlar en uygunudur.

CSV'deki `eskiSistemUISonucu` sütununu eski uygulamadan aldığın sonuçla, `yeniSistemSonucu` sütununu yeni sistemin sonucuyla doldur.

**Yeni sistemde otomatik test**
CSV dosyası JUnit 5'te doğrudan kullanılabilir:

```java
@ParameterizedTest(name = "{0}")
@CsvFileSource(resources = "/04-test-senaryolari.csv", delimiter = ';', numLinesToSkip = 1)
void eskiSistemleAyniSonuc(ArgumentsAccessor satir) {
    LocalDate hesapTarihi = LocalDate.parse(satir.getString(1), DateTimeFormatter.ofPattern("dd/MM/yyyy"));
    // girdi: sütun 2-9, hizmetler sütun 9 ("tur,baslangic,bitis,sure,fiili|...")
    // beklenen: sütun 10 (SONUC/HATA), 11 (mesaj), 12-25 (sonuç alanları)
}
```

Yeni sistemde hesaplama tarihi **parametre** olmalıdır (03 → bölüm 14); aksi halde bu testler yalnızca 14/10/2026'da geçer.

**CSV alan biçimleri**
- `hizmetler`: `tür,başlangıç,bitiş,süre,fiili` kayıtları `|` ile ayrılır. Tür kodları: 1 Bağkur, 2 SSK, 3 Emekli Sandığı, 4 Diğer Sandıklar, 5 Yurt dışı borçlanması, 6 Askerlik borçlanması, 7 Vergi borçlanması. 3201 modunda başlangıç boştur, bitiş ödeme tarihidir.
- Süreler ekrandaki biçimdedir (ör. `3 yıl 2 ay 19 gün`, `1800 gün`).
- `hizmet süresi` değerleri, tarih alanlarından çıkınca ekranın otomatik doldurduğu değerle aynıdır (KR-02); istisnalar senaryo açıklamasında belirtilmiştir.

## 3. Kural Kapsamı

| Kural | Senaryolar |
| --- | --- |
| KR-02, KR-12 hizmet süresi | TS-52, TS-53 |
| KR-03 hesaplama tarihi | TS-01, TS-02 |
| KR-04 – KR-07 giriş | TS-34, MS-01 – MS-07 |
| KR-08 hizmet türleri | TS-35 – TS-39 |
| KR-10, KR-11 doğum, çakışma | TS-49 – TS-51 |
| KR-13 fiili hizmet | TS-54, TS-55 |
| KR-14, KR-15 dönem bölme | TS-56 – TS-58, TS-60, TS-61 |
| KR-16 askerlik | TS-59 |
| KR-19 3201 | TS-45 – TS-48 |
| KR-20, KR-21 ön kontroller | TS-38 – TS-44 |
| KR-22, KR-23 sigortalılık başlangıcı | TS-35 – TS-37, TS-46 |
| KR-25, KR-26 18 yaş, kanun öncesi | TS-15 – TS-17 |
| KR-29, KR-30 yaş, sigortalılık | TS-01, TS-23 |
| KR-31 tam 4447 | TS-03 – TS-04, TS-09, TS-11 – TS-16 |
| KR-32 EYT | TS-05 – TS-09, TS-35 – TS-37, TS-46 |
| KR-33, KR-34 tam 1999 sonrası | TS-10, TS-18 – TS-23 |
| KR-35 – KR-37 kısmi | TS-24 – TS-27 |
| KR-38 maluliyet | TS-28 – TS-30 |
| KR-39 erken yaşlanma | TS-31 |
| KR-41 emeklilik tarihi | TS-01, TS-03, TS-06, TS-23, TS-42 |
| KR-42 bakıma muhtaç çocuk | TS-32 – TS-34 |
| KR-01 tarih aritmetiği | BT-01 – BT-21 |

## 4. Otomatik Senaryolar ve Beklenen Sonuçlar

Tablolarda süreler kısaltılmıştır (`3y 2a 19g` = 3 yıl 2 ay 19 gün). Ayrıntılı alanlar (sigortalılık başlangıç ve bitişi, yaşınız, sigortalılık süreniz, hizmet süreniz) CSV'dedir.

### A. Referans ve hesaplama tarihi

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-01 | KR-29, KR-30, KR-33, KR-40, KR-41 | Ekran görüntüsündeki örnek | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2000–31/12/2019, 7200 gün | 60 yıl · 25 yıl · 9000 gün | 3y 2a 19g · 0y 0a 0g · 1800 gün | **12/10/2031** |
| TS-02 | KR-03 | TS-01 ile aynı girdi, iki gün sonra | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2000–31/12/2019, 7200 gün | 60 yıl · 25 yıl · 9000 gün | 3y 2a 17g · 0y 0a 0g · 1800 gün | **14/10/2031** |

### B. Tam emeklilik – 4447 geçiş ve EYT

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-03 | KR-31, KR-41 | Kalan hizmet ≤ 720: yaş şartı yok; tarih = bitiş + 1 gün | Tam, Erkek, doğum 01/01/1955<br>Bağkur 01/01/1979–31/12/2005, 9719 gün | 0 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2006** |
| TS-04 | KR-31 | Tablo yaşı (52), EYT yok (yaş 2023 öncesi doluyor) | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/06/1990–31/12/2015, 9210 gün | 52 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **01/01/2022** |
| TS-05 | KR-31, KR-32 | EYT: tablo yaşı 51, yaş 2026'da doluyor → yaş şartı kalkar | Tam, Kadın, doğum 01/01/1975<br>Bağkur 01/01/1995–31/12/2025, 11160 gün | 0 yıl · 20 yıl · 7200 gün<br>(EYT öncesi yaş şartı:51) | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-06 | KR-32, KR-41 | EYT: şartlar 2015'te tamam → emeklilik tarihi 03/03/2023'e çekilir | Tam, Kadın, doğum 01/06/1972<br>Bağkur 01/01/1995–31/12/2015, 7560 gün | 0 yıl · 20 yıl · 7200 gün<br>(EYT öncesi yaş şartı:51) | 0y 0a 0g · 0y 0a 0g · 0 gün | **03/03/2023** |
| TS-07 | KR-32 | EYT sınırı: yaş 02/03/2023'te doluyor → EYT yok | Tam, Kadın, doğum 02/03/1972<br>Bağkur 01/01/1995–31/12/2015, 7560 gün | 51 yıl · 20 yıl · 7200 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/03/2023** |
| TS-08 | KR-32 | EYT sınırı: yaş 03/03/2023'te doluyor → EYT var | Tam, Kadın, doğum 03/03/1972<br>Bağkur 01/01/1995–31/12/2015, 7560 gün | 0 yıl · 20 yıl · 7200 gün<br>(EYT öncesi yaş şartı:51) | 0y 0a 0g · 0y 0a 0g · 0 gün | **03/03/2023** |
| TS-09 | KR-31, KR-32, T-01 | Giriş tam 08/09/1999 → 4447 grubu (dahil) | Tam, Erkek, doğum 01/01/1975<br>Bağkur 08/09/1999–31/12/2025, 9473 gün | 0 yıl · 25 yıl · 9000 gün<br>(EYT öncesi yaş şartı:58) | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-10 | KR-33, T-01 | Giriş 09/09/1999 → 60 yaş grubu | Tam, Erkek, doğum 01/01/1975<br>Bağkur 09/09/1999–31/12/2025, 9472 gün | 60 yıl · 25 yıl · 9000 gün | 8y 2a 17g · 0y 0a 0g · 0 gün | **01/01/2035** |
| TS-11 | KR-31, T-05 | Kadın tablo 58 → 56 sınırı (EYT yok) | Tam, Kadın, doğum 01/01/1960<br>Bağkur 01/08/1999–31/12/2025, 9510 gün | 56 yıl · 20 yıl · 7200 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-12 | KR-31, T-05 | Erkek tablo 60 → 58 sınırı (EYT yok) | Tam, Erkek, doğum 01/01/1960<br>Bağkur 01/08/1999–31/12/2025, 9510 gün | 58 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-13 | KR-31, T-06 | Kalan hizmet tam 720 → yaş şartı yok | Tam, Erkek, doğum 01/01/1955<br>Bağkur 01/06/1979–31/12/2005, 9569 gün | 0 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2006** |
| TS-14 | KR-31 | Kalan hizmet 721 → yaş 45 | Tam, Erkek, doğum 01/01/1955<br>Bağkur 02/06/1979–31/12/2005, 9568 gün | 45 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2006** |
| TS-15 | KR-25, KR-26, T-03 | 18 yaş öncesi başlayan tek kayıt: kaydın tamamı düşülür | Tam, Erkek, doğum 01/01/1962<br>Bağkur 01/01/1978–31/12/2005, 10079 gün | 58 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **01/01/2020** |
| TS-16 | KR-25, KR-26, T-03 | TS-15'in 18 yaşta bölünmüş hali: yalnızca ilk kayıt düşülür | Tam, Erkek, doğum 01/01/1962<br>Bağkur 01/01/1978–31/12/1979, 719 gün<br>Bağkur 01/01/1980–31/12/2005, 9359 gün | 45 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **01/01/2007** |
| TS-17 | KR-26, KR-28 | Kanun öncesi hizmete fiili hizmet dahil | Tam, Erkek, doğum 01/01/1965<br>Bağkur 01/01/1990–31/05/2002, 4469 gün (fiili 1000)<br>Bağkur 01/06/2002–31/12/2020, 6690 gün | 50 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2021** |

### C. Tam emeklilik – 1999 sonrası girişler

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-18 | KR-33 | Kadın, giriş 2005 → 58 yaş | Tam, Kadın, doğum 01/01/1980<br>Bağkur 01/01/2005–31/12/2030, 9360 gün | 58 yıl · 25 yıl · 9000 gün | 7y 0a 1g · 0y 0a 0g · 0 gün | **01/01/2038** |
| TS-19 | KR-34 | 5510: bitiş 31/12/2035 → 60 | Tam, Erkek, doğum 01/01/1985<br>Bağkur 01/01/2010–31/12/2035, 9360 gün | 60 yıl · 25 yıl · 9000 gün | 9y 0a 1g · 0y 0a 0g · 0 gün | **01/01/2045** |
| TS-20 | KR-34, T-07 | 5510: bitiş 01/01/2036 → 61 | Tam, Erkek, doğum 01/01/1985<br>Bağkur 01/01/2010–01/01/2036, 9361 gün | 61 yıl · 25 yıl · 9000 gün | 10y 0a 0g · 0y 0a 0g · 0 gün | **01/01/2046** |
| TS-21 | KR-34 | 5510: kadın, bitiş 2048 → 65 | Tam, Kadın, doğum 01/01/1990<br>Bağkur 01/01/2015–31/12/2048, 12240 gün | 65 yıl · 25 yıl · 9000 gün | 6y 0a 1g · 0y 0a 0g · 0 gün | **01/01/2055** |
| TS-22 | KR-34, T-07 | 5510: bitiş bugün girilmiş, hizmet eksik → 60 yaş tablosu | Tam, Erkek, doğum 01/01/1990<br>Bağkur 01/01/2012–14/10/2026, 5324 gün | 60 yıl · 25 yıl · 9000 gün | 23y 2a 17g · 10y 2a 16g · 3676 gün | **01/01/2050** |
| TS-23 | KR-29, KR-30, KR-41 | Bitiş ileri tarihli: yaş ve süreler bitişe göre | Tam, Erkek, doğum 01/01/1975<br>Bağkur 01/01/2000–31/12/2030, 11160 gün | 60 yıl · 25 yıl · 9000 gün | 4y 0a 1g · 0y 0a 0g · 0 gün | **01/01/2035** |

### D. Kısmi emeklilik

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-24 | KR-35, KR-27 | 4447 kısmi: kalan yaş belirleyici (52) | Kısmi, Kadın, doğum 01/01/1955<br>Bağkur 01/01/1985–31/12/2005, 7559 gün | 52 yıl · 15 yıl · 5400 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **01/01/2007** |
| TS-25 | KR-35 | 4447 kısmi: kalan süre ≤ 720 → yaş yok | Kısmi, Erkek, doğum 01/01/1945<br>Bağkur 01/01/1980–31/12/2000, 7559 gün | 0 yıl · 15 yıl · 5400 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2001** |
| TS-26 | KR-36, T-01 | Kısmi, giriş tam 08/09/1999 → 62 yaş grubu (hariç) | Kısmi, Erkek, doğum 01/01/1965<br>Bağkur 08/09/1999–31/12/2020, 7673 gün | 62 yıl · 15 yıl · 5400 gün | 0y 2a 17g · 0y 0a 0g · 0 gün | **01/01/2027** |
| TS-27 | KR-37 | 5510 kısmi: kadın, bitiş 2037 → 62 | Kısmi, Kadın, doğum 01/01/1980<br>Bağkur 01/01/2010–31/12/2037, 10080 gün | 62 yıl · 15 yıl · 5400 gün | 4y 0a 1g · 0y 0a 0g · 0 gün | **01/01/2042** |

### E. Maluliyet ve erken yaşlanma

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-28 | KR-38 | Maluliyet %40-49 | Tam, Erkek, doğum 01/01/1975, maluliyet %40-49<br>Bağkur 01/01/2005–31/12/2025, 7560 gün | 0 yıl · 18 yıl · 4680 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-29 | KR-38 | Maluliyet %50-59 | Tam, Erkek, doğum 01/01/1975, maluliyet %50-59<br>Bağkur 01/01/2005–31/12/2025, 7560 gün | 0 yıl · 16 yıl · 4320 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-30 | KR-38 | Maluliyet %60+ (kısmi seçili, tür etkisiz) | Kısmi, Erkek, doğum 01/01/1975, maluliyet %60+<br>Bağkur 01/01/2005–31/12/2025, 7560 gün | 0 yıl · 15 yıl · 3960 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-31 | KR-39 | Erken yaşlanma: 4447 tablosu 58 → 55, hizmet 5400 | Tam, Erkek, doğum 01/01/1960, erken yaşlanma<br>Bağkur 01/08/1999–31/12/2025, 9510 gün | 55 yıl · 15 yıl · 5400 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |

### F. Bakıma muhtaç malül çocuk

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-32 | KR-42 | Kadın, bakılan 1000 gün → 250 gün düşüş | Tam, Kadın, doğum 01/01/1980, bakıma muhtaç çocuk 1000 gün<br>Bağkur 01/01/2005–31/12/2030, 9360 gün | 58 yıl · 25 yıl · 9000 gün | 6y 3a 21g · 0y 0a 0g · 0 gün<br>Düşülen: 250 gün | **25/04/2037** |
| TS-33 | KR-42 | Bakılan 1003 gün → 250 gün (küsurat atılır) | Tam, Kadın, doğum 01/01/1980, bakıma muhtaç çocuk 1003 gün<br>Bağkur 01/01/2005–31/12/2030, 9360 gün | 58 yıl · 25 yıl · 9000 gün | 6y 3a 21g · 0y 0a 0g · 0 gün<br>Düşülen: 250 gün | **25/04/2037** |
| TS-34 | KR-06, T-14 | Erkek + bakıma muhtaç (ekran engeller, sunucu engellemez) | Tam, Erkek, doğum 01/01/1980, bakıma muhtaç çocuk 1000 gün<br>Bağkur 01/01/2005–31/12/2030, 9360 gün | 60 yıl · 25 yıl · 9000 gün | 9y 0a 1g · 0y 0a 0g · 0 gün | **01/01/2040** |

### G. Borçlanmalar ve sigortalılık başlangıcı

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-35 | KR-22, KR-23, H-03 | Askerlik borçlanması geriye kaydırılır | Tam, Erkek, doğum 01/01/1975<br>Askerlik borçlanması 01/01/1995–01/01/1997, 720 gün<br>Bağkur 01/01/2000–31/12/2025, 9360 gün | 0 yıl · 25 yıl · 9000 gün<br>(EYT öncesi yaş şartı:57) | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-36 | KR-22, KR-23 | Askerlik + yurt dışı borçlanması birlikte | Tam, Erkek, doğum 01/01/1975<br>Askerlik borçlanması 01/01/1995–01/01/1997, 720 gün<br>Yurt dışı borçlanması 01/01/1997–01/01/1999, 360 gün<br>Bağkur 01/01/2002–31/12/2025, 8640 gün | 0 yıl · 25 yıl · 9000 gün<br>(EYT öncesi yaş şartı:58) | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-37 | KR-22, T-11 | Vergi borçlanması gerçek hizmet sayılır, kaydırılmaz | Tam, Erkek, doğum 01/01/1975<br>Vergi borçlanması 01/01/1996–31/12/1997, 719 gün<br>Bağkur 01/01/2000–31/12/2025, 9360 gün | 0 yıl · 25 yıl · 9000 gün<br>(EYT öncesi yaş şartı:57) | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |

### H. Hesaplama öncesi kontroller

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-38 | KR-20 | Yalnızca SSK hizmeti → Bağkur hizmetinizi giriniz | Tam, Erkek, doğum 01/01/1975<br>SSK 01/01/2000–31/12/2025, 9360 gün | – | – | Hata: Bağkur hizmetinizi giriniz |
| TS-39 | KR-20, KR-08 | Vergi borçlanması KR-20'yi geçirir; son 7 yıl SSK olduğu için KR-21 hatası | Tam, Erkek, doğum 01/01/1975<br>Vergi borçlanması 01/01/1996–31/12/1997, 719 gün<br>SSK 01/01/2000–31/12/2025, 9360 gün | – | – | Hata: Son 7 yıldaki fiili çalışma süresine göre Bağkurdan emekli olamazsınız. |
| TS-40 | KR-21 | Giriş 2008 öncesi, son 7 yıl SSK → son 7 yıl mesajı | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/1995–31/12/2010, 5760 gün<br>SSK 01/01/2011–31/12/2025, 5400 gün | – | – | Hata: Son 7 yıldaki fiili çalışma süresine göre Bağkurdan emekli olamazsınız. |
| TS-41 | KR-21 | Giriş 2008 sonrası, toplamda SSK fazla → 5510 mesajı | Tam, Erkek, doğum 01/01/1985<br>Bağkur 01/01/2010–31/12/2014, 1800 gün<br>SSK 01/01/2015–31/12/2025, 3960 gün | – | – | Hata: Toplam hizmet süresine göre en fazla çalışılan Kurum Bağkur olmadığından 5510 SK. 4/b maddesine göre emekli olamazsınız |
| TS-42 | KR-21, T-12 | Son 2520 gün hizmet günüyle sayılır (aradaki boşluk atlanır) | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/1990–31/12/2010, 7560 gün<br>SSK 01/01/2020–31/12/2022, 1080 gün | 52 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 360 gün | **14/10/2027** |
| TS-43 | KR-21, T-12 | Son 2520 günde eşitlik (1260/1260), son kurum Bağkur → geçer | Tam, Erkek, doğum 01/01/1970<br>SSK 01/01/1990–01/07/2022, 11701 gün<br>Bağkur 01/07/2022–01/01/2026, 1260 gün | 52 yıl · 25 yıl · 9000 gün | 0y 0a 0g · 0y 0a 0g · 0 gün | **02/01/2026** |
| TS-44 | KR-21, T-12 | Son 2520 günde eşitlik (1260/1260), son kurum SSK → hata | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/1990–01/07/2022, 11701 gün<br>SSK 01/07/2022–01/01/2026, 1260 gün | – | – | Hata: Son 7 yıldaki fiili çalışma süresine göre Bağkurdan emekli olamazsınız. |

### I. Sadece yurt dışı borçlanması (3201)

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-45 | KR-19, KR-07 | 3201: başlangıç = ödeme − süre; ön kontrol yok | Tam, Kadın, doğum 01/01/1970, sadece 3201<br>Yurt dışı borçlanması: ödeme 01/06/2020, 5400 gün | 58 yıl · 25 yıl · 9000 gün | 1y 2a 17g · 3y 7a 16g · 3600 gün | **14/10/2036** |
| TS-46 | KR-19 | 3201: başlangıç 18 yaşa çekilir | Tam, Erkek, doğum 01/01/1980, sadece 3201<br>Yurt dışı borçlanması: ödeme 01/01/2010, 9000 gün | 0 yıl · 25 yıl · 9000 gün<br>(EYT öncesi yaş şartı:57) | 0y 0a 0g · 0y 0a 0g · 0 gün | **03/03/2023** |
| TS-47 | KR-19 | 3201: ikinci kayıt eklenemez | Tam, Erkek, doğum 01/01/1970, sadece 3201<br>Yurt dışı borçlanması: ödeme 01/01/2010, 5400 gün<br>Yurt dışı borçlanması: ödeme 01/01/2015, 360 gün | – | – | Hata: Yurt dışı borçlanma hizmeti tek kalemde girilir |
| TS-48 | KR-19 | 3201: ödeme tarihi 18 yaştan önce → hata | Tam, Erkek, doğum 01/01/1980, sadece 3201<br>Yurt dışı borçlanması: ödeme 01/01/1997, 360 gün | – | – | Hata: Ödeme tarihi doğum tarihi ile uyumlu değil. (Borçlanma başlangıç tarihi 18 yaşından önce olamaz) |

### J. Hizmet girişi doğrulamaları

| ID | Kural | Amaç | Girdi | Şart (yaş · sigortalılık · hizmet) | Kalan (yaş · sigortalılık · hizmet) | Beklenen sonuç |
| --- | --- | --- | --- | --- | --- | --- |
| TS-49 | KR-10 | Başlangıç doğum tarihine eşit → hata | Tam, Erkek, doğum 01/01/1980<br>Bağkur 01/01/1980–31/12/2000, 1000 gün | – | – | Hata: Hizmet başlangıç tarihi doğum tarihinden önce olamaz |
| TS-50 | KR-11 | Çakışan hizmet → hata | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2000–31/12/2010, 3960 gün<br>Bağkur 01/06/2010–31/12/2020, 3810 gün | – | – | Hata: Girmeye çalıştığınız hizmet aralığı girilmiş hizmetlerle çakışmaktadır. |
| TS-51 | KR-11 | Biten gün başlayan hizmet → çakışma sayılmaz | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2000–01/01/2010, 3601 gün<br>Bağkur 01/01/2010–31/12/2020, 3960 gün | 60 yıl · 25 yıl · 9000 gün | 3y 2a 17g · 0y 0a 0g · 1439 gün | **13/10/2030** |
| TS-52 | KR-02, KR-12 | Bitiş 2008 öncesi, süre = fark + 1 → hata | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2000–31/12/2005, 2161 gün | – | – | Hata: Hizmet süresi fazla girilmiş |
| TS-53 | KR-02, KR-12 | Bitiş 2008 sonrası, süre = fark + 1 → kabul | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2009–31/12/2025, 6120 gün | 60 yıl · 25 yıl · 9000 gün | 3y 2a 17g · 7y 2a 16g · 2880 gün | **14/10/2034** |
| TS-54 | KR-13 | Fiili = hizmet / 4 + 1 → hata | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2003–01/01/2007, 1440 gün (fiili 361) | – | – | Hata: Fiili hizmet süresi fazla girilmiş. (Hizmet süresinin 1/4'ünden fazla olmamalı.) |
| TS-55 | KR-13 | Fiili = hizmet / 4 → kabul | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2003–01/01/2007, 1440 gün (fiili 360)<br>Bağkur 01/01/2007–31/12/2025, 6840 gün | 60 yıl · 25 yıl · 9000 gün | 3y 2a 17g · 1y 2a 16g · 360 gün | **01/01/2030** |
| TS-56 | KR-14 | Tam: 01/06/2002'yi kesen eksik süreli kayıt → hata | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2000–31/12/2005, 2000 gün | – | – | Hata: Hizmet aralığını 01/06/2002 öncesi ve sonrası şeklinde ayrı ayrı giriniz |
| TS-57 | KR-14 | Tam: 01/06/2002'yi kesen tam süreli kayıt → kabul | Tam, Erkek, doğum 01/01/1970<br>Bağkur 01/01/2000–31/12/2025, 9360 gün | 60 yıl · 25 yıl · 9000 gün | 3y 2a 17g · 0y 0a 0g · 0 gün | **01/01/2030** |
| TS-58 | KR-15 | Kısmi: 01/10/1999'u kesen eksik süreli kayıt → hata | Kısmi, Erkek, doğum 01/01/1960<br>Bağkur 01/01/1995–31/12/2005, 3000 gün | – | – | Hata: Hizmet aralığını 01/10/1999 öncesi ve sonrası şeklinde ayrı ayrı giriniz |
| TS-59 | KR-16 | Kadın askerlik borçlanması → hata | Tam, Kadın, doğum 01/01/1975<br>Askerlik borçlanması 01/01/1995–01/01/1996, 360 gün | – | – | Hata: Bayanlar içn askerlik borçlanması girilemez |
| TS-60 | KR-14, H-04, T-02 | Tam + maluliyet: muafiyet olmalı ama hata veriyor | Tam, Erkek, doğum 01/01/1970, maluliyet %40-49<br>Bağkur 01/01/2000–31/12/2005, 2000 gün | – | – | Hata: Hizmet aralığını 01/06/2002 öncesi ve sonrası şeklinde ayrı ayrı giriniz |
| TS-61 | KR-15 | Kısmi + maluliyet: muafiyet çalışıyor → kabul | Kısmi, Erkek, doğum 01/01/1960, maluliyet %40-49<br>Bağkur 01/01/1995–31/12/2005, 3000 gün | 0 yıl · 18 yıl · 4680 gün | 0y 0a 0g · 0y 0a 0g · 1680 gün | **14/06/2031** |
## 5. Elle Yapılacak Senaryolar

Bu senaryolar ekran, tarayıcı ve `validation.xml` davranışını kapsar; eski uygulamada elle denenmeli, yeni sistemde de aynı davranış (veya bilinçli olarak verilen yeni karar) doğrulanmalıdır.

| ID | Kural / Hata | Ekran | Adımlar | Beklenen (eski sistem) |
| --- | --- | --- | --- | --- |
| MS-01 | KR-04 | EK-01 | Emeklilik türü, cinsiyet veya doğum tarihini boş bırakıp TAMAM | "{alan} alanı boş bırakılamaz." EK-01'de kalınır. İlk hatada durulur (yalnızca bir mesaj). |
| MS-02 | KR-04, T-13 | EK-01 | Doğum tarihi 10/01/1000 | Kabul edilir (sınır yok) |
| MS-03 | KR-05 | EK-01 | Maluliyet Evet, derece seçmeden TAMAM | "Maluliyet derecesi alanı boş bırakılamaz." |
| MS-04 | KR-05 | EK-01 | Maluliyet Evet seç | Derece açılır; erken yaşlanma ve 3201 kapanır |
| MS-05 | KR-05 | EK-01 | Erken yaşlanma Evet seç | Maluliyet, derece ve 3201 kapanır |
| MS-06 | KR-06 | EK-01 | Kadın + bakıma muhtaç çocuk işaretle; gün sayısı boş / 0 | Gün sayısı satırı açılır; boşsa "boş bırakılamaz", 0 ise "Bakılan toplam gün sayısı geçersiz" |
| MS-07 | KR-06, KR-07 | EK-01 | Erkek seç; sonra 3201 işaretle | Erkekte çocuk kutusu temizlenir; 3201'de maluliyet, erken yaşlanma ve çocuk kutusu kapanır |
| MS-08 | KR-09 | EK-03 | Başlangıç, bitiş veya süreyi boş bırak | "{alan} alanı boş bırakılamaz." |
| MS-09 | KR-09 | EK-03 | Bitiş = başlangıç veya bitiş < başlangıç | "Başlangıç tarihi bitiş tarihinden büyük veya eşit olamaz" |
| MS-10 | KR-09 | EK-03 | Süre 0 / fiili 0 | "Hizmet süresi geçersiz" / "Fiili hizmet süresi geçersiz" |
| MS-11 | KR-02 | EK-03 | 01/01/2009 ve 31/12/2025 yaz, alandan çık | Süre otomatik 6120 olur |
| MS-12 | KR-17 | EK-03 | Hizmet türü olarak askerlik, yurt dışı veya vergi borçlanması seç | Fiili hizmet alanı kapanır ve temizlenir |
| MS-13 | 03 → 9 | EK-02 | Liste boşken ekran | "Emeklilik Hesapla" butonu görünmez; başlık "Tüm hizmet bilgilerinizi ekleyiniz" |
| MS-14 | 03 → 9 | EK-02 | Satır seçmeden Güncelle / Sil | "Lütfen bir kayıt seçiniz!" uyarısı, istek gitmez |
| MS-15 | 03 → 9 | EK-02 | Satır seç, Sil | "Kaydı silmek istediğinizden emin misiniz?" onayı; onaylanırsa "Hizmetiniz silinmiştir" |
| MS-16 | H-01 | EK-02 | 3 hizmet ekle, 1. kaydı sil, yeni bir hizmet ekle, sonra 3. sıradaki kaydı güncelle | Eski sistemde iki kaydın `siraNo`'su 3 olur; güncelleme yanlış kaydı etkileyebilir. Yeni sistemde doğru kayıt güncellenmeli. |
| MS-17 | H-03 | EK-02 | TS-35'i gir, Emeklilik Hesapla, sonra listeye bak | Eski sistemde askerlik kaydı 01/01/1998–01/01/2000 görünür (girilen: 01/01/1995–01/01/1997). Yeni sistemde girilen tarihler kalmalı. |
| MS-18 | H-05 | herhangi | 20 dakika bekle, bir butona bas | Eski sistemde genel sunucu hata sayfası. Yeni sistemde anlamlı bir mesaj ve başa dönüş beklenir. |

## 6. Tarih Aritmetiği Birim Vakaları

Yeni sistemdeki 360/30 tarih sınıfı (03 → bölüm 14) için birim testler. Değerler eski `GetTarih` sınıfı çalıştırılarak üretilmiştir.

| ID | İşlem | Girdi | Beklenen | Not |
| --- | --- | --- | --- | --- |
| BT-01 | İki tarih arası gün | 01/01/2000 → 31/12/2019 | 7199 gün | 31'i 30 sayılır |
| BT-02 | İki tarih arası gün | 01/01/2000 → 30/12/2019 | 7199 gün | BT-01 ile aynı sonuç |
| BT-03 | İki tarih arası gün | 31/01/2000 → 01/03/2000 | 31 gün | Şubat 30 gün sayılır |
| BT-04 | İki tarih arası gün | 28/02/2001 → 01/03/2001 | 3 gün | Takvimde 1 gün, burada 3 gün |
| BT-05 | İki tarih arası gün | 15/03/2000 → 14/03/2001 | 359 gün |  |
| BT-06 | İki tarih arası gün | 01/12/2000 → 01/01/2001 | 30 gün |  |
| BT-07 | İki tarih arası gün | 31/12/2000 → 31/01/2001 | 30 gün |  |
| BT-08 | İki tarih arası gün | 08/09/1999 → 01/06/2002 | 983 gün |  |
| BT-09 | Tarihe süre ekle | 30/06/2005, 0y 0a 1g | 01/07/2005 |  |
| BT-10 | Tarihe süre ekle | 31/12/2005, 0y 0a 1g | 02/01/2006 | **T-18**: 01/01 değil 02/01 |
| BT-11 | Tarihe süre ekle | 31/01/2005, 0y 0a 1g | 02/02/2005 | **T-18** |
| BT-12 | Tarihe süre ekle | 01/01/2000, 0y 0a 9000g | 01/01/2025 | 25 yıl |
| BT-13 | Tarihe süre ekle | 30/01/1970, 60y 0a 0g | 31/01/2030 | **T-19**: yaş tarihi 1 gün kayar |
| BT-14 | Tarihe süre ekle | 31/01/1970, 60y 0a 0g | 01/02/2030 | **T-19**: sonraki ayın 1'i |
| BT-15 | Tarihe süre ekle | 29/02/1972, 60y 0a 0g | 29/02/2032 |  |
| BT-16 | Tarihe süre ekle | 15/05/1970, 60y 0a 0g | 15/05/2030 |  |
| BT-17 | Tarihten süre çıkar | 01/01/2000, 2y 0a 0g | 01/01/1998 |  |
| BT-18 | Tarihten süre çıkar | 01/01/2038, 0y 0a 250g | 25/04/2037 | **T-20**: gün > 30 iken takvim günüyle çıkarır |
| BT-19 | Tarihten süre çıkar | 01/01/2038, 0y 8a 10g | 21/04/2037 | Aynı süre 360/30 ile: 21/04/2037 |
| BT-20 | Tarihten süre çıkar | 15/03/2010, 0y 0a 20g | 25/02/2010 |  |
| BT-21 | Tarihten süre çıkar | 01/06/2020, 15y 0a 0g | 01/06/2005 |  |

## 7. Karar Bekleyen Senaryolar

Aşağıdaki senaryolarda eski sistemin sonucu 02 → bölüm 10'daki şüpheli davranışlara dayanır. Her biri için "aynen koru" veya "düzelt" kararı verilmeli. Düzeltilenlerin CSV'deki beklenen değeri **yeni kurala göre güncellenmeli** ve `durum` sütununa "Bilinçli fark" yazılmalı.

| Senaryo | Konu | Eski sistemin sonucu | Düzeltilirse |
| --- | --- | --- | --- |
| TS-09 / TS-26 | T-01: 08.09.1999 tam emeklilikte dahil, kısmide hariç | TS-09 EYT ile 02/01/2026; aynı tarihteki TS-10 (bir gün sonra giriş) 01/01/2035 | Mevzuata göre tek bir sınır seçilecek |
| TS-60 | T-02 / H-04: tam emeklilikte maluliyet muafiyeti çalışmıyor | Hata mesajı | Kayıt kabul edilir |
| TS-15 | T-03: 18 yaş öncesi başlayan kaydın tamamı düşülüyor | Yaş 58, tarih 01/01/2020 (bölünmüş halinde TS-16: yaş 45, 01/01/2007) | Yalnızca 18 yaş öncesi kısım düşülürse TS-16'ya yakın sonuç |
| TS-20, TS-22 | T-07: 5510 kademeli yaş, girilen bitiş tarihine göre | Bitişin bir gün değişmesi yaşı 60'tan 61'e çıkarıyor (TS-19 → TS-20) | Şartların tamamlandığı tarihe göre |
| TS-03, TS-13, TS-14 | T-18: 31'ine 1 gün ekleme | 31/12/2005 + 1 gün = 02/01/2006 | 01/01/2006 |
| TS-32, TS-33 | T-20: bakıma muhtaç çocuk düşüşü takvim günüyle | 25/04/2037 | 360/30 ile 21/04/2037 |
| BT-13, BT-14 | T-19: ayın 30/31'inde doğanlarda yaş tarihi kayması | 30/01 doğumlu 60 yaşını 31/01'de doldurur | Doğum günü |
| TS-34 | T-14: erkek için bakıma muhtaç çocuk sunucuda engellenmiyor | Sonucu etkilemiyor | Sunucuda reddedilir |
| MS-02 | T-13: doğum tarihi sınırı yok | Kabul | Reddedilir |
| MS-16, MS-17, MS-18 | H-01, H-03, H-05 | Hatalı davranış | Düzeltilmiş davranış |

## 8. Beklenen Sonuçları Başka Bir Tarih İçin Üretmek

Senaryoların sonuçları bir test düzeneğiyle (eski kod + senaryo dosyası) üretildi. Eski uygulamayı farklı bir günde çalıştırırsan, aynı gün için beklenen değerleri bu düzenekle yeniden üretebilirim; ayrıca yeni senaryo eklemek de birkaç dakika sürer. Senaryo tanımlarındaki hesaplama tarihini değiştirmen yeterli.

Bu düzenek ileride **yeni sistemle eski sistemi yan yana çalıştıran karşılaştırma testine** de dönüştürülebilir: aynı girdiyi iki koda verip sonuçları otomatik karşılaştırmak, rastgele üretilmiş yüzlerce kişiyle bile mümkün olur.
