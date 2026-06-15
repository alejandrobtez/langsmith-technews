# Tech News Agent — LangGraph + LangSmith

Agente que busca las noticias tecnológicas del día, las resume con IA y espera aprobación humana antes de guardar el informe.

---

## ¿Qué hace?

1. **Busca titulares** con Tavily (`get_tech_headlines`) y extrae los 3 temas más relevantes del día.
2. **Analiza cada tema** en paralelo con un subgrafo: busca contexto adicional y genera un resumen ejecutivo.
3. **Compone un informe** Markdown con los 3 resúmenes.
4. **Pausa y espera** la aprobación humana antes de guardar (`interrupt_before`).
5. **Guarda `tech_report.md`** si el usuario aprueba, o cancela sin escribir nada.

---

## Arquitectura

```
START → agent ──(tool call)──→ tools → agent
                  │
                  └─(JSON topics)──→ parse_topics → analyze_topics → write_report
                                                                           │
                                                          ┌── INTERRUPT ───┘
                                                          ▼
                                                    human_review → save → END
```

### Subgrafo (por cada tema)
```
fetch_details → generate_summary → END
```

---

## Tecnologías

| Componente | Uso |
|---|---|
| **LangGraph** | Orquestación del grafo y subgrafo |
| **LangSmith** | Tracing automático de cada nodo y LLM call |
| **Azure OpenAI** (`gpt-4o-mini`) | Extracción de temas y generación de resúmenes |
| **Tavily** | Búsqueda web en tiempo real |
| **MemorySaver** | Persistencia del estado entre las dos fases de ejecución |

---

## Conceptos LangGraph utilizados

| Concepto | Dónde aparece |
|---|---|
| `StateGraph` + `TypedDict` | `AgentState` (grafo principal) y `TopicState` (subgrafo) |
| `add_messages` | Reducer que acumula mensajes sin sobrescribir |
| `bind_tools` | Vincula `get_tech_headlines` al LLM del agente |
| `add_conditional_edges` | Routing `agent→tools/parse_topics` y `parse_topics→analyze/no_topics` |
| `interrupt_before` | Pausa el grafo antes de `human_review` para aprobación manual |
| `MemorySaver` | Checkpointer que persiste el estado entre las dos invocaciones |
| Subgrafo compilado | `topic_subgraph` invocado desde `node_analyze_topics` |

---

## Ejecución

### Requisitos
```bash
pip install langchain langchain-openai langgraph tavily-python langsmith
```

### Claves necesarias (celda 2 del notebook)
- `AZURE_OPENAI_ENDPOINT` / `AZURE_OPENAI_API_KEY` / `AZURE_OPENAI_DEPLOYMENT`
- `TAVILY_API_KEY`
- `LANGCHAIN_API_KEY` + `LANGCHAIN_TRACING_V2=true` → activa LangSmith automáticamente

### Pasos
```
1. Ejecutar celda 5  →  compila el grafo
2. Ejecutar celda 7  →  Fase 1: el agente busca, analiza e imprime el informe
3. Ejecutar celda 8  →  Fase 2: aprueba (True) o cancela (False)
```

El archivo `tech_report.md` aparece en la carpeta del proyecto si se aprueba.

---

## Human in the Loop

El grafo usa `interrupt_before=["human_review"]`: se detiene antes de guardar y espera que el usuario ejecute la celda 8.  
La reanudación se hace con `update_state` + `invoke(None)` — pasar `None` indica "continúa desde donde estabas".

---

## Tracing en LangSmith

Con `LANGCHAIN_TRACING_V2=true` cada ejecución queda registrada en [smith.langchain.com](https://smith.langchain.com) bajo el proyecto `tech-news-agent`. Se pueden ver los nodos recorridos, los inputs/outputs de cada LLM call y el tiempo de cada paso.

---

## Estructura del proyecto

```
lang-graph-smith-tachnews/
├── tech_news_agent.ipynb   # notebook principal (9 celdas)
├── tech_report.md          # informe generado (se crea al aprobar)
└── README.md
```
