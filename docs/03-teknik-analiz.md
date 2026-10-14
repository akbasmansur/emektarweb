# 4/b Ne Zaman Emekli Olabilirim? Uygulaması
## 03 – Teknik Analiz ve Kod Haritası

| | |
| --- | --- |
| Doküman sürümü | 0.1 (taslak) |
| Tarih | 13.10.2026 |
| Kaynak | `uygulama.zip` (pom.xml sürümü 0.0.1-SNAPSHOT, uygulama sürümü `SabitLokal.version` = 1.1.0) |
| İlgili dokümanlar | 01 Uygulama Genel Tanımı, 02 İş Kuralları, 04 Test Senaryoları |

Bu doküman mevcut uygulamanın **nasıl yapıldığını** anlatır. Hesaplama kurallarının iş anlamı 02-is-kurallari.md dokümanındadır. Ekran kimlikleri (EK-xx) 01 ile aynıdır.

---

## 1. Özet

- Tek modüllü, küçük bir Struts 1.2.9 web uygulamasıdır: 8 Action, 3 form, 3 iş sınıfı, 4 ekran.
- **Veri tabanı kullanılmaz.** DB2 bağlantısı tanımlı ama hiçbir kod çağırmıyor. Tüm veri HTTP session'da tutulur.
- **Dış sisteme bağlantı yoktur.** Kafka, LDAP, Hibernate gibi bağımlılıklar pom.xml'de var ama kullanılmıyor.
- Hesaplamanın tamamı üç sınıfta: `HakKazanmaService`, `HizmetBilgileriService`, `EmeklilikSartlari`. Tarih aritmetiği `GetTarih` sınıfında, **1 yıl = 360 gün, 1 ay = 30 gün** kabulüyle yapılır.
- Doğrulama üç katmana dağılmış: `validation.xml`, form sınıflarındaki Java kodu ve JSP içindeki JavaScript.

## 2. Teknik Ortam

| Bileşen | Mevcut | Sürüm / Not |
| --- | --- | --- |
| Dil | Java | 1.8 |
| Web çatısı | Struts | 1.2.9 (`*.do`, tek struts-config) |
| Sayfa yerleşimi | Struts Tiles | `tiles-defs.xml` |
| Arayüz | JSP + Struts taglib + kurum içi `emektar` taglib | JSP'ler ISO-8859-9 kodlamalı |
| Liste tablosu | displaytag | Kurum içi sürüm `tr.gov.sgk.emektar4b.util:displaytag:1.2.9` |
| Doğrulama | Struts Validator (Commons Validator) | `validation.xml`, `validator-rules.xml` |
| Sunucu | Open Liberty | `liberty-maven-plugin` 3.12.0; servlet-4.0, jsp-2.3 |
| Derleme | Maven | `war` paketleme, `local-test` / `local-prod` profilleri |
| Loglama | Log4j2 (SLF4J üzerinden) | Ortama göre `log4j2_Local/Test/Prod.xml` |
| Veri tabanı | DB2 (`jdbc/EMEKTAR_DB`) | Tanımlı, **kullanılmıyor** |
| JVM ayarı | `Europe/Istanbul`, `tr_TR`, UTF-8 | `jvm.options` |

Context root: `/EMEKTARWEB`. Uygulama ortamı `UYG_PROFIL` ortam değişkeninden okunur (`LOKAL`, `TEST`, `PROD`).
<div style="page-break-before: always;"></div>

## 3. Proje Yapısı

```
src/main/java/tr/gov/sgk/emektarWeb/
├── emeklilikHakKazanma/
│   ├── action/       8 Struts Action (her ekran işlemi için bir sınıf)
│   ├── baseForm/     EmeklilikBilgileriForm, HizmetBilgileriForm, HakKazanmaSonucForm
│   └── business/     HakKazanmaService, HizmetBilgileriService, EmeklilikSartlari
└── genel/
    ├── comparator/   DateComparator (liste sıralama)
    ├── decorator/    HizmetTuruDecorator, DateDecorator (displaytag)
    ├── filter/       SecureCookieFilter, HealthCheckFilter, SetCharacterEncodingFilter, Log4jInitListener
    ├── resource/     Mesajlar.properties
    ├── taglib/       AddButton, EditButton, DeleteButton ve diğer buton tag'leri
    └── util/         GetTarih, Sabit, SabitLokal, Arac, ValidationUtil, KarakterKontrolleri, DBBroker
src/main/webapp/
├── WEB-INF/config/   struts, tiles, validation, tld
├── WEB-INF/jsp/emeklilikHakKazanma/   ekran JSP'leri
├── layouts/sgkTemaUygulama/           sayfa şablonu (header, footer, menü, mesajlar)
└── js/               tablo.js, tarih maskesi ve eski kütüphaneler
src/main/liberty/config/  server.xml, server.env, bootstrap.properties, jvm.options
```

