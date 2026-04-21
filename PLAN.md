# Workshop 3 - Asistente de Documentos con Gemini y Gradio

## Materia: Introducción a la Inteligencia Artificial - EAFIT
## Fuente: https://github.com/yejaramilm/Introduccion_a_la_Inteligencia_Artificial/tree/main/WorkShops/workshop_3

---

## Contexto

Construir un asistente que responda preguntas **exclusivamente** sobre el paper
"Attention is All You Need" (Vaswani et al., 2017). Técnica base de RAG.

## Objetivo

App Gradio donde el usuario: (1) carga PDF, (2) hace preguntas, (3) obtiene respuestas solo del documento.

## Entregable

**`05_ejercicio.ipynb`** — único archivo a subir a GitHub. Todo el código va en celdas del notebook.

---

## Estructura del Proyecto

```
Workshop3-IA/
├── PLAN.md                    # Este archivo - plan detallado
├── requirements.txt           # Dependencias Python
├── .env                       # API key Gemini (NO subir a GitHub)
├── .gitignore                 # Excluir .env, data/, __pycache__/, .venv/
├── data/
│   └── attention_is_all_you_need.pdf  # Paper de ArXiv
└── 05_ejercicio.ipynb         # Notebook final para entrega
```

---

## Fase 0: Setup del Entorno

### 0.1 Crear entorno virtual
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```
Aisla dependencias del proyecto. Evita conflictos con paquetes globales.

### 0.2 Instalar dependencias
```powershell
pip install pypdf gradio google-genai python-dotenv
```
- `pypdf`: Lee PDFs y extrae texto plano de cada página
- `gradio`: Crea interfaces web desde Python con mínimo código
- `google-genai`: SDK oficial de Google para modelos Gemini
- `python-dotenv`: Carga variables de entorno desde `.env`

### 0.3 Descargar PDF
- URL: `https://arxiv.org/pdf/1706.03762`
- Guardar como: `data/attention_is_all_you_need.pdf`
- ~15 páginas, ~10K palabras, ~13K tokens

### 0.4 Crear `.env`
```
GEMINI_API_KEY="tu_api_key_aqui"
```
Ya está en `.gitignore` → no se sube a GitHub.

### 0.5 Verificar `.gitignore`
```
.env          # API key secreta
data/         # PDF descargable de ArXiv
__pycache__/  # Basura compilada Python
*.pyc         # Basura compilada Python
.venv/        # Entorno virtual local
```

---

## Fase 1: Celda de Instalación (Paso 0 del ejercicio)

**Ubicación:** Primera celda de código del notebook

```python
# %%bash
# pip install pypdf gradio google-genai python-dotenv
```
Comentada porque en Colab/JupyterLab suelen estar instaladas. `%%bash` = magic command de Jupyter para shell.

---

## Fase 2: Celda de Extracción de PDF (Paso 1 del ejercicio)

**Ubicación:** Segunda celda de código

```python
from pypdf import PdfReader
```
Importa `PdfReader` — clase que abre PDF y permite acceder a páginas y extraer texto.

```python
def extract_text_from_pdf(pdf_path: str) -> str:
    """Extrae todo el texto de un archivo PDF.
    Args: pdf_path - Ruta al PDF (ej: "data/attention_is_all_you_need.pdf")
    Returns: Texto completo como string, todas las páginas concatenadas
    """
    reader = PdfReader(pdf_path)  # Abre el PDF, lee estructura interna
    text = ""
    for page in reader.pages:     # reader.pages = lista de objetos Page (1 por página)
        text += page.extract_text() + "\n"  # extract_text() = texto plano de la página
    return text                   # String gigante con todo el documento
```
- `PdfReader(pdf_path)`: Abre el archivo y lee su estructura
- `reader.pages`: Lista de objetos Page, uno por cada página del PDF
- `page.extract_text()`: Devuelve texto plano de una página (sin formato)
- `+ "\n"`: Salto de línea entre páginas para que no se peguen
- Retorna un string con todo el texto concatenado

```python
# Prueba
pdf_path = "data/attention_is_all_you_need.pdf"
document_text = extract_text_from_pdf(pdf_path)  # Variable que se reutiliza después

reader = PdfReader(pdf_path)
print(f"Numero de paginas: {len(reader.pages)}")       # ~15
print(f"Caracteres extraidos: {len(document_text):,}")  # ~45,000 (coma de miles)

print("\n--- Primeros 500 caracteres ---")
print(document_text[:500])  # Verificar que el texto tiene sentido (título, autores, abstract)
```
- `document_text`: Variable clave — se usa después en el system prompt y como input de Gradio
- `[:500]`: Slice de Python, primeros 500 caracteres para verificación visual

