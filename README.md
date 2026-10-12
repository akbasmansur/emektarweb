# 1. Genel Tanım

| Alan | Bilgi |
|---|---|
| **Uygulama adı** | 4/b Ne Zaman Emekli Olabilirim? |
| **Doküman sürümü** | 0.1 (taslak) |
| **Hazırlayan** | Xxxxx YYYYYYYY |
| **Kapsam** | Mevcut sistemin davranışı; yeni sistem tasarımı bu dokümanda yer almaz. |
| **İlgili dokümanlar** | 2. İş Kuralları, 3. Kod Haritası, 4. Veri Tabanı, 5. Test Senaryoları |

## Uygulamanın Amacı

5510 4/1-b kapsamında sigortalı olan kişilerin; sigortalılık başlangıç tarihi, doğum tarihi, cinsiyet, prim ödeme gün sayısı ve ilgili mevzuatta belirtilen diğer koşullarını değerlendirerek, yaşlılık aylığına hak kazanabilecekleri tarihi hesaplayan yazılım uygulamasıdır.

### Çözdüğü İş Problemi

Vatandaşın emeklilik tarihini elle hesaplama ihtiyacını ortadan kaldırmak.

### Kapsadığı Statüler

5510 4/1-b kapsamında olanlar.

### Dayandığı Mevzuat

5510, 1479, 2926.

---

# 2. Kullanıcılar ve Roller

| Rol | Kim? | Ne yapabilir? | Giriş şekli |
|---|---|---|---|
| **Vatandaş** | 5510 4/1-b kapsamında sigortalı olan kişi | Sigortalılık başlangıç tarihi, doğum tarihi, cinsiyet, prim ödeme gün sayısı ve gerekli diğer bilgileri girerek yaşlılık aylığına hak kazanabileceği tarihi hesaplayabilir ve sonucu görüntüleyebilir. | Giriş yok |

---

# 3. Ekran listesi

Uygulamada 4 ekran ve bunlardan tetiklenen 9 action vardır (`hakKazanma-struts-config.xml`, Struts 1.2). Tüm ekranlar `content4` Tiles şablonunu kullanır (`/layouts/sgkTemaUygulama/layout.jsp`: header, navigation, content, footer, messages); yalnızca content kısmı ekrana göre değişir. Hesaplama sonucu ayrı bir ekranda değil, Hizmet Bilgileri ekranının (EK-02) alt kısmında gösterilir.

| Ekran ID | Ekran adı | JSP (Tiles tanımı) | Ekranı açan action | Ekrandan gönderilen action (buton) | Form bean | Ne yapılır | Sonraki ekran |
| --- | --- | --- | --- | --- | --- | --- | --- |
| EK-01 | Emeklilik bilgileri girişi | `emeklilikBilgileri.jsp` | `/hakKazanma.do` (ForwardAction, açılış sayfası) | `/EmeklilikBilgileriAction.do` (TAMAM) | EmeklilikBilgileriForm (validate=true) | Emeklilik türü, cinsiyet, doğum tarihi ve özel durumlar (maluliyet, erken yaşlanma, bakıma muhtaç malül çocuk, yurt dışı borçlanması) girilir | Başarılı: EK-02 · Hata: EK-01 |
| EK-02 | Hizmet bilgileri ve hesaplama sonucu | `hizmetBilgileri.jsp` | `/EmeklilikBilgileriAction.do`, `/HizmetBilgileriListeleAction.do`, `/EmeklilikHesaplaAction.do` (forward) | Hizmet Ekle, Hizmet Güncelle, Hizmet Sil, Emeklilik Hesapla | – | EK-01 bilgileri salt okunur özetlenir, hesaplamadan sonra sigortalılık bilgileri, emeklilik şartları, kalan süreler ve emeklilik tarihi aynı sayfanın altında gösterilir | EK-03, EK-04 veya yine EK-02 |
| EK-03 | Hizmet bilgisi ekleme | `hizmetBilgileriEkle.jsp` | `/HizmetBilgileriEkleSayfaAction.do` | `/HizmetBilgileriEkleAction.do` (Kaydet); Listeye Dön | HizmetBilgileriForm (validate=true) | Hizmet türü, başlangıç ve bitiş tarihi, hizmet süresi ve eklenecek fiili hizmet süresi girilir | Başarılı: EK-02 (redirect) · Hata: EK-03 |
| EK-04 | Hizmet bilgisi güncelleme | `hizmetBilgileriGuncelle.jsp` | `/HizmetBilgileriGuncelleSayfaAction.do` | `/HizmetBilgileriGuncelleAction.do` (Güncelle); Listeye Dön | HizmetBilgileriForm (validate=true) | Seçilen hizmet döneminin alanları düzenlenir | Başarılı: EK-02 (redirect) · Hata: EK-04 |

