# KahveGo Mobil Kahve Sipariş Uygulaması

## Görev 1: Mobil sipariş akışı (Sözde kod)

Seçenek B tercih edilmiştir.

```text
BAŞLA
  Uygulamayı aç ve oturum durumunu kontrol et.
  EĞER kullanıcı giriş yapmamış İSE
    Giriş Ekranı'na yönlendir.
    DÖNGÜ giriş başarılı olana kadar
      Kullanıcı bilgilerini al ve doğrula.
      EĞER kullanıcı vazgeçerse İSE BİTİR.
      EĞER bilgiler geçersiz İSE hata mesajı göster.
    DÖNGÜ SONU
  DEĞİLSE
    Ürün ekranına geç.
  KOŞUL SONU

  Ürünleri göster ve boş bir sepet oluştur.
  DÖNGÜ kullanıcı ürün seçmeye devam ettiği sürece
    Kahveyi, boyutunu ve pozitif tam sayı adet bilgisini al.
    Ürünü sepete ekle ve sepet tutarını güncelle.
  DÖNGÜ SONU

  EĞER sepet boş İSE
    "Sepetiniz boş" mesajını göster ve BİTİR.
  KOŞUL SONU
  Sepet özetini ve toplam tutarı göster.
  EĞER kullanıcı siparişi onaylamaz İSE BİTİR.

  Sunucudan güncel cüzdan bakiyesini al.
  EĞER bakiye sorgusu başarısız İSE
    Hata mesajını göster ve BİTİR.
  KOŞUL SONU
  EĞER bakiye < sepet tutarı İSE
    "Bakiye Yükle" uyarısını ve bakiye yükleme ekranını göster.
    Bu sipariş girişimini sonlandır; yüklemeden sonra yeniden onay iste.
  DEĞİLSE
    Sipariş paketini POST /api/v1/siparisler adresine gönder.
    SUNUCUDA:
      Oturumu ve ürün/adet bilgilerini doğrula.
      Toplamı sunucudaki güncel ürün fiyatlarıyla yeniden hesapla.
      Tek bir veritabanı işlemi başlat ve cüzdan kaydını kilitle.
      EĞER güncel bakiye < doğrulanmış toplam İSE
        İşlemi geri al ve yetersiz bakiye hatası döndür.
      DEĞİLSE
        Sipariş kaydını oluştur.
        Yeni bakiye = güncel bakiye - doğrulanmış toplam.
        Yeni bakiyeyi kaydet ve işlemi tamamla.
        201 Created ile sipariş bilgilerini ve yeni bakiyeyi döndür.
      KOŞUL SONU
      EĞER kayıt sırasında hata oluşursa İSE işlemi geri al.
    UYGULAMADA:
      EĞER yanıt 201 İSE
        Sipariş onayını ve yeni bakiyeyi göster, sepeti temizle.
      DEĞİLSE
        İlgili hata mesajını göster; sepeti koru.
        EĞER yanıt 401 İSE Giriş Ekranı'na yönlendir.
        EĞER hata yetersiz bakiye İSE "Bakiye Yükle" uyarısını göster.
      KOŞUL SONU
  KOŞUL SONU
BİTİR
```

Mobil uygulamadaki bakiye kontrolü kullanıcıya erken bilgi verir; son kontrol sunucuda yapılır. Sipariş kaydı ve bakiye düşümü aynı veritabanı işlemi içinde tamamlanır; hata halinde ikisi de geri alınır. Ağ zaman aşımı siparişin kesin başarısız olduğu anlamına gelmediğinden, POST isteği körlemesine tekrarlanmamalıdır.

## Görev 2: REST API uç noktaları ve JSON tasarımı

### 2.1 Sipariş oluşturma

**HTTP metodu:** `POST`  
**Endpoint:** `/api/v1/siparisler`

**İstek başlıkları:**

```http
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
```

**Örnek request body:**

```json
{
  "urunler": [
    {
      "urun_id": "kahve_001",
      "kahve_adi": "Latte",
      "boyut": "ORTA",
      "adet": 2,
      "birim_fiyat": 75.00
    }
  ],
  "toplam_tutar": 150.00,
  "para_birimi": "TRY"
}
```