---

## Fase 3: Celda de Cliente Gemini (Paso 2 del ejercicio)

**Ubicación:** Tercera celda de código

```python
import os                       # Para os.getenv() — leer variables de entorno
from dotenv import load_dotenv  # Carga .env como variables de entorno
from google import genai        # SDK de Google — clase genai.Client
from google.genai import types  # types.Content, types.Part, types.GenerateContentConfig
```
- `os.getenv()`: Lee variables de entorno del sistema
- `load_dotenv()`: Busca `.env` y registra sus líneas como variables de entorno
- `genai.Client`: Punto de entrada para llamadas a la API de Gemini
- `types`: Clases de datos que Gemini requiere para mensajes y configuración

```python
load_dotenv()  # Sin esto, os.getenv("GEMINI_API_KEY") retorna None

if os.getenv("GEMINI_API_KEY"):
    print("Gemini API Key cargada correctamente")
else:
    print("ERROR: GEMINI_API_KEY no encontrada. Verifica tu archivo .env")
```
Verificación temprana — si la key falta, todo fallará después.

```python
client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))
```
Crea la instancia del cliente. Tiene los métodos `.models.generate_content()` y
`.models.generate_content_stream()`. Sin key válida, la API rechaza con 401/403.

```python
MODELO = "gemini-2.5-flash-lite"
```
- "flash-lite" = rápido y económico, ideal para ejercicios
- Ventana de contexto: 1,000,000 tokens (1M) — el paper son ~13K = 1.3% del límite
- Constante para cambiar modelo en un solo sitio

---

## Fase 4: Celda de Chat con Contexto (Paso 3 del ejercicio) — PARTE CENTRAL

**Ubicación:** Cuarta celda de código. Contiene 3 funciones:

### Función 1: `get_text(content) -> str`

```python
def get_text(content) -> str:
    """Extrae texto de un contenido de mensaje de Gradio,
    sin importar si llega como string o lista de dicts."""
    if isinstance(content, str):
        return content                                    # Caso simple: texto puro
    elif isinstance(content, list):
        return content[0].get("text", "") if content else ""  # Gradio 5.x multimodal
    return str(content)                                   # Fallback (None, etc.)
```
**Por qué existe:** En Gradio 5.x, `content` puede ser string `"Hola"` o lista
`[{"text": "Hola"}]` (formato multimodal). Si pasamos una lista a `types.Part(text=...)`,
Gemini crashea. Esta función normaliza ambos formatos a string.

### Función 2: `build_system_prompt(document_text) -> str`

```python
def build_system_prompt(document_text: str) -> str:
    """Construye el system prompt con el texto completo del documento.
    El system prompt se pasa vía system_instruction en Gemini — tiene mayor
    peso que mensajes del usuario (directiva de comportamiento, no conversación)."""
    return f"""Eres un experto asistente académico especializado en el paper
"Attention is All You Need" (Vaswani et al., 2017).

A continuación se te proporciona el texto COMPLETO del paper:

--- INICIO DEL DOCUMENTO ---
{document_text}
--- FIN DEL DOCUMENTO ---

INSTRUCCIONES IMPORTANTES:
1. Responde EXCLUSIVAMENTE con información que esté en el documento proporcionado.
2. Si la pregunta no puede ser respondida con la información del documento,
   responde: "La información solicitada no se encuentra en el documento proporcionado."
3. NO uses tu conocimiento general para responder, incluso si conoces la respuesta.
4. Cuando sea relevante, menciona la sección del paper donde se encuentra la información.
5. Responde en el mismo idioma en que el usuario hace la pregunta."""
```
**Línea por línea del contenido:**
- **Rol "experto asistente"**: Los LLMs funcionan mejor con identidad clara
- **Delimitadores INICIO/FIN**: Sin ellos, el modelo confunde documento con instrucciones
- **`{document_text}`**: Se inserta TODO el paper (~13K tokens). Posible porque el límite es 1M
- **Instrucción 1**: Restricción clave — diferencia este chatbot de uno genérico
- **Instrucción 2**: Sin esto, el modelo alucina. Forzamos respuesta honesta
- **Instrucción 3**: Gemini conoce este paper de su entrenamiento — sin esto mezclaría fuentes
- **Instrucción 4**: Hace respuestas verificables
- **Instrucción 5**: Si preguntan en español, responde en español