**Ekranı olmayan işlem action'ları**

| Action | Sınıf | Tetiklendiği ekran | Ne yapar | Sonuç |
| --- | --- | --- | --- | --- |
| `/HizmetBilgileriSilAction.do` | HizmetBilgileriSilAction | EK-02 | Seçilen hizmet kaydını siler | EK-02'ye redirect |
| `/EmeklilikHesaplaAction.do` | EmeklilikHesaplaAction | EK-02 | Emeklilik hesaplamasını yapar (asıl iş kuralları burada başlar) | EK-02'ye forward (redirect değil; sonuç request'te taşınıyor olabilir) |

**Koddan çıkan notlar ve açık sorular**

- Tüm form'lar `scope=request` olduğu hâlde EK-01 bilgileri ve hizmet listesi sayfalar arasında korunuyor; büyük ihtimalle session'da tutuluyor. Liste ve Hesapla action'larında doğrulanacak.
- "Emeklilik Hesapla" butonu yalnızca listede en az bir hizmet varken görünüyor.
- Ekranda satır seçim kutusu görünmüyor. Güncellenecek veya silinecek satır nasıl seçiliyor (satıra tıklama, JavaScript)?
- "Maluliyet Derecesi" alanı, "Maluliyet Var mı?" Hayır iken pasif; bu bir JavaScript kuralı, yeni sistemde korunmalı.
- EK-02'den EK-01'e dönüş butonu yok. Emeklilik bilgilerini değiştirmek için baştan mı başlanıyor?
- `<data-sources/>` boş; veri tabanı bağlantısı Liberty `server.xml` üzerinden (JNDI) geliyor olmalı.
- Doğrulama `validation.xml` ile yapılıyor ve ilk hatada duruyor (`stopOnFirstError=true`). Hata mesajları `tr.gov.sgk.emektarWeb.genel.resource.Mesajlar` dosyasında, ekranda `messages.jsp` ile gösteriliyor.
- Tiles için ikinci bir dosya yükleniyor: `istatistikAylikBaglama/tiles-defs.xml`. Bu modül bu uygulamaya mı ait, yoksa başka bir projeden kalma mı?

Her ekranın ekran görüntüsünü bu tablonun altına ekle; yeni arayüz tasarlanırken referans olur.

# 4. Kullanıcı akışı

Kullanıcı önce kişisel emeklilik bilgilerini, sonra hizmet dönemlerini girer ve tek tuşla hesaplama yapar; akış ekran görüntülerinden çıkarılmıştır.

**Ana akış**

1. Kullanıcı `/hakKazanma.do` ile açılış ekranına (EK-01) gelir. Giriş veya oturum açma ekranı görünmüyor [doğrulanacak].
2. EK-01'de emeklilik türü, cinsiyet, doğum tarihi ve varsa özel durumları girip TAMAM'a basar; `EmeklilikBilgileriAction` formu doğrular.
3. EK-02 açılır: girilen bilgiler salt okunur özetlenir, hizmet listesi boştur.
4. "Hizmet Ekle" ile EK-03'e geçer, bir hizmet dönemi girip Kaydet'e basar ve listeye döner. Birden çok dönem eklenebilir.
5. Gerekirse "Hizmet Güncelle" (EK-04) veya "Hizmet Sil" ile listeyi düzeltir.
6. "Emeklilik Hesapla"ya basar; sonuç EK-02'nin altında üç blokta gösterilir: Sigortalılık Bilgileriniz, Emeklilik Şartlarınız, Hak Kazanma Sonuç. En altta emeklilik tarihi kırmızı ile yazılır.

**Alternatif ve hata akışları**

