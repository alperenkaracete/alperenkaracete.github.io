# N11 Proje Notları

Projemize başlarken ilk olarak backend ve frontend olmak üzere projemiz ile ilgili 2 repo oluşturuyoruz.

Sonrasında spring initializr’dan backendimiz için gerekli olan servislerimizi oluşturmaya başlıyoruz.

## Discovery Server:

Tüm mikroservislerin ayağa kalktığında kendini kaydettiği merkezi rehber. "Hangi servis hangi IP ve portta çalışıyor?" sorusunun cevabı burada. Servisler birbirleriyle direkt IP üzerinden değil, isim üzerinden haberleşir. Gerçek adresi Eureka'dan öğrenir.

Initializr’dan oluşturduğumuz yapıyı indirdikten sonra, öncelikle: 

DiscoveryServerApplication’a

`@EnableEurekaServer`

Annotationu ekliyoruz. Bu sayede servisimiz Euroke server olarak çalışıyor.

Sonrasında application.properties dosyamıza

```bash
spring.application.name=discovery-server
server.port=8761

eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

Portu ve gerekli ayarları giriyoruz.

Spring'de Eureka Server bağımlılığı içine Eureka Client kodu da gömülü geliyor. Yani sadece `@EnableEurekaServer` yazınnca, arka planda client davranışı da aktif oluyor. Biz sadece server özelliklerini istediğimiz için onu kapatıyoruz.

Bu yüzden:

Euroka serverin kendini de bir servis olarak görmemesi ve kayıt etmemesi için porttan sonra gelen ilk satırı ekledik.

Euroka clientlar normalde diğer servislerin listesini Euroka serverdan çekiyor (fetch). Euroka server zaten buna sahip olduğu için bu özelliği de kapatıyoruz.

## Config Server:

 Tüm mikroservislerin `application.properties` dosyalarının tek merkezden yönetildiği yer. Her servis kendi içinde config tutmak yerine ayağa kalkarken Config Server'a sorar ve ayarlarını oradan çeker. Bir şeyi değiştirmek istediğinde tek bir yerden yaparsın, tüm servisler etkilenir.

### NOT: Eğer 2 application.properties dosyasında bir şekilde çakışma varsa, her zaman Config Serverdaki kazanır.

Initializr’dan oluşturduğumuz yapıyı indirdikten sonra, öncelikle: 

ConfigServerApplication’a

`@EnableConfigServer`

Annotationu ekliyoruz. Bu sayede servisimiz config server olarak çalışıyor.

Sonrasında application.properties dosyamıza

```bash
spring.application.name=config-server
server.port=8888

# Eureka'ya kaydet
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/

# Config dosyalarını local klasörden oku 
spring.cloud.config.server.native.search-locations=classpath:/configs
spring.profiles.active=native
```

Portu ve gerekli ayarları giriyoruz.
Eureka’nın nerde olduğunu belirtmek için porttan sonraki ilk satırdaki komutu kullanıyoruz.

Bu sayede servis ayağa kalktığında Eureka’ya 8761 portundan gidiyor ve ben config serverım. 8888 portundan ayağı kalktım diyor. Diğer servisler de Euroke’ya config server’ı sorunca bu sayede haberleri oluyor.

Sonraki ilk komutumuz, config serverın gerekli diğer mikroservislerin config dosyalarını nerde barındıracağını belirtiyoruz (yol). `classpath:/configs` diyince `src/main/resources/configs/` klasörüne bakıyor.

En sondaki satır ise, config dosyalarının yerel olarak okunacağını belirtiyor.

Bu ayarları yaptıktan sonra,`src/main/resources`’a gidip, configs klasörünü oluşturuyoruz.

Bir örnek ile anlatacak olursak, eğer auth-service.properties dosyasını `src/main/resources/configs/`

klasörüne kaydettiysek, auth-service ayağa kalkarken Config Server'a sorar, bu dosyayı bulur ve gönderir.

## API-Gateway

Dışarıdan gelen tüm isteklerin ilk karşılandığı nokta. Client (React veya Postman) hangi servise gideceğini bilmez, sadece Gateway'e istek atar. Gateway isteği bakarak "bu `/auth/**` ile başlıyor, auth-service'e yönlendir" der.

### Dependencies:

**Gateway (Spring Cloud Gateway — Reactive)**

Reactive seçiyoruz çünkü Gateway'in görevi istek yönlendirmek . Reactive (WebFlux) bu tür I/O ağırlıklı işlerde çok daha az thread kullanarak daha fazla isteği kaldırır.

**Olması gerektiği gibi, Eureka Discovery Client içermeli.**

Gateway hangi servise yönlendireceğini IP:port olarak hardcode etmek yerine Eureka'dan sorar. "auth-service nerede?" diye sorar, Eureka cevap verir. Servis portu değişse bile Gateway etkilenmez.

**Spring Boot Actuator**
Gateway'in sağlık durumunu izlemek için. `/actuator/health` endpoint'i otomatik açılır, Eureka buraya bakarak "bu servis hâlâ ayakta mı?" diye kontrol eder.

### Application Properties:

```bash
spring.application.name=api-gateway
server.port=8080

# Config Server'dan ayarları çek
spring.config.import=configserver:http://localhost:8888

# Eureka'ya kaydol
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

Sonrasında, Config Server'da configs içinde api-gateway.properties dosyasını oluşturup,

```bash
# Eureka'daki servis adına göre otomatik route oluştur
spring.cloud.gateway.discovery.locator.enabled=true
spring.cloud.gateway.discovery.locator.lower-case-service-id=true
```

İlk komut ile, Gateway Eureka’ya kayıtlı her servisi otomatik olarak route eder. Yani diyelim ki order-service Eureka’ya kayıtlıysa. order-service'e yani http://localhost:8080/order-service/** atılan istekler otomatik olarak order-service’e gider. Elle tanımlamak zorunda kalmayız.

En alttaki komut ile de Gateway servis isimkerini küçük harfe çevirir. Eureka servis isimlerini büyük harfle kaydeder. Bu sayede küçük harflerle arama yaparız urlde.