İstek içindeki fiyat ve toplam yalnızca istemcinin gördüğü tutarı temsil eder; sunucu ürün kimliği ve boyutuna göre kendi fiyat kaydını kullanır. Kullanıcı kimliği Bearer token üzerinden belirlenir. Para hesaplamaları kuruş cinsinden tam sayı veya sabit hassasiyetli ondalık türle yapılır.

**Başarılı yanıt:** `201 Created`

```http
Content-Type: application/json
Location: /api/v1/siparisler/sip_1001
```

```json
{
  "siparis_id": "sip_1001",
  "durum": "ONAYLANDI",
  "toplam_tutar": 150.00,
  "onceki_bakiye": 185.50,
  "kalan_bakiye": 35.50,
  "para_birimi": "TRY"
}
```

**Giriş yapılmamış, token geçersiz veya süresi dolmuşsa:** `401 Unauthorized`

```http
WWW-Authenticate: Bearer
Content-Type: application/json
```

```json
{
  "hata": "YETKISIZ",
  "mesaj": "Sipariş vermek için giriş yapmalısınız."
}
```

**Yetersiz bakiye:** Bu API tasarımında `409 Conflict` kullanılır; sipariş oluşturulmaz ve bakiye düşülmez.

```json
{
  "hata": "YETERSIZ_BAKIYE",
  "mesaj": "Bakiye Yükle",
  "bakiye": 100.00,
  "gerekli_tutar": 150.00,
  "eksik_tutar": 50.00,
  "para_birimi": "TRY"
}
```

Bozuk JSON için `400 Bad Request`, boş sepet veya geçersiz adet gibi anlamsal doğrulama hataları için `422 Unprocessable Content` döndürülür.

### 2.2 Cüzdan bakiyesi sorgulama

**HTTP metodu:** `GET`  
**Endpoint:** `/api/v1/kullanici/bakiye`

**İstek başlıkları:**

```http
Authorization: Bearer <token>
Accept: application/json
```

GET isteğinde body gönderilmez.

**Başarılı yanıt:** `200 OK`

```http
Content-Type: application/json
Cache-Control: no-store
```

```json
{
  "bakiye": 185.50,
  "para_birimi": "TRY"
}
```

Bu bakiye, yukarıdaki örnek sipariş verilmeden önceki durumu gösterir. Sipariş tamamlandıktan sonraki sorgu `35.50` döndürür.

**Sunucuda beklenmeyen hata:** `500 Internal Server Error`

```json
{
  "hata": "SUNUCU_HATASI",
  "mesaj": "Bakiye şu anda sorgulanamıyor. Lütfen daha sonra tekrar deneyin."
}
```

Bu endpoint de oturum gerektirir; token eksik veya geçersizse `401 Unauthorized` döndürülür. Hata yanıtlarında sunucunun iç hata ayrıntıları paylaşılmaz.

### Mini mülakat sorusu (1 cümle)

GET eşgüçlüdür çünkü aynı isteğin tekrarı sunucudaki bakiyeyi değiştirmez; POST ise bu tasarımda eşgüçlü değildir çünkü aynı sipariş isteğinin tekrarı yeni siparişler ve ek bakiye düşümleri oluşturabilir.

## Görev 3: Clean Code ve SOLID prensip teşhisi

### 3.1 SRP ihlali (2 cümle)

`KahveSiparisYoneticisi`, sepet ve indirim hesabı, ödeme tahsilatı, veritabanına kayıt ve SMS bildirimi gibi farklı değişme nedenlerini tek sınıfta birleştirerek Tek Sorumluluk Prensibi'ni (SRP) ihlal eder. Sınıf `SepetHesaplayici`, `IndirimServisi`, `OdemeServisi`, `SiparisRepository` ve `BildirimServisi` olarak ayrılmalı; `KahveSiparisYoneticisi` yalnızca sipariş sürecini koordine etmelidir.

### 3.2 Yeni müşteri tipi ve OCP