- EK-01'de doğrulama hatası: EK-01 yeniden gösterilir, mesaj `messages.jsp` alanında çıkar [hangi mesajlar?].
- EK-03 / EK-04'te doğrulama hatası: aynı ekran yeniden gösterilir [hangi mesajlar?].
- EK-03 / EK-04'te "Listeye Dön": kayıt yapılmadan EK-02'ye dönülür.
- Kişi şartları zaten sağlıyorsa: [sonuç ekranı ne gösteriyor?]
- Veri tabanı veya sistem hatası: [genel hata sayfası var mı? `global-exceptions` boş]

# 5. Girdiler ve çıktılar

Girdiler EK-01 ve EK-03/04 ekranlarından, çıktılar hesaplamadan sonra EK-02'nin altında gösterilen bloklardan alınmıştır. Zorunluluk bilgisi `validation.xml` ile doğrulanacaktır.

**Girdiler**

| Alan | Ekran | Tür / değerler | Zorunlu | Not |
| --- | --- | --- | --- | --- |
| Emeklilik türü | EK-01 | Seçenek: Tam / Kısmi | [ ] | Varsayılan Tam |
| Cinsiyet | EK-01 | Seçenek: Erkek / Kadın | [ ] | Varsayılan Erkek |
| Doğum tarihi | EK-01 | Metin, gg/aa/yyyy | [ ] |  |
| Maluliyet var mı? | EK-01 | Liste: Hayır / [Evet] | [ ] |  |
| Maluliyet derecesi (%) | EK-01 | Liste [seçenekler?] | [ ] | "Maluliyet var mı?" Hayır iken pasif; özette 0 görünür |
| Erken yaşlanma var mı? | EK-01 | Liste: Hayır / [Evet] | [ ] |  |
| Bakıma muhtaç malül çocuğunuz var mı? | EK-01 | Onay kutusu | Hayır |  |
| Sadece yurt dışı borçlanması ile emekli olmak istiyorum | EK-01 | Onay kutusu | Hayır | 3201 sayılı Kanuna göre yurt dışı borçlanması |
| Hizmet türü | EK-03 / EK-04 | Liste: Bağkur, [diğer seçenekler?] | [ ] | Sözleşmeli aylıklarda yurt dışı süreleri Bağkur olarak girilmeli (ekrandaki uyarı) |
| Başlangıç tarihi | EK-03 / EK-04 | Metin, gg/aa/yyyy | [ ] |  |
| Bitiş tarihi | EK-03 / EK-04 | Metin, gg/aa/yyyy | [ ] |  |
| Hizmet süresi | EK-03 / EK-04 | Gün | [ ] | Örneklerde 360 günlük yıl ile çıkıyor (01/01/2000–31/12/2019 = 7200 gün). Otomatik mi hesaplanıyor, elle mi giriliyor? |
| Hizmete eklenecek fiili hizmet süresi | EK-03 / EK-04 | Gün | Hayır | Boş bırakılabiliyor |

**Çıktılar**

Örnek: 01/01/1970 doğumlu erkek, tam emeklilik, 01/01/2000–31/12/2019 Bağkur hizmeti, 12/10/2026'da hesaplandı.

| Blok | Çıktı | Format | Örnek değer |
| --- | --- | --- | --- |
| Sigortalılık Bilgileriniz | Yaşınız | yıl ay gün | 56 yıl 9 ay 11 gün |
| Sigortalılık Bilgileriniz | Sigortalılık süreniz | yıl ay gün | 26 yıl 9 ay 12 gün |
| Sigortalılık Bilgileriniz | Hizmet süreniz | gün | 7200 gün |
| Emeklilik Şartlarınız | Yaş | yıl | 60 yıl |
| Emeklilik Şartlarınız | Sigortalılık süresi | yıl | 25 yıl |
| Emeklilik Şartlarınız | Hizmet süresi | gün | 9000 gün |
| Hak Kazanma Sonuç | Kalan yaşınız | yıl ay gün | 3 yıl 2 ay 19 gün |
| Hak Kazanma Sonuç | Kalan sigortalılık süreniz | yıl ay gün | 0 yıl 0 ay 0 gün |
| Hak Kazanma Sonuç | Kalan hizmet süreniz | gün | 1800 gün |
| Hak Kazanma Sonuç | Emeklilik tarihi | "gg/aa/yyyy tarihinde emekli olabilirsiniz." | 12/10/2031 |