### Función 3: `chat_con_documento(message, history, document_text)`

```python
def chat_con_documento(message: str, history: list, document_text: str):
    """Función que Gradio llama cada vez que el usuario envía un mensaje.
    Args:
        message: Mensaje actual del usuario (string)
        history: Historial en formato Gradio 5.x: [{"role": "user"/"assistant", "content": "..."}]
        document_text: Texto del documento (pasado vía additional_inputs de Gradio)
    Yields: Respuesta acumulada (streaming token por token)
    """
    # 1. Construir system prompt con el documento
    system_prompt = build_system_prompt(document_text)

    # 2. Convertir historial de Gradio al formato Gemini (types.Content)
    contenido = []
    for entry in history:
        if isinstance(entry, dict):
            # MAPEO CRÍTICO: Gradio "assistant" → Gemini "model"
            role = "model" if entry["role"] == "assistant" else "user"
            contenido.append(types.Content(
                role=role,
                parts=[types.Part(text=get_text(entry["content"]))]
            ))
        elif isinstance(entry, (list, tuple)) and len(entry) == 2:
            # Formato alternativo: [user_msg, assistant_msg]
            user_msg, assistant_msg = entry
            contenido.append(types.Content(role="user", parts=[types.Part(text=get_text(user_msg))]))
            if assistant_msg:
                contenido.append(types.Content(role="model", parts=[types.Part(text=get_text(assistant_msg))]))

    # 3. Agregar mensaje actual del usuario
    contenido.append(types.Content(role="user", parts=[types.Part(text=message)]))

    # 4. Llamar a Gemini en modo streaming
    respuesta = ""
    for chunk in client.models.generate_content_stream(
        model=MODELO,
        config=types.GenerateContentConfig(system_instruction=system_prompt),
        contents=contenido
    ):
        if chunk.text:
            respuesta += chunk.text
            yield respuesta  # Gradio detecta yield → activa streaming automático
```
**Puntos clave:**
- `system_prompt` se construye dentro de la función (no afuera) para soportar subida dinámica de PDF
- `contenido = []`: Gemini requiere `types.Content`, no acepta strings sueltos
- **Mapeo `"assistant"` → `"model"`**: Si enviamos "assistant", Gemini lanza error de rol inválido
- `types.Content(role=..., parts=[types.Part(text=...)])`: Formato obligatorio de Gemini
- `generate_content_stream()`: Devuelve chunks progresivamente en vez de esperar toda la respuesta
- `yield respuesta`: Hace la función generador → Gradio activa streaming automático (texto aparece token por token)

---

## Fase 5: Celda de Interfaz Gradio (Paso 4 del ejercicio)

**Ubicación:** Quinta celda de código

```python
import gradio as gr  # Convierte funciones Python en interfaces web sin HTML/CSS/JS
```

```python
# Extraer texto del PDF
pdf_path = "data/attention_is_all_you_need.pdf"
document_text = extract_text_from_pdf(pdf_path)
reader = PdfReader(pdf_path)
print(f"Documento cargado: {len(reader.pages)} paginas, {len(document_text):,} caracteres")
```

```python
demo = gr.ChatInterface(
    fn=chat_con_documento,  # Gradio llama: fn(message, history, *additional_inputs)
    title="Asistente del Paper: Attention is All You Need",
    description="""Pregúntale cualquier cosa sobre el paper "Attention is All You Need"
    (Vaswani et al., 2017). El asistente responde EXCLUSIVAMENTE con información del documento.""",
    additional_inputs=[
        gr.Textbox(value=document_text, visible=False)  # Oculto: ~45K chars, se pasa como document_text
    ],
    examples=[
        "¿Cuál es la arquitectura principal propuesta en el paper?",
        "¿Qué es el mecanismo de atención multi-cabeza?",
        "¿Cuántas capas tiene el encoder del modelo base?",
        "¿Quiénes son los autores del paper?",
        "¿Cuál es el resultado del modelo en WMT 2014 English-to-German?",
    ],
    flagging_mode="never"  # Desactiva botón de reportar respuestas
)
```
**Puntos clave:**
- `gr.ChatInterface`: Componente de alto nivel para chatbots — maneja burbujas, historial, scroll
- `additional_inputs`: Componentes extra que Gradio pasa como args adicionales a `fn`
- `gr.Textbox(value=document_text, visible=False)`: Oculto al usuario, pero su valor se pasa
  automáticamente a `chat_con_documento` como parámetro `document_text`
