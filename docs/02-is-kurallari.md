# 4/b Ne Zaman Emekli Olabilirim? Uygulaması
## 02 – İş Kuralları ve Hesaplama

| | |
| --- | --- |
| Doküman sürümü | 0.1 (taslak) |
| Tarih | 13.10.2026 |
| Kaynak | Mevcut uygulamanın kaynak kodu (v1.1.0) |
| İlgili dokümanlar | 01 Uygulama Genel Tanımı, 03 Teknik Analiz, 04 Test Senaryoları |

Bu doküman, mevcut uygulamanın **kodda yazılı olan** kurallarını anlatır. Kuralların mevzuata uygunluğu ayrıca doğrulanmalıdır; kod yorumlarında geçen mevzuat atıfları "Dayanak" satırlarında verilmiştir. Kodla mevzuat arasında şüpheli görülen noktalar bölüm 10'da toplanmıştır.

Kural kimlikleri (KR-xx) 04 Test Senaryoları'nda kullanılır. Ekran kimlikleri (EK-xx) 01 ile aynıdır.

---

## İçindekiler

1. Temel kavramlar ve tarih aritmetiği (KR-01 – KR-03)
2. Emeklilik bilgileri giriş kuralları (KR-04 – KR-07)
3. Hizmet girişi kuralları (KR-08 – KR-19)
4. Hesaplama öncesi kontroller (KR-20 – KR-21)
5. Sigortalılık başlangıcı ve bitişi (KR-22 – KR-24)
6. Ara hesaplar (KR-25 – KR-30)
7. Emeklilik şartları (KR-31 – KR-39)
8. Kalan süreler ve emeklilik tarihi (KR-40 – KR-43)
9. Örnek hesaplama
10. Bilinen tuhaflıklar ve mevzuatla doğrulanacak noktalar
11. Mesaj kataloğu
12. Test için sınır değerler

---

## 1. Temel Kavramlar ve Tarih Aritmetiği

### KR-01 – 360/30 gün aritmetiği
- **Kural:** Bütün süre hesapları **1 yıl = 360 gün, 1 ay = 30 gün** kabulüyle yapılır. Ayın 31'i 30 sayılır.
- **İki tarih arası:** Yıl, ay ve gün farkları ayrı ayrı alınır; gün negatifse aydan 30, ay negatifse yıldan 12 ödünç alınır. Sonuç = yıl × 360 + ay × 30 + gün.
- **Tarihe süre ekleme/çıkarma:** Gün sayısı 30'luk aylara, aylar 12'lik yıllara bölünerek eklenir/çıkarılır.
- **Gösterim:** Gün olarak tutulan süreler ekranda aynı kabulle yıl/ay/güne çevrilir (ör. 1159 gün = 3 yıl 2 ay 19 gün).
- **Kodda:** `GetTarih.getDateDifferenceForBagkur`, `getDateDifferenceInYilAyGun`, `tarihinUstuneYilAyGunEkle`, `tarihinUstuneYilAyGunCikart`; `HizmetBilgileriService.getYilAyGunArray`
- **Not:** Takvim hesabından (Java `LocalDate`) farklı sonuç verir. Yeni sistemde aynen korunmalıdır.

### KR-02 – Hizmet süresinin hesaplanması
- **Kural:** Hizmet süresi = başlangıç ile bitiş arasındaki gün farkı (KR-01). **Bitiş tarihi 01.10.2008 veya sonrasıysa +1 gün** eklenir.
- **Ekranda:** Tarih alanlarından çıkıldığında hizmet süresi bu kuralla otomatik doldurulur; kullanıcı değeri değiştirebilir.
- **Sunucuda:** Girilen süre bu değeri aşamaz (KR-12).
- **Kodda:** `hizmetBilgileriEkle.jsp` / `Guncelle.jsp` → `gunFarkiniBul()`; `HizmetBilgileriForm.girilenHizmetSuresiniKontrolEt`
- **Dayanak:** 01.10.2008 = 5510 sayılı Kanun'un yürürlük tarihi [+1 günün dayanağı doğrulanacak]

### KR-03 – Hesaplama tarihi
- **Kural:** Yaş, sigortalılık süresi ve emeklilik tarihi, **hesaplamanın yapıldığı gün** (sistem tarihi) esas alınarak bulunur. Aynı girdi farklı günlerde farklı sonuç verir.
- **Kodda:** `new Date()`, `GetTarih.getTarihString()`, `GetTarih.getBugununTarihi()`
- **Not:** Test senaryolarında hesaplama tarihi mutlaka kaydedilmeli; yeni sistemde bu tarih parametre olmalıdır.

---

## 2. Emeklilik Bilgileri Giriş Kuralları (EK-01)

### KR-04 – Zorunlu alanlar
- Emeklilik türü (Tam / Kısmi), cinsiyet ve doğum tarihi zorunludur.
- Doğum tarihi için biçim dışında **hiçbir sınır yoktur** (ör. 10/01/1000 kabul edilir). Nokta ve tire `/` ile değiştirilir.
- **Kodda:** `validation.xml` (EmeklilikBilgileriForm), `EmeklilikBilgileriForm.setDogumTarihi`

### KR-05 – Özel durumlar birbirini dışlar
- Maluliyet "Evet" ise **maluliyet derecesi zorunludur** (%40-49, %50-59, %60 ve üzeri kuruma girmeden önce malul olanlar).
- Maluliyet, erken yaşlanma ve "sadece yurt dışı borçlanması" aynı anda seçilemez: biri seçilince diğerleri kapanır.
- **Kodda:** `validation.xml` (requiredif); `emeklilikBilgileri.jsp` → `maluliyetKontrol`, `erkenYaslanmaKontrol`, `sadece3201Kontrol`
- **Not:** Karşılıklı dışlama yalnızca tarayıcıda yapılır; sunucu kontrol etmez. Kapalı alanlar gönderilmediği için "Hayır" sayılır.