## 4. İstek Akışı

```
Tarayıcı ──GET/POST *.do──▶ ActionServlet (Struts)
                               │
                               ├─ Filtreler: SetCharacterEncoding, SecureCookie, displaytag ResponseOverride
                               ├─ ActionForm doldurulur, validate="true" ise doğrulanır
                               │     hata → input ekranı + messages.jsp
                               ▼
                         Action.execute()
                               │  session'dan veri okur / yazar
                               │  gerekirse HakKazanmaService'i çağırır
                               ▼
                    forward / redirect ──▶ Tiles tanımı ──▶ layout.jsp + ekran JSP'si
```

Uygulamada servis katmanı ince; iş kurallarının bir kısmı doğrudan Action ve form sınıflarında (bkz. bölüm 6 ve 8).

## 5. Ekran – Action Haritası

Bütün ekranlar `content4` Tiles şablonunu kullanır (`/layouts/sgkTemaUygulama/layout.jsp`); yalnızca içerik JSP'si değişir. Giriş adresi `index.jsp`'dir, `welcome` global forward'ı ile `/hakKazanma.do`'ya yönlendirir.

| ID | Ekran | JSP | Form bean |
| --- | --- | --- | --- |
| EK-01 | Emeklilik bilgileri girişi | `emeklilikBilgileri.jsp` | EmeklilikBilgileriForm<br>(validate=true) |
| EK-02 | Hizmet bilgileri ve sonuç | `hizmetBilgileri.jsp` | – |
| EK-03 | Hizmet ekleme | `hizmetBilgileriEkle.jsp` | HizmetBilgileriForm<br>(validate=true) |
| EK-04 | Hizmet güncelleme | `hizmetBilgileriGuncelle.jsp` | HizmetBilgileriForm<br>(validate=true) |

| ID | Ekranı açan action | Ekrandan gönderilen action | Sonraki ekran |
| --- | --- | --- | --- |
| EK-01 | `/hakKazanma.do`<br>(ForwardAction) | `/EmeklilikBilgileriAction.do`<br>(TAMAM) | Başarılı: EK-02<br>Hata: EK-01 |
| EK-02 | `/EmeklilikBilgileriAction.do`<br>`/HizmetBilgileriListeleAction.do`<br>`/EmeklilikHesaplaAction.do` | Ekle<br>Güncelle<br>Sil<br>Emeklilik Hesapla | EK-03, EK-04<br>veya EK-02 |
| EK-03 | `/HizmetBilgileriEkleSayfaAction.do` | `/HizmetBilgileriEkleAction.do` (Kaydet)<br>Listeye Dön | Başarılı: EK-02 (redirect)<br>Hata: EK-03 |
| EK-04 | `/HizmetBilgileriGuncelleSayfaAction.do` | `/HizmetBilgileriGuncelleAction.do` (Güncelle)<br>Listeye Dön | Başarılı: EK-02 (redirect)<br>Hata: EK-04 |

### EK-01 · Emeklilik bilgileri girişi
- **JSP:** `emeklilikBilgileri.jsp`
- **Açan action:** `/hakKazanma.do` (ForwardAction)
- **Gönderilen action:** `/EmeklilikBilgileriAction.do` (TAMAM)
- **Form bean:** EmeklilikBilgileriForm, validate=true
- **Sonraki ekran:** Başarılı → EK-02 · Hata → EK-01


**Action listesi**

