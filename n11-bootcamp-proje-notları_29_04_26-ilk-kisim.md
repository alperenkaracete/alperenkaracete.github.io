# N11 Proje Notları 29.04.26 İlk Kısım

## Bugün kalan servislerimizin de tamamını implement etmeye çalışacağız. İlk servisimiz product-service. Burda ürünler hakkında stok haricindeki bilgileri tutacağız. Çünkü stok bilgisini tutmak stok servisinin işi.

# Product Service:

- `Spring Web`
- `Spring Data JPA`
- `PostgreSQL Driver`
- `Lombok`
- `Eureka Discovery Client`
- `Config Client`
- `Validation`
- `Spring Boot Actuator`
- `Spring for RabbitMQ`
- `Testcontainers`
- `Spring Doc` Swagger’da istiyorsak onu da eklemeliyiz

Spring Initializr’a bunları ekleyeceğiz.

Burda yeni olarak gördüğümüz sadece Testcontainers var.

## Testcontainers:

Integration test için. Gerçek bir PostgreSQL container'ı ayağa kaldırıp testleri o DB üzerinde çalıştırıyoruz.

```java
@SpringBootTest
@Testcontainers
class ProductServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:15");

    // Test bitti mi? Container otomatik siliniyor.
}
```

Unit test      → DB yok, her şey mock
Integration test → Gerçek PostgreSQL, ama geçici container içinde

Burda mockdan kastımız, gerçek bir veritabanı yok, her şey bizim yazdığımız if-else’ler üzerinden yürüyor. Bunu dublör gibi düşünebiliriz. Veritabanını bizim yazdığımız senaryo doğrultusunda taklit eder.

Integration Test ise lokaldeki database’e dokunmadan, bir test db containerı ayağı kaldırır ve tüm testleri bunun üzerinden gerçekleştirir.

Hemen product classını yazarak devam edelim.

## Product:

Ürün bilgileri ve gerekli işlemler.

```java
@Entity
@Table(name = "products")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, length = 1000)
    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private String imageUrl;

    @Column(nullable = false)
    private String category;
}
```

Burda en başta da belirttiğimiz gibi stok bilgisi bulunmayacak. Çünkü bu stok servisinin işi.

Price’ın BigDecimal olmasının sebebi ise, virgülden sonra hataların gerçekleşmesini istemememiz.

```java
// double ile
0.1 + 0.2 = 0.30000000000000004  // Yanlış

// BigDecimal ile
0.1 + 0.2 = 0.3  // Doğru
```

## Bu yüzden para birimi için her zaman  BigDecimal.

## ProductRepository:

Bilgidğimiz üzere, veritabanı ile java üzerinden konuşabilmek için bu classımızı açıyoruz.

```java
public interface ProductRepository extends JpaRepository<Product, UUID> {

    // Kategori bazlı filtreleme
    Page<Product> findByCategory(String category, Pageable pageable);

    // İsme göre arama
    Page<Product> findByNameContainingIgnoreCase(String name, Pageable pageable);
}
```

### **Page nedir, neden kullanıyoruz?**

Ürün listesinde 1000 ürün varsa hepsini bir anda döndürmek hem yavaş hem de gereksiz. `Page` ile şunu yapıyoruz:

"Bana 1. sayfayı ver, sayfada 10 ürün olsun"
→ sadece 10 ürün gelir

```java
page.getContent()      // o sayfadaki ürünler
page.getTotalElements() // toplam kaç ürün var (1000)
page.getTotalPages()    // toplam kaç sayfa var (100)
page.getNumber()        // şu an kaçıncı sayfadasın (0)
```

Frontend bunu alıp "Sayfa 1/100" şeklinde gösterir.

`Optional` ile farkı:

Optional → tek bir nesne var mı yok mu? (getProductById)
Page     → birden fazla nesneyi sayfalı getir (getAllProducts)

```java
// Tek ürün → Optional
Optional<Product> findById(UUID id);

// Çok ürün → Page
Page<Product> findAll(Pageable pageable);
```

Artık DTO’larımıza geçebiliriz.

## DTO’lar

Kullanıcı (Admin) yeni ürün oluşturabilir ve mevcut bir ürünü güncelleyebilir ve bir ürün silebilir.

