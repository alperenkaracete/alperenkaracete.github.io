# FastAPI: GET vs POST Mimari Farkları ve Python'da Tip Güvenliği

Bugün OlyKube otonom AI ajan projesinin API altyapısını kurarken, hem HTTP protokolünün doğasıyla hem de Python'ın bellek/tip yönetimiyle ilgili kritik mimari kararları test ettim.

## 1. Olay: GET vs POST ve Request Body Mantığı

Sisteme dışarıdan veri eklemek için Pydantic ile bir veri modeli (`Item`) oluşturdum ve bunu bir `@app.get` uç noktasıyla dışarı açmaya çalıştım. Ancak sistem beklediğim gibi çalışmadı ve benden bir JSON gövdesi istemedi.

**Kök Neden Analizi (Root Cause):**
Sorunun FastAPI'den değil, HTTP protokolünün ta kendisinden kaynaklandığını fark ettim.
* **GET Metodu:** Sunucudan veri okumak içindir. URL üzerinden parametre alır ama doğası gereği bir **veri gövdesi (JSON body) taşımaz.**
* **POST Metodu:** Sunucuya yeni veri göndermek içindir. Veriler JSON formatında isteğin gövdesinde (body) gider.

**Çözüm:** Yeni veri yaratacağımız ve JSON olarak veri alacağımız için endpoint'i `@app.post` olarak güncelledim.

---

## 2. Olay: C++ Disiplinini Python'a Taşımak (Type Safety)

Verileri kalıcı bir veritabanına yazmadan önce, mantığı kavramak için RAM üzerinde geçici bir sözlük (`fake_db = {}`) oluşturdum. Ancak dinamik tipli bir dil olan Python'da değişkenleri bu şekilde başıboş bırakmanın, görünmez hatalara (bug) davetiye çıkardığını biliyorum. 

C++ geçmişimden gelen katı bellek ve tip güvenliği (Type Safety) alışkanlıklarımı, bu projede Python'a iki farklı katmanda uyguladım:

### Katman 1: Dış Güvenlik (Pydantic ile Çalışma Zamanı Doğrulaması)
Dışarıdan (internetten) gelen ne olduğu belirsiz bir JSON verisini doğrudan sisteme almak yerine `class Item(BaseModel)` şablonundan geçirdim. Böylece kötü niyetli veya hatalı bir isteğin (örneğin fiyat kısmına harf girilmesinin) kodumu çökertmesini engelledim. FastAPI bunu otomatik olarak yakalayıp kullanıcıya hata döndürüyor.

### Katman 2: İç Güvenlik (Type Hinting ile Bellek Yönetimi)
Geçici veritabanımı standart bir Python sözlüğü olarak bırakmak yerine, Pydantic modellerimle birleştirdim. 

* **Kötü Kullanım:** `fake_db = {}` (İçinde ne tutulduğu belli değil).
* **Doğru Mimari:** `fake_db: dict[str, Item] = {}`

Bu tanımlama sayesinde, aslında arka planda C++'taki **`std::unordered_map<std::string, Item*>`** mantığına benzer bir yapı kurmuş oldum. Bu sözlüğün anahtarlarının kesinlikle metin (string), değerlerinin ise bellekteki Pydantic objelerimin referansları (pointer) olduğunu IDE'me ve benden sonra kodu okuyacak geliştiricilere açıkça belirttim.

## Sonuç
Modern backend sistemleri inşa ederken, HTTP protokol standartlarına (REST) uymak işin sadece bir yüzü. Asıl kaliteyi belirleyen şey; C++ gibi statik dillerin derleme zamanı (compile-time) disiplinini, Python'ın çalışma zamanı (run-time) esnekliğiyle doğru noktalarda birleştirebilmek.