| Action | Sınıf | Ne yapar | Session'a etkisi | Sonuç |
| --- | --- | --- | --- | --- |
| `/hakKazanma` | `org.apache.struts.actions.ForwardAction` | EK-01'i açar | – | EK-01 |
| `/EmeklilikBilgileriAction` | EmeklilikBilgileriAction | EK-01 formunu kaydeder | Formu yazar, hizmet listesini **sıfırlar** | EK-02 (forward) |
| `/HizmetBilgileriListeleAction` | HizmetBilgileriListeleAction | Listeyi ekrana hazırlar | Listeyi request'e `Sabit.LISTE` olarak kopyalar | EK-02 (forward) |
| `/HizmetBilgileriEkleSayfaAction` | HizmetBilgileriEkleSayfaAction | EK-03'ü açar; 3201'de ikinci kaydı engeller | – | EK-03 veya hata ile listeye |
| `/HizmetBilgileriEkleAction` | HizmetBilgileriEkleAction | Yeni hizmeti listeye ekler | `siraNo = liste boyutu + 1` ile ekler | EK-02 (redirect) |
| `/HizmetBilgileriGuncelleSayfaAction` | HizmetBilgileriGuncelleSayfaAction | `siraNo`'lu kaydı forma koyar | – | EK-04 |
| `/HizmetBilgileriGuncelleAction` | HizmetBilgileriGuncelleAction | Eski kaydı silip yenisini ekler | Listeyi değiştirir | EK-02 (redirect) |
| `/HizmetBilgileriSilAction` | HizmetBilgileriSilAction | `siraNo`'lu kaydı siler | Listeyi değiştirir | EK-02 (redirect) |
| `/EmeklilikHesaplaAction` | EmeklilikHesaplaAction | Ön kontroller + hesaplama | Listeyi **sıralar ve borçlanma tarihlerini değiştirir** (bkz. 12) | EK-02 (forward), sonuç request'te |

## 6. Formlar ve Doğrulama

Doğrulama üç yerde yapılır; yeni sistemde hepsinin karşılığı olmalıdır.

### 6.1 Form alanları

**EmeklilikBilgileriForm** (`ValidatorForm`)

| Alan | Tür | Değerler (Sabit) |
| --- | --- | --- |
| emeklilikTuru | String | `1` Tam, `0` Kısmi |
| cinsiyet | String | `1` Erkek, `0` Kadın |
| dogumTarihi | String `dd/MM/yyyy` | `.` ve `-` otomatik `/` yapılır |
| maluliyetVarmi | String | `1` Hayır, `2` Evet (boşsa Hayır) |
| maluliyetDerecesi | String | `1` %40-49, `2` %50-59, `3` %60 ve üzeri |
| erkenYaslanmaVarmi | String | `1` Hayır, `2` Evet (boşsa Hayır) |
| bakMuhCoc | boolean | Bakıma muhtaç malül çocuk |
| bakGunSayisi | String | 01.10.2008 sonrası bakılan gün sayısı (yalnızca kadın ve bakMuhCoc seçiliyse görünür) |
| sadece3201 | boolean | Sadece yurt dışı borçlanması |

**HizmetBilgileriForm** (`ValidatorForm`, `Comparable` – başlangıç tarihine göre sıralanır)

| Alan | Tür | Not |
| --- | --- | --- |
| siraNo | Integer | Liste içi sıra numarası, kalıcı kimlik değil |
| hizmetTuru | Integer | `1` Bağkur, `2` SSK, `3` Emekli Sandığı, `4` Diğer Sandıklar, `5` Yurt dışı borçlanması, `6` Askerlik borçlanması, `7` Vergi borçlanması |
| baslangicTarihi, bitisTarihi | String `dd/MM/yyyy` | 3201 modunda başlangıç gizli alanla `01/01/1900` gönderilir, sunucuda hesaplanır |
| hizmetSuresi | String (gün) | Ekranda JavaScript ile otomatik doldurulur, elle değiştirilebilir |
| fiiliHizmetSuresi | String (gün) | İsteğe bağlı; borçlanma türlerinde kapatılır |

**HakKazanmaSonucForm**: yaş, sigortalılık süresi, hizmet süresi, şartlar (`EmeklilikSartlari`), kalan süreler, emeklilik tarihi, bakıma muhtaç çocuk için düşülen süre. Tüm süreler gün olarak tutulur, ekranda 360/30 kabulüyle yıl-ay-güne çevrilir.

### 6.2 validation.xml kuralları

| Form | Alan | Kural | Mesaj anahtarı |
| --- | --- | --- | --- |
| EmeklilikBilgileriForm | emeklilikTuru, cinsiyet, dogumTarihi | required | `errors.required` |
| EmeklilikBilgileriForm | maluliyetDerecesi | requiredif maluliyetVarmi = 2 | `errors.required` |
| EmeklilikBilgileriForm | bakGunSayisi | requiredif bakMuhCoc = true; boş ya da > 0 | `error.hakKazanma.gecersizBakTopGunSayisi` |
| HizmetBilgileriForm | baslangicTarihi | required | `errors.required` |
| HizmetBilgileriForm | bitisTarihi | required, dateRange (bitiş > başlangıç) | `errors.basBitTar` |
| HizmetBilgileriForm | hizmetSuresi | required, > 0 | `error.hakKazanma.gecersizHizmetSuresi` |
| HizmetBilgileriForm | fiiliHizmetSuresi | boş ya da > 0 | `error.hakKazanma.gecersizFiiliHizmetSuresi` |