- `examples`: Preguntas clicables debajo del campo de entrada
- `flagging_mode="never"`: Sin botón de flag

```python
demo.launch(server_name="0.0.0.0", server_port=8080, show_error=True)
```
- `"0.0.0.0"`: Escucha en todas las interfaces (accesible desde otra máquina)
- `8080`: Puerto del servidor web
- `show_error=True`: Muestra excepciones en la interfaz (útil para debugging)

---

## Fase 6: Pruebas (Paso 5 del ejercicio)

**Celda markdown** con las preguntas a probar:

1. "¿Cuál es la arquitectura principal propuesta en el paper?"
2. "¿Qué es el mecanismo de atención?"
3. "¿Cuántas capas tiene el encoder del modelo base?"
4. "¿Quiénes son los autores del paper?"
5. "¿Cuál es el resultado del modelo en la tarea WMT 2014 English-to-German?"
6. **Pregunta trampa:** "¿Qué es GPT-4?" → NO está en el paper. Verificar que el modelo
   respeta la instrucción de ceñirse al documento y NO usa conocimiento general.

---

## Fase 7: Mejora Opcional (Paso 6 del ejercicio)

Elegir **una** de tres:

### A) Indicador de tokens
Mostrar cuántos tokens usa el system prompt vs el límite de 1M de Gemini 2.5 Flash.
```python
# Estimación: ~4 caracteres por token en inglés
estimated_tokens = len(system_prompt) // 4
print(f"Tokens estimados del system prompt: {estimated_tokens:,}")
print(f"Porcentaje del límite 1M: {estimated_tokens/1_000_000*100:.2f}%")
```

### B) Subida dinámica de PDF
Agregar `gr.File` en `additional_inputs` para subir cualquier PDF desde la interfaz.
```python
additional_inputs=[
    gr.Textbox(value=document_text, visible=False),  # texto por defecto
    gr.File(label="Sube otro PDF", file_types=[".pdf"])  # uploader
]
```
Modificar `chat_con_documento` para extraer texto del archivo subido cuando llegue.

### C) Citas del documento
Modificar system prompt para que el modelo incluya cita textual del paper en cada respuesta.
Agregar a las instrucciones: "Siempre incluye una cita textual del documento que respalde tu respuesta."

---

## Fase 8: Reflexión Final (Celda markdown)

Documentar en el notebook:

1. **¿Cuál es la limitación principal de este enfoque?** — Con un documento de 1000 páginas
   se excedería el límite de tokens del modelo
2. **¿Por qué existe RAG?** — RAG resuelve el problema buscando solo los fragmentos
   relevantes del documento en vez de inyectar todo el texto
3. **¿Qué información podría "filtrarse"?** — Gemini conoce este paper de su entrenamiento.
   Para verificar que responde desde el documento y no desde su conocimiento, comparar
   respuestas con y sin el documento en el system prompt

---

## Fase 9: Entrega

1. Verificar que todas las celdas del notebook ejecutan sin error
2. Agregar comentarios explicativos en cada celda
3. Subir `05_ejercicio.ipynb` a GitHub (sin `.env`, sin `data/`)

---

## Notas Técnicas

- **Modelo:** `gemini-2.5-flash-lite` (1M tokens contexto)
- **Paper:** ~10K palabras ≈ ~13K tokens = 1.3% del límite
- **Streaming:** `yield` en Gradio = streaming automático
- **Mapeo de roles:** Gradio `"assistant"` → Gemini `"model"` (ERROR si se envía "assistant")
- **Historial Gradio 5.x:** Lista de dicts `{"role": "user"/"assistant", "content": ...}`
- **additional_inputs:** Gradio los pasa como parámetros extra a `fn(message, history, *additional)`

---

## Bugs Corregidos (vs versión original)

### Bug 1: `page.extract_text()` retorna None en páginas imagen
**Ubicación:** `cell-5`, línea `text += page.extract_text() + "\n"`

