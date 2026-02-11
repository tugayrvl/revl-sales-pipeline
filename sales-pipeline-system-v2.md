# B2B Satış Pipeline Sistemi — Tam Sistem Tasarımı v2

---

## 1. SİSTEM GENEL BAKIŞ

Bu sistem, kişi sayım cihazı satışı için tüm satış sürecini yöneten bir **Chrome Extension + Web App** çözümüdür.

**Temel bileşenler:**
- Chrome Extension (sahada ve LinkedIn'de hızlı veri girişi)
- Web App (dashboard, planlama, takip, raporlama)
- Cold Mail API (otomatik mail gönderimi)
- Veri birleştirme motoru (Excel import + fuzzy matching)

### UX Tasarım Prensipleri

1. **Maksimum veri, minimum scroll:** Kullanıcı görsel odaklı çalışır. Bir ekranda görebileceği maks veriyi görmeli. Kartlar kompakt, tablolar yoğun, istatistikler tek satırda.
2. **Tek bakışta hakimiyet:** Dashboard ve haftalık ekran açıldığında durum anında anlaşılmalı. Renk kodları, badge'ler ve sayısal özetler ile.
3. **Yerinde düzenleme:** Ayrı sayfaya gitmeye gerek yok. Firma kartı genişler, kontakt bilgisi tabloda düzenlenir, not yerinde eklenir.
4. **Akıllı sıralama:** Firmalar otomatik önceliklendirilir — telefonu olan en üstte, kontaktı olmayan en altta.
5. **Hızlı ve akıcı:** Minimum tıklama ile iş yapılabilmeli. Tek tıkla LinkedIn'e git, tek tıkla ara, tek tıkla not ekle.
6. **Veri girişi haftalık akıştan ayrı:** Firma ekleme, Excel yükleme, fuzzy match onaylama gibi işlemler ayrı "Veri Yönetimi" bölümünde. Haftalık ekran sadece o haftanın işine odaklanır.
7. **Excel önizleme:** Yüklenen Excel dosyaları tablo olarak önizlenir, içinden ne çıktığı anlaşılır, onaylandıktan sonra sisteme aktarılır.

### Senkronizasyon Davranışı

- Extension ve App aynı veritabanını (Supabase) kullanır — değişiklikler otomatik senkronize olur
- Her düzenleme sonrası küçük bir bildirim gösterilir: "Senkronize edildi ✓"
- Otomatik sync çalışmasa bile manuel sync butonu her zaman erişilebilir
- Sync durumu header'da küçük bir ikon ile gösterilir (yeşil = bağlı, sarı = bekliyor, kırmızı = bağlantı yok)

---

## 2. VERİ KAYNAKLARI VE İÇE AKTARMA

### 2.1 AVM Analiz Exceli (Sahadan)
- AVM'ye gidildiğinde oluşturulan Excel
- İçerik sütunları:
  - **Sıra No** — Ziyaret sırası
  - **Firma adı**
  - **Cihaz durumu** — Boş = cihaz yok, marka adı = cihaz var
  - **Fotoğraf dosya ismi** — Sahada çekilen fotoğrafın referansı
  - **Diğer olası sütunlar** — Esnek yapı, ek sütunlar olabilir
- **Cihaz yok → Direkt hedef müşteri**
- **Cihaz var → Rakip firma notu + gelecek hedef**
- Her AVM ziyareti ayrı bir kayıt olarak saklanır (tarih, AVM adı, kaç firma görüldü)

### 2.2 Lusha Excel Dosyaları (Kontakt Verisi)
- 25'li Excel dosyaları halinde indirilir
- **En güvenilir veri kaynağı** — çakışmalarda Lusha verisi önceliklidir
- İçerik: İsim, ünvan, firma, email, telefon, LinkedIn URL
- Toplu yükleme: Birden fazla Excel seçilip tek seferde yüklenebilir
- **Çatı firmalar da Lusha'dan gelir** — normal firmalar gibi kontakt satırı olarak. Alt firma ile bağlantısı sistem içinde kurulur
- **Mevcut kontaktlar tekrar gelebilir** — aynı kişi yeni bilgiyle (ör: eskiden email yoktu, şimdi var). Sistem duplicate oluşturmaz, mevcut kontağa eksik bilgiyi ekler
- Tekrarlanan satır eklenmez, sadece yeni bilgi güncellenir

### 2.3 Manuel Kontakt Girişi
- Lusha dışı kaynaklardan bulunan kontakt bilgileri
- Extension veya App üzerinden kolayca eklenebilir (Excel yüklemeye gerek yok)
- Tek bir alan bile eklenebilir (ör: sadece telefon numarası bulundu)

### 2.4 Veri Birleştirme Motoru

#### Fuzzy Matching (Firma İsmi Eşleştirme)
- AVM Exceli'ndeki isimler ile Lusha/mevcut verideki isimler tam eşleşmeyebilir
- Sistem benzer isimleri tespit eder
  - Örnek: "Superstep" ↔ "SUPERSTEP Mağazacılık A.Ş." ↔ "Super Step"

**Türkçe Karakter Desteği (Kritik):**
- Türkçe büyük/küçük harf dönüşümü doğru yapılmalı
- `yargici` → `YARGICI` değil, `YARGİCİ` olmalı
- `İ` ↔ `i` ve `I` ↔ `ı` dönüşümleri doğru uygulanmalı
- Karşılaştırma sırasında Türkçe locale kullanılmalı (`tr-TR`)
- Tüm karşılaştırmalar case-insensitive + Türkçe-aware yapılmalı

**Etkileşimli Eşleştirme Akışı:**
- Benzer firma taraması **her zaman** manuel tetiklenebilir (sadece import sırasında değil)
- Sistem eşleşme adaylarını tek tek sunar
- Her aday için iki buton:
  - **AYNI** → Lusha'daki isim doğru kabul edilir, diğer isim buna eşlenir, veriler birleştirilir
  - **FARKLI** → Bu eşleşme reddedilir, ekrandan kaldırılır, sıradaki adaya geçilir
- Reddedilen eşleşmeler tekrar sorulmaz (blacklist)
- Tüm adaylar bitene kadar devam eder

#### Veri Öncelik Sırası
1. Lusha verisi (en doğru)
2. Manuel giriş
3. AVM Excel verisi

#### Duplicate Kontrolü
- Aynı kişi farklı Excel'lerde gelirse tekrar eklenmez
- Aynı firma farklı isimlerle geldiyse fuzzy match ile yakalanır

#### Sürekli Genişleme
- Her yeni Excel yüklenmesinde mevcut data genişler
- Yeni firmalar otomatik eklenir
- Mevcut firmalara yeni kontaktlar eklenir
- Yeni bilgi varsa güncellenir (ör: eksik telefon artık var)

---

## 3. VERİ MODELİ

### 3.1 Firma (Company)

| Alan | Açıklama |
|------|----------|
| Firma Adı | Ana firma adı |
| Çatı Firma | Varsa bağlantı (ör: Eren Perakende → Superstep) |
| Cihaz Durumu | Yok / Var |
| Mevcut Cihaz Markası | Cihaz varsa hangi rakip (ör: vcount, td next) |
| Mevcut Cihaz Durumu | Yeni mi, eski mi, modeli ne |
| AVM Lokasyonu | Hangi AVM'de/AVM'lerde görüldü |
| Şube Sayısı | Toplam şube adedi |
| Pipeline Aşaması | Mevcut aşama |
| Hedef Hafta | Hangi haftaya atandı |
| LinkedIn Hesabı | Firmanın LinkedIn sayfası (tıklanabilir) |
| Website | Firmanın web sitesi (tıklanabilir) |
| Notlar | Genel notlar |
| Teklif Fiyatı | Ne fiyat verildi |
| Kaç Cihaz İstendi | Talep edilen cihaz sayısı |
| Kontrat Bitiş Tarihi | Mevcut rakip kontratı ne zaman bitiyor |

### 3.2 Çatı Firma Yapısı
- Çatı firmalar da Lusha Excel'inden gelir (ayrı bir kaynak değil, normal kontakt satırı olarak)
- Sistem içinde çatı firma ↔ alt firma bağlantısı manuel kurulur
- Bir çatı firma birden fazla alt firmaya sahip olabilir
- Örnek: Eren Perakende → Superstep, Eren Giyim, vs.
- Alt firma hedef ise ve kontaktı yoksa → çatı firmadaki kontaktlar otomatik listelenir alt firmanın kartında
- Çatı firma kartında: sahip olduğu tüm alt firmalar, kaç tanesi hedef, kaçının kontaktı var
- **Bağlantı veritabanında tutulur** — hem Extension hem App aynı DB'yi (Supabase) kullanır

### 3.3 Kontakt (Contact)

| Alan | Açıklama |
|------|----------|
| İsim Soyisim | |
| Ünvan | Karar verici pozisyonu |
| Firma | Bağlı olduğu firma |
| Telefon | Varsa numara, yoksa boş |
| Work Email | Varsa adres, yoksa boş |
| Kişisel Email | Varsa |
| LinkedIn Profili | Tıklanabilir link |
| Kaynak | Lusha / Manuel / Diğer |
| Durum | Aktif / Yanlış numara / Geçersiz mail / vs. |
| Oluşturulma Tarihi | Ne zaman eklendi |

**Karakter Profili (Arama sonrası doldurulan):**

| Alan | Açıklama |
|------|----------|
| İletişim Tarzı | Resmi / Samimi / Kısa-net / Çok konuşkan |
| Karar Verme Yetkisi | Tek karar verici / Üstüne danışacak / Komite / Belirsiz |
| Karar Verme Hızı | Hızlı / Yavaş-düşünür / Erteleyici |
| Argüman Tercihi | Maliyet / Teknoloji / Referans-kanıt / ROI / Marka bilinirliği |
| Rakip Görüşü | Memnun / Şikayetçi / Nötr / Bilgisi yok |
| Fiyat Hassasiyeti | Çok hassas / Makul / Değer odaklı |
| Aciliyet Durumu | Acil / Planlı bütçe / Acele yok / Bilgi topluyor |
| Kişisel Gözlemler | Serbest not (hobiler, tercihler, dikkat çeken şeyler) |

### 3.4 Kontakt Önceliklendirme
- Her firma içinde kontaktlar sıralanabilir (1., 2., 3. öncelik)
- Anlam: "Önce bunu ara, açmazsa ikincisini, o da olmazsa üçüncüsünü"
- Sürükle-bırak veya numara ile sıralama
- Arama günü ekranında bu sıra görünür

### 3.5 Kontakt Karakter Profili

Her aramadan sonra sistem otomatik olarak karakter profili sorularını açar. Kullanıcı not girer veya hızlı seçim yapar. Amaç: bir sonraki aramada bu kişiyle nasıl konuşulacağını bilmek.

**Profil Soruları (Kurumsal B2B Satış Dinamiklerine Göre):**

| # | Soru | Seçenekler / Not Alanı |
|---|------|------------------------|
| 1 | İletişim tarzı nasıldı? | Resmi / Samimi / Kısa-net konuşuyor / Çok konuşkan |
| 2 | Karar verme yetkisi var mı? | Tek karar verici / Üstüne danışacak / Komite kararı / Belirsiz |
| 3 | Karar verme hızı nasıl? | Hızlı karar alır / Yavaş, düşünür / Erteleyici |
| 4 | Ne tür argümanlara açık? | Maliyet odaklı / Teknoloji meraklısı / Referans/kanıt istiyor / ROI odaklı / Marka bilinirliği önemsiyor |
| 5 | Rakip hakkında ne düşünüyor? | Memnun / Şikayetçi / Nötr / Bilgisi yok |
| 6 | Fiyat hassasiyeti? | Çok hassas / Makul / Fiyat umurunda değil, değer önemli |
| 7 | Aciliyeti var mı? | Acil ihtiyaç / Planlı bütçe dönemi / Acele yok / Sadece bilgi topluyor |
| 8 | Kişisel gözlemler | Serbest not — "futbol sever", "sabah aramaları tercih ediyor", "asistanı üzerinden iletiyor" vs. |

**Akış:**
1. Arama yapılır, sonuç girilir
2. Sistem otomatik olarak karakter profili formunu açar
3. Daha önce girilmiş bilgiler pre-filled olarak görünür
4. Kullanıcı yeni bilgileri ekler veya mevcut bilgileri günceller
5. Tüm alanlar opsiyonel — bilmiyorsan boş bırak
6. Sonraki aramada bu profil kontakt kartında görünür
7. "Kişisel gözlemler" alanı en değerli — satışta bağ kurma için

### 3.6 Kontakt Düzenleme Kuralları
- Yeni kontakt ekleme (extension veya app üzerinden, Excel olmadan)
- Mevcut kontakt bilgisi güncelleme (yeni telefon bulundu vs.)
- Yanlış bilgi silme (ör: aradım numara yanlış çıktı → sil)
- Yeni bilgi ekleme (ör: email yoktu, buldum → ekle)
- Tüm düzenlemeler hem Extension hem App üzerinden yapılabilmeli

---

## 4. PİPELINE AŞAMALARI

### 4.1 Cihaz YOK — Ana Satış Pipeline'ı

```
YENİ HEDEF
  │ Firma keşfedildi, haftaya atandı
  ▼
KONTAKT ARANIYOR
  │ LinkedIn/Lusha'dan karar verici aranıyor
  ▼
KONTAKT HAZIR
  │ Telefon numarası mevcut, arama bekliyor
  ▼
ARAMA YAPILDI
  ├──→ TOPLANTI ALINDI (tarih belirlendi)
  │       ▼
  │     DEMO YAPILDI
  │       ▼
  │     DEMO SONRASI SÜREÇ (bkz. 4.3)
  │
  ├──→ TANIŞILDI, TOPLANTI YOK
  │       ▼
  │     DÜZENLI TAKİP (periyodik arama)
  │
  └──→ ULAŞILAMADI
          ▼
        TEKRAR ARAMA (sonraki haftaya planla)
```

### 4.2 Cihaz VAR — Alternatif Kanallar

```
CİHAZI VAR (Rakip firma kullanıyor)
  │
  ├──→ Email VARSA → COLD MAIL DİZİSİ
  │     Salı: 1. mail → Cuma: 2. mail → Sonraki Salı: tekrar
  │     Günde max 12 kişi, mailler arası 20-40 dk pause
  │     API ile otomatik gönderim
  │
  ├──→ Email YOK, LinkedIn VARSA → LINKEDIN OUTREACH
  │     Bağlantı isteği gönder (notlu veya notsuz)
  │       ▼
  │     Kabul bekleme (periyodik kontrol)
  │       ▼
  │     Kabul edildi → Mesaj at → Telefon numarası iste
  │
  └──→ KONTRAT TAKİBİ
        Kontrat bitiş tarihi öğren
        Bitiş tarihine 1 ay kala → Aktif takip başlat
```

### 4.3 Demo Sonrası Süreç (Closing Pipeline)

```
DEMO YAPILDI
  ▼
TEKLİF İLETİLDİ (fiyat, cihaz sayısı, şube bilgisi)
  ▼
┌─────────────────────────────────────┐
│  PARALEL SÜREÇLER                   │
│                                     │
│  SÖZLEŞME         JİRA KAYDI        │
│  (veri akışına     (paralel          │
│   kadar            başlayabilir)     │
│   imzalanmalı)                       │
└─────────────────────────────────────┘
  ▼
KARGO / LOJİSTİK
  ▼
CİHAZ KURULUMU
  ▼
VERİ AKIŞI BAŞLADI MI? (sözleşme bu noktada imzalı olmalı)
  ▼
TEKNİK SORUN VAR MI?
  ▼
TAMAMLANDI ✓
```

Her aşamada durum takibi:
- Sözleşme: Gönderildi / İnceleniyor / İmzalandı
- Jira: Kayıt açıldı / Devam ediyor / Tamamlandı
- Kargo: Hazırlanıyor / Kargolandı / Teslim edildi
- Kurulum: Planlandı / Kuruldu
- Veri akışı: Başlamadı / Test / Aktif
- Teknik sorun: Yok / Var (açıklama notu)

---

## 5. HAFTALIK ÇALIŞMA DÖNGÜSÜ

### 5.1 Arama Günü (Haftada 1 Gün)

**Arama Günü Ekranı şunları gösterir:**
1. Bu haftanın hedef firmaları (telefonu olanlar — aranacak liste)
2. Düzenli takip firmaları (önceki haftalardan biriken)
3. Tekrar aranacaklar (geçen hafta ulaşılamayan)

**Her aramadan sonra:**
- Not ekleme alanı (ne konuşuldu)
- Sonuç seçimi:
  - ✅ Toplantı alındı → tarih gir
  - 🔄 Tanışıldı, toplantı yok → düzenli takibe al
  - ❌ Ulaşılamadı → tekrar dene tarihi seç
  - ⚠️ Yanlış numara → kontaktı güncelle/sil
  - 🚫 İlgilenmiyorlar → arşivle

**Gün sonu:**
- Kaç kişi arandı (otomatik sayım)
- To-do listesi güncellendi
- Önümüzdeki hafta planı görünür

### 5.2 Hafta Boyunca (Her Gün)

**Otomatik:**
- Cold mail dizisi gönderimi (API ile)
  - Salı: İlk mail
  - Cuma: İkinci mail (3 gün sonra)
  - Sonraki Salı: Döngü tekrar
  - Günde max 12 kişi
  - Mailler arası 20-40 dk pause

**Manuel:**
- LinkedIn bağlantı istekleri kontrol
- Kabul edilenlere mesaj at
- Yeni firma/kontakt keşifleri ekle
- Demo süreçlerini takip et

### 5.3 Hafta Sonu / Planlama

- Yeni AVM Excel'i yükle (varsa)
- Yeni Lusha Excel'leri yükle
- Fuzzy match onaylarını yap
- Gelecek hafta hedeflerini belirle
- Pipeline genel durum kontrolü
- Rakip analizi güncelle

---

## 6. COLD MAİL SİSTEMİ

### 6.1 Konfigürasyon (Extension/App Üzerinden)

| Ayar | Açıklama |
|------|----------|
| Mail şablonu | Başlık + metin + ek (birden fazla şablon olabilir) |
| Hedef ünvan | Hangi ünvandaki kişilere gidecek (ör: sadece IT Müdürü) |
| Jenerik mi ünvana özel mi | Aynı mail herkese mi, ünvana göre farklı mı |
| Gönderim takvimi | Salı + Cuma (varsayılan) |
| Günlük limit | 12 kişi/gün |
| Mailler arası bekleme | 20-40 dk rastgele pause |
| Ekler | PDF, dosya vs. |

### 6.2 Gönderim Mantığı
- Firma cihazı VAR + kontaktın emaili VAR + telefonu YOK → Cold mail havuzuna gir
- API ile otomatik gönderim
- Spam koruması: günlük limit + bekleme süresi + farklı saatler

### 6.3 Takip ve Metrikler
- Hangi şablonu / başlığı kullandım
- Hangi ünvana hangi maili attım
- Kaç mail gönderildi (toplam, bu hafta)
- Hangilerinden dönüş aldım
- Hangi şablon daha fazla dönüş alıyor (dönüşüm oranı)
- Hangi ünvan daha fazla dönüş veriyor
- Hangi başlık daha iyi performans gösteriyor

---

## 7. LINKEDIN OUTREACH SİSTEMİ

### 7.1 Bağlantı İsteği Gönderimi
- Kontaktın sadece LinkedIn'i varsa (email/telefon yok)
- İki senaryo:
  - **Notlu gönderim:** Bağlantı isteğiyle birlikte kısa mesaj
  - **Notsuz gönderim:** Sadece bağlantı isteği
- Her kontakt için hangi senaryonun kullanıldığı kaydedilir
- Not kullanıldıysa not içeriği de kaydedilir

### 7.2 Takip Süreci
- Bağlantı isteği gönderildi → tarih kaydı
- Periyodik kontrol: Kabul etti mi?
- Kabul edildi → Mesaj gönder (telefon numarası iste)
- Kabul edilmedi → Bekleme süresi sonra tekrar dene veya geç

### 7.3 Metrikler
- Notlu gönderime kaç kabul geldi
- Notsuz gönderime kaç kabul geldi
- Hangi not metni daha iyi kabul oranı veriyor
- Kabul → mesaj → telefon alma dönüşüm oranı

---

## 8. RAKİP ANALİZİ

### 8.1 Rakip Firma Kartı

Her rakip firma için:

| Alan | Açıklama |
|------|----------|
| Firma Adı | Rakip firmanın adı |
| Veri Kaynağı | CCTV'den mi kendi donanımından mı |
| Donanım | Kendi üretimi mi, 3. parti mi |
| Cihaz Modelleri | Bilinen modeller ve durumları (yeni/eski) |
| Fiyatlandırma | Bilinen fiyat bilgileri |
| Aylık Fee | Alıyor mu, ne kadar |
| Güçlü Yönler | Nerelerde iyi |
| Zayıf Yönler | Nerelerde zayıf |
| Bizimle Kıyaslama | Bize göre avantaj/dezavantajları |
| Bilgi Kaynağı | Sahadan mı, müşteriden mi, web'den mi |
| Notlar | Genel notlar, duyumlar |

### 8.2 Rakip Veri Toplama Kaynakları
- **Sahadan (AVM):** Cihazı gördüğünde marka ve model notu
- **Müşteriden:** Görüşmelerde öğrenilen bilgiler (fiyat, kontrat süresi vs.)
- **Web araştırması:** Rakip firmanın sitesi, haberler, vs.

### 8.3 Firma-Rakip İlişkisi
- Bir firmanın kartında mevcut rakip firma bilgisi görünür
- Rakip cihazı yeni mi eski mi
- Rakip cihazın modeli ne
- Kontrat bitiş tarihi (öğrenildiyse)

### 8.4 Rakip Karşılaştırma Tablosu
- Tüm rakipler yan yana
- Fiyat, donanım, veri kaynağı, fee, güçlü/zayıf yönler
- Bu tablo satış görüşmelerinde referans olarak kullanılır

---

## 9. OTOMATİK TO-DO SİSTEMİ

Sistem aşağıdaki durumlarda otomatik to-do üretir:

| # | Tetikleyici | Otomatik To-Do |
|---|-------------|----------------|
| 1 | Yeni firma eklendi, cihaz yok | "Kontakt bul: [Firma]" |
| 2 | Alt firmanın kontaktı yok, çatı firmada var | "Çatı firma kontaktlarını incele: [Çatı] → [Alt Firma]" |
| 3 | Kontakt telefonu bulundu | "[Firma] — Arama yap (Hafta X)" |
| 4 | Arama yapıldı, ulaşılamadı | "Tekrar ara: [Firma] (Hafta X+1)" |
| 5 | Tanışıldı, toplantı yok | "Takip araması: [Firma] (Hafta X+2)" |
| 6 | Toplantı alındı | "Toplantı: [Firma] — [Tarih]" |
| 7 | Demo yapıldı | "Teklif hazırla: [Firma]" |
| 8 | Teklif iletildi | "Teklif takibi: [Firma] (3 gün sonra)" |
| 9 | Sözleşme gönderildi | "Sözleşme takibi: [Firma]" |
| 10 | Jira kaydı açılacak | "Jira kaydı aç: [Firma]" |
| 11 | Cihaz kargolanacak | "Kargo takibi: [Firma]" |
| 12 | Kurulum planlandı | "Kurulum: [Firma] — [Tarih]" |
| 13 | Veri akışı kontrol | "Veri akışı kontrol: [Firma]" |
| 14 | Cold mail dizisi başlatıldı | "Mail dönüş takibi: [Firma]" |
| 15 | LinkedIn isteği gönderildi | "LinkedIn kontrol: [Kişi] kabul etti mi?" |
| 16 | LinkedIn kabul edildi | "LinkedIn mesaj at: [Kişi]" |
| 17 | Kontrat bitiş tarihi girildi | "Takip başlat: [Firma] kontrat bitiyor ([Tarih] - 1 ay)" |
| 18 | Teknik sorun bildirildi | "Teknik sorun takibi: [Firma]" |
| 19 | Firma kontaktı eksik (hiç kontakt yok) | "Kontakt topla: [Firma]" |
| 20 | Düzenli takip zamanı geldi | "Düzenli takip araması: [Firma]" |

---

## 10. DASHBOARD VE GÖRSEL METRİKLER

### 10.1 Ana Dashboard — Makro Tablo (Tek Bakışta Her Şey)

#### Genel Durum Kartları (Üst Kısım)
- **Toplam firma sayısı** | Cihaz yok | Cihaz var
- **Toplam kontakt sayısı** | Telefonu var | Emaili var | Sadece LinkedIn
- **Çatı firma sayısı** | Alt firma sayısı
- **Bu hafta hedef** | Aranacak | Takip edilecek

#### Pipeline Dağılımı (Görsel Akış)
- Her aşamada kaç firma var (sayı + yüzde)
- Yeni Hedef → Kontakt Aranıyor → Kontakt Hazır → Arama Yapıldı → Toplantı → Demo → Closing
- Renk kodlu ilerleme

#### AVM Analiz Özeti
- Kaç AVM ziyareti yapıldı (toplam)
- Son ziyaret: hangi AVM, ne zaman
- Toplam kaç firma görüldü
- Kaçında cihaz var / kaçında yok
- AVM bazlı dağılım

#### Çatı Firma Haritası
- Hangi çatı firma kaç alt firmaya sahip
- Her çatı firmanın altındaki firmalar listesi
- Alt firmaların hedef durumu
- Kontakt durumu (çatı firmadan mı alt firmadan mı)

### 10.2 Haftalık Performans

#### Arama Metrikleri
- Bu hafta kaç kişi arandı
- Kaçına ulaşıldı
- Kaç toplantı alındı
- Kaç düzenli takibe eklendi
- Ulaşılamayan sayısı

#### Cold Mail Metrikleri
- Bu hafta kaç kişiye mail atıldı
- Hangi şablon/başlık kullanıldı
- Hangi ünvana hangi mail gitti
- Kaç dönüş alındı
- Şablon bazlı dönüşüm oranı
- Başlık bazlı dönüşüm oranı
- Ünvan bazlı dönüşüm oranı

#### LinkedIn Metrikleri
- Bu hafta kaç bağlantı isteği gönderildi
- Notlu mu notsuz mu
- Hangi not metni kullanıldı
- Kaç kabul geldi
- Senaryo bazlı kabul oranı (notlu vs notsuz)
- Kabul sonrası kaç mesaj atıldı
- Kaç telefon numarası elde edildi

### 10.3 Düzenli Takip Durumu
- Toplam kaç firma düzenli takipte
- Her birinin son aranma tarihi
- Ne zaman tekrar aranacak
- Son görüşme notu özeti
- Sıcaklık durumu (ilgili / ilgisiz / belirsiz)

### 10.4 Demo & Closing Durumu
- Kaç firma demo aşamasında
- Her birinin mevcut durumu:
  - Teklif iletildi mi? → Fiyat ne verildi?
  - Sözleşme durumu
  - Jira kaydı durumu
  - Kargo / kurulum durumu
  - Veri akışı durumu
  - Teknik sorun var mı?
- Kaç cihaz istendi (toplam talep)
- Şube bilgileri
- İçerideki rakip firma hangisi

### 10.5 Rakip Analizi Dashboard
- Rakip firma listesi
- Rakip bazlı karşılaştırma tablosu
- Kaç firmada hangi rakip var
- Hangi rakibin cihazları daha çok eski (fırsat)
- Rakip fiyat karşılaştırması
- CCTV vs kendi donanım dağılımı

### 10.6 Kontakt Eksiklik Raporu
- Hangi hedef firmanın hiç kontaktı yok
- Hangi firmanın kontaktı var ama telefonu yok (sadece email/LinkedIn)
- Çatı firmada kontakt var mı ama alt firmada yok mu
- Öncelik sırası: en acil kontakt bulunması gereken firmalar

---

## 11. CHROME EXTENSION ÖZELLİKLERİ

### 11.1 Firma Kartı Görüntüleme
- Extension açıldığında firma arama
- Firma kartındayken:
  - LinkedIn sayfasına tek tıkla git
  - Website'e tek tıkla git
  - Kontaktları gör
  - Not ekle
  - Pipeline aşamasını güncelle

### 11.2 Kontakt İşlemleri
- Yeni kontakt ekleme (elle, Excel olmadan)
- Mevcut kontakt düzenleme
- Yanlış bilgi silme
- LinkedIn profil sayfasındayken: "Bu kişiyi [Firma]'ya kontakt olarak ekle"

### 11.3 Excel Yükleme
- Lusha Excel (tek veya toplu)
- AVM Excel
- Fuzzy match onay ekranı

### 11.4 Cold Mail Konfigürasyonu
- Mail şablonu oluşturma/düzenleme (başlık + metin + ek)
- Ünvana özel mi jenerik mi seçimi
- Gönderim başlatma/durdurma
- Gönderim durumu görüntüleme

### 11.5 Hızlı İşlemler
- Arama notu ekleme
- To-do ekleme/tamamlama
- Firma durumu güncelleme
- Rakip bilgisi ekleme (sahada gördüğünde)

---

## 12. WEB APP EKRANLARI

### 12.1 Dashboard (Ana Sayfa)
- Bölüm 10'daki tüm metrikler
- Makro tablo görünümü — tek bakışta her şey

### 12.2 Haftalık Sayfa (Ana Çalışma Ekranı)

Bu ekran, haftalık çalışmanın merkezidir. Hafta seçildiğinde tek sayfada her şey görünür, minimum scroll ile.

**NOT:** Firma ekleme, Lusha Excel yükleme, AVM Excel yükleme ve Fuzzy Match onaylama gibi veri girişi işlemleri bu ekranda değildir. Bunlar header'daki global aksiyonlar veya ayrı "Veri Yönetimi" sayfası üzerinden yapılır (bkz. 12.11). Haftalık sayfa sadece o haftanın iş akışına odaklanır.

#### Üst Bölüm — Hafta Seçici ve Tarih Aralığı
- Hafta numarası + tarih aralığı görünür (ör: "Hafta 7 — 10 Şub – 16 Şub, 2026")
- ◀ ▶ butonları ile hafta değiştirme
- "Bu Hafta" butonu ile hızlı dönüş

#### Tıklanabilir İstatistik Kartları (Filtre Görevi Görür)
Sayfanın en üstünde, tek satırda özet kartlar. **Her karta tıklamak o filtreyi aktif eder:**

- **🔵 Hedef Firma** (X) — DEFAULT filtre. Bu haftaya atanmış tüm firmalar
- **🟢 Aramaya Hazır** (X) — Tıkla → sadece telefon numarası olan firmaları göster
- **🔴 Kontakt Eksik** (X) — Tıkla → sadece telefonu olmayan firmaları göster (kontakt bulmaya dalabilirsin)
- **🟡 Yapılan Arama** (X) — Bilgi kartı (filtre değil)
- **🟣 Toplantı Alınan** (X) — Bilgi kartı (filtre değil)
- **🟠 Düzenli Takip** (X) — Tıkla → sadece düzenli takip firmalarını göster

Aktif filtre kartının çerçevesi belirgin, diğerleri soluk. "✕ Filtreyi kaldır" ile default'a dönülür.

#### İki Ana Bölüm

**1. 🎯 Hedef Firmalar** — Bu haftaya atanmış firmalar (default görünüm)
- Akıllı sıralama: 🟢 aramaya hazır → 🟡 kontakt var telefon yok → 🔴 kontakt yok

**2. 🔄 Düzenli Takip** — Pipeline'da "düzenli takip" aşamasındaki tüm firmalar
- Sıralama: En uzun süredir aranmayan en üstte
- Default görünümde (hedef firma filtresi) her iki bölüm de görünür, üst üste
- Düzenli takip filtresine tıklandığında sadece takip firmaları görünür

#### Arama Ritmi Önerisi
Aramalara başlandığında önerilen sıra: 1 düzenli takip firması → 1 yeni hedef firma → 1 düzenli takip → 1 yeni hedef... Bu ritim interleaved call order olarak sistem tarafından desteklenir.

#### Firma Listesi — Akıllı Sıralama
Firmalar otomatik sıralanır:
1. **🟢 Aramaya Hazır** (telefon numarası var) — EN ÜSTTE
2. **🟡 Kontakt Var, Telefon Yok** (email veya LinkedIn var)
3. **🔴 Kontakt Yok** (henüz hiç kontakt bulunmadı)

Her firma satırında kompakt görünüm:
```
┌─────────────────────────────────────────────────────────────────┐
│ 🟢 SUPERSTEP (Eren Perakende)                    [Ara] [Detay] │
│    Kontakt: Ahmet Yılmaz — IT Müdürü — 📞 0532...              │
│    Son not: "İlgileniyor, hafta içi tekrar aranacak"            │
├─────────────────────────────────────────────────────────────────┤
│ 🟢 BOYNER                                        [Ara] [Detay] │
│    Kontakt: Mehmet K. — Operasyon Md. — 📞 0533...              │
│    Kontakt: Ayşe T. — IT Direktörü — 📞 0541...                │
│    2 aramaya hazır kontakt                                       │
├─────────────────────────────────────────────────────────────────┤
│ 🟡 KOTON                                              [Detay]  │
│    Kontakt: Ali V. — CTO — ✉️ ali@koton.com                    │
│    Telefon yok → Cold mail adayı                                │
├─────────────────────────────────────────────────────────────────┤
│ 🔴 IPEKYOL                                            [Detay]  │
│    Kontakt yok — Çatı firma: Ipekyol Grup                       │
│    → Çatı firmada 2 kontakt mevcut (tıkla gör)                 │
└─────────────────────────────────────────────────────────────────┘
```

- Firmalar kompakt, scrolla gerek kalmadan 10-15 firma görülebilmeli
- [Ara] butonu sadece telefonu olan kontaktlarda aktif
- [Detay] firma kartını açar (LinkedIn, website, tüm kontaktlar, notlar)
- Çatı firma kontaktları alt firmada yoksa otomatik gösterilir

#### Firma Tıklama → Genişleyen Kart (Detay)
Firmaya tıklandığında kart yerinde genişler, sayfa değişmez:
- Tüm kontaktlar listesi (bilgi durumu ikonlarıyla: 📞 ✉️ 🔗)
- Her kontaktın bilgilerini **yerinde düzenleme** (edit/sil/yeni bilgi ekle)
- Her kontaktın LinkedIn profiline **tek tıkla** erişim
- Firmanın LinkedIn sayfasına tek tıkla erişim
- Firmanın web sitesine tek tıkla erişim
- Firma notları ve geçmiş arama logları

#### Kontakt Önceliklendirme (Arama Sırası)
Bir firmada birden fazla kontakt varsa arama sırası belirlenebilir:
- Sürükle-bırak ile sıralama
- Mantık: "1. önce bunu ara → açmazsa 2. bunu → o da olmazsa 3. bunu"
- Sıralama kaydedilir, arama günü ekranında bu sıraya göre gösterilir
- Öncelik her zaman değiştirilebilir
- Arama sonucu "ulaşılamadı" ise sıradaki kontakt otomatik öne çıkar

#### Arama Sonrası Karakter Profili Soruları
Her aramadan sonra not girişinin altında **otomatik olarak** karakter profili soruları çıkar. Amaç: bir sonraki aramada bu kişiyle nasıl konuşman gerektiğini bilmen.

**Sorular (kurumsal B2B satış dinamiklerine göre):**

| # | Soru | Seçenekler / Not Alanı |
|---|------|------------------------|
| 1 | İletişim tarzı nasıldı? | Resmi / Samimi / Kısa-net konuşuyor / Çok konuşkan |
| 2 | Karar verme yetkisi var mı? | Tek karar verici / Üstüne danışacak / Komite kararı / Belirsiz |
| 3 | Karar verme hızı nasıl? | Hızlı karar alır / Yavaş, düşünür / Erteleyici |
| 4 | Ne tür argümanlara açık? | Maliyet odaklı / Teknoloji meraklısı / Referans/kanıt istiyor / ROI odaklı / Marka bilinirliği önemsiyor |
| 5 | Rakip hakkında ne düşünüyor? | Memnun / Şikayetçi / Nötr / Bilgisi yok |
| 6 | Fiyat hassasiyeti? | Çok hassas / Makul / Fiyat umurunda değil, değer önemli |
| 7 | Aciliyeti var mı? | Acil ihtiyaç / Planlı bütçe dönemi / Acele yok / Sadece bilgi topluyor |
| 8 | Kişisel gözlemler | Serbest not alanı — "futbol sever", "sabah aramaları tercih ediyor", "asistanı üzerinden iletiyor" vs. |

- Tüm alanlar opsiyonel — bilmiyorsan boş bırak
- Sonraki aramalarda bu bilgiler kontakt kartında görünür
- "Kişisel gözlemler" alanı en değerli — satışta bağ kurma için

#### Arama Notu Girişi
[Ara] butonuna basıldığında veya arama yapıldıktan sonra:
- Not alanı açılır
- Sonuç seçimi: Toplantı Alındı / Tanışıldı / Ulaşılamadı / Yanlış Numara / İlgilenmiyor
- Toplantı alındıysa tarih seçici
- Tüm aksiyonlar otomatik loglanır (tarih, saat, sonuç, not)

#### Cold Mail Seçim Bölümü
Firma listesinin altında veya yan panelde cold mail bölümü:

**Adım 1 — Kontakt Seçimi:**
- Email adresi olan ve telefonu olmayan kontaktlar listelenir
- Checkbox ile seçim yapılır (toplu veya tek tek)
- Her kontakta firma adı ve ünvanı görünür

**Adım 2 — Şablon/Başlık Atama:**
- Seçilen kontaklara hangi mail şablonu gidecek
- Hangi başlık kullanılacak
- Ünvana özel mi jenerik mi
- Ek dosya seçimi
- Toplu atama yapılabilir (ör: tüm IT Müdürlerine Şablon A)
- Tek tek de değiştirilebilir

**Adım 3 — Onay ve Tetikleme:**
- Seçimlerin özet görünümü: kaç kişiye, hangi şablonla, hangi günler
- **Hafta Boyunca Gönder** butonu → Strateji onaylanır
- App ve Extension bu stratejiyi hafta boyunca otomatik uygular
- Salı: İlk mailler gider, Cuma: İkinci mailler gider
- Günde max 12, 20-40 dk arası
- Gönderim durumu canlı takip edilebilir

#### Hafta Sonu — Otomatik Log
Hafta bittiğinde o haftanın tüm aktiviteleri otomatik loglanır:
- Kaç arama yapıldı, sonuçları ne
- Kaç cold mail gönderildi, kaç dönüş geldi
- Kaç LinkedIn isteği gönderildi, kaç kabul geldi
- Kaç toplantı alındı
- Kaç firma düzenli takibe eklendi
- Hangi firmalarla ne konuşuldu (not özetleri)
- Pipeline'da ne değişti
- To-do tamamlanma oranı

Bu log değiştirilemez (immutable) — performans analizi için güvenilir veri oluşturur.

### 12.3 Firma Listesi (Hedef Seçme ve Filtreleme Merkezi)

Bu sayfa, firmaları filtreleyip hedef haftaya atamanın ana merkezidir.

#### Hızlı Filtre Presetleri (Tek Tıkla)
- **Tümü** — Tüm firmalar
- **🎯 Cihaz Yok (Hedef)** — AVM'den "boş" olarak gelen firmalar, direkt hedef müşteri
- **📞 Telefon Var** — Lusha'dan telefon numarası mevcut kontaktı olan firmalar
- **❌ Telefon Yok** — Henüz telefon numarası bulunamamış firmalar
- **🎯📞 Cihaz Yok + Tel Var** — En ideal hedefler: hem cihaz yok hem aranabilir
- **⏳ Haftaya Atanmamış** — Henüz hiçbir haftaya hedef olarak atanmamış firmalar
- **⚔️ Cihaz Var (Rakip)** — İçinde rakip cihaz olan firmalar

#### Metin Araması
- Firma adı ile arama (Türkçe karakter destekli)
- Filtreler + arama birlikte çalışır

#### Hedef Haftaya Atama Barı
Tablonun üstünde sabit bir bar:
- **Bu Hafta** / **Sonraki Hafta** / **+2 Hafta** hızlı butonları
- **Özel hafta numarası** girme alanı (tarih aralığı gösterilir)
- Her firma satırında **"→ H7"** butonu — tıkla, o firmayı seçili haftaya ata

#### Tablo Sütunları
- Firma adı (renkli durum noktası: 🟢 telefon var / 🟡 kontakt var / 🔴 kontakt yok)
- Çatı firma
- Cihaz durumu (Yok ✓ / Marka adı)
- Pipeline aşaması
- Kontakt özeti (📞 kaç telefon, ✉️ kaç email, 🔗 kaç LinkedIn)
- Mevcut hedef haftası
- Hedef ata butonu

#### Tipik Kullanım Akışı
1. "🎯 Cihaz Yok" filtresine tıkla → AVM'den gelen boş firmaları gör
2. İnce, firmayı tanı, "→ H7" butonuyla bu haftaya veya gelecek haftaya ata
3. "📞 Telefon Var" filtresine tıkla → Lusha'dan telefonu olan firmaları gör
4. Bunları da ilgili haftaya ata
5. Haftalık sayfaya geç, aramaya başla

### 12.4 Çatı Firma Görünümü
- Çatı firmalar listesi
- Her çatı firmanın alt firmaları
- Alt firmaların durumları
- Çatı firma kontaktları

### 12.5 Arama Günü Ekranı
- Bugün aranacaklar listesi
- Arama sırası
- Her firma için: kontakt bilgileri, son notlar, geçmiş
- Arama sonucu giriş formu
- Gün sonu özeti

### 12.6 Cold Mail Yönetimi
- Şablon oluşturma/düzenleme
- Aktif kampanyalar
- Gönderim takvimi
- Performans raporu (şablon/başlık/ünvan bazlı)

### 12.7 LinkedIn Outreach Yönetimi
- Gönderilen istekler listesi
- Bekleyenler / Kabul edilenler
- Mesaj atılacaklar
- Senaryo performans raporu

### 12.8 Rakip Analizi Sayfası
- Rakip firma kartları
- Karşılaştırma tablosu
- Firma-rakip ilişki haritası
- Rakip bazlı fırsat analizi

### 12.9 Demo & Closing Takip
- Aktif demo süreçleri
- Her sürecin aşama durumu
- Sözleşme / Jira / Kargo / Kurulum / Veri akışı takibi
- Teklif detayları

### 12.10 To-Do Listesi
- Otomatik üretilen to-do'lar
- Manuel eklenen to-do'lar
- Bugün yapılacaklar
- Bu hafta yapılacaklar
- Geciken to-do'lar (kırmızı)
- Tamamlanan to-do'lar

### 12.11 Veri Yönetimi (Global — Hafta Bağımsız)

Bu sayfa haftalık akıştan bağımsızdır. Header'dan veya navigasyondan her zaman erişilebilir.

#### Firma Ekleme
- Manuel firma ekleme formu
- Çatı firma mı alt firma mı seçimi
- Hedef haftaya atama

#### Excel İçe Aktarma
- Lusha Excel yükleme (tek/toplu)
- AVM Excel yükleme
- **Yükleme sonrası tablo önizlemesi** — içinden ne çıktığını görmek için Excel verisi tablo olarak ekranda gösterilir, onaylandıktan sonra sisteme aktarılır
- İçe aktarma raporu (kaç firma eklendi, kaç kontakt eklendi, kaç güncellendi, kaç birleştirildi)

#### Fuzzy Match Onay Ekranı
- Benzer firma taraması (her zaman tetiklenebilir)
- Eşleşme adayları tek tek sunulur
- AYNI / FARKLI butonları
- Reddedilenler blacklist'e eklenir, tekrar sorulmaz

### 12.12 Performans Analizi ve Raporlama

#### Çeyrek Dilim Görünümü
- Yılın 4 çeyreği seçilebilir: Q1 (Hafta 1-13), Q2 (14-26), Q3 (27-39), Q4 (40-52)
- Her çeyrek içinde hafta hafta performans tablosu

#### Hafta Hafta Karşılaştırma Tablosu

Her hafta için aşağıdaki metrikler yan yana:

| Metrik | H1 | H2 | H3 | ... | H13 | Çeyrek Ort. |
|--------|----|----|----|----|------|-------------|
| Hedef firma sayısı | | | | | | |
| Yapılan arama | | | | | | |
| Ulaşılan kişi | | | | | | |
| Toplantı alınan | | | | | | |
| Arama → Toplantı oranı | | | | | | |
| Gönderilen cold mail | | | | | | |
| Cold mail dönüş | | | | | | |
| Cold mail dönüşüm % | | | | | | |
| LinkedIn istek gönderilen | | | | | | |
| LinkedIn kabul | | | | | | |
| LinkedIn kabul oranı % | | | | | | |
| Yeni firma eklenen | | | | | | |
| Yeni kontakt eklenen | | | | | | |
| Düzenli takipteki firma | | | | | | |
| Demo yapılan | | | | | | |
| Teklif verilen | | | | | | |
| Sözleşme imzalanan | | | | | | |
| To-do tamamlanma % | | | | | | |

#### Trend Grafikleri
- Haftalık arama sayısı trendi (çizgi grafik)
- Toplantı dönüşüm oranı trendi
- Cold mail performansı trendi
- Pipeline büyüme trendi
- Kontakt toplama hızı trendi

#### Çeyrekler Arası Kıyaslama
- Q1 vs Q2 vs Q3 vs Q4 yan yana
- Her çeyrekteki ortalamaların karşılaştırması
- İyileşme/kötüleşme gösteren metrikler renkli vurgulanır (yeşil ↑ / kırmızı ↓)

#### Detay Drilldown
- Herhangi bir haftaya tıklandığında o haftanın tam loguna gidilir
- O hafta hangi firmayla ne yapıldı, tüm notlar, tüm sonuçlar
- O hafta hangi cold mail şablonu kullanıldı, dönüşüm oranları

#### Kanal Performansı
- Telefon araması vs Cold mail vs LinkedIn — hangi kanal daha çok toplantı getiriyor
- Şablon bazlı performans: Hangi mail şablonu en iyi dönüşüm
- Başlık bazlı performans: Hangi başlık en çok açılma/dönüş
- Ünvan bazlı performans: Hangi ünvandaki kişiler daha çok dönüyor
- LinkedIn: Notlu vs notsuz istek kabul oranı karşılaştırması

---

## 13. TEKNİK MİMARİ

### 13.1 Veritabanı — Supabase (PostgreSQL)
- **Supabase** kullanılacak (hosted PostgreSQL + Auth + API)
- **ÖNEMLİ:** Dashboard'da 1000 satır limiti kaldırılmalı (Supabase ayarlarından)
- Hem Extension hem App aynı Supabase DB'sine bağlanır
- Real-time sync: Extension'da yapılan değişiklik App'te anında görünür
- Row Level Security: Tek kullanıcı olsa da güvenlik katmanı

### 13.2 Chrome Extension
- Frontend: HTML/CSS/JS (veya React)
- Storage: Supabase JS Client ile doğrudan DB'ye bağlantı
- Popup: Firma arama, hızlı işlemler
- Content Script: LinkedIn sayfasında kontakt ekleme

### 13.3 Web App
- Frontend: React
- Backend: Supabase Edge Functions (veya ayrı Node.js/Python backend gerekirse)
- Database: Supabase (PostgreSQL)
- Excel parsing: SheetJS
- Fuzzy matching: Türkçe-aware Levenshtein distance (tr-TR locale)
- Auth: Tek kullanıcı (Supabase Auth)

### 13.4 Cold Mail API — Microsoft Graph API (Outlook)
- **Gönderim adresi:** tugay.demircan@remvisionlab.com
- **Altyapı:** Microsoft 365 / Outlook — Microsoft Graph API kullanılacak
- Mailler direkt şirket mailinden çıkar
- "Gönderildi" klasöründe görünür
- Spam riski düşük (kendi domain'inden gidiyor)
- Mail kuyruk sistemi (Supabase Edge Function veya ayrı worker)
- Zamanlama: Salı + Cuma
- Rate limiting: 12/gün, 20-40 dk rastgele pause
- Şablon değişken desteği (firma adı, kişi adı, ünvan)
- Ek dosya desteği
- Extension'dan konfigüre edilir: şablon, başlık, ek, ünvan seçimi

### 13.5 Veritabanı Tabloları
- `companies` — Firma bilgileri, cihaz durumu, çatı firma ilişkisi
- `contacts` — Kontakt bilgileri, firma bağlantısı
- `parent_companies` — Çatı firma - alt firma ilişkileri
- `pipeline_stages` — Her firmanın pipeline geçmişi
- `weekly_targets` — Haftalık hedef atama
- `call_logs` — Arama kayıtları ve notları
- `todos` — To-do listesi (otomatik + manuel)
- `cold_mail_templates` — Mail şablonları
- `cold_mail_campaigns` — Kampanya bilgileri
- `cold_mail_sends` — Tek tek gönderim kayıtları
- `cold_mail_responses` — Dönüş kayıtları
- `linkedin_outreach` — LinkedIn istek/mesaj takibi
- `linkedin_notes` — Bağlantı isteğinde kullanılan notlar
- `competitors` — Rakip firma bilgileri
- `company_competitors` — Firma-rakip ilişkisi (hangi firmada hangi rakip var)
- `avm_visits` — AVM ziyaret kayıtları
- `excel_imports` — Yüklenen dosyaların kaydı
- `demo_processes` — Demo ve closing süreci takibi
- `offers` — Teklif detayları (fiyat, cihaz sayısı, şube)
- `contact_profiles` — Kontakt karakter profili (iletişim tarzı, karar yapısı, gözlemler)
- `fuzzy_match_blacklist` — Reddedilen eşleştirmeler (tekrar sorulmasın)

---

## 14. GELİŞTİRME ÖNCELİK SIRASI

### Faz 1 — Temel Altyapı
- [ ] Veritabanı ve veri modeli kurulumu
- [ ] Excel yükleme (Lusha + AVM)
- [ ] Fuzzy matching ile firma birleştirme
- [ ] Firma kartı görüntüleme
- [ ] Kontakt ekleme/düzenleme/silme
- [ ] Çatı firma yapısı

### Faz 2 — Pipeline ve Planlama
- [ ] Pipeline aşama yönetimi
- [ ] Haftalık hedef atama
- [ ] Arama günü ekranı
- [ ] Arama notu ekleme
- [ ] Otomatik to-do sistemi
- [ ] Düzenli takip yönetimi

### Faz 3 — Chrome Extension
- [ ] Firma arama ve kart görüntüleme
- [ ] LinkedIn entegrasyonu (kontakt ekleme)
- [ ] Hızlı not ve düzenleme
- [ ] LinkedIn/Website hızlı erişim
- [ ] Excel yükleme

### Faz 4 — İletişim Otomasyonu
- [ ] Cold mail şablon sistemi
- [ ] Mail API entegrasyonu
- [ ] Otomatik gönderim (zamanlama + rate limit)
- [ ] LinkedIn outreach takibi
- [ ] Mail/LinkedIn performans metrikleri

### Faz 5 — Demo & Closing Süreci
- [ ] Demo sonrası süreç takibi
- [ ] Sözleşme / Jira / Kargo / Kurulum / Veri akışı
- [ ] Teklif yönetimi
- [ ] Teknik sorun takibi

### Faz 6 — Rakip Analizi
- [ ] Rakip firma kartları
- [ ] Karşılaştırma tablosu
- [ ] Firma-rakip ilişkisi
- [ ] Kontrat bitiş takibi

### Faz 7 — Dashboard ve Raporlama
- [ ] Ana dashboard (makro tablo)
- [ ] Tüm görsel metrikler (bölüm 10)
- [ ] Haftalık/aylık raporlar
- [ ] Performans karşılaştırmaları

---

*Bu doküman, sistemin eksiksiz haritasıdır. Her bölüm bağımsız referans alınabilir. Geliştirme sırasında bu doküman canlı tutulur ve güncellenir.*