Doğum tarihi için **biçim veya aralık kontrolü yoktur** (ör. 10/01/1000 kabul edilir). Tarih alanlarında yalnızca tarayıcıdaki giriş maskesi (`InputMask.js`) var.

### 6.3 Java doğrulamaları (`HizmetBilgileriForm.validate`)

Normal mod (sadece3201 = false):

| Metot | Kontrol | Mesaj (koda gömülü) |
| --- | --- | --- |
| dogumTarihiniKontrolEt | Başlangıç doğum tarihinden sonra olmalı | Hizmet başlangıç tarihi doğum tarihinden önce olamaz |
| hizmetlerCakisiyormu | Listedeki diğer kayıtlarla tarih çakışması olmamalı | Girmeye çalıştığınız hizmet aralığı girilmiş hizmetlerle çakışmaktadır. |
| girilenHizmetSuresiniKontrolEt | Süre, tarih farkını (01.10.2008 ve sonrası bitişte +1) aşmamalı | Hizmet süresi fazla girilmiş |
| fiiliHizmetSuresiniKontrolEt | Fiili hizmet ≤ hizmet süresinin 1/4'ü | Fiili hizmet süresi fazla girilmiş… |
| tamEmeklilikHizmetKontrolEt | Tam emeklilikte, 01.06.2002'yi kesen eksik süreli veya fiili hizmetli kayıt bölünmeli | Hizmet aralığını 01/06/2002 öncesi ve sonrası şeklinde ayrı ayrı giriniz |
| kismiEmeklilikHizmetKontrolEt | Kısmi emeklilikte aynı kontrol 01.10.1999 için | Hizmet aralığını 01/10/1999 öncesi ve sonrası… |
| askerlikBorclanmasiKontrol | Kadın için askerlik borçlanması girilemez | Bayanlar içn askerlik borçlanması girilemez |
| Yas18AltiHizmetKontroluYap | 18 yaş altı hizmet kontrolü | **Kapatılmış** (jazz:74126, 28.05.2020) |

3201 modu: başlangıç tarihi `bitiş − süre` olarak hesaplanır, 18 yaşından önceyse 18 yaş tarihine çekilir; doğum tarihi kontrolü ve "başlangıç ≥ bitiş" kontrolü yapılır ("Ödeme tarihi doğum tarihi ile uyumlu değil…").

### 6.4 Action içindeki kontroller

| Yer | Kontrol | Mesaj |
| --- | --- | --- |
| HizmetBilgileriEkleSayfaAction | 3201 modunda listede kayıt varsa ekleme açılmaz | Yurt dışı borçlanma hizmeti tek kalemde girilir |
| EmeklilikHesaplaAction | (3201 değilse) Bağkur veya vergi borçlanması kaydı olmalı | Bağkur hizmetinizi giriniz |
| EmeklilikHesaplaAction | (3201 değilse) `getSon7YilKurum` Bağkur dönmeli | 5510 SK. 4/b'ye göre emekli olamazsınız / Son 7 yıldaki fiili çalışmaya göre… |

### 6.5 Ekrandaki JavaScript kuralları

| Ekran | Fonksiyon | Davranış |
| --- | --- | --- |
| EK-01 | maluliyetKontrol | Maluliyet Evet → derece açık; erken yaşlanma ve 3201 kapalı |
| EK-01 | erkenYaslanmaKontrol | Erken yaşlanma Evet → maluliyet, derece ve 3201 kapalı |
| EK-01 | sadece3201Kontrol | 3201 seçili → maluliyet, erken yaşlanma, bakıma muhtaç çocuk kapalı |
| EK-01 | bakimaMuhtacKontrol | Erkek seçilirse çocuk kutusu temizlenir; kadın + kutu işaretli → gün sayısı satırı açılır, 3201 kapanır |
| EK-03/04 | fiiliHizmetKontrolu | Borçlanma türlerinde (5, 6, 7) fiili hizmet alanı kapanır ve temizlenir |
| EK-03/04 | gunFarkiniBul | Tarih alanından çıkınca hizmet süresini 360/30 kuralıyla hesaplar; bitiş 01.10.2008 ve sonrasıysa +1 gün |

Bu kuralların hiçbiri sunucuda tekrar kontrol edilmez (ör. erkek için bakıma muhtaç çocuk sunucuda engellenmez, sadece hesaplamada etkisizdir).