**Problema:** Si una página del PDF es solo imagen (sin texto seleccionable), `extract_text()`
retorna `None`. La expresión `None + "\n"` lanza `TypeError` y el notebook se rompe.

**Fix aplicado:**
```python
# Antes (bug):
text += page.extract_text() + "\n"

# Después (corregido):
text += (page.extract_text() or "") + "\n"
```
`or ""` convierte `None` en string vacío antes de concatenar.

### Bug 2: `"""` visible al final de 4 celdas markdown
**Ubicación:** Celdas markdown de los Pasos 1, 2, 3 y 4 (cells 4, 6, 8, 10)

**Problema:** Cada celda terminaba con `"""` como último elemento del source array JSON.
Al renderizar el notebook en Jupyter/VS Code, ese `"""` aparecía como texto literal al
final de las instrucciones — feo y confuso para el lector.

**Fix aplicado:** Se eliminó el `"""` sobrante del final de cada celda markdown.

---

## Mejora Implementada: C — Citas Textuales

Implementada en `build_system_prompt()` mediante la instrucción 6:
```python
"6. Incluye siempre una cita textual del documento (entre comillas) que respalde tu respuesta."
```

**Por qué Mejora C y no A o B:**
- **A (tokens):** Solo imprime un número, no cambia el comportamiento del asistente
- **B (PDF dinámico):** Requiere más cambios (modificar `fn` y `additional_inputs`)
- **C (citas):** Una sola línea en el system prompt, impacto directo y visible en la UI —
  cada respuesta ahora incluye una cita del paper que el usuario puede verificar

La celda `cell-14` incluye además el cálculo de tokens (Mejora A) como demostración.

---

## Guía de Sustentación — Preguntas y Respuestas

### BLOQUE 1: Extracción de PDF

**P: ¿Por qué usamos `pypdf` y no simplemente abrir el archivo como texto?**
R: Los PDFs no son archivos de texto plano. Contienen streams comprimidos con tipografía,
posicionamiento, imágenes y metadatos. `pypdf` parsea esa estructura interna y extrae el
texto seleccionable de cada página como string legible.

**P: ¿Qué pasa si el PDF es un escaneo (imagen)?**
R: `page.extract_text()` retorna `None` porque no hay texto seleccionable. El fix `or ""`
previene el crash. Para PDFs escaneados se necesitaría OCR (p.ej. `pytesseract`).

**P: ¿Por qué se agrega `"\n"` entre páginas?**
R: Sin `"\n"`, la última palabra de una página y la primera de la siguiente quedarían pegadas
("...attentionThe encoder..."). El salto de línea mantiene la legibilidad del texto extraído.

---

### BLOQUE 2: Cliente Gemini

**P: ¿Para qué sirve `load_dotenv()`?**
R: Lee el archivo `.env` del disco y registra sus pares `CLAVE=valor` como variables de entorno
del proceso Python. Sin `load_dotenv()`, `os.getenv("GEMINI_API_KEY")` retorna `None` y el
cliente se crea sin key, fallando en la primera llamada a la API.

**P: ¿Por qué `MODELO` es una constante y no un string directo en la llamada?**
R: Si el string `"gemini-2.5-flash-lite"` aparece en múltiples lugares y necesitas cambiar de
modelo, tienes que buscarlo y reemplazarlo en todos lados. Con la constante, cambias una línea.

**P: ¿Qué es `genai.Client`?**
R: Es el objeto principal del SDK de Google. Actúa como sesión HTTP autenticada con la API de
Gemini. Contiene el sub-objeto `.models` con los métodos `generate_content()` y
`generate_content_stream()` para hacer las llamadas.

---

### BLOQUE 3: Sistema de Chat (parte más importante)

**P: ¿Qué es el system prompt y por qué se pasa como `system_instruction` y no como un mensaje?**
R: El system prompt es una directiva de comportamiento para el modelo, con mayor peso semántico
que los mensajes del usuario. Gemini lo procesa como contexto permanente (no como turno de
conversación). Si se pasara como mensaje `user`, el modelo lo trataría como input del usuario,
no como instrucción.

**P: ¿Por qué el system prompt se construye DENTRO de `chat_con_documento` y no fuera?**
R: Para soportar documentos dinámicos. Si en el futuro `document_text` cambia (por ej., el
usuario sube otro PDF en la misma sesión), la función siempre construye el prompt con el
texto actual. Si se construyera fuera, quedaría "congelado" con el texto inicial.

