# Ejercicio 2 — Extracción de información de reseñas de videojuegos con LLMs

A partir de un dataset "crudo" de 20.000 reseñas de videojuegos
(`videogames_reviews.csv`), el notebook "Ejercicio 2.ipynb" extrae información estructurada (sentimiento,
aspectos positivos/negativos, dificultad, etc.) usando **dos LLMs encadenados**.
El resultado final se exporta a `resultado_ejercicio_2.csv`.

---

## Pipeline del notebook

El proceso está dividido en pasos, cada uno en su propia celda:

| Paso | Qué hace | Salida |
|------|----------|--------|
| **0. Exploración** | Carga el CSV, revisa nulos, tipos y balance de clases. | — |
| **1. Preprocesado** | Elimina nulos y selecciona las **100 reseñas con el contenido más largo**. | `df_top_100` |
| **2. Filtrado (LLM 1)** | Un primer LLM descarta las reseñas que no aportan información útil. | `df_relevantes` |
| **3. Extracción (LLM 2)** | Un segundo LLM extrae las entidades de cada reseña relevante. | `df_analisis` |
| **4. Exportación** | Guarda el resultado en `resultado_ejercicio_2.csv`. | CSV final |

---

## Diseño y claridad de los prompts

Se usan **dos prompts distintos**, cada uno con una única tarea bien acotada:

- **Prompt de filtrado (Paso 2):** define qué es una reseña "relevante"
  (menciona gameplay, gráficos, historia, dificultad, bugs, precio, sonido…) y
  obliga al modelo a responder **únicamente con un JSON válido**
  (`{"relevantes": [ids...]}`), sin texto extra. Esto hace la respuesta
  directamente parseable con `json.loads`.
- **Prompt de extracción (Paso 3):** pide, para cada reseña, un objeto JSON con
  campos cerrados (p. ej. `SentimientoGeneral` restringido a
  `Positivo|Negativo|Neutral`, `Dificultad` a una lista fija de valores). Al
  limitar los valores posibles se reduce la variabilidad de la salida y se
  facilita el análisis posterior.
- **Robustez por ID:** cada reseña se envía precedida de su identificador real
  (`[ID=numero]`) y se le exige al modelo devolver ese mismo `Id` sin
  modificarlo. El emparejamiento posterior se hace **por ID, no por posición**,
  de modo que aunque el LLM omita o reordene reseñas, cada resultado conserva su
  identidad correcta.

---

## Separación de tareas entre distintos LLMs

El trabajo se reparte en **dos llamadas conceptualmente separadas** en lugar de
pedir todo de una vez:

1. **LLM 1 — Clasificar/filtrar:** decide *qué* reseñas merece la pena analizar.
2. **LLM 2 — Extraer:** obtiene las entidades solo de las reseñas que pasaron el
   filtro.

Los modelos rinden mejor cuando se les pide **una tarea a la vez**. Separar
"filtrar" de "extraer" evita mezclar dos objetivos en un mismo prompt, mejora la
calidad de cada salida y hace el pipeline más fácil de depurar (si algo falla,
se sabe en qué etapa).

---

## Elección justificada del proveedor y modelo

- **Proveedor:** **Google Gemini** vía API (SDK `google-genai`).
- **Modelo:** `gemini-3.1-flash-lite`, un modelo **ligero y rápido**, suficiente
  para tareas de clasificación y extracción estructurada, con un coste y una
  latencia menores que un modelo "full".
- **Vía API vs. local (Hugging Face):** se optó por API porque no requiere GPU
  local ni descargar pesos, y ofrece buena calidad inmediata. La contrapartida
  son los **rate limits** del plan, que se gestionan explícitamente (ver abajo).
- La API key se lee de un archivo `.env` (`GOOGLE_API_KEY`) con `python-dotenv`,
  para **no exponer credenciales** en el notebook.

---

## Gestión de limitaciones

### Tokens máximos por llamada
- Se envían **lotes de 20 reseñas con el texto completo** (sin truncar).
- Peor caso estimado por lote: `20 × ~2.650 tokens ≈ 53.000 tokens` de entrada,
  muy por debajo del límite de contexto del modelo, dejando margen para la
  respuesta.

### Rate limits (TPM — tokens por minuto)
- El plan usado tiene un límite aproximado de **250.000 TPM**.
- Con `SLEEP_ENTRE_LOTES = 15s`, las 5 llamadas de cada paso se reparten a lo
  largo de ~60s, de forma que **nunca coinciden más de 4 lotes** en una ventana
  deslizante de 60 segundos: `4 × ~56.300 ≈ 225.000 tokens < 250.000 TPM`.
- Como **red de seguridad** adicional, `generar_con_backoff()` captura errores
  transitorios **429 (rate limit)** y **503 (sobrecarga)** y reintenta con
  **espera exponencial + jitter** (5s, 10s, 20s… + aleatorio), hasta
  `MAX_REINTENTOS = 4`. El *jitter* evita que varias llamadas reintenten a la vez.

### Tamaño de los lotes de filas por llamada
- `BATCH_SIZE = 20` en ambos pasos. Es un equilibrio entre:
  - **Menos llamadas** (menos overhead y menos riesgo de tocar el límite de
    peticiones por minuto), y
  - **No saturar** el contexto ni el TPM por llamada.
- 100 reseñas ÷ 20 = **5 llamadas por paso**.

---

## Justificación de decisiones técnicas

- **`time.sleep(SLEEP_ENTRE_LOTES)` = 15s entre lotes:** reparte las llamadas en
  el tiempo para mantener el TPM por debajo del límite. Un sleep menor (p. ej.
  4s) agrupaba las 5 llamadas en ~20s, sumando ~266K tokens en una misma ventana
  de 60s y arriesgando un error 429 por TPM.
- **Número de filas por batch = 20:** justificado arriba (equilibrio entre
  número de llamadas y tokens por llamada).
- **Número de llamadas al modelo:** **5 por paso** (2 pasos → 10 llamadas en
  total para las 100 reseñas). El diseño busca minimizar llamadas sin superar
  los límites por llamada.
- **Backoff exponencial + jitter:** hace el pipeline tolerante a fallos
  transitorios en lugar de perder un lote entero ante el primer 429/503.

---

## Requisitos y ejecución

1. Crear un archivo `.env` en la raíz con la clave de la API:
   ```
   GOOGLE_API_KEY=tu_clave_aqui
   ```
2. Instalar dependencias:
   ```
   pip install -r requirements.txt
   ```
   (principales: `pandas`, `google-genai`, `python-dotenv`)
3. Abrir `Ejercicio 2.ipynb` y ejecutar las celdas **de arriba abajo**.

## Salida

El notebook genera **`resultado_ejercicio_2.csv`** (codificado en `utf-8-sig`
para que los acentos se muestren bien también en Excel) con la información
extraída de cada reseña relevante.
