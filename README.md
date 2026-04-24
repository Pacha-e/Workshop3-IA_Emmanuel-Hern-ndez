# Workshop 3 — Asistente de Documentos con Gemini y Gradio

**Materia:** Introducción a la Inteligencia Artificial — EAFIT  
**Repositorio base:** [yejaramilm/Introduccion_a_la_Inteligencia_Artificial](https://github.com/yejaramilm/Introduccion_a_la_Inteligencia_Artificial/tree/main/WorkShops/workshop_3)  
**Entregable:** `05_ejercicio.ipynb`

![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python) ![Gradio](https://img.shields.io/badge/Gradio-5.x-orange) ![Gemini](https://img.shields.io/badge/Gemini-2.5%20Flash%20Lite-blue?logo=google) ![License](https://img.shields.io/badge/License-Academic-lightgrey)

---

## Descripción

Asistente conversacional de documentos con RAG real: el texto del PDF se procesa con embeddings locales (`sentence-transformers`), se indexa por chunks y se recuperan solo los fragmentos más relevantes para responder cada pregunta. La interfaz corre en Gradio directamente desde Jupyter Notebook.

## Stack Técnico

| Componente | Biblioteca | Rol |
|---|---|---|
| Extracción PDF | `pypdf` | Lee el PDF y extrae texto plano por página |
| LLM | `google-genai` + Gemini 2.5 Flash Lite | Genera las respuestas |
| Embeddings | `sentence-transformers` (`all-MiniLM-L6-v2`) | Embeddings locales de 384 dims — sin API key |
| Interfaz web | `gradio` | Chat UI en el browser, sin HTML/CSS/JS |
| Credenciales | `python-dotenv` | Carga la API key desde `.env` |
| Entorno | Conda (`environment.yml`) | Aísla dependencias, corre Jupyter |

---

## Mejoras Implementadas

| ID | Descripción | Puerto |
|---|---|---|
| **A** | Conteo real de tokens del system prompt via API de Gemini | — |
| **B** | Subida dinámica de PDF por el usuario | 8081 |
| **C** | El modelo siempre incluye cita textual del paper entre comillas | — |
| **D** | Multi-PDF con citación automática de fuente `[archivo.pdf]` | 8082 |
| **RAG V4** | RAG con embeddings locales (`all-MiniLM-L6-v2`) y recuperación por similitud coseno | 8083 |

---

## Estructura del Proyecto

```
Workshop3-IA/
├── 05_ejercicio.ipynb          # Entregable principal — todas las celdas
├── environment.yml             # Entorno conda (Python 3.11 + dependencias)
├── requirements.txt            # Dependencias pip (referencia rápida)
├── README.md                   # Este archivo
├── .env                        # GEMINI_API_KEY (NO subir — en .gitignore)
├── .gitignore                  # Excluye .env, data/, __pycache__, etc.
└── data/
    ├── attention_is_all_you_need.pdf   # Paper principal (ArXiv 1706.03762)
    └── test/                           # PDFs de prueba para Mejora D
```

---

## Configuración

### Prerrequisito — API key

Crear el archivo `.env` en la raíz del proyecto:

```
GEMINI_API_KEY=tu_clave_aqui
```

### Primera vez (crear entorno conda)

```bash
conda env create -f environment.yml
conda activate workshop3-ia
python -m ipykernel install --user --name=workshop3-ia --display-name "Workshop3-IA"
```

### Cada sesión de trabajo

```bash
conda activate workshop3-ia
jupyter notebook 05_ejercicio.ipynb
```

Jupyter abre en `http://localhost:8888`.  
En el notebook: **Kernel → Change Kernel → Workshop3-IA**

### Actualizar dependencias

```bash
conda env update -f environment.yml --prune
```

### `environment.yml`

```yaml
name: workshop3-ia
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.11
  - jupyter
  - notebook
  - ipykernel
  - pip
  - pip:
    - pypdf>=4.0.0
    - gradio>=5.0.0
    - google-genai>=1.0.0
    - python-dotenv>=1.0.0
    - sentence-transformers>=2.0.0
```

---

## Flujo del Notebook

| Celda | Tipo | Contenido |
|---|---|---|
| 1 | markdown | Contexto y objetivos del ejercicio |
| 2 | markdown | Instrucciones Paso 0 (instalación) |
| 3 | code | `pip install` comentado (conda ya lo tiene) |
| 4 | markdown | Instrucciones Paso 1 |
| 5 | code | **Paso 1** — `extract_text_from_pdf()` + lectura del paper |
| 6 | markdown | Instrucciones Paso 2 |
| 7 | code | **Paso 2** — cliente Gemini + `load_dotenv()` |
| 8 | markdown | Instrucciones Paso 3 |
| 9 | code | **Paso 3** — `get_text`, `build_system_prompt`, `chat_con_documento` |
| 10 | markdown | Instrucciones Paso 4 |
| 11 | code | **Paso 4** — `gr.ChatInterface` en puerto 8080 |
| 12 | markdown | Preguntas de prueba (Paso 5) |
| 13 | markdown | Descripción de las 4 mejoras |
| 14 | code | **Mejora A** — `count_tokens()` real |
| 15 | code | **Mejora B** — subida dinámica de PDF, puerto 8081 |
| 16 | code | **Mejora D** — multi-PDF con citación, puerto 8082 |
| 17–18 | markdown | Reflexión final y recursos |
| 19 | markdown | Documentación RAG V4 |
| 20 | code | `pip install sentence-transformers` |
| 21 | code | **RAG V4** — `build_index`, `buscar_chunks`, modelo `all-MiniLM-L6-v2` |
| 22 | code | Interfaz RAG V4 en puerto 8083 |

---

## Puertos de las Interfaces

| Puerto | Versión | Descripción |
|---|---|---|
| **8080** | V1 — Base | Demo principal, paper fijo inyectado como system prompt |
| **8081** | V2 — Mejora B | Subida dinámica de PDF |
| **8082** | V3 — Mejora D | Multi-PDF con citación por fuente |
| **8083** | V4 — RAG | Embeddings locales + recuperación por similitud coseno |

---

## Referencia Rápida — Comandos

| Tarea | Comando |
|---|---|
| Crear entorno | `conda env create -f environment.yml` |
| Activar entorno | `conda activate workshop3-ia` |
| Registrar kernel | `python -m ipykernel install --user --name=workshop3-ia --display-name "Workshop3-IA"` |
| Abrir notebook | `jupyter notebook 05_ejercicio.ipynb` |
| Actualizar entorno | `conda env update -f environment.yml --prune` |
| Eliminar entorno | `conda env remove -n workshop3-ia` |

---

## Análisis Técnico

<details>
<summary><strong>extract_text_from_pdf(pdf_path)</strong></summary>

```python
reader = PdfReader(pdf_path)
for page in reader.pages:
    text += (page.extract_text() or "") + "\n"
```

- `PdfReader` parsea los streams comprimidos del PDF (no es texto plano).
- `or ""` — `extract_text()` retorna `None` en páginas solo-imagen; `None + "\n"` lanzaría `TypeError`.
- `+ "\n"` — sin el salto, la última palabra de una página y la primera de la siguiente quedarían pegadas.

</details>

<details>
<summary><strong>build_system_prompt(document_text)</strong></summary>

El documento se inyecta completo entre marcadores `--- INICIO/FIN DEL DOCUMENTO ---`. Las instrucciones del prompt definen el comportamiento:

| Instrucción | Motivo |
|---|---|
| Solo del documento | Diferencia el chatbot de uno genérico |
| Respuesta honesta si no está | Sin esto el modelo alucina |
| Sin conocimiento previo | Gemini ya conoce este paper |
| Citar sección | Respuestas verificables |
| Idioma del usuario | Responde en español si se pregunta en español |
| Cita textual (Mejora C) | El usuario puede verificar la fuente |

`system_instruction` tiene mayor peso semántico que un mensaje `user` — el modelo lo trata como directiva permanente, no como input conversacional.

</details>

<details>
<summary><strong>chat_con_documento(message, history, document_text)</strong></summary>

Puntos críticos:

**Mapeo de roles:** Gradio usa `"assistant"`, Gemini acepta solo `"user"` o `"model"`. Sin el mapeo, la API retorna `Invalid role "assistant"`.

**System prompt dentro de la función:** Si el documento cambia (Mejora B), la función siempre usa el texto actual. Construirlo fuera lo "congela" con el texto inicial.

**`yield` vs `return`:** `yield` convierte la función en generador — Gradio activa streaming automático. El texto aparece token por token en vez de esperar 5-10 segundos.

**`get_text(content)`:** En Gradio 5.x, `content` puede llegar como `"Hola"` (string) o `[{"text": "Hola"}]` (lista multimodal). `types.Part(text=...)` requiere siempre un string.

</details>

<details>
<summary><strong>RAG V4 — build_index / buscar_chunks</strong></summary>

```python
_st_model = SentenceTransformer("all-MiniLM-L6-v2")  # 384 dims, ~80 MB, descarga una vez

def build_index(docs):
    all_chunks, all_meta = [], []
    for filename, text in docs.items():
        for i, chunk in enumerate(chunking(text)):
            all_chunks.append(chunk)
            all_meta.append({"file": filename, "idx": i})
    return all_chunks, embed_texts(all_chunks), all_meta

def buscar_chunks(query, chunks, embeddings, meta, k=4):
    query_emb = embed_texts([query])[0]
    norms = np.linalg.norm(embeddings, axis=1) * np.linalg.norm(query_emb)
    similitudes = embeddings @ query_emb / np.where(norms == 0, 1e-10, norms)
    top_k = np.argsort(similitudes)[::-1][:k]
    return [(chunks[i], meta[i], float(similitudes[i])) for i in top_k]
```

- Chunking: 800 chars con overlap de 150 para no cortar ideas a la mitad.
- Similitud coseno: producto punto dividido por producto de normas → valor entre -1 y 1.
- Solo los `k=4` chunks más similares se inyectan en el prompt (~2,000 tokens) en vez del documento completo.

</details>

<details>
<summary><strong>Mejora D — Multi-PDF con citación</strong></summary>

**`docs.update(base_docs)` vs `docs = base_docs`:** La segunda opción es una referencia al mismo objeto. Al agregar un upload con `docs[nombre] = texto` se modificaría `base_docs` para llamadas futuras. `update()` copia los pares sin referencia compartida.

**`isinstance(uploaded_files, list) else [uploaded_files]`:** Gradio puede pasar un string (un archivo) o una lista (varios). Sin esto, `for f in "ruta/archivo.pdf"` iteraría carácter por carácter.

**`os.path.basename(f)`:** Gradio guarda uploads en rutas temporales. `basename` extrae solo `"mi_paper.pdf"` para las citas del modelo.

**Nombre en apertura Y cierre del bloque:** Con documentos largos (miles de tokens), el modelo puede "olvidar" de cuál documento venía el texto. El cierre nombrado reduce errores de atribución.

</details>

---

## Limitaciones y Por Qué Existe RAG

### Inyección completa de documento (V1–V3)

| Tamaño documento | Tokens | % del límite (1M) | Viable |
|---|---|---|---|
| ~15 páginas (paper) | ~11K | 1.1% | ✅ |
| ~100 páginas | ~86K | 8.6% | ✅ costoso |
| ~800 páginas | ~690K | 69% | ⚠️ muy costoso |
| ~1,200 páginas | >1M | >100% | ❌ falla |

Adicionalmente: cada pregunta envía todos los tokens del documento → el costo API crece linealmente.

### RAG (V4) — solución a escala

1. **Indexación previa:** el documento se divide en chunks (~800 chars) y cada chunk se convierte en un embedding de 384 dimensiones.
2. **Búsqueda por relevancia:** la pregunta se embebeda y se calculan similitudes coseno con todos los chunks.
3. **Contexto selectivo:** solo los top-4 chunks (~2,000 tokens) se inyectan en el prompt — independiente del tamaño del documento.

Resultado: documentos de millones de páginas son manejables. El contexto enviado es siempre ~2,000 tokens.

### ¿Por qué "no uses conocimiento general" no es 100% efectivo?

Gemini fue entrenado con texto de internet, que incluye este paper (es público desde 2017). La instrucción influye fuertemente pero no desactiva el conocimiento previo. Para verificarlo: probar con un documento ficticio con datos que el modelo no puede conocer.

---

## Guía de Preguntas — Sustentación

<details>
<summary><strong>Bloque 1 — Extracción de PDF</strong></summary>

**¿Por qué `pypdf` y no abrir el archivo como texto?**  
Los PDFs no son texto plano — contienen streams comprimidos con tipografía, posicionamiento e imágenes. `pypdf` parsea esa estructura y extrae el texto seleccionable de cada página.

**¿Qué pasa si el PDF es un escaneo?**  
`extract_text()` retorna `None`. El `or ""` previene el crash. Para PDFs escaneados se necesitaría OCR (ej. `pytesseract`).

**¿Por qué se agrega `"\n"` entre páginas?**  
Sin el salto, la última palabra de una página y la primera de la siguiente quedan pegadas: `"...attentionThe encoder..."`.

</details>

<details>
<summary><strong>Bloque 2 — Cliente Gemini y configuración</strong></summary>

**¿Para qué sirve `load_dotenv()`?**  
Lee el archivo `.env` y registra sus pares como variables de entorno del proceso Python. Sin él, `os.getenv("GEMINI_API_KEY")` retorna `None`.

**¿Por qué `MODELO` es constante y no un string directo?**  
Si el string aparece en múltiples lugares, cambiar de modelo requiere buscarlo en todos. Con la constante, se cambia una línea.

**¿Por qué `gemini-2.5-flash-lite` y no `gemini-2.5-pro`?**  
Flash-lite es más rápido y económico. Para un paper que ocupa solo el 1.1% del límite de contexto, sus capacidades son más que suficientes.

</details>

<details>
<summary><strong>Bloque 3 — Sistema de chat</strong></summary>

**¿Qué es el system prompt y por qué va en `system_instruction`?**  
Es una directiva de comportamiento con mayor peso que los mensajes del usuario. `system_instruction` en Gemini lo procesa como contexto permanente, no como input conversacional.

**¿Por qué `system_prompt` se construye DENTRO de `chat_con_documento`?**  
Para soportar documentos dinámicos. Si el documento cambia (Mejora B), la función siempre usa el texto actual.

**¿Qué pasa si se envía `role="assistant"` a Gemini?**  
La API retorna error: `Invalid role "assistant". Expected "user" or "model"`.

**¿Por qué `yield` y no `return`?**  
`yield` convierte la función en generador — Gradio detecta el generador y activa streaming automático. El texto aparece token por token, como ChatGPT.

**¿Para qué sirve `get_text(content)`?**  
En Gradio 5.x, `content` puede llegar como string `"Hola"` o lista `[{"text": "Hola"}]`. `types.Part(text=...)` requiere un string. `get_text` normaliza ambos formatos.

</details>

<details>
<summary><strong>Bloque 4 — Interfaz Gradio</strong></summary>

**¿Por qué `gr.Textbox(value=document_text, visible=False)`?**  
`gr.ChatInterface` no tiene parámetro `document_text`. `additional_inputs` son componentes cuyo valor Gradio pasa automáticamente como argumentos extra a `fn`. `visible=False` lo oculta al usuario pero su valor sigue enviándose al servidor.

**¿Por qué `server_name="0.0.0.0"`?**  
Por defecto Gradio escucha en `127.0.0.1` (solo local). `"0.0.0.0"` escucha en todas las interfaces → otras personas en la red pueden acceder por IP.

</details>

<details>
<summary><strong>Bloque 5 — RAG y limitaciones</strong></summary>

**¿Cuál es la limitación principal de inyectar el documento completo?**  
(1) Falla con documentos >~1,200 páginas (supera 1M tokens), (2) costo API crece linealmente con el tamaño en cada pregunta, (3) *lost-in-the-middle*: el modelo puede ignorar información en el centro de contextos muy largos.

**¿Cómo funciona RAG?**  
Divide el documento en chunks, crea embeddings (vectores de significado semántico), y busca los chunks más similares a la pregunta. Solo esos ~2,000 tokens se inyectan en el prompt. Escala a millones de páginas.

**¿Qué es similitud coseno?**  
Mide el ángulo entre dos vectores — 1 si apuntan en la misma dirección (muy similares), 0 si son perpendiculares, -1 si opuestos. No depende de la magnitud de los vectores, solo de su dirección.

</details>

<details>
<summary><strong>Bloque 6 — Mejoras A, B, C, D</strong></summary>

**A — ¿Por qué `count_tokens()` y no estimar con `len(texto)//4`?**  
La estimación de 4 chars/token tiene ±20% de error. `count_tokens()` llama a la API y retorna el conteo exacto del tokenizador real de Gemini. Resultado real: ~10,989 tokens = 1.10% del límite.

**B — ¿Cómo sabe la función si el usuario subió un PDF o no?**  
Gradio pasa la ruta del archivo temporal si se subió algo, o `None` si no. `if pdf_file is not None` detecta el caso.

**C — ¿Dónde está implementada la Mejora C?**  
En la instrucción 6 del system prompt: `"Incluye siempre una cita textual del documento (entre comillas)."` Una sola línea con impacto visible en cada respuesta.

**D — ¿Por qué `docs.update(base_docs)` y no `docs = base_docs`?**  
`docs = base_docs` crea una referencia al mismo objeto. Al agregar un upload, se modificaría `base_docs` para futuras llamadas. `update()` copia los pares sin referencia compartida.

**D — ¿Cuántos documentos pueden cargarse simultáneamente?**  
Con Gemini 2.5 Flash (1M tokens) y papers de ~15 páginas (~13K tokens c/u), el límite teórico es ~76 documentos. En la práctica, 10-15 es un rango estable.

</details>

---

## Bugs Corregidos

**Bug 1 — `page.extract_text()` retorna `None`**

PDFs con páginas solo-imagen devuelven `None`. `None + "\n"` lanza `TypeError`.

```python
# Antes (bug):
text += page.extract_text() + "\n"

# Después (fix):
text += (page.extract_text() or "") + "\n"
```

**Bug 2 — `"""` visible al final de celdas markdown**

Varias celdas markdown tenían `"""` como último elemento en el `source` del JSON del notebook. Se eliminó de las celdas 4, 6, 8 y 10.

---

## Recursos

- [Paper original — ArXiv 1706.03762](https://arxiv.org/abs/1706.03762)
- [Documentación pypdf](https://pypdf.readthedocs.io)
- [Documentación Gradio](https://www.gradio.app/docs)
- [Documentación Gemini API](https://ai.google.dev/gemini-api/docs)
- [sentence-transformers](https://www.sbert.net/docs/sentence_transformer/pretrained_models.html)
