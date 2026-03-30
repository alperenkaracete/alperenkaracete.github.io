---
layout: post
title: "FastAPI ve SQLAlchemy: Yaptığım 3 Kritik Mimari Hata ve Çözümleri"
date: 2026-03-30
categories: [Backend, Python, API Mimarisi]
tags: [fastapi, sqlalchemy, debugging, pydantic, RESTful]
author: "Alperen Karaçete"
---

AIOps süreçleri ve otonom ajanlar için Zero Trust (Sıfır Güven) mimarisine uygun bir backend sistemi geliştirirken, FastAPI ve SQLAlchemy'nin birleştiği noktalarda oldukça öğretici mimari zorluklarla karşılaştım. Dışarıdan bakıldığında basit bir hata gibi görünen ama arka planda web mimarisinin temel kurallarına dayanan bu 3 kritik sorunu ve çözümlerini bu yazıda derledim.

## Hata 1: Pydantic `response_model` Tuzağı (Görünmez Veri Kaybı)

**Sorun:**
Yeni bir veri (örneğin bir AI Ajanı) oluşturmak için POST metodu yazıyordum. Veritabanı objeyi başarıyla oluşturuyor, `id` ve `created_at` (oluşturulma tarihi) gibi alanları otomatik olarak atıyordu. Ancak API'den dönen JSON yanıtında bu değerler tamamen eksikti!

**Arka Plandaki Neden:**
FastAPI, Pydantic şemalarını inanılmaz katı bir gümrük memuru gibi kullanır. `response_model` olarak dışarıdan kullanıcının girdiği alanları kapsayan `AgentCreate` şemasını verdiğim için, sistem kullanıcıya veriyi göndermeden hemen önce şemada tanımlı olmayan `id` ve `status` gibi verileri güvenlik gereği (veri sızıntısını önlemek için) acımasızca kesip atıyordu.

**Çözüm:**
Yanıt modeli, POST isteğinde kullanılsa bile her zaman veritabanı objesinin tam halini temsil eden bir `Response` şeması olmalıdır.

```python
# YANLIŞ: Girdi şeması yanıt olarak kullanılıyor
@app.post("/agents/", response_model=schemas.AgentCreate)

# DOĞRU: Çıktı şeması (ID ve tarih gibi ekstra alanları içerir) kullanılıyor
@app.post("/agents/", response_model=schemas.AgentResponse)

## Hata 2: Name Shadowing — `models` Klasörü ve `models.py` Çakışması

### Sorun

`AttributeError: module 'models' has no attribute 'Agent'`

Sınıf dosyada açıkça tanımlanmış olmasına rağmen Python onu bulamıyordu.

### Neden Oldu?

Proje yapısı şöyleydi:
```
models/
├── __init__.py
└── models.py   ← Agent sınıfı burada
```

`main.py` içinde `import models` yazdığımda Python,
`models.py` dosyasını değil `models/` klasörünü (ve içindeki
boş `__init__.py`'ı) referans aldı.
Klasörün kendisinde `Agent` sınıfı olmadığı için hata verdi.
Buna **Name Shadowing** (İsim Gölgeleme) deniyor.

### Çözüm

Modüler yapılarda import işlemleri tam yolu göstererek yapılmalı:
```python
# Yanlış
import models
db_agent = models.Agent(...)

# Doğru
from models.models import Agent
db_agent = Agent(**agent.model_dump())

```

Bir diğer yöntem ise
__init__.py'nin içine böyle kaydetmek.
```
from .models import Agent
```

---

## Hata 3: Routing Collision — Aynı Path'te İki Farklı Endpoint

### Sorun

Aynı tablo için iki GET endpoint yazmak istedim:
```python
@app.get("/agents/{agent_name}")  # İsme göre ara
@app.get("/agents/{agent_id}")    # ID'ye göre ara
```

`/agents/5` isteği attığımda `422 Unprocessable Entity` alıyordum
ya da uygulama çöküyordu.

### Neden Oldu?

FastAPI route'ları **yukarıdan aşağıya** okur.
`/agents/5` geldiğinde ilk kuralla eşleşti:
"URL'de `/agents/` var ve devamında bir değer var →
bu `agent_name` olmalı."

`5` değerini isim zannedip veritabanında "5" isminde ajan aradı.
İkinci endpoint'e hiç ulaşamadı.

### Çözüm

Path'leri birbirinden ayırt edilebilir yap:
```python
# İsme göre arama 
@app.get("/agents/name/{agent_name}")
def get_agent_by_name(agent_name: str, db: Session = Depends(get_db)):
    ...

# ID'ye göre arama — standart yapı
@app.get("/agents/{agent_id}")
def get_agent_by_id(agent_id: int, db: Session = Depends(get_db)):
    ...
```

`/agents/name/olykube` → isim araması
`/agents/5` → ID araması

Artık çakışma yok.

---

## Öğrendiklerim

Bu üç hata şunu gösterdi: kodun çalışması yetmez,
framework'ün nasıl düşündüğünü de anlamak gerekiyor.

- Python'da klasör ve dosya isimlerini çakıştırma
- FastAPI route'ları yukarıdan aşağıya okur,
  sıralama ve path tasarımı kritik
- Modüler yapılarda her zaman tam path ile import yap