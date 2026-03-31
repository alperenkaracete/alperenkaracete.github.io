# LangGraph ve Tavily Entegrasyonu: Yaptığım Hatalar ve Çözümleri

Ajan mimarisini LangGraph ve Tavily ile internete bağlarken
sürüm uyuşmazlıkları ve asenkron mimari yüzünden birkaç hata yaptım.
Bunları ve nasıl çözdüğümü paylaşıyorum.

---

## 1. Sanal Ortam (venv) Dışında Uvicorn Çalıştırmak

### Hata
```bash
uvicorn main:app --reload
```

Paketleri sanal ortama kurmama rağmen sunucuyu başlatınca
`ModuleNotFoundError: No module named 'langchain_community'` aldım.
İşletim sistemi `uvicorn` komutunu global Python ortamında aradı.

### Çözüm
```bash
python -m uvicorn main:app --reload
```

Bu şekilde aktif sanal ortamın Python motorunu kullanmaya zorladım.

---

## 2. LangChain Sürüm Kaosu — Tavily Ayrı Pakete Taşındı

### Hata
```python
from langchain_community.tools.tavily_search import TavilySearchResults
```

LangChain ekibi Tavily'i `community` paketinden çıkarmış,
eski import çöküyordu.

### Çözüm
```bash
pip install -U langchain-tavily
```
```python
from langchain_tavily import TavilySearchResults
```

---

## 3. Eski Nesil ve Yeni Nesil Ajan Çatışması

### Hata
```python
from langchain.agents import create_agent

agent_app = create_agent(
    llm,
    tools,
    checkpointer=memory,
    state_modifier=SystemMessage(...)  # sürümden sürüme değişiyor
)
```

IDE'nin "deprecated" uyarısına kanıp `create_agent` kullandım.
Eski LangChain mimarisi LangGraph'ın `checkpointer` parametresini
tanımadığı için `TypeError` fırlattı.
Parametre isimleri de sürümden sürüme değişiyordu:
`state_modifier` → `prompt` gibi.

### Çözüm
```python
from langgraph.prebuilt import create_react_agent

agent_app = create_react_agent(
    llm,
    tools,
    checkpointer=memory,
    prompt=SystemMessage(content=system_prompt)
)
```

Modern LangGraph motorunu kullandım.
IDE uyarıları her zaman doğru yönlendirmiyor —
kaynak koda ve resmi dokümana bakmak şart.

---

## 4. Ollama Kapalıyken API'nin Kilitlenmesi

### Hata
```python
llm = ChatOllama(model="qwen3.5:9b", base_url="http://192.168.1.6:11434")
result = await agent_app.ainvoke(...)
```

Ollama kapalıysa LangChain saniyelerce bekliyordu.
`ChatOllama` tembel çalıştığı (lazy evaluation) için
hatayı en başta yakalamıyordu, API kilitleniyordu.

### Çözüm — Fail-Fast Prensibi
```python
import httpx

try:
    async with httpx.AsyncClient() as client:
        await client.get("http://192.168.1.6:11434/", timeout=2.0)
except httpx.RequestError:
    raise HTTPException(
        status_code=503,
        detail="Yapay Zeka motoruna ulaşılamıyor."
    )
```

Ağır işlemi başlatmadan önce 2 saniyelik bir health check attım.
Cevap gelmezse anında 503 döndürüyorum.
Buna **Fail-Fast** prensibi deniyor —
hata erken yakalanırsa sistem daha az hasar görür.

---

## Öğrendiklerim

- LangChain ekosisteminde paketler çok hızlı kırılabiliyor,
  güncel dokümantasyon şart
- IDE'lerin "deprecated" uyarıları her zaman doğru yönlendirmiyor,
  kaynak koda bakmak en sağlıklısı
- `python -m uvicorn` kullanmak venv sorunlarını kökünden çözüyor
- Dış servislere bağımlı işlemlerde her zaman
  önceden health check yap