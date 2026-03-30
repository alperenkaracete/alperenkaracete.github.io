# FastAPI'de Kullanıcı Kimlik Doğrulama: Yaptığım Hatalar ve Çözümleri

Roadmap'imin 2. haftasında FastAPI ile `/register` ve `/login` 
endpoint'lerini yazarken birkaç hata yaptım. 
Bunları ve nasıl çözdüğümü paylaşıyorum.

---

## 1. Login'de Şifreyi Tekrar Hash'lemek

### Hata
```python
hashed_pw = hash_password(user.password)
verify = verify_password(user.password, hashed_pw)
```

Kullanıcının girdiği şifreyi tekrar hash'leyip 
girdiği şifreyle karşılaştırıyordum. 
Bu her zaman `True` döner — şifre yanlış olsa bile.

### Doğrusu
```python
verify = verify_password(user.password, existing_user.hashed_password)
```

Veritabanındaki hash'i alıp onunla karşılaştırmak gerekiyor. 
`verify_password` içinde zaten hash'leme yapılıyor.

---

## 2. Login Endpoint'ini GET Yazmak

### Hata
```python
@app.get("/login")
def login(user: UserRegister, ...):
```

Şifre URL'de görünür hale gelir. 
Tarayıcı geçmişi, sunucu logları, her yerde saklanır.

### Doğrusu
```python
@app.post("/login")
def login(user: UserRegister, ...):
```

Şifre gibi hassas veriler her zaman POST ile body'de gitmeli.

---

## 3. Email Bulunamayınca Crash

### Hata
```python
existing_user = get_user_by_email(db, user.email)
verify = verify_password(user.password, existing_user.hashed_password)
```

`existing_user` None dönerse `None.hashed_password` 
diyince uygulama crash olur.

### Doğrusu
```python
if not existing_user:
    raise HTTPException(status_code=404, detail="Email bulunamadı")

if not verify_password(user.password, existing_user.hashed_password):
    raise HTTPException(status_code=401, detail="Yanlış şifre")
```

Önce email kontrolü, sonra şifre kontrolü. 
`raise HTTPException` fonksiyonu durdurur, 
sonraki satır çalışmaz.

---

## 4. C++ Alışkanlığı: `bool verify = ...`

### Hata
```python
bool verify = verify_password(user.password, hashed_pw)
```

C++ geçmişimden gelen bir alışkanlık. 
Python'da tip bildirimi bu şekilde yapılmıyor.

### Doğrusu
```python
verify = verify_password(user.password, hashed_pw)
```

Python'da tip belirtmek zorunlu değil. 
İstersen `verify: bool = ...` yazabilirsin ama zorunlu değil.

---

## 5. SQLAlchemy ve Pydantic Modellerini Karıştırmak

### Hata
`models/user.py` içinde hem Pydantic hem SQLAlchemy modeli vardı:
```python
# Aynı dosyada ikisi birden — yanlış
class User(Base):  # SQLAlchemy
    ...

class UserRegister(BaseModel):  # Pydantic
    ...
```

Import sırasında Python ikisini karıştırınca 
`ImportError` aldım.

### Doğrusu
İkisini ayır:

- `models/models.py` → SQLAlchemy modelleri (`User`, `Todo`)
- `models/user.py` → Pydantic modelleri (`UserRegister`)

SQLAlchemy modeli = veritabanı tablosu  
Pydantic modeli = API'ye gelen verinin şekli

---

## Öğrendiklerim

- `verify_password` veritabanındaki hash ile karşılaştırır, 
  yeni hash üretmez
- Hassas veriler POST body'de gitmeli, GET URL'sine değil
- None kontrolü her zaman önce yapılmalı
- SQLAlchemy ve Pydantic modelleri farklı amaçlar için 
  ayrı dosyalarda tutulmalı