## 7. Durum Yönetimi

| Anahtar | Kapsam | İçerik | Yazan | Okuyan |
| --- | --- | --- | --- | --- |
| `EmeklilikBilgileriForm` (`SESSION_EMEKLILIKBILGILERI`) | session | EK-01 formu | EmeklilikBilgileriAction | Tüm hizmet action'ları, form doğrulaması, hesaplama, EK-02 |
| `hizmetListesi` (`SESSION_HIZMETLISTESI`) | session | `ArrayList<HizmetBilgileriForm>` | EmeklilikBilgileri, Ekle, Güncelle, Sil | Listele, Hesapla, doğrulama, EK-02 |
| `_LISTE_` (`Sabit.LISTE`) | request | Listenin kopyası (displaytag için) | HizmetBilgileriListeleAction | EK-02 tablo |
| `HakKazanmaSonucForm` | request | Hesaplama sonucu | EmeklilikHesaplaAction | EK-02 sonuç bölümü |

Session süresi 20 dakikadır (`web.xml`). Session dolarsa action'lar `null` nesneye eriştiği için hata alır; `global-exceptions` boş olduğundan sunucunun genel hata sayfası görünür.

## 8. Hesaplama Akışı

`EmeklilikHesaplaAction.execute()` içinden çağrı sırası:

```
1. Ön kontroller (sadece3201 değilse)
   ├─ HizmetBilgileriService.bagkurHizmetiVarmi(liste)
   └─ HizmetBilgileriService.getSon7YilKurum(liste, dogumTarihi, sadece3201)
        ├─ getSBaslangicTarihi()  → listeyi sıralar, borçlanmaları geri kaydırır (hizmetDuzelt)
        ├─ getSBitisTarihi()      → son kaydın bitiş tarihi
        └─ başlangıç > 30.04.2008 ise tüm hizmet, değilse getSon7YilinBasladigiTarih() ile son 2520 gün
2. HakKazanmaService.hakKazanmaSorgula(form, liste)  → EmeklilikSartlari
   ├─ getYas18OncesiHizmet()      (18 yaş öncesi SSK/Bağkur hizmeti)
   ├─ getKanunOncesiHizmet()      (Tam: 01.06.2002, Kısmi: 01.10.1999 öncesi hizmet, fiili dahil)
   ├─ getSBaslangicTarihi(), getSBitisTarihi()
   ├─ get_01_10_1999_YasiGunOlarak()
   └─ new EmeklilikSartlari(...)  → yaş, hizmet ve sigortalılık şartı
        ├─ tamEmeklilikGecis4447() / kismiEmeklilikGecis4447()   (08.09.1999 öncesi girişler)
        ├─ tamEmeklilikGecis5510() / kismiEmeklilikGecis5510()   (01.05.2008 ve sonrası girişler)
        ├─ maluliyet derecesine göre süreler
        ├─ erken yaşlanma sınırı
        └─ EYT kapsamı (03.03.2023)
3. HakKazanmaService.getHakKazanmaSonuc(sartlar, form, liste)  → HakKazanmaSonucForm
   ├─ getSigortaliYas()           (bugüne veya ileri tarihli bitişe göre)
   ├─ getSigortalilikSuresi()     (bugün 01.10.2008'den sonraysa +1 gün)
   ├─ getToplamHizmet()           (hizmet + fiili hizmet)
   ├─ emeklilikSartlariKarsilastir()  (EYT yaş indirimi, kalan süreler)
   ├─ emeklilikTarihiHesapla()        (yaş, sigortalılık, hizmet tarihlerinin en geçi; EYT ve bakıma muhtaç çocuk düzeltmeleri)
   └─ bakimaMuhtacCocukHizmetiDus()   (kadın: bakılan gün / 4 kadar düşüş)
4. Sonuç request'e konur, EK-02'ye forward edilir.
```

Hesaplama **sistem tarihine** bağlıdır (`new Date()`, `GetTarih.getTarihString()`); aynı girdi farklı günde farklı sonuç verir. Her kuralın iş anlamı ve eşik değerleri 02-is-kurallari.md'de ayrıntılanacak.

### 8.1 Tarih aritmetiği (`GetTarih`)