### KR-06 – Bakıma muhtaç malül çocuk
- Yalnızca **kadın** sigortalı seçebilir; erkek seçilirse kutu temizlenir.
- Seçildiğinde **bakılan gün sayısı** (01.10.2008 sonrası) zorunludur ve 0'dan büyük olmalıdır.
- "Sadece yurt dışı borçlanması" ile birlikte seçilemez.
- Etkisi: KR-42.
- **Kodda:** `validation.xml` (bakGunSayisi), `bakimaMuhtacKontrol()`

### KR-07 – Sadece yurt dışı borçlanması (3201 modu)
- Seçildiğinde maluliyet, erken yaşlanma ve bakıma muhtaç çocuk kapanır.
- Hizmet girişi farklı çalışır (KR-19) ve hesaplama öncesi kontroller yapılmaz (KR-20, KR-21).
- **Dayanak:** 3201 sayılı Kanun (yurt dışı borçlanması)

---

## 3. Hizmet Girişi Kuralları (EK-03, EK-04)

### KR-08 – Hizmet türleri

| Kod | Hizmet türü | Fiili hizmet girilebilir mi | Son 7 yıl hesabında | Sigortalılık başlangıcında |
| --- | --- | --- | --- | --- |
| 1 | Bağkur | Evet | Bağkur | Gerçek hizmet |
| 2 | SSK | Evet | SSK | Gerçek hizmet |
| 3 | Emekli Sandığı | Evet | Emekli Sandığı | Gerçek hizmet |
| 4 | Diğer Sandıklar | Evet | Diğer | Gerçek hizmet |
| 5 | Yurt dışı borçlanması | Hayır | **Bağkur sayılır** | Geriye kaydırılır (KR-23) |
| 6 | Askerlik borçlanması | Hayır | **Bağkur sayılır** | Geriye kaydırılır (KR-23) |
| 7 | Vergi borçlanması | Hayır | **Bağkur sayılır** | Gerçek hizmet |

Ekranda sözleşmeli aylıklarda yurt dışında çalışılan sürelerin "Bağkur" olarak girilmesi gerektiği uyarısı gösterilir.

### KR-09 – Zorunlu alanlar ve temel kontroller
- Başlangıç tarihi, bitiş tarihi ve hizmet süresi zorunludur.
- Bitiş tarihi başlangıçtan **sonra** olmalıdır (eşit olamaz).
- Hizmet süresi 0'dan büyük olmalıdır; fiili hizmet süresi boş ya da 0'dan büyük olmalıdır.
- **Kodda:** `validation.xml` (HizmetBilgileriForm), `ValidationUtil.validateDateRange`

### KR-10 – Hizmet doğumdan sonra başlamalı
- Başlangıç tarihi doğum tarihinden **sonra** olmalıdır.
- **Kodda:** `HizmetBilgileriForm.dogumTarihiniKontrolEt`

### KR-11 – Hizmetler çakışamaz
- Yeni veya güncellenen kayıt, listedeki diğer kayıtlarla aynı tarih aralığına düşemez.
- Bir kaydın bittiği gün diğerinin başlaması **çakışma sayılmaz**.
- **Kodda:** `HizmetBilgileriForm.hizmetlerCakisiyormu`, `GetTarih.tarihAraliklariKesisiyormu`

### KR-12 – Hizmet süresi üst sınırı
- Girilen hizmet süresi, KR-02 ile bulunan değeri aşamaz.
- **Kodda:** `HizmetBilgileriForm.girilenHizmetSuresiniKontrolEt`

### KR-13 – Fiili hizmet süresi sınırı
- Fiili hizmet süresi, hizmet süresinin **1/4'ünden** fazla olamaz (fiili × 4 ≤ hizmet süresi).
- **Kodda:** `HizmetBilgileriForm.fiiliHizmetSuresiniKontrolEt`

### KR-14 – Tam emeklilikte 01.06.2002 bölmesi
- **Kural:** Emeklilik türü Tam ise, 01.06.2002'yi **kesen** bir kayıt (başlangıç < 01.06.2002 < bitiş) şu durumlarda iki ayrı kayıt olarak girilmelidir:
  - hizmet süresi tarih farkından azsa (arada prim ödenmeyen günler varsa), veya
  - fiili hizmet süresi girilmişse.
- **Neden:** Kanun öncesi hizmet (KR-26) eşiği kesen kayıtlarda takvim farkıyla hesaplanır; eksik günler veya fiili hizmet eşiğin hangi tarafında kaldığı bilinemez.
- **Muafiyet:** Maluliyet veya erken yaşlanma varsa kontrol yapılmamalı; ancak koddaki hata nedeniyle **muafiyet çalışmıyor** (03 → H-04).
- **Kodda:** `HizmetBilgileriForm.tamEmeklilikHizmetKontrolEt`

### KR-15 – Kısmi emeklilikte 01.10.1999 bölmesi
- KR-14'ün aynısı, Kısmi emeklilik için **01.10.1999** eşiğiyle. Maluliyet veya erken yaşlanma varsa kontrol yapılmaz (muafiyet çalışıyor).
- **Kodda:** `HizmetBilgileriForm.kismiEmeklilikHizmetKontrolEt`

### KR-16 – Kadın için askerlik borçlanması
- Kadın sigortalı askerlik borçlanması giremez.
- **Kodda:** `HizmetBilgileriForm.askerlikBorclanmasiKontrol`