### CreateProductRequest:

```java
public record CreateProductRequest(

    @NotBlank(message = "Name cannot be blank")
    String name,

    @NotBlank(message = "Description cannot be blank")
    String description,

    @NotNull(message = "Price cannot be null")
    @DecimalMin(value = "0.0", inclusive = false, message = "Price must be greater than 0")
    BigDecimal price,

    @NotBlank(message = "Image URL cannot be blank")
    String imageUrl,

    @NotBlank(message = "Category cannot be blank")
    String category
) {}
```

### UpdateProductRequest:

```java
public record UpdateProductRequest(

    String name,
    String description,
    BigDecimal price,
    String imageUrl,
    String category
) {}
```

Burda validation yok, çünkü kullanıcı sadece bazı alanları güncellemek isteyebilir, null gelen alanlar güncellenmeyecek.

`@NotNull` vs `@NotBlank`

@NotNull   → null olamaz, ama boş string olabilir → ""  geçer
@NotBlank  → null da olamaz, boş string de olamaz, sadece boşluk da olamaz → "   " geçmez

String alanlar için her zaman @NotBlank kullanmalıyız.  @NotNull yetmez, "" geçirebilirler.

BigDecimal için @NotBlank kullanamayız çünkü String değil, o yüzden @NotNull kullandık.

### ProductResponse:

```java
public record ProductResponse(

    UUID id,
    String name,
    String description,
    BigDecimal price,
    String imageUrl,
    String category
) {}
```

Bunu yapmamızın amacı, direkt entity nesnesini döndürmemek. Böylece:

**1. Güvenlik:** Entity'de hassas alan olabilir. Direkt döndürürsek istemeden dışarı çıkar, ayrıca record immutable’dır yani sadece oluşturulurken set edilebilirler. Sonrasında değiştirilemezler.

**2. Esneklik: Response her zaman entity ile birebir aynı olmak zorunda değil.**

```java
// Entity'de stok yok (Stock Service'te)
// Ama response'da stok bilgisi göstermek isteyebiliriz
public record ProductResponse(
    UUID id,
    String name,
    BigDecimal price,
    int stock        // ← entity'de yok, Stock Service'den geliyor
) {}
```

## Recordu hatırlamak için tekrar bir gözden geçirelim:

`record` yapısının asıl varoluş amacı **Immutable (Değiştirilemez)** veri taşıyıcıları olmaktır. Bir `record` nesnesi yaratıldıktan sonra içindeki veriler bir daha asla değiştirilemez. Bu yüzden Java, setter metotlarını üretmez ve bizim de manuel olarak yazmamıza izin vermez.

### 1. Ne YOKTUR?

- **Setter Metotları:** `setName()`, `setPrice()` gibi metotlar yoktur. Çünkü alanların hepsi arka planda `private final` olarak tanımlanır. Değerleri sadece ilk yaratılış anında verebilirsin.

### 2. Ne VARDIR?

- **Constructor (Yapıcı Metot):** İçindeki tüm parametreleri alan bir constructor (canonical constructor) otomatik olarak oluşturulur.
- **Getter (Okuyucu) Metotlar (Ama bir farkla):** Okuyucu metotları vardır ancak alışık olduğumuz `get...` ön ekini kullanmazlar. Doğrudan alanın (field) adıyla çağrılırlar.
    - *Class yöntemi:* `product.getName()`
    - *Record yöntemi:* `product.name()`
- **Yardımcı Metotlar:** `toString()`, `equals()` ve `hashCode()` metotları arka planda otomatik olarak kusursuz bir şekilde yazılır.

```java
// 1. Constructor ile oluşturma (Zorunludur)
ProductResponse response = new ProductResponse(uuid, "Laptop", 1500.0);

// 2. Değerleri okuma (Getter yerine doğrudan ismini kullanırsın)
String productName = response.name(); 

// 3. Değerleri değiştirme denemesi (HATA VERİR!)
response.setName("Telefon"); // Böyle bir metot yoktur!
response.name = "Telefon";   // Alanlar final olduğu için erişilemez/değiştirilemez!
```

## Product Service Impl