Sonucun altında iki sabit metin var: genel sağlık sigortası dahil prim borcu olmaması gerektiği uyarısı ve sonuçların bilgilendirme amaçlı olduğu, resmi işlemlerde kullanılamayacağı notu.

**Örnekten çıkan gözlemler (İş Kuralları dokümanında doğrulanacak)**

- Yaş ve sigortalılık süresi hesaplama gününe (sistem tarihi) göre bulunuyor. Aynı girdi farklı günde farklı sonuç verir; test senaryolarında tarih sabitlenmelidir.
- Emeklilik tarihi, bugünden itibaren kalan 1800 günün 360 günlük yıl ile eklenmesine (5 yıl) karşılık geliyor; hizmetin bugünden sonra kesintisiz devam ettiği varsayılıyor olabilir.
- Yaş 11 gün, sigortalılık 12 gün çıkıyor; sigortalılık hesabı başlangıç gününü de sayıyor olabilir.

# 6. Teknik ortam

Uygulama Java 8 üzerinde, gömülü Open Liberty içinde çalışan bir Struts 1 web uygulamasıdır; veri erişimi doğrudan JDBC ile yapılır.

| Bileşen | Mevcut | Sürüm | Nereden bakılır |
| --- | --- | --- | --- |
| Dil | Java | 8 [tam sürüm] | `pom.xml` / `build.gradle` / derleme betiği |
| Web çatısı | Struts 1 | [1.x] | Bağımlılık listesi, `WEB-INF/lib` |
| Arayüz | JSP + [JSTL / Struts taglib] | [ ] | `WEB-INF` altındaki JSP'ler |
| Veri erişimi | JDBC | [sürücü sürümü] | DAO sınıfları |
| Sunucu | Open Liberty (gömülü) | [ ] | `server.xml`, başlatma sınıfı |
| Veri tabanı | [Oracle / PostgreSQL / ...] | [ ] | `server.xml` datasource tanımı |
| Derleme | [Maven / Gradle / Ant] | [ ] | Proje kök dizini |

- **Uygulama nasıl başlatılıyor?** [main sınıfı, komut, betik]
- **Ortamlar:** [geliştirme / test / canlı, hangi adreslerde?]
- **Yapılandırma dosyaları:** [`server.xml`, `.properties` dosyaları, ortam değişkenleri]
- **Loglama:** [log4j / java.util.logging, log dosyalarının yeri]

# 7. Dış bağımlılıklar

Uygulamanın kendi veri tabanı dışında konuştuğu her sistem bir satırdır. Kodda `URL`, `HttpURLConnection`, `Socket`, `InitialContext` (JNDI), `ldap`, `File` ve dış JAR'ları arayarak bulunur.

| Sistem | Ne için kullanılıyor? | Bağlantı şekli | Kodda yeri |
| --- | --- | --- | --- |
| [Ör. kimlik doğrulama / LDAP] | [ ] | [ ] | [ ] |
| [Ör. SGK veya kurum servisi] | [ ] | [SOAP / REST / DB link] | [ ] |
| [Ör. raporlama / PDF] | [ ] | [JasperReports, iText ...] | [ ] |

Bağımlılık yoksa "Dış bağımlılık yoktur" yaz; bu da önemli bir bilgidir.

# 8. Açık sorular ve sonraki adımlar

Analiz sırasında cevaplanamayan her soru buraya yazılır; kime sorulacağı not edilir.

**Açık sorular**

- [ ] Hesaplama kuralları en son hangi mevzuat değişikliğine göre güncellendi?
- [ ] Sonuçların doğruluğunu teyit edebilecek bir iş birimi veya kişi var mı?
- [ ] Kullanılmayan (ölü) ekran veya Action var mı?

**Sonraki adımlar**

- [ ] Uygulamayı baştan sona kullanıp ekran görüntülerini ekle
- [ ] Ekran listesini `hakKazanma-struts-config.xml` ile karşılaştırıp tamamla
- [ ] Doküman 3: Teknik Analiz ve Kod Haritası'na başla