### KR-17 – Borçlanmalarda fiili hizmet
- Yurt dışı, askerlik ve vergi borçlanmalarında fiili hizmet alanı kapanır ve temizlenir.
- **Kodda:** `fiiliHizmetKontrolu()` (yalnızca tarayıcıda)

### KR-18 – 18 yaş altı hizmet kontrolü (kapalı)
- Eskiden 18 yaşından önce başlayan hizmetler kısıtlanıyordu; kontrol **28.05.2020'de kapatılmıştır** (jazz:74126). Şu an 18 yaş altı hizmet girilebilir.
- **Kodda:** `HizmetBilgileriForm.Yas18AltiHizmetKontroluYap` (çağrılmıyor)

### KR-19 – Sadece yurt dışı borçlanması (3201) ile hizmet girişi
- Yalnızca **tek** kayıt girilebilir; ikinci kayıt için "Yurt dışı borçlanma hizmeti tek kalemde girilir" hatası verilir.
- Hizmet türü otomatik olarak "Yurt dışı borçlanması"dır.
- Kullanıcı **ödeme tarihi** (bitiş tarihi olarak) ve **borçlanılan süreyi** girer; başlangıç tarihi girilmez.
- Başlangıç tarihi = ödeme tarihi − borçlanılan süre (KR-01). Bu tarih **18 yaşından önceyse 18 yaş tarihine çekilir**.
- Başlangıç doğum tarihinden önceyse veya ödeme tarihinden önce değilse hata verilir.
- **Kodda:** `HizmetBilgileriEkleSayfaAction`, `HizmetBilgileriForm.validate` (3201 dalı), `hizmetBilgileriEkle.jsp`

---

## 4. Hesaplama Öncesi Kontroller

Bu kontroller yalnızca "sadece yurt dışı borçlanması" **seçili değilse** yapılır. Kontrol geçilemezse hesaplama yapılmaz, EK-02'de hata mesajı gösterilir.

### KR-20 – Bağkur hizmeti zorunluluğu
- Listede en az bir **Bağkur** veya **vergi borçlanması** kaydı olmalıdır.
- Mesaj: "Bağkur hizmetinizi giriniz"
- **Kodda:** `EmeklilikHesaplaAction`, `HizmetBilgileriService.bagkurHizmetiVarmi`

### KR-21 – Ağırlıklı kurum Bağkur olmalı
- **Kural:** Belirli bir dönemde en çok hizmeti olan kurum Bağkur değilse 4/b'den hesaplama yapılmaz.
- **Dönem:**
  - Sigortalılık başlangıcı (KR-22) **30.04.2008'den sonraysa:** tüm hizmet süresi (5510 kuralı).
  - **Değilse:** "son 7 yıl" – son kayıttan geriye doğru **2520 gün hizmet** sayılarak bulunan tarihten sonrası (4447 kuralı). Geriye sayım takvim değil hizmet günüyle yapılır; aradaki boşluklar atlanır. Toplam hizmet 2520 günden azsa ilk kaydın başlangıcından itibaren.
- **Sayım:**
  - Yurt dışı, askerlik ve vergi borçlanmaları **Bağkur sayılır**.
  - Tamamen dönem içindeki kayıtların tüm hizmet süresi; dönemi kesen kayıtların dönem içindeki takvim farkı (hizmet süresini aşmamak üzere) sayılır.
  - Fiili hizmet süreleri bu hesaba **katılmaz**.
  - **Eşitlik** durumunda en son çalışılan kurum seçilir.
  - 1260 gün (3,5 yıl) asgari süre kontrolü **yapılmaz** (kodda bilinçli olarak kapatılmış, "MEVZUAT" notuyla).
- **Mesajlar:**
  - Başlangıç 30.04.2008 sonrası: "Toplam hizmet süresine göre en fazla çalışılan Kurum Bağkur olmadığından 5510 SK. 4/b maddesine göre emekli olamazsınız"
  - Diğer: "Son 7 yıldaki fiili çalışma süresine göre Bağkurdan emekli olamazsınız."
- **Kodda:** `HizmetBilgileriService.getSon7YilKurum`, `getSon7YilinBasladigiTarih`
- **Dayanak:** 4447 sayılı Kanun (son 7 yıl), 5510 sayılı Kanun (toplam hizmet)

---

## 5. Sigortalılık Başlangıcı ve Bitişi

### KR-22 – Sigortalılık başlangıç tarihi
- Hizmetler başlangıç tarihine göre sıralanır.
- Baştaki **askerlik ve yurt dışı borçlanmaları** atlanır; ilk gerçek hizmetin başlangıç tarihi bulunur.
- Atlanan borçlanma varsa, toplam süreleri kadar (KR-01) **geriye gidilir**; sonuç sigortalılık başlangıcıdır.
- Vergi borçlanması gerçek hizmet sayılır, geriye götürülmez.
- 3201 modunda geriye gitme yapılmaz; başlangıç, KR-19'da hesaplanan tarihtir.
- 18 yaş sınırı uygulanmaz (eskiden başlangıç 18 yaşına çekiliyordu; EYT çalışmasıyla kapatılmış).
- **Kodda:** `HizmetBilgileriService.getSBaslangicTarihi`