```java
@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;

    @Override
    public Page<ProductResponse> getAllProducts(Pageable pageable) {
        return productRepository.findAll(pageable)
                .map(this::toResponse);
    }

    @Override
    public ProductResponse getProductById(UUID id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found"));
        return toResponse(product);
    }

    @Override
    public Page<ProductResponse> getProductsByCategory(String category, Pageable pageable) {
        return productRepository.findByCategory(category, pageable)
                .map(this::toResponse);
    }

    @Override
    public Page<ProductResponse> searchProducts(String name, Pageable pageable) {
        return productRepository.findByNameContainingIgnoreCase(name, pageable)
                .map(this::toResponse);
    }

    @Override
    public ProductResponse createProduct(CreateProductRequest request) {
        Product product = Product.builder()
                .name(request.name())
                .description(request.description())
                .price(request.price())
                .imageUrl(request.imageUrl())
                .category(request.category())
                .build();

        return toResponse(productRepository.save(product));
    }

    @Override
    public ProductResponse updateProduct(UUID id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found"));

        // Sadece null olmayan alanları güncelle
        if (request.name() != null) product.setName(request.name());
        if (request.description() != null) product.setDescription(request.description());
        if (request.price() != null) product.setPrice(request.price());
        if (request.imageUrl() != null) product.setImageUrl(request.imageUrl());
        if (request.category() != null) product.setCategory(request.category());

        return toResponse(productRepository.save(product));
    }

    @Override
    public void deleteProduct(UUID id) {
        // Ürün var mı kontrol et
        if (!productRepository.existsById(id)) {
            throw new RuntimeException("Product not found");
        }
        productRepository.deleteById(id);
    }

    // Entity → Response dönüşümü
    private ProductResponse toResponse(Product product) {
        return new ProductResponse(
                product.getId(),
                product.getName(),
                product.getDescription(),
                product.getPrice(),
                product.getImageUrl(),
                product.getCategory()
        );
    }
}
```

Az önce de konuştuğumuz için güvenlik ve esneklik sebebi ile, direkt entity’i döndürmek yerine, ProductResponse döndürüyoruz. Bu zaten entitymizin tüm içerikilerini barındırıyor.

**Dikkat edilecek noktalar:**

`map(this::toResponse)` — `Page` içindeki her `Product`'ı `ProductResponse`'a çevirir. `stream().map()` gibi düşünüyoruz.

`updateProduct`'ta null kontrolü — sadece gönderilen alanlar güncellenir, gönderilmeyenler olduğu gibi kalır.

`deleteProduct`'ta önce `existsById` kontrolü — ürün yoksa anlamlı bir hata fırlatıyoruz.

### return productRepository.findAll(pageable)
.map(this::toResponse); Ne yapar?

### 1. `.map()` Ne İşe Yarar? (Dönüştürücü Fabrika)

