# N11 Proje Notları 28.04.26

## User Service Initialize

Öncelikle spring initializr’dan gerekli ayarlamaları yapıyoruz.

Spring Web → Dış dünyaya açılan kapı, get,post gibi metodları içeriyor.

Spring Security → Kullanıcı şifrelerini veritabanına hash'leyerek (örneğin BCrypt ile) kaydetmek ve JWT (JSON Web Token) tabanlı giriş/yetkilendirme mekanizmasını kurmak için şarttır.

Spring Data JPA → Veritabanı ile nesne tabanlı (ORM) iletişim kurmanı sağlar. `UserRepository` gibi interfaceler oluşturarak sql sorguları yazmadan veritabanından kayıt çekmeyi ve kayıt atmayı sağlar.

PostgreSQL Driver → JPA'nın, Spring Boot ile PostgreSQL veritabanı arasında iletişim kurabilmesini sağlar.

Spring Boot Actuator →  Mikroservisin sağlığını izlemek için eklenir.

Config Client →   `user-service`'in veritabanı şifresi veya port numarası gibi ayarları kendi içine gömmeyip, ağdaki merkezi **Config Server**'dan çekerek ayağa kalkmasını sağlar.

Validation →  Kullanıcı kayıt olurken gönderdiği verileri (DTO'ları) kontrol etmek içindir. E-posta formatı doğru mu (`@Email`), şifre boş mu (`@NotBlank`) gibi gibi.

Spring for RabbitMQ → Bir kullanıcı başarıyla sisteme kayıt olduğunda `Notification Service`'e "Bu kullanıcıya Hoş Geldin e-postası gönder" şeklinde asenkron bir mesaj (event) fırlatmak için kullanılır.

Eureka Discovery Client → Uygulama ayağa kalktığı saniye gidip **Discovery Server**'a (Eureka) "Ben `user-service`'im ve şu IP/Port'ta çalışıyorum" diyerek kendini kaydettirmesi içindir.

Spring Data Redis (Access+Driver) → JWT Token işlemleri için kullanıyoruz. Refresh token tutup, süre uzatma vb. gibi.

Dosyamızı indirip IDE’mizden açıyoruz.

## User Service mikroservisimizi yazmaya başlıyoruz.

İlk olarak, entity package’ı açıyoruz. Buraya database’imize kaydedeceğimiz tableları koyacağız. Yani aslında user classımız gibi şeyler burada bulunacak. 

User classımızı açıp bunun bir entity olduğunu belirtmek için annotaion yazıyoruz. Ayrıc @Table annotationu kullanıp tablomuza bir isim veriyoruz.

```java
@Entity
@Table(name = "users")

```

En önemli şey primary key’imizi belirlemek, bu da tabiki id columnu olacak.

```java
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @NotBlank
    @Size(max = 20)
    private String username;

    @NotBlank
    @Size(max = 50)
    private String email;

    @NotBlank
    @Size(max = 120)
    private String password;

    @NotBlank
    @Size(max = 20)
    private Role role;

```

Sonrasında constructlarımızı ve tüm değerler için getter ve setterlarımızı yazıyoruz. JPA için kesinlikle **boş bir constructor** bulunması gerekiyor!!!

### Lombok ile getter, setter, constructor elle yazmaktan kurtulmak.

### 1. Neden Lombok Kullanmalıyız? (Clean Code)

- **Okunabilirlik:** İçinde 15 tane özellik (field) olan bir sınıfta manuel getter/setter yazıldığında sınıf 100 satıra çıkar. Lombok kullandığında sadece özellikler görünür. Bu, Clean Code prensiplerine uyar; kodu okuyan kişi "iş mantığına" odaklanır, gereksiz kod kalabalığına değil.
- **Bakım Kolaylığı:** Sınıfa yeni bir alan eklediğinde veya bir alanın adını değiştirdiğinde, gidip manuel olarak getter/setter'ı da güncellemek zorunda kalmayız.

### Ne Zaman Elle Yazmalıyız?

Eğer bir field'a değer atanırken veya okunurken **özel bir iş mantığı (business logic) çalışacaksa** o metodu elle yazmalıyız.

- *Örnek:* Bir e-ticaret sisteminde `setDiscount(double discount)` metodumuz var. İndirimin %100'den büyük olamayacağını kontrol etmek istiyorsak, bu `setDiscount` metodunu Lombok'a bırakmaz, kendimiz yazarız. Spring, geri kalan metotlar için Lombok'u kullanmaya devam eder.

### Önemli Uyarı:

JPA Entity'lerinde (`@Entity`) Lombok kullanırken yapılan en büyük hata sınıfın tepesine **`@Data`** anotasyonunu koymaktır.

`@Data` anotasyonu arka planda `toString()`, `equals()` ve `hashCode()` metotlarını da üretir. Veritabanı ilişkilerinde (Örn: `OneToMany`, `ManyToOne`) nesneler birbirini referans alır. `toString()` metodu çalıştığında sonsuz bir döngüye (StackOverflowError) girer uygulaman çöker veya performans (Lazy Loading) sorunları yaşarsın.

 Entity sınıflarında asla `@Data` kullanma. Onun yerine sınıfın başına sadece **`@Getter`** ve **`@Setter`** ekle. DTO (Data Transfer Object) sınıflarında ise `@Data` kullanmakta hiçbir sakınca yoktur.

```java
@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false)
    private String firstName;

    @Column(nullable = false)
    private String lastName;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;

    @CreationTimestamp
    @Column(updatable = false)
    private LocalDateTime createdAt;

}
```

@Getter → Getter Metodunu tanımlar
@Setter → Setter Metodunu tanımlar
@NoArgsConstructor → Argümansuz consturctor tanımlar
@AllArgsConstructor → Tüm argümanlarla consturctor tanımlar
@Builder → Okunabilirliği artırmak için 

```java
// Hangi parametre ne, belirsiz
User user = new User(UUID, "john@mail.com", "123456", "John", "Doe", Role.CUSTOMER, LocalDateTime.now());
```

```java
// Her field'ın adı görünüyor, okunabilir
User user = User.builder()
    .email("john@mail.com")
    .password("123456")
    .firstName("John")
    .lastName("Doe")
    .role(Role.CUSTOMER)
    .build();
```

### Bunları yaptıktan sonra burada işimiz bitiyor. Role için bir enum classı kullanıldı.

```java
public enum Role {
    CUSTOMER,
    ADMIN
}
```

## UserRepository:

Database işlemleri için SQL sorguları yazmak yerine, bu işi JPA Repository ile çözüyoruz. UserRepository interface’i açıp, bunu JpaRepository’den extend ediyoruz.

```java
public interface UserRepository extends JpaRepository<User, UUID> {

	Optional<User> findByEmail(String email);
	Boolean existsByEmail(String email);
}
```

Burası aslında sihirli bir yer, çünkü bizim bu metodları tanımlamamamıza, SpringDataJpa belirli bir kuralı takip edersek ne yapmak istediğimizi anlıyor ve buna karşılık gelen sorguyu database’e atıyor. Bu da bizim işimizi çok kolaylaştırıyor. Normalde burada bulunan metotlar:

`save()`, `findById()`, `findAll()`, `delete()` gibi metotlar.

Peki nasıl çalışıyor?

- `findByUsername` ismini parçalarına ayırır:
    - **`find...By`**: "Bir arama işlemi yapacağım" komutu.
    - **`Username`**: "User entity'si içindeki `username` alanına (field) bakacağım" komutu.
- **SQL Oluşturma:** Spring arka planda bizim için otomatik olarak bir SQL sorgusu hazırlar. Örneğin `findByUsername` için:
`SELECT * FROM users WHERE username = ?` sorgusu oluşturulur.

### Neden Optional<User> kullandık?

Eğer böyle kullanmazsak, User’ın her daim boş gelip gelmediğini kontrol etmemiz gerekiyor, ancakböyle kullanırsak, bize içi dolu da olabilen, boş da olabilen optional bir User nesnesi döner.

Bu sayede Java bize şunu söyler: *"Hey! Bu kutunun içi boş olabilir, içindeki User'ı doğrudan kullanamazsın. Önce kutunun içinin boş olup olmadığını kontrol etmeli veya boşsa ne yapacağını bana söylemelisin!"*

Biz de bu durumdan faydalanarak, eğer boşsa exception fırlatabiliriz.

```java
// Eğer doluysa User'ı ver, boşsa Exception fırlat!
User user = userRepository.findByUsername("alperen")
        .orElseThrow(() -> new RuntimeException("Kullanıcı bulunamadı!"));
```

## Servisleri Yazmak:

Üç adet servisimiz olacak, Auth Servis → login, register, logout, token üretme, doğrulama, refresh token yönetimi gibi işlemler burada, User Servis → kullanıcı bilgilerini güncelleme işlemleri burda, Redis Servis → kullanıcının tokenı (JWT) burada tutulacak.

Servisleri yazarken, bunları bir interfaceten başlatıp, Ondan implement ettiğimiz bir classa dönüştürmek best practicetir. Yani AuthService interface’imiz olacak, bundan bir AuthServiceImpl classı türeteceğiz. Dikkat edeceğimiz bir diğer nokta, interfacelere servis annotationu ekleyemeyiz. Çünkü bu annotationu koyunca Spring’e bu sınıftan bir bean oluştur diyoruz.Interfaceten bean oluşturamayız.

```java
// ❌ YANLIŞ
@Service
public interface AuthService { ... }

// ✅ DOĞRU
public interface AuthService { ... }

@Service
public class AuthServiceImpl implements AuthService { ... }
```

### Bean nedir?

Spring uygulaması ayağa kalktığında tüm `@Service`, `@Repository`, `@Controller` gibi annotationlu sınıfları tarar ve bunlardan birer nesne oluşturur. Bu nesnelere **bean** denir. Spring bu bean'leri **IoC Container**'da saklar.

Biz `AuthServiceImpl`'a ihtiyaç duyduğunda `new AuthServiceImpl()` yazmıyoruz, Spring zaten oluşturmuş, constructor injection ile bizeveriyor. Buna **Dependency Injection** denir.

### Neden servislerimizi interface’ten türetiyoruz?

Yarın `AuthServiceImpl` yerine farklı bir implementasyon yazmak gerekekirse, örneğin test için `MockAuthServiceImpl` ,sadece yeni bir sınıf yazarız, controller'a dokunmayız.. Bu **Open/Closed Principle**. Controller `AuthService` interface'ini inject eder, Spring arkada `AuthServiceImpl`'ı otomatik olarak bağlar. Bu yapı sayesinde implementasyon detayları üst katmandan gizlenir. **Dependency Inversion Principle**.

```java
// Controller sadece interface'i görür
private final AuthService authService;

// Arkada ne olduğunu bilmez
// AuthServiceImpl mi? MockAuthServiceImpl mi? fark etmez
```

### Auth Service:

```java
public interface AuthService {

    // Yeni kullanıcı kaydı
    void register(RegisterRequest request);

    // Kullanıcı girişi, token döner
    TokenResponse login(LoginRequest request);

    // Çıkış işlemi, token Redis'ten silinir
    void logout(String token);

    // Access token süresi dolunca yeniler
    TokenResponse refreshToken(String refreshToken);
}
```

**Dönüş tipleri neden böyle?**

`register` → `void` — kayıt sonrası token döndürmeye gerek yok, kullanıcı girişe yönlendirilebilir.

`login` → `TokenResponse` — access token + refresh token + expiresIn içeren bir DTO.

`logout` → `void` — sadece Redis'ten siler, dönecek bir şey yok.

`refreshToken` → `TokenResponse` — yeni access token üretir, aynı DTO'yu döner.

## DTO:

```java
public record LoginRequest(
        String email,
        String password
) {}
```

```java
public record RegisterRequest(
        String firstName,
        String lastName,
        String email,
        String password
) {}
```

```java
public record TokenResponse(
        String accessToken,
        String refreshToken,
        long expiresIn
) {}
```

### **DTO (Data Transfer Object) nedir?**

Katmanlar arasında veri taşıyan nesnedir. Tek amacı veri taşımaktır. İş mantığı içermez.

`Frontend  →  RegisterRequest (DTO)  →  Controller  →  Service
Service   →  TokenResponse (DTO)    →  Controller  →  Frontend`

**Neden entity'yi direkt döndürmüyoruz?**

```java
// ❌ YANLIŞ — entity direkt dönerse
public User login() { return user; } 
// password, tüm alanlar dışarı çıkar, güvenlik açığı
```

```java
// ✅ DOĞRU — DTO ile sadece gerekli alanlar gider
public TokenResponse login() { return new TokenResponse(...); }
// sadece token bilgisi gider
```

### Neden Record?

```java
// Normal class ile DTO
public class TokenResponse {
    private final String accessToken;
    private final String refreshToken;
    
    public TokenResponse(String accessToken, String refreshToken) {
        this.accessToken = accessToken;
        this.refreshToken = refreshToken;
    }
    
    public String getAccessToken() { return accessToken; }
    public String getRefreshToken() { return refreshToken; }
}

// record ile aynı şey, tek satır
public record TokenResponse(String accessToken, String refreshToken) {}
```

`record` otomatik olarak şunları üretir:

- Constructor
- Getter metodları
- `equals`, `hashCode`, `toString`

## Redis Service:

```java
public interface RedisService {

    // Token'ı Redis'e kaydet, TTL saniye cinsinden
    void saveToken(String key, String token, long ttlSeconds);

    // Token'ı getir, yoksa null döner
    String getToken(String key);

    // Token'ı sil (logout)
    void deleteToken(String key);

    // Token hâlâ geçerli mi?
    boolean isTokenValid(String key, String token);
}
```

```java
@Service
@RequiredArgsConstructor
public class RedisServiceImpl implements RedisService {

    private final StringRedisTemplate stringRedisTemplate;

    @Override
    public void saveToken(String key, String token, long ttlSeconds) {
        stringRedisTemplate.opsForValue()
                .set(key, token, ttlSeconds, TimeUnit.SECONDS);
    }

    @Override
    public String getToken(String key) {
        return stringRedisTemplate.opsForValue().get(key);
    }

    @Override
    public void deleteToken(String key) {
        stringRedisTemplate.delete(key);
    }

    @Override
    public boolean isTokenValid(String key, String token) {
        String savedToken = getToken(key);
        // Token Redis'te var mı ve eşleşiyor mu?
        return savedToken != null && savedToken.equals(token);
    }
}
```

**`StringRedisTemplate` nedir?**

Spring'in Redis ile konuşmak için verdiği hazır araç. `String key → String value` formatında çalışır, bizim için ideal çünkü `userId → token` saklayacağız.

**`opsForValue()`** — Redis'in en basit veri yapısı olan String operasyonlarına erişim sağlar.

**`TTL` neden önemli?**

Token Redis'e kaydedilirken süre veriyoruz. JWT access token ne zaman expire olacaksa Redis'teki kopyası da aynı anda silinir. Logout olmasa bile token otomatik temizlenir.

Burda ayrıca şifreleme yaparken hangi encoder metodu seçeceğimizi belirtmemiz gerekiyor.`JpaRepository`, `StringRedisTemplate` gibi şeyler Spring tarafından otomatik yönetiliyor. Ama `PasswordEncoder` için hangi algoritma kullanacağını Spring bilemez — BCrypt mi, SHA256 mi? Buna biz karar verip tanımlıyoruz. Bunun için

SecurityConfig classı oluşturuyoruz.

```java
@Configuration
public class SecurityConfig {

    // PasswordEncoder bean'ini tanımlıyoruz
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

@Configuration, bu sınıfın içindeki @Bean metodlarını Spring context'e kaydet demek.

@Bean bu metodun döndürdüğü nesneyi Bean olarak kaydet demek,

Artık `PasswordEncoder` inject edilebilir.

## JWT Util

Token üretimi ve kontrolleri yapmak için Jwt Util classımızı oluşturuyoruz.

API Gateway'den geçiş yapmak için bir JWT (JSON Web Token) gerekiyor. İşte bu `JwtUtil` sınıfı, o token'ları **üreten (imzalayan), içini okuyan ve sahte olup olmadığını kontrol eden** görevlimizdir.

```java
@Component
@RequiredArgsConstructor
public class JwtUtil {

    // Secret key, application.properties'den gelecek
    @Value("${jwt.secret}")
    private String secretKey;

    // Access token süresi — 1 saat (milisaniye)
    @Value("${jwt.access-token.expiration}")
    private long accessTokenExpiration;

    // Refresh token süresi — 7 gün (milisaniye)
    @Value("${jwt.refresh-token.expiration}")
    private long refreshTokenExpiration;

    // Secret key'i imzalamak için Key nesnesine çevir
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));
    }

    // Access token üret
    public String generateAccessToken(UUID userId, Role role) {
        return Jwts.builder()
                .subject(userId.toString())
                .claim("role", role.name())
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + accessTokenExpiration))
                .signWith(getSigningKey())
                .compact();
    }

    // Refresh token üret
    public String generateRefreshToken(UUID userId) {
        return Jwts.builder()
                .subject(userId.toString())
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + refreshTokenExpiration))
                .signWith(getSigningKey())
                .compact();
    }

    // Token'dan userId bul
    public UUID extractUserId(String token) {
        return UUID.fromString(getClaims(token).getSubject());
    }

    // Token'dan role çıkar
    public Role extractRole(String token) {
        return Role.valueOf(getClaims(token).get("role", String.class));
    }

    // Token geçerli mi?
    public boolean validateToken(String token) {
        try {
            getClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    // Token'dan claims'leri çıkar
    private Claims getClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

Burda dikkat etmemiz gereken,Value importumuz, springframework’e ait olan olmalıdır. Lombok seçersek ortalık karışır ve hatalar alırız.

### 1. Ayarlar ve Gizli Anahtar (Secret Key)

```java
@Value("${jwt.secret}")
private String secretKey;
// ... (süreler)

private SecretKey getSigningKey() {
    return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));
}
```

**Ne işe yarar?**`application.properties` veya `application.yml` dosyandan gizli bir şifre (`secretKey`) ve token sürelerini çekiyor.

**Kritik Detay — `getSigningKey`:**
JWT'ler dijital olarak imzalanır. Çektiği bu metin halindeki (Base64) gizli şifreyi, kriptografik bir `SecretKey` nesnesine (HMAC-SHA algoritması ile) dönüştürüyor. Bu anahtar olmadan kimse geçerli bir token üretemez.

### 2. Token Üretimi (Kimlik Kartı Basma)

```java
public String generateAccessToken(UUID userId, Role role) { ... }
public String generateRefreshToken(UUID userId) { ... }
```

Burada sistem **iki farklı kart** basıyor:

**Access Token (Geçiş Kartı)**
Kısa sürelidir (örn: 1 saat). İçine kullanıcının `userId`'sini (`subject` olarak) ve `role` bilgisini (`claim` olarak) gömer. API Gateway bu karta bakıp şunu söyler: *"Hımm, bu bir ADMIN, bu uç noktaya girebilir."*

**Refresh Token (Yenileme Kartı)**
Uzun sürelidir (örn: 7 gün). İçine rol bilgisi konulmamıştır. Çünkü bu kartın tek amacı, Access Token'ın süresi dolduğunda Auth servisine gidip *"Bana yeni bir Access Token ver"* demektir.

### 3. Token'ın İçini Okuma (Claims)

```java
public UUID extractUserId(String token) { ... }
public Role extractRole(String token) { ... }
private Claims getClaims(String token) { ... }
```

**Ne işe yarar?**
Bir kullanıcı bize token gönderdiğinde, bu token'ın içindeki verilere **Claims (Beyanlar)** denir.

- `getClaims` metodu, senin gizli anahtarıımızla (`verifyWith(getSigningKey())`) kasanın kilidini açar ve içindeki bilgileri (Payload) okur.
- `extract` metotları ile bu kasanın içinden kimliği (`userId`) ve yetkiyi (`role`) çekip sistemde kullanırsın.

### 4. Güvenlik Kontrolü (Sahte mi? Süresi Dolmuş mu?)

```java
public boolean validateToken(String token) {
    try {
        getClaims(token);
        return true;
    } catch (JwtException | IllegalArgumentException e) {
        return false;
    }
}
```

**Ne işe yarar?**
Token'ın geçerli olup olmadığını doğrular.

| Senaryo | Sonuç |
| --- | --- |
| Token geçerli ve süresi dolmamış | `true` döner |
| Hacker token içeriğini değiştirmiş (örn: rolü ADMIN yaptı) | İmza bozulur → `JwtException` → `false` |
| Token'ın süresi (expiration) dolmuş | `ExpiredJwtException` → `false` |

**Özetle:** Eğer bir hacker token'ın içindeki rolü `"CUSTOMER"` yerine `"ADMIN"` olarak değiştirmeye çalışırsa, imza bozulacağı için `getClaims` metodu anında exception fırlatır. Biz de bu hataları `catch` bloğunda yakalayıp `false` (Geçersiz Token) döndürürsün.

## Artık Auth servisimizi yazmaya başlayabiliriz.

### `AuthServiceImpl` 4 metodu implement edecek

1. register

1. Email daha önce alınmış mı? → existsByEmail kontrolü
2. Şifreyi encode et → passwordEncoder.encode()
3. User nesnesi oluştur, DB'ye kaydet → userRepository.save()

### 2. login

1. Email var mı? → findByEmail, yoksa exception fırlat
2. Şifre doğru mu? → passwordEncoder.matches()
3. Access token üret → jwtUtil.generateAccessToken()
4. Refresh token üret → jwtUtil.generateRefreshToken()
5. Access token'ı Redis'e kaydet → redisService.saveToken()
6. TokenResponse döndür

### 3. logout

1. Token'dan userId'yi çıkar → jwtUtil.extractUserId()
2. Redis'ten token'ı sil → redisService.deleteToken()

### 4. refreshToken

1. Refresh token geçerli mi? → jwtUtil.validateToken()
2. Token'dan userId'yi çıkar → jwtUtil.extractUserId()
3. Yeni access token üret → jwtUtil.generateAccessToken()
4. Redis'i güncelle → redisService.saveToken()
5. Yeni TokenResponse döndür

```java
@Service
@RequiredArgsConstructor  //Lombok sayesinde constructor injectiona gerek kalmadı.
public class AuthServiceImpl implements AuthService {

    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;
    private final JwtUtil jwtUtil;
    private final RedisService redisService;

    @Override
    public void register(RegisterRequest request) {
        // Email başka biri tarafından alınmış mı?
        if (userRepository.existsByEmail(request.email())) {
            throw new RuntimeException("Email already exists");
        }

        // Kullanıcıyı oluştur ve kaydet
        User user = User.builder()
                .email(request.email())
                .password(passwordEncoder.encode(request.password()))
                .firstName(request.firstName())
                .lastName(request.lastName())
                .role(Role.CUSTOMER)
                .build();

        userRepository.save(user);
    }

    @Override
    public TokenResponse login(LoginRequest request) {
        // Kullanıcı var mı?
        User user = userRepository.findByEmail(request.email())
                .orElseThrow(() -> new RuntimeException("User not found"));

        // Şifre doğru mu?
        if (!passwordEncoder.matches(request.password(), user.getPassword())) {
            throw new RuntimeException("Invalid password");
        }

        // Token üret
        String accessToken = jwtUtil.generateAccessToken(user.getId(), user.getRole());
        String refreshToken = jwtUtil.generateRefreshToken(user.getId());

        // Access token'ı Redis'e kaydet (1 saat)
        redisService.saveToken("auth:token:" + user.getId(), accessToken, 3600);

        return new TokenResponse(accessToken, refreshToken, 3600);
    }

    @Override
    public void logout(String token) {
        // Token'dan userId çıkar
        UUID userId = jwtUtil.extractUserId(token);

        // Redis'ten token'ı sil
        redisService.deleteToken("auth:token:" + userId);
    }

    @Override
    public TokenResponse refreshToken(String refreshToken) {
        // Refresh token geçerli mi?
        if (!jwtUtil.validateToken(refreshToken)) {
            throw new RuntimeException("Invalid refresh token");
        }

        // Token'dan userId çıkar
        UUID userId = jwtUtil.extractUserId(refreshToken);

        // Kullanıcıyı DB'den çek
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        // Yeni access token üret
        String newAccessToken = jwtUtil.generateAccessToken(user.getId(), user.getRole());

        // Redis'i güncelle
        redisService.saveToken("auth:token:" + userId, newAccessToken, 3600);

        return new TokenResponse(newAccessToken, refreshToken, 3600);
    }
}
```

## Auth Controller:

İşte burada bize gelen HTTP isteklerini (post,get gibi alıyoruz, ve yerine getirmesi için gerekli servisimizi çağırıyoruz. Sonrasında cevabı döndürüyoruz.

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    // Kayıt ol
    @PostMapping("/register")
    public ResponseEntity<Void> register(@Valid @RequestBody RegisterRequest request) {
        authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    // Giriş yap
    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@Valid @RequestBody LoginRequest request) {
        TokenResponse response = authService.login(request);
        return ResponseEntity.ok(response);
    }

    // Çıkış yap
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@RequestHeader("Authorization") String authHeader) {
        // "Bearer token123" → "token123"
        String token = authHeader.substring(7);
        authService.logout(token);
        return ResponseEntity.ok().build();
    }

    // Token yenile
    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refreshToken(@RequestBody String refreshToken) {
        TokenResponse response = authService.refreshToken(refreshToken);
        return ResponseEntity.ok(response);
    }
}
```

@RequestBody → HTTP isteğinin body'sindeki JSON'ı Java nesnesine çevirir:

```java
// Frontend'den gelen JSON
{
  "email": "john@mail.com",
  "password": "123456"
}
// @RequestBody bunu LoginRequest nesnesine çevirir
public ResponseEntity<TokenResponse> login(@RequestBody LoginRequest request)
```

@Valid → RegisterRequest ve LoginRequest içindeki validation kurallarını tetikler:

```java
public record RegisterRequest(
    @NotBlank String email,       // boş olamaz
    @Size(min=6) String password  // en az 6 karakter
) {}
```

@RequestHeader → HTTP isteğinin header'ından değer çeker. Logout'ta token body'de değil header'da gelir:

```java
Authorization: Bearer eyJhbGc...
// Header'dan Authorization değerini çeker
public ResponseEntity<Void> logout(@RequestHeader("Authorization") String authHeader)
```

Frontend her korumalı isteğe bu header'ı ekler, biz de oradan token'ı alırız.

### Security Config:

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                // CSRF'yi kapat — JWT kullandığımız için gerek yok
                .csrf(csrf -> csrf.disable())

                // Session tutma — JWT stateless
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

                // Endpoint yetkilendirme
                .authorizeHttpRequests(auth -> auth
                        // Bu endpoint'ler herkese açık
												 .requestMatchers(
												    "/api/auth/register",
												    "/api/auth/login",
												    "/api/auth/refresh",    
												    "/swagger-ui/**",
												    "/v3/api-docs/**"
												).permitAll()
                        // Geri kalanlar token ister
                        .anyRequest().authenticated())

                // JWT filter'ı Spring Security'nin önüne ekle
                .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    // PasswordEncoder bean'ini tanımlıyoruz,
    // PasswordEncoder için hangi algoritma kullanacağını Spring bilmiyor.BCrypt mi, SHA256 mi? Buna karar verip tanımlıyoruz. Standart olduğu için BCrypt seçtim.
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### JwAuthFilter:

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final RedisService redisService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // Header'dan token'ı al
        String authHeader = request.getHeader("Authorization");

        // Token yoksa veya Bearer ile başlamıyorsa geç
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // "Bearer " kısmını at
        String token = authHeader.substring(7);

        // Token geçerli mi?
        if (!jwtUtil.validateToken(token)) {
            filterChain.doFilter(request, response);
            return;
        }

        // Token'dan userId çıkar
        UUID userId = jwtUtil.extractUserId(token);

        // Redis'te bu token var mı?
        if (!redisService.isTokenValid("auth:token:" + userId, token)) {
            filterChain.doFilter(request, response);
            return;
        }

        // Role'ü çıkar
        Role role = jwtUtil.extractRole(token);

        // Spring Security'ye kullanıcıyı tanıt
        UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                        userId,
                        null,
                        List.of(new SimpleGrantedAuthority("ROLE_" + role.name()))
                );

        SecurityContextHolder.getContext().setAuthentication(authentication);
        filterChain.doFilter(request, response);
    }
}
```

### 1. `SecurityConfig` Sınıfı (Güvenlik Kuralları)

Bu sınıf, sistemin genel güvenlik politikalarını belirler. "Kim girebilir, kapıda nasıl bir prosedür uygulanacak?" sorularının cevabı buradadır.

- **`csrf.disable()` (CSRF'yi Kapatmak):** CSRF, tarayıcılardaki "Session (Oturum)" çerezlerini çalan bir saldırı türüdür. Biz JWT kullanıyoruz ve durumu tarayıcıda değil, HTTP Header'da (`Bearer ...`) taşıyoruz. Bu yüzden bu kalkanı kapatıyoruz.
- **`sessionCreationPolicy(STATELESS)` (Durumsuz Oturum):** Spring'e şunu söylüyoruz: *"Gelen hiçbir kullanıcıyı hafızanda tutma (Session açma). Kişi her geldiğinde kimliğini (Token'ını) baştan göstermek zorunda."* Bu, mikroservislerin (özellikle yüksek trafikli sistemlerin) en önemli kuralıdır; RAM'i şişirmez.
- **`requestMatchers(...).permitAll()` (Serbest Geçiş):** Kullanıcının token alabilmesi için önce kayıt olması veya giriş yapması gerekir. Bu yüzden `/register` ve `/login` kapılarını herkese sonuna kadar açıyorsun.
- **`.anyRequest().authenticated()` (Kimlik Sorma):** Kalan tüm API uç noktaları için kapıyı kilitliyorsun. Token'ı olmayan giremez.
- **`addFilterBefore(...)` (Filtre Sıralaması):** Spring Security'nin kendi varsayılan bir kimlik doğrulama filtresi (`UsernamePasswordAuthenticationFilter`) vardır. Sen diyorsun ki: *"Spring, sen devreye girmeden önce benim kendi özel güvenlik görevlim (`JwtAuthFilter`) kapıya baksın."*
- **`passwordEncoder()`:** Şifreleri veritabanına "123456" diye düz metin değil, BCrypt algoritmasıyla karmaşık bir hash olarak (`$2a$10$...`) kaydetmek için gereken motoru sisteme tanıtıyorsun.

### 2. `JwtAuthFilter` Sınıfı (Güvenlik Görevlisi)

Bu sınıf, `OncePerRequestFilter`'dan miras aldığı için sisteme gelen **istisnasız her istekte (request) bir kez çalışır**. Görevi, gelen kişinin cebindeki bileti (JWT) kontrol etmektir.

**Adım Adım Çalışma Mantığı:**

1. **Bilet Kontrolü (`authHeader`):** İstek geldiğinde güvenlik görevlisi Header'a bakar. *"Authorization başlığı var mı ve Bearer kelimesiyle başlıyor mu?"* Eğer yoksa, doğrudan pas geçer (`filterChain.doFilter`).
2. **Biletin Sahteliği (`validateToken`):** "Bearer" kelimesini kesip atar ve saf token'ı alır. `JwtUtil`'e sorar: *"Bu bilet sahte mi? Süresi geçmiş mi?"*
3. **Redis Kontrolü (Ekstra Güvenlik):** Bilet gerçek bile olsa, `RedisService`'e sorar: *"Bu kullanıcı çıkış yapmış mı? (Veya bu token hala geçerli mi?)"* Bu, daha önce konuştuğumuz mükemmel bir güvenlik katmanıdır.
4. **Kimlik ve Rol Çıkarımı (`extractUserId`, `extractRole`):** Bilet her testten başarıyla geçerse, içinden kullanıcının ID'si ve Rolü (Örn: CUSTOMER) okunur.
5. **Spring'e Haber Verme (`SecurityContextHolder`):** Burası en kritik noktadır. Görevli (Filtre), içerideki sisteme (Spring Security) telsizle anons geçer: *"İçeri giren kişiyi ben doğruladım. ID'si bu, Yetkisi de bu (`ROLE_CUSTOMER`)."*
    - *Not:* Spring Security yetkileri tanırken başına her zaman **`ROLE_`** ön ekinin gelmesini bekler. Oraya eklediğin `List.of(new SimpleGrantedAuthority("ROLE_" + role.name()))` kodu tam olarak bu standardı sağlar