### KR-23 – Borçlanmaların geriye kaydırılması
- KR-22'de atlanan askerlik ve yurt dışı borçlanmaları, ilk gerçek hizmetin **hemen öncesine, ardışık olarak** yerleştirilir: her birinin bitişi bir sonraki kaydın başlangıcı, başlangıcı da bitişten süresi kadar öncesidir.
- Bu kaydırılmış tarihler kanun öncesi hizmet (KR-26) ve son 7 yıl (KR-21) hesaplarında kullanılır.
- İlk gerçek hizmetten **sonra** gelen borçlanmalara dokunulmaz.
- **Kodda:** `HizmetBilgileriService.hizmetDuzelt`
- **Not:** Mevcut sistem kaydırılmış tarihleri kullanıcının listesine yazıyor (03 → H-03). Kural korunmalı, yan etki kaldırılmalı.

### KR-24 – Sigortalılık bitiş tarihi
- Başlangıç tarihine göre sıralı listede **son kaydın bitiş tarihidir**.
- **Kodda:** `HizmetBilgileriService.getSBitisTarihi`

---

## 6. Ara Hesaplar

### KR-25 – 18 yaş öncesi hizmet
- Şu koşulları sağlayan kayıtların **tüm hizmet süresi** toplanır (fiili hariç):
  - SSK hizmeti ve başlangıcı **01.04.1981'den önce**, veya Bağkur hizmeti ve başlangıcı **04.05.1979'dan önce**, ve
  - başlangıcı **18 yaşını doldurduğu tarihten önce**.
- Bu süre kanun öncesi hizmetten düşülür (KR-26).
- **Kodda:** `HakKazanmaService.getYas18OncesiHizmet`
- **Not:** Kaydın yalnızca 18 yaş öncesi kısmı değil, tamamı sayılıyor (bkz. 10).

### KR-26 – Kanun öncesi hizmet
- **Eşik tarihi:** Tam emeklilikte **01.06.2002**, kısmi emeklilikte **01.10.1999**.
- Bitişi eşikten sonra olmayan kayıtlar: hizmet süresi + fiili hizmet süresi.
- Eşiği kesen kayıtlar: başlangıç ile eşik arasındaki takvim farkı (KR-01) + fiili hizmetin **tamamı**.
- Eşikten sonra başlayan kayıtlar sayılmaz.
- Sonuçtan KR-25'teki 18 yaş öncesi hizmet düşülür; negatifse 0 alınır.
- Hizmet türü ayrımı yapılmaz; tüm kayıtlar sayılır.
- **Kodda:** `HakKazanmaService.getKanunOncesiHizmet`
- **Dayanak:** Fiili hizmetler 2013-26 sayılı genelge ile dahil edilmiş (09.07.2013); 18 yaş öncesi düşümü 18.01.2012 tarihli değişiklik.

### KR-27 – 01.10.1999 tarihindeki yaş
- Doğum tarihinden 01.10.1999'a kadar geçen süre, gün olarak (KR-01). Kısmi emeklilik 4447 geçişinde kullanılır (KR-35).
- **Kodda:** `HakKazanmaService.get_01_10_1999_YasiGunOlarak`

### KR-28 – Toplam hizmet süresi ("Hizmet süreniz")
- Tüm kayıtların hizmet süresi + fiili hizmet süresi toplamı.
- **Kodda:** `HizmetBilgileriService.getToplamHizmet`

### KR-29 – Yaşınız
- Doğum tarihinden, sigortalılık bitişi **ileri bir tarihse o tarihe**, değilse **bugüne** kadar geçen süre (KR-01).
- **Kodda:** `HakKazanmaService.getSigortaliYas`

### KR-30 – Sigortalılık süreniz
- Sigortalılık başlangıcından (KR-22), bitiş ileri bir tarihse o tarihe, değilse bugüne kadar geçen süre (KR-01).
- Bugün 01.10.2008'den sonraysa **+1 gün** eklenir (artık her zaman eklenir).
- **Kodda:** `HakKazanmaService.getSigortalilikSuresi`

---

## 7. Emeklilik Şartları

Şartlar üç değerden oluşur: **yaş**, **hizmet süresi** (gün) ve **sigortalılık süresi** (gün). Yaş 0 ise yaş şartı yoktur. Aşağıdaki "başlangıç" sigortalılık başlangıcıdır (KR-22). Kod: `EmeklilikSartlari` yapıcısı.

### Genel karar tablosu

| Durum | Başlangıç | Hizmet şartı | Sigortalılık şartı | Yaş şartı |
| --- | --- | --- | --- | --- |
| Tam | ≤ 08.09.1999 | E 9000 / K 7200 | Hizmet şartı ile aynı | KR-31 (4447 geçiş tablosu) + KR-32 (EYT) |
| Tam | 08.09.1999 – 30.04.2008 | 9000 | 9000 | E 60 / K 58 |
| Tam | ≥ 01.05.2008 | 9000 | 9000 | KR-34 (5510 kademeli) |
| Kısmi | < 08.09.1999 | 5400 | 5400 | KR-35 (4447 geçiş tablosu) |
| Kısmi | 08.09.1999 – 30.04.2008 | 5400 | 5400 | E 62 / K 60 |
| Kısmi | ≥ 01.05.2008 | 5400 | 5400 | KR-37 (5510 kademeli) |
| Maluliyet | – | KR-38 | KR-38 | Yok |
| Erken yaşlanma | – | 5400 | 5400 | En fazla 55 (KR-39) |

E: erkek, K: kadın. 9000 gün = 25 yıl, 7200 = 20 yıl, 5400 = 15 yıl.

Tam emeklilikte 08.09.1999 **dahil** (≤), kısmi emeklilikte **hariç** (<) tutulmuştur (bkz. 10).

