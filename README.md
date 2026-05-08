# SHAPER V0.5

**SHAPER**, OpenClaw ve VPS altyapısı üzerinde geliştirilen hafif, pratik ve kişisel bir takip agent’ıdır.  
Telegram üzerinden sigara kullanımı, yemek/kalori girişi, kilo değişimi ve günlük durum raporlarını takip etmek için tasarlanmıştır.

SHAPER’ın amacı genel amaçlı bir sohbet botu olmak değildir. Daha çok, günlük verileri düzenli şekilde kaydeden, doğal dili gerektiğinde anlayabilen ama kritik verileri kontrollü ve deterministik şekilde saklayan bir kişisel takip sistemidir. Bu yaklaşım sayesinde sistem hem daha güvenilir çalışır hem de zaman içinde güvenli şekilde geliştirilebilir.

---

## SHAPER Ne Yapar?

SHAPER V0.5 dört temel alana odaklanır:

- Sigara kullanımı takibi
- Yemek ve kalori kaydı
- Kilo takibi
- Günlük ve detaylı raporlar

Kullanıcı SHAPER ile Telegram üzerinden kısa komutlarla veya doğal dilde yazılmış mesajlarla etkileşime geçebilir. Örneğin sigara kullanımı tek harflik bir komutla kaydedilebilirken, yemek girişleri doğal cümlelerle yapılabilir.

SHAPER gelen mesajı işler, canonical JSON verisini günceller ve kullanıcıya kısa, anlaşılır bir yanıt döner.

---

## Projenin Temel Yaklaşımı

SHAPER’ın temel mimari prensibi şudur:

> LLM anlar.  
> Deterministik çekirdek yazar, sayar ve veriyi korur.

İlk denemelerde sistem daha çok prompt-only davranışa dayanıyordu. Bu da basit işlemlerde bile kırılganlık oluşturabiliyordu. V0.5 ile birlikte SHAPER hibrit bir mimariye geçirildi:

- Tekrarlanan basit işlemler deterministik çalışır.
- Yemek gibi doğal dil gerektiren girişler LLM desteğiyle anlaşılır.
- Tüm takip verileri yapılandırılmış JSON/JSONL dosyalarında tutulur.
- Raporlar eski sohbet hafızasından değil, canonical veriden üretilir.

Bu ayrım, SHAPER’a daha sağlam bir temel kazandırır: komutlar hızlı kalır, doğal dil esnekliği korunur ve veri yazımı kontrol altında tutulur.

---

## Temel Özellikler

### Sigara Takibi

SHAPER, sigara kullanımını çok hızlı şekilde kaydetmek için deterministik komutlar destekler:

```text
s
S
sigara
```

Bu komutlar LLM çağrısı gerektirmez. Sistem günlük sigara sayısını artırır, olayı canonical log dosyasına yazar ve kullanıcıya mevcut sayı ile günlük limiti içeren kısa bir durum mesajı döner.

Bu yapı sayesinde sigara takibi hızlı, ucuz ve güvenilir çalışır.

---

### Yemek ve Kalori Kaydı

Yemek kaydı kontrollü bir semantik parser ile yapılır.

Kullanıcı şu tarz doğal cümleler yazabilir:

```text
1 adet tavuk dürüm yedim
yaklaşık 170 gram tavuk yedim
200 gram tavuk yedim
öğlen tavuklu salata yedim
```

SHAPER, mesajın gerçekten bir yemek kaydı olup olmadığını, ne yenildiğini ve yaklaşık kaç kalori olduğunu anlamak için gömülü bir LLM çağrısı kullanır. Ancak LLM doğrudan dosya yazmaz ve veriyi değiştirmez.

LLM yalnızca anlamlandırma yapar. Sonrasında SHAPER’ın deterministik çekirdeği sonucu doğrular, günlük JSON state dosyasını günceller ve yapılandırılmış event log’a kayıt ekler.

Sistem ayrıca yemek kaydı ile yemek hakkında sorulan genel soruları ayırt edebilir. Örneğin:

```text
tavuk kaç kalori?
```

bu mesaj otomatik öğün kaydı olarak değerlendirilmez; bilgi sorusu olarak ele alınır.

---

### Kilo Takibi

Kilo girişleri deterministik şekilde anlaşılır ve kaydedilir.

Desteklenen örnek girişler:

```text
kilo 80.7
kilom 80.7
bugün 80,7 kiloyum
```

SHAPER değeri kilo takip dosyasına yazar ve yapılandırılmış bir geçmiş tutar. Mevcut temiz başlangıç yapısında eski kilo verileri taşınmaz; sistem gerçek kullanım verisiyle sıfırdan beslenmeye hazırdır.

---

### Raporlar

SHAPER hem kısa hem de detaylı rapor üretebilir.

#### Kısa Rapor

```text
rapor
```

Kısa rapor, mevcut günü özetler:

- Sigara sayısı
- Kalori alımı
- Öğün sayısı
- Güncel duruma dair kısa yorum

#### Detaylı Rapor

```text
drapor
detaylı rapor
detayli rapor
```

Detaylı rapor daha geniş bir özet sunmak için tasarlanmıştır. İçeriğinde şunlar yer alabilir:

- Bugünün özeti
- Dünün özeti
- Bu hafta
- Geçen hafta
- Son dönem eğilimleri
- Kilo durumu
- Kısa ve pratik yorum