| Metot | Ne yapar | Dikkat |
| --- | --- | --- |
| getDateDifferenceForBagkur(d1, d2) | İki tarih arasını gün olarak verir | 1 yıl = 360, 1 ay = 30 gün; ayın 31'i 30 sayılır |
| getDateDifferenceInYilAyGun | Fark yıl-ay-gün dizisi | Gün > 30 ise 30'a çekilir |
| tarihinUstuneYilAyGunEkle | Tarihe yıl/ay/gün ekler | 30 günlük ay kabulüyle; ay sonlarında takvimden sapar |
| tarihinUstuneYilAyGunCikart | Tarihten yıl/ay/gün çıkarır | Aynı kabul |
| getYilAyGunArray (HizmetBilgileriService) | Gün → [yıl, ay, gün] | 360/30 kabulü |
| tarihAraliklariKesisiyormu | Aralık çakışması | Aynı gün biten/başlayan aralıklar çakışmaz sayılır |
| convertStringToDate | `dd/MM/yyyy` ayrıştırır | Hatada sessizce `null` döner |

Bu davranış Java 8 `LocalDate`/`Period` ile birebir aynı **değildir**. Yeni sistemde 360/30 aritmetiği ayrı bir yardımcı sınıf olarak aynen taşınmalı ve testlerle doğrulanmalıdır.

## 9. Ön Yüz Davranışları

- **Satır seçimi:** `display:table` (`trClick="true"`, `trClickParams="siraNo"`) satıra `sec(this, 'siraNo=N')` ekler. `tablo.js` içindeki `sec()` satırı `selected` sınıfıyla boyar, parametreyi `sEleman`'a yazar.
- **Butonlar:** Kurum içi tag'ler `submitjs(hedef, silmeIslemi, seciliOlmali)` üretir.

| Tag | Parametreler | Kullanıldığı yer |
| --- | --- | --- |
| AddButton | (false, false) | Hizmet Ekle, Emeklilik Hesapla, Listeye Dön |
| EditButton | (false, true) | Hizmet Güncelle |
| DeleteButton | (true, true) | Hizmet Sil |

- Seçim gerektiren butonlarda seçim yoksa `alert(label.genel.kayitSeciniz)`, silmede `confirm(label.genel.silinecek)`. İstek `window.location` ile **GET** olarak gider: `HizmetBilgileriSilAction.do?dispatch=…&siraNo=N`. `dispatch` parametresi Action'lar DispatchAction olmadığı için kullanılmaz.
- "Emeklilik Hesapla" butonu `logic:notEmpty` ile yalnızca session listesi boş değilse gösterilir.
- Sonuç bölümü `logic:present name="HakKazanmaSonucForm"` ile yalnızca hesaplama isteğinde görünür.
- Tablo varsayılan olarak başlangıç tarihine göre sıralanır (`defaultsort="2"`, `DateComparator`), sayfa başına 30 kayıt.

## 10. Veri Tabanı ve Dış Bağımlılıklar

**Veri tabanı:** `server.xml`'de `jdbc/EMEKTAR_DB` (DB2) tanımlı, `DBBroker` sınıfı bağlantı açabiliyor; ancak `DBBroker` hiçbir yerden çağrılmıyor, SQL sorgusu yok. **Yeni sistemde veri tabanı gerekmez.**

**pom.xml'de olup kodda kullanılmayan bağımlılıklar**

| Bağımlılık | Durum |
| --- | --- |
| hibernate-core 3.6.9, ehcache 1.2.3, javassist | Kullanılmıyor |
| kafka-clients 3.7.1, visolog 3.0.3 | Kodda ve log yapılandırmasında kullanım yok |
| commons-httpclient 2.0.2 | Yalnızca `DateParseException` sınıfı için import ediliyor |
| com.ibm.db2:jcc | Veri tabanı kullanılmadığı için gereksiz |
| itext 2.1.7, poi 3.0.1, displaytag-export-poi | displaytag dışa aktarma için; ekranda dışa aktarma kapalı (`export="false"`) |
| javax.jws-api | Kullanılmıyor |
| junit 3.8.1 | Test yok (`src/test` boş) |

LDAP ve veri tabanı ortam değişkenleri (`server.env`) tanımlı ama kodda okunmuyor.

## 11. Yapılandırma ve Çalışma Ortamı

| Dosya | İçerik |
| --- | --- |
| `web.xml` | ActionServlet (`*.do`), filtreler, session 20 dk, welcome `index.jsp` |
| `hakKazanma-struts-config.xml` | Form bean'ler, action'lar, Tiles ve Validator eklentileri, `Mesajlar` kaynak dosyası |
| `tiles-defs.xml` | `content4` şablonu ve 4 ekran tanımı |
| `server.xml` | Liberty özellikleri, HTTP/HTTPS uç noktası, SSL, DB2 datasource |
| `bootstrap.properties` | Portlar, keystore parolası |
| `server.env` | `UYG_PROFIL`, LDAP ve DB değişkenleri |
| `log4j2_*.xml` | Ortama göre log yapılandırması (`Log4jInitListener` seçer) |

