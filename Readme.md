
# 🧠 Proyecto: Conversación → Tripletas → Cypher / SQL

Este proyecto transforma lenguaje natural del usuario en **tripletas semánticas**, que posteriormente se convierten en consultas **Cypher** (Neo4j) o **SQL** (SQLite).

El flujo completo puede operar **desde una conversación real** o **desde textos simulados**, y está especialmente diseñado para entornos clínicos y de interacción con personas mayores.

---

## ⚙️ 1. Crear entorno virtual

```bash
py -3.12 -m venv .venv
.venv\Scripts\activate
```

---

## 📦 2. Instalar dependencias

```bash
pip install -r requirements.txt
```

---

## 🔐 3. Configurar variables de entorno

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

# --- App ---
USER_ID=***id_usuario***
```

---

## 🗣️ 4. Módulo `conv` — Conversación → Paquetitos

El módulo `conv/` implementa un asistente conversacional que:

1. Detecta el nombre del usuario automáticamente.
2. Mantiene un histórico interno.
3. **Genera paquetitos** del tipo:

```bash
LLM: <último mensaje del asistente>
user_<nombre>: <texto del usuario>
```

Estos paquetitos alimentan el pipeline.

### ▶️ Ejecutar el conversador

```bash
python -m conv.main_conv
```

**Ejemplo real:**

```bash
Bot: Hola, ¿cómo te llamas?
Tú: me llamo Luis
[conv] Nombre detectado: Luis

Tú: quiero hablar sobre mi día

--- Último paquetito ---
LLM: Mucho gusto, Luis. ¿En qué puedo ayudarte hoy?
user_Luis: quiero hablar sobre mi día
------------------------
```

### Funciones internas clave

* `start_conversation()`
* `conversation_turn()`
* `chat_turn()`
* `name_extractor()`

---

## 🧠 5. Módulo `text2triplets` — Texto → Tripletas

Extrae tripletas desde texto libre usando LLM o KG-Gen.

```bash
python -m text2triplets.main_kg --text TEXT3
```

Flags principales (resumen):

| Flag                | Uso                    |
| ------------------- | ---------------------- |
| `--mode llm/kggen`  | Motor de extracción    |
| `--text`            | Selección de texto     |
| `--model`           | Modelo LLM             |
| `--no-drop`         | No descartar inválidas |
| `--generate-report` | Informe SQL            |

---

## 🚀 6. Módulo `triplets2bd` — Tripletas → SQL / Cypher

```bash
python -m triplets2bd.main_tripletas_bd
```

Permite inyectar las tripletas en SQLite o Neo4j.

---

## 🗣️ 7. Módulo `conv2text` — Conversación → Resumen semántico

```bash
python -m conv2text.main_conv2text --text-key TEXT1
```

Convierte una conversación en frases limpias y explícitas.

---

## 🔄 8. Pipelines del proyecto

Aquí viene la parte más importante: **cómo funciona realmente el proyecto**.

## 🧩 Vista general del sistema de pipelines

El proyecto usa **3 scripts**, cada uno con un propósito claro:

| Script                           | Conversación | Resetea BD          | Imprime en consola   | Guarda en fichero                   | Uso                      |
| -------------------------------- | ------------ | ------------------- | -------------------- | ----------------------------------- | ------------------------ |
| **conversation_pipeline.py**     | Sí (conv/)   | Sí (solo al inicio) | Sí                   | Log interno via processing_pipeline | Flujo real completo      |
| **processing_pipeline.py**       | No           | No                  | No                   | **Sí: `/pipelines/pipeline.txt`**   | Producción / integración |
| **processing_pipeline_debug.py** | No           | Opcional            | **Sí, imprime TODO** | No                                  | Depuración exhaustiva    |

---

## 🔥 8.1 `conversation_pipeline.py` (El más importante)

**Este es el que debes ejecutar para que todo funcione de forma automática.**
Gestiona:

✔ Conversación real
✔ Generación de paquetitos
✔ Envío del paquetito al pipeline
✔ Reset inicial de BD
✔ Ejecución completa hasta SQL/Neo4j

### ▶️ Ejecutarlo

```bash
python -m conversation_pipeline
```

Cuando hablas con el bot:

1. El conversador genera un *paquetito*.
2. El paquetito se envía automáticamente a `processing_pipeline.py`.
3. Este guarda el resultado del pipeline en:

```bash
/pipelines/pipeline.txt
```

👉 **Esto evita saturar la consola** cuando se envían muchos paquetitos seguidos.

---

## 🧩 8.2 `processing_pipeline.py` (Pipeline silencioso)

Este es el pipeline real que procesa el texto (resumen → tripletas → BD), pero:

* **No imprime nada por consola**
* **No resetea la base de datos**
* Guarda todo en:

```bash
pipelines/pipeline.txt
```

### Se usa automáticamente desde

🡆 `conversation_pipeline.py`

### Úsalo cuando

* Quieras procesar decenas de paquetitos sin ruido.
* Necesites un pipeline “de producción”.

---

## 🧪 8.3 `processing_pipeline_debug.py` (Modo depuración total)

Es igual que el pipeline principal, pero:

* Ouiere **debug completo**
* Imprime todo en consola:

  * resumen conv2text
  * tripletas
  * scripts SQL/Cypher
  * leftovers
  * tiempos
* Puede resetear dominios si `CONFIG["reset"] = True`

### ▶️ Ejecutar

```bash
python -m processing_pipeline_debug
```

### Cuándo usarlo

* Para depurar resultados del extractor.
* Para ver exactamente qué entra y sale.
* Para probar la app sin iniciar conversación (**modo simulado**).

---

## 🧠 9. Ejemplo completo de flujo

```bash
# 1. Conversación real
python -m conversation_pipeline

# 2. Revisar el log del pipeline silencioso
type pipelines/pipeline.txt

# 3. Debug manual sin conversación
python -m processing_pipeline_debug
```

---

## 🧾 10. Notas adicionales

* `conversation_pipeline.py` realiza el único reset seguro del proyecto.
* `processing_pipeline.py` existe para no saturar la consola cuando llegan muchos paquetitos.
* `processing_pipeline_debug.py` es tu herramienta de inspección completa.
* Todos los módulos son independientes y se pueden usar de forma aislada.

---

## 📍 11. Créditos

Proyecto desarrollado dentro del entorno de investigación de la **Universidad de Sevilla**, integrando modelos LLM, generación de tripletas, resúmenes semánticos y persistencia en grafos/bases de datos para aplicaciones clínicas y asistenciales.
