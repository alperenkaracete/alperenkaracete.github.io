🐳 Docker Dünyasında "localhost" Çıkmazı: 3 Adımda Kesin Çözüm
Dockerize ettiğiniz uygulamanız lokalde canavar gibi çalışırken, konteyner içine girdiğinde veritabanına küsüyorsa doğru yerdesiniz. OlyKube projesini Docker’a taşırken karşılaştığımız o meşhur engelleri ve profesyonel çözüm yollarını inceleyelim.

1. Veritabanı Bağlantı Adresi (Networking)
Konteyner içinden dışarıdaki (host) veritabanına ulaşmaya çalışırken en sık yapılan hata budur.

❌ HATALI KOD: Hardcoded Localhost
```python
SQLALCHEMY_DATABASE_URL = "postgresql://postgres:1234@localhost:5432/olykube"
```
Sorun: Konteyner içindeki localhost, konteynerin kendisidir. Kendi içinde de bir veritabanı servisi çalışmadığı için Connection Refused hatası kaçınılmazdır.


✅ DOĞRU ÇÖZÜM: Sihirli DNS
```python
SQLALCHEMY_DATABASE_URL = "postgresql://postgres:1234@host.docker.internal:5432/olykube"
```
Neden Çalıştı? host.docker.internal, Docker'ın konteynerden çıkıp ana makineye (senin bilgisayarına) ulaşması için sunduğu özel bir "çıkış kapısı"dır.


2. Yapılandırma Yönetimi (Environment Variables)
Uygulamanın esnekliğini artırmak için ayarları kodun içine gömmemek gerekir.

❌ HATALI KOD: Sağır Uygulama
```python
# database.py
SQLALCHEMY_DATABASE_URL = "postgresql://user:pass@172.23.0.1:5432/olykube"
```
Sorun: Kodun dış dünyaya "sağır" kalır. Docker çalıştırırken dışarıdan istediğin kadar .env dosyası ver, Python kodu inatla içindeki o sabit metni okur.


✅ DOĞRU ÇÖZÜM: Dinamik Yapılandırma
```python
# database.py
import os

SQLALCHEMY_DATABASE_URL = os.getenv(
    "SQLALCHEMY_DATABASE_URL", 
    "postgresql://postgres:1234@host.docker.internal:5432/olykube"
)
```

Neden Çalıştı? os.getenv sayesinde kodun artık "duyabiliyor". Önce dışarıdaki .env dosyasına bakar, orada bir talimat bulamazsa (yedek plan olarak) koddaki varsayılan adresi kullanır.

3. Redis ve Diğer Servisler İçin IP Yönetimi
❌ HATALI KOD: Sabit IP Tuzağı
```python
r = redis.Redis(host="172.23.0.1", port=6379, db=0)
```
Sorun: 172... ile başlayan IP'ler Docker ağ köprüsüne (bridge) aittir ve değişkendir. Bilgisayarınızı yeniden başlattığınızda bu IP değişirse kodunuz çöker.

✅ DOĞRU ÇÖZÜM: İsimle Haberleşme
```python
import os

# REDIS_HOST'u .env'den al, bulamazsan host.docker.internal'ı dene
redis_host = os.getenv("REDIS_HOST", "host.docker.internal")
r = redis.Redis(host=redis_host, port=6379, db=0)
```
Neden Çalıştı? IP adresini takip etme zahmetinden kurtulduk. REDIS_HOST değişkenini .env içinden yöneterek sistemi her ortamda taşınabilir hale getirdik.

🛠️ Final Operasyonu: Kurşun Geçirmez Komut
Kodunuzu her değiştirdiğinizde Docker'ın o "fotoğrafı" (imajı) güncellemesi gerektiğini unutmayın. İşte o "Altın Seri" komutlar:

İmajı Yeniden İnşa Et (Build):

```Bash
docker build -t olykube-backend .
```
Eski Konteyneri Sil:

```Bash
docker rm -f olykube-api
```

Yeni Ayarlarla Çalıştır:

```Bash
Bash
docker run -d \
  --name olykube-api \
  -p 8000:8000 \
  --add-host=host.docker.internal:host-gateway \
  --env-file .env \
  olykube-backend
```
💡 Mühendislik Özeti
Koddaki String'den Kurtul: Ayarları her zaman os.getenv ile dışarıdan al.

IP Adreslerini Unut: host.docker.internal gibi isimler kullan.

Build Etmeyi Atlamayın: Kod değiştiyse imaj da değişmelidir.