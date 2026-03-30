# FastAPI'de JWT Token ve Korumalı Endpoint: Öğrendiklerim

Bu yazıda `/protected` endpoint'ini yazarken yaptığım hataları
ve JWT token konusunda öğrendiklerimi paylaşıyorum.

---

## 1. Token URL'de Gitmez, Header'da Gider

### Hata
```
GET /protected?token=eyJhbG...
```

Token URL'de görününce tarayıcı geçmişine, sunucu loglarına
ve proxy'lere düşer. Güvenlik açığı.

### Doğrusu
```bash
curl -X GET http://localhost:8000/protected \
  -H "Authorization: Bearer eyJhbG..."
```

Token `-H` ile header'da gidiyor. HTTPS ile şifrelenmiş
iletiliyor, ortada kimse göremez.

---

## 2. GET mi POST mu?

### Yanılgı
`/protected`'ı önce POST olarak yazdım.
"POST body'e gönderir, daha güvenli" diye düşündüm.

### Doğrusu
GET/POST seçimi güvenlikle değil, **ne yaptığınla** ilgili:

| Method | Ne zaman? |
|---|---|
| GET | Veri okurken |
| POST | Veri oluştururken |
| PUT | Veri güncellerken |
| DELETE | Veri silerken |

`/protected`'da sadece "ben kimim" diye soruyoruz —
veri okuyoruz. Bu yüzden GET.

---

## 3. `get_current_user`'ı Manuel Çağırmak

### Hata
```python
@app.get("/protected")
def protected(token: str):
    return get_current_user(token)  # manuel çağrı
```

`get_current_user` içinde `db: Session = Depends(get_db)` var.
Manuel çağırınca FastAPI devreye girmiyor —
`db`'ye gerçek session yerine `Depends` objesi gidiyor.
`'Depends' object has no attribute 'query'` hatası alıyorsun.

### Doğrusu
```python
@app.get("/protected")
def protected(current_user = Depends(get_current_user)):
    return {"email": current_user}
```

`Depends` FastAPI'ye "bu fonksiyonu sen çağır,
gerekli parametreleri sen inject et" diyor.
`db`, `token` — hepsini FastAPI kendisi hallediyor.

---

## 4. URL'e Ne Gider, Ne Gitmez?

### URL'e gidebilecekler:
- Path parametresi: `/todos/5`
- Query parametresi: `/todos?skip=0&limit=10`

### URL'e asla gitmemeli:
- Şifreler
- Token'lar
- Kişisel veriler

Bunlar ya **header'da** ya da **body'de** gitmeli.

---

## curl Parametreleri

Testlerde curl kullandım, parametreleri öğrendim:
```bash
curl -X POST http://localhost:8000/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@test.com", "password": "123"}'
```

- `-X` → HTTP metodu (GET, POST, PUT, DELETE)
- `-H` → Header (metadata: kim olduğun, hangi format)
- `-d` → Body (asıl veri)

Zarfa benzetirsek:
- `-H` → zarfın üstündeki bilgiler
- `-d` → zarfın içindeki mektup

---

## Öğrendiklerim

- Token header'da gider, URL'de değil
- GET/POST seçimi güvenlikle değil işlemin türüyle ilgili
- `Depends` olan fonksiyonlar manuel çağrılamaz,
  FastAPI inject etmeli
- URL'e sadece path ve query parametreleri gider,
  hassas veri asla