### KR-31 – Tam emeklilik, 08.09.1999 ve öncesi girişler (4447 geçiş)
1. Hizmet şartı: erkek **9000**, kadın **7200** gün. Sigortalılık şartı aynı.
2. Kalan hizmet = hizmet şartı − kanun öncesi hizmet (KR-26, eşik 01.06.2002).
3. Kalan hizmet **720 günü aşmıyorsa yaş şartı yoktur** (yaş = 0).
4. Aşıyorsa yaş aşağıdaki tablodan bulunur, sonra **erkek için 58, kadın için 56 ile sınırlanır**.

**Kadın (kalan hizmete göre)**

| Kalan hizmet (gün) | Yaş | | Kalan hizmet (gün) | Yaş |
| --- | --- | --- | --- | --- |
| ≤ 720 | 40 → yaş şartı yok (madde 3) | | 3241 – 3600 | 48 |
| 721 – 1080 | 41 | | 3601 – 3960 | 49 |
| 1081 – 1440 | 42 | | 3961 – 4320 | 50 |
| 1441 – 1800 | 43 | | 4321 – 4680 | 51 |
| 1801 – 2160 | 44 | | 4681 – 5040 | 52 |
| 2161 – 2520 | 45 | | 5041 – 5400 | 53 |
| 2521 – 2880 | 46 | | 5401 – 5760 | 54 |
| 2881 – 3240 | 47 | | 5761 – 6120 | 55 |
| | | | > 6120 | 58 → **56** (sınır) |

**Erkek (kalan hizmete göre)**

| Kalan hizmet (gün) | Yaş | | Kalan hizmet (gün) | Yaş |
| --- | --- | --- | --- | --- |
| ≤ 720 | 44 → yaş şartı yok (madde 3) | | 4501 – 5040 | 52 |
| 721 – 1260 | 45 | | 5041 – 5580 | 53 |
| 1261 – 1800 | 46 | | 5581 – 6120 | 54 |
| 1801 – 2340 | 47 | | 6121 – 6660 | 55 |
| 2341 – 2880 | 48 | | 6661 – 7200 | 56 |
| 2881 – 3420 | 49 | | 7201 – 7740 | 57 |
| 3421 – 3960 | 50 | | 7741 – 7920 | 58 |
| 3961 – 4500 | 51 | | > 7920 | 60 → **58** (sınır) |

- **Dayanak:** 4447 sayılı Kanun geçiş hükümleri

### KR-32 – EYT (yalnızca tam emeklilik, 08.09.1999 ve öncesi girişler)
- **Koşul:** KR-31'e göre yaş şartı > 0, erken yaşlanma yok ve kişinin bu yaşı doldurduğu tarih (doğum + yaş) **03.03.2023 veya sonrası**.
- **Sonuç:** Yaş şartı kaldırılır (0). Eski yaş "EYT öncesi yaş şartı" olarak saklanır ve ekranda "(EYT öncesi yaş şartı: X)" diye gösterilir.
- Yaşı 03.03.2023'ten önce doldurmuş olanların yaş şartı değişmez.
- Emeklilik tarihi 03.03.2023'ten önceye düşemez (KR-41, madde 6).
- Kısmi emeklilikte, maluliyette ve erken yaşlanmada uygulanmaz.
- **Kodda:** `EmeklilikSartlari` (eytKapsamindaMi, yas_eytOncesi), `HakKazanmaService.emeklilikSartlariKarsilastir`, `emeklilikTarihiHesapla`
- **Dayanak:** EYT düzenlemesi, yürürlük 03.03.2023 [kanun numarası ve maddesi doğrulanacak]

### KR-33 – Tam emeklilik, 08.09.1999 – 30.04.2008 girişler
- Hizmet ve sigortalılık şartı 9000 gün; yaş erkek **60**, kadın **58**.

### KR-34 – Tam emeklilik, 01.05.2008 ve sonrası girişler (5510 kademeli)
- Hizmet ve sigortalılık şartı 9000 gün. Yaş, **sigortalılık bitiş tarihine** (KR-24) göre:

| Sigortalılık bitişi | Erkek | Kadın |
| --- | --- | --- |
| 01.01.2036'dan önce | 60 | 58 |
| 01.01.2036 – 31.12.2037 | 61 | 59 |
| 01.01.2038 – 31.12.2039 | 62 | 60 |
| 01.01.2040 – 31.12.2041 | 63 | 61 |
| 01.01.2042 – 31.12.2043 | 64 | 62 |
| 01.01.2044 – 31.12.2045 | 65 | 63 |
| 01.01.2046 – 31.12.2047 | 65 | 64 |
| 01.01.2048 ve sonrası | 65 | 65 |

### KR-35 – Kısmi emeklilik, 08.09.1999 öncesi girişler (4447 geçiş)
1. Hizmet ve sigortalılık şartı **5400** gün.
2. Kalan hizmet = 5400 − kanun öncesi hizmet (KR-26, eşik 01.10.1999).
3. Kalan yaş = (kadın **18000**, erkek **19800** gün; yani 50 ve 55 yaş) − 01.10.1999'daki yaş (KR-27).
4. Kalan süre = kalan hizmet ve kalan yaştan **büyük olanı**.
5. Yaş:

| Kalan süre (gün) | Kadın | Erkek |
| --- | --- | --- |
| ≤ 720 | yaş şartı yok | yaş şartı yok |
| 721 – 1440 | 51 | 56 |
| 1441 – 2160 | 52 | 56 |
| 2161 – 2880 | 53 | 57 |
| 2881 – 3600 | 54 | 57 |
| > 3600 | 56 | 58 |

- Kısmi emeklilikte EYT uygulanmaz.

### KR-36 – Kısmi emeklilik, 08.09.1999 – 30.04.2008 girişler
- Hizmet ve sigortalılık şartı 5400 gün; yaş erkek **62**, kadın **60**.

