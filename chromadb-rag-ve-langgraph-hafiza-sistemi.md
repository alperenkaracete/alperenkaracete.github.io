# ChromaDB, RAG Pipeline ve LangGraph Hafıza Sistemi: Öğrendiklerim

Bu yazıda vektör veritabanı, RAG mimarisi ve LangGraph'ın
hafıza sistemi üzerine öğrendiklerimi ve yaptığım hataları paylaşıyorum.

---

## Bölüm 1: ChromaDB ve RAG Pipeline

### Embedding Nedir?

ChromaDB metinleri direkt karşılaştırmıyor — önce sayı vektörlerine
çeviriyor, sonra bu vektörler arasında mesafe hesaplıyor.
```
"kubernetes güvenlik"
        ↓
   Embedding modeli
        ↓
   [0.23, 0.87, 0.12, ...]
        ↓
   ChromaDB'deki vektörlerle karşılaştır
        ↓
   En yakın vektörleri bul → dökümanları döndür
```

"kubernetes" kelimesini yazmasan bile "container orkestrasyon" yazsan
Kubernetes dökümanları geliyor — kelime eşleşmesi değil, anlam benzerliği.

Varsayılan olarak ChromaDB `all-MiniLM-L6-v2` modelini kullanıyor,
ilk çalıştırmada internet üzerinden indiriyor, sonra local cache'de kalıyor.

---

### Chunk'lama Nedir?

Büyük dökümanları direkt ChromaDB'ye atmak yerine küçük parçalara
(chunk) bölüyoruz. Nedeni basit — embedding modelleri uzun metinlerde
anlam kaybı yaşıyor, küçük parçalar daha isabetli sonuç veriyor.
```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    chunk_overlap=20
)
```

`chunk_size=200` → her parça ~200 karakter
`chunk_overlap=20` → her parça bir öncekiyle 20 karakter kesişiyor

Overlap neden var? Overlap olmadan bir cümle iki chunk'ın tam ortasında
bölünebilir, anlam kaybolur. 20 karakterlik geri dönüş bu problemi önlüyor.

---

### Hata: ChromaDB Verisinin Uçması

#### Sorun
```python
client = Client()  # varsayılan
```

Uygulamayı her yeniden başlatınca ChromaDB sıfırlanıyordu.
`Client()` RAM'de çalışıyor — uygulama kapanınca her şey siliniyor.

#### Çözüm
```python
client = chromadb.PersistentClient(path="./chroma_db")
```

`./chroma_db` klasörüne yazıyor, uygulama kapansa da veri kalıyor.

---

### RAG Agent Mimarisi

Agent'a iki tool verdim — sıralama önemli:
```python
tools = [search_knowledge_base, TavilySearch(max_results=2)]
```

1. `search_knowledge_base` → önce local ChromaDB'ye bak
2. `TavilySearch` → bilmiyorsa internette ara

Agent tool listesini yukarıdan aşağıya değerlendiriyor.
Kubernetes sorusunda ChromaDB yetti, Tavily'e hiç gitmedi.
Güncel haber sorusunda ChromaDB boş döndü, Tavily devreye girdi.

---

## Bölüm 2: LangGraph Hafıza Sistemi

### MemorySaver vs PostgreSQL Checkpointer

Başlangıçta `MemorySaver` kullandım:
```python
from langgraph.checkpoint.memory import MemorySaver
memory = MemorySaver()
```

Çalışıyordu ama bir sorunu vardı — uygulama yeniden başlayınca
tüm konuşma geçmişi siliniyor. RAM'de tuttuğu için kalıcı değil.

Çözüm olarak `AsyncPostgresSaver`'a geçtim:
```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async with AsyncPostgresSaver.from_conn_string(DATABASE_URL) as checkpointer:
    await checkpointer.setup()
    agent_app = create_react_agent(llm, tools, checkpointer=checkpointer)
```

Artık konuşma geçmişi PostgreSQL'e yazılıyor.
Uygulama kapanıp açılsa da model her şeyi hatırlıyor.

---

### Hata: Connection Scope Hatası

#### Sorun
```
psycopg.OperationalError: the connection is closed
```

`async with` bloğu sadece `setup()` satırını kapsıyordu,
geri kalan tüm kod bloğun dışındaydı.
Connection kapanıyor, sonra kullanmaya çalışınca crash oluyordu.
```python
# Yanlış
async with AsyncPostgresSaver.from_conn_string(DATABASE_URL) as checkpointer:
    await checkpointer.setup()  # sadece bu içeride

# Bloğun dışında — connection kapanmış
agent_app = create_react_agent(llm, tools, checkpointer=checkpointer)
result = await agent_app.ainvoke(...)
```

#### Çözüm
Tüm kodu `async with` bloğunun içine almak:
```python
# Doğru
async with AsyncPostgresSaver.from_conn_string(DATABASE_URL) as checkpointer:
    await checkpointer.setup()
    agent_app = create_react_agent(llm, tools, checkpointer=checkpointer)
    result = await agent_app.ainvoke(...)
    return result["messages"][-1].content
```

`async with` bir context manager — bloğun dışına çıkınca
connection otomatik kapanıyor. Connection'a bağlı her şey içeride olmalı.

---

### İlginç Keşif: İki Paralel Hafıza Sistemi

Başlangıçta hem kendi `chat_histories` tabloma yazıyordum
hem de LangGraph kendi tablolarına yazıyordu.
```
PostgreSQL tabloları:
├── chat_histories        ← benim yazdığım servis
├── checkpoints           ← LangGraph'ın kendi tablosu
├── checkpoint_blobs      ← LangGraph'ın kendi tablosu
└── checkpoint_writes     ← LangGraph'ın kendi tablosu
```

`chat_histories` tablosunu düşürünce model hâlâ geçmişi hatırlıyordu.
Çünkü LangGraph'ın tablolarına dokunmamıştım.

Sonuç: `chat_histories` tablosu ve `save_chat_history` servisi gereksizdi.
LangGraph zaten her şeyi kendi tablolarına yazıyor ve model her zaman
okuyabiliyor. Kendi servisimi kaldırdım, tek kaynak LangGraph checkpointer oldu.

---

## Öğrendiklerim

- ChromaDB varsayılan olarak RAM'de çalışır,
  `PersistentClient` ile kalıcı hale getirilmeli
- Chunk overlap cümle bütünlüğünü korumak için var,
  sıfıra düşürülmemeli
- `async with` context manager'larda connection tüm işlemler
  boyunca açık kalmalı, scope dışına çıkınca kapanıyor
- Framework'lerin kendi iç mekanizmalarını anlamadan
  üzerine servis yazmak gereksiz kod üretiyor —
  LangGraph checkpointer'ı keşfedince kendi hafıza servisim gereksiz kaldı