`map()` fonksiyonu bir koleksiyonun (bir Liste, Stream veya Spring'deki `Page` nesnesi) içindeki elemanları tek tek dönmek ve onları **başka bir formata dönüştürmek** için kullanılır.

Bir fabrika bandı gibi düşün:

- Banttan çiğ et (Veritabanından gelen `Product` entity'si) giriyor.
- `map()` makinesi bu eti işliyor.
- Bandın diğer ucundan paketlenmiş sosis (Dışarıya verilecek `ProductResponse` DTO'su) çıkıyor.

### 2. `this::toResponse` Ne Anlama Geliyor? (Method Reference)

İşte işin büyüsü burada. Bu yazım şekline **Method Reference (Metot Referansı)** denir ve aslında uzun uzun yazacağın bir Lambda ifadesinin kısaltılmış halidir.

Bu kodun tam olarak neyin kısaltması olduğunu görünce taşlar anında yerine oturacak:

```java
return productRepository.findAll(pageable)
        .map(product -> this.toResponse(product));
```

### Adım Adım Arka Planda Neler Oluyor?

1. `productRepository.findAll(pageable)` veritabanına gider ve örneğin 20 tane `Product` entity'si ile dolu bir `Page` (Sayfa) nesnesi getirir.
2. `.map(...)` devreye girer. Bu 20 ürünü tek tek eline alır.
3. Elindeki 1. ürünü alır, senin daha önce yazdığın `toResponse(Product product)` metodunun içine fırlatır.
4. O metot sana bir `ProductResponse` record'u döner. `map` bunu alır, yeni oluşturduğu boş kutuya (yeni Page nesnesine) koyar.
5. İşlemi 20 ürün için de tekrarlar.
6. İşlem bittiğinde, elinde artık içi `Product` dolu değil, içi `ProductResponse` dolu tertemiz bir `Page` nesnesi vardır ve bunu `return` ile dışarı atar.

## Security Config

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/swagger-ui/**",
                    "/v3/api-docs/**"
                ).permitAll()
                .anyRequest().authenticated());

        return http.build();
    }
}
```

Önceki günlerde burdakilerin açıklamasını yaptık, buraya sadece ekstradan  `@EnableMethodSecurity` eklendi. Buranın Userdakinden farkı, userdaki gibi jwt filter kullanmıyoruz.

### `@EnableMethodSecurity:`

Güvenliği sadece URL (endpoint) bazında değil, **doğrudan Java metotları bazında** yönetmemizi sağlar.

### 1. Güçlü Anotasyonların Kilidini Açar

Bu anotasyonu eklediğimiz an Spring'in `@PreAuthorize`, `@PostAuthorize`, `@Secured` gibi metot bazlı güvenlik anotasyonları aktif hale getiririz.

### 2. SpEL (Spring Expression Language) Kullanımı

Artık metotlarının tepesine gidip, bu metodu kimlerin çalıştırabileceğini doğrudan söyleyebiliriz.

```java
@PostMapping
@PreAuthorize("hasRole('ADMIN')") // Sadece Admin girebilir!
public ProductResponse createProduct(...) {
    return productService.createProduct(request);
}
```

Gibi.

`EnableMethodSecurity` sayesinde sadece Controller'ları değil, içerideki **Service katmanındaki metotları da koruyabiliriz.** Örneğin `ProductServiceImpl` içindeki `deleteProduct` metodunun başına `@PreAuthorize("hasRole('ADMIN')")` koyarsak, sistemin neresinden çağrılırsa çağrılsın (başka bir servisten, bir zamanlanmış görevden vs.) o anki kullanıcının Admin yetkisi yoksa metot çalışmaz.

### 1. Neden `@Bean` Olarak Tanımlıyoruz?

Spring Boot'a şunu diyoruz: *"Al bu benim güvenlik kurallarım. Bunu kendi merkezine (IoC Container) kaydet. Dışarıdan uygulamama bir HTTP isteği geldiğinde, onu benim Controller'larıma ulaştırmadan önce **zorunlu olarak** bu kurallardan geçir."*

### 2. "Chain" (Zincir) Mantığı Nedir?

Spring Security arka planda tek bir devasa güvenlik kontrolü yapmaz. İşi küçük, spesifik görevleri olan parçalara böler ve bir "Filtreler Zinciri" oluşturur.
Dışarıdan bir istek geldiğinde bu istek uzun bir koridora girer ve sırayla kapılardan (filtrelerden) geçer:

- **Kapı 1 (CORS/CSRF Filtresi):** "Bu istek güvenilir bir kaynaktan/tarayıcıdan mı geldi?" (Biz `.csrf.disable()` diyerek bu kapının kilidini açık bıraktın).
- **Kapı 2 (Özel Filtre):** `JwtAuthFilter` burada devreye girer. "Cebinde geçerli bir bilet (Token) var mı?"
- **Kapı 3 (Authorization Filtresi):** "Bileti var ama, bu uç noktaya girmeye yetkisi var mı?" (`.anyRequest().authenticated()` veya `@PreAuthorize` kısmı).

Eğer istek bu filtrelerin herhangi birinden geçemezse (örneğin token geçersizse veya yetki yoksa), zincir anında kopar ve istek `ProductController`'ına **asla** ulaşamaz. Geriye doğrudan `401 Unauthorized` veya `403 Forbidden` hatası döner. İşin en güzel yanı da budur; iş mantığı (Service) katmanın bu kötü niyetli trafikle hiç muhatap olmaz.

### 3. `http.build()` Ne Yapar?

Bizim `http` üzerinden yazdığın `.csrf()`, `.sessionManagement()`, `.authorizeHttpRequests()` gibi metotların hepsi birer Builder (İnşa edici) ayarıdır.
En sondaki `http.build()` komutu ise inşaatın bitişini temsil eder. Tüm bu soyut ayarları alır, derler ve trafiği bizzat yönetecek o **nihai, somut nesneyi** oluşturur.

Özetle; bu metot bizim tüm web trafiğimizi göpüsleyen, filtreleyen ve sadece kurallara uyan istekleri içeri alan muazzam bir kalkan sağlıyor.

## Product Controller:

```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    // Tüm ürünleri sayfalı getir
    @GetMapping
    public ResponseEntity<Page<ProductResponse>> getAllProducts(Pageable pageable) {
        return ResponseEntity.ok(productService.getAllProducts(pageable));
    }

    // Tek ürün getir
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProductById(@PathVariable UUID id) {
        return ResponseEntity.ok(productService.getProductById(id));
    }

    // Kategori bazlı getir
    @GetMapping("/category/{category}")
    public ResponseEntity<Page<ProductResponse>> getProductsByCategory(
            @PathVariable String category,
            Pageable pageable) {
        return ResponseEntity.ok(productService.getProductsByCategory(category, pageable));
    }

    // İsme göre ara
    @GetMapping("/search")
    public ResponseEntity<Page<ProductResponse>> searchProducts(
            @RequestParam String name,
            Pageable pageable) {
        return ResponseEntity.ok(productService.searchProducts(name, pageable));
    }

    // Ürün ekle (Admin)
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductResponse> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(productService.createProduct(request));
    }

    // Ürün güncelle (Admin)
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable UUID id,
            @RequestBody UpdateProductRequest request) {
        return ResponseEntity.ok(productService.updateProduct(id, request));
    }

    // Ürün sil (Admin)
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteProduct(@PathVariable UUID id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}
```

Burda her bir işlemde best practice için uygun kodu döndürüyoruz.

| Kod | Anlamı | Ne zaman |
| --- | --- | --- |
| 200 OK | Başarılı | GET, güncelleme |
| 201 Created | Yeni kaynak oluşturuldu | POST (ürün ekle, kayıt ol) |
| 204 No Content | Başarılı ama dönecek veri yok | DELETE |
| 400 Bad Request | İstek hatalı | Validation hatası |
| 401 Unauthorized | Token yok | Kimlik doğrulama gerekli |
| 403 Forbidden | Token var ama yetki yok | Admin endpoint'ine customer girmeye çalışıyor |
| 404 Not Found | Kaynak bulunamadı | Olmayan ürün ID'si |
| 500 Internal Server Error | Sunucu hatası | Beklenmedik exception |

```java
ResponseEntity.ok(...)                              // 200
ResponseEntity.status(HttpStatus.CREATED).body(...) // 201
ResponseEntity.noContent().build()                  // 204
ResponseEntity.badRequest().build()                 // 400
ResponseEntity.notFound().build()                   // 404
// 401, 403, 500 → Spring otomatik döner
```

**Yeni annotationlar:**

`@PathVariable` — URL'deki değişkeni alır:

```java
GET /api/products/550e8400-...
@PathVariable UUID id → 550e8400-...
```

`@RequestParam` — URL'deki query parametresini alır:

```java
GET /api/products/search?name=laptop
@RequestParam String name → "laptop"
```

`@PreAuthorize("hasRole('ADMIN')")`  Az önce de bahsettiğimiz gibi, sadece ADMIN rolündeki kullanıcılar çağırabilir.

Bunun aktif olması için kesinlikle:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // ← bunu eklemeliyiz.
@RequiredArgsConstructor
public class SecurityConfig { ... }
```

Şimdilik her şey tamam! Şimdi server-configimize product-servie’imizin config dosyasını yerleştirmeliyiz.

config-server/src/main/resources/configs/product-service.properties dosyasını oluşturup:

```java
spring.datasource.url=jdbc:postgresql://localhost:5432/product_db
spring.datasource.username=postgres
spring.datasource.password=123456
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
```

Kendi propertysine ise:

```java
spring.application.name=product-service
server.port=8082
spring.config.import=configserver:http://localhost:8888
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

Yine Eureka’nın,configin adresini ve kendi portnu giriyoruz.