Yeni bir `DOKTOR` müşteri tipi eklemek için mevcut `if-else` zincirini değiştirmek, **Open/Closed Principle (OCP - Açık/Kapalı Prensibi)** ile çelişir: yazılım genişletmeye açık, mevcut kodun değiştirilmesine kapalı olmalıdır. Ortak bir `IndirimStratejisi` arayüzünün öğrenci, öğretmen ve doktor için ayrı uygulamaları oluşturulabilir; uygun strateji dışarıdan verildiğinde hesaplama metodu değişmeden yeni indirim davranışları eklenebilir.

Ek Clean Code notu: Örnekte `tutar * 0.80` indirim miktarını değil, indirimden sonra ödenecek tutarı verdiğinden `indirimHesapla` yerine `indirimliTutarHesapla` adı daha açıktır.

## Görev 4: Git, branching ve GitHub Release

### Uygulanacak akış

1. `kahvego-tasarim` deposunu oluştur ve başlangıç dalını `main` olarak hazırla.
2. `main` üzerinden `feature/kahvego-tasarim` dalını aç.
3. Bu `ODEV.md` dosyasını dalın kök dizinine ekle; Görev 1'de sözde kod seçildiği için ek görsel gerekli değildir.
4. Şu Conventional Commit mesajıyla commit oluştur:

   ```text
   feat: kahvego akis semasi, rest api ve solid analizi
   ```

5. Dalı GitHub'a gönder ve hedefi `main` olan bir Pull Request aç.
6. Aşağıdaki açıklamayla PR'ı birleştir.
7. Birleştirilen `main` üzerinde `v1.2.0` etiketiyle aşağıdaki sürümü yayımla.
8. Gerçek sürüm bağlantısını SoftITO LMS'deki ilgili ödeve teslim et.

### Hazır Pull Request metni

**Başlık:** `feat: kahvego akis semasi, rest api ve solid analizi`

**Açıklama:**

KahveGo sipariş senaryosunun tasarım yanıtlarını tek bir ODEV.md dosyasında toplar. Sipariş akışında oturum ve bakiye kontrolleri, REST API tasarımında sipariş oluşturma ve bakiye sorgulama istekleri, SOLID analizinde SRP ve OCP ihlalleri açıklanır.

- Giriş yapılmadığında giriş ekranına yönlendirme tanımlandı.
- Yetersiz bakiyede siparişin engellenmesi ve bakiye yükleme uyarısı tanımlandı.
- Başlıklar, JSON örnekleri, HTTP durum kodları ve eşgüçlülük yanıtı eklendi.
- Sipariş kaydı ile bakiye düşümünün birlikte tamamlanması açıklandı.
- SRP için sorumlulukların ayrılması, OCP için indirim stratejileri önerildi.

**Doğrulama:** PDF'deki zorunlu maddelerle içerik karşılaştırılır, JSON örnekleri ayrıştırılır ve örnek tutarlar kontrol edilir; bu çalışma bir tasarım belgesidir, çalışan uygulama veya API içermez.

### Hazır Release metni

**Etiket:** `v1.2.0`  
**Hedef:** PR birleştirildikten sonraki `main`  
**Başlık:** `Release v1.2.0 - KahveGo Mimari ve API Tasarımı`

**Sürüm notları:**

- Kullanıcı oturumu, ürün seçimi, sepet onayı ve bakiye kontrolünü içeren sipariş sözde kodu.
- `POST /api/v1/siparisler` ve `GET /api/v1/kullanici/bakiye` için HTTP ve JSON tasarımları.
- GET ve POST eşgüçlülük karşılaştırması.
- Tek Sorumluluk ve Açık/Kapalı prensiplerinin analizi ve iyileştirme önerileri.
- Tüm yanıtları içeren `ODEV.md` belgesi.

### İşlem durumu

GitHub Desktop ile `kahvego-tasarim` yerel deposu ve `main` üzerinden `feature/kahvego-tasarim` dalı oluşturuldu; `ODEV.md` eklendi. Altı JSON örneği ayrıştırılarak ve örnek bakiye hesabı doğrulanarak kontrol edildi. PR ve Release metinleri hazırdır; GitHub üzerinde yayımlama ve LMS teslimi henüz tamamlanmamıştır.