**P: ¿Por qué hay dos formatos en el manejo del historial (dict y list/tuple)?**
R: Gradio 5.x usa el formato dict `{"role": "...", "content": "..."}`. Gradio 4.x usaba
tuplas `[user_msg, assistant_msg]`. El código maneja ambos para máxima compatibilidad.

**P: ¿Qué pasa si se envía `role="assistant"` a Gemini en vez de `"model"`?**
R: La API retorna un error: `Invalid role "assistant". Expected "user" or "model"`. Por eso el
mapeo es crítico: `role = "model" if entry["role"] == "assistant" else "user"`.

**P: ¿Por qué `yield` y no `return`?**
R: `return` enviaría toda la respuesta al final (el usuario espera 5-10 segundos sin ver nada).
`yield` convierte la función en generador — cada vez que el bucle recibe un chunk de Gemini,
lo entrega inmediatamente a Gradio, que lo muestra en pantalla. El usuario ve el texto
aparecer token por token, como en ChatGPT.

**P: ¿Qué es `generate_content_stream` vs `generate_content`?**
R: `generate_content` espera a que Gemini complete toda la respuesta y la retorna de una vez.
`generate_content_stream` retorna un iterador de chunks — cada chunk es un fragmento del texto
a medida que el modelo lo genera. Para streaming en UI, siempre se usa la versión `_stream`.

**P: ¿Para qué sirve `get_text(content)`?**
R: En Gradio 5.x con componentes multimodales, el campo `content` puede llegar como string
`"Hola"` o como lista `[{"text": "Hola"}]`. `types.Part(text=...)` requiere un string.
`get_text` normaliza ambos formatos para que el código no crashee con el formato lista.

---

### BLOQUE 4: Interfaz Gradio

**P: ¿Por qué `gr.Textbox(value=document_text, visible=False)`?**
R: `gr.ChatInterface` no tiene parámetro `document_text`. Para pasar datos extra a la función
de chat, Gradio ofrece `additional_inputs`. Estos son componentes UI cuyo valor se pasa
automáticamente como argumento extra a `fn`. `visible=False` lo oculta de la pantalla pero
su valor (`document_text`) sigue siendo parte del estado del componente y Gradio lo envía.

**P: ¿Qué son los `examples`?**
R: Preguntas pre-escritas que aparecen como botones clicables bajo el campo de texto. Al hacer
clic, la pregunta se inserta automáticamente en el input. Son útiles para demostrar la
funcionalidad al evaluar la app.

**P: ¿Por qué `server_name="0.0.0.0"`?**
R: Por defecto Gradio escucha en `127.0.0.1` (solo local). `"0.0.0.0"` significa "escucha en
todas las interfaces de red", incluyendo la IP de la máquina en la red local. Permite que
otras personas en la misma red accedan a la app (útil para demos en clase).

**P: ¿Qué hace `flagging_mode="never"`?**
R: Desactiva el botón "Flag" que Gradio muestra por defecto para que usuarios reporten
respuestas malas. Lo quitamos porque no tenemos backend para manejar esos reportes.

---

### BLOQUE 5: RAG y limitaciones

**P: ¿Cuál es la limitación de este enfoque con documentos grandes?**
R: Este enfoque inyecta el documento COMPLETO en el context window. Con Gemini 2.5 Flash (1M
tokens) funciona para documentos hasta ~800 páginas, pero: (1) el costo API sube con cada
token enviado, (2) con documentos más grandes excede el límite y falla, (3) el modelo puede
"perderse" en contextos muy largos (lost in the middle problem).

**P: ¿Cómo funciona RAG para resolver esto?**
R: RAG divide el documento en chunks, crea embeddings (vectores numéricos del significado de
cada chunk), y cuando llega una pregunta, busca los chunks más similares semánticamente.
Solo esos chunks (no el documento completo) se inyectan en el prompt. Así, documentos de
10,000 páginas son manejables porque el contexto siempre tiene ~2,000 tokens relevantes.

**P: ¿Por qué la instrucción "no uses conocimiento general" no es 100% efectiva?**
R: Los LLMs no tienen un interruptor que desactive su conocimiento previo. La instrucción
influye en el comportamiento pero no lo garantiza. El modelo puede mezclar lo que lee en el
documento con lo que ya sabe, especialmente para papers famosos como este.