Raporlar eski sohbet hafızasından değil, canonical JSON verilerinden üretilir.

---

## Veri Modeli

SHAPER canonical takip verilerini yapılandırılmış bir workspace data dizini altında tutar.

Önemli veri dosyaları:

```text
profile.json
today.json
daily/YYYY-MM-DD.json
events/YYYY-MM-DD.jsonl
weight.json
```

Sistem bu dosyaları gerçek veri kaynağı olarak kabul eder.

Sohbet hafızası veya `MEMORY.md`, takip verisi için canonical kaynak olarak kullanılmaz. Bu yaklaşım, bağlam karmaşasını azaltır ve sistemi yedekleme, sıfırlama ve inceleme açısından daha güvenli hale getirir.

---

## Mimari Genel Bakış

SHAPER, OpenClaw üzerinde çalışan bir agent olarak konumlanır ve `shaper-core` adlı local plugin’i kullanır.

Bu plugin, Telegram mesajlarını normal agent akışına düşmeden önce yakalamak için `before_dispatch` hook’unu kullanır. Böylece kritik komutlar doğrudan SHAPER tarafından işlenebilir ve gereksiz LLM kullanımı azaltılır.

Yüksek seviyeli akış:

```text
Telegram mesajı
      ↓
OpenClaw gateway
      ↓
shaper-core before_dispatch hook
      ↓
Deterministik komut işleme
veya
Kontrollü semantik yemek parse işlemi
      ↓
Canonical JSON / JSONL güncellemesi
      ↓
Telegram yanıtı
```

Bu yapı sistemi pratik tutar: hızlı komutlar hızlı kalır, doğal dil esnekliği korunur ve veri yazımı kontrollü şekilde yapılır.

---

## WearOS Entegrasyonu

SHAPER, hızlı kullanım için bir WearOS yardımcı akışı da içerir.

Bu entegrasyon sayesinde kullanıcı, Telegram’ı manuel olarak açmadan bazı temel SHAPER işlemlerini akıllı saat üzerinden tetikleyebilir. Mevcut WearOS arayüzü şu özellikleri destekler:

- Sigara kullanımı gönderimi
- Kısa rapor isteği
- Detaylı rapor isteği
- Sesli komut girişi

Uygulama içinde sade bir komut arayüzü bulunur. Kullanıcı hazır aksiyonları gönderebilir veya mikrofon aracılığıyla özel bir komut verebilir. Ayrıca gönderim ve işlem durumunu anlamayı kolaylaştıran bir aktiflik/sinyal göstergesi yer alır.

Arayüzün alt kısmında iki pratik alan bulunur:

- Gönderilen son mesaj
- SHAPER’dan alınan yanıt

Bu yapı, WearOS uygulamasını tam bir sohbet istemcisinden çok SHAPER için küçük ve hızlı bir uzaktan kumanda gibi konumlandırır.

Teknik olarak WearOS uygulaması, HTTPS üzerinden VPS tarafındaki relay servisiyle haberleşir. Relay, saatten gelen komutu SHAPER/OpenClaw akışına iletir, uygun durumlarda agent yanıtını bekler ve en güncel cevabı tekrar saat arayüzüne döndürür.

Böylece akıllı saatten verilen komut, VPS üzerinde çalışan agent işleme katmanına ve Telegram tabanlı takip sistemine bağlanmış olur.

<img width="405" height="459" alt="image" src="https://github.com/user-attachments/assets/95ab7d53-5053-4ee3-b785-62753f7e6965" />
<img width="421" height="404" alt="image" src="https://github.com/user-attachments/assets/57132e59-697c-4528-8efd-085d80580df4" />


---

## Mevcut Sürüm: V0.5

V0.5, SHAPER’ın ilk stabil çalışan sürümüdür.

Bu sürümde tamamlananlar:

- Deterministik sigara takibi
- Deterministik kilo takibi
- Semantik yemek ve kalori parse sistemi
- Canonical JSON/JSONL veri saklama
- Kısa ve detaylı raporlar
- Sohbet hafızasına bağımlılığın azaltılması
- Daha güvenilir yemek niyeti algılama
- WearOS komut ve yanıt köprüsü
- Gerçek takip verileri için temiz başlangıç reseti

Sistem artık gerçek günlük verilerle kullanılmaya ve V1’e doğru adım adım geliştirilmeye hazırdır.

---

## V1 Yol Haritası

Planlanan geliştirmeler:

- Öğün düzeltme ve silme komutları
- Daha güçlü haftalık ve aylık trend raporları
- Gerçek kullanım verisi biriktikçe daha isabetli kalori tahminleri
- Akıllı saatten yakılan kalori verisi entegrasyonu
- Ana kişisel AI agent’ına günlük özet gönderimi
- Yemek ve sigara azaltma konusunda daha kişiselleştirilmiş öneriler
- Daha güvenli bakım ve yedekleme akışları

Öncelik, SHAPER’ı stabil tutarak pratik özellikleri kontrollü şekilde eklemektir.

---

## Notlar

SHAPER kişisel ve deneysel bir agent projesidir. Tıbbi cihaz değildir ve profesyonel sağlık tavsiyesi yerine geçmez.

Bu proje; günlük farkındalık, öz takip ve yapılandırılmış kişisel veri kaydı için geliştirilmiş kontrollü bir kişisel yapay zeka akışıdır.