**Filtreler**

| Filtre | Ne yapar |
| --- | --- |
| SetCharacterEncodingFilter | İstek kodlamasını UTF-8 yapar |
| SecureCookieFilter | Güvenli cookie başlığı eklemeye çalışır (bkz. 13) |
| ResponseOverrideFilter | displaytag dışa aktarma için |
| HealthCheckFilter | `/health/liveness` ve `/health/readiness` uçları (`{"status":"UP"}`) |

## 12. Kullanılmayan Kod ve Dosyalar

| Dosya / sınıf | Neden kullanılmıyor |
| --- | --- |
| `DBBroker`, `ListService` (boş sınıf) | Çağrılmıyor |
| `emeklilikBilgileri2.jsp`, `hizmetBilgileriE.jsp`, `hizmetBilgileriEkle2.jsp`, `hizmetBilgileriGuncelle2.jsp` | Hiçbir Tiles tanımında yok |
| `tiles-defsEE.xml`, `layouts/emeklilikHakKazanma/*`, `layout2.jsp` | Yüklenmiyor |
| `tiles/istatistikAylikBaglama/tiles-defs.xml` | Yükleniyor ama işaret ettiği JSP ve action yok; başka projeden kalma |
| `HakKazanmaService.ilkGercekHizmetMi` | 2013-26 sayılı genelgeyle devre dışı (her zaman `true` kullanılıyor) |
| `HizmetBilgileriForm.Yas18AltiHizmetKontroluYap` | 2020'de kapatılmış |
| `Sabit` içindeki banka, tahsis, KPS, aylık durum sabitleri; `Mesajlar` içindeki banka değişikliği mesajları | Başka uygulamalardan kalma |
| `IntegerTag`, `ResetButtonTag`, `SubmitButtonTag` | Ekranlarda kullanılmıyor |
| `js/` altındaki çoğu kütüphane (prototype, window, calendar, jquery 1.2.6) | Şablonda yükleniyor ama ekranlarda işlevi yok |

## 13. Bilinen Hatalar ve Riskler

| # | Konu | Ayrıntı | Etki |
| --- | --- | --- | --- |
| H-01 | Tekrarlayan `siraNo` | `siraNo = liste boyutu + 1`; silme sonrası eklemede aynı numaralı iki kayıt oluşabilir. Güncelle ve Sil yalnızca ilkini bulur. | Yanlış kayıt güncellenir/silinir |
| H-02 | `siraNo` doğrulanmıyor | Parametre yoksa `NumberFormatException`; olmayan numara gelirse Sil yine "silindi" der. | Hata sayfası, yanlış mesaj |
| H-03 | Hesaplama session verisini değiştiriyor | `getSBaslangicTarihi` listeyi yerinde sıralar; askerlik/yurt dışı borçlanması varsa `hizmetDuzelt` bu kayıtların başlangıç ve bitiş tarihlerini değiştirir. | Hesaplamadan sonra listede kullanıcının girmediği tarihler görünür |
| H-04 | Tam emeklilikte özel durum muafiyeti çalışmıyor | `tamEmeklilikHizmetKontrolEt` `String` değeri `Integer` ile karşılaştırıyor (`getMaluliyetVarmi().equals(Sabit.EVET)`), sonuç hep `false`. Kısmi emeklilikteki benzer kontrol doğru yazılmış. | Malul/erken yaşlanan kişiden de 01.06.2002 bölmesi isteniyor |
| H-05 | Session süresi dolması | Action'lar session nesnelerini kontrolsüz okur; `global-exceptions` boş. | Genel sunucu hata sayfası |
| H-06 | Doğum tarihi sınırı yok | `required` dışında kontrol yok; 10/01/1000 kabul ediliyor. | Anlamsız sonuçlar |
| H-07 | Bozuk cookie başlığı | `SecureCookieFilter` `Set-Cookie` başlığını cookie adı ve değeri olmadan yazıyor (`; Path=/EMEKTARWEB; …`). | Güvenli cookie amacı gerçekleşmiyor |
| H-08 | Thread güvenliği | `Sabit.dateFormat` paylaşılan `SimpleDateFormat`; `DateDecorator` kullanıyor. | Eşzamanlı isteklerde nadir hatalı tarih |
| H-09 | Sessiz hata yutma | `convertStringToDate` hatada `null`, `compareTo` hatada `3` döner; çoğu `catch` boş. | Hatalı tarih girdisi sonradan `NullPointerException` üretebilir |
| H-10 | Silme GET ile yapılıyor | Durum değiştiren işlem URL ile tetikleniyor, CSRF koruması yok. | Güvenlik |
| H-11 | Kod tekrarı | Hizmet süresi hesabı hem `gunFarkiniBul` (JS) hem Java'da; 01.10.2008 +1 gün kuralı iki yerde ayrı yazılmış. | Tutarsızlık riski |
| H-12 | Bakıma muhtaç çocuk düşüşü | Kalan sürelerden düşüş sıfırın altına inebilir; ekranda getter'lar 0 gösteriyor. | Görsel olarak sorun yok, veri negatif |

