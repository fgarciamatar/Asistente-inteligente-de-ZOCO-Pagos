# Asistente Inteligente de ZOCO Pagos

Agente conversacional que responde consultas sobre los productos y servicios
de ZOCO Pagos usando exclusivamente información pública del sitio oficial.
Recupera el contexto por búsqueda semántica, cita la fuente de cada respuesta
y deriva a un asesor humano cuando no tiene el dato.

## Stack

| Componente | Tecnología |
|---|---|
| Orquestación | n8n |
| Scraping | Firecrawl |
| Base vectorial y logs | Supabase (PostgreSQL + pgvector) |
| Canales | Chat web, API HTTP, Telegram |

### Modelos

| Uso | Modelo |
|---|---|
| Agentes conversacionales (general y precios) | `models/gemini-3.5-flash` |
| Analista de métricas | `models/gemini-3.5-flash` |
| Clasificador de intención | `models/gemini-2.5-flash` |
| Embeddings | `models/gemini-embedding-001` (3072 dimensiones) |

El clasificador corre sobre un modelo más liviano a propósito: elegir entre
seis categorías es una tarea acotada que no justifica el costo ni la latencia
del modelo que responde. Los agentes, que además ejecutan herramientas y
razonan sobre el contexto recuperado, sí usan el modelo mayor.

## Diagrama de Arquitectura
```mermaid
flowchart LR
    subgraph CANALES["Canales de entrada"]
        CH[Chat web]
        API[API HTTP]
        TG[Telegram]
    end

    subgraph N8N["n8n · orquestación"]
        W1["Workflow 1<br/>Ingesta"]
        W2["Workflow 2<br/>Asistente"]
        W3["Workflow 3<br/>Métricas"]
    end

    subgraph SUPA["Supabase · PostgreSQL"]
        VEC[("documents<br/>pgvector")]
        LOGS[("conversaciones<br/>derivaciones")]
    end

    WEB["Web pública<br/>de ZOCO"]
    FC["Firecrawl"]
    GEM["Google Gemini"]

    WEB -->|HTML renderizado| FC
    FC -->|markdown limpio| W1
    W1 -->|embeddings| VEC

    CH --> W2
    API --> W2
    TG --> |consulta autorizada| W3
    TG --> W2

    W2 -->|búsqueda semántica| VEC
    W2 -->|registro de cada consulta| LOGS
    W3 -->|SQL solo lectura| LOGS

    GEM -.->|LLM + embeddings| W1
    GEM -.->|LLM| W2
    GEM -.->|LLM| W3
```