### KR-37 – Kısmi emeklilik, 01.05.2008 ve sonrası girişler (5510 kademeli)
- Hizmet ve sigortalılık şartı 5400 gün. Yaş, sigortalılık bitiş tarihine göre:

| Sigortalılık bitişi | Erkek | Kadın |
| --- | --- | --- |
| 01.01.2036'dan önce | 63 | 61 |
| 01.01.2036 – 31.12.2037 | 64 | 62 |
| 01.01.2038 – 31.12.2039 | 65 | 63 |
| 01.01.2040 – 31.12.2041 | 65 | 64 |
| 01.01.2042 ve sonrası | 65 | 65 |

### KR-38 – Maluliyet
- Yaş şartı yoktur. Emeklilik türü (Tam/Kısmi) ve giriş tarihi dikkate alınmaz.

| Maluliyet derecesi | Sigortalılık şartı | Hizmet şartı |
| --- | --- | --- |
| %40 – 49 | 18 yıl (6480 gün) | 4680 gün |
| %50 – 59 | 16 yıl (5760 gün) | 4320 gün |
| %60 ve üzeri, kuruma girmeden önce malul | 15 yıl (5400 gün) | 3960 gün |

### KR-39 – Erken yaşlanma
- Diğer şartlar bulunduktan sonra uygulanır: yaş şartı **en fazla 55** olur; hizmet ve sigortalılık şartı **5400** gün olur.
- Erken yaşlanmada EYT uygulanmaz.

---

## 8. Kalan Süreler ve Emeklilik Tarihi

### KR-40 – Kalan süreler
- **Kalan yaş** = yaş şartı × 360 − yaşınız (KR-29)
- **Kalan sigortalılık** = sigortalılık şartı − sigortalılık süreniz (KR-30)
- **Kalan hizmet** = hizmet şartı − toplam hizmet (KR-28)
- Negatif çıkan değer 0 gösterilir. EYT ile yaş şartı kaldırıldıysa kalan yaş 0'dır.
- **Kodda:** `HakKazanmaService.emeklilikSartlariKarsilastir`

### KR-41 – Emeklilik tarihi
Aşağıdaki tarihlerin **en geç olanı** alınır:

1. Sigortalılık bitiş tarihi (KR-24)
2. Yaş şartı varsa: doğum tarihi + yaş şartı
3. Sigortalılık başlangıcı + sigortalılık şartı (gün)
4. Sigortalılık başlangıcı + hizmet şartı (gün)
5. Kalan hizmet varsa: (sigortalılık bitişi ileri tarihliyse o tarih, değilse **bugün**) + kalan hizmet (gün). Yani eksik hizmetin bugünden itibaren **kesintisiz** tamamlanacağı varsayılır.

Ardından sırasıyla:

6. **EYT kanun tarihi:** EYT uygulandıysa (KR-32), bulunan tarih 03.03.2023'ten önceyse ve eski yaş şartıyla bulunacak tarih (doğum + EYT öncesi yaş) bundan sonraysa, emeklilik tarihi **03.03.2023** alınır. Yani kişi, yaş şartı kalkmış olsa da yasanın yürürlüğünden önce emekli olamaz.
7. **Bakıma muhtaç çocuk:** Kadın sigortalı için, 6. madde uygulanmadıysa, tarihten **bakılan gün / 4** gün düşülür (KR-42).
8. Sonuç sigortalılık bitişinden önceyse sigortalılık bitişi alınır.
9. Sonuç sigortalılık bitişine **eşitse 1 gün sonrası** alınır.

Ekranda "gg/aa/yyyy tarihinde emekli olabilirsiniz." olarak gösterilir.
- **Kodda:** `HakKazanmaService.emeklilikTarihiHesapla`

### KR-42 – Bakıma muhtaç malül çocuk düşüşü
- Yalnızca kadın sigortalıda uygulanır.
- Düşülen süre = bakılan gün sayısı / 4 (tam sayı bölmesi, küsurat atılır).
- Bu süre **kalan yaş, kalan sigortalılık ve kalan hizmetin her birinden** düşülür ve emeklilik tarihini öne çeker (KR-41, madde 7).
- Ekranda "düşülen süre" olarak gösterilir.
- **Kodda:** `HakKazanmaService.bakimaMuhtacCocukHizmetiDus`
- **Dayanak:** [5510 sayılı Kanun ilgili madde doğrulanacak]

### KR-43 – Sonuç ekranındaki sabit uyarılar
- "Yazılı talepte bulunduğu tarih itibariyle genel sağlık sigortası primi dahil kendi sigortalılığı nedeniyle prim ve prime ilişkin her türlü borcunun olmaması zorunludur."
- "Bu sonuçlar girdiğiniz verilerin sonucudur. Emeklilik işlemlerinizde Kurum kayıtları esastır. Bilgilendirme amaçlıdır. Resmi işlemlerde kullanılamaz."

---

## 9. Örnek Hesaplama

Ekran görüntülerindeki örnek; kurallar bu örnekle doğrulanmıştır.

**Girdi:** Tam emeklilik, erkek, doğum 01/01/1970, özel durum yok. Tek hizmet: Bağkur, 01/01/2000 – 31/12/2019, 7200 gün. Hesaplama tarihi **12/10/2026**.

| Adım | Kural | Hesap | Sonuç |
| --- | --- | --- | --- |
| Ön kontroller | KR-20, KR-21 | Bağkur var; başlangıç 30.04.2008'den önce, son 2520 günün tamamı Bağkur | Geçti |
| Sigortalılık başlangıcı / bitişi | KR-22, KR-24 | | 01/01/2000 / 31/12/2019 |
| Şartlar | KR-33 | Tam, başlangıç 08.09.1999 – 30.04.2008 arası, erkek | Yaş 60, hizmet 9000, sigortalılık 9000 (25 yıl) |
| Yaşınız | KR-29 | Bitiş geçmişte → 01/01/1970'ten 12/10/2026'ya | 56 yıl 9 ay 11 gün (20441 gün) |
| Sigortalılık süreniz | KR-30 | 01/01/2000'den 12/10/2026'ya = 26 yıl 9 ay 11 gün, +1 | 26 yıl 9 ay 12 gün (9642 gün) |
| Hizmet süreniz | KR-28 | | 7200 gün |
| Kalan yaş | KR-40 | 60 × 360 − 20441 | 1159 gün = 3 yıl 2 ay 19 gün |
| Kalan sigortalılık | KR-40 | 9000 − 9642 < 0 | 0 |
| Kalan hizmet | KR-40 | 9000 − 7200 | 1800 gün |
| Tarih adayları | KR-41 | Bitiş 31/12/2019 · yaş 01/01/2030 · başlangıç + 9000 gün = 01/01/2025 · bugün + 1800 gün = 12/10/2031 | En geç: 12/10/2031 |
| **Emeklilik tarihi** | | | **12/10/2031** |

Bu sonuçlar uygulamanın ekran çıktısıyla birebir aynıdır.

---

## 10. Bilinen Tuhaflıklar ve Mevzuatla Doğrulanacak Noktalar

Kodun yaptığı ama doğruluğu şüpheli olan davranışlar. Yeni sistemde her biri için **"aynen koru"** veya **"düzelt"** kararı verilmeli; düzeltilenler 04'te "eski sistemden farklı sonuç beklenir" olarak işaretlenmelidir.

| # | Kural | Gözlem | Soru |
| --- | --- | --- | --- |
| T-01 | KR-31, KR-35 | 08.09.1999 tam emeklilikte dahil (≤), kısmide hariç (<). Tam bu tarihte başlayan biri iki türde farklı gruba düşüyor. | Hangisi doğru? |
| T-02 | KR-14 | Tam emeklilikte maluliyet/erken yaşlanma muafiyeti kod hatası nedeniyle çalışmıyor (03 → H-04). | Düzeltilsin mi? |
| T-03 | KR-25 | 18 yaş öncesi başlayan kaydın 18 yaş sonrası kısmı da düşülüyor. | Yalnızca 18 yaş öncesi kısmı mı düşülmeli? |
| T-04 | KR-26 | Eşiği kesen kayıtta fiili hizmetin tamamı kanun öncesine sayılıyor. Normalde KR-14/15 bu kaydı böldürüyor; ama muafiyetli durumlarda (maluliyet, erken yaşlanma) bu senaryo oluşabilir. | Kabul edilebilir mi? |
| T-05 | KR-31 | Erkek tablosundaki 60, kadın tablosundaki 58 değerleri 58/56 sınırı nedeniyle hiç kullanılmıyor. | Sınır mevzuata uygun mu? |
| T-06 | KR-31, KR-35 | Kalan ≤ 720 günde yaş şartı tamamen kaldırılıyor; kodda "TODO yaşa bugünkü yaş atanmalı mı?" notu var. Tablonun ilk satırı (kadın 40, erkek 44) bu yüzden kullanılmıyor; kodda "birinci koşul sorulacak" notu var. | Doğru mu? |
| T-07 | KR-34, KR-37 | 5510 kademeli yaş, **sigortalılık bitiş tarihine** (girilen son hizmetin bitişine) göre belirleniyor; şartların tamamlandığı tarihe göre değil. Hizmeti devam eden biri bitiş tarihini bugün girerse 60 yaş tablosu uygulanır. | Mevzuata uygun mu? |
| T-08 | KR-41 | Eksik hizmetin bugünden itibaren kesintisiz tamamlanacağı varsayılıyor; son hizmeti yıllar önce bitmiş biri için de. | Kabul edilen varsayım mı? |
| T-09 | KR-41 | Madde 4 (başlangıç + hizmet şartı) hizmetlerin kesintisiz olduğunu varsayıyor; gerçek eksik madde 5 ile yakalanıyor. | Madde 4 gereksiz mi? |
| T-10 | KR-30 | Sigortalılık süresine her zaman +1 gün ekleniyor. | Dayanağı nedir? |
| T-11 | KR-08, KR-21, KR-22 | Vergi borçlanması ön kontrolde ve son 7 yılda Bağkur sayılıyor, ama sigortalılık başlangıcında gerçek hizmet sayılıp geriye kaydırılmıyor. Askerlik ve yurt dışı borçlanması geriye kaydırılıyor. | Tutarlı mı? |
| T-12 | KR-21 | Son 7 yıl, takvim değil hizmet günüyle geriye sayılıyor; eşitlikte son kurum seçiliyor; 1260 gün kontrolü kapalı. | Doğru mu? |
| T-13 | KR-04 | Doğum tarihi için sınır yok. | Yeni sistemde sınır konacak mı? (ör. 1900 öncesi ve gelecek tarih reddedilsin) |
| T-14 | KR-05, KR-06, KR-17 | Özel durumların birbirini dışlaması, erkekte bakıma muhtaç çocuk ve borçlanmalarda fiili hizmet yalnızca tarayıcıda engelleniyor. | Yeni sistemde sunucuda da kontrol edilecek mi? (Önerilen: evet) |
| T-15 | KR-32 | EYT kodda "TODO 1.EYT_XXX" notlarıyla duruyor. | Güncel mevzuatla uyumlu mu? |
| T-16 | KR-39 | Erken yaşlanma maluliyet şartlarının da üzerine yazıyor; ekran ikisini birlikte seçtirmiyor. | Sunucuda da engellensin mi? |
| T-17 | KR-42 | Düşüş, sıfır olan kalan sürelerden de yapılıyor (sonuç negatif, ekranda 0). | Sorun yok, bilgi amaçlı. |

