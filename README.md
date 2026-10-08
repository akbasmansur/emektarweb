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

# 3. Ekran Listesi

| Ekran ID | Ekran adı | URL (.do) | JSP | Ne yapılır | Sonraki ekran |
|---|---|---|---|---|---|
| EK-01 | Kişi Bilgi Girişi | `/giris.do` | `giris.jsp` | Doğum tarihi, cinsiyet, ilk sigorta tarihi, prim günü ve gerekli diğer bilgiler girilir. | EK-02 |
| EK-02 | Hesaplama Sonucu | `/hesapla.do` | `sonuc.jsp` | Emeklilik tarihi ve varsa eksik emeklilik şartları gösterilir. | EK-01 |
| EK-03 | — | — | — | — | — |