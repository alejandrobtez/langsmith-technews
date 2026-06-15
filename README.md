# Tech News Agent — LangGraph + LangSmith

Agente que busca las noticias tecnológicas del día, las resume con IA y espera aprobación humana antes de guardar el informe.

---

## Flujo general

```
START → agent ──(tool call)──→ tools → agent
                  │
                  └─(JSON topics)──→ parse_topics → analyze_topics → write_report
                                                                           │
                                                          ┌── INTERRUPT ───┘
                                                          ▼
                                                    human_review → save → END
```

---

## Celda 2 — Claves y configuración

Se definen las variables de entorno necesarias: Azure OpenAI (el LLM), Tavily (búsqueda web) y LangSmith.  
Con solo poner `LANGCHAIN_TRACING_V2=true` y la API key, LangSmith registra automáticamente cada ejecución sin tocar el código.

---

## Celda 3 — Las 3 Tools

Las tools son funciones Python decoradas con `@tool` que el agente puede invocar durante su ejecución.

**`get_tech_headlines`** — llama a Tavily buscando los titulares tech más importantes del día de hoy. Es la herramienta principal: el agente la llama primero para tener información sobre la que razonar.

**`get_topic_details`** — dado un tema concreto, hace una segunda búsqueda en Tavily para obtener contexto adicional. La usa el subgrafo, no el agente principal.

**`save_report`** — abre `tech_report.md` en disco y escribe el informe final. Solo se ejecuta si el usuario aprueba en la fase 2.

---

## Celda 4 — El Subgrafo

El subgrafo es un grafo LangGraph independiente (`StateGraph`) con su propio estado (`TopicState`). Se invoca una vez por cada tema detectado y tiene dos nodos:

**`fetch_details`** — llama a `get_topic_details` con el nombre del tema para obtener más contexto de Tavily.

**`generate_summary`** — envía al LLM el tema + los titulares originales + el contexto adicional y pide un resumen ejecutivo de máximo 4 líneas con una línea de "Relevancia:".

El subgrafo encapsula la lógica de análisis por tema y se reutiliza 3 veces, una por cada tema, desde `node_analyze_topics`.

---

## Celda 5 — El Grafo Principal

Define `AgentState` (el estado compartido por todos los nodos) y construye el grafo con estos nodos:

**`node_agent`** — el LLM con las tools vinculadas (`bind_tools`). En la primera pasada llama a `get_tech_headlines`; en la segunda procesa los resultados y devuelve un JSON con los 3 temas.

**`node_tools`** — ejecuta la tool que el agente pidió y devuelve el resultado como `ToolMessage` para que el agente lo lea.

**`node_parse_topics`** — extrae el JSON que devolvió el agente con los 3 temas usando regex, los parsea y los guarda en `state["topics"]`.

**`node_analyze_topics`** — itera sobre los 3 temas e invoca el subgrafo para cada uno, acumulando los resúmenes en `state["summaries"]`.

**`node_write_report`** — combina temas y resúmenes en un documento Markdown y lo guarda en `state["report_content"]` para persistirlo en el checkpoint.

**`node_human_review`** — nodo vacío que actúa como punto de pausa. El grafo se compila con `interrupt_before=["human_review"]`, lo que hace que el grafo se detenga justo antes de ejecutarlo.

**`node_save`** — lee `state["approved"]`; si es `True` llama a `save_report` y escribe el `.md`, si es `False` cancela sin hacer nada.

### Routing condicional

`should_continue` — tras cada respuesta del agente, mira si hay `tool_calls`: si los hay va a `tools`, si no va a `parse_topics`.

`has_topics` — tras parsear, si hay temas va a `analyze_topics`, si no va a `no_topics` y termina.

---

## Celda 6 — Visualización del grafo

Genera una imagen PNG del grafo completo (incluyendo el subgrafo con `xray=True`) usando Mermaid, útil para mostrar la arquitectura visualmente.

---

## Celda 7 — Fase 1: arrancar el agente

Se crea un `config` con un `thread_id` único (UUID) que identifica esta sesión en el checkpoint.  
Se invoca el grafo con el estado inicial: el agente busca, analiza e imprime el informe. El grafo se detiene solo antes de `human_review` y devuelve el control al usuario.

---

## Celda 8 — Fase 2: Human in the Loop

El usuario decide con `approved = True / False`.  
Se llama a `graph.update_state(config, {"approved": approved})` para inyectar la decisión en el checkpoint, y luego `graph.invoke(None, config=config)` con `None` para reanudar exactamente donde se quedó — no reinicia, continúa.  
Si está aprobado, `node_save` escribe `tech_report.md` en disco.

---

## Celda 9 — Debug

Inspecciona el estado final del checkpoint: cuántos temas se extrajeron, cuántos resúmenes se generaron, si fue aprobado y el historial completo de mensajes.

---

## Tracing en LangSmith

Cada nodo, cada LLM call y cada tool quedan registrados en [smith.langchain.com](https://smith.langchain.com) bajo el proyecto `tech-news-agent`.  
Se puede ver el grafo recorrido, los inputs/outputs de cada paso y el tiempo total de ejecución.

---

## Estructura del proyecto

```
lang-graph-smith-tachnews/
├── tech_news_agent.ipynb   # notebook principal (9 celdas)
├── tech_report.md          # informe generado al aprobar en fase 2
└── README.md
```
