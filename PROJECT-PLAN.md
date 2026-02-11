# RemVision Sales Pipeline — Proje Planı (Claude Code)

---

## MEVCUT DURUM

### Çalışan Şeyler
- Tek HTML dosyası, tarayıcıda açılıyor (React + Babel + SheetJS CDN)
- localStorage ile veri kalıcılığı
- AVM Excel (.xlsx) import + önizleme
- Lusha Excel (.xlsx/.csv) import + önizleme (çoklu dosya)
- Firma ekleme, kontakt ekleme/düzenleme/silme
- Haftalık ekran: tıklanabilir stat kartları, filtreler, firma kartları
- Firmalar sayfası: filtre presetleri, hedef haftaya atama
- Dashboard: temel istatistikler, pipeline dağılımı
- Arama notu + karakter profili formu
- Çatı firma ataması (Firmalar sayfasında dropdown)
- Cihaz durumu: Yok / Var / Bilinmiyor / Bizde
- To-Do sayfası (otomatik + manuel)
- Düzenli takip bölümü haftalık ekranda

### Bilinen Hatalar / Eksikler
1. **HTML dosyası Babel ile runtime compile yapıyor** — yavaş, production build'e çevrilmeli
2. **Excel import** — bazı edge case'lerde sütun eşleştirme hataları olabilir
3. **Fuzzy match ekranı** — henüz yapılmadı (firma birleştirme UI'ı yok)
4. **Kontakt önceliklendirme** — drag-and-drop sıralama yok
5. **Cold mail sistemi** — henüz yok
6. **LinkedIn outreach tracking** — henüz yok
7. **Rakip analizi** — temel var ama detaylı kart ve karşılaştırma tablosu yok
8. **Demo & kapanış süreci** — henüz yok
9. **Performans raporu** — hafta bazlı karşılaştırma yok
10. **AVM fill-down** — SheetJS merged cell'leri düz okuyabilir, AVM sütunu fill-down yapılmalı

---

## MİMARİ KARAR

### Şimdilik: Tek Sayfa Uygulama (SPA)
```
sales-pipeline/
├── index.html
├── package.json
├── vite.config.js
├── src/
│   ├── main.jsx              # Entry point
│   ├── App.jsx               # Ana layout + routing
│   ├── store.js              # localStorage data yönetimi
│   ├── utils/
│   │   ├── turkish.js        # Türkçe karakter utils
│   │   ├── fuzzy.js          # Levenshtein + similarity
│   │   ├── week.js           # Hafta hesaplama
│   │   └── uid.js            # ID generator
│   ├── parsers/
│   │   ├── lushaParser.js    # Lusha CSV/XLSX parse
│   │   ├── avmParser.js      # AVM Excel parse
│   │   └── excelUtils.js     # Ortak Excel utils
│   ├── components/
│   │   ├── Header.jsx        # Üst bar + global butonlar
│   │   ├── StatCard.jsx      # İstatistik kartı
│   │   ├── Badge.jsx         # Badge bileşenleri
│   │   ├── CompanyCard.jsx   # Firma kartı (genişleyen)
│   │   ├── ContactTable.jsx  # Kontakt tablosu (yerinde edit)
│   │   ├── ImportModal.jsx   # Excel import + önizleme modal
│   │   ├── CallLogModal.jsx  # Arama notu modal
│   │   ├── CharProfileModal.jsx  # Karakter profili modal
│   │   ├── AddCompanyModal.jsx   # Firma ekleme modal
│   │   └── AddContactModal.jsx   # Kontakt ekleme modal
│   ├── pages/
│   │   ├── WeeklyView.jsx    # Haftalık çalışma ekranı
│   │   ├── Companies.jsx     # Firma listesi + filtre + atama
│   │   ├── Dashboard.jsx     # Dashboard + istatistikler
│   │   ├── Competitors.jsx   # Rakip analizi
│   │   ├── Todos.jsx         # To-Do listesi
│   │   ├── ColdMail.jsx      # Cold mail yönetimi (Faz 2)
│   │   └── LinkedIn.jsx      # LinkedIn outreach (Faz 2)
│   └── styles/
│       └── theme.js          # Renk, font, stil sabitleri
```

### Sonra: Supabase Backend
- Supabase PostgreSQL veritabanı
- Row Level Security
- Realtime sync
- Edge Functions (cold mail scheduler)
- Auth (tek kullanıcı)

---

## EXCEL DOSYA YAPILARI (Referans)

### AVM Analiz Excel
```
Sütunlar: AVM | Cihaz | Firma | REFERANS FOTO | Tarih
- AVM: Fill-down (merged cell, üstten devam eder)
- Cihaz: "boş" = cihaz yok, marka = rakip, "" = bilinmiyor, "biz"/"bizde" = bizim
- Firma: Mağaza adı
- REFERANS FOTO: Fotoğraf referans kodu
- Tarih: Ziyaret tarihi
- 197 satır, 2 AVM (İstinye Park, Emaar)
```

### Lusha 25'lik CSV Export
```
56 sütun, BOM karakter (\ufeff) var başında
Önemli sütunlar:
- First Name + Last Name (ayrı sütunlar)
- Phone 1, Phone 1 Type, Phone 2, Phone 2 Type
- Work Email, Direct Email, Additional Email 1
- Job Title, Seniority
- LinkedIn URL
- Company Name, Company Website, Company linkedin URL, Company Domain
```

### Combined Contacts XLSX
```
50 sütun, 2752 satır
- Aynı Lusha sütunları + ek alanlar:
- _manuel: true (elle eklenmiş satırlar)
- _yeniFirma: true (sadece firma ismi, kontakt yok)
- _tarih: ekleme tarihi
- Email ve Work Email ayrı sütunlarda
- Telefon formatı: "05497446696" veya "+90 533 554 20 70"
```

---

## CİHAZ DURUMU KURALLARI

| Durum | Kaynak | Badge |
|-------|--------|-------|
| `none` | Sadece AVM Excel ("boş", "yok") | ✅ Cihaz Yok |
| `competitor` | Sadece AVM Excel (marka adı) | 🔴 Marka adı |
| `ours` | AVM Excel ("biz", "bizde") | 🔵 Bizde |
| `unknown` | Lusha/manuel eklenen, AVM'de yok | ❓ Bilinmiyor |

---

## GELİŞTİRME FAZLARI

### FAZ 1 — Altyapı ve Temel Düzeltmeler (ÖNCELİK)
- [ ] Vite + React projesi kurulumu (Babel runtime yerine proper build)
- [ ] Mevcut kodu dosyalara ayır (yukarıdaki yapıya göre)
- [ ] SheetJS düzgün npm import (`npm install xlsx`)
- [ ] localStorage store düzgün çalıştığını doğrula
- [ ] AVM import: fill-down düzelt (AVM sütunu merge cell)
- [ ] Lusha import: BOM karakter, sütun eşleştirme doğrula
- [ ] Firma-only satırlar (isim olmayan) düzgün firma oluşturuyor mu doğrula
- [ ] Tüm importlarda önizleme tablosu düzgün render ediliyor mu
- [ ] "İçe Aktar" butonu hata vermeden çalışıyor mu
- [ ] Hot reload çalışır durumda (geliştirme hızı için)

### FAZ 2 — Haftalık Ekran İyileştirmeleri
- [ ] Firmalar arası geçişte expand/collapse düzgün çalışması
- [ ] Kontakt sıralama (drag-and-drop veya numara ile)
- [ ] Arama sonrası pipeline otomatik güncelleme
- [ ] Karakter profili daha önce girilmişse ön-doldurma
- [ ] Düzenli takip firmaları: son arama tarihi, gün sayısı
- [ ] Interleaved arama sırası gösterimi (1 takip, 1 yeni)

### FAZ 3 — Firmalar Sayfası İyileştirmeleri
- [ ] Çoklu firma seçip toplu haftaya atama
- [ ] Firma kartına tıklayınca detay görünümü (aynı haftalık ekrandaki gibi)
- [ ] AVM bazlı filtreleme
- [ ] Sıralama seçenekleri (isim, tarih, kontakt sayısı)

### FAZ 4 — Fuzzy Match Sistemi
- [ ] Firma ismi benzerlik taraması (tüm firmalar arası)
- [ ] Match adayları ekranı: AYNI / FARKLI butonları
- [ ] AYNI: Lusha ismi kabul, diğeri mapped, veriler birleştirilir
- [ ] FARKLI: Blacklist'e ekle, bir daha sorma
- [ ] İstediğin zaman tetiklenebilir (sadece import sonrası değil)
- [ ] Türkçe karakter destekli Levenshtein mesafesi

### FAZ 5 — Cold Mail Sistemi
- [ ] Mail şablon oluşturma (konu + gövde + ek dosya)
- [ ] Ünvana göre şablon seçimi
- [ ] Mail kuyruğu yönetimi
- [ ] Microsoft Graph API entegrasyonu
- [ ] Gönderim takvimi: Salı + Cuma, 12/gün, 20-40dk arası
- [ ] Gönderim logları ve metrikler
- [ ] Şablon bazlı dönüşüm oranı

### FAZ 6 — LinkedIn Outreach
- [ ] Bağlantı isteği gönderme takibi
- [ ] Not ile / notsuz senaryo kaydı
- [ ] Kabul takibi
- [ ] Mesaj gönderme takibi
- [ ] Performans metrikleri (kabul oranı, not vs notsuz)

### FAZ 7 — Rakip Analizi Detay
- [ ] Rakip firma kartları (veri kaynağı, donanım, fiyat, güçlü/zayıf)
- [ ] Rakip karşılaştırma tablosu
- [ ] Firma-rakip ilişkisi (hangi firmada hangi rakip)
- [ ] Kontrat bitiş tarihi takibi

### FAZ 8 — Demo & Kapanış Süreci
- [ ] Teklif yönetimi (fiyat, cihaz sayısı, şube bilgisi)
- [ ] Sözleşme durumu takibi
- [ ] Jira kaydı takibi
- [ ] Kargo / lojistik takibi
- [ ] Kurulum takibi
- [ ] Veri akışı kontrolü
- [ ] Teknik sorun takibi

### FAZ 9 — Dashboard & Raporlama
- [ ] Hafta bazlı performans karşılaştırma tablosu
- [ ] Çeyrek (Q1-Q4) görünümü
- [ ] Cold mail metrikleri dashboard'da
- [ ] LinkedIn metrikleri dashboard'da
- [ ] Kontakt gap raporu
- [ ] Pipeline flow görselleştirme

### FAZ 10 — Supabase Migration
- [ ] Supabase proje kurulumu
- [ ] Tablo şemaları oluşturma
- [ ] localStorage → Supabase migration script
- [ ] Realtime sync
- [ ] Row Level Security
- [ ] Auth

### FAZ 11 — Chrome Extension
- [ ] Manifest v3 yapısı
- [ ] Popup: firma arama, hızlı işlemler
- [ ] Content Script: LinkedIn sayfasında "Kontakt Ekle" butonu
- [ ] Supabase bağlantısı (aynı DB)
- [ ] Quick note, pipeline güncelleme

---

## CLAUDE CODE İÇİN TALİMATLAR

### Projeyi başlatırken:
```bash
npm create vite@latest sales-pipeline -- --template react
cd sales-pipeline
npm install xlsx
npm install # diğer dependencies
```

### Mevcut app.jsx'i parçalarken:
1. Önce `src/styles/theme.js` — renk ve stil sabitleri
2. Sonra `src/utils/` — turkish.js, fuzzy.js, week.js, uid.js
3. Sonra `src/store.js` — localStorage yönetimi
4. Sonra `src/parsers/` — lushaParser.js, avmParser.js
5. Sonra `src/components/` — küçük bileşenler
6. Sonra `src/pages/` — sayfa bileşenleri
7. Son olarak `src/App.jsx` ve `src/main.jsx`

### Stil yaklaşımı:
- Inline styles kullanılıyor (mevcut kodda)
- İsterseniz Tailwind'e geçilebilir ama öncelik işlevsellik
- Tema sabitleri `theme.js`'den import edilmeli

### Test ederken:
- AVM Excel: `/mnt/user-data/uploads/revl_avm_analiz.xlsx`
- Lusha CSV: `/mnt/user-data/uploads/Export_Contacts_2026-02-11.csv`
- Combined XLSX: `/mnt/user-data/uploads/combined_contacts.xlsx`
- Bu dosyaları import testlerinde kullan

### Önemli kurallar:
- Türkçe karakter dönüşümü her zaman `turkishLower()` ile yapılmalı
- `İ` ↔ `i` ve `I` ↔ `ı` dönüşümü KRİTİK
- Firma eşleştirme threshold: 0.8 (Lusha), 0.75 (AVM)
- Cihaz durumu sadece AVM'den gelir, Lusha'dan gelen = "unknown"
- Import sonrası haftaya otomatik atama YOK, kullanıcı Firmalar sayfasından atar

---

## SİSTEM TASARIM DOKÜMANI

Tam sistem tasarımı `sales-pipeline-system-v2.md` dosyasında.
İçerik: veri modeli, pipeline aşamaları, haftalık iş akışı, cold mail sistemi,
LinkedIn outreach, rakip analizi, dashboard metrikleri, to-do sistemi,
Chrome Extension özellikleri, web app ekranları, teknik mimari.