## 14. Spring Boot Karşılıkları ve Geçiş Notları

| Mevcut | Spring Boot karşılığı |
| --- | --- |
| Action + action-mapping | `@Controller` + `@GetMapping` / `@PostMapping` |
| ActionForm (`ValidatorForm`) | DTO + Bean Validation (`@NotNull`, `@Positive`, özel `@ValidDateRange`) |
| Form `validate()` içindeki Java kuralları | Servis katmanında doğrulayıcı sınıf veya özel `ConstraintValidator` |
| Session nesneleri | `@SessionAttributes` veya `@SessionScope` bean |
| Tiles şablonu | Thymeleaf layout (veya JSP + ortak şablon) |
| displaytag + `tablo.js` + `emektar` tag'leri | Thymeleaf tablo + satır seçimi için radio/buton, POST formları |
| `Mesajlar.properties` + koda gömülü mesajlar | `messages.properties` (`MessageSource`) |
| `GetTarih` 360/30 aritmetiği | Ayrı, birim testli `BagkurTarihHesap` sınıfı |
| `HakKazanmaService`, `EmeklilikSartlari` | `@Service` sınıfları; durum değiştirmeyen, saf fonksiyonlar |
| HealthCheckFilter | Spring Boot Actuator (`/actuator/health`) |
| Log4jInitListener + ortam dosyaları | `application-{profil}.yml` + Logback/Log4j2 |
| Open Liberty `server.xml` | Gömülü Tomcat + `application.yml` |
| DB2 datasource | Gerekmiyor |

**Önerilen yapı**

1. Hesaplamayı önce Spring'den bağımsız saf Java olarak taşı (`EmeklilikSartlari`, `HakKazanmaService`, `HizmetBilgileriService`, `GetTarih`'in kullanılan metotları). 04 dokümanındaki senaryolarla `@ParameterizedTest` yaz.
2. **Hesaplama tarihi parametre olsun.** `new Date()` yerine `Clock` veya `LocalDate bugun` parametresi; testler sabit tarihle çalışsın.
3. Hesaplama girdi listesini değiştirmesin (H-03); kopyası üzerinde çalışsın.
4. `siraNo` yerine benzersiz kimlik (UUID veya artan sayaç) kullan (H-01, H-02).
5. Silme ve güncelleme POST + CSRF korumalı olsun (H-10).
6. JavaScript'teki alan kuralları sunucuda da uygulansın (bölüm 6.5).
7. Kullanılmayan bağımlılık ve dosyalar taşınmasın (bölüm 10, 12).
8. Bilinen hatalar (bölüm 13) için "aynen koru / düzelt" kararı verilsin; düzeltilenler 04'te eski sistemden farklı sonuç beklenen senaryolar olarak işaretlensin.

## 15. Açık Sorular

- [ ] H-04 gibi hatalar yeni sistemde düzeltilecek mi, yoksa sonuçların birebir aynı olması için korunacak mı?
- [ ] EYT kuralı (03.03.2023) kodda "TODO" notlarıyla duruyor; güncel mevzuatla uyumlu mu?
- [ ] `getSigortalilikSuresi` içindeki "bugün 01.10.2008'den sonraysa +1 gün" kuralının mevzuat dayanağı nedir?
- [ ] `tamEmeklilikGecis4447` içinde "TODO: birinci koşul sorulacak" notu var; kural doğrulandı mı?
- [ ] Yeni sistemde arayüz Thymeleaf mi olacak, yoksa ön yüz ayrı (REST + SPA) mı yazılacak?
- [ ] `/health/*` uçları Kubernetes tarafından kullanılıyor mu? Actuator yollarına geçiş için altyapı ekibiyle konuşulmalı.
