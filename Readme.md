# 🧠 Proyecto: Conversación → Tripletas → Cypher / SQL

Este proyecto implementa un **pipeline conversacional completo** capaz de transformar una conversación natural con un LLM en **información estructurada**, almacenada en **SQLite** o **Neo4j**.

Su objetivo es extraer de forma fiable información clínica y de hábitos personales procedente de diálogos con personas mayores, garantizando coherencia, validación y una estructura de datos completamente cerrada.

---

## 🧩 Funcionamiento global del sistema

El sistema opera como un **ETL conversacional** compuesto por cuatro capas encadenadas:

### 1) Conversación con LLM → Paquetitos (`conv/`)

El usuario habla con un LLM.  
Cada turno genera un paquetito del estilo:

```bash
LLM: <pregunta o mensaje del asistente>
user_<nombre>: <respuesta del usuario>
```

El módulo:

- Detecta automáticamente el nombre
- Mantiene el historial
- Produce los paquetitos que alimentan todo el pipeline

---

### 2) Paquetitos → Resumen semántico (`conv2text/`)

El paquetito se transforma en frases explícitas y normalizadas, diseñadas para extraer información útil.

**Ejemplo:**

```bash
Luis practica yoga una vez por semana.
Luis toma lorazepam desde 2024-03.
Luis duerme mal desde hace dos semanas.
```

Estas frases están pensadas para ser procesadas por el extractor de tripletas.

---

### 3) Resumen → Tripletas (`text2triplets/`)

El módulo convierte el texto en tripletas del tipo:

```bash
(sujeto, predicado, objeto)
```

Las tripletas **deben encajar obligatoriamente** en la estructura fija del dominio.  
Cualquier elemento que no sea compatible se descarta o se registra como *leftover*.

---

### 4) Tripletas → Base de Datos (`triplets2bd/`)

La última capa toma las tripletas válidas y genera:

- **SQL** (SQLite)
- **Cypher** (Neo4j)

Inyectando solo nodos y relaciones existentes en la estructura fija del dominio.

---

## 🧩 Estructura fija del dominio

El dominio del proyecto es **cerrado**.  
No se permiten entidades ni relaciones fuera de lo definido aquí.

### 🗂️ TABLAS PRINCIPALES

```bash
persona(id, user_id, nombre, edad)

sintoma(
  id,
  sintoma_id,
  tipo,
  fecha_inicio,
  fecha_fin,
  categoria,
  frecuencia,
  gravedad
)

actividad(
  id,
  actividad_id,
  nombre,
  categoria,
  frecuencia
)

medicacion(
  id,
  medicacion_id,
  tipo,
  periodicidad
)
```

### 🔗 TABLAS RELACIONALES

```bash
persona_toma_medicacion(persona_id, medicacion_id)
persona_padece_sintoma(persona_id, sintoma_id)
persona_realiza_actividad(persona_id, actividad_id)
```

✔ El pipeline **solo inyecta** información dentro de esta estructura  
✔ No se crean nodos ni campos adicionales  
✔ Las tripletas deben mapear de forma estricta a estas tablas o relaciones

---

## ⚙️ Configuración

### 1. Crear entorno virtual

```bash
py -3.12 -m venv .venv
.venv\Scripts\activate
```

### 2. Instalar dependencias

```bash
pip install -r requirements.txt
```

### 3. Configurar variables de entorno

Crea un archivo `.env` en la raíz del proyecto:

```bash
# --- Neo4j ---
NEO4J_URI=***neo4j_url***
NEO4J_USER=***neo4j_user***
NEO4J_PASSWORD=***neo4j_password***

# --- Backend LLM ---
LLAMUS_BACKEND=OPENAI
LLAMUS_API_KEY=***tu_key***
LLAMUS_URL=***url_base_llamus***
OLLAMA_URL=***url_base_ollama***

# --- OPENAI ---
OPENAI_API_BASE=***url***
OPENAI_API_KEY=***tu_key***

# --- Modelos ---
MODEL_TRIPLETAS_CYPHER=qwen2.5:32b
MODEL_KG_GEN=openai/qwen2.5:14b
MODEL_CONV2TEXT=qwen2.5:32b
MODEL_CONV=qwen2.5:32b

```

---

## 🗣️ Módulos

### 4. Módulo `conv` — Conversación → Paquetitos

El módulo gestiona toda la conversación con el asistente:

- Detecta el nombre del usuario
- Mantiene el historial
- Genera paquetitos listos para el pipeline

**Ejecutar:**

```bash
python -m conv.main_conv
```

**Ejemplo:**

```bash
Bot: Hola, ¿cómo te llamas?
Tú: me llamo Luis
[conv] Nombre detectado: Luis

--- Último paquetito ---
LLM: Mucho gusto, Luis. ¿En qué puedo ayudarte hoy?
user_Luis: quiero hablar sobre mi día
------------------------
```

**Funciones clave:**

- `start_conversation()`
- `conversation_turn()`
- `chat_turn()`
- `name_extractor()`

---

### 5. Módulo `conv2text` — Conversación → Resumen

Convierte los paquetitos en texto limpio.

```bash
python -m conv2text.main_conv2text --text-key TEXT1
```

---

### 6. Módulo `text2triplets` — Texto → Tripletas

```bash
python -m text2triplets.main_kg --text TEXT3
```

**Flags principales:**

| Flag | Uso |
|------|-----|
| `--mode llm/kggen` | Motor de extracción |
| `--text` | Selección de texto |
| `--model` | Modelo LLM |
| `--no-drop` | No descartar inválidas |
| `--generate-report` | Informe SQL |

---

### 7. Módulo `triplets2bd` — Tripletas → SQL / Cypher

```bash
python -m triplets2bd.main_tripletas_bd
```

Inyecta las tripletas en SQLite o en Neo4j.

---

## 🔄 Pipelines del proyecto

El proyecto contiene tres pipelines distintos:

| Script | Conversación | Resetea BD | Imprime | Guarda en archivo | Uso |
|--------|--------------|------------|---------|-------------------|-----|
| **conversation_pipeline.py** | Sí | Sí (inicio) | Sí | Sí (`/pipelines/pipeline.txt`) | Flujo real completo |
| **processing_pipeline.py** | No | No | No | Sí (`/pipelines/pipeline.txt`) | Producción / integración |
| **processing_pipeline_debug.py** | No | Opcional | Sí | No | Depuración compleja |

---

### 🔥 8.1 `conversation_pipeline.py`

El script principal:

- Mantiene conversación real
- Genera paquetitos
- Llama al pipeline silencioso
- Resetea las bases al inicio

**Ejecutar:**

```bash
python -m conversation_pipeline
```

---

### 🧩 8.2 `processing_pipeline.py`

Pipeline silencioso que procesa:

1. conv2text
2. text2triplets  
3. triplets2bd

Guarda todo en:

```bash
pipelines/pipeline.txt
```

---

### 🧪 8.3 `processing_pipeline_debug.py`

Imprime TODO: resumen, tripletas, scripts SQL/Cypher, leftovers, tiempos…

Ideal para depurar.

**Ejecutar:**

```bash
python -m processing_pipeline_debug
```

---

## 🧠 Ejemplo completo de flujo

```bash
# 1. Conversación real
python -m conversation_pipeline

# 2. Ver resultados del pipeline
type pipelines/pipeline.txt

# 3. Debug manual sin conversación
python -m processing_pipeline_debug
```