---

## 11. Mesaj Kataloğu

| Kural | Mesaj | Kaynak |
| --- | --- | --- |
| KR-04, KR-09 | {alan} alanı boş bırakılamaz. | `errors.required` |
| KR-09 | Başlangıç tarihi bitiş tarihinden büyük veya eşit olamaz | `errors.basBitTar` |
| KR-09 | Hizmet süresi geçersiz | `error.hakKazanma.gecersizHizmetSuresi` |
| KR-09 | Fiili hizmet süresi geçersiz | `error.hakKazanma.gecersizFiiliHizmetSuresi` |
| KR-06 | Bakılan toplam gün sayısı geçersiz | `error.hakKazanma.gecersizBakTopGunSayisi` |
| KR-10 | Hizmet başlangıç tarihi doğum tarihinden önce olamaz | Koda gömülü |
| KR-11 | Girmeye çalıştığınız hizmet aralığı girilmiş hizmetlerle çakışmaktadır. | Koda gömülü |
| KR-12 | Hizmet süresi fazla girilmiş | Koda gömülü |
| KR-13 | Fiili hizmet süresi fazla girilmiş. (Hizmet süresinin 1/4'ünden fazla olmamalı.) | Koda gömülü |
| KR-14 | Hizmet aralığını 01/06/2002 öncesi ve sonrası şeklinde ayrı ayrı giriniz | Koda gömülü |
| KR-15 | Hizmet aralığını 01/10/1999 öncesi ve sonrası şeklinde ayrı ayrı giriniz | Koda gömülü |
| KR-16 | Bayanlar içn askerlik borçlanması girilemez | Koda gömülü (yazım hatası aynen) |
| KR-19 | Yurt dışı borçlanma hizmeti tek kalemde girilir | Koda gömülü |
| KR-19 | Ödeme tarihi doğum tarihi ile uyumlu değil. (Borçlanma başlangıç tarihi 18 yaşından önce olamaz) | Koda gömülü |
| KR-20 | Bağkur hizmetinizi giriniz | Koda gömülü |
| KR-21 | Toplam hizmet süresine göre en fazla çalışılan Kurum Bağkur olmadığından 5510 SK. 4/b maddesine göre emekli olamazsınız | Koda gömülü |
| KR-21 | Son 7 yıldaki fiili çalışma süresine göre Bağkurdan emekli olamazsınız. | Koda gömülü |
| – | Hizmetiniz eklenmiştir / güncellenmiştir / silinmiştir | `mesaj.hakKazanma.*` |
| – | Lütfen bir kayıt seçiniz! · Kaydı silmek istediğinizden emin misiniz? | `label.genel.*` |

---

## 12. Test İçin Sınır Değerler

04 Test Senaryoları dokümanında her kural için en az bir tipik ve bir sınır senaryo olmalıdır. Öncelikli sınırlar:

| Kural | Sınır değerler |
| --- | --- |
| KR-01, KR-02 | Ayın 31'inde biten hizmet; 30.09.2008 ve 01.10.2008'de biten hizmet; Şubat sonu |
| KR-11 | Bir kaydın bittiği gün başlayan yeni kayıt (çakışma sayılmamalı) |
| KR-13 | Fiili = hizmet / 4 ve bir gün fazlası |
| KR-14, KR-15 | 01.06.2002 / 01.10.1999'u kesen, eksik süreli ve fiili hizmetli kayıtlar |
| KR-19 | 18 yaşından önceye düşen borçlanma |
| KR-21 | Başlangıç 30.04.2008 ve 01.05.2008; son 2520 günde Bağkur ile SSK eşit; toplam hizmet < 2520 |
| KR-22, KR-23 | Askerlik + yurt dışı borçlanması + Bağkur; vergi borçlanmasıyla başlayan liste |
| KR-25 | 18 yaşından önce başlayan SSK (01.04.1981 öncesi/sonrası) ve Bağkur (04.05.1979 öncesi/sonrası) |
| KR-31 | Başlangıç 07.09.1999, 08.09.1999, 09.09.1999; kalan hizmet 720 / 721; tablo sınırları; 58/56 sınırı |
| KR-32 | Yaşı doldurma tarihi 02.03.2023 ve 03.03.2023; EYT ile emeklilik tarihi 03.03.2023'e çekilen kişi |
| KR-34, KR-37 | Sigortalılık bitişi 31.12.2035 ve 01.01.2036, ... 01.01.2048 |
| KR-35 | Kalan hizmet ile kalan yaştan hangisinin büyük olduğu; 720 / 721 sınırı |
| KR-38 | Üç maluliyet derecesi |
| KR-39 | Erken yaşlanma + tam emeklilik 4447 (yaş 55'e sınırlanması) |
| KR-41 | Sigortalılık bitişi ileri tarihli; emeklilik tarihi sigortalılık bitişine eşit (+1 gün); EYT'de 03.03.2023'e çekilme |
| KR-42 | Bakılan gün 4'ün katı ve katı olmayan (ör. 1000, 1001) |

Hesaplama tarihine bağlı oldukları için (KR-03) tüm senaryolarda hesaplama tarihi kaydedilmeli ve eski sistemde aynı gün çalıştırılmalıdır.
