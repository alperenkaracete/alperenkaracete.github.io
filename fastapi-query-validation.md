# FastAPI'de Girdi Doğrulama (Validation) ve Fail-Fast Prensibi

OlyKube projesinde otonom ajanları filtrelemek ve aramak için API uç noktaları (endpoints) tasarlarken, dışarıdan gelen verinin (Query Parameters) güvenliğini sağlamak üzerine testler yaptım. Bu süreçte FastAPI'nin sadece bir web framework'ü değil, aynı zamanda harika bir koruyucu olduğunu gördüm.

İşte bu mimariyi kurarken aldığım notlar ve uyguladığım "Temiz Kod" (Clean Code) prensipleri:

## 1. Bağımlılıkları Ayırmak (Decoupling) ve `Annotated` Kullanımı
Eski nesil FastAPI kodlarında parametre kısıtlamaları doğrudan varsayılan değer olarak (`q: str = Query(...)`) veriliyordu. Ancak bu, yazdığımız fonksiyonu FastAPI dışındaki bir ortamda (örneğin bir test script'inde) kullanılamaz hale getiriyordu.

Modern Python'ın `Annotated` yapısına geçiş yaptım.
```python
async def search_agents(q: Annotated[str | None, Query(min_length=3)] = None):

Bu sayede fonksiyonum saf Python kurallarına sadık kalıyor. Editörüm (IDE) tip hatalarını anında yakalarken, FastAPI de sadece Annotated içindeki "metadata"yı okuyarak doğrulama yapıyor. Fonksiyonun framework'e olan göbek bağını kesmiş oldum.

## 2. Otomatik Tip Dönüşümü (Type Casting) ve Güvenlik
Ajanlara toplu görev atamak için URL'den bir liste almam gerektiğinde sadece list demek yerine list[int] tipini zorunlu kıldım.

URL'den gelen ?id=1&id=2 verisi normalde metindir (string).

list[int] tanımı sayesinde FastAPI arka planda bunu otomatik olarak sayıya çeviriyor. İçine harf karışırsa koda hiç sokmadan 422 Unprocessable Entity hatası basarak sistemi çöp veriden (ve olası SQL Injection denemelerinden) koruyor.

## 3. Dökümantasyon ve Validasyonun Birleşimi
Swagger UI (/docs) için ayrı bir döküman yazmak yerine, bunu kodun içine gömdüm:

Python
Query(
    alias="agent-name", 
    title="Ajan İsmi", 
    pattern="^oly-[a-z0-9]+$"
)
Python'da değişken isimlerinde tire (-) kullanamıyoruz ama alias sayesinde API kullanıcılarına temiz bir URL (?agent-name=...) sunabildim. pattern ile yazdığım Regex ise sadece belirli bir isimlendirme standartına uyan ajanların aranmasına izin veriyor.

## 4. Test Aşamasındaki Keşfim: Fail-Fast (Hızlı Başarısız Ol) Prensibi
Sistemi zorlamak için pattern (Regex) kuralına uymayan ama max_length kuralını aşan çok uzun bir metin (100 'a' harfi,sınır 50) gönderdim. Beklentim iki kuralın da patlamasıydı ama sadece "Regex Hatası" aldım.

Kök Neden (Root Cause): FastAPI (ve altındaki Pydantic çekirdeği), CPU ve RAM optimizasyonu için "Short-Circuit" (Kısa Devre) mantığıyla çalışıyor. Sistem, girdinin Regex'e uymadığını (tamamen bozuk olduğunu) anladığı an işlemi anında kesiyor. Zaten çöpe gidecek bir verinin kaç karakter olduğunu saymak için sunucunun işlemci gücünü harcamıyor. Bu, uygulamayı DDoS tarzı yorma saldırılarına karşı doğal bir koruma kalkanına alıyor.

## 5. Alias ve Zorunlu (Required) Parametrelerin Çalışma Mantığı

API'yi tasarlarken sadece veri tipini doğrulamakla kalmayıp, bazı parametrelerin *kesinlikle* gönderilmesini zorunlu tutmamız gereken durumlar (örneğin bir arama yapılacaksa aranacak kelimenin girilmesi) oldu.

**Deney ve Gözlem:**
Parametrenin sonundaki `= None` (opsiyonel) ifadesini silip, yerine Python'ın **Ellipsis (`...`)** operatörünü koyarak parametreyi zorunlu hale getirdim:

```python
q: Annotated[
    str, 
    Query(alias="item-query", pattern="^fixedquery$")
] = ...  # None yerine ... (Zorunlu) koydum

Bunu yaptıktan sonra, sistemi kandırmak ve kısıtlamaları aşıp aşamayacağımı görmek için tarayıcıdan alakasız bir parametre ile kasıtlı olarak hatalı bir istek attım:
http://127.0.0.1:8000/items/?a=a

Sonuç ve Sistem Davranışı:

FastAPI URL'deki ?a=a parametresini tamamen görmezden geldi (çünkü güvenlik prensipleri gereği API şemasında tanımadığı hiçbir veriyi içeri almaz).

Sistem, zorunlu tuttuğum parametreyi bulamadığı için anında 422 Unprocessable Entity hatası fırlattı.

Ancak asıl kritik mimari detay hata mesajının kendisindeydi:
{"detail":[{"type":"missing","loc":["query","item-query"],"msg":"Field required","input":null}]}

Mimari Çıkarım (Abstraction):
Python kodumda değişkenin adını q olarak tanımlamış olmama rağmen, hata mesajında benden q'yu değil, doğrudan alias adını (item-query) istedi.

## 6. Alias Olup Zorunlu Parametetre Olmasaydı

Eğer o kodu tekrar = None (Opsiyonel) yapsaydın ve tarayıcıdan http://127.0.0.1:8000/items/?a=a adresine gitseydik, arka planda tam olarak şu senaryo yaşanacaktı:

Görmezden Gelme (Ignore): FastAPI URL'deki ?a=a kısmına bakacak, "Benim sözleşmemde (API şemamda) böyle bir parametre yok" deyip onu anında çöpe atacaktı.

Varsayılan Değer Ataması: URL'de beklediği item-query parametresini bulamadığı için, kodda belirttiğin varsayılan değere bakacak ve q = None atamasını yapacaktı.

Validasyonları Bypass Etme: En can alıcı nokta burası. q'nun değeri None (yani yokluk) olduğu için, FastAPI o yazdığın katı min_length=3 ve pattern="^fixedquery$" kurallarının hiçbirini çalıştırmayacaktı. Çünkü ortada kontrol edilecek bir metin yok!

Hatasız Çalışma: Kodun içindeki if q: (eğer q varsa) bloğu False döneceği için o satırı atlayacak ve ekrana doğrudan hatasız, şu JSON'ı basacaktı